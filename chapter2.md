# 第2章｜OSM 数据模型与 ODbL 合规（底层拓扑与法律边界）

## 2.1 开篇段落

在开始下载和处理 TB 级别的 OSM 数据之前，必须纠正一个常见的认知偏差：**OpenStreetMap 不是一张地图，而是一个没有任何图层概念的拓扑数据库**。

当你看着 Google Maps 或高德地图时，你看到的是经过渲染的瓦片（Tile）；但当你作为开发者面对 OSM 时，你面对的是一个由数亿个节点和连线构成的巨型图（Graph）。对于导航 API 和 Navigation MCP 而言，视觉上的“路”没有任何意义，只有数据库中的“连通性”才决定了 LLM 能否规划出一条从 A 到 B 的合法路线。

本章将深入 OSM 的原子结构，揭示导航引擎（如 OSRM）是如何理解“路口”、“立交桥”和“禁止左转”的。同时，我们将详细拆解 ODbL 协议，特别是对于商业应用和 LLM Agent 场景，明确“数据使用”与“数据衍生”的界限，确保你的服务在合规的轨道上运行。

---

## 2.2 OSM 的三大基本要素 (The Primitives)

OSM 的数据模型极其精简，仅由三种原语（Primitives）构成。理解它们的相互引用关系是理解整个系统的关键。

### 1. Node (节点) —— 空间锚点
- **定义**：地理空间的基础原子，由 `ID` (64-bit integer)、`Latitude` (纬度)、`Longitude` (经度) 组成。
- **导航语义**：
  - **几何支撑**：定义道路弯曲的形状。
  - **拓扑连接点**：当两个 Way 引用同一个 Node ID 时，导航引擎认为它们是物理连接的（即路口）。
  - **POI 载体**：孤立的 Node 可以代表一个商店、红绿灯或公交站。

### 2. Way (路径/区域) —— 线性连接
- **定义**：由 2 到 2,000 个 Node ID 构成的**有序列表**。注意是引用”，Way 本身不存储坐标，只存储 Node ID 的序列。
- **导航语义**：
  - **方向性**：Node 的存储顺序决定了 Way 的方向。这对 `oneway=yes`（单行道）至关重要。
  - **Open Way (线)**：首尾 Node 不同。代表道路、河流、铁路。
  - **Closed Way (面)**：首尾 Node ID 相同。代表环岛（Roundabout）、建筑物、公园。
  - **Polyline 甚至 Polygon**：在 PostGIS 中，Open Way 映射为 `LineString`，Closed Way 视 Tag 不同映射为 `Polygon` 或 `LineString`（如环岛）。

### 3. Relation (关系) —— 逻辑约束
- **定义**：最抽象也最强大的元素。它是一个有序列表，成员可以是 Node、Way 或其他 Relation。每个成员都有一个 `role` (角色) 字符串。
- **导航语义**：这是初学者最容易忽视的部分。
  - **转向限制 (Turn Restrictions)**：定义“禁止左转”、“必须直行”。
  - **线路 (Route)**：定义公交线、地铁线（由成百上千条 Way 组）。
  - **多边形 (Multipolygon)**：带孔洞的建筑物或复杂的行政区边界。

> **Rule of Thumb (拓扑第一定律)**：
> 在 OSM 中，两条线在地图上视觉交叉（坐标重合）**并不代表它们相交**。只有当它们共享同一个 **Node ID** 时，导航引擎才认为它们是相通的。否则，这就是立体交叉（立交桥/隧道）。

---

## 2.3 深入 Tags：从文本到导航权重

Tags 是附加在上述三个元素上的键值对（Key-Value）。导航引擎（OSRM/Valhalla）本质上就是一个**将 Tags 映射为 Edge Weights（边权重）的编译器**。

### 关键 Tag 解析

#### 1. 道路层级 (`highway=*`)
这是决定路由偏好的核心。
- `motorway` / `trunk`: 高速与快速路。通常隐含 `oneway=yes` 和高限速。
- `primary` / `secondary` / `tertiary`: 主干道到支路。
- `residential` / `service`: 居住区与内部道路（权重极低，非终点尽量不走）。
- `track` / `path`: 农径或小路（通常汽车不行）。

