## 查询计划(Query Plan)转换与动态代价评估
![关键帧](keyframes/frame_00000000.jpg)
优化过程中应用的转换规则极大地影响了查询计划的搜索空间(Search Space)。系统通常采用动态规划(Dynamic Programming)方法，其代价评估(Cost Estimation)与剪枝(Pruning)机制会因转换结构的不同而有所差异。优化器评估和选择潜在执行路径(Execution Path)的方式直接决定了整体查询性能。

## System R 优化器与左深树(Left-Deep Tree)约束
![关键帧](keyframes/frame_0025583.jpg)
System R 优化器开创了自底向上(Bottom-Up)优化策略的早期实现。传入的查询首先被分解为多个查询块(Query Blocks)，这些块可表示阻塞操作符(Blocking Operators)、嵌套子查询(Nested Subqueries)或整体计划的其他子组件。针对每个逻辑块内的逻辑操作符(Logical Operators)，优化器会生成一组可行的物理操作符(Physical Operators)，重点关注连接算法(Join Algorithms)和数据访问路径(Data Access Paths)（如顺序扫描(Sequential Scan)与索引查找(Index Lookup)）。为限制指数级增长的搜索空间，System R 将评估范围限定为左深连接树。尽管该设计源于 20 世纪 70 年代的硬件限制，且在现代数据库中仍被广泛沿用，但它本质上限制了优化器发现可能更高效的宽树(Bushy Tree)或右深树(Right-Deep Tree)执行计划的机会。

## 独立访问路径选择与连接枚举(Join Enumeration)
![关键帧](keyframes/frame_00122966.jpg)
优化过程首先独立地为查询中涉及的每张表确定最佳访问路径。例如，优化器可能为 `artists` 和 `appears` 表选择顺序扫描，而为 `album` 表的 `name` 列选择索引查找。这些访问决策均在评估连接策略(Join Strategies)之前完成。确定访问路径后，优化器会枚举所有可能的连接顺序与组合，实质上生成了潜在连接计划的笛卡尔积(Cartesian Product)。在启动自底向上的动态规划评估前，系统会立即剪枝无效或低效的选项（如显式的笛卡尔积连接），以简化搜索过程。

## 自底向上动态规划执行
![关键帧](keyframes/frame_00186933.jpg)
![关键帧](keyframes/frame_00220083.jpg)
自底向上的搜索过程以树状结构构建查询计划，其中基础表(Base Tables)位于底层，最终结果位于顶层。优化器从底层开始，评估所有可用于合并两张表或中间结果的物理连接操作符（如哈希连接(Hash Join)、归并连接(Merge Join)）。对于上一层的每个节点，系统会依据代价模型(Cost Model)计算执行代价，并仅保留当前最优的执行路径。该迭代过程随着计划树的扩展不断向上推进。而诸如谓词下推(Predicate Pushdown)或聚合策略(Aggregation Strategies)等额外转换，通常在分层搜索中单独处理，或作为独立的查询块对待；因为若将它们直接嵌入自底向上的动态规划搜索中，会显著增加状态空间的复杂度。

![关键帧](keyframes/frame_00262849.jpg)
![关键帧](keyframes/frame_00276166.jpg)
当搜索到达计划树的根节点时，优化器即可确定最终的目标结果集，并自上而下回溯(Trace Back)以重构完整的最低代价执行路径。这种分阶段方法的一个显著局限性在于访问路径选择与连接顺序枚举的解耦。在未确定连接顺序的情况下预先选定访问路径，可能导致次优决策。例如，某种特定的连接顺序可能使得基于索引的嵌套循环连接(Index Nested Loop Join)的代价远低于哈希连接，但由于早期的解耦设计，优化器在缺乏统一联合搜索策略的情况下，无法识别并利用这种协同优化(Synergistic Optimization)潜力。

## 物理属性(Physical Properties)与排序顺序(Sort Order)的处理
![关键帧](keyframes/frame_00296433.jpg)
System R 动态规划搜索的一个早期局限在于，其初始版本未充分考虑物理数据属性（例如 `ORDER BY` 子句所要求的排序顺序）。为弥补这一缺陷，优化器被扩展为能够同时追踪满足与不满足特定物理属性要求的最佳执行计划。当查询需要排序输出时，系统会权衡两种方案：一是在未排序计划的末尾显式添加独立的排序操作符(Sort Operator)；二是利用天然可产生有序输出的归并连接。通过估算显式排序操作的代价并将其叠加至无序计划的基准代价上，优化器能够对比得出总代价最低的策略。该机制有效地将物理属性纳入了全局代价评估体系，而非将其作为初始自底向上搜索的硬性前置条件。