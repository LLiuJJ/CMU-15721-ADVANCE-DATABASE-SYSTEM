## Velox 架构与外部集成
Velox 采用针对 Int32 和浮点数等特定数据类型预编译的原语(Pre-compiled Primitives)，以确保优化的执行路径(Execution Path)。尽管它与 Apache Arrow 紧密集成以实现向量化数据兼容(Vectorized Data Compatibility)，但也融入了轻量级的自适应查询优化(Adaptive Query Optimization)技术，使引擎能够在扫描(Scan)过程中动态调整表达式求值(Expression Evaluation)，而无需对整个查询计划(Query Plan)进行全局重新优化。目前，Velox 支持排序合并连接(Sort-Merge Join)与哈希连接(Hash Join)，但刻意剥离了查询优化器(Query Optimizer)、网络层(Network Layer)或专有数据格式(Proprietary Data Format)等上层组件。相反，它定位于数据湖架构(Data Lake Architecture)中运行，依赖外部连接器(External Connector)从 Amazon S3 等对象存储(Object Storage)中检索数据。类似于 Presto 或 Trino，这些连接器充当标准化 API，支持跨异构物理数据库与文件系统的联合查询(Federated Query)。此外，Velox 采用适配器(Adapter)来解码与编码多种列式存储格式(Columnar Storage Format)，包括 Parquet、ORC，以及 Meta 内部扩展的 ORC 变体（如 Dworf 和 Alpha）。
![关键帧](keyframes/part003_frame_00000000.jpg)

## 系统级内存管理
高性能执行引擎(High-Performance Execution Engine)需要对内存分配(Memory Allocation)进行精确控制，这也是 Velox 实现自定义内存池(Custom Memory Pool)而非依赖标准语言运行时(Language Runtime)或操作系统分配器(OS Allocator)的原因。频繁的 `malloc` 与 `free` 调用会引入显著的系统开销，即便是 Java 等托管语言(Managed Language)也难以避免不可预测的垃圾回收停顿(Garbage Collection Pause)。通过采用 Arena 风格的内存池(Arena-style Memory Pool)，引擎会预先分配大块连续内存(Contiguous Memory)并在内部进行统一管理。该机制独立于特定编程语言的底层内存模型(Memory Model)，使数据库能够完全掌控缓冲区(Buffer)的生命周期，有效减少内存碎片(Memory Fragmentation)，并确保基于 C++ 或 Rust 的实现均能获得确定性性能(Deterministic Performance)。
![关键帧](keyframes/part003_frame_00363816.jpg)

## 算子输出与元组物化策略
查询执行(Query Execution)中的一个关键设计决策是确定数据如何在算子(Operator)之间打包与传递。所选策略在很大程度上取决于底层的处理模型(Processing Model)与存储布局(Storage Layout)。在向量化列式系统(Vectorized Columnar System)中，引擎必须精确决定何时将离散的列数据组合并物化(Materialization)为完整的元组(Tuple)。这一时机选择直接影响内存带宽利用率(Memory Bandwidth Utilization)与 CPU 缓存效率(CPU Cache Efficiency)，因为在流水线(Pipeline)中传输冗余数据会引发严重的性能瓶颈。解决该问题的两种主流策略分别为早期物化(Early Materialization)与延迟物化(Late Materialization)。
![关键帧](keyframes/part003_frame_00467533.jpg)

## 早期物化机制
早期物化(Early Materialization)在查询计划的叶子节点(Leaf Node)层（通常处于初始表扫描(Table Scan)阶段）即组装完整的元组(Tuple)。数据一旦读取，所有所需列便会立即拼接，并将完整的行记录向上游传递至后续算子(Operator)。该方法逻辑简单直接，且与面向行的存储(Row-oriented Storage)或 PAX 布局(PAX Layout)天然契合，因为在此类布局中，相关数据在物理磁盘上已紧密聚集。其核心优势在于，下游算子(Downstream Operator)（如过滤器(Filter)或连接算子(Join Operator)）接收到的已是完全物化的行，无需再次发起额外数据请求。然而，针对宽表(Wide Table)或仅访问少量列的查询，该方法会因重建与传输非必要属性而浪费宝贵的内存带宽(Memory Bandwidth)与 CPU 周期(CPU Cycles)。
![关键帧](keyframes/part003_frame_00506683.jpg)

## 列式引擎中的延迟物化
诸如 Velox 等现代 OLAP 系统(Online Analytical Processing System)高度青睐延迟物化(Late Materialization)，旨在最大化列式存储(Columnar Storage)环境下的执行效率(Execution Efficiency)。在此模式下，算子(Operator)不会预先组装完整元组，而是仅传递当前计算所必需的最小属性集，例如连接键(Join Key)或元组 ID(Tuple ID)。这一设计显著减少了数据在执行流水线(Execution Pipeline)中的物理移动量。当后续算子最终需要访问额外列时，系统会利用先前传递的标识符(Identifier)，按需从原始列向量(Column Vector)或底层存储文件中提取所需数据。
![关键帧](keyframes/part003_frame_00522600.jpg)

## 权衡与执行灵活性
延迟物化模型(Late Materialization Model)在前期数据移动成本与延迟获取开销之间引入了典型的权衡(Trade-off)。例如，在仅依赖连接键完成连接操作(Join Operation)后，后续的投影操作(Projection)可能仍需生成日期等额外列。此时，执行引擎(Execution Engine)可动态检索这些缺失属性，并将其拼接至最终结果集(Result Set)。尽管该机制要求系统在执行期间持续维护对底层基础数据的引用与获取能力，但在分析型工作负载(Analytical Workload)场景下，它通常优于早期物化，因为它确保了仅有实际被访问的列才会消耗内存带宽。最终的最优策略选择取决于表宽度(Table Width)、查询选择率(Query Selectivity)，以及延迟获取列的额外开销与在全局计划中传输宽行(Wide Row)成本之间的相对比值。
![关键帧](keyframes/part003_frame_00556566.jpg)