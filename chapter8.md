# 第8章｜OSRM 实战：从 OSM 数据到路由服务（含 Match/Table/Trip）

## 8.1 开篇：从坐标点到“可行驶网络”

在上一部分（Nominatim），我们解决了“这是哪里”的问题。现在，我们进入导航系统的核心——解决“怎么去”的问题。

OSRM (Open Source Routing Machine) 是高性能路由引擎的工业级标准。许多初学者认为 OSRM 只是一个“加载数据就能跑”的黑盒，但在构建商业级导航 API 时，这种理解是远远不够的。你需要控制车辆能不能走这条路（权限）、走得有多快（速度模型）、转弯有多难（惩罚模型），以及如何处理 GPS 漂移（匹配算法）。

**本章学习目标：**
1.  **解构数据流**：理解 Extract/Partition/Customize 三步走的必要性与内部产物。
2.  **掌握 Lua 动力学建模**：学会阅读和修改 Lua Profile，定义不同车型的通行逻辑（不仅是速度，更是权重）。
3.  **算法决策（CH vs MLD）**：深入理解两种加速算法的原理、内存开销与适用场景。
4.  **精通核心 API 机制**：不仅会调用，更理解 `/match` 的 HMM 模型和 `/table` 的计算边界。
5.  **资源与性能**：理解内存映射（mmap）、共享内存与并发模型。

---

## 8.2 OSRM 架构原理：为什么它这么快？

OSRM 之所以能在大洲级路网上实现毫秒级响应，是因为它**不使用**传统的 A* 算法直接在原始路网上搜索，而是使用**预计算（Pre-computation）**技术。

### 8.2.1 数据处理流水线 (Pipeline)

这一流程将 OSM 的原始 XML/PBF 数据转化为针对路由算法优化的二进制图文件。

```
[ OSM 原始数据 (.osm.pbf) ]
         |
         | (1) osrm-extract (提取器)
         | 输入: .pbf + profile.lua
         v
[ 图拓扑 (.osrm) ] ---> [ 节点、边、以及基础权重 ]
         |
         +---------------------------------------+
         | 决策分支: 选择算法后端                |
         |                                       |
         v                                       v
[ 路径 A: Contraction Hierarchies (CH) ]   [ 路径 B: Multi-Level Dijkstra (MLD) ]
   (极致速度，静态权重)                       (灵活更新，动态权重)
         |                                       |
         | (2a) osrm-contract                    | (2b) osrm-partition
         v                                       |      (图切分)
[ 收缩层级图 (.osrm.hsgr) ]                      v
         |                                 [ 分区文件 (.osrm.partition) ]
         |                                       |
         |                                       | (2c) osrm-customize
         |                                       |      (权重应用)
         v                                       v
   [ 最终索引文件 ] <---------------------- [ 定制化图数据 (.osrm.cell_metrics) ]
         |
         v
   (3) osrm-routed (HTTP Server)
   (加载至 RAM / Shared Memory)
```

### 8.2.2 关键步骤详解

1.  **Extract (提取)**：
    *   **解析器**：遍历 OSM 的 Nodes, Ways, Relations。
    *   **脚本执行**：对每个 Way 执行 Lua 脚本，决定该路段是否保留、它是单行还是双行、它的基础速度是多少。
    *   **流式处理**：这是最耗 CPU 和 IO 的阶段，通常需要处理数小时。

2.  **Contract (CH 模式)**：
    *   **原理**：在图中不断移除“不重要”的节点，并添加“快捷边（Shortcut）”来保持连通性。
    *   **例子**：从北京到上海，中间经过成千上万个路口。CH 算法会预先计算出“北京->...->上海”的长距离逻辑边。查询时，算法只在这些快捷边上跳跃，大大减少了搜索空间。
    *   **代价**：一旦图被“收缩”，权重就固化了。如果一条快捷边代表了 100 公里长的路，其中哪怕 1 公里堵车，你也无法只修改那 1 公里的权重，必须重新运行 Contract。

3.  **Partition + Customize (MLD 模式)**：
    *   **Partition**：使用惯性流（Inertial Flow）等算法将地图切割成若干个**Cell（单元格）**。
    *   **Customize**：计算 Cell 内部以及 Cell 边界之间的权重。
    *   **优势**：当你需要更新路况（如：全城限速调整、实时拥堵）时，只需要重新运行毫秒级的 `customize` 阶段，而无需重新切分地图。

---

## 8.3 动力学建模：Lua Profile 实战