#### 2. 通行权限 (`access=*`)
这是决定“能不能走”的布尔开关。
- **默认值逻辑**：如果没写 `access`，默认对所有模式开放（除非 `highway=motorway` 默认禁行人/骑行）。
- **层级覆盖**：`access=no` + `bus=yes` = 只有公交车能进。
- **常见陷阱**：`access=private` (私有道路，导航通常避开除非是终点)、`access=customers` (仅限顾客)。

#### 3. 单行与方向 (`oneway=*`)
- `yes` / `1`: 仅允许沿 Node 序列正向通行。
- `-1`: 仅允许沿 Node 序列**逆向**通行。
- `reversible`: 潮汐车道（需要结合时间段，复杂处理）。

#### 4. 物理限制
- `maxspeed=*`: 影响 ETA 计算。
- `maxheight=*` / `maxweight=*`: 卡车导航的命门。
- `width=*`: 狭窄道路避让。

### ASCII 图解：Tag 如何构建有向图

```text
       Node A (id:1)        Node B (id:2)         Node C (id:3)
          o--------------------o---------------------o
          
Scenario 1: 普通双向路
Way 1 (A->B->C): highway=residential
Graph Edge: A->B (cost:10), B->A (cost:10), B->C (cost:10), C->B (cost:10)

Scenario 2: 单行道
Way 1 (A->B->C): highway=primary, oneway=yes
Graph Edge: A->B (cost:5), B->C (cost:5)
(注意：B->A 和 C->B 的 Edge 不存在)

Scenario 3: 逆向单行道
Way 1 (A->B->C): highway=primary, oneway=-1
Graph Edge: C->B (cost:5), B->A (cost:5)
(注意：车辆实际行驶方向与 Node 存储顺序相反)
```

---

## 2.4 复杂路口建模：Relation 的核心作用

如果不处理 Relation，你的导航就会变成“违章指挥器”。最典型的场景是 **Turn Restriction (转向限制)**。

### 数据结构：Type=restriction
一个标准的禁止左转 Relation 包含三个成员：
1.  **from** (Way ID): 车辆当前所在的道路。
2.  **via** (Node ID): 路口节点。
3.  **to** (Way ID): 车辆想要进入的道路。
4.  **Tag**: `restriction=no_left_turn` 或 `restriction=only_straight_on`。

### ASCII 演示：为什么需要 Relation

```text
      |   |
      | C | Way C (To)
      |   |
------+   +-------
Way A   X   Way B
(From)+   +-------
      |   |
      | D |
      |   |

车辆从 A 行驶到路口 X。
物理上：A 连接 B, C, D，看起来都能走。
逻辑上：
Relation (ID: 999):
  - type: restriction
  - from: Way A
  - via: Node X
  - to: Way C
  - restriction: no_left_turn

导航引擎解析：
在生成图时，生成 Edge (A->B), Edge (A->D)，但**显式删除** Edge (A->C) 或将其权重设为无限大。
```

> **进阶**：如果是 `restriction:conditional = no_left_turn @ (07:00-09:00)`，这被称为**时变限制**。标准的 OSRM 处理不了这个（它需要预处理成静态图），需要使用 Valhalla 或 GraphHopper 这种支持动态 Costing 的引擎。

---

## 2.5 ODbL 协议详解：合规的红线

Open Database License (ODbL) 是悬在每个 OSM 开发者头上的达摩克利斯之剑。很多人因为害怕它而放弃 OSM。其实只要理清 **Derivative Database (衍生库)** 与 **Produced Work (作品)** 的区别，商业应用完全可行。

### 1. 核心定义
- **Derivative Database (衍生数据库)**：如果你修改了 OSM 数据（修正坐标、添加属性），并且这个修改足以构成一个新的数据库。
  - **触发条件**：将 OSM 数据与你的私有数据（如私有路网）进行**物理融合**或**逻辑强绑定**。
  - **义务**：必须以 ODbL 协议公开这个新数据库（即“传染性”）。

