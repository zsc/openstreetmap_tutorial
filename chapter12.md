# 第12章｜Navigation MCP：MCP Server 设计、工具编排、观测与安全

## 1. 开篇段落

在前面的 11 章中，我们从零搭建了数据管道、部署了 Nominatim 和 OSRM 引擎，并通过 Facade API 统一了接口。现在，我们来到了系统的“最后一公里”：如何让大语言模型（LLM）“长出双腿”，能够自主地感知物理世界并规划行动。

**Model Context Protocol (MCP)** 是当前连接 LLM 与外部工具的事实标准。与传统 REST API 面向开发者不同，MCP Server 是直接面向 LLM 的。这意味着你的设计核心必须从“结构化数据的精确传输”转向**“语义信息的压缩与引导”**。

本章将详细拆解如何设计一个生产级的 **Navigation MCP Server**。你将学习如何将导航能力映射为 MCP 的 `Tools` 和 `Resources`，如何处理 LLM 上下文窗口的限制（对庞大的地理数据进行“近似”与“压缩”），以及如何设计安全边界，防止 Agent 滥用你的地图服务。

---

## 2. 核心概念：从 API 到 MCP 的思维转换

### 2.1 上下文经济学 (Context Economics)

在设计 MCP 时，最关键的约束是 LLM 的**上下文窗口（Context Window）**。

-   **传统 App**：前端请求路由 API，获得一个 5MB 的 GeoJSON，包含 10,000 个坐标点，然后在地图上绘制出平滑曲线。
-   **LLM Agent**：如果把这 5MB 数据喂给 LLM，不仅会瞬间耗尽 Token 预算，还会因为无关信息过多导致模型“注意力发散”，甚至产生幻觉。

> **Rule of Thumb (设计法则)**：**Navigation MCP 的核心是“有损压缩”。**
> 不要给 LLM 看“地图（Geometry）”，要给 LLM 看“路书（Itinerary）”。只有当用户明确要求下载文件时，才通过 Resources 提供原始数据。

### 2.2 架构分层：MCP Server 的位置

Navigation MCP Server 充当了“语义翻译官”和“流量守门人”的角色。

```ascii
[ User / Client ]
      |
      v
[ LLM (Reasoning Core) ] <===(JSON Schema)=== [ Tool Definitions ]
      |
      | (Tool Call: "Plan a route from A to B")
      v
[ MCP Server (Navigation) ]  <-- (The focus of this chapter)
      | 1. Parameter Validation
      | 2. Semantic Compression
      | 3. Error Translation
      v
[ Facade API (Chapter 9) ]
      |
   +--+--+
   |     |
[Nominatim] [OSRM]
```

---

## 3. Tool 设计：核心工具集 (Core Tools)

我们需要将导航能力映射为一组互相配合的原子工具。这里的关键是**单一职责原则**，防止一个工具过于复杂。

### 3.1 `search_location` (地理编码)

这是导航的入口。LLM 必须先通过此工具获取明确的坐标，才能进行后续操作。

*   **输入 Schema**：
    *   `query` (string, required): 自然语言地名。
    *   `limit` (integer): 限制返回数量，建议默认 3-5，防止撑爆上下文。
    *   `viewbox` (string): 偏好区域（如当前地图视野），格式 `x1,y1,x2,y2`。
*   **输出设计 (关键)**：
    *   不要返回原始的所有 OSM Tags。
    *   **仅返回**：`place_id`、`display_name`（清洗后的）、`lat`、`lon`、`type`（如 hospital, restaurant）。
    *   **歧义处理**：如果搜到多个结果，不要自动选第一个。返回列表，并在 prompt 中暗示 Agent 需要向用户澄清（例如：“找到多个‘朝阳区’，请问是指北京的还是长春的？”）。

### 3.2 `get_route_summary` (路由规划)

注意工具名称是 `summary` 而不是 `geometry`。这是为了强调返回的是“指令摘要”。

*   **输入 Schema**：
    *   `start_coord` (string, required): `lon,lat` 格式。
    *   `end_coord` (string, required): `lon,lat` 格式。
    *   `mode` (enum): `driving`, `walking`, `cycling`。
    *   `alternatives` (boolean): 是否返回备选路线。
*   **输出设计 (关键)**：
    *   **概览**：总距离（km）、总时间（min）、ETA。
    *   **关键指令 (Maneuvers)**：提取 OSRM response 中的 `steps`，将其转化为文本列表。
        *   *Good*: "1. 向北出发 200米; 2. 右转进入主路; 3. 直行 5公里"。
        *   *Bad*: 包含 lat/lon 数组的原始 JSON。
    *   **路况/警告**：是否有收费、轮渡、未铺装路面。

### 3.3 `reverse_geocode_context` (位置理解)