Profile 是 OSRM 的灵魂。它不是简单的配置文件，而是**代码**。OSRM 通过嵌入 Lua 解释器，让开发者拥有极大的自由度来定义“路权”。

### 8.3.1 核心函数与生命周期

处理一个 Way 时，Lua 脚本会按顺序执行以下逻辑：

1.  **`process_node(profile, node, result)`**：
    *   检查节点属性。
    *   *用途*：识别 `barrier=gate`（大门）、`highway=traffic_signals`（红绿灯）。
    *   *产出*：是否可以通过，通过的延时惩罚。

2.  **`process_way(profile, way, result)`**：
    *   **最核心的函数**。决定路段属性。
    *   *输入*：OSM tags (key-value)。
    *   *输出*：
        *   `result.forward_mode` / `result.backward_mode`：能走吗？（如：`mode.driving`）
        *   `result.forward_speed` / `result.backward_speed`：速度（km/h）。
        *   `result.duration`：直接指定通过时长（覆盖速度计算）。
    *   *典型逻辑*：
        ```lua
        -- 伪代码示例
        local highway = way:get_value_by_key("highway")
        if highway == "motorway" then
            result.forward_speed = 100
            if profile.vehicle_type == "bicycle" then
                 result.forward_mode = mode.inaccessible -- 自行车禁上高速
            end
        elseif highway == "track" then
            result.forward_speed = 10 -- 乡间土路
        end
        ```

3.  **`process_turn(profile, turn)`**：
    *   定义转弯成本。
    *   *输入*：进入边的角度、离开边的角度、是否是 U-turn。
    *   *逻辑*：
        *   直行（0°）：成本 0。
        *   右转（90°）：成本小。
        *   左转（-90°）：成本大（假设右侧通行，需等待）。
        *   U-turn（180°）：成本极大，除非没有其他路可走。

### 8.3.2 速度 vs 权重 (Speed vs Rate)

初学者常犯的错误是混淆“物理速度”和“路由权重”。

*   **物理速度 (Speed)**：用于计算 ETA（预计到达时间）。即 `距离 / 速度 = 时间`。
*   **路由权重 (Rate/Weight)**：用于 Dijkstra 寻路算法的代价值。

**Rule of Thumb (经验法则)**：
如果你想让导航**避开**某些路（例如：狭窄的胡同），但又不想完全禁止（因为可能终点就在胡同里），你应该：
1.  保持 `speed` 为真实值（例如 15km/h），保证 ETA 准确。
2.  引入 `rate` 概念，在内部计算 cost 时人为增加阻力。
3.  或者简单粗暴地：在 Lua 中将该路段 `speed` 设得极低（如 5km/h），这样算法会认为走这条路太慢而绕行，但副作用是计算出的 ETA 会偏大。

---

## 8.4 核心 API 设计深度解析

OSRM 暴露的 API 不仅是端点，它们代表了不同的图算法应用。

### 1. `/route`：不仅是寻路

*   **Phantom Nodes (幻影节点)**：
    *   用户输入的坐标 `(lon, lat)` 通常不会精确落在路网的节点上。
    *   OSRM 会在最近的边（Edge）上创建一个虚拟节点，将原始边切分为两段。这个过程叫 **Snapping**。
    *   *Gotcha*：如果你的起点在立交桥下，Snapping 可能会吸附到桥面上。可以通过 `bearings`（朝向）参数来强制指定吸附方向。
*   **几何压缩**：
    *   默认返回 `polyline`（Google 编码算法，精度 1e-5）或 `polyline6`（精度 1e-6）。这是一个压缩字符串，大大减小了 JSON 体积。前端（Leaflet/Mapbox GL）需要解码。

### 2. `/match`：隐马尔可夫模型 (HMM)

这是最复杂的 API，用于将充满噪声的 GPS 轨迹“纠偏”到路网上。

*   **问题模型**：
    给定观测序列 $O_1, O_2, ...$ (GPS点)，寻找最可能的隐藏状态序列 $S_1, S_2, ...$ (路段位置)。
*   **两大核心概率**：
    1.  **Emission Probability (发射概率)**：GPS 点距离路段越近，概率越高（高斯分布）。
    2.  **Transition Probability (转移概率)**：
        *   从点 A 的候选路段走到点 B 的候选路段，其**网络距离**与**直线距离**（或时间差下的最大速度距离）越接近，概率越高。
        *   如果两点直线距离 10米，但路网距离需绕行 5公里，则转移概率极低。