- **Produced Work (作品)**：使用数据产生的输出。
  - **范畴**：渲染出的地图图片（Tiles）、计算出的路径（Route Geometry）、地理编码返回的坐标（Lat/Lon）、文本指令。
  - **义务**：仅需署名。**不需要开源你的算法，也不需要开源生成的这些结果。**

- **Collective Database (集合数据库)**：将 OSM 与其他数据放在一起，但保持独立。
  - **范畴**：你在 PostGIS 里建了两个 schema，一个放 `osm_roads`，一个放 `my_proprietary_pois`。它们各自独立更新，仅在询时 join。
  - **后果**：这是安全的。私有数据不会被 ODbL 传染。

### 2. 常见场景判定表

| 场景 | 属于分类 | 是否需要开源私有数据？ | 备注 |
| :--- | :--- | :--- | :--- |
| **场景 A**：用 OSM 数据渲染地图瓦片，叠加自己采集的门店 POI 图层。 | Produced Work (瓦片) + Collective DB (叠加) | **否** | 最常见的商业模式。 |
| **场景 B**：下载 OSM 路网，用算法修复了断头路，然后作为 API 出售路网文件。 | Derivative Database | **是** | 你必须把修复后的路网开源。 |
| **场景 C**：用户请求导航，服务器用 OSM 算路，返回 JSON 给用户。 | Produced Work | **否** | 你的路由算法和返回的 JSON 都是你的。 |
| **场景 D**：提取 OSM 建筑物轮廓，用来训练 AI 识别卫星图。 | Produced Work (模型权重) | **否** (目前社区共识) | 训练出的模型通常视为作品。 |

### 3. Attribution (署名) 具体要求
ODbL 要求“合理的署名”对于 MCP Tool 和 Agent：
- **Tool Description**: 在工具描述中注明 "Data from OpenStreetMap".
- **Tool Output**: 在 JSON 结果中包含 `attribution` 字段。
- **UI**: 如果有前端展示，必须在地图角落显示链接。

---

## 2.6 数据版本与唯一标识符 (ID) 的易变性

这是 OSM 开发中最容易踩的坑之一：**不要持久化存储 OSM ID**。

- **ID 不稳定**：如果一个用户删除了 Way A 并重新画了一条一模一样的，新 Way 的 ID 会变化。
- **操作原子性**：如果一个 Way 被切割（例如中间加了个红绿灯），原来的 ID 可能保留给其中一段，另一段生成新 ID；或者旧 ID 消失，生成两个新 ID。
- **应对策略**：
  - 如果你需要存储“某家店的位置”，存 **Lat/Lon 坐标**，不要存 `node_id`。
  - 如果你需要存储“某条路”，存 **OSM ID** 仅作为临时缓存键值，并接受它随时可能失效（Latent Link）。定期（如每周）重新匹配。

---

## 2.7 常见的几何与拓扑陷阱 (Gotchas)

在处理数据时，以下情况会导致导航引擎崩溃或产生幻觉：

1.  **非平面交叉 (Grade Separation)**：
    - 两条路 Lat/Lon 相交，但没有共享 Node。
    - *原因*：这是立交桥或隧道。
    - *检查*：查看 `layer=*` 或 `bridge=*` / `tunnel=*` 标签，虽然这些主要用于渲染，但拓扑上主要看**是否共享 Node**。

2.  **区域 (Area) 陷阱**：
    - 你想找“人民广场”，结果路由把你带到了广场边缘的一条线上。
    - *原因*：OSM 中广场通常是一个 Closed Way (Polygon)。路由引擎通常只能吸附（Snap）到最近的 Edge 上。如果你没有计算 Polygon 的质心（Centroid）作为导航终点，结果可能很奇怪。

3.  **虚拟连接线 (Implied Connection)**：
    - 有些数据中，道路在小区门口断开了，但实际能走。
    - *原因*：OSM 是众包数据，质量参差不齐。
    - *解决*：商业引擎通常会有“路网修”预处理步骤（Snap to nearest within 5 meters），但 OSRM 默认通过 Profile 也可以容忍一定误差，或者数据本身需要人工修复。

