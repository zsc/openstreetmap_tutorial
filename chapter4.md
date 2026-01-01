# 第4章｜PostGIS 与空间索引（地理查询的物理学）

## 4.1 开篇段落：PostGIS 是 Nominatim 的心脏

在构建 OSM 导航栈时，许多开发者容易将数据库视为一个“存数据的黑盒”，认为只要导入了 OSM 数据，查询自然就会快。这是一个危险的误区。Nominatim 和 OSRM 的底层逻辑截然不同：OSRM 将路网加载到内存中构建图（Graph），而 Nominatim（以及大多数自定义 POI 搜索服务）完全依赖 **PostgreSQL + PostGIS** 进行磁盘上的几何运算。

当你的 Agent 试图调用工具查找“距离当前位置最近的充电站”时，PostGIS 内部发生的不是简单的查表，而是复杂的计算几何过程。如果不懂 **SRID**、**GiST 索引分裂机制**以及**执行计划器（Planner）的脾气**，你的服务很容易在面对“北京周边”这种高密度数据查询时，从 20ms 劣化到 5s 甚至超时。

本章的学习目标：
1.  **彻底理解坐标系（SRID）**：为什么 4326 和 3857 混用会导致灾难，以及何时该用哪一个。
2.  **解剖 GiST 索引**：不仅知道它存了 BBOX，还要知道它是如何生长、分裂和腐烂的。
3.  **掌握空间查询的两步法**：Filter（粗筛）与 Refine（精筛）的底层逻辑。
4.  **学会读懂 Explain**：在地理查询中，什么样的执行计划意味着“危险”。

---

## 4.2 文字论述

### 4.2.1 核心世界观：SRID、投影与变形

PostGIS 中的所有几何体必须依附于一个 **SRID (Spatial Reference System Identifier)**。在导航领域，你必须形成下意识的“双轨制”思维。

#### 1. 存储标准：EPSG:4326 (WGS 84)
*   **物理本质**：这是一个椭球体模型。坐标是 `(经度, 纬度)`，单位是“度”。
*   **OSM 的原生形态**：OpenStreetMap 数据库中的 node 也就是存的这个。
*   **陷阱**：“度”不是长度单位。在赤道，1度 ≈ 111km；在瑞典，1度可能只有 50km。
*   **应用场景**：**数据存储**、**人机交互**（输入输出）、**LLM Toolcall 参数**。

#### 2. 计算/渲染标准：EPSG:3857 (Web Mercator)
*   **物理本质**：这是一个将地球强行展开的平面投影。坐标是 `(X, Y)`，单位是“米”。
*   **陷阱**：**面积与距离随纬度剧烈变形**。格陵兰岛在地图上看起来跟非洲一样大，实际小得多。这意味着在 3857 坐标系下直接画圈搜索（Buffer），在高纬度地区会覆盖比预期大得多的真实面积。
*   **应用场景**：**切片地图（Slippy Map）显示**、**局部范围内的简单欧氏距离计算**。

**Rule-of-Thumb经验法则）**：
> **“存储用 4326，计算转 Geography，渲染切片才用 3857”**
> 
> *   不要在数据库里存 3857，每次转换会有精度损失。
> *   不要试图在 4326 上用勾股定理算距离。
> *   如果你需要“搜索半径 5km”，请使用 PostGIS 的 `Geography` 类型自动处理投影。

### 4.2.2 数据类型抉择：Geometry vs Geography

这是初学者最容易纠结的地方。Nominatim 选择了 **Geometry**，为什么？

| 特性 | Geometry (几何) | Geography (地理) |
| :--- | :--- | :--- |
| **数学模型** | 笛卡尔平面 (X, Y) | 球体/椭球体 (Lat, Lon) |
| **计算复杂度** | 低 (CPU 友好) | 高 (涉及大量三角函数) |
| **跨越 180°经线** | 需要特殊处理 (Shift/Wrap) | 自动处理 (原生支持) |
| **函数支持** | 全面 | 有限 (很多函数不支持) |
| **索引效率** | 极高 | 略低 |

