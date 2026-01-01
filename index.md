# Nominatim & OpenStreetMap（OSM）导航 API 与 Navigation MCP 中文教程

> 面向两类读者：  
> 1) 想用 **OSM 数据自建导航能力**（地理编码/逆地理编码/POI 搜索/路径规划/地图匹配等），对外提供 **Navigation API**；  
> 2) 想把上述能力封装成 **LLM toolcall**（函数调用）与 **Agent** 可编排的 **Navigation MCP（Model Context Protocol）** 工具集，供智能体稳定、可控地调用。

---

## 本教程你将搭建的目标系统（概览）

- **数据层**：OSM Extract / Planet → 更新 Diff → 数据版本管理
- **地理查询层**：PostGIS +（可选）自建 OSM POI 表
- **地理编码层**：Nominatim（search / reverse / lookup）
- **路由层**：OSRM（为主）+（选学）GraphHopper / Valhalla
- **业务 API 层**：统一的 “Geocode → Disambiguate → Route → Instructions” 接口设计
- **LLM 工具层**：Tool Schema、错误模型、澄清策略、回退策略
- **MCP 层**：Navigation MCP Server（连接 Nominatim/OSRM/缓存/审计/限流）

---

## 全局约定（建议在第 1 章开始前先扫一遍）

- **坐标顺序**：  
  - Nominatim 常见输出为 `lat` / `lon` 字段（注意字段名）  
  - 路由引擎（例如 OSRM）常见请求坐标序列为 `lon,lat`（尤其是路径规划参数拼接时）  
  - 本教程会明确标注每一处坐标顺序，并给出统一的内部规范与转换函数
- **坐标系**：默认 WGS84 / EPSG:4326（经纬度）；涉及瓦片/渲染时再引入 EPSG:3857
- **单位**：距离 m / km；速度 km/h；时间 s / min
- **语言**：示例以中文输入为主，同时讨论多语言、别名、拼音等检索体验优化
- **合规**：贯穿全文强调 ODbL、署名（Attribution）、公开服务限速与反滥用

---

## 文件结构

- `index.md`（本文件）
- `chapter1.md`
- `chapter2.md`
- …
- `chapter12.md`

---

## 目录（12 章）

> 阅读建议：  
> - 只想快速自建可用服务：1 → 3 → 5 → 8 → 9 → 10  
> - 想做“对话式导航 + Agent”：在上述避免跳过 11、12  
> - 想把中文搜索做得更好：重点看 6、9、11

---

## 第一部分：OSM / Nominatim 基础与数据管道

### 第1章｜全景与快速上手（从 0 跑通一次 geocode + route）
文件：[`chapter1.md`](chapter1.md)

- 1.1 本教程范围：OSM 导航 API vs 地图渲染 vs 位置智能
- 1.2 OSM 导航栈全景：数据、搜索、路由、匹配、指令、瓦片
- 1.3 最小可运行架构（MVP）：Nominatim + OSRM + Facade API
- 1.4 开发/生产环境差异：单机、分层、多副本、冷/热数据
- 1.5 一次完整请求链路演示：自然语言地点 → 坐标 → 路线 → 文本指令
- 1.6 常见坑速查：坐标顺序、编码、语言、限速、超时、数据版本
- 1.7 本章练习：用 curl 跑通 “search → route → instructions”
- 1.8 本章产出：可复现的本地运行清单与验收用例

---

### 第2章｜OSM 数据模型与 ODbL 合规（你到底在用什么、需要怎么标）
文件：[`chapter2.md`](chapter2.md)

- 2.1 OSM 基本要素：Node / Way / Relation（为什么关系很重要）
- 2.2 Tags 体系：键值对、约定优于强制、社区演进
- 2.3 地址与行政区：`addr:*`、`place=*`、`boundary=administrative`
- 2.4 道路与通行属性：`highway=*`、`oneway=*`、`access=*`、`maxspeed=*`
- 2.5 ODbL 关键概念：数据库、衍生数据库、Produced Work（直觉误区澄清）
- 2.6 Attribution（署名）怎么做：API、Web、App、导出报告的不同方式
- 2.7 Share-Alike（共享相同方式）与你的“加工数据”边界
- 2.8 公共服务与反滥用：为何不建议直接打官方 Nominatim、如何自建
- 2.9 合规检查清单：上线前必须确认的 10 项事项

---

### 第3章｜OSM 数据获取、裁剪与增量更新（构建可持续的数据管道）
文件：[`chapter3.md`](chapter3.md)

