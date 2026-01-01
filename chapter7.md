# 第7章｜路由引擎选型与路网建模（OSRM / GraphHopper / Valhalla 怎么选）

## 7.1 开篇段落：从几何连线到动力学搜索

在上一章中，我们通过 Nominatim 解决了“地点在哪里”（Geocoding）的问题。现在的核心挑战变为：“如何从 A 到 B”。

对于初学者来说，路由（Routing）往往被误解为在地图上寻找两点间的最短几何连线。然而，在导航工程的视角下，这是一个极具挑战的 **加权有向图搜索（Weighted Directed Graph Search）** 问题。你需要处理的不仅仅是距离，而是**动力学成本（Dynamic Cost）**：
*   距离短的路可能限速低；
*   限速高的路可能在修路；
*   看似连通的路口可能禁止左转；
*   对于卡车，3.5米高的桥洞就是一堵墙。

本章将带你深入 OSM 导航栈的“心脏”。我们将剖析市面上三大主流开源引擎——**OSRM**、**GraphHopper** 和 **Valhalla**——的架构差异，理解 **CH (Contraction Hierarchies)** 与 **MLD (Multi-Level Dijkstra)** 等核心算法的取舍，并学习如何编写 **Profile**，将 OSM 的 Tag 翻译成计算机可理解的路网权重。

**学习目标**：
1.  **能力图谱**：深入理解 Matrix、Match、Isochrone 背后的计算代价。
2.  **引擎内核**：掌握 OSRM（速度）、GraphHopper（灵活）、Valhalla（动态瓦片）的架构区别。
3.  **建模原理**：学会设计 Cost Function（成本函数），处理转向惩罚、路况权重与物理限制。
4.  **决策框架**：根据业务场景（通用导航 vs 运筹优化 vs 户外运动）做出正确的选型。

---

## 7.2 导航能力清单：API 背后的算力与场景

在选型之前，必须明确你的 API 需要提供哪些计算能力。不同的能力对内存和 CPU 的消耗模式截然不同。

### 1. 基础路由 (Point-to-Point Routing)
最基本的需求。输入起点、终点及途经点，返回几何（Geometry）和指令（Instructions）。
*   **核心挑战**：**歧义性处理**。起点在立交桥下，用户到底是在桥上还是桥下？
*   **性能要求**：P99 延迟通常要求在 50ms-200ms 内。

### 2. 距离矩阵 (Distance Matrix)
输入 $N$ 个起点和 $M$ 个终点，计算两两之间的行程时间/距离。
*   **场景**：派单系统（哪个司机离乘客最近？）、VRP（车辆路径规划）、TSP（旅行商问题）。
*   **算力特征**：计算复杂度是 $O(N \times M)$。虽然可以用单源最短路算法优化，但依然是 CPU 密集型操作。
*   **注意**：OSRM 在这方面是绝对的王者，支持极大规模的矩阵计算。

### 3. 地图匹配 (Map Matching / Snap to Road)
输入一串带有 GPS 噪声、漂移的轨迹点序列，输出这辆车实际行驶在路网上的哪条路上。
*   **原理**：**HMM (隐马尔可夫模型)**。
    *   *发射概率*：GPS 点距离某条路越近，概率越大。
    *   *转移概率*：车辆从路段 A 开到路段 B 是否符合拓扑逻辑（如是否连通、是否逆行）。
*   **场景**：网约车判责（是否绕路）、按里程计费、轨迹回放修正。

```ascii
      Map Matching 示意
      
      GPS 点:      o       o       o (漂移到路外)
                   |       |       |
      匹配路径:  ==+=======+=======+== (道路)
                Node A           Node B
```

### 4. 等时圈 (Isochrone)
给定一个中心点，计算在 $T$ 时间（或距离）内能到达的所有区域的多边形。
*   **算法**：通常基于 Dijkstra 或 BFS 的变体，向外扩散直到成本耗尽，然后连接边缘点生成多边形。
*   **场景**：外卖配送范围、房产选址（“步行 15 分钟生活圈”）。

---

## 7.3 三大引擎深度剖析：OSRM vs GraphHopper vs Valhalla

没有“完美”的引擎。这是关于 **预处理时间（Preprocessing Time）**、**查询速度（Query Speed）** 和 **灵活性（Flexibility）** 的“不可能三角”。

