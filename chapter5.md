# 第5章｜Nominatim 部署与核心 API（search / reverse / lookup）

## 5.1 开篇：不仅仅是查表，它是地理语言翻译机

在构建导航系统时，开发者常犯的一个错误是把 Nominatim 当作一个普通的键值数据库（Key-Value Store）：输入地址字符串，返回坐标。

实际上，Nominatim 是一个**基于规则的地理语言搜索引擎**。它的工作是将 OSM 极其灵活（有时甚至混乱）的拓扑数据（Nodes/Ways/Relations），“拍平”成符合人类认知习惯的地址层级结构。

*   **OSM 的原始世界**：一条线段（Way）标记为 `highway=residential`，它连接着另一个关系（Relation）标记为 `type=boundary`。
*   **Nominatim 的转换世界**：这条线段是“长安街”，它位于“东城区”内，东城区位于“北京市”内。

本章目标：
1.  理解 Nominatim 的**数据扁平化（Flattening）**与**层级索引（Hierarchy Indexing）**机制。
2.  掌握核心 API 的设计意图，特别是参数组合对**LLM Agent 决策**的影响。
3.  学会区分“找不到结果”是数据问题、查询策略问题，还是索引配置问题。

---

## 5.2 核心架构与设计哲学

Nominatim 的底层虽然是 PostgreSQL + PostGIS，但它并没有使用常规的全文本检索（如 Elasticsearch），而是使用了一套专为地理名称优化的**倒排索引**和**地址树**。

### 5.2.1 数据的“拍平”与层级树 (Hierarchy Tree)

在数据导入阶段（`osm2pgsql`），Nominatim 会执行极其昂贵的预计算。它会遍历所有 OSM 对象，根据几何包含关系（ST_Contains）构建父子链。

**核心概念：Place Rank（地址层级）**
Nominatim 为每种 OSM 标签分配了一个 `Rank`（0-30）。这决定了该对象在地址字符串中的位置，以及它能包含哪些子对象。

**ASCII 图解：地址层级构建过程**

```text
[OSM Raw Data]                   [Nominatim Hierarchy Table]
                                
Relation(Beijing)   --------->   Rank 4  [Country/State]
    | (contains)                               ^
    v                                          | (parent_of)
Relation(Haidian)   --------->   Rank 10 [District/County]
    | (contains)                               ^
    v                                          | (parent_of)
Way(Zhongguancun)   --------->   Rank 26 [Street/Road]
    | (close_to)                               ^
    v                                          | (parent_of)
Node(Cafe)          --------->   Rank 30 [POI/House Number]
```

> **Rule of Thumb (经验法则)**：
> 无论 OSM 原数据多么复杂，Nominatim 永远试图将其入上述层级树中。如果一个对象（如大型购物中心）在 OSM 中只画了轮廓但没有标记 `name`，它可能只会作为计算父级关系的“背景板”，而无法被搜到。

### 5.2.2 索引与分词策略 (Tokenization)

当用户搜索“St. James St”时，Nominatim 如何区分哪个是圣（Saint）哪个是街（Street）？

它依赖于**Word 表**和**Token 分析**：
1.  **归一化 (Normalization)**：将 `Street`, `Str.`, `St.` 映射到同一个 Token ID。
2.  **部分词 (Partial Terms)**：用于前缀匹配（输入一半时提示）。
3.  **频繁词 (Frequent Terms)**：高频出现的词（如 "Road"）在索引中会被特殊处理，避免全表扫描。

这意味着：**Nominatim 不支持模糊搜索（Fuzzy Search）**。它不支持拼写纠错（如把 "Goggle" 纠正为 "Google"），除非你在导入时专门配置了同义词库。这对于 LLM 尤其重要——**LLM 必须输出拼写正确的地名**，否则工具调用将直接返回空

---

## 5.3 部署考量：资源与瓶颈

在生产环境自建 Nominatim，你需要了解它的“脾气”。

### 5.3.1 I/O 密集型怪兽
Nominatim 对磁盘 I/O 的要求极高。
*   **导入阶段**：大量的随机读写。导入全星球数据（Planet）在机械硬盘上可能需要几个月，而在 NVMe SSD 上只需要几天。
*   **查询阶段**：一次 `/search` 可能触发数十次索引查找和表连接。

> **架构建议**：
> 如果你的 Navigation MCP 需要高并发，**不要**仅仅增加 CPU 核心。优先升级内存（让 PostgreSQL 缓存更多索引）和 NVMe SSD。如果必须在机械硬盘上运行，必须做**分层存储**（将索引放在 SSD，数据放在 HDD）。

