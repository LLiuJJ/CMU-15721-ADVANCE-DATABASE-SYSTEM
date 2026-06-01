## 数据传输的演进与智能存储下推
![关键帧](keyframes/part005_frame_00000000.jpg)
历史上，分布式数据库普遍倾向于采用“将计算推向数据(Move-Compute-to-Data)”模型，尤其是在无共享架构(Shared-Nothing Architecture)中。这是因为在过去，跨节点网络带宽(Network Bandwidth)往往是比本地磁盘 I/O 更为严重的性能瓶颈。现代云对象存储(Cloud Object Storage)通过将基础计算能力直接下沉至存储层，从根本上颠覆了这一传统范式。以 AWS S3 Select 为代表的服务，允许客户端在标准的 HTTP `GET` 请求中直接嵌入类 SQL 的过滤(Filter)与投影(Projection)谓词(Predicate)。得益于存储层原生支持对 CSV、JSON 及 Parquet 等格式的解析，系统能够仅在存储端处理并返回相关的数据子集(Data Subset)，从而大幅削减跨网络传输的数据量。尽管该机制在共享磁盘(Shared-Disk)环境中实现了高效的谓词下推(Predicate Pushdown)，但它并非在所有场景下均为最优解。当多个并发查询(Concurrent Query)携带不同的过滤条件反复访问同一数据块(Data Block)时，一次性拉取完整数据块并在计算节点本地缓存(Local Cache)，通常比频繁发起局部读取请求(Partial Read Request)更具成本效益。诸如 Amazon Redshift 等高级查询引擎会通过成本模型动态评估上述权衡(Trade-off)，以实现系统吞吐量(Throughput)最大化与存储 API 调用成本的最小化。
![关键帧](keyframes/part005_frame_00017049.jpg)

## 经典的无共享架构
![关键帧](keyframes/part005_frame_00067966.jpg)
无共享架构(Shared-Nothing Architecture)起源于 20 世纪 80 年代，并在随后的三十余年间主导了分布式数据库(Distributed Database)的设计范式。在该架构下，每个节点均作为独立且自包含(Self-Contained)的计算单元运行，独占专属的 CPU、内存及本地直连存储(Direct-Attached Storage)。数据通常依据范围分区(Range Partitioning)或哈希分区(Hash Partitioning)策略在集群内明确切分，节点间的所有通信均依赖 TCP 或 UDP 等标准网络协议(Network Protocol)。借助标准 POSIX API（如 `open`、`read`、`fstat`），数据库内核能够对底层物理文件保持直接且精细的控制，从而实现对 I/O 路径(I/O Path)与缓冲区管理(Buffer Management)的高精度优化。然而，计算与存储的紧密耦合(Tight Coupling)也带来了显著的运维复杂性(Operational Complexity)。水平扩缩容(Horizontal Scaling)需伴随谨慎的数据重分布(Data Rebalancing)；此外，若系统未显式管理数据复制(Replication)与一致性(Consistency)，单点节点故障将直接威胁整体数据的可用性(Data Availability)。

## 数据局部性与元数据管理的挑战
![关键帧](keyframes/part005_frame_00091983.jpg)
尽管 NFS 或 AFS 等传统分布式文件系统(Distributed File System)看似提供了更为简便的共享存储替代方案，但它们往往对数据库引擎隐藏了关键的数据局部性(Data Locality)信息。这类系统提供高度抽象的透明 POSIX 接口，致使数据库无法感知数据在物理存储介质上的确切位置及其底层磁盘分布(Disk Layout)。这种物理位置的不可见性严重阻碍了查询规划器(Query Planner)优化数据放置策略(Data Placement)、减少网络传输跳数(Network Hop)以及实施基于局部性感知(Locality-Aware)的智能调度。现代分布式数据库亟需显式且高保真(High-Fidelity)的元数据管理(Metadata Management)机制——通常依托专用的目录服务(Catalog Service)（如基于 FoundationDB 或 Cassandra 构建）——来精准追踪数据分区状态、副本分布及物理存储位置。相较于传统不透明的文件系统挂载(Mount)，云对象存储暴露了丰富的元数据接口，并对数据地理分布(Geographic Distribution)、生命周期策略(Lifecycle Policy)及一致性模型(Consistency Model)提供细粒度控制，赋能数据库引擎做出智能且具备成本感知(Cost-Aware)的调度决策。

## 共享磁盘系统中的计算与存储解耦
![关键帧](keyframes/part005_frame_00397516.jpg)
云原生架构(Cloud-Native Architecture)已全面转向共享磁盘模型(Shared-Disk Model)，该模型的核心特征在于实现计算层(Compute Layer)与持久化存储层(Persistent Storage Layer)的严格解耦(Decoupling)。在此架构设计中，计算节点彻底实现无状态化(Stateless)，并通过经过深度优化的用户空间 API(User-Space API)（而非传统的 FUSE 文件系统挂载）直接访问中心化的高持久性(High-Durability)对象存储。这种架构分离从根本上重塑了系统的弹性扩缩容(Elastic Scaling)与资源管理范式。
![关键帧](keyframes/part005_frame_00412399.jpg)
增加或削减计算资源变得即时且无缝(Seamless)，因为底层数据分区(Data Partition)无需进行任何物理重分布或额外复制。无论计算层实例的生命周期如何变化，核心数据始终安全、持久地驻留于独立的存储层中。企业可按需动态启动大规模计算集群以应对繁重的分析型工作负载(Analytical Workload)，并在业务低谷期完全释放计算资源。这种按实际处理时间计费(Pay-as-you-go)的模式，在保留全部历史数据的同时大幅降低了总体拥有成本(TCO)。这种极致的弹性彻底消除了传统无共享架构所必需的刚性容量规划(Rigid Capacity Planning)——因为在无共享系统中，节点下线本质上意味着其物理承载的数据随之不可用。
![关键帧](keyframes/part005_frame_00507199.jpg)

## 平滑扩缩容与查询容错
![关键帧](keyframes/part005_frame_00524033.jpg)
计算节点的无状态特性虽极大简化了底层基础设施管理，但也为查询执行(Query Execution)的韧性(Resilience)引入了新的工程考量。在计划内的扩缩容或系统维护期间，协调器(Coordinator)会优雅排空(Graceful Drain)活跃的工作负载：通知目标节点在安全下线前，完成当前队列中查询片段(Query Fragment)的处理，确保数据零丢失，并安全交接中间状态(Intermediate State)。然而，突发的硬件故障(Hardware Failure)将不可避免地导致正在执行的查询丢失其内存状态(Memory State)与中间结果(Intermediate Result)。尽管底层持久化数据在对象存储中依然绝对安全，但已丢失的计算成果必须通过重新执行(Re-execution)来恢复。现代查询调度器(Query Scheduler)通常采用任务重调度(Task Rescheduling)策略，将因故障孤立(unassigned)的计划片段重新分发至健康的计算节点。这种机制以牺牲单次查询执行的连续性为代价，换取了系统基础设施弹性、整体成本效益与运维简便性的大幅跃升。