### 1. OSRM (Open Source Routing Machine)
*   **核心哲学**：**以灵活性换取极致速度**。
*   **技术栈**：C++ / Lua (Profile)。
*   **核心算法**：
    *   **CH (Contraction Hierarchies)**：预处理阶段，它会生成大量的“捷径（Shortcuts）”。比如 A->B->C 如果是必经之路，它会生成一条 A->C 的虚拟边。查询时，它不在原始图上搜索，而是在包含捷径的层次图上搜索。
    *   **MLD (Multi-Level Dijkstra)**：将图切分为分区（Partition）。支持更快的权重更新。
*   **适用场景**：高并发 API、大规模矩阵计算（Matrix）。
*   **致命弱点**：**权重固化**。一旦预处理完成（耗时数小时），你不能在请求时说“这次我想开高速”。要改变策略，必须修改 Lua 脚本并重新预处理。

### 2. GraphHopper
*   **核心哲学**：**均衡与生态**。
*   **技术栈**：Java。
*   **核心算法**：
    *   支持 CH（类似 OSRM，快但不灵活）。
    *   支持 **A* (A-Star) + Landmark (ALT)**：允许在查询时动态调整权重（Flexible Mode）。
*   **适用场景**：户外运动（Komoot 就基于此）、定制化路线（如：避开特定区域）、Java 后端集成。
*   **优势**：内存管理优秀，甚至可以在 Android 设备上离线运行。

### 3. Valhalla
*   **核心哲学**：**动态瓦片化 (Tiled) 与多模式融合**。
*   **技术栈**：C++。
*   **架构特点**：它不把整个地球加载到内存。它将路网切分为 Protocol Buffer 格式的瓦片。路由时，即时加载需要的瓦片并构建图。
*   **适用场景**：
    *   **多模式路由**：只有 Valhalla 能优雅处理“步行 -> 地铁 -> 共享单车”的混合路由。
    *   **资源受限环境**：由于按需加载瓦片，内存占用极低。
    *   **复杂动态成本**：支持极细粒度的请求参数（如：卡车宽2.5米、长10米、避开隧道、喜欢坡度小于5%的路）。
*   **弱点**：纯粹的 A-to-B 驾车导航速度比 OSRM 慢一个数量级；配置和部署极其复杂。

### 选型决策矩阵 (Rule-of-Thumb)

| 维度 | OSRM (CH) | OSRM (MLD) | GraphHopper | Valhalla |
| :--- | :--- | :--- | :--- | :--- |
| **查询速度** | 🚀 极快 (<5ms) | ⚡ 快 (<20ms) | ⚡ 快 (CH) / 🐢 中 (Flex) | 🐢 中等 |
| **预处理速度** | 🐢 极慢 | 🐇 中等 | 🐇 中等 | 🚀 快 |
| **内存消耗** | 🐘 巨大 | 🐘 巨大 | 🐕 中等 | 🐜 极低 |
| **实时路况** | ❌ 困难 | ✅ 支持 (Customize) | ❌ 困难 | ✅ 支持 |
| **请求时参数** | ❌ 不支持 | ❌ 不支持 | ✅ 支持 (Flex) | ✅✅ 极其丰富 |
| **多模式混合** | ❌ 不支持 | ❌ 不支持 | ⚠️ 勉强支持 | ✅✅ 原生支持 |

---

## 7.4 路网建模：Profile 与 Cost Function

路由引擎不看地图，它只看**图（Graph）**。你的核心工作是编写 Profile，将 OSM 的 Tag 转化为 **Edge Weight（边权重）** 和 **Turn Cost（转向成本）**。

公式：
$$ Weight = \frac{Length}{Speed \times Factor} + TurnPenalty + Bias $$

### 1. 边的建模 (Edge Modeling)
这是最直观的部分，主要依据 `highway` 标签。

*   **速度映射 (Speed Mapping)**:
    *   `highway=motorway` $\rightarrow$ 120 km/h
    *   `highway=residential` $\rightarrow$ 30 km/h
    *   `surface=unpaved` (土路) $\rightarrow$ 速度打 5 折。
*   **通行性 (Access)**:
    *   `access=private` $\rightarrow$ 除非是终点，否则权重 $\infty$。
    *   `bollard` (路障) $\rightarrow$ 汽车权重 $\infty$，自行车权重 0。

### 2. 节点的建模 (Turn Modeling)
初学者容易忽略：**路口是有成本的**。

*   **转向角度**：
    *   直行 (0°)：成本 0s。
    *   右转 (90°)：成本 2-5s（速）。
    *   左转 (-90°)：成本 15-30s（需等待对面车流）。
    *   掉头 (180°)：成本极高，除非无路可走。
*   **交通设施**：
    *   `highway=traffic_signals` (红绿灯)：增加平均等待时间（如 20s）。
    *   `highway=stop` (停车让行)：增加 5-10s。

