# 第9章｜设计“面向业务”的 OSM 导航 API（把 Nominatim/OSRM 变成产品级接口）

## 1. 开篇段落

在企业级架构或 MCP（Model Context Protocol）体系中，直接暴露底层的 Nominatim 或 OSRM 接口是架构上的“反模式”。底层引擎关注的是“图算法”和“索引查询”，而上层业务关注的是“地点”、“行程”和“异常处理”。

本章的目标是构建一个 **Facade API（外观模式层）**。将学习如何定义通用的**资源模型（Resource Model）**，制定严格的**坐标与单位规范**，设计支持**多模态（GET/POST）**的路由接口，并建立一套能让 LLM 理解的**错误反馈机制**。无论底层是 OSRM、Valhalla 还是 Google Maps，这层 API 协议都应保持不变。这不仅是为了解耦，更是为了让你的 Navigation MCP 能够稳定运行。

---

## 2. 文字论述

### 2.1 为什么要设计 Facade API（中间层架构）

在将导航能力接入 LLM Agent 时，我们面临三个核心挑战：
1.  **协议碎片化**：Nominatim 返回 XML/JSON，OSRM 返回 GeoJSON + 自定义注解。Agent 需要统一的 Schema。
2.  **上下文缺失**：底层引擎不保留状态。例如，OSRM 不知道“家”在哪里，它只认坐标。Facade 层需要处理“别名解析”。
3.  **容错性差**：当 OSRM 返回 `NoRoute` 时，它不会告诉你是因为“起点在海里”还是“终点在步行街”。Facade 层需要通过空间析（PostGIS）补充错误上下文，告诉 Agent 如何修正。

```ascii
+-----------------------+       +-----------------------------+
| LLM Agent / Mobile App|       |      Navigation Facade      |
| (Client)              | <---> |      (API Gateway)          |
+-----------------------+       +--------------+--------------+
                                               |
         +-------------------------------------+----------------------------------+
         |                     |                       |                          |
+--------v-------+    +--------v----------+    +-------v-------+          +-------v-------+
|  Nominatim     |    |   OSRM (Driving)  |    | OSRM (Biking) |          | PostGIS (DB)  |
| (Raw Geocoder) |    |  (Routing Engine) |    | (Routing Eng) |          | (Auth/Cache)  |
+----------------+    +-------------------+    +---------------+          +---------------+
```

### 2.2 核心资源模型设计（The Resource Model）

为了保证 API 的长期演进，必须定义标准化的“名词”。不要在 API 中混用 `lat`、`latitude`、`y`。

#### 2.2.1 坐标原子（Coordinate Atom）

**Rule-of-Thumb（坐标黄金法则）**：
1.  **输入/输出**：强制使用 **`[longitude, latitude]`** 数组格式（GeoJSON 标准）。
2.  **精度控制**：在 API 层强制截断为 **6位小数**（约 0.11米精度）。7位以上是测量噪声，只会破坏缓存命中率。
3.  **命名约定**：如果在对象中使用，必须叫 `lon` 和 `lat`。

#### 2.2.2 地点对象（Location Object）

不要只返回一个坐标，要返回一个“对象”。

```json
// Location Schema 定义
{
  "id": "osmid_12345_node",      // 唯一标识，用于反查
  "name": "北京大兴国际机场",      // 最佳显示名
  "point": [116.4105, 39.5097],  // 统一坐标
  "address": {                   // 结构化地址（可选，按需返回）
    "city": "北京市",
    "district": "大兴区"
  },
  "type": "aerodrome",           // 业务分类（映自 OSM tags）
  "confidence": 0.95             // 匹配置信度（对搜索很重要）
}
```

#### 2.2.3 路线对象（Route Object）

OSRM 的原始返回非常庞大，包含大量几何点。Facade API 应提供简化版，仅当 `details=full` 时才返回完整几何。

*   **`summary`**：摘要（距离、时间、路况）。
*   **`geometry`**：建议使用 **Polyline Algorithm** 编码字符串（如 `_p~iF~ps|U_ulLnnqC_mqNvxq`），将几十 KB 的坐标数组压缩为几百字节，大幅降低传输延迟和 Token 消耗。
*   **`legs`**：分段信息（途经点之间的路段）。
*   **`instructions`**：文本化的导航指令（给人类读，或给 LLM 朗读）。

---

### 2.3 关键端点设计（API Endpoints）

我们将 API 分为三个核心能力域：**G（Geocoding）、R（Routing）、S（Spatial Analysis）**。

#### 2.3.1 地理编码域：`/v1/geocode`

*   **GET `/v1/geocode/search`**
    *   **核心参数**：
        *   `q` (string): 自由文本。
        *   `bias_point` (coord): 偏置坐标（优先搜索该点附近）。
        *   `filter` (string): 类别过滤，如 `category:restaurant`。
    *   **设计陷阱**：不要透传 Nominatim 的 `viewbox` 参数，设计者应该将其封装为更易懂的 `region` 或 `city_code` 参数。

