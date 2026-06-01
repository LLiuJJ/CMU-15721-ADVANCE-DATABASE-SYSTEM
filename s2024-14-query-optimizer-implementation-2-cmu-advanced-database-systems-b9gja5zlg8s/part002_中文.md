## 备忘录表查找与代价计算
![关键帧](keyframes/part002_frame_00000000.jpg)
优化器利用备忘录表(Memo Table)避免冗余的树遍历(Tree Traversal)与重复的代价估算(Cost Estimation)。在评估子树(Subtree)时，优化器会检查备忘录表中先前已计算的扫描代价(Scan Cost)（例如针对表 `v` 和 `a` 的代价）并予以复用。针对特定的连接操作(Join Operation)，系统会生成所有可行的物理多表达式(Physical Multi-expression)，并应用代价模型(Cost Model)以确定最高效的执行算法。在本示例中，哈希连接(Hash Join)被判定为最优选项，其综合代价为 80（包含表 `a` 和 `b` 的扫描代价）。该最优代价随后将被记录至备忘录表中，供后续查询复用。

## Cascades 框架概述
![关键帧](keyframes/part002_frame_00030000.jpg)
搜索过程将递归评估其他分支（例如连接表 `C`），探索表 `A`、`B` 与 `C` 的不同连接顺序(Join Order)，直至搜索空间(Search Space)被充分遍历，最终确定最低的整体代价（例如 125）。该工作流程体现了 Cascades 优化器(Optimizer)的核心机制：它起始于逻辑查询计划(Logical Query Plan)，无需预先指定连接顺序、访问路径(Access Path)或连接算法(Join Algorithm)。优化器自顶向下(Top-down)遍历表达式树(Expression Tree)，持续查询并更新备忘录表，以追踪并保留搜索过程中发现的最优代价计划(Cost-based Plan)。

## 规则应用与物理表达式生成
![关键帧](keyframes/part002_frame_00060000.jpg)
尽管 Cascades 本质上是基于代价的优化器(Cost-based Optimizer)，它仍会在特定节点处一致地应用转换规则(Transformation Rules)与实现规则(Implementation Rules)。当处理表访问操作（如表 `A`）时，优化器会自动生成多个物理多表达式，例如顺序扫描(Sequential Scan)及所有可用的索引扫描(Index Scan)。为避免组合爆炸(Combinatorial Explosion)，系统不会预先物化(Materialize)所有可能的候选方案。相反，系统会利用排序或启发式策略引导探索过程，随后进入基于代价的选择阶段，过滤劣质选项，仅保留最具潜力的物理表达式(Physical Expression)。

## Cascades 与 Volcano 的对比及统一搜索过程
![关键帧](keyframes/part002_frame_00210000.jpg)
Cascades 与 Volcano 优化器框架等早期架构的主要差异在于搜索策略(Search Strategy)与规则集成(Rule Integration)机制。Volcano 采用严格的穷举搜索(Exhaustive Search)，缺乏基于优先级的剪枝机制(Pruning Mechanism)，且将逻辑转换(Logical Transformation)与物理代价计算严格分离。Cascades 将这些步骤整合为单一的动态过程，摒弃了僵化的两阶段流水线(Two-phase Pipeline)设计。备忘录表中的逻辑组(Logical Group)（如 `A`、`B`、`C`）仅定义*必须执行哪些*逻辑操作，而不规定*具体如何执行*。在遍历过程中，优化器会同步应用转换规则、将逻辑表达式(Logical Expression)转换为物理表达式、评估代价并剪枝搜索空间，从而形成紧密集成的优化循环(Optimization Loop)。

## 终止条件与代价估算数据
![关键帧](keyframes/part002_frame_00330000.jpg)
搜索过程受终止条件(Termination Condition)控制，触发条件可能包括执行计时器超时、达到转换规则应用次数上限，或当代价改善趋于平缓时提前终止。与早期系统在评估前物化所有候选方案的做法不同，Cascades 能够动态控制搜索广度(Search Breadth)。代价估算依赖于通过 `ANALYZE` 命令收集的统计信息(Statistics)、历史查询执行数据、数据概要(Data Sketches)或局部数据采样(Partial Data Sampling)（SQL Server 中的典型技术）。代价模型对这些输入进行抽象，利用选择率(Selectivity)与 I/O/CPU 估算来评估操作符(Operator)代价，在规划阶段无需访问实际表数据。

## 自底向上与自顶向下优化策略
![关键帧](keyframes/part002_frame_00480000.jpg)
本讲座对比了自底向上(Bottom-up)与自顶向下(Top-down)的优化策略。纯自底向上策略要求在应用任何分支定界剪枝(Branch-and-Bound Pruning)之前完全评估叶节点(Leaf Nodes)。这会延迟剪枝时机，但允许物理操作符(Physical Operator)的增量物化，并有助于尽早确立代价上界(Cost Upper Bound)。Cascades 采用结合记忆化(Memoization)的自顶向下方法，在搜索空间探索与积极剪枝之间取得平衡。高级优化技术（如基于超图的分组(Hypergraph-based Grouping)）可进一步细化搜索过程。相关研究（含一篇 2006 年的引用文献）表明，特定的自底向上或混合优化技术(Hybrid Optimization Technique)有时能比传统 Cascades 实现更快地识别最优连接顺序。最后，物理数据属性(Physical Data Property)（如归并连接(Merge Join)所需的预排序输入）会在遍历过程中动态校验，以确保所选计划满足所有操作符的前置条件(Preconditions)。