### 3. 优先级与偏好 (Priority/Bias)
有时候我们希望“虽远必达”。
*   **场景**：骑行导航。
*   **逻辑**：即便距离更短，也要避开 `highway=primary`（主干道），优先走 `highway=cycleway`。
*   **实现**：给主干道施加一个 `Priority < 1.0` 的系数，人为“拉长”它的感知距离。

```ascii
      转向成本模型示意
      
          |
          | (A) Incoming Edge
          v
      -------- (Node: Traffic Light) ---------
          |
          | (B) Outgoing: Straight (Cost: 20s light wait)
          |
      (C) Outgoing: Left Turn 
          (Cost: 20s light + 15s crossing traffic = 35s)
```

---

## 7.5 约束路由与特种车 (Truck Routing)

为卡车或特种车辆建模是 OSM 导航中利润最高但也最复杂的领域。这里涉及 **硬约束** 与 **软约束**。

### 1. 物理维度的硬约束 (Hard Constraints)
如果车辆物理上通过不了，权重必须为 $\infty$。
*   `maxheight=*`: 隧道、桥梁高度。
*   `maxweight=*` / `maxaxleload=*`: 桥梁承重。
*   `maxwidth=*`: 窄路。

**陷阱**：在使用 OSRM (CH) 时，你无法动态传入 `height=3.8`。你必须为“4.0米卡车”、“3.5米卡车”分别预处理不同的 Profile。这就是为什么 OSRM 不适合做通用卡车导航，而 Valhalla（支持动态参数）更适合。

### 2. 法律与行政约束
*   `hazmat=*`: 危化品禁行。
*   `hgv=no`: 货车禁行区域（市中心）。
*   时间限制：比如“7:00-9:00 公交专用道”。这需要引擎支持**时间依赖路由 (Time-dependent Routing)**，目前开源引擎对此支持极其有限（通常仅作为静态惩罚处理）。

---

## 7.6 动态因素：实时交通 (Real-time Traffic)

OSM 数据是静态的。要实现“躲避拥堵”，你需要更新图的权重。

### OSRM 的 Traffic 更新流
OSRM 的 MLD 算法支持 `osrm-customize` 命令，它能在几秒钟内更新全图权重，而无需重新构图。

1.  **映射**：将外部交通流数据（Traffic Speed）匹配到 OSM 的 Way ID 上。
2.  **生成 CSV**：格式通常为 `from_node, to_node, speed`。
3.  **更新**：运行 `osrm-customize` 刷新 `.osrm.cells` 文件。
4.  **热加载**：向运行中的 OSRM 进程发送 `SIGHUP` 信号或调用 reload API。

### 历史路况 vs 实时路况
*   **实时**：基于当前探测数据。
*   **历史**：基于“周一早上8点”的统计数据。在 Profile 中，可以通过 CSV 查找表根据时间段设置基础速度。

---

## 7.7 评测框架：如何验证路由质量？

不要只看“能不能通”，要看“好不好走”。

### 1. 连通性测试 (Connectivity Ratio)
随机生成 10,000 对同城坐标。
*   如果 > 99.5% 成功规划：合格。
*   如果大量失败：检查是否有“孤岛”区域（数据断裂）。

### 2. 绕路系数 (Detour Factor)
$$ \text{Detour} = \frac{\text{Route Distance}}{\text{Great Circle Distance}} $$
*   城市环境：通常在 1.2 - 1.5 之间。
*   如果某路线系数 > 2.0，需人工审查：是因为中间有河没有桥？还是因为单行道数据标反了？

### 3. 地图匹配率 (Match Rate)
用真实的 GPS 轨迹跑 `match` 接口。
*   如果匹配率低，说明 OSM 路网缺失，或者 Profile 的容差设置太小。

### 4. 违章检查 (Violation Check)
编写脚本，验证生成的 GeoJSON 是否穿过了 `highway=pedestrian` (步行街) 或 `oneway=-1` 的路段。

---

## 7.8 本章小结

*   **没有银弹**：OSRM 是跑车（快但不可定制），GraphHopper 是 SUV（通用且舒适），Valhalla 是变形金刚（结构复杂但无所不能）。
*   **Profile 即法律**：路由引擎的行为完全由 Profile 定义。Tag 输入，Weight 是输出。
*   **成本不仅仅是时间**：$Cost = Time + Penalty$。通过 Penalty 我们可以控制“偏好”，而不仅仅是求快。
*   **预处理的代价**：越快的查询通常意味着越慢的预处理（OSRM CH）。在设计系统时，要考虑数据更新频率（每天？每小时？）。

