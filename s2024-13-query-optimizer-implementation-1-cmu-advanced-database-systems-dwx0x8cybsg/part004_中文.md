## 结合物理属性(Physical Properties)确定最终计划
![关键帧](keyframes/frame_00000000.jpg)
在自底向上(Bottom-Up)的搜索过程中，优化器会同时维护已排序和未排序的候选计划(Candidate Plans)。在最终的评估阶段，系统会将“为未排序计划（例如哈希连接(Hash Join)）附加专用排序操作符的代价”与“直接采用天然可产生有序输出的排序合并连接(Sort-Merge Join)”进行对比。基于代价估算，系统将选择总体代价更低的方案。然而，这暴露了一个根本性局限：排序顺序(Sort Order)等物理属性仅被视为后处理步骤(Post-processing Steps)，而非原生集成到初始搜索空间(Search Space)中。因此，优化器无法在搜索初期全面评估所有可能的计划排列组合。

## 搜索后的启发式规则(Heuristic Rules)与逻辑转换(Logical Transformations)
![关键帧](keyframes/frame_00042200.jpg)
由于核心的动态规划(Dynamic Programming)搜索主要聚焦于连接顺序(Join Order)与访问路径(Access Paths)，额外的逻辑转换通常在独立阶段进行处理。这些规则多以过程式检查(Procedural Checks)的形式实现，往往缺乏对底层物理算法(Physical Algorithms)或数据属性的深度感知。在 PostgreSQL 等系统中，启发式优化会在主代价搜索(Cost-Based Search)完成后应用。优化器会执行一个最终的“微调”阶段，将物理优化策略叠加至已构建的计划之上。这种分阶段方法导致了逻辑与物理优化间的割裂；例如，若在确定连接顺序前便固化访问方法，系统便可能错失良机：特定的连接顺序本可使索引嵌套循环连接(Index Nested Loop Join)显著优于其他方案。

## 搜索终止策略(Search Termination Strategies)
![关键帧](keyframes/frame_00160899.jpg)
查询优化(Query Optimization)本质上是一个 NP 完全问题(NP-Complete Problem)，这意味着其搜索空间理论上呈指数级膨胀。为防止优化过程无限期运行，优化器实施了严格的终止条件。最直接的方法是设置硬性时钟超时(Hard Clock Timeout)。更高级的策略则采用代价阈值(Cost Threshold)：当新生成计划带来的边际收益(Marginal Gain)微乎其微（例如改善幅度低于 10%）时，立即终止搜索。基于预估查询复杂度的动态超时(Dynamic Timeout)虽理论可行，但在计划生成前精准预测复杂度往往极为困难。另一种方案是，一旦遍历完给定子计划的所有排列组合，搜索即告结束。微软引入了一种巧妙的硬件无关策略(Hardware-Agnostic Strategy)：通过统计已考虑的转换操作(Transformation Operations)总数来限制搜索。设定被评估计划数量的上限，可确保优化器在从移动设备到高端服务器的各类硬件环境中保持行为一致性与可预测性。

## 优缺点与剪枝权衡(Pruning Trade-offs)
![关键帧](keyframes/frame_00370033.jpg)
传统的自底向上动态规划方法至今仍极具实用性，并构成了众多现代数据库优化器的核心基石。为控制搜索复杂度，系统会应用严格的剪枝策略(Pruning Strategies)，例如将搜索范围限定于左深树(Left-Deep Trees)。此举虽大幅压缩了搜索空间(Search Space)并将优化耗时控制在合理范围内，但也引入了显著的权衡(Trade-off)：优化器可能会系统性地舍弃那些依赖宽树(Bushy Trees)或右深树(Right-Deep Trees)结构的全局最优计划(Global Optimal Plans)。此外，由于剪枝决策多依赖启发式规则而非详尽的代价分析，加之物理属性需额外后处理，最终生成的执行计划(Execution Plan)往往难以保证绝对的理论最优性。

## 过程式优化器代码(Procedural Optimizer Code)的挑战
![关键帧](keyframes/frame_00414316.jpg)
历史上，实现此类搜索策略的查询优化器(Query Optimizer)多直接采用过程式代码编写。开发者大量依赖 `if-then-else` 语句嵌入模式匹配(Pattern Matching)逻辑与转换规则，以识别特定查询结构并施加相应修改。这种紧耦合的实现方式在维护与扩展方面历来备受诟病。硬编码(Hard-coding)过程式优化器不可避免地会引发代码冗余、隐蔽缺陷(Hidden Bugs)及复杂的模块间依赖；例如，应用某项转换规则可能会意外破坏另一规则所依赖的前置条件。随着查询语法的日益复杂与数据库功能的不断演进，持续维护此类硬编码优化器已变得愈发难以为继。

## 优化器生成器(Optimizer Generator)与基于规则 DSL
![关键帧](keyframes/frame_00523083.jpg)
为突破过程式代码的局限，数据库界于 20 世纪 80 年代末至 90 年代初引入了优化器生成器。开发者不再硬编码业务逻辑，转而采用高级领域特定语言(Domain-Specific Language, DSL)来声明转换规则与模式匹配条件。生成器会自动将这些声明编译为高效的底层代码，并在数据库系统构建阶段完成集成。该架构实现了搜索引擎(Search Engine)与优化规则的清晰解耦，支持多团队并行开发。优化规则得以集中管控、灵活扩展，且无需改动核心搜索框架即可迭代更新。IBM 的 Starburst 项目（至今仍是 DB2 的基石）以及 Exodus/Volcano/Cascades 框架等开创性工作确立了这一范式，为现代数据库优化器的分层搜索与统一架构奠定了坚实基础。