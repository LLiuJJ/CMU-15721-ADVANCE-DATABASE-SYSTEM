## 拉式调度与局部性权衡
在拉式调度(Pull-based Scheduling)架构中，工作节点(Worker)会自主从全局队列中获取任务，这在负载均衡(Load Balancing)与数据局部性(Data Locality)之间引入了一个根本性的权衡。与严格的先进先出(First-In-First-Out, FIFO)队列不同，Hyper 等现代系统通常采用优先级队列(Priority Queue)或哈希表(Hash Table)，允许工作节点根据任务可用性而非僵化的顺序进行选择。虽然工作窃取(Work-Stealing)机制允许空闲节点从繁忙的队列中获取任务，但如果被窃取任务的数据位于远程节点(Remote Node)上，这可能会损害数据局部性。各系统的策略不尽相同：Umbra 和 SAP HANA 等系统严格执行局部性原则，绝不窃取远程任务；而其他系统则可能为了不让 CPU 核心空闲而选择执行非本地任务(Non-local Task)。这凸显了一个关键的设计分歧：是优先追求极致的数据局部性，还是最大化整体线程利用率(Thread Utilization)。
![关键帧](keyframes/part003_frame_00000000.jpg)

## 分布式缓存与跨节点干扰
在分布式共享磁盘(Shared-Disk)架构中，系统通常利用本地挂载的临时存储（如云实例上的 NVMe 驱动器）作为高速缓存(Cache)，以规避远程对象存储(Object Storage)的延迟与成本。更复杂的策略则探索利用其他计算节点(Compute Node)作为邻近缓存来满足数据请求。然而，系统通常会刻意避免直接从其他节点获取缓存数据。若某节点已出现性能降级(Performance Degradation)（这通常是触发工作窃取的原因），向其路由额外的跨节点数据请求(Cross-Node Data Request)只会进一步加剧其瓶颈。因此，调度器通常更倾向于直接从集中式存储(Centralized Storage)获取数据，而非冒着干扰已超负荷节点的风险。
![关键帧](keyframes/part003_frame_00143633.jpg)

## 数据分区与文件级放置策略
数据分区(Data Partitioning)通过哈希(Hash)、范围(Range)或轮询(Round-Robin)等策略将数据集分散至不同计算资源上，以平衡负载并最小化连接操作(Join Operations)时的数据洗牌(Shuffle)。在现代数据湖仓(Data Lakehouse)环境中，数据以不可变文件(Immutable Files)的形式存储于云端，复杂的重新分区(Repartitioning)往往不切实际。因此，系统通常默认在文件或行组(Row Group)级别采用简单的轮询分布策略。该方法大幅降低了目录元数据(Catalog Metadata)的开销，并简化了数据放置(Data Placement)逻辑。系统目录(System Catalog)会记录哪个工作节点或服务器负责处理哪个文件，从而使调度器能够将扫描任务(Scan Task)精准路由至指定的负责节点。这种结构化的分配方式在无需承担动态数据重平衡(Data Rebalancing)开销的前提下，最大化了本地缓存的利用率。

## 将物理计划转换为可执行任务
在明确工作节点分配、推/拉(Push/Pull)策略以及数据放置(Data Placement)策略后，系统必须将优化后的物理查询计划(Physical Query Plan)转换为可执行的任务。流水线阻断点(Pipeline Breakers)自然划定了这些任务的边界，将独立的执行阶段隔离开来。对于联机事务处理(Online Transaction Processing, OLTP)负载，此转换相对直接，因为查询通常仅包含依赖极少的单一流水线。调度器只需下发任务以立即执行，其核心职责侧重于并发控制(Concurrency Control)与资源仲裁(Resource Arbitration)，而非复杂的任务编排(Task Orchestration)。
![关键帧](keyframes/part003_frame_00464099.jpg)

## OLAP 复杂度与静态调度基础
由于存在深度的流水线依赖(Pipeline Dependencies)，联机分析处理(Online Analytical Processing, OLAP)负载的调度变得尤为复杂。为确保执行正确性并避免结果遗漏，调度器必须强制遵循严格的执行顺序：在所有上游依赖流水线完全执行完毕并物化其中间结果(Materialization of Intermediate Results)之前，下游流水线严禁启动。此类依赖链天然限制了完全并行化(Full Parallelization)的潜力，因而需要精细的协调。同样重要的是，调度决策必须基于物理查询计划(Physical Query Plan)而非逻辑查询计划(Logical Query Plan)，因为前者包含了将算子映射至具体硬件资源所必需的算法实现细节。应对此类复杂性的基础方法是静态调度(Static Scheduling)，即在查询生命周期的初始阶段，由优化器或调度器预先计算出完整的任务分配、资源划分及执行拓扑。
![关键帧](keyframes/part003_frame_00540366.jpg)