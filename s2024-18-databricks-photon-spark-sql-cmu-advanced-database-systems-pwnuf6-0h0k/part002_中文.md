## 高级执行与 Shuffle 架构
该系统采用多阶段执行模式，执行器(Executor)可将中间数据写入类似 Unifor(Unifor) 或 Apache Spark 的远程 Shuffle(Remote Shuffle) 服务中。在后续阶段，执行器会从上一阶段的输出中拉取(Pull)数据。若未启用远程 Shuffle 服务，执行器必须保持驻留，直至下一阶段初始化。调度器(Scheduler)可通过将多个阶段进行同节点部署(Co-locating)来优化此过程，从而复用本地内存。宏观而言，该执行模型与 Dremel(Dremel) 高度相似。尽管不同系统的 Shuffle 实现各异，但“从上一阶段拉取数据”这一核心机制始终保持一致。

![关键帧](keyframes/part002_frame_00000000.jpg)

![关键帧](keyframes/part002_frame_00013333.jpg)

## Photon 简介与预编译原语
Photon(Photon) 并非独立的服务或进程，而是直接运行于每个执行器(Executor)内部，并由 Java 代码调用。它实现了一个拉取式(Pull-based)向量化引擎(Vectorized Engine)，该引擎底层依赖预编译(Pre-compiled) C++ 代码，论文中将其称为算子内核(Operator Kernels)或原语(Primitives)。这些原语针对特定数据类型进行了模板化，能够高效处理各类计算任务（如比较、算术运算等），且无需依赖即时(Just-In-Time, JIT)代码生成。通过将函数指针调用(Function Pointer Call)的开销分摊至整个数据向量，并结合表达式融合(Expression Fusion)技术，Photon 实现了极高的执行效率。该论文广受好评的原因在于，它并未仅停留在结果展示，而是透明且详尽地剖析了这些设计决策背后的工程考量。

![关键帧](keyframes/part002_frame_00073750.jpg)

## 权衡取舍：向量化引擎 vs. JIT 编译
Photon 的一项核心工程决策是采用预编译(Pre-compiled)向量化模型，而非 JIT 编译(JIT Compilation)方案（例如 Hyper(Hyper) 数据库）。尽管 JIT 引擎在初期可能带来更高的原始性能，但其调试与性能剖析(Profiling)的开销极为庞大。JIT 编译生成的代码一旦发生崩溃，开发人员将直接面对缺乏源码上下文的汇编代码，迫使团队不得不开发大量定制化工具来进行问题诊断。相比之下，预编译原语更利于迭代开发，对专业技能的门槛要求较低，能够让更多工程师参与其中。从长远来看，该方案不仅能达到与 JIT 引擎相当甚至更优的性能，对于大规模工程团队而言，也是更具可持续性的技术选型。

## SIMD 优化与位置列表跟踪
Photon 高度依赖编译器的自动向量化(Auto-vectorization)功能；鉴于其工作负载中的数据规模通常较为适中，该功能表现优异。在必要时，开发人员会手动插入 SIMD 内在函数(SIMD Intrinsics)以确保指令执行达到最优。为管理数据流，算子通过调用 `get_next` 函数来生成列批(Column Batches)。当过滤器(Filter)剔除部分元组后，Photon 并未对向量进行物理压缩，而是采用位置列表(Position List，即有效偏移量向量)来追踪活跃行。论文将该方法与基于位图(Bitmap-based)的追踪方式进行了对比，结论表明：除非在几乎所有元组均保持活跃的极端边缘场景下，否则位置列表在绝大多数情况下的性能更优。这一结论随后得到了后续研究的进一步验证，例如卡内基梅隆大学(Carnegie Mellon University, CMU)的一篇硕士学位论文，该研究在多种工作负载与硬件配置下对这两种方案进行了详尽的基准测试(Benchmark)。

![关键帧](keyframes/part002_frame_00409399.jpg)

![关键帧](keyframes/part002_frame_00427799.jpg)

## 为避免垂直算子融合而牺牲用户剖析
Photon 刻意避免了类似 Hyper(Hyper) 的垂直算子融合(Vertical Operator Fusion)技术，该技术通常会将多个算子合并至单一执行管道(Execution Pipeline)中，并形成深层嵌套循环。这一决策主要基于两方面考量：一是保持工程实现的简洁性，二是满足面向用户的性能剖析(Profiling)需求。一旦算子进行垂直融合，将实际执行时间精确映射回查询计划(Query Plan)中的具体 SQL 算子将变得极其困难。而编写 SQL 的用户亟需清晰的诊断反馈，以准确定位查询计划中的性能瓶颈。作为垂直融合的替代方案，Photon 在单个算子内部采用了水平融合(Horizontal Fusion)或表达式融合(Expression Fusion)。尽管该方法以批处理(Batch)而非逐元组(Tuple-at-a-time)的方式处理数据，但它保留了透明且可解释的执行模型。该模型与逻辑查询计划高度对齐，从而能够生成准确且具备可操作性的性能分析报告。

![关键帧](keyframes/part002_frame_00463916.jpg)

![关键帧](keyframes/part002_frame_00540183.jpg)

![关键帧](keyframes/part002_frame_00549000.jpg)