- 3.1 数据来源选择：Planet、区域 Extract、镜像站点（策略与取舍）
- 3.2 裁剪方式：bbox / polygon（多形裁剪与跨海域边界问题）
- 3.3 工具链：osmium / osm2pgsql / osmosis（何时用哪个）
- 3.4 数据版本：快照、构建号、回滚点、可追溯元数据
- 3.5 增量更新：Replication diffs（分钟/小时/天）与延迟容忍
- 3.6 数据一致性校验：缺失关系、坏几何、异常 tag 的处理
- 3.7 任务编排：定时、队列、幂等、失败重试、告警
- 3.8 资源评估：CPU/内存/IO/磁盘（导入与更新的瓶颈在哪里）
- 3.9 本章产出：一条可自动化运行的“下载→裁剪→更新→发布”流水线草案

---

### 第4章｜PostGIS 与空间索引（让你的地理查询跑得快、跑得稳）
文件：[`chapter4.md`](chapter4.md)

- 4.1 为什么 Nominatim/导航服务几乎绕不开 PostGIS
- 4.2 SRID 与坐标系：4326 vs 3857（别在距离计算上踩坑）
- 4.3 geometry vs geography：精度、性能与适用场景
- 4.4 空间索引：GiST、SP-GiST（索引失效的常见原因）
- 4.5 常用空间函数速览：距离、包含、相交、缓冲区、最近点
- 4.6 OSM 导入模式概览：osm2pgsql slim/非 slim、schema 设计思路
- 4.7 查询优化：EXPLAIN、热点表、VACUUM、并发连接与连接池
- 4.8 权限与安全：最小权限、只读账号、SQL 注入面防护
- 4.9 本章产出：可复用的“空间表 + 索引 + 查询模板”清单

---

### 第5章｜Nominatim 部署与核心 API（search / reverse / lookup）
文件：[`chapter5.md`](chapter5.md)

- 5.1 Nominatim 解决什么问题：地理编码、逆地理编码、地址结构化
- 5.2 架构概览：导入、索引、查询、排名（ranking）与语言支持
- 5.3 部署方式：Docker / 裸机 / 集群化（适用场景对比）
- 5.4 导入流程：准备数据 → import → 索引 → 验证（性能与耗时要点）
- 5.5 API：`/search`（自由文本 vs 结构化参数）
- 5.6 API：`/reverse`（逆地理编码与“最近 POI/道路”的差别）
- 5.7 API：`/lookup`（OSM ID 批量查详情，适合缓存回填）
- 5.8 回字段解释：`class/type`、`importance`、`boundingbox`、`addressdetails`
- 5.9 限速与缓存：避免“雪崩式”打爆数据库
- 5.10 本章产出：可对外发布的 Nominatim 使用指南与调用样例集

---

### 第6章｜中文地址、POI 搜索与体验优化（让用户搜得到、搜得准）
文件：[`chapter6.md`](chapter6.md)

- 6.1 中文地址结构：省市区、街道门牌、园区/楼宇/单元（难点清单）
- 6.2 Nominatim 的语言机制：`accept-language` 与多语言 display_name
- 6.3 归一化：简繁、全半角、别名、括号、数字与楼栋格式
- 6.4 拼音与模糊匹配策略（以及什么时候不该做）
- 6.5 歧义消解：同名地点、多候选排序、最小澄清问题设计
- 6.6 POI 搜索：类目、热度、距离与行政区过滤的组合
- 6.7 质量评估：离线用例集、召回/准确、Top-K 命中与人工审查
- 6.8 本章产出：一套“中文输入 → 稳定候选列表”的检索管线方案

---

## 第二部：路由引擎与面向业务的导航 API

### 第7章｜路由引擎选型与路网建模（OSRM / GraphHopper / Valhalla 怎么选）
文件：[`chapter7.md`](chapter7.md)

- 7.1 导航能力清单：route、matrix、match、trip、isochrone、snapping
- 7.2 OSRM / GraphHopper / Valhalla：特性、生态、部署成本与取舍
- 7.3 Profile/Costing 概念：把 OSM tag 变成“可走/成本/限制”
- 7.4 关键道路属性映射：限速、单行、禁行、转向惩罚、收费路
- 7.5 多模式路由：驾车/步行/骑行（以及公共交通的边界与方案）
- 7.6 约束路由：限高/限重/危化/车辆类型（何时需要引擎支持）
- 7.7 动态因素：实时交通、封路、临时施工（可行路线与现实差距）
- 7.8 评测框架：避免“能跑就行”的错觉（成本、精度、稳定性）
- 7.9 本章产出：一份可落地的选型决策模板与试运行指标

---

### 第8章｜OSRM 实战：从 OSM 数据到路由服务（含 match/table/trip）
文件[`chapter8.md`](chapter8.md)