用于 Agent 理解“我现在在哪”或“这个坐标周围是什么”。

*   **设计意图**：不仅仅返回地址，还要返回**上下文**。
*   **输出增强**：除了返回标准地址（Admin Hierarchy），建议调用 Nominatim 的 `extratags`，如果附近有重要 POI（如“距离故宫东门 50 米”），应一并返回。这有助于 LLM 生成更具“人类感”的回复。

---

## 4. 展能力：编排与高级工具

### 4.1 扩展工具集 (Extended Tools)

当基础导航满足后，Agent 需要更高级的空间推理能力。

| 工具名称 | 对应后端逻辑 | 应用场景 | 设计注意点 |
| :--- | :--- | :--- | :--- |
| `explore_nearby` | POI Search + Circle Filter | "帮我找附近的加油站" | 必须强制分页。一次只给 LLM 看 5 个，并提供 `offset` 参数供翻页。 |
| `check_isochrone` | Isochrone / Service Area | "我 10 分钟能走到哪" | **绝对不要**返回 Polygon 坐标串。应返回“文字描述边界”（如：最北到A街，最南到B路）。 |
| `match_trace` | OSRM Match | "分析这条轨迹对应的道路" | 仅供高级分析 Agent 使用，普通对话 Agent 慎用（数据量大）。 |

### 4.2 智能体编排模式 (Orchestration Patterns)

MCP Server 不仅仅是被动响应，它可以通过设计引导 Agent 的**思维链 (Chain of Thought)**。

**推荐模式：G-C-R (Geocode - Confirm - Route)**

1.  **用户**：去万达广场。”
2.  **Agent** 调用 `search_location("万达广场")`。
3.  **MCP** 返回列表：`[A: 北京CBD万达, B: 北京通州万达, ...]`。
4.  **Agent (思考)**：结果不唯一，且没有用户偏好。
5.  **Agent (回复)**：“北京有多个万达广场，您想去 CBD 的还是通州的？”
6.  **用户**：“CBD 的。”
7.  **Agent** 调用 `get_route(start, A.coord)`。

> **Gotcha**：如果你在 MCP 中不仅做 Search 还在内部自动 Route（把两步合并），会导致 Agent 失去“澄清歧义”的机会，极易导致导航错误。**工具的原子性（Atomicity）至关重要。**

---

## 5. 错误处理模型：把 Error 变成 Context

在 LLM 交互中，HTTP 500 是没有意义的。MCP 必须捕获后端错误，并将其转化为**自然语言的行动建议**。

### 5.1 错误转化表

| 后端情况 | 传统 API 响应 | MCP 响应 (作为 Tool Output) | 目的 |
| :--- | :--- | :--- | :--- |
| **无结果** | 404 Not Found | `{"error": "找到地点，建议：1.检查拼写 2.尝试更广泛的区域 3.使用 search_nearby 替代"}` | 引导 Agent 换个策略尝试，而不是直接死机。 |
| **不可路由** | 400 No Route | `{"error": "无法规划路线。原因可能是：起点/终点在孤岛（如湖心），或两点间无连通道路（跨海）。尝试切换为 walking 模式？"}` | 提供具体的物理世界解释。 |
| **超时** | 504 Gateway Timeout | `{"error": "计算过于复杂。请尝试将长途路线切分为两段进行规划。"}` | 给出降级方案。 |

---

## 6. Resources 与 Prompts 的高级用法

MCP 不止有 Tools，善用 `Resources` 和 `Prompts` 能极大提升体验。

### 6.1 Resources：处理静态重资产

当用户说“把路线文件发给我”时，不要试图让 LLM 生成 XML 文本。

*   **定义 Resource**：`nav://routes/{id}/gpx`
*   **行为**：MCP Server 在后台将 OSRM 结果转换为 GPX 文件，暂存到 S3/本地，并返回一个 URI 或文件句柄。
*   **LLM 交互**：Agent 读取该 Resource，直接作为附件展示给用户，完全绕过上下文窗口。

### 6.2 Prompts：预置“老司机”人格

在 MCP Server 中预埋 System Prompt 片段。

*   **Prompt Name**: `navigation_expert`
*   **内容范例**：
    > "你是一个专业的导航助手。在规划路线前，你必须始终校验起点和终点的距离。如果距离超过 2000km，主动建议用户考虑飞机或火车。如果用户输入模糊（如'回家'），必须先查询用户的预设地址或要求澄清。在描述路线时，不要念经纬度，要使用‘左转’、‘沿XX路行驶’等人类语言。"

---

## 7. 生产化：安全、观测与限流

### 7.1 安全沙箱 (Security Sandbox)

MCP Server 允许 LLM 访问你的内网 API，这带来了 SSRF（服务端请求伪造）风险。