*   **Viterbi 算法**：
    OSRM 使用 Viterbi 算法在所有可能的路径组合中找到**全局概率最大**的一条路径。这就是为什么有时候中间的个点的漂移会被前后的点“拉回来”。

### 3. `/table`：矩阵计算的边界

*   **算法**：对于 CH，使用双向搜索；对于 MLD，使用图分割加速。
*   **性能瓶颈**：
    *   计算量是 $O(N \times M)$。
    *   如果 $N=1000, M=1000$，就是 100 万次路径计算。即便是 OSRM 也需要时间。
    *   *设计建议*：如果你需要计算大规模矩阵（如全城物流优化），不要一次性发 1000x1000。应拆分为多个小批次（如 100x100），或在本地部署 OSRM 并通过 C++ 绑定直接调用，避开 HTTP 开销。

---

## 8.5 生产级部署与运维

### 8.5.1 内存管理

OSRM 是典型的**内存密集型**应用。

*   **加载机制**：
    *   OSRM 使用 `mmap` (Memory Map) 将巨大的 `.osrm` 文件映射到虚拟内存。
    *   **优点**：操作系统负责分页调度，未访问的数据不占物理 RAM。
    *   **缺点**：初次访问（冷启动）会产生大量 Page Fault，导致响应慢。
*   **预 (Warmup)**：
    *   在服务接入流量前，建议运行一个脚本，随机请求 `/route` 覆盖主要区域，强制 OS 将热数据加载到物理内存（RSS）中。

### 8.5.2 并发与多线程

*   `osrm-routed` 是基于 libuv 的异步 I/O，但路径计算本身是 CPU 密集的。
*   **线程模型**：它维护一个线程池。并发请求数超过线程数时，请求会排队。
*   **配置**：生产环境务必显式设置 `-t` (threads) 参数，通常设为 `CPU 核心数 - 1`，留一个核给操作系统和网络 I/O。

### 8.5.3 算法选型决策矩阵

| 特性 | Contraction Hierarchies (CH) | Multi-Level Dijkstra (MLD) |
| :--- | :--- | :--- |
| **预处理速度** | 慢 (需要深度收缩) | 快 (只需切分) |
| **查询速度** | 极快 (毫秒级) | 快 (通常是 CH 的 2-3 倍耗时) |
| **路况更新** | **不支持** (需完全重跑 Pipeline) | **支持** (仅需 `osrm-customize`) |
| **内存占用** | 较低 (图结构精简) | 较高 (需存储多层级 Cell 权重) |
| **适用场景** | 基础导航、距离矩阵、无需实时路况 | 网约车、实时物流、避让拥堵 |

---

## 8.6 本章小结

1.  **数据即代码**：OSM 数据本身是不够的，必须通过 Lua Profile 赋予其物理含义。
2.  **预处理换时间**：OSRM 的高性能源于 Extract/Contract 阶段的繁重计算。
3.  **匹配非查找**：`/match` 使用概率模型还原轨迹，比单纯的“最近点查找”要健壮得多。
4.  **架构权衡**：在 CH 的极致速度和 MLD 的动态灵活性之间，必须根据业务需求（是否需要实时路况）做二选一。

---

## 8.7 练习题

### 基础题

1.  **Profile 修改**：在默认的 `car.lua` 中，如果我想让导航彻底避开所有 `highway=track`（农林土路），应该修改哪个函数？将 `forward_mode` 设为什么？
    <details><summary>Hint</summary>关注 `process_way` 函数。</details>
    <details><summary>答案</summary>修改 `process_way` 函数。找到处理 `highway` 标签的逻辑，将 `track` 对应的 `mode` 设为 `mode.inaccessible`。</details>

2.  **API 参数**：使用 `/route` 接口时，如果我只想要距离和时间，不需要具体的路线坐标点（为了节省带宽），应该传什么参数？
    <details><summary>Hint</summary>overview 参数。</details>
    <details><summary>答案</summary>`overview=false`。这会禁止返回 `geometry` 字段。</details>

3.  **算法识别**：你下载了一个别人的 `.osrm` 数据包，发现里面包含 `.osrm.partition` 和 `.osrm.cell_metrics` 文件。请问这是为哪种算法准备的数据？
    <details><summary>Hint</summary>Partition 是分区的概念。</details>
    <details><summary>答案</summary>MLD (Multi-Level Dijkstra)。CH 算法会生成 `.osrm.hsgr` 文件。</details>