*   **GET `/v1/geocode/reverse`**
    *   **核心参数**：`point` (coord), `layers` (address | poi | road)。
    *   **业务逻辑**：OSM 原生反查通常返回最近的任何对象。业务 API 应该允许用户指定：“我只要最近的**道路**”或“我只要所在的**行政区**”。

#### 2.3.2 路由域：`/v1/route`

*   **POST `/v1/route/directions`** (推荐使用 POST)
    *   **设计理由**：GET 请求 URL 长度有限。当请求包含 50 个途经点（TSP 问题）或复杂的避让区域（Polygons）时，必须用 POST。
    *   **Request Schema**:
        ```json
        {
          "waypoints": [
            {"point": [116.3, 39.9], "type": "break"},  // 必停点
            {"point": [116.4, 39.8], "type": "through"} // 途经点（不停车，仅用于塑形）
          ],
          "profile": "driving-car", // 模式：驾车、骑行、步行
          "preference": "fastest",  // 偏好：最快、最短、经济
          "exclude": ["toll", "ferry"], // 避让：收费、轮渡
          "options": {
            "geometry_format": "polyline6"
          }
        }
        ```

*   **POST `/v1/route/matrix`**
    *   **功能**：计算 N x M 的时距矩阵（不返回几何路线）。
    *   **场景**：派单系统、最近门店查找。
    *   **性能警示**：务必在 API 层限制 N 和 M 的大小（例如 `N*M <= 2500`），防止 OSRM 内存爆炸。

#### 2.3.3 空间分析域：`/v1/spatial`

*   **POST `/v1/spatial/match` (Map Matching)**
    *   **功能**：将原始 GPS 轨迹“吸附”到路网上。
    *   **输出**：除了修正后的坐标，**必须返回 `tracepoints` 的元数据**，特别是 `waypoint_index`（对应输入的第几个点）和 `matchings_confidence`。
    *   **Gotcha**：如果中间有一段轨迹缺失（如隧道），OSRM 可能会“猜”一条路。API 需要标记这段是“推测（interpolated）”还是“匹配（matched）”。

---

### 2.4 动力学与几何近似的 API 表达

导航不仅是几何连线，还涉及物理世界的限制。API 必须显式表达这些概念。

1.  **Ghost Start / Snapping (幽灵起点)**
    *   **问题**：用户在公园内部（无车行道）发起“驾车”导航。
    *   **处理**：引擎会将起点吸附到最近的公路（距离 200米）。
    *   **API 响应**：
        ```json
        "metadata": {
          "query_point": [116.300, 39.900],   // 用户点的
          "snapped_point": [116.302, 39.901], // 实际规划起点
          "snapping_distance_m": 250          // 偏差距离
        }
        ```
    *   **作用**：LLM 可以据此提示用户：“导航将从 250 米外的 xx 路开始，请先前往那里。”

2.  **Time Window (时间窗)**
    *   OSRM 支持基于时间段的限行（例如 `conditional_access`）。
    *   API 请求应包含 `departure_time` (ISO 8601)。如果为空，默认为 `now`。这决定了是否避开早晚高峰禁行路段。

---

### 2.5 错误模型：让 Agent 知道该怎么办

HTTP 状态码不足以表达导航业务错误。你需要定义 **`error_code`** 和 **`resolution_hint`**。

| HTTP | error_code | 含义 | 给 Agent 的建议 (resolution_hint) |
| :--- | :--- | :--- | :--- |
| 422 | `ROUTE_NOT_FOUND` | 两点间物理不可达 | 检查是否有跨海、跨国界或进入了封闭区域（如军事区）。 |
| 422 | `START_POINT_ISOLATED` | 起点无法吸附到道路 | 起点可能在荒漠或水域，建议扩大搜索半径或更改出行模式（驾车->步行）。 |
| 400 | `WAYPOINTS_LIMIT_EXCEEDED` | 途经点太多 | 将请求拆分为多个子航段。 |
| 429 | `RATE_LIMIT_EXCEEDED` | 限流 | 指数退避重试。 |
| 503 | `ENGINE_BUILDING` | 数据在更新 | 稍后重试（系统维护中）。 |

**LLM Friendly Error 示例**：
```json
{
  "error": {
    "code": "ROUTE_NOT_FOUND",
    "message": "Cannot find route between point A and B.",
    "details": {
      "reason": "DifferentComponents", // OSRM 术语，意为孤岛
      "hint": "Start point is on an island without bridges. Try mode='ferry' or check coordinates."
    }
  }
}
```

---

### 2.6 安全、限流与元数据

*   **Request ID & Trace ID**：每个请求必须分配 UUID，贯穿 Nginx -> Facade -> OSRM。日志中必须能串联。
*   **Rate Limiting**：
    *   **基于计算量限流**：Matrix 接口的权重应是 Simple Route 的 10 倍。不要只限制 QPS，要限制“计算单元”。
    *   **Header**：返回 `X-RateLimit-Remaining`，让 Agent 知道是否需要减速。

---

## 3. 本章小结

