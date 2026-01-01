# 第11章｜LLM Toolcall & Agent：导航工具设计、澄清策略与可靠执行

## 1. 开篇：跨越“概率”与“确定”的鸿沟

在前十章中，我们构建了一个基于 OSM 数据的**确定性系统**：Nominatim 无论是搜索一次还是搜索一万次，对于相同的输入，其返回的坐标是固定的；OSRM 对于相同的起终点，计算出的路线也是数学上唯一的（最短路径或最优路径）。

然而，Agent 的核心——大语言模型（LLM）——是一个**概率系统**。它擅长“模糊理解”和“创造”，却极其不擅长“精确记忆”和“空间计算”。

当你把这两者结合时，会发生奇妙但危险的化学反应：
*   **理想情况**：Agent 像一位经验丰富的向导，理解你“想去海边看日落”的意图，自动搜索朝西的沙滩，规划路线，并告诉你“现在出发刚好能赶上”。
*   **灾难情况**：Agent 编造了一个不存在的经纬度（幻觉），调用路由引擎，引擎报错，Agent 试图“圆谎”，告诉你“正在为您开启飞行模式”。

**本章目标**：
1.  **定义标准**：设计一套让 LLM “不得不”遵守的 Tool Schema 规范。
2.  **管理不确定性**：构建一套“搜索-澄清-确认”的对话流，解决自然语言的歧义。
3.  **编排链路**：设计中间件逻辑，确保地理参数在工具间传递时不被篡改。

---

## 2. Agent 导航的认知架构与动力学

在设计工具之前，我们必须先理解 Agent 是如何“思考”导航任务的。我们需要在 System Prompt 和工具描述中植入正确的思维模型。

### 2.1 任务动力学模型（The Navigation Dynamics）

一个成熟的导航 Agent 处理任务的流程并非性的，而是一个包含**感知、决策、行动、反馈**的循环。

```ascii
USER INTENT: "I want to grab a coffee, then go to the office."
      |
      v
[ PHASE 1: Semantic Parsing & Planning ]
      |-- Intent: Multi-stop Routing (Waypoint Navigation)
      |-- Entities: "Coffee" (Category), "Office" (Specific Place)
      |
      v
[ PHASE 2: Information Retrieval (The Loop) ] <-------+
      |                                               |
      +--> Tool: search_location("Coffee")            |
      |     |-> Returns: [Starbucks A, Peet's B...]   |
      |                                               |
      +--> DECISION: Ambiguity Detected?              |
      |     |-> YES: Ask User "Which coffee shop?" ---+
      |     |-> NO:  Select Candidate                 |
      |                                               |
      +--> Tool: search_location("Office")            |
      |     |-> Returns: [User Saved Loc: (lat, lon)] |
      |
      v
[ PHASE 3: Spatial Reasoning & Execution ]
      |-- Construct Sequence: Current Loc -> Starbucks A -> Office
      |-- Tool: get_route(start, waypoints=[Starbucks A], end=Office)
      |     |-> Returns: { "geometry": "...", "duration": 1200s }
      |
      v
[ PHASE 4: Response Synthesis ]
      |-- Translate seconds to minutes
      |-- Summarize traffic/turns
      |-- OUTPUT: "Route set. First stop Starbucks A, then Office. Total 20 mins."
```

### 2.2 核心设计原则 (Rule of Thumb)

1.  **LLM 是调度员，不是计算器**：永远不要让 LLM 计算“两个坐标的中间点”或“预计到达时间”。这些必须是工具返回的**硬数据**。
2.  **上下文优先**：所有的搜索工具必须支持 `viewbox`（视野框）或 `location_bias`（位置偏置）。当用户在“上海”问“去人民广场”时，绝不能搜出“重庆人民广场”。
3.  **失败是常态**：工具必须返回 LLM 能理解的错误信息（如 `{"error": "LocationNotFound", "suggestion": "Try removing distinct name"}`），而不是抛出 500 异常导致对话中断。

---

## 3. Tool Schema 设计：强类型的艺术

LLM 通过阅读 JSON Schema 的 `description` 和 `type` 来理解工具。Schema 就是 Prompt 的一部分。

### 3.1 搜索工具：不仅仅是 query

最简单的搜索是 `search(q)`，但这在 Agent 场景下远远不够。

#### 推荐 Schema 设计思路：

*   **Tool Name**: `search_poi` (比 `search` 更具体，暗示寻找兴趣点)
*   **Parameters**:
    *   `query` (string, required): 搜索关键词。*Tip: 提示 LLM 在此去除“去”、“导航到”等动词。*
    *   `category` (string, enum): 限定类别，如 `parking`, `fuel`, `hotel`。*这能利用 OSM 的 `amenity` 标签进行精准过滤。*
    *   `near_location` (string, optional): “在...附近”。
    *   `limit` (integer): 限制返回数量。*默认值设为 5，防止 Token 爆炸。*

**关键技巧**：在 `description` 中明确告诉 LLM：“如果用户提供的地点有多个同名结果，此工具返回一个列表。请务必检查返回列表的 `display_name` 字段以区分。”

