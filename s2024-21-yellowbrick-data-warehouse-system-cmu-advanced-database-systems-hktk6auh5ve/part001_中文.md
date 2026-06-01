## 高层系统架构与核心特性
![关键帧](keyframes/part001_frame_00000000.jpg)
Yellowbrick 的云架构基于计算与存储分离(Disaggregated Compute and Storage)的共享磁盘(Shared-Disk)模型构建。该系统采用推送式(Push-Based)向量化查询处理，并高度依赖 LLVM 进行代码生成(Code Generation)，将查询计划动态编译为可执行的 C++ 代码。它具备类似 Snowflake 的客户端缓存(Client-Side Caching)功能，以及为优化数据导入而设计的行列混合存储(Hybrid Row-Column Storage)引擎。新写入的数据最初以行式格式(Row Format)处理，随后由后台进程将其重组为 PAX (Partition Attributes Across) 列式存储格式。该引擎支持排序合并连接(Sort-Merge Join)、哈希连接(Hash Join)和嵌套循环连接(Nested Loop Join)。尽管基于 PostgreSQL 9.5 分支构建，但该系统通过自定义优化阶段(Optimization Passes)替代了标准查询执行器，并利用深度的内核级系统工程优化来提升性能。

## 前端服务与元数据管理
![关键帧](keyframes/part001_frame_00093183.jpg)
系统的前端由一个集中式实例管理，作为工作节点(Worker Nodes)与辅助服务的主要控制平面(Control Plane)。该层保留了 PostgreSQL 前端模块，负责处理客户端连接、SQL 解析(SQL Parsing)、查询计划生成(Query Planning)与优化(Query Optimization)。同时，它沿用 PostgreSQL 的多版本并发控制(Multiversion Concurrency Control, MVCC)机制来管理事务隔离级别。元数据与数据分布映射通过 PostgreSQL 系统目录(System Catalog)进行维护；为避免频繁查询目录带来的性能开销，Yellowbrick 对元数据实施了积极缓存策略(Aggressive Caching Strategy)。

## 工作节点、调度与共享磁盘缓存模型
![关键帧](keyframes/part001_frame_00202649.jpg)
查询计划从 PostgreSQL 前端传递至集中式调度器(Centralized Scheduler)与编译服务(Compilation Service)。调度器以 100 毫秒为周期向轻量级工作节点分发任务(Task Dispatching)，以保持紧密的执行同步。工作节点本质上是轻量级的执行容器，负责运行已编译的代码、管理节点间的数据移动(Data Shuffling)，并维护本地 NVMe 缓存(NVMe Cache)。当发生缓存未命中时，工作节点会从云对象存储(Cloud Object Storage，如 Amazon S3)中拉取数据块，并采用近似最近最少使用(Approximate LRU) 淘汰策略(Eviction Policy)更新缓存。尽管该系统最初源于本地部署的无共享架构(Shared-Nothing Architecture)，但其云版本目前已演进为共享磁盘架构(Shared-Disk Architecture)。写回缓存(Write-Back Cache)负责将脏数据同步至对象存储；而对于中间查询产生的溢出数据(Spill Data)，系统则通过背压机制(Backpressure Mechanism)将其暂存于本地 NVMe 驱动器，而非直接回写至云端。

## Kubernetes 编排与硬件控制
![关键帧](keyframes/part001_frame_00415900.jpg)
Yellowbrick 的基础设施全面采用容器化(Containerization)技术，并通过 Kubernetes 进行统一编排(Orchestration)。所有核心组件均以长期运行(Long-Running)的微服务形式部署，由 Kubernetes 负责状态管理(State Management)、节点配置(Node Provisioning)及自动故障转移(Automatic Failover)。为简化运维操作，复杂的 Kubernetes 与 Helm 图表(Helm Charts)配置被封装在 `CREATE CLUSTER` 或 `CREATE INSTANCE` 等标准 SQL 命令之后，用户可直接通过 SQL 进行集群管理。值得注意的是，该系统在工作节点 Pod 与物理服务器(Physical Node)之间强制实施严格的一对一映射(Strict One-to-One Mapping)。每个 Pod 独占底层硬件资源并采用直接内存分配(Direct Memory Allocation)，有效避免了多 Pod 共享单节点时可能引发的启动延迟与资源争用(Resource Contention)。在系统内部，节点间的数据移动(Data Movement)与通信均依赖自定义设备驱动程序(Custom Device Drivers)与专有网络协议(Proprietary Protocol)，而外部客户端连接则继续兼容标准的 PostgreSQL ODBC/JDBC 线路协议(Wire Protocol)。

## 执行引擎与向量化至行式的转换
![关键帧](keyframes/part001_frame_00571283.jpg)
执行引擎采用推送式执行模型(Push-Based Execution Model)，专门针对列式数据结构(Columnar Data Structures)的向量化处理(Vectorized Processing)进行了深度优化。尽管数据扫描(Data Scanning)与初始过滤(Initial Filtering)在列式格式下能够高效执行，但系统会在处理复杂操作时动态切换数据表示形式。当数据流入连接(Join)等操作时，专用的“转置算子”(Transpose Operator)会将向量化的列式数据批次(Columnar Batches)实时转换为行式格式(Row Format)。这种提前物化策略(Eager Materialization)使 Yellowbrick 能够同时兼顾行列两种存储布局(Storage Layout)的优势，从而显著优化内存访问模式(Memory Access Patterns)并提升连接操作的执行效率。