### 5.3.2 动态更新的代价
Nominatim 支持每分钟从 OSM 获取 Diff 更新。
*   **代价**：更新过程会锁定部分索引，导致查询变慢（Jitter）。
*   **策略**：对于高可用系统，通常采用 **Blue/Green 部署** 或 **读写分离**。一台机器负追赶最新数据（Write），另一台负责稳定查询（Read），定期交换或同步 WAL 日志。

---

## 5.4 核心 API 深度解析：`/search`

这是最复杂的端点。为了让 Agent 能够准确使用，必须理解其参数逻辑。

### 5.4.1 自由文本 vs. 结构化查询

*   **自由文本 (`q=...`)**：
    Nominatim 必须猜测逗号在哪里分割，哪部分是城市，哪部分是路名。
    *   *风险*：输入 "Paris Texas" 可能被误解为 "Paris, France" 的某个叫 Texas 的店（如果存在）。
    *   *性能*：慢，因为需要尝试多种组合。

*   **结构化查询 (`street=`, `city=`, `county=`, `country=`, `postalcode=`)**：
    直接命中对应层级的索引。
    *   *优势*：速度快，几乎无歧义。
    *   *LLM 适配*：**这是 Navigation MCP 的最佳实践**。在 Tool Definition 中，强制要求 LLM 提取 `city` 和 `poi`，而不是把整句话扔进去。

### 5.4.2 空间偏置：Viewbox

当用户在移动端地图上索“星巴克”时，他绝不希望看到地球背面的结果。

*   **`viewbox=<x1,y1,x2,y2>`**：定义一个矩形框。
*   **`bounded=1`**：**硬限制**。严格只返回框内的结果。
*   **`bounded=0` (默认)**：**软限制**。优先返回框内的，但如果框内没有，会扩大范围找（防止由近及远）。

**ASCII 图解：Viewbox 的作用**

```text
+-----------------------+
| World Map             |
|       [NYC]           |
|                       |
|   +-------+           |
|   |Viewbox|           | <--- 你的手机屏幕范围
|   | [A]   |           |
|   +-------+           |
|                       |
|         [B]           |
+-----------------------+

搜索 "Pizza":
- bounded=1: 仅返回 [A]
- bounded=0: 返回 [A], [B], [NYC] (按重要性排序)
```

### 5.4.3 结果去重 (`dedupe`)
OSM 数据中，同一个地点可能被画成一个点（Node），又被画成一个轮廓（Way）。默认情况下，Nominatim 可能会返回两个极其相似的结果。
*   **策略**：开启 `dedupe=1`。它会根据名称和空间距离，合并重复项。对于给用户的列表，这能显著提升体验。

---

## 5.5 核心 API 深度解析：`/reverse`

逆地理编码不仅是“找坐标”，而是“找上下文”。

### 5.5.1 `zoom` 参数的真实含义

这是最容易误解的参数。它**不控制地图的缩放级别**，它控制**地址树的截断层级**。

*   `zoom=18` (Building/House)：试图找到门牌号或建筑物名称。
*   `zoom=16` (Street)：忽略门牌，只返回到街道级别。
*   `zoom=10` (City)：忽略街道，只返回城市名称。
*   `zoom=3` (Country)：只返回国家。

> **应用场景**：
> *   **显示当前位置**：用 `zoom=18`。
> *   **天气预报查询**：Agent 拿到用户坐标查询天气时，应该用 `zoom=10` 调用 `/reverse`，获取城市名，然后再去调天气 API。如果用 18，可能会拿到一个天气 API 不认识的街道名。

### 5.5.2 搜索半径与“最近”的义

`/reverse` 并不是简单的 KNN（K-Nearest Neighbors）。
它首先寻找极小半径内的对象。如果找不到，它会**不**扩大半径去寻找远处的 POI，而是寻找**包含**该点的行政区划。

**常见坑**：你在高速公路中间调用 `/reverse`，期望得到“G4高速”。但如果坐标稍微偏离路面，且周围荒凉，Nominatim 可能直接返回“某某县”，而完全忽略了旁边 50 米外的高速公路。
*   **解决**：这通常需要结合 OSRM 的 `/match`（地图匹配）先将坐标吸附到道路上，再进行 Reverse。

---

## 5.6 核心 API 深度解析：`/lookup`