**Nominatim 的策略分析**：
Nominatim 甚至 OSRM 的后端数据库，通常使用 **Geometry (EPSG:4326)** 存储。
*   **原因**：OSM 数据量极大（数百亿行）。Geography 的计算开销太昂贵。
*   **妥协**：当需要计算距离时，通常使用 `ST_Distance(geom::geography, ...)` 进行即时转换，或者在小范围内假设平面进行近似计算。

### 4.2.3 动力学建模：GiST 索引的内部机制

普通的 B-Tree 索引（用于 ID、数字）是一维的，易于排序。地理数据是二维甚至多维的，无法简单排序。PostGIS 使用 **GiST (Generalized Search Tree)**，其核心实现通常是 **R-Tree (Region Tree)**。

#### 1. R-Tree 的核心：包围盒嵌套 (Nesting Bounding Boxes)
索引不存储复杂的街道形状，只存储街道的 **MBR (Minimum Bounding Rectangle，最小外包矩形)**。

```text
    层级结构图解 (R-Tree)
    
    [ 根节点 (覆盖整个城市) ---------------------------------------]
       |
       +--- [ 子节点 A (海淀区) ]           [ 子节点 B (朝阳区) ]
       |       |                               |
       |       +--- [ 叶节点 1 (某街道) ]      +--- [ 叶节点 3 ]
       |       |       (实际数据指针)             (实际数据指针)
       |       |
       |       +--- [ 叶节点 2 (某公园) ]
       |
       +--- [ 子节点 C (重叠区域) ] ...
```

#### 2. 索引的动力学特征
*   **重叠 (Overlap) 是性能杀手**：与 B-Tree 不同，R-Tree 的分支是可以空间重叠的。如果一条路横跨了海淀和朝阳，查询这两个区域都可能遍历到这条路。如果索引构建得不好（重叠率高），查询就会退化为全表扫描。
*   **分裂 (Split)**：当插入新数据，某个节点的 BBOX 满了，索引需要分裂。PostGIS 采用多种算法（如 Guttman, Linear, Quadratic）来决定怎么切分，目的是尽量减少重叠，让 BBOX 尽可能“方”且“小”。
*   **膨胀 (Bloat)**：OSM 数据频繁更新（Diff update）会导致索引产生大量“死节点”。GiST 索引对膨胀非常敏感。**定期 `REINDEX` 对于维持航 API 的性能至关重要。**

### 4.2.4 查询执行流：Filter & Refine（粗筛与精筛）

当你发送一个 SQL：`SELECT * FROM pois WHERE ST_DWithin(geom, point, 1000)`，PostGIS 实际上在做两件事：

**第一步：索引扫描 (Index Scan) —— 粗筛**
*   **操作符**：`&&` (Bounding Box Overlap)。
*   **逻辑**：只比较矩形。只要目标的 BBOX 和索引里的 BBOX 哪怕碰了一点边，就算命中。
*   **速度**：极快。
*   **结果**：一组“候选者 (Candidates)”。其中包含很多“假阳性”结果（例如：点在矩形角里，但不在圆形半径内）。

**第二步：精确计算 (Heap Fetch & Recheck) —— 精筛**
*   **函数**：`ST_Distance` (真实几何计算)。
*   **逻辑**：去磁盘读取真实的 Geometry 数据（可能几千个顶点的多边形），解压，算数学距离。
*   **速度**：慢，CPU 密集。
*   **结果**：最终精准结果。

**常见陷阱**：
如果你写 `WHERE ST_Distance(geom, p) < 100`，PostGIS 默认**不会**使用索引（除非是新版本且有特殊配置），因为它是一个函数调用。
**必须使用** `ST_DWithin`，因为它被设计为自动展开成 `geom && BBOX(p, 100) AND ST_Distance(geom, p) < 100`。