4.  **混合模式路径**：
    - `highway=pedestrian` (步行街) 连接了 `highway=primary`。
    - 汽车导航会在接口处断开，这是正确的。但如果这是唯一的入口（例如只能步行最后 50 米），你的 API 需要支持“多模态最后就在一公里”（Car -> Walk）。

---

## 2.8 本章小结

1.  **拓扑至上**：Node 是连接点，Way 是连线，Relation 是规则。没有共享 Node 就不连通。
2.  **Tag 决定权重**：同样的几何形状，`highway` 和 `access` 标签决定了它是高速公路还是人行道。
3.  **Relation 决定合法性**：必须解析 Relation 才能处理转向限制，否则导航是非法的。
4.  **ODbL 边界**：输出结果（Route/Image）是安全的 Produced Work；混合并分发原始数据是危险的 Derivative Database。
5.  **ID 易变**：永远不把 OSM ID 当作永久主键，存坐标才是王道。

---

## 2.9 练习题

### 基础题
1.  **数据解读**：查找一个 `junction=roundabout` 的 Way。它的 `oneway` 标签通常是什么？如果没有写 `oneway`，根据 OSM 规范它隐含的方向是什么？
2.  **连通性判断**：
    - Way A Nodes: `[101, 102, 103]`
    - Way B Nodes: `[104, 105, 106]`
    - Way C Nodes: `[102, 105]`
    - 问：从 Way A 能走到 Way B 吗？路径是什么？
3.  **Tag 优先级**：一条路同时标记了 `highway=track` 和 `motor_vehicle=yes`。普通的家用轿车（Car Profile）通常会避开它吗？为什么？

### 挑战题 (Thinking)
4.  **复杂 Relation**：有些禁止左转是“仅对卡车生效”的（`restriction:hgv = no_left_turn`）。设计一个简单的 JSON 结构来描述这个规则，以便你的 Navigation MCP 能够告诉 LLM “如果你开的是卡车，这里不能转”。
5.  **Schema 设计**：在 PostGIS 中，你该如何存储 `Closed Way`（比如一个建筑物）？是用 `LineString` 还是 `Polygon`？这对计算“我是否在建筑物内”有什么性能影响？
6.  **合规思考**：你的公司是一个外卖平台。你下载了 OSM 数据，然后让骑手在送餐过程中标记“这里其实有个门”或者“这条路其实不通”。
    - 场景 A：你把这些标记只用于优化自家骑手的路线调度。
    - 场景 B：你在 App 地图上把这条路画了出来给用户看。
    - 哪种场景触发 ODbL 的 Share-Alike？如何规避？

<details>
<summary>点击展开提示 (Hint)</summary>

1.  Roundabout 通常隐含 `oneway=yes`。行驶方向取决于国家（右侧通行国家为逆时针）。
2.  能。A (102) -> C (102 -> 105) -> B (105)。
3.  会避开（权重惩罚），但不会视为“不可通行”。`track` 基础速度很慢，且路况默认很差，除非是唯一路径，否则路由算法倾向于绕路。
4.  你需要解析 `restriction:conditional` 或特定的交通工具后缀。JSON 需包含 `conditions` 字段。
5.  应存储为 `Polygon`（或 Geography 类型）。计算 `ST_Contains` 时，Polygon 效率远高于判断点是否在 LineString 围成的圈内。且 PostGIS 对 Closed Way 导入时通常默认处理为 Polygon。
6.  场景 A 属于内部使用（Internal Use），通常不强制开源，但处于灰色地带（如果分发到了骑手手机端 app，可能被视为公开分发）。场景 B 属于公开展示 Produced Work（地图），但如果地图本身包含了你们私有修正的路网数据（Derivative DB），则风险极高。规避方法：将修改点作为 Overlay（覆盖层/补丁）存储，不直接修改 OSM 原始表，仅在内存路由图中动态应用权重调整（Edge Weight customization）。

</details>