### 挑战题

4.  **幽灵直行问题**：你在一个十字路口，直行方向是绿灯，但 OSRM 规划的路线却让你右转再掉头回来（P-turn），而不是直接左转。检查 OSM 数据发现路口是连通的。请从 Lua Profile 的 `process_turn` 角度分析可能原因。
    <details><summary>Hint</summary>左转惩罚成本过高。</details>
    <details><summary>答案</summary>可能是 Profile 中对“左转”设置了极高的惩罚时间（Turn Penalty），例如 60秒。而“右转+掉头+直行”的总计算成本（行驶时间 + 较小的转弯惩罚）小于 60秒。这在交通流优化中是合理的（避免阻断直行流），但如果是误设，需要调整 Profile 中的左转权重。</details>

5.  **Match 漂移调试**：一辆车在立交桥下的辅路行驶，GPS 信号显示在主路上。`/match` 接口强行把轨迹匹配到了主路上（虽然你设置了很小的半径）。这是为什么？如何利用 `radiuses` 以外的参数来修正？
    <details><summary>Hint</summary>不仅仅是位置，还有方向。</details>
    <details><summary>答案</summary>HMM 算法不仅考虑距离，还考虑拓扑连通性。如果辅路和主平行且靠得很近，仅靠坐标很难区分。应使用 `bearings` 参数，传入 GPS 记录的行进方向（如 90度）。主路和辅路虽然平行，但在进出口附近或弯道处可能有细微的方向差异，或者通过 `timestamps` 的速度计算（主路速度快，辅路速度慢）来辅助算法区分。</details>

6.  **Table 性能优化**：你需要计算 10,000 个用户到 5 个门店的距离矩阵。直接请求 10000x5 会导致 URL 过长或超时。除了分批次请求，如果要在**一次计算中**利用 OSRM 的反向搜索特性优化，应该如何构造请求？（假设从 A 到 B 和 B 到 A 路径相同）
    <details><summary>Hint</summary>源与目标的概念。Table API 允许指定 sources 和 destinations。</details>
    <details><summary>答案</summary>Table API 支持 `sources` 和 `destinations` 参数。你可以将 5 个门店作为 sources，10,000 个用户作为 destinations（或者反过来，取决于 API 限制）。计算 5x10000 的阵比 10000x5 通常在内部处理上并无本质区别，但关键在于利用非对称参数，避免计算完整的 10005x10005 矩阵。注意：如果是单行道多的城市，A->B 不等于 B->A，必须严格按照业务方向请求。</details>

---

## 8.8 常见陷阱与错误 (Gotchas)

1.  **Lua Profile 更改不生效**：
    *   *现象*：修改了 `car.lua`，重启 `osrm-routed`，路线没变。
    *   *原因*：Profile 的逻辑是在 `osrm-extract` 阶段“烧录”进图数据的。
    *   *解决*：修改 Lua 后，必须**从头运行** `extract` -> `partition/contract` -> `customize` 整个流程。

2.  **Shared Memory 锁死**：
    *   *现象*：使用 `osrm-datastore` 更新数据时报错，或者旧进程无法退出。
    *   *原因*：OSRM 在共享内存中加了锁。
    *   *解决*：如果进程非正常死亡，可能留下僵尸锁。使用 `ipcs -m` 查看并用 `ipcrm` 清理，或者重启机器。

3.  **Coordinates Precision (坐标精度)**：
    *   *现象*：路径看起来呈锯齿状，或者匹配不准。
    *   *原因*：在 Python 或 JS 中处理坐标时，不小心将浮点数截断了（例如保留了 3 位小数）。
    *   *Rule*：经纬度至少保留 **5位小数**（约 1米精度），推荐 **6位**（约 0.1米）。

4.  **忽略了 `continue_straight`**：
    *   *现象*：在只有一条路的中间点重新规划，路线居然让车掉头回去。
    *   *原因*：OSRM 默认 `/route` 的起点是无方向的，可能会为了找“更近的起点”而向后搜索。
    *   *解决*：在做分段导航时，设置 `continue_straight=true` 或指定 `bearings`。

5.  **Docker 卷权限**：
    *   *现象*：Docker 容器内无法写入 `.osrm` 文件。
    *   *原因*：宿主机映射目录的 Owner 不是容器内的用户（通常是 root 或 osrm 用户）。
    *   *解决*：确保 `chmod/chown` 映射目录，允许容器写入。
