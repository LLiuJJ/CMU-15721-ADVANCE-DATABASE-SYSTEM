## 课程简介与数据编码回顾
欢迎来到卡内基梅隆大学(Carnegie Mellon University)高级数据库系统(Advanced Database Systems)课程。今天，我们将从理论数据结构转向查询的实际执行。在过去的两节课中，我们重点探讨了数据布局(Data Layout)以及编码方案(Encoding Schemes)的设计，旨在最小化从磁盘或远程对象存储(Remote Object Storage)加载至内存的数据量。通过高效的数据编码，我们力求直接在压缩或编码格式下处理数据，并借助游程编码(Run-Length Encoding, RLE)等技术显著缓解 I/O 瓶颈(I/O Bottleneck)。
![关键帧](keyframes/part000_frame_00000000.jpg)
![关键帧](keyframes/part000_frame_00009766.jpg)
![关键帧](keyframes/part000_frame_00026883.jpg)

## OLAP 查询执行与优化“技巧库”
在联机分析处理(Online Analytical Processing, OLAP)范式中，查询执行主要依赖于顺序扫描(Sequential Scan)，而非索引查找或位图索引(Bitmap Index)。尽管 B+树(B+ Tree)等结构仍可能用于定位单条记录，但核心工作负载(Workload)在于大规模数据块的扫描。我们将深入探讨一套全面的“优化技巧库”以加速此类扫描。尽管区域映射(Zone Maps)能够有效跳过无关数据块，但我们的重点将放在更宏观的优化策略上，例如任务与查询并行化(Task and Query Parallelism)、高效连接算法(Join Algorithms)以及代码特化(Code Specialization)。这些概念将在本学期内逐步引入。我们的核心目标是构建能够充分榨取硬件性能的数据库系统，通过优化的乘法叠加效应实现更快的执行速度。然而，每项技术都伴随着工程实现与维护成本上的权衡(Trade-offs)。例如，尽管即时查询编译(Just-In-Time, JIT Query Compilation)能带来卓越的性能，但其复杂性往往促使生产系统转而采用预编译的向量化原语(Vectorized Primitives)。
![关键帧](keyframes/part000_frame_00075750.jpg)

## 核心优化目标与编译器考量
为最大化查询性能，我们确立了三项核心工程目标。其一是减少执行特定工作负载(Workload)所需的 CPU 指令总数。尽管编译器可通过 `-O2` 等编译标志(Compilation Flags)提供辅助，但生产级数据库通常避免使用 `-O3`。因为过于激进的优化可能会重排指令，从而损害程序正确性；Linux 之父 Linus Torvalds 曾就系统稳定性对此提出过著名警告。因此，数据库引擎的设计宗旨是从算法底层减少指令使用量，而非依赖激进的编译器优化标志。本次讲座将为任务并行化(Task Parallelism)奠定基础，并确立支持未来向量化执行(Vectorized Execution)所必需的查询处理模型(Query Processing Model)。
![关键帧](keyframes/part000_frame_00134133.jpg)
![关键帧](keyframes/part000_frame_00290349.jpg)

## 最小化每条指令周期（CPI）与对齐 CPU 架构
第二个目标是降低每条指令周期数(Cycles Per Instruction, CPI)。这要求确保数据操作数(Data Operands)驻留在 CPU 最快的存储层级中，即寄存器(Registers)或 L1/L2 缓存(L1/L2 Cache)。通过最大化算子(Operators)与执行计划内的数据局部性(Data Locality)，我们可以极大程度地避免代价高昂的缓存未命中(Cache Misses)。在此过程中，流水线技术(Pipelining)、算子融合(Operator Fusion)以及基于推送的执行模型(Push-based Execution Model)至关重要。实现与 CPU 架构的最佳对齐颇具挑战，因为人类易读的代码逻辑往往与现代乱序执行(Out-of-Order Execution)、超标量处理器(Superscalar Processors)的实际执行模式相冲突。编译器无法自动弥合这一鸿沟；我们必须刻意设计契合硬件预期的算法。第三个目标——并行化(Parallelism)——则顺应了业界向多核 CPU、异构/混合架构(Heterogeneous/Hybrid Architectures)以及大规模 GPU 集群演进的趋势，旨在将计算能力扩展至多线程与多节点环境。
![关键帧](keyframes/part000_frame_00344533.jpg)

## 查询处理术语：计划、算子与任务
为确保表述清晰，我们统一了查询执行相关的术语。查询计划(Query Plan)以算子的有向无环图(Directed Acyclic Graph, DAG)形式组织。一条 SQL 查询会被转化为物理计划(Physical Plan)，其中包含扫描(Scan)、投影(Projection)、过滤(Filter)和连接(Join)等操作。随后，系统会将其实例化为“算子实例”(Operator Instances)，以代表具体的执行调用。这种区分使得并行化成为可能；例如，针对一张超大表的单个扫描算子可被拆分为多个实例，每个实例负责处理不同的行组(Row Groups)或文件。“任务”(Task)将一个或多个算子实例打包为一个可调度单元，其形态通常类似于流水线，扫描与过滤等操作在此被融合执行。最后，“任务集”(Task Set)则是这些任务的集合，已准备就绪以便在系统资源中进行调度与执行。
![关键帧](keyframes/part000_frame_00516200.jpg)