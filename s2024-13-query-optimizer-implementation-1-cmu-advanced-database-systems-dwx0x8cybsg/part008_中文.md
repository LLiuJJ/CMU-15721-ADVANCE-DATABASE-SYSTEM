## 搜索终止与分支定界剪枝(Branch and Bound Pruning)
![关键帧](keyframes/part008_frame_00000000.jpg)
Cascades 优化器采用积极的分支定界剪枝策略，以确保搜索空间在计算上保持可控。一旦框架发现一个完整的物理计划(Physical Plan)及其初始代价（例如 80 或 125），便会以此设定严格的代价上界(Cost Upper Bound)。在后续遍历中，若任何部分构建计划(Partial Plan)的累积代价超过该阈值，优化器将立即剪除(Prune)该分支。这种代价限制机制有效避免了搜索过程在呈指数级增长的次优执行路径(Suboptimal Execution Paths)上浪费资源，并确保优化过程能在实际可行的时间窗口内完成。

## 处理估算误差(Estimation Errors)与备忘录表的生命周期
在实际运行中，代价估算(Cost Estimation)往往与实际运行时性能(Runtime Performance)存在偏差。大多数传统数据库系统会选择按既定路线执行已选计划，即便在查询执行中途发现估算误差。然而，先进的架构允许注入执行钩子(Execution Hooks)，以触发运行时重优化(Runtime Re-optimization)或收集执行期反馈(Execution Feedback)。此类反馈可用于动态更新后续查询的基数估算(Cardinality Estimation)与代价模型(Cost Model)。尽管存在复用潜力，备忘录表(Memo Table)通常会在查询执行完毕后立即释放。跨查询的备忘录复用机制(Cross-Query Memo Reuse)极少被实现，因为最优代价与计划结构高度依赖于特定查询参数（如 `WHERE` 子句谓词(Predicates)与过滤选择率(Filter Selectivity)）。维护一个持久化且具备参数感知能力(Parameter-Aware)的备忘录结构将引入显著的存储与查找开销(Lookup Overhead)，其成本通常远超所能带来的性能收益。

## 后续主题：Cascades、随机搜索(Stochastic Search)与超图(Hypergraph)
![关键帧](keyframes/part008_frame_00108750.jpg)
课程后续部分将深入剖析 Cascades 框架，重点探讨规则应用(Rule Application)、优先级管理(Priority Management)及物理属性强制执行(Physical Property Enforcement)的底层运作机制。接下来的课程还将探讨随机搜索算法(Stochastic Search Algorithms)，但需注意，出于系统稳定性与行为可预测性(Stability and Predictability)的考量，PostgreSQL 等生产级数据库已基本弃用此类算法。其他高级议题将涵盖 HyPer 数据库中的子查询展开(Subquery Unnesting)策略，以及基于超图的动态规划(Hypergraph-based Dynamic Programming)。后者为突破传统左深树(Left-Deep Tree)限制、优化复杂多表连接(Complex Multi-table Joins)提供了坚实的数学理论基础。
![关键帧](keyframes/part008_frame_00158850.jpg)

## 查询优化推荐行业资源
![关键帧](keyframes/part008_frame_00136849.jpg)
对于渴望深入掌握业界前沿洞察的读者，强烈建议研读精选的各大数据库厂商技术演讲列表。Microsoft SQL Server 的演讲堪称权威资料，详尽展示了业界顶尖的优化技术以及分层与统一混合搜索策略(Layered and Unified Hybrid Search Strategies)，其设计理念与 HyPer 等现代研究型数据库高度契合。此外，CockroachDB 的架构演讲亦不容错过，它呈现了一个高度精炼的 Cascades 实现(Highly Refined Cascades Implementation)，在经典框架之上融合了诸多传统商业优化器所不具备的现代增强特性。这些技术分享为理解理论优化器框架如何落地转化为生产级查询处理引擎(Query Processing Engine)提供了极具价值的工程背景。