*   **白名单机制**：MCP Server 只能访问预定义的 Facade API 地址，严禁访问 `localhost` 或其他内网 IP。
*   **只读强制**：除非你有极其格的鉴权，否则 Navigation MCP **严禁暴露** OpenStreetMap 的编辑/写入接口。你不想让 Agent 因为幻觉把某条高速公路删了。
*   **输入清洗**：虽然 Facade API 有防御，但在 MCP 层仍需校验 `query` 长度（如限制 100 字符），防止恶意的超长 Prompt 攻击后端数据库。

### 7.2 可观测性 (Observability)

传统的 API 监控关注 QPS 和 Latency，MCP 监控关注**交互质量**。

*   **关键指标**：
    *   **Token Efficiency**：`output_tokens / raw_data_size`。如果比例过高，说明压缩做得不够。
    *   **Tool Sequence**：监控 Agent 的调用链。正常的链条是 `Search -> Route`。如果你发现大量的 `Search -> Search -> Search`，说明你的搜索排序有问题，Agent 找不到想要的结果。
    *   **Truncation Rate**：有多少次返回因为超长被 MCP 强制截断了？这意味你需要优化输出结构。

---

## 8. 本章小结

*   **角色转变**：MCP Server 是 LLM 的感官延，不是单纯的数据管道。
*   **压缩是核心**：将几何数据（Geometry）转化为语义指令（Instruction）是 MCP 开发者的首要任务。
*   **原子化工具**：保持 `Search` 和 `Route` 分离，把“决策权”和“歧义消除”留给 Agent。
*   **错误即引导**：良好的错误信息能让 Agent 在失败中自我修正，实现鲁棒的导航体验。

---

## 9. 练习题

### 基础题
1.  **Schema 编写**：为 `search_location` 编写一个完整的 MCP Tool Definition (JSON Schema)，要求包含 `query`、`city_filter`、`limit` 字段，并对 `limit` 设定最大值 10。
2.  **数据压缩**：给定一段 OSRM 的 `route` JSON 响应（包含 geometry string 和 steps），请手写一个伪代码函数，将其转换为只包含“总时间”、“总距离”和“路名序列”的纯文本摘要。
3.  **歧义处理**：设计一段 Prompt，当 Agent 调用 `search_location` 返回 5 个结果时，指导 Agent 如何向用户提问以消除歧义。

### 挑战题
4.  **等时圈文本化**：用户询问“我 15 分钟能走到哪些地方？”Valhalla/OSRM 返回了一个 GeoJSON Polygon。请设计一个算法思路，将这个多边形转化为一句人类可读的描述（例如：“向北最远到达 A 公园，向东最远到达 B 地铁站”）。*提示：利用 Reverse Geocode 查询多边形顶点的 POI。*
5.  **多模态 MCP**：假设你的 MCP Client 支持渲染图片。请设计一个 `get_static_map` 工具，利用 MCP 的 `Resources` 特性，返回一张带有路线高亮的静态地图图片（Base64 或 URL）。
6.  **安全审计**：你的 MCP Server 被部署在公网，Agent 可能会被 Prompt Injection 攻击。请设计一套机制，防止恶意用户诱导 Agent 通过你的 MCP Server 对 Nominatim 发起 DDoS 攻击（例如高频查询）。

---

## 10. 常见陷阱与错误 (Gotchas)

| 陷阱 | 现象 | 原因与解决方案 |
| :--- | :--- | :--- |
| **GeoJSON 倾倒 (The Blob)** | Agent 响应极慢变笨，或直接报错“Context Length Exceeded”。 | **原因**：直接将 OSRM 完整的 Polyline 返回给了 LLM。<br>**解法**：在 MCP 层剥离 `geometry` 字段，仅保留 `legs.steps` 中的 instruction。 |
| **坐标幻觉** | Agent 没调用搜索，直接编造了一个 `lat: 39.9, lon: 116.4` 去规划路线。 | **原因**：Prompt 中未强调必须先搜索。<br>**解法**：在 System Prompt 中加入强约束：“获取坐标必须通过 search 工具，严禁猜测。” |
| **指令死循环** | Agent 反复调用 `search` 查找同一个地名，陷入死循环。 | **原因**：Nominatim 返回空，且错误信息太简略。<br>**解法**：当结果为空时，MCP 应返回具体的 Query 修改建议（如“尝试去掉行政区前缀”），打破循环。 |
| **过度拟合** | Agent 总喜欢推荐步行，即使距离 50 公里。 | **原因**：Tool Definition 中 `mode` 的默认值设置不当或描述误导。<br>**解法**：在 Schema description 中明：“超过 2km 建议使用 driving，短途使用 walking”。 |

---

（全教程完）
