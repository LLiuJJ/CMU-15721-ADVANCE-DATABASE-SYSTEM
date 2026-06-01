## 现代 OLAP 中的查询性能与故障概率
讨论首先将现代云数据仓库(Cloud Data Warehouse)与 Hadoop/MapReduce 等传统分布式系统(Distributed Systems)进行了对比。在早期架构中，查询往往在数千台廉价的“披萨盒”(Pizza-box)服务器上运行数小时，这使得硬件或网络故障在统计学上几乎不可避免。然而，当代系统借助高度优化的查询规划器(Query Planner)、流水线执行(Pipeline Execution)以及显著缩减的节点数量，即可高效处理海量数据集。由于现代查询执行速度极快，且运行于规模更小、可靠性更高的集群之上，执行期间节点发生故障的概率极低，从而大幅降低了对局部任务重试(Local Task Retry)等复杂、重量级容错机制(Fault Tolerance Mechanism)的依赖。

![关键帧](keyframes/part003_frame_00000000.jpg)

## 去中心化工作窃取机制
为在不依赖中央协调器(Central Coordinator)的前提下应对工作负载倾斜(Workload Skew)，该系统实现了一种去中心化(Decentralized)的工作窃取(Work-Stealing)模型。空闲的工作进程(Worker Process)无需被动等待调度器(Scheduler)重新分配任务，而是主动搜寻额外工作。当某个工作进程处理完当前查询阶段(Query Stage)分配的文件后，它会探查执行相同阶段的对等(Peer)节点，以识别进度落后的节点。为避免给落后节点增加额外负担，“窃取者”(Stealer)进程不会从对等节点的本地缓存中拉取(Pull)数据，而是直接指向 Amazon S3 获取所需数据。该策略确保慢节点不会因额外的网络 I/O(Network I/O)而过载，同时有效维持了集群的负载均衡(Load Balancing)。

![关键帧](keyframes/part003_frame_00082300.jpg)

## 弹性计算与跨租户资源利用
Snowflake 引入了一项名为“弹性计算”(Flexible Compute)的功能，旨在解决固定规模虚拟仓库(Virtual Warehouse)中资源配置不足的问题。当查询优化器(Query Optimizer)识别出特定查询阶段将成为性能瓶颈时，系统可动态将该执行计划(Execution Plan)片段转移至其他租户(Tenant)的空闲计算节点上执行。这种资源池化(Resource Pooling)机制对用户完全透明：借用方获得了更快的查询执行速度，出借方的预付费计算容量得到更高效的利用，而平台则避免了额外的基础设施投入。系统会智能拆分查询计划(Query Plan Splitting)（例如利用 `UNION ALL` 生成子计划），从而在专用硬件与共享借用的硬件之间合理分配工作负载(Workload)。

![关键帧](keyframes/part003_frame_00322033.jpg)

## 临时存储与中间结果的 S3 溢出机制
由于借用的计算节点具有临时性，且一旦所属租户恢复活动便会立即被抢占(Preemption)，因此系统无法安全地将中间结果(Intermediate Results)持久化于本地磁盘缓存中。为确保数据持久性并规避因缓存淘汰(Cache Eviction)引发的查询失败，借用节点生成的所有中间结果均会作为临时表分区(Temp Table Partitions)直接写入 Amazon S3。下游查询操作符(Downstream Query Operators)会无缝地将这些 S3 对象识别为标准数据源(Data Source)。这种溢出(Spilling)机制确保即便在激进的资源抢占策略下，临时性的资源共享也绝不会损害数据完整性(Data Integrity)或查询正确性。

![关键帧](keyframes/part003_frame_00414383.jpg)

## 查询优化、结果缓存与多租户隔离
该架构引入了诸如显式“连接过滤器”(Join Filter)等专用操作符，能够将布隆过滤器(Bloom Filter)从连接的构建侧(Build Side)推送至探测侧(Probe Side)，从而实现早期数据跳过(Early Data Skipping)。此外，系统可将已完成的查询片段(Query Fragments)缓存至 S3 并同步更新元数据目录(Metadata Catalog)，使后续相同查询能够直接绕过计算层，复用已物化的结果(Materialized Results)（默认不自动刷新）。针对跨租户计算共享(Cross-Tenant Compute Sharing)带来的安全挑战，Snowflake 实施了严格的隔离策略(Isolation Policy)：查询执行完毕后，工作进程会立即终止，从而彻底杜绝内存或上下文状态泄漏(State Leakage)的风险。作为一项仅运行经审计平台代码的全托管服务(Fully-Managed Service)，尽管底层物理硬件存在多路复用(Multiplexing)，系统依然维持着坚固的租户边界(Tenant Boundaries)。

![关键帧](keyframes/part003_frame_00513016.jpg)