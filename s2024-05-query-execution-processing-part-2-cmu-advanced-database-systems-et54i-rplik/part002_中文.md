## 数据倾斜、哈希与混洗术语
在跨工作节点(Worker Node)分发数据时，系统必须谨慎管理数据倾斜(Data Skew)，以避免单个节点成为性能瓶颈。重分区算子(Repartition Operator)通常通过对连接键进行哈希(Hash)处理来路由元组(Tuple)，但如果数据分布高度倾斜，则可能需要额外的哈希轮次或特殊处理逻辑。具体实现细节在不同平台间差异显著。例如，BigQuery 专门设立了一项独立服务来处理数据混洗(Shuffle)，并利用定制硬件(Custom Hardware)最大化处理速度，同时为未来的自适应执行(Adaptive Execution)铺平道路。尽管具体架构各异，但核心概念是通用的：微软历来称其为“重分区(Repartition)”，而 Google、Databricks 以及更广泛的 MapReduce 生态系统则称之为“混洗(Shuffle)”。该机制泛化了更基础的数据路由模式(Data Routing Pattern)，允许系统根据实际数据量而非静态的查询优化器估算(Query Optimizer Estimation)，在流水线阶段(Pipeline Stage)之间动态调整工作节点的数量。
![关键帧](keyframes/part002_frame_00000000.jpg)

## 算子间并行与流水线并行
除了并行化单个算子(Operator)外，数据库系统还可以在单个查询计划(Query Plan)内并发执行多个流水线(Pipeline)，这种技术被称为算子间并行(Inter-operator Parallelism)或流水线并行(Pipeline Parallelism)。这种异步方法允许上游算子(Upstream Operator)持续产生数据流(Data Stream)，供下游算子(Downstream Operator)实时消费，从而无需等待整个阶段完成即可持续推进执行。
![关键帧](keyframes/part002_frame_00158799.jpg)

例如，在笛卡尔积(Cartesian Product)场景中，多个表扫描(Table Scan)可以并发运行，生成的元组(Tuple)一旦产出便会立即传递给连接算子(Join Operator)。这种生产者-消费者模型(Producer-Consumer Model)有效减少了线程空闲时间(Idle Time)并提升了执行速度，尤其是在数据依赖性(Data Dependency)较低的场景下。
![关键帧](keyframes/part002_frame_00235516.jpg)

尽管该模式并非适用于所有查询结构(Query Structure)，但流水线并行在 Apache Kafka、Spark Streaming 和 RisingWave 等流处理平台(Stream Processing Platform)中被广泛采用，以维持连续、低延迟的数据处理流程。该模型依赖交换算子(Exchange Operator)无缝桥接独立的流水线段(Pipeline Segment)，确保数据在并发执行阶段之间高效流转。
![关键帧](keyframes/part002_frame_00251049.jpg)

## OLAP 执行引擎的标准化与通用化
在现代数据库领域，OLAP 执行引擎(OLAP Execution Engine)已高度标准化。大多数系统均趋同于由 Snowflake 和 MonetDB 等早期先驱开创的向量化(Vectorized)、推式架构(Push-based Architecture)。开发者不再为每个新工作负载(Workload)或运行环境从头构建复杂的执行引擎，而是越来越多地依赖可组合的、开箱即用的执行库(Execution Library)。这些库提供了标准化且高度优化的算子实现(Operator Implementation)，使工程团队能够专注于领域特定功能(Domain-specific Features)，而无需重新发明基础的查询处理逻辑(Query Processing Logic)。这一转变有效解决了一个常见的行业痛点：内部代码碎片化与重复开发。例如，大型组织过去常常维护数十种实现细节各异的 `substring` 等基础函数，导致语义不一致(Semantic Inconsistency)、空值处理行为(Null Value Handling Behavior)各异，以及跨系统数据交换困难。
![关键帧](keyframes/part002_frame_00302933.jpg)

## Velox：架构与设计哲学
由 Meta 开发的 Velox 正是向可复用执行库(Reusable Execution Library)转型的典型代表。其创建初衷旨在整合内部碎片化的数据处理工作流，并将底层优化(Optimization)工作收敛至一个经过高度调优的代码库(Codebase)中。严格意义上，Velox 是一个后端执行引擎(Backend Execution Engine)；它刻意剥离了 SQL 解析器(SQL Parser)、查询优化器(Query Optimizer)、元数据目录(Metadata Catalog)或查询前端(Query Frontend)等高级组件。相反，它完全专注于底层算子执行(Low-level Operator Execution)、内存与线程管理(Memory & Thread Management)以及数据连接器(Data Connector)。当集成至更大的系统架构时，Velox 负责接收物理查询计划(Physical Query Plan)并生成最终结果，其利用预编译原语(Pre-compiled Primitives)与 C++ 代码生成(C++ Code Generation)技术来加速表达式求值(Expression Evaluation)。通过提供标准化、向量化(Vectorized)且推式(Push-based)的处理层(Processing Layer)，Velox 使开发团队能够在不牺牲执行性能(Execution Performance)的前提下，快速构建定制化的数据库系统。