- 8.1 OSRM 工作流：extract → partition → customize（为什么不是一次就完）
- 8.2 Lua Profile 编写：速度、通行、道路分类、转向与惩罚
- 8.3 CH vs MLD：性能、灵活性、在线更新能力对比
- 8.4 HTTP API：`/route`（多点、替代路线、避让策略）
- 8.5 HTTP API：`/table`（时距矩阵，用于 ETA、派单、选点）
- 8.6 HTTP API：`/match`（地图匹配，GPS 轨迹对齐道路）
- 8.7 HTTP API：`/trip`（多点最短回路/旅行商近似）
- 8.8 编码与几何：polyline/geojson、抽稀、返回体大小控制
- 8.9 性能与并发：线程、内存、缓存、冷启动与超时治理
- 8.10 本章产出：一套可复用的 OSRM 部署与调用示例集合

---

### 第9章｜设计“面向业务”的 OSM 导航 API（把 Nominatim/OSRM 变成产品级接口）
文件：[`chapter9.md`](chapter9.md)

- 9.1 为什么要做 Facade API：统一协议、隐藏差异、稳定演进
- 9.2 资源模型：Place、Coordinate、Route、Instruction、Candidate
- 9.3 坐标/语言/单位规范：避免跨系统混乱（含转换与校验）
- 9.4 典型端点设计：`/geocode`、`/reverse`、`/route`、`/matrix`、`/match`
- 9.5 错误模型：可重试 vs 不可重试、用户可理解错误、内部错误码映射
- 9.6 版本与兼容：v1/v2、字段新增策略、弃用流程
- 9.7 安全与治理：鉴权、限流、配额、审计、反爬与滥用检测
- 9.8 文档与 SDK：OpenAPI、示例、最佳实践、对外契约测试
- 9.9 本章产出：一份可直接落地的 Navigation API 规范（草案）

---

### 第10章｜生产化：性能、更新、监控、成本与合规（上线之后才是真正开始）
文件：[`chapter10.md`](chapter10.md)

- 10.1 SLO/SLI：延迟、成功率、超时率、数据新鲜度
- 10.2 缓存体系：候选缓存、反查缓存、路线缓存（与失效策略）
- 10.3 数据更新发布：双版本切换、灰度、回滚、无损迁移
- 10.4 可观测性：日志、指标、追踪（关键维度与告警阈值）
- 10.5 容灾与多地域：冷备/热备、数据同步、流量切换
- 10.6 性能调优：数据库热点、IO 瓶颈、CPU 密集阶段拆分
- 10.7 成本拆解：磁盘/内存/带宽/计算（如何做容量规划）
- 10.8 安全与隐私：位置数据的敏感性、脱敏、保留策略
- 10.9 ODbL 合规落地：署名展示、导出数据说明、对外声明模板
- 10.10 本章产出：上线检查清单 + 运维手册目录（可复制到你的项目）

---

## 第三部分：LLM Toolcall 与 Navigation MCP（把导航能力变成“智能体可用工具”）

### 第11章｜LLM Toolcall & Agent：导航工具设计、澄清策略与可靠执行
文件：[`chapter11.md`](chapter11.md)

- 11.1 对话式导航的任务分解：意图 → 地点 → 路线 → 指令 → 解释
- 11.2 工具边界：哪些该让 LLM 决策、哪些必须让工具返回
- 11.3 Tool Schema 设计：输入校验、枚举参数、默认值与幂等
- 11.4 多候选与清：最小追问、可解释候选列表、用户选择机制
- 11.5 可靠执行：先 geocode 再 route、失败回退、超时与重试策略
- 11.6 抗提示注入与越权：把“用户指令”与“工具参数”隔离
- 11.7 输出表达：把路线变成可读指令（含转向、路名、距离、时间）
- 11.8 评估与测试：离线用例、回归、对话一致性、工具成功率
- 11.9 本章产出：一套“导航 Agent 工具调用范式”与可复用 prompt 模板

---

### 第12章｜Navigation MCP：MCP Server 设计、工具编排、观测与安全
文件：[`chapter12.md`](chapter12.md)

- 12.1 MCP 概念速览：为什么需要 MCP、与普通 REST 工具的关系
- 12.2 Navigation MCP 能力分层：Core（geocode/route）与 Extended（match/isochrone）
- 12.3 MCP Server 架构：连接 Nominatim + OSRM + 缓存 + 审计 + 限流
- 12.4 Tool 定义：`geocode` / `reverse_geocode` / `route` / `matrix` / `match` / `isochrone`
- 12.5 统一错误模型：错误码、可重试信号、用户可解释信息
- 12.6 资源（Resources）：地图链接、调试 trace、结果复现信息
- 12.7 安全与权限：API Key、租户隔离、可见范围（地理围栏/国家）
- 12.8 观测与评估：tool telemetry、成功率、延迟分布、成本归因
- 12.9 端到端示例：自然语言 → 多工具链路 → 可解释输出
- 12.10 上线清单与扩展路线：公交/交通/EV 路由、实时事件、离线包