### 3.2 路由工具：参数防御

路由工具是极其敏感的。错误的参数会导致 Agent 产生幻觉。

#### 推荐 Schema 设计思路：

*   **Tool Name**: `plan_route`
*   **Parameters**:
    *   `origin` (string, required): 格式 `lat,lon`。
    *   `destination` (string, required): 格式 `lat,lon`。
    *   `mode` (enum): `driving`, `walking`, `cycling`。**严禁**使用 string 让 LLM 随意发挥。
    *   `waypoints` (array of strings): 途经点列表。
    *   `preference` (enum): `fastest`, `shortest`, `avoid_tolls`。

**防御性 Prompting (在 Description 中)**：
> "Critical: The `origin` and `destination` MUST be precise coordinate strings format 'latitude,longitude' obtained from the `search_poi` tool. DO NOT invent or estimate coordinates."

### 3.3 辅助工具：增强 Agent 的空间感

除了搜和跑，Agent 还需要理解世界：

*   **`get_address_details(lat, lon)`**: 逆地理编码。用于当用户发送定位时，Agent 理解他在哪里。
*   **`check_location_in_bounds(lat, lon, city_name)`**: 逻辑检查工具。确认计算出的坐标是否真的在目标城市内（防漂移）。

---

## 4. 歧义消解（Disambiguation）：多轮对话的灵魂

这是 LLM 区别于传统 API 的最大价值。传统 API 只能返回 Top 1，而 Agent 可以说：“我不确定。”

### 4.1 歧义检测算法

当 Agent 收到 `search_poi` 的结果时，不应盲目选择第一个。我们可以教 Agent（或在 MCP Server 层编写逻辑）执行以下判断：

1.  **数量检查**：结果集 `length > 1`。
2.  **分布检查**：候选点之间的地理距离是否显著？
    *   如果两个候选点相距 100 米（如主门和侧门），通常可以直接选 Top 1。
    *   如果相距 1000 公里（如朝阳区@北京 vs 朝阳区@长春），必须触发澄清。
3.  **类别检查**：如果用户搜“万达”，结果里有“万达广场（购物中心）”和“万达影城（电影院）”，这属于语义歧义。

### 4.2 澄清对话生成策略

Agent 不应该把 JSON 原始数据扔给用户。

*   **Bad**: "我找到了：1. 北京南站 (Node 12345), 2. 北京南站 (Way 67890)。" (OSM 内部数据泄露)
*   **Good**: "您是指丰台区的**北京南站**（火车站），还是附近的**北京南站地铁站**？"

**实现技巧**：在 Tool 返回的 Prompt 中注入指令——"Found multiple distinct locations. Summarize their differences (e.g., district, category) and ask the user to clarify."

---

## 5. 可靠执行：Geocode-Route 链路编排

从自然语言到路径规划，是一条脆弱的链路。我们需要在代码层面或 Prompt 层面加固。

### 5.1 "两步验证"模式 (The Two-Step Verification)

最常见的错误是 LLM 试图一次性完成任务，编造坐标。

**强制链式思维（Chain of Thought）流程**：
1.  **Step 1: Locate** -> 调用 `search_poi`。
2.  **Step 2: Inspect** -> 观返回结果的 `confidence` 或 `type`。
3.  **Step 3: Extract** -> 提取 `lat,lon`。
4.  **Step 4: Route** -> 将提取的 `lat,lon` 填入 `plan_route`。

**如何在代码中强制执行？**
在 MCP Server 或 Agent 框架中，可以将 `plan_route` 工具的输入参数校验逻辑设为：检查输入的坐标是否曾在当前的 `conversation_history` 的 Tool Output 中出现过。如果是一个从未出现过的坐标，极其可能是幻觉，予以拦截并警告 Agent。

### 5.2 错误回退 (Fallback) 机制

现实世界中 API 调用经常失败。Agent 需要具备**弹性**。

| 错误场景 | 原始错误 (API) | Agent 的应对策略 (System Prompt 预设) |
| :--- | :--- | :--- |
| **搜不到地点** | `404 Not Found` 或 `[]` | **策略**：去特殊字符、去修饰词（如“美丽的”）、扩大搜索范围。最后询问用户是否有别名。 |
| **无路径** | `OSRM NoRoute` | **策略**：通常是因为起终点在封闭区域（如校内）。尝试开启 `snapping`（吸附）模式，或建议“步行”模式。 |
| **超时** | `504 Gateway Timeout` | **策略**：安抚用户“地图服务响应稍慢”，自动重试一次。 |
| **距离过远** | `Route > 5000km` | **策略**：提醒用户“距离过远，驾车不可行，建议查询航班或火车”。 |

---

## 6. 输出表达：从结构化数据到人性化建议

OSRM 的返回结果包含了成百上千个坐标点和 `turn-by-turn` 指令。如果让 LLM 直接总结，它很容易迷失在数据海洋中（Context Window Overflow）。

### 6.1 信息摘要金字塔 (Information Pyramids)

我们需要在 Tool 返回给 LLM 之前，在代码层做一次**预处理（Pre-processing）**。

