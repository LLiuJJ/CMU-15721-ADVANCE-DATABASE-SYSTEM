![关键帧](keyframes/part001_frame_00000000.jpg)
## 硬件瓶颈转移与早期 Spark SQL 的挑战
初始方案的计算开销逐渐凸显，尤其是在 2015 至 2020 年左右的硬件环境下。彼时磁盘读写速度相对较慢，数据仓库(Data Warehouse)与联机分析处理(Online Analytical Processing, OLAP)系统通常属于“磁盘受限(disk-bound)”而非“CPU受限(CPU-bound)”。如今，情况已发生反转：CPU 取代磁盘成为了主要性能瓶颈。然而，初代 Spark SQL 的架构设计正是基于早期磁盘输入/输出(Input/Output, I/O)为主要限制因素（而非 CPU 算力）的时代背景所构建的。

## JVM 垃圾回收与堆外内存变通方案
随着计算开销的凸显，早期 Spark SQL 的工作负载迅速转变为 CPU 受限(CPU-bound)。其中的核心瓶颈之一在于 Java 虚拟机(Java Virtual Machine, JVM)的内存管理模型。在 Java 中，每次通过 `new` 关键字分配的对象均驻留于堆内存(Heap Memory)中，垃圾回收器(Garbage Collection, GC)需在此追踪对象引用并定期扫描内存，以回收可释放的空间。由于 JVM 未提供直接的原生内存分配函数（如 `malloc`），工程师不得不自行实现堆外(off-heap)内存管理。借助 JVM 内部的 `Unsafe` API，内存被分配至进程拥有但对 GC 不可见的区域。为规避 GC 停顿(GC Pause)拖慢执行速度，开发团队投入了大量工程资源将数据迁移至堆外，但这也不可避免地为系统引入了显著的复杂性。

## JIT 编译限制与工程可扩展性
另一个关键挑战源于即时编译(Just-In-Time Compilation, JIT)机制。若要榨取最佳性能，开发者必须深入掌握 JVM 底层机制，这导致团队在扩展 Spark SQL 优化方向的工程人力时面临困难。与多数开发者能够相对快速上手 C++ 代码库的生态不同，JVM 级别的性能优化门槛极高且专业性极强。此外，JVM 对编译后方法体的代码体积存在严格限制。正如 Photon 相关论文所述，当数据表的列数达到数百级别时，JVM 会因代码过大而拒绝编译该查询，并自动降级至速度较慢的解释器模式(Interpreter Mode)。随着硬件算力与分析需求的同步演进，这些底层约束充分暴露了 2010 年代初基于 JVM 架构的局限性。

## Photon 的设计：嵌入式内核库
为突破传统架构的瓶颈，Photon 并未被设计为类似 Dremel 或 Snowflake 的独立数据库系统，而是作为一个可嵌入的向量化加速库。它能够直接集成至 Spark 或 Databricks Runtime 等现有运行时环境(Runtime Environment)中。相较于 Velox 等底层执行框架，Photon 的定位更为轻量，不直接负责线程池或内存池的管理。相反，它提供细粒度的计算内核(Computation Kernels)与算子(Operators)，可被精确注入至查询执行计划(Query Execution Plan)中。所有资源的编排、内存分配与任务调度，均交由宿主运行时(Host Runtime)统一接管。

## 无缝集成与用户透明性
Photon 的核心设计目标是与 Databricks Runtime 实现无缝且非侵入式的集成。这使得 Photon 能够以渐进式(Progressive)接管特定计算组件，用户无需重写 SQL 查询或调整现有工作流。保持严格的结果一致性(Result Consistency)是重中之重：启用或升级 Photon 绝不应改变查询输出或系统语义。此外，Photon 在传统 Spark 的逐行处理(Row-at-a-time Processing)模式与现代向量化(Vectorized Execution)、列式执行(Columnar Execution)之间架起了桥梁。若某个查询算子暂未提供 Photon 加速版本，系统会自动触发优雅降级(Graceful Fallback)至原始 Spark 运行时路径，从而保障业务的连续性与稳定性。

![关键帧](keyframes/part001_frame_00449799.jpg)
## 高层架构与优化重点
Photon 的整体架构与现代分布式查询引擎(Distributed Query Engine)一脉相承，采用中心化的驱动节点(Driver/Coordinator)负责 SQL 解析、生成执行计划，并在集群中进行任务调度。其核心优化方向聚焦于基于批处理(Batch Processing)的向量化执行与预拷贝内存原语(Pre-copy Memory Primitives)。系统原生支持基于数据混洗(Shuffle)的数据分发机制，底层高效实现了归并连接(Merge Join)与哈希连接(Hash Join)，并深度集成了查询优化器(Query Optimizer)与自适应执行(Adaptive Execution)技术，可根据运行时统计信息(Runtime Statistics)进行动态计划调整。

![关键帧](keyframes/part001_frame_00543166.jpg)
## 分布式执行流程与 Shuffle 阶段
在执行阶段，Driver 节点将计算任务分发至 Worker 节点，各节点从分布式存储系统(Distributed Storage System)中拉取数据并完成初始阶段的计算。随后，系统进入 Shuffle 阶段，中间结果在此阶段按分区(Partition)切分并在不同 Worker 节点间进行网络交换。在标准 Spark 部署架构中，通常借助本地哈希表实现内存型 Shuffle(In-memory Shuffle)，以便下游 Worker 节点直接高效拉取数据。Photon 深度嵌入该分布式执行流程中，在加速核心计算算子的同时，完全兼容宿主运行时原有的任务调度与数据移动机制。

![关键帧](keyframes/part001_frame_00572850.jpg)