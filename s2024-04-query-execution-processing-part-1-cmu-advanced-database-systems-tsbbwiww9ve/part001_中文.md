## 查询流水线与执行边界
流水线(Pipeline)定义了查询计划内的执行边界，决定了在遇到阻塞算子(Block Operator)之前能够连续处理的元组(Tuples)或数据批次(Batches)数量。以典型的哈希连接(Hash Join)为例，在开始探测(Probe Phase)之前，必须完整构建(Build)构建端(Build Side)的数据。过早进行探测可能导致漏报(False Negatives)，因为匹配的元组可能仍位于构建端尚未处理的数据中。为最大化数据复用(Data Reuse)并最小化时钟周期(Clock Cycles)开销，应将算子分组为紧密协作的流水线，并尽可能沿查询计划向上推送数据。将执行计划拆分为相互独立的流水线并物化(Materialize)中间结果，会因不必要的内存或磁盘写入引入显著开销，最终降低执行速度。
![关键帧](keyframes/part001_frame_00000000.jpg)

## 里程碑式的 2005 年 MonetDB/X100 论文
尽管这篇指定阅读论文(Required Reading Paper)发表于2005年，但其至今仍极具参考价值，因为它奠定了现代高性能联机分析处理(Online Analytical Processing, OLAP)系统的基础设计原则。X100 原型最初源自 MonetDB 项目，该研究证明了在算子之间逐元组传递数据(Tuple-at-a-Time)的传统模型会严重制约 CPU 性能。这项研究直接催生了商业数据库引擎 Vectorwise，该引擎随后被 Actian 收购，并演进为 Actian Vector 和 AdVelanche 等产品。该论文近期荣获 CIDR 时间检验奖(Test of Time Award)，进一步巩固了其作为数据库领域基石研究的地位。该论文作者此后主导或深刻影响了众多主流现代数据库系统(Database Systems)，包括 DuckDB、MonetDB、Snowflake 和 Apache Flink。
![关键帧](keyframes/part001_frame_00205883.jpg)

## 以 CPU 为中心的数据库架构
X100 论文的核心突破在于指出：传统数据库执行模型(Database Execution Model)与现代 CPU 架构从根本上存在不匹配(Mismatch)。该论文主张，系统设计不应仅侧重于代码可读性或常规软件工程实践，而应精心编排数据流(Data Flow)与指令序列(Instruction Sequences)，以显式匹配乱序执行(Out-of-Order Execution)与超标量处理器(Superscalar Processor)的硬件行为。尽管此举可能增加实现复杂度与维护成本，但能显著提升硬件执行指令的效率。通过优先顺应 CPU 的运行特性——例如保持可预测的内存访问模式(Memory Access Patterns)并最小化分支预测失败(Branch Misprediction)——数据库引擎能够实现仅依赖编译器优化(Compiler Optimizations)无法企及的显著性能提升。
![关键帧](keyframes/part001_frame_00495449.jpg)

## 面向查询执行的 CPU 架构基础
深入理解数据库性能需掌握现代 CPU 的基本工作原理。CPU 的运行机制类似于装配线，通过多个流水线阶段(Pipeline Stages)逐级处理指令。硬件设计的首要目标是确保持续为指令流水线提供待处理的工作负载(Workload)。当某条指令发生停滞(Stall)时（例如因缓存未命中(Cache Miss)需从主存(Main Memory)获取数据），CPU 会尝试调度执行其他独立指令，以避免时钟周期(Clock Cycles)空闲。在超标量架构(Superscalar Architecture)中，多个执行单元可并行运行，使处理器能够在单个时钟周期内调度(Dispatch)并完成多条指令。因此，数据库系统在设计时必须提供稳定且缓存友好(Cache-Friendly)的操作流，以充分释放这种并行处理与乱序执行(Out-of-Order Execution)的硬件潜力。
![关键帧](keyframes/part001_frame_00557783.jpg)