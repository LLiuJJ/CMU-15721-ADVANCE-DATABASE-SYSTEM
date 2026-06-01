## 核心架构与存储优化
现代湖仓一体（Lakehouse）与 OLAP 引擎通常需要支持共享磁盘聚合存储（Shared-Disk Aggregated Storage）与因子化查询处理（Factorized Query Processing）。尽管学术文献往往略去深入的内部实现细节，但已确认 BigQuery 采用 TriX 作为其执行引擎（Execution Engine）。在通用存储方面，Google 依赖其专有的 Capacitor 列式存储格式（Columnar Storage Format），其工作原理与 Apache Parquet 类似。该格式融合了区域映射（Zone Maps）、谓词过滤器（Filters）、数据标记（Ticks）以及列级压缩（Columnar Compression）等成熟的优化技术。其索引机制仅限于倒排搜索索引（Inverted Search Index），主要用于 `LIKE` 与正则表达式（Regex）匹配等字符串操作；而连接（Join）操作则仅支持哈希连接（Hash Join）。查询优化采用启发式规则与基于代价的优化器（Cost-Based Optimizer, CBO）相结合的混合策略。然而，由于底层统计信息常存在缺失或偏差，系统高度依赖自适应查询执行（Adaptive Query Execution），能够在运行时根据实际观测的数据特征动态调整执行计划（Execution Plan）。这一架构代表了数据库系统从单节点执行向分布式多阶段处理（Distributed Multi-Stage Processing）的基础性转变。
![关键帧](keyframes/part002_frame_00000000.jpg)

## 查询执行、确定性与元数据管理
查询提交后，系统会将其转换为逻辑计划（Logical Plan），并划分为多个执行阶段（Execution Stages），这些阶段大致对应于数据处理流水线（Data Pipeline）。每个阶段包含多个并行任务（Parallel Tasks），分布于各个工作节点（Worker Nodes）上。架构上的一项严格要求是：每个任务必须具备确定性（Determinism）与幂等性（Idempotency）。这确保了只要输入一致，重试任务总能生成完全相同的结果。因此，系统可以安全地终止、重启或在不同工作节点间迁移慢任务（Straggling Tasks），而不会破坏数据完整性。即便是 `random()` 这类非确定性函数，系统也强制要求其无论由哪个工作节点执行，都必须生成相同的值序列。为防止元数据目录（Catalog）过载，根节点或协调器（Coordinator）会集中负责任务分发与元数据检索。协调器不会允许成千上万的工作节点独立查询目录，而是预先批量处理元数据请求，并将文件路径与数据模式（Schema）详情直接嵌入逻辑计划中。工作节点接收到的任务包是完全自包含的（Self-Contained），从而彻底消除了运行时动态查询目录的需求。
![关键帧](keyframes/part002_frame_00104550.jpg)

## 内存 Shuffle 架构与数据流
每个工作节点均配备专用的本地内存与磁盘。当任务处理的数据量超过内存阈值时，系统允许将数据溢出（Spillover）至本地磁盘。然而，阶段间通信摒弃了传统的工作节点直传方式，转而采用集中式的内存 Shuffle 服务（In-Memory Shuffle Service）。这一可水平扩展的键值存储（Key-Value Store）充当了中间缓冲区的角色：上游阶段的工作节点会将中间结果直接写入 Shuffle 服务，而非直接传递给下游节点。Shuffle 服务节点会将数据量与分布情况的元数据回传至协调器。借助这种实时可见性（Real-Time Visibility），协调器能够动态计算出下一阶段所需的最优工作节点数量，并向调度器（Scheduler）请求资源分配。新启动的工作节点随后将直接从 Shuffle 服务中拉取分配给它们的数据分区（Data Partitions）。这种解耦模型确保了各执行阶段之间绝不进行点对点（Peer-to-Peer, P2P）通信，所有中间数据均完全通过 Shuffle 基础设施进行路由。
![关键帧](keyframes/part002_frame_00251349.jpg)
![关键帧](keyframes/part002_frame_00275899.jpg)

## 设计原理与对比分析
明确采用强制性且可水平扩展的 Shuffle 服务，有效解决了关键的性能瓶颈、容错机制与资源管理挑战。通过阶段解耦，上游工作节点一旦将数据刷新（Flush）至 Shuffle 服务，便可立即被终止或重新分配任务，无需持续存活以等待下游消费者（Downstream Consumers）。若下游工作节点发生故障，系统可无缝地从 Shuffle 服务重新拉取数据，无需依赖可能已被回收的上游节点。尽管该服务专为在内存中处理海量数据集而设计，但它同样支持将溢出数据无缝写入 Google 的分布式文件系统（GFS/Colossus）。这一设计与默认将中间结果写入本地磁盘的 Hadoop，以及将 Shuffle 数据保留在计算节点上的 Spark 形成了鲜明对比。尽管相较于直接数据路由（Direct Data Routing），引入 Shuffle 会产生一定开销，但 BigQuery 与 Dremel 强制在每个执行阶段使用该机制，以确保协调器能够精确掌握中间数据的规模。这使得系统能够实现精准的即时资源弹性扩缩容（Elastic Scaling）与强大的自适应执行规划。在大规模分析工作负载（Analytical Workloads）场景下，该架构带来的整体收益远超其引入的延迟成本。
![关键帧](keyframes/part002_frame_00323966.jpg)