### 4.2.5 聚簇 (Clustering)：让物理存储服从地理分布

在 Nominatim 的 `placex` 表中，数据默认可能是按“插入时间”排序的。这意味着：
*   第 1 行是北京的数据。
*   第 2 行可能是上海的数据（因为是下一秒同步进来的）。

当你查询“北京的所有加油站”时，硬盘磁头需要在磁盘的不同位置疯狂跳跃（Random I/O），这是机械硬盘时代的噩梦，在 SSD 时代依然消耗 IOPS。

**CLUSTER 命令**：
`CLUSTER placex USING idx_placex_geometry;`
这个命令会重写整张表，**强制把地理位置相近的数据，在物理硬盘上也挨在一起存**。这能极大地提升“范围搜索”的性能，因为一次 I/O 读取的 Page 包含了更多相关数据。

---

## 4.3 本章小结

1.  **SRID 隔离**：严格区分存储 (4326) 和 计算 (Geography/Projected)。不要混用。
2.  **BBOX 决定生死**：索引查的是矩形。对于斜着的长条形物体（如对角线道路），索引效率天然较低。
3.  **函数即索引**：必须使用支持索引操作符的函数（如 `ST_DWithin`, `ST_Intersects`），避免裸写数学计算。
4.  **维护成本**：GiST 索引比 B-Tree 更容易“脏”。高频更新的 OSM 数据库必须有自动化的 VACUUM 和 REINDEX 策略。
5.  **IO 优化**：空间查询本质是 IO 密集型。利用 `CLUSTER` 进行物理排序是提升生产环境性能的“核武器”。

---

## 4.4 练习题

### 基础题

**Q1. 投影的直觉**
在 Web Mercator (3857) 投影下，为什么在俄罗斯北部计算的 `ST_Buffer(point, 1000)`（1000米缓冲区）画在地图上是个完美的圆，而在 WGS84 (4326) 下直接对经纬度加减画出来的“圆”，在高纬度地区看起来像个压扁椭圆？
<details>
<summary>查看提示</summary>
思考经线收敛。在极地附近，1经度的距离远小于1纬度的距离。如果在 lat/lon 轴上取相同数值半径，实际地面距离在东西方向上会短得多。
</details>

**Q2. 索引失效分析**
你有一个包含 1000 万个 POI 的表，建了 GiST 索引。执行 `SELECT * FROM pois WHERE NOT ST_Intersects(geom, region)`（查询所有**不在**区域内的点）。请问这个查询会走索引吗？为什么？
<details>
<summary>查看提示</summary>
几乎不会。索引擅长“包含”和“重叠”。“不包含”意味着可能在地球的任何其他地方，潜在结果集几乎是全表。数据库优化器会选择 Seq Scan（全表扫描）。
</details>

**Q3. 简单的 KNN**
在 PostGIS 中，操作符 `<->` 代表“中心距离排序”。以下 SQL：`SELECT * FROM shop ORDER BY geom <-> my_point LIMIT 10;` 为什么比 `ORDER BY ST_Distance(geom, my_point) LIMIT 10` 快几个数量级？
<details>
<summary>查看提示</summary>
前者是 **Index-Assisted KNN**。PostGIS 会利用 R-Tree 结构，从离点最近的索引分支开始遍历，找到 10 个就停止。后者必须计算全表所有点到目标的距离，排序后取前 10。
</details>

### 挑战题

**Q4. 太平洋的幽灵 (Dateline Wrapping)**
你的导航服务支持全球数据。用户在斐济（经度 179°）搜索附近的点，结果漏掉了东边仅仅 50km 外（经度 -179°）的一个岛屿。这是为什么？如何从 SQL 层面解决（不改动数据的前提下）？
<details>
<summary>查看提示</summary>
数学上 179 到 -179 的差是 358，PostGIS 认为它们隔了整个地球。
解决思路：
1. 使用 `Geography` 类型（原生支持）。
2. 在 Geometry 下，将查询框切分为两部分：`WHERE (geom && box_part_1) OR (geom && box_part_2)`。
3. 或者使用 `ST_Shift_Longitude` 将所有负经度平移到 180-360 范围再比较。
</details>