*   **功能**：通过 `osm_ids=W1234,N5678` 批量获取详情。
*   **设计价值**：这是构建**极其高效的缓存层**的基础。
    *   流程：`/search` 返回结果 -> 用户点击 -> App 仅存储 `osm_type/id` -> 下次展示时调用 `/lookup` 获取最新名称和属性。
*   **稳定性警告**：虽然比 `place_id` 稳定，但在 OSM 中，如一个 Way 被拆分（例如道路中间加了隔离带），旧 ID 可能会失效。生产环境需要处理 `404` 并触发重新搜索机制。

---

## 5.7 关键返回字段的设计意图

当 Nominatim 返回 JSON 时，以下字段决定了你的业务逻辑：

1.  **`display_name`**：
    *   *说明*：这是 Nominatim 拼凑出的完整字符串。
    *   *陷阱*：格式不可控。不要试图用正则去解析它。
2.  **`address` 对象**：
    *   *说明*：这是结构化的部分（city, road, house_number...）。
    *   *用法*：**必须使用此字段**来构建 UI 或提取 Agent 需要的信息。
3.  **`importance`**：
    *   *说明*：基于 Wikipedia 链接数、PageRank 等计算的 0-1 分数。
    *   *用法*：当存在多个重名地点时（例如 10 个“人民公园”），**必须**按此字段排序展示给用户。
4.  **`boundingbox`**：
    *   *说明*：`[minLat, maxLat, minLon, maxLon]`。
    *   *用法*：**Camera Fitting**。当用户选择一个地点后，地图引擎（Mapbox/Leaflet）应该调用 `fitBounds(bbox)`，而不是 `flyTo(center)`。这能确保用户看到整个城市的轮廓或整个建筑。

---

## 5.8 本章小结

1.  **本质**：Nominatim 是一个将空间关系转化为层级地址树的引擎，它依赖精确拼写而非模糊匹配。
2.  **API 策略**：
    *   Agent 交互优先使用**结构化查询**。
    *   利用 `viewbox` 解决同名地点冲突。
    *   利用 `zoom` 控制逆地理编码的颗粒度（是查天气还是查门牌）。
3.  **数据标识**：持久化存储 `osm_type` + `osm_id`，永远不要存 `place_id`。
4.  **性能**：它是 I/O 密集型应用，缓存和 SSD 是核心优化点。

---

## 5.9 练习题

### 基础题 (50%) - 熟悉基本逻辑

1.  **参数构建**：你需要查询“北京市海淀区的中关村地铁站”。请写出两种请求 URL 的参数部分：
    *   A: 自由文本模式。
    *   B: 结构化查询模式（更推荐）。
    *   *Hint: 查阅文档关于 `amenity`, `station`, `city` 等结构化字段的支持。*

2.  **逆地理编码颗粒度**：用户在一个大型森林公园内。
    *   请求 A 使用 `zoom=18`，可能返回什么类型的对象？
    *   请求 B 使用 `zoom=10`，通常会返回什么？
    *   *Hint: 考虑 `highway=path` (小路) vs `boundary=administrative` (行政区)。*

3.  **边界框适配**：Nominatim 返回的 bbox 格式是 `["39.9", "40.1", "116.3", "116.5"]` (LatMin, LatMax, LonMin, LonMax)。Leaflet 地图库要求的 `fitBounds` 格式是 `[[LatMin, LonMin], [LatMax, LonMax]]`。请写出转换逻辑（伪代码）。

### 挑战题 (50%) - Agent 与系统设计

4.  **场景设计：多语言导航 Agent**
    用户用中文问：“带我去 Time Square”。
    系统如果直接搜 "Time Square" 可能会搜到纽约的时代广场。
    *   如何利用用户的当前 GPS 坐标（假设在上海）和 `Accept-Language` 头，确保 Agent 返回的是“上海时代广场（如果有）”的相关信息，且名字显示为中文？
    *   *Hint: 结合 `viewbox`, `bounded`, `accept-language` 三个参数。*

5.  **容错设计：消失的 ID**
    你的 App 收藏夹存了一个地点的 `osm_id=W123456`。三个月后用户点击它，Nominatim 返回 404（该建筑被重新绘制了）。
    请设计一个**回退（Fallback）策略**。你手头有三个月前存的 `display_name`, `lat`, `lon`, `osm_type`, `osm_id`。
    *   *Hint: 不要直接报错。利用手头的旧坐标或者旧名称，应该发起什么样的新请求？*

