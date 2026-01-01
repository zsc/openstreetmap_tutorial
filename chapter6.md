# 第6章｜中文地址、POI 搜索与体验优化（让用户搜得到、搜得准）

## 1. 开篇段落

在搭建好 Nominatim 基础服务后，你很快会撞上一堵“现实之墙”：直接使用开源配置处理中文地址，效果往往令人沮丧。用户输入的“朝阳大悦城”可能搜不到，必须输入“朝阳北路101号”才行；或者输入“人民路”，却返回了全国几百条同名道路。

这是因为 Nominatim 的核心算法（基于 Token 的词频反向索引）最初是为西方地址体系设计的，依赖空格分词和明确的层级。而中文地址是**连续文本**，且严重依赖**隐含的层级关系**。

本章将带你构建一个**“生产级”的搜索中间层**。我们将不局限于 Nominatim 本身，而是从**输入预处理（Normalization）**、**查询构建策略（Query Strategy）**、**结果重排序（Re-ranking）** 以及 **LLM 交互设计** 四个维度，打造一套能听懂“中文人话”的导航搜索系统。

---

## 2. 核心概念与设计论述

### 2.1 深入解剖：OSM 数据结构 vs. 中文地址习惯

要优化搜索，首先必须理解数据（OSM）和输入（用户）之间的错位。

#### 中文地址的“隐式层级”
用户习惯输入：`省 - 市 - 区 - 街道 - 道路 - 门牌 - POI`。但在实际输入中，**前三级常被省略，后三级常被混用**。

#### OSM 的存储现状
OSM 不是一张巨大的 Excel 表，而是一张网。
*   **行政区（Administrative Boundary）**：这是多边形（Relation）。Nominatim 通过空间包含关系（ST_Contains）来计算一个点属于哪个区，而不是靠字段存储。
*   **道路（Highway）**：是线（Way）。道路名存储在 `name` 标签。
*   **门牌（Housenumber）**：大部分是离散的点（Node），少部分附着在建筑轮廓（Way）上。
*   **POI**：独立的点或多边形，与道路没有物理连接，只有空间相邻关系。

**错位图示**：

```text
[用户输入] "北京朝阳区工体北路通盈中心"
      |
      v
[语义切分]
1. "北京" (City) ---------> 匹配 OSM Relation (admin_level=4)
2. "朝阳区" (District) ---> 匹配 OSM Relation (admin_level=6)
3. "工体北路" (Road) -----> 匹配 OSM Way (highway=primary)
4. "通盈中心" (POI) ------> 匹配 OSM Node/Way (name=通盈中心)
      |
      v
[Nominatim 的困境]
如果没有空格，Nominatim 可能会把 "工体北路通盈中心" 当作一个长单词去索引。
如果 "通盈中心" 在 OSM 里没有记录 `addr:street=工体北路`，单纯搜文本可能因权重过低被过滤。
```

### 2.2 关键策略一：输入归一化管道 (The Normalization Pipeline)

在请求到达 Nominatim 之前，必须经过一层“清洗。这是低成本提升命中率的关键。不要指望 Nominatim 的分词器能完美处理中文杂乱输入。

**管道设计建议**：

1.  **字符集清洗 (NFKC)**：
    *   统一全角/半角：`ＡＢＣ` -> `ABC`，`１２３` -> `123`。
    *   统一标点：`（` -> `(`，`＃` -> `#`。
    *   去除无意义字符：`空格`、`\t`、`\n`（注意：英文地址需要保留空格，中文地址通常建议去除内部空格以辅助特定分词，或者统一替换为逗号）。

2.  **中文数字转写 (Digit Normalization)**：
    *   虽然 Nominatim 内部有转换规则，但很难覆盖所有口语习惯。
    *   规则：`一号` -> `1号`，`八十八号` -> `88号`。
    *   *Rule of Thumb*：只转换“号/弄/巷”前面的数字。不要转换路名中的数字（如“一汽大街”不能变成“1汽大街”）。

3.  **同义词/后缀标准化 (Synonym Expansion)**：
    *   这是解决“搜不到路”的核心。
    *   建立映射表：
        *   `Rd`, `Lu` -> `路`
        *   `Ave`, `Dadao` -> `大道`
        *   `St`, `Jie` -> `街`
    *   *Gotcha*：小心“道”字，既可是“大道”的后缀，也可能是行政区“北海道”或“街道”的一部分。通常只替换结尾处的特定词。

4.  **去干扰词**：
    *   用户常输入“我要去...”、“...附近”。检测并剥离这些非地址词汇。

### 2.3 关键策略二：结构化查询构建 (Structured Query Strategy)

Nominatim 提供了 `/search?q=...`（自由文本）和 `/search?street=...&city=...`（结构化）两种模式。

**极力推荐：混合查询策略（Hybrid Strategy）**