**Q5. 大对象 (Large Geometry) 陷阱**
Nominatim 入了某个国家的行政边界（Relation），由 50,000 个点组成的多边形。每次判断“当前坐标在哪个国家”时，CPU 占用极高。除了简化多边形，有什么**存储结构上**的优化策略？
<details>
<summary>查看提示</summary>
**Subdivision (ST_Subdivide)**。
将那个拥有 5万个点的大多边形，切碎成 500 个拥有 100 个点的小多边形，存入辅助表。
查询时，点只需落入某个小多边形的 BBOX，随后进行的精确计算（Recheck）只需针对那 100 个点，而不是 5万个点。这是极大的性能提升。
</details>

**Q6. 解释计划分析**
在 `EXPLAIN (ANALYZE, BUFFERS)` 的输出中，看到了 `Heap Fetches: 0` 和 `Heap Blocks: lossy`。这通常发生在 Bitmap Heap Scan 中。这代表了什么物理现象？此时 `work_mem` 可能出现了什么问题？
<details>
<summary>查看提示</summary>
`Lossy` 意味着分配给 Bitmap Scan 的内存（`work_mem`）不够用了。
数据库无法精确记录“哪行”命中了，只能退化为记录“哪一页（Page）”命中了。
结果就是数据库必须读取整个页面并重新检查每一行（Recheck），导致 CPU 和 IO 浪费。增加 `work_mem` 可能解决此问题。
</details>

---

## 4.5 常见陷阱与错误 (Gotchas)

### 1. TOAST 表引发的性能抖动
PostgreSQL 的 Page 大小默认是 8KB。如果一个 OSM Way（比如长距离高速公路）的几何数据超过 2KB，它会被压缩并切片存到 **TOAST** 辅助表中。
*   **现象**：查询大多数点很快，一旦查到复杂的行政区或长路，耗时突然增加 10 倍。
*   **对策**：在 `SELECT` 列表中不要无脑 `SELECT *`。如果你只需要 ID 和名字，就不要把 `geometry` 列取出来，避免触发 TOAST 解压和读取。

### 2. 坐标顺序 (Lat/Lon) 的永恒诅咒
*   **标准**：PostGIS 的 `ST_MakePoint(x, y)` 接受的是 `(Lon, Lat)`。
*   **错误习惯**：Google Maps API 和很多前端库习惯用 `Lat, Lon`。
*   **后果**点会被投射到错误的位置。比如 `(39, 116)` 是北京附近，但 `ST_MakePoint(39, 116)` 指的是北纬 116 度（不存在）或仅仅是翻转了 X/Y。
*   **调试**：始终用 GeoJSON 验证你的输入点位置。

### 3. `ST_Intersects` 并不总是你想要的
在导航场景中，用户点击地图上的路，往往点不到路的正中心，而是点在路边。
*   使用 `ST_Intersects(road, point)` 通常返回 False。
*   **应该用**：`ST_DWithin(road, point, tolerance)`。给用户手指的抖动留出几米的容错空间。

### 4. 忽略了 `VACUUM ANALYZE`
大量导入或更新 OSM 数据后，如果没有运行 `ANALYZE`，PostgreSQL 的统计信息是旧的。
*   **后果**：规划器可能会认为表是空的，从而选择全表扫描而不是索引扫描。
*   **操作**：在任何批量数据操作后，手动执行一次 `ANALYZE table_name;`。

### 5. 滥用 Geometry Collection
OSM 有些 Relation 导出的数据是 `GeometryCollection`（混合了点、、面）。
*   **问题**：很多标准函数不支持 Collection，或者索引效率极低。
*   **对策**：在入库清洗阶段（ETL），使用 `ST_CollectionExtract` 强行将其拆分为单一类型的几何体。
