## 传统算子追踪的局限性
准确追踪单个算子(Operator)的精确耗时并将其归因映射以评估性能表现，历来是一项公认的难题，且在大规模场景下难以保持良好的扩展性。为了向用户清晰传达系统行为，这种碎片化的追踪方法并不切实际。因此，系统转向了**表达式融合(Expression Fusion)**技术，该技术旨在简化执行流程并提升系统透明度。
![关键帧](keyframes/part003_frame_00000000.jpg)

## 表达式融合的机制
表达式融合通过将多个连续的基础原语(Primitives)合并为单一的统一计算步骤，从而有效突破性能瓶颈。Photon 不会在算子内依次执行两个独立的函数，而是生成一个模板化原语(Templated Primitive)，同步处理这两项计算。
![关键帧](keyframes/part003_frame_00019899.jpg)
例如，日期范围查找(`BETWEEN`)会被转换为一个 `<=` 和一个 `>=` 条件，并通过逻辑与(AND)子句进行连接。若不进行融合，该操作需要调用两个独立的原语，生成两个偏移量列表(Offset Lists)，后续还必须对这些列表求交集才能定位匹配的记录。
![关键帧](keyframes/part003_frame_00042733.jpg)
融合技术消除了重复函数调用和列表求交所带来的开销。代码生成器(Code Generator)不再执行独立的条件检查，而是生成一个单一的模板化函数，仅需一次数据遍历即可同时评估这两个条件，并为所有相关数据类型编译对应的支持代码。
![关键帧](keyframes/part003_frame_00050416.jpg)
![关键帧](keyframes/part003_frame_00074066.jpg)

## 云环境中的遥测驱动优化
历史上，Vectorwise 早期版本等本地部署(On-Premises)系统只能依赖对常见查询模式的经验性推测，来决定需要融合哪些操作。云原生(Cloud-Native)架构从根本上改变了这一范式。Databricks 和 Redshift 等平台能够直接采集所有用户查询与系统遥测数据(Telemetry Data)。通过分析真实世界的工作负载(Workloads)，它们可以基于经验精准识别出哪些操作经常协同执行，并据此对系统进行定向优化。例如，Redshift 通过日志分析发现，用户意外地在读优化型(Read-Optimized)系统上运行了大量繁重的 `UPDATE` 查询，这一发现直接促成了针对性的性能改进。这种数据驱动的反馈循环(Data-Driven Feedback Loop)使现代数据库能够基于实际使用情况实现自我优化，而非依赖于预设的最坏情况假设。

## Spark 内部的动态内存管理
Photon 并非独立运行的引擎，而是作为嵌入式组件深度集成在 Spark 的 JVM 生态系统中。为了保持无缝集成并实现准确的内存核算(Memory Accounting)，Photon 避免了直接调用 `malloc`。相反，它将内存分配委托给 Spark 现有的 Java 内存管理器。JVM 负责分配内存块并执行原生内存跟踪，随后将安全指针(Safe Pointers)传递给基于 C++ 的 Photon 代码，使其自动继承 Spark 的性能剖析(Profiling)、监控报告及垃圾回收(Garbage Collection)基础设施。
![关键帧](keyframes/part003_frame_00171433.jpg)
由于数据湖仓(Lakehouse)工作负载经常查询对象存储(Object Storage)中非结构化或此前未见过的文件，系统极少能提前获取准确的选择性统计信息(Selectivity Statistics)。因此，算子在执行过程中可能突然需要超出初始预算的内存。为动态应对这一挑战，Photon 采用了集中式的内存压力管理(Memory Pressure Management)机制。当某个算子需要额外内存时，它会向 JVM 内存管理器发起请求。管理器随后会扫描所有活跃任务，定位可释放内存最多的任务，并触发其将数据溢写(Spill)至磁盘或释放已占用的内存。这种集中式溢写机制免去了每个独立算子自行实现磁盘溢写逻辑的负担，在灵活适应不可预测数据量的同时，有效防止了内存溢出(Out Of Memory, OOM)故障。

## Catalyst 优化器与物理计划到物理计划的转换
Spark SQL 传统上依赖 Catalyst 优化器(Catalyst Optimizer)，该优化器遵循级联(Cascades)风格的架构，依次经历逻辑计划优化与物理计划生成阶段。Photon 在此基础上引入了一个新颖的**物理计划到物理计划转换(Physical-to-Physical Transformation)**阶段。在 Catalyst 生成标准的 Spark SQL 物理计划后，系统会执行一次额外的自底向上遍历，专门用于注入 Photon 原生算子。
![关键帧](keyframes/part003_frame_00581550.jpg)
这种注入策略的首要目标是尽可能最小化 JVM 与 C++ 之间的上下文切换(Context Switching)。通过自底向上遍历执行计划（通常位于数据吞吐量最高的底层节点），Photon 会尽可能多地用其原生实现替换兼容的 Spark 算子。这确保了在 C++ 层能够实现长距离、不间断的流水线执行，显著降低了数据序列化与跨语言边界调用的开销。关键在于，这些替换决策均在查询优化阶段完成，而非运行时，从而使引擎能够在实际数据处理开始前就锁定最高效的执行策略。

## 实际的执行计划转换
考虑一个简单的查询计划：`File Scan -> Filter -> Shuffle -> Output`。在传统的 Spark 环境中，所有阶段均在 JVM 内执行。借助 Photon 的优化器遍历机制，系统会识别出由于受 I/O 限制，`File Scan` 阶段仍保留在 Java 层执行，但会立即为 `Filter` 和 `Shuffle` 阶段注入 Photon 原生算子。这种针对性替换在计算密集型步骤上最大化了原生向量化执行(Vectorized Execution)，同时保持了对 Spark 现有基础设施的完全兼容，在无需彻底重写引擎的前提下实现了显著的性能飞跃。
![关键帧](keyframes/part003_frame_00592333.jpg)