不要把所有内容都塞进 `q` 参数。Nominatim 对 `q` 的解析非常消耗资源且容易误判。

**算法流程**：

1.  **正则提取行政区**：
    *   使用简单的正则库提取输入开头的 `省/市/区/县`。
    *   例如输入：“北京市海淀区丹棱街5号”。
    *   提取：`city=北京市`, `county=海淀区`。
    *   剩余：`丹棱街5号`。

2.  **构建第一次请求（精确结构化）**：
    *   `street=丹棱街5号`
    *   `city=北京市`
    *   `county=海淀区`
    *   *优点*：利用数据库索引，速度极快，结果极准。

3.  **构建第二次请求（降级/回退）**：
    *   如果第一次返回空（常见原因：门牌号不匹配，或提取错误）。
    *   剥离门牌号，只查路：`street=丹棱街`, `city=...`
    *   *目的*：至少把用户导向正确的道路，而不是报错。

4.  **构建第三次请求（自由文本兜底）**：
    *   `q=北京市海淀区丹棱街5号`
    *   `viewbox=...` (如果已知用户大致位置)
    *   *目的*：利用全文索引尝试模糊匹配。

### 2.4 关键策略三：拼音与多语言处理

OSM 数据中包含大量 `name:en`, `name:pinyin` 标签，但 Nominatim 默认配置通常**不索引**所有拼音，或者索引方式不支持“首字母搜索”。

**三种实现路径**：

| 方案 | 复杂度 | 原理 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **A. 前端转换 (推荐)** | 低 | 在应用层将“Beij”转为汉字“北京”或“北井”，构建多个汉字 Query 并行请求。 | MVP，用户量不大，主要服务中文母语者。 |
| **B. 辅助索引 (推荐)** | 中 | 搭建一个 Elasticsearch，只存 POI 名称和拼音。用户输入拼音时，ES 返回 OSM ID，再通过 Nominatim `/lookup` 查详情。 | 生产环境，要求高性能 Autocomplete。 |
| **C. 深度魔改** | 高 | 修改 Nominatim 导入脚本，引入 `pinyin-tokenizer` 或 `pg_jieba`，重建数据库。 | 极其依赖 OSM 纯栈，不想引入 ES 的团队。 |

> **Rule of Thumb**：不要试图在 PostGIS/Nominatim 里做复杂的模糊拼音匹配。它会让数据库 CPU 爆炸。把这个逻辑以前置（前端/中间层）或旁路（ES）的方式解决。

### 2.5 关键策略四：歧义消解与 Agent 交互 (Disambiguation)

在 Navigation MCP 中，搜索不是终点，而是对话的点。

**场景**：用户对 Agent 说“导航去万达”。
Nominatim 返回列表：
1.  通州万达广场 (Distance: 15km, Importance: 0.7)
2.  CBD万达广场 (Distance: 5km, Importance: 0.8)
3.  丰台万达广场 (Distance: 20km, Importance: 0.6)

**Agent 工具链设计**：

不应直接返回 Top 1。MCP Tool 的返回 Schema 应该包含足够的信息供 LLM 判断是否需要“追问”。

```json
// Tool Response 示例
{
  "status": "ambiguous",
  "message": "Found 3 high-confidence results.",
  "candidates": [
    {
      "name": "CBD万达广场",
      "address": "建国路93号, 朝阳区, 北京市", // 结构化地址很重要
      "distance_km": 5.0,
      "type": "mall",
      "osm_id": 12345
    },
    {
      "name": "通州万达广场",
      "address": "新华西街58号, 通州区, 北京市",
      "distance_km": 15.0,
      "type": "mall",
      "osm_id": 67890
    }
  ]
}
```

**LLM 的 Prompt 策略**：
*   如果 `status == ambiguous` 且 `distance` 差异巨大 -> **自动选择最近的**（假设用户意图是就近）。
*   如果 `status == ambiguous` 且 `distance` 相近 -> **发起澄清**：“找到了两个万达广场，一个在建国路，一个在新华西街，您要去哪个？”

---

## 3. 本章小结

*   **OSM 的数据现状**：中文地址在 OSM 中是碎片化的，不要指望完美匹配。
*   **归一化是第一生产力**：建立包含字符清洗、数字转写、正则提取的预处理管道，能解决 80% 的“搜不到”问题。
*   **分层搜索策略**：结构化搜索（精准） > 剥离门牌搜索（保底） > 自由文本搜索（模糊）。
*   **Agent 的智慧在于“懂分寸”**：通过设计包含 `address details` 和 `distance` 的 API 响应，让 LLM 具备像人一样区分同名地点的能力。
*   **拼音处理**：尽量在 Nominatim 外部解决（前端转换或 ES 旁路），避免拖累数据库性能。

---

## 4. 练习题

