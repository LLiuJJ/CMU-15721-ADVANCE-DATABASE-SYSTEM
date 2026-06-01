## 运行时数据收集与目录更新(System Catalog Updates)
自适应查询优化(Adaptive Query Optimization)最基础的实现形式，是在查询执行过程中收集关于数据分布和谓词选择率(Predicate Selectivity)的运行时信息(Runtime Information)。传统优化器通常依赖预先收集的直方图(Histograms)和统计模型(Statistical Models)来估算扫描操作(Scan Operations)将输出的元组(Tuples)数量，该估算结果直接决定了连接顺序(Join Order)与整体执行计划(Execution Plan)的结构。然而，在实际执行阶段，数据库能够直接观测到真实的选择率(Selectivity)。若代价模型(Cost Model)预估的匹配率为 1%，而运行时评估(Actual Runtime Evaluation)却显示高达 99%，系统可通过两种途径利用该差异：一是动态调整当前查询的执行策略；二是将精确的运行时指标(Runtime Metrics)反馈至全局系统目录(System Catalog)，从而优化未来所有查询的执行计划生成。
![关键帧](keyframes/part001_frame_00000000.jpg)

## 基于回退的计划修正(Fallback-based Plan Correction)
应对查询性能退化(Performance Regression)的一项基础技术是**基于回退的计划修正**。现代商业数据库系统通常维护一个内部的执行历史存储库(Execution History Repository)，用于追踪每个已执行查询的详情、对应的执行计划、估算代价(Estimated Cost)以及真实的运行时指标（如 CPU 耗时、处理的元组数量等）。当高频查询遭遇环境变更（例如统计信息更新或新增索引）时，优化器可能会为其生成全新的执行计划。若新计划的实际运行时性能劣于历史缓存的旧计划，系统将自动回退(Fallback)至先前经过验证的稳定策略。这种全局回退机制有效保障了在底层物理设计或统计模型发生变更时，查询执行的稳定性(Stability)。
![关键帧](keyframes/part001_frame_00224983.jpg)
![关键帧](keyframes/part001_frame_00301316.jpg)

## 计划拼接(Plan Stitching)：超越整体回退
尽管简单的回退机制提供了稳定性，但其运作方式较为粗放，且呈现非此即彼的二元特性。一种更为精细的替代方案是**计划拼接**技术，该方案由微软率先提出。系统不会直接全盘丢弃表现不佳或失效的执行计划，而是从中提取运行高效的子计划(Subplan)，并将其重新组装为全新的混合计划(Hybrid Plan)。其核心在于，这些复用的子计划无需源自完全相同的查询语句；只要它们在逻辑上等价(Logically Equivalent)，来自不同查询执行上下文的组件即可安全地集成至同一计划中。当物理设计变更(Physical Design Changes)导致计划局部失效时（例如索引扫描(Index Scan)所依赖的索引被删除），拼接技术能够挽救剩余合法的算子(Valid Operators)，从而避免强制触发全量的重新优化(Re-optimization)。
![关键帧](keyframes/part001_frame_00392749.jpg)
![关键帧](keyframes/part001_frame_00438950.jpg)
![关键帧](keyframes/part001_frame_00447749.jpg)

## 用于子计划优化的动态规划(Dynamic Programming)
为高效构建此类拼接计划，系统采用了一种自底向上(Bottom-Up)的动态规划搜索算法。该算法逐层评估算子代价(Operator Cost)，在每一层级筛选出最高效的子计划，进而组装出全局代价最低的执行路径。尽管这种启发式(Heuristic)方法无法保证寻得绝对最优解，亦不能确保始终生成完全合法的执行计划，但其以极低的计算开销显著提升了系统的自适应能力。例如，若两个被挽救的子计划段运行时开销(Runtime Cost)分别为 600 和 150，组合后的拼接总代价仅为 750，该结果不仅优于原始计划的 1000 代价，也远胜于新生成的失败计划。此拼接过程独立于标准查询优化器(Standard Query Optimizer)运行，使得系统能够充分利用历史运行时数据，动态拼装出高效的执行路径。
![关键帧](keyframes/part001_frame_00522983.jpg)
![关键帧](keyframes/part001_frame_00577150.jpg)