## 无分支执行与硬件对齐
在向量化查询执行(Vectorized Query Execution)中，强烈不建议在热路径(Hot Path)中实现复杂的条件逻辑或动态压缩(Dynamic Compaction)已过滤的元组。当谓词(Predicate)过滤掉批次中的大部分数据时（例如 1,024 个元组中仅有 1 个有效），将无效数据传递至下游(Downstream)看似效率低下。然而，显式跟踪空槽位、填补数据空隙或引入重度分支(Heavy Branching)会创建控制依赖(Control Dependency)，从而导致现代超标量 CPU(Superscalar CPU) 发生流水线停顿(Pipeline Stall)。严格遵循无分支执行(Branchless Execution)原则，即仅通过轻量级元数据(Lightweight Metadata)传递有效信息并直接跳过“垃圾”元组，实际消耗的 CPU 周期(CPU Cycles)反而更少。通过保持直接且线性的执行路径(Execution Path)，避免在循环内部插入条件判断，该设计与底层硬件的架构偏好高度契合，优先保障可预测的指令流(Instruction Flow)与绝对吞吐量(Absolute Throughput)，而非追求逻辑层面的紧凑性。
![关键帧](keyframes/part008_frame_00000000.jpg)

## 选择向量的大小与元数据管理
为了在不重构内存结构的前提下高效追踪有效元组，选择向量(Selection Vector)（亦称位置列表(Position List)）通常按最大向量大小(Vector Size)进行预分配(Pre-allocation)，默认容量一般为 1,024 个元素。该结构不依赖动态扩缩容(Dynamic Resizing)，而是通过简单的长度计数器(Length Counter)或结束偏移量(End Offset)来界定有效数据的边界。此外，为最大限度压缩内存占用，此类位置列表普遍采用 16 位整型(16-bit Integer)而非 64 位指针(64-bit Pointer)。由于偏移量仅需索引单个固定大小批次内的相对位置，16 位数值已完全满足需求。这种定长且紧凑的元数据结构确保了可预测的内存访问模式，使辅助数据具备缓存友好性(Cache-friendly)，并彻底规避了在执行期管理变长数组(Variable-length Array)的额外开销。

## 禁止运行时内存分配
尽管采用 Slab 分配(Slab Allocation)或动态大小缓冲区(Dynamic-sized Buffer)看似能将中间结果紧密打包并驻留于 L1 缓存(L1 Cache)中，但高性能数据库执行引擎严格禁止在查询执行循环内调用 `malloc` 或触发任何操作系统级(Operating System-level)内存管理例程(Memory Management Routine)。系统调用(System Call)会引入不可预测的延迟(Unpredictable Latency)，破坏指令流水线(Instruction Pipeline)，并严重拖累系统吞吐量。相反，所有数据缓冲区、向量及元数据结构均在查询执行前完成预分配(Pre-allocation)。尽管该策略可能导致少量内存冗余，但其整体权衡(Trade-off)极为有利：它彻底消除了运行时分配开销，确保 CPU 维持紧密且无中断的执行循环(Tight Execution Loop)。这深刻体现了一项核心工程原则——架构简洁性(Simplicity)与硬件对齐(Hardware Alignment)的优先级，始终高于精巧却复杂的动态内存管理(Dynamic Memory Management)策略。

## 课程总结与并行执行预览
本模块圆满结束了对查询执行模型(Query Execution Model)的基础探讨，重点梳理了数据库架构从传统迭代拉取式(Iterator Pull-based)系统向向量化(Vectorized)与推送式(Push-based)设计的演进历程。核心结论明确指出：现代数据库引擎(Database Engine)优先采用无分支(Branchless)、预分配(Pre-allocated)且具备硬件感知(Hardware-aware)特性的执行路径，旨在最大化 CPU 执行效率，并最小化指令流水线停顿(Instruction Pipeline Stall)。在奠定上述单线程执行(Single-threaded Execution)基础后，后续章节将深入探讨并行执行策略(Parallel Execution Strategy)，剖析如何将查询工作负载(Query Workload)高效调度至多核 CPU(Multi-core CPU)，从而实现线性可扩展性(Linear Scalability)并进一步加速分析型数据处理(Analytical Processing)。
![关键帧](keyframes/part008_frame_00179799.jpg)
![关键帧](keyframes/part008_frame_00187733.jpg)
![关键帧](keyframes/part008_frame_00193899.jpg)
![关键帧](keyframes/part008_frame_00200266.jpg)
![关键帧](keyframes/part008_frame_00206233.jpg)
![关键帧](keyframes/part008_frame_00212399.jpg)
![关键帧](keyframes/part008_frame_00218866.jpg)