---

## 7.9 练习题

### 基础题
1.  **权重计算**：假设某路段长度 1km，限速 60km/h。如果设置 `turn_cost = 0`，计算其 Base Cost（秒）。如果此时 Profile 将该类道路的 `rate` 设为 0.8（稍微不推荐），Cost 会变成多少？
2.  **引擎差异**：为什么说 OSRM 的 CH 算法不适合做“实时躲避拥堵”的导航？（提示：思考权重的固化）。
3.  **Tag 理解**：`highway=service` 和 `highway=residential` 在一般汽车导航 Profile 中，谁的优先级应该更低？为什么？

### 挑战题
4.  **U-Turn 陷阱**：在双向六车道的主干道上，OSM 通常将其画为两条独立的 `oneway` 线。如果你的路径规划在路段中间让你“掉头”，这是为什么？应该如何通过 Graph 模型来禁止这种行为？
5.  **等时圈原理**：Valhalla 生成的 Isochrone（等时圈）边缘通常比基于网格（Grid-based）的方法更平滑，它是基于什么图论算法实现的？
6.  **架构设计**：你需要为全国的外卖骑手设计导航。考虑到电动车可以走部分人行道、必须避开天桥（推行困难）、且需要极高的并发（百万级订单/分钟）。你会如何组合使用这些引擎？

<details>
<summary>点击查看练习题提示 (Hints)</summary>

1.  *Hint*: 基础时间 = 60s。Cost = Time / Rate。Rate < 1 会导致 Cost 变大（即引擎认为这条路“更长”了）。
2.  *Hint*: CH 预处理时把所有最短路径都算死并压缩了。一旦改权重，整个压缩结构就失效了，必须重算。
3.  *Hint*: `service` 通常是停车场内部道路或小巷，仅仅是“能走”，但绝不应该作为“通过性道路。
4.  *Hint*: 引擎可能认为那是一个 Node，且没有禁止转向限制。需要在 Profile 中检测 `via_node` 的连接角度，或者依赖 Relation 里的 `restriction=no_u_turn`。
5.  *Hint*: Valhalla 使用 SPT (Shortest Path Tree) 扩展，然后对边缘点进行多边形化 (Polygonization)，而不是简单的栅格着色。
6.  *Hint*: 极高并发指向 OSRM。但特殊规则（避开天桥）需要定制 Profile。可以考虑用 OSRM (MLD) 跑定制的 E-Bike Profile，或者为了更细致的避让逻辑使用 GraphHopper 的定制实例。

</details>

---

## 7.10 常见陷阱与错误 (Gotchas)

### 1. 吸附（Snapping）导致的“瞬移”
*   **场景**：用户在高架桥下的辅路，GPS 漂移到了高架桥上。
*   **问题**：导航引擎默认将起点吸附到距离最近的几何边上（可能是高架）。结果用户明明在地面，导航却让他“沿高架行驶”。
*   **对策**：
    *   检查 `snapped_distance`，过大则报警。
    *   利用 `hint` 或 `bearings`（方向）参数过滤候选边。如果用户朝向与高架流向垂直，就不该吸附上去。

### 2. 小区“孤岛” (Gated Communities)
*   **现象**：OSM 上很多小区是封闭的，大门节点标记了 `access=private` 或 `barrier=gate`。
*   **问题**：如果终点在小区里，且 Profile 设置了严格的 `private` 禁行，路由会失败，或者只导航到门口。
*   **对策**：设计“最后100米”逻辑。允许在终点附近的一定半径内“打破” `access=private` 的限制（Destinaton Access）。

### 3. 渡轮与跨海 (Ferries)
*   **现象**：路线突然跨越了大海，或者绕了地球一圈也不走直线距离仅 5km 的跨海渡轮。
*   **原因**：`route=ferry` 的路段通常在 OSM 中也是线，但需要特殊的 `mode` 支持。如果 Profile 没配置渡轮速度，权重可能是无穷大。
*   **调试**：检查 Profile 中是否启用了 `ferry`，并给它设置了合理的平均速度。

### 4. 错误的 `oneway` 导致绕路
*   **现象**：明明可以直行，导航非要绕一圈。
*   **原因**：最常见的 OSM 数据错误——单行道标反了，或者一段双向道误标为单向。
*   **技巧**：在开发环境中，使用路由引擎的 **Debug UI**（如 OSRM Frontend），它能显示底层的 Graph 连通性。如果看到一条线是红色的（不可通行）或者是单向箭头，哪怕地图渲染看起来是路，引擎也走不过去。