1.  **Level 1: 概览 (Summary)**
    *   总距离、总时间、主要道路（提取 `ref` 或出现频率最高的路名）。
    *   *用途*：回答“去那里远吗？”
2.  **Level 2: 关键节点 (Key Maneuvers)**
    *   只保留 `type` 为 `turn`（转弯）且角度 > 45度的指令或者是 `off ramp`（下匝道）。过滤掉所有 `continue`（直行）。
    *   *用途*：回答“怎么走？”
3.  **Level 3: 完整指引**
    *   完整的步骤列表。
    *   *用途*：仅当用户明确要求“详细列表”时提供，且建议分页。

### 6.2 语气与风格控制

在 System Prompt 中定义 Agent 的“人设”：
*   **副驾驶风格**：“前方 500 米左转，注意可能会堵车。”
*   **本地通风格**：“去那条路建议别走主路，走辅路可以直接进停车场。”（这需要结合 POI 数据的高级推理）。

---

## 7. 本章小结

*   **设计哲学**：Agent 导航不仅是 API 的串联，更是对“模糊意图”到“精确执行”这一过程的管理。
*   **Schema 决定智商**：精准的类型定义（Enum, LatLon String）能减少 80% 的 LLM 幻觉。
*   **上下文是王道**：没有 `viewbox` 的搜索就是大海捞针。务必维护用户的当前位置上下文。
*   **预处理至关重要**：要把 OSRM 的原始 JSON 扔给 LLM，必须经过摘要和清洗，节省 Token 并聚焦注意力。

---

## 8. 练习题

### 基础题
1.  **Schema 纠错**：给定一个设计糟糕的 JSON Schema（如使用 `object` 类型接收坐标，没有描述），请指出其潜在的 3 个风险，并重写为规范版本。
    *   *Hint*: 思考 LLM 会如何误解 "location": "object"。
2.  **对话流设计**：设计一段对话脚本，模拟 Agent 处理“去家乐福”这一指令时，发现该城市有 5 家家乐福，并最终引导用户选择最近一家的过程。
3.  **摘要练习**：给定一段 OSRM 的 JSON 响应（包含 50 个 step），编写一个伪代码函数，将其压缩为一段不超过 100 个 token 的自然语言描述。

### 挑战题
1.  **多意图混合处理**：用户说：“先去接我朋友（在国贸），然后我们一起去环球影城，只要不堵车的路。”
    *   设计 Agent 的思维链。
    *   这涉及 `search` (x2) -> `route` (waypoints) -> `preference` (traffic) 的复杂组合。如何确保顺序正确？
2.  **反向地理推理**：用户发来一张包含地标描述的文本：“我在一个大烟囱旁边，对面是红色的商场。”
    *   设计一个 Tool 组合策略，利用 Nominatim 的 POI 数据和 PostGIS 的空间查询（`ST_DWithin`）来推测用户的可能坐标。
3.  **对抗性测试**：设计 5 个“恶意”Prompt，试图诱导你的导航 Agent 输出错误的路线或执行越权操作（如搜索保密地点），并给出防御方案。

---

## 9. 常见陷阱与错误 (Gotchas)

### 1. Token 耗尽 (Context Window Exhaustion)
*   **现象**：路线太长，Step 太多，Tool 返回后的下一轮对话 LLM 报错或“失忆”。
*   **陷阱**：OSRM 默认返回完整的 `geometry`（成千上万个坐标点）。
*   **对策**：在 MCP Server 端，调用 OSRM 时加上 `overview=simplified` 或 `geometries=polyline`。**绝对不要**把 geometry 字段传给 LLM，它读不懂且没必要读。只传文本指令（Instructions）。

### 2. 坐标系幻觉 (Coordinate Swap Hallucination)
*   **现象**：LLM 完美地提取了坐标，但在传入 `plan_route` 时把经纬度反了（Lat, Lon 变成了 Lon, Lat），导致路线定位到了南极或海洋。
*   **陷阱**：训练数据中 GeoJSON (Lon, Lat) 和 Google Maps (Lat, Lon) 格式共存。
*   **对策**：
    1.  Schema 中参数名显式命名为 `latitude` 和 `longitude` 而不是 `coords`。
    2.  代码层增加“合理性校验”：如果 latitude > 90，自动报错提示“坐标反了”。

### 3. "我以为你在那里" (Assumption of Location)
*   **现象**：用户问“去机场多久？”，Agent 报错，因为它不知道“起点”是哪里。
*   **陷阱**：忽略了导航必须有起点。
*   **对策**：Agent 必须维护一个 `user_current_location` 状态。如果没有，第一步必须反问：“请问您现在从哪里出发？”或者请求获取定位权限。

### 4. 递归用死循环 (Infinite Loop)
*   **现象**：Search 返回空 -> Agent 尝试 Search -> 返回空 -> Agent 尝试 Search...
*   **陷阱**：缺乏终止条件。
*   **对策**：在 System Loop 中设置 `max_tool_calls`（如 5 次）。如果连续调用同一工具且参数相似，强制中断并向用户求助。