6.  **搜索体验优化：分类过滤**
    用户搜索“加油站”。Nominatim 的 `/search` 支持 `q=加油站`，但结果可能包含名为“加油站烧烤”的餐馆。
    如何利用 Nominatim 的 Special Phrases 或者 `limit` + 后处理逻辑，来确保只返回真正的加油站？
    *   *Hint: 观察返回 JSON 中的 `class` 和 `type` 字段（如 `amenity=fuel`）。是在 API 侧过滤还是在客户端过滤？*

<details>
<summary>点击查看参考答案思路</summary>

1.  **参数构建**：
    *   A: `q=中关村地铁站, 海淀区, 北京市`
    *   B: `poi=中关村地铁站&city=北京市&district=海淀区` (注：Nominatim 的结构化字段通常是 `city`, `street`, `county`, `state`, `country`。POI 往往还是要放在 `q` 里或者作为 `query` 组合 `viewbox`。最标准的结构化其实是 `q=中关村地铁站&city=Beijing`，完全纯粹的 KV 结构化支持有限)。
2.  **颗粒度**：
    *   `zoom=18`：可能返回公园内的某条小路名称，或者公共厕所、长椅等微小设施。
    *   `zoom=10`：返回该公园所在的城市或区县名称（如“海淀区”）。
3.  **BBox 转换**：
    *   Input: `[lat1, lat2, lon1, lon2]`
    *   Output: `[[lat1, lon1], [lat2, lon2]]` (注意字符串转浮点数)。
4.  **多语言 Agent**：
    *   Headers: `Accept-Language: zh-CN`
    *   Params: `q=Time Square&viewbox=121.xx,31.xx,121.xx,31.xx&bounded=0` (bounded=0 允许搜不到上海时回退到纽约，或者 bounded=1 强行只搜上海)。
5.  **ID 消失回退**：
    *   策略：捕获 404 -> 降级为 `/search` -> 参数 `q={旧名称}&lat={旧lat}&lon={旧lon}&bounded=1`。如果名字也没了，尝试 `/reverse` -> 参数 `lat={旧lat}&lon={旧lon}` 找最近的新地点。
6.  **分类过滤**：
    *   Nominatim 本身对类别过滤支持较弱（除了 `exclude_place_ids` 和 `featuretype`）。
    *   最佳实践：请求更多数据 `limit=20` -> 开启 `addressdetails=1` -> 客户端/中间层遍历结果，只保留 `type=='fuel'` 的条目。

</details>

---

## 5.10 常见陷阱与错误 (Gotchas)

### 1. 坐标系的“沉默杀手” (EPSG:3857 vs 4326)
*   **现象**：前端发来的坐标是巨大的数字（如 12950000），传入 Nominatim 后报错或结果在南极。
*   **原因**：地图瓦片（如 Mapbox/Google Maps）常用 Web Mercator (EPSG:3857)，单位是米。而 Nominatim 严格只接受 WGS84 (EPSG:4326)，即经度。
*   **Debug**：看到坐标值超过 180，立刻警觉，这是投影坐标，必须转换 (`ST_Transform`) 后再传。

### 2. 空格与 URL 编码
*   **现象**：搜索 "Main St & 5th Ave" 失败。
*   **原因**：简单的字符串拼接未做 URL Encoding。`&` 符号截断了参数。
*   **对策**：始终使用标准的 URL 构造函数（如 Python 的 `urllib.parse.urlencode` 或 JS 的 `URLSearchParams`）。

### 3. 被官方封禁 (HTTP 403/429)
*   **现象**：程序跑得好好的，突然所有请求都返 403。
*   **原因**：使用了 `nominatim.openstreetmap.org` 公共服务，且未设置 `User-Agent`，或者并发过高。
*   **规则**：
    *   公共服务严格要求：必须带 `User-Agent`（标识你的 App 名字）。
    *   **绝对禁止**批量并发请求。
    *   **生产环境必须自建**。

### 4. “村”与“居委会”的缺失
*   **现象**：在中国的城市搜索，只能精确到区县，很难搜到具体的“某某社区”“某某村”。
*   **原因**：OSM 在中国的行政区划数据（Boundary Relations）往往只完善到区县级（Level 6）。街道（Level 8）和社区（Level 10）数据大量缺失。
*   **对策**：不要指望 Nominatim 能处理中国所有的行政层级。如果业务强依赖社区搜索，需要引入专门的 POI 库做补充索引。