*   **Facade 即使命**：Facade API 存在的意义是将“地理数据问题”转化为“业务逻辑交互”，屏蔽底层的复杂性。
*   **坐标铁律**：永远使用 `[lon, lat]` 数组，并控制在 6 位小数。
*   **显式吸附**：总是告诉调用者“我实际从哪里开始规划的”，这是解决“导航起点不准”投诉的关键。
*   **错误可解释**：不要只扔出 404，要用错误码指导 Agent 是“换个点”、“换个模式”还是“拆分请求”。
*   **动词选择**：简单查询用 GET，复杂规划与矩阵计算用 POST。

---

## 4. 练习题

**基础题**
1.  **JSON 构造**：请编写一个 `/v1/route/directions` 的 POST 请求体示例，要求：从“北京西站”到“天安门”，中间途经“军事博物馆”，出行模式为“骑行”，且要求返回详细的导航文本指令。
2.  **精度计算**：如果 API 不做截断，传入了 10 位小数的坐标。请计算 0.0000000001 度在赤道上大约代表多少毫米？这对 OSRM 的缓存机制有什么坏处？
3.  **吸附逻辑**：用户在一条跨江大桥的**下方**（河岸公园）请求导航。GPS 坐（二维）看起来是在桥上。Facade API 应该如何通过参数（如 `radiuses` 或 `bearings`）协助 OSRM 区分“桥上”和“桥下”？

**挑战题**
4.  **接口幂等与缓存键设计**：
    为了节省 OSRM 算力，你在 Facade 层实现了 Redis 缓存。对于 POST 请求，你需要生成一个 Cache Key。请设计一个算法，将 JSON Body 转化为唯一的 Key。
    *   *提示：字段顺序不同（{"a":1, "b":2} vs {"b":2, "a":1}）应该生成相同的 Key 吗？坐标 `116.3000001` 和 `116.3` 应该命中同一个 Key 吗？*
5.  **多模态混合路由设计**：
    Agent 想要规划“先骑行到地铁站，再坐地铁，最后步行”的路线。目前的 OSRM 实例通常是单模式（只有车或只有步行）。请构思 Facade API 如何通过编排多次 OSRM 调用和一次 Public Transit API（如 GTFS）调用来实现此功能，并设计响应结构。
6.  **隐私脱敏**：
    作为 API 提供方，你需要在日志中记录用户的起终点以便 Debug，但不能泄露用户隐私。请提出一种针对地理坐标的日志脱敏方案。

<details>
<summary>点击查看练习题提示 (Hints)</summary>

1.  *提示：注意 `waypoints` 数组的顺序；`mode` 应为 `cycling`；`steps=true`。*
2.  *提示：赤道周长约 40000km。1度 ≈ 111km。10位小数是原子级别的精度。这会导致几乎所有的请求 Hash 都不同，缓存命中率降为 0。*
3.  *提示：利用 OSRM 的 `bearings`（指定起点的行进方向）或在应用层先判断用户是否在桥的投影范围内，结合高度信息（如果 GPS 有）或用户手动选择。*
4.  *提示：先对 Object 进行 Key 排序，再序列化；对坐标进行 Round(6) 处理。使用 Canonical JSON 方案。*
5.  *提示：Facade 需要将其拆解为：Route(A->Station1, bike) + Transit(Station1->Station2) + Route(Station2->B, foot)。响应中需包含 `segments` 数组，每个 segment 有不同的 `mode`。*
6.  *提示：GeoHash 截断（保留前5位），或者对坐标保留 2 位小数（城市级别精度）。*
</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 默认速度的陷阱
*   **问题**：用户抱怨 ETA（预计到达时间）极其不准。
*   **原因**：OSRM 默认使用的 `car.lua` 配置文件中，各级道路速度是静态的（如 highway=primary 默认为 60km/h）。在城市拥堵路段这太快了，在高速空闲路段又太慢了。
*   **Facade 对策**：不要盲信 OSRM 的 `duration`。Facade 层应允许接入简单的线性修正公式（如 `duration * 1.2`）或对接实时路况系数。

### 5.2 忽略 `country_code`
*   **问题**：在欧洲边境或行政区划复杂地区，地理编码搜索“Cambridge”可能返回英国的，也可能返回美国的。
*   **对策**：API 必须提供 `country_codes` 参数（ISO 3166-1 alpha-2），并强烈建议 Client 传该参数。

### 5.3 Polyline 的精度丢失
*   **问题**：前端解码 Polyline 后，发现路线在地图上呈锯状，甚至切过了建筑物。
*   **原因**：标准的 Google Polyline Algorithm 只有 5 位小数精度（约 1 米）。对于高精地图或复杂的立交桥，这不够。
*   **对策**：OSRM 支持 `polyline6`（6位精度）。务必在 options 中明确指定使用哪种编码，并在文档中告知 Client。

### 5.4 所有的路看起来都一样
*   **问题**：指令返回 "Turn right"，但没说是“向右转进入辅路”还是“向右转进入高速”。
*   **对策**：OSRM 的 response 中包含 `modifier` 字段（如 `slight right`, `sharp right`）。API 层应透传这些修饰符，这对语音播报至关重要。