### 基础题
1.  **API 对比实验**：
    *   找一个带门牌号的地址（如“北京市朝阳区工体北路4号”）。
    *   分别用 `q=...` 和 `street=...&city=...` 发起请求。
    *   观察响应时间（`time` 字段）和执行计划（如果有权限看日志）的区别。
2.  **归一化脚本**：编写一个 Python 函数 `normalize_address(raw_text)`，实现：
    *   全角转半角。
    *   移除所有空格。
    *   提取并移除末尾的“市/区”关键词作为 metadata 返回。
3.  **结果解析**：调用 Nominatim `/search` 接口，解析返回的 JSON。提取 `address` 字段，编写逻辑判断：如果结果中包含 `postcode`，则打印“精确到邮编区域”；如果只有 `state` 和 `country`，则打印“仅匹配到行政区”。

<details>
<summary>点击查看提示 (Hints)</summary>

*   **题1提示**：结构化搜索通常会直接命中索引，响应时间应在 50ms 以内；自由文本搜索涉及分词和排列组合，可能需要 200ms+。
*   **题2提示**：使用 `unicodedata.normalize('NFKC', text)` 处理全角。使用 `re.search` 提取行政区。
*   **题3提示**：检查 `address.postcode` 字段是否存在。注意 Nominatim 的 `address` 字段内容是动态的，取决于数据中有什么 tag。
</details>

### 挑战题
4.  **构建“门牌回退”递归逻辑**：
    *   设计一个伪代码函数 `smart_geocode(address)`。
    *   逻辑：先搜全量地址 -> 若失败，正则去掉门牌号 -> 再搜道路名 -> 若成功，返回道路中心点，并标记 `accuracy_level: "street"`。
    *   思考：如何告知用户“我们没找到门牌，但把你导到了路上”？

5.  **LLM 澄清生成器**：
    *   给定 Nominatim 返回的 3 个候选地点（包含 `display_name` 和 `class/type`）。
    *   编写一个 Prompt 模板，输入这些 JSON 数据，要求 LLM 输出一句自然语言，通过区分这三个地点的“行政区”或“周边地标”来询问用户。
    *   示例输入：`[{"name":"KFC", "district":"Haidian"}, {"name":"KFC", "district":"Chaoyang"}]`
    *   期望输出：“您是指海淀区的肯德基，还是朝阳区的？”

<details>
<summary>点击查看提示 (Hints)</summary>

*   **题4提示**：在 API 响应体中增加一个自定义字段 `meta`，包含 `match_type: "exact" | "street_fallback" | "city_fallback"`。前端或 Agent 根据这个字段决定话术。
*   **题5提示**：Prompt 需要包含指令：“Identify the distinguishing feature (like road name or district) in the provided candidates and formulate a clarification question.”
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 陷阱 1：OSM ID 的易变性
**现象**：你缓存了某个地点的 `osm_id`（如 `node/12345`），一周后查询发现 ID 变了或失效了。
**原因**：在 OSM 中，如果用户删除了一个点并新建了一个更完善的点，ID 会改变。或者编辑者将 Node 升级为 Way（例如给建筑物画了轮廓），ID 也会变。
**对策**：不要将 OSM ID 作为永久的主键存储在业务数据库中。使用 `place_id`（Nominatim 内部 ID）也不完全保险。建议**存储经纬度和名称**，必要时重新 Geocode，或者定期通过 `/lookup` 刷新。

### 陷阱 2：高估了 `boundary` 的覆盖率
**现象**：期望通过反向地理编码（Reverse Geocoding）获取用户所在的“街道/乡镇（Town/Village）”。
**现实**：在中国，OSM 的 `admin_level=4`（省）和 `6`（地级市）覆盖较好，但 `8`（乡镇/街道）覆盖率极低。
**对策**：业务逻辑不要强依赖“街道办”这一层级。如果必须要有，通常需要自建多边形图层进行空间关联（Point-in-Polygon），而不是依赖 Nominatim 的原生数据。

### 陷阱 3：由于 `dedupe` 导致的 POI 消失
**现象**：搜“肯德基”，明明数据里有点（Node）也有房（Way），却只返回了一个。
**原因**：Nominatim 默认有去重逻辑（Deduplication），如果两个同名对象距离很近，会合并显示
**对策**：通常这是好特性。但在调试数据时，如果你需要看到所有原始数据，可以使用 `&dedupe=0` 参数。

### 陷阱 4：坐标系漂移（火星坐标系）
**现象**：搜出来的坐标在 OSM 底图上是对的，但在高德/百度/腾讯地图上显示偏了。
**原因**：OSM 是 WGS84 坐标。国内商业地图使用 GCJ-02 或 BD-09。
**对策**：作为 Navigation API，**输入输出必须严格定义为 WGS84**。如果客户端使用国内地图 SDK，必须在**最前端**（Client App）做坐标转换，严禁在后端 API 层混用坐标系，否则后续的路径规划（OSRM）会全部错乱。
