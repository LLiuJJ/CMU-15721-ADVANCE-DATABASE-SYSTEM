## 云架构中的目录管理与弹性扩缩容
![关键帧](keyframes/part006_frame_00000000.jpg)
即使在现代的共享磁盘(Shared-Disk)云环境中，逻辑数据分区(Logical Data Partitioning)对于查询优化(Query Optimization)依然至关重要。中央目录服务(Centralized Catalog Service)会持续追踪哪些计算节点(Compute Node)负责对象存储(Object Storage)中的特定文件或数据范围(Data Range)。当集群容量进行弹性扩缩容(Elastic Scaling)时，系统仅需更新目录元数据以重新分配数据归属权(Data Ownership)，而无需在物理层面迁移底层数据文件。借助一致性哈希(Consistent Hashing)等技术，数据库能够在新节点间重新均衡工作负载(Workload Rebalancing)，同时最大程度减少数据重分布(Data Redistribution)开销，从而使传统的分布式数据库(Distributed Database)设计原则得以无缝适配云原生架构(Cloud-Native Architecture)。

## 向云原生共享磁盘模型的战略转变
![关键帧](keyframes/part006_frame_00041966.jpg)
尽管无共享架构(Shared-Nothing Architecture)凭借严格的数据局部性(Data Locality)能提供极致的底层性能，但在弹性扩缩容(Elastic Scaling)与运维开销(Operational Overhead)方面却面临显著瓶颈。计算与存储解耦(Compute-Storage Decoupling)所带来的工程灵活性与成本优势，已促使共享磁盘架构(Shared-Disk Architecture)成为现代联机分析处理(OLAP)系统的主流范式，甚至倒逼传统的无共享数据库供应商进行架构迁移。以 Amazon S3 为代表的云对象存储正以极低的成本持续迭代新功能（例如支持谓词下推(Predicate Pushdown)的 S3 Select）。尽管访问远程对象存储的网络延迟(Network Latency)显著高于本地直连磁盘，但现代数据库系统通过引入复杂的缓存层(Caching Layer)与精细的缓冲池管理(Buffer Pool Management)有效掩盖了该延迟开销，使得共享磁盘模型成为构建可扩展云分析(Scalable Cloud Analytics)平台最为务实的选择。

## 历史先例：云前的共享磁盘一体机
![关键帧](keyframes/part006_frame_00122683.jpg)
计算与存储解耦(Compute-Storage Decoupling)的理念最早可追溯至 20 世纪 80 年代，尽管早期实现往往受制于当时的网络带宽瓶颈(Network Bottleneck)。现代高性能本地部署(On-Premises)共享磁盘系统的典范当属 Oracle Exadata。这款企业级数据库一体机(Enterprise Database Appliance)集成了专用的计算与存储服务器机架，并通过 InfiniBand 或光纤通道(Fibre Channel)等高带宽、低延迟网络进行互联。其核心优势在于，存储节点(Storage Node)具备在本地执行谓词下推(Predicate Pushdown)的能力。这充分证明，只要借助专用硬件(Dedicated Hardware)与私有互连网络(Private Interconnect)消除网络延迟瓶颈，共享磁盘模型同样能展现出卓越的性能表现。

## 对象存储集成与 PAX 文件格式
![关键帧](keyframes/part006_frame_00305483.jpg)
从数据库内核的视角来看，云对象存储(Cloud Object Storage)的功能等效于远程磁盘(Remote Disk)，需通过云厂商特定的专属 API 而非传统的 POSIX 系统调用进行访问。现代 OLAP 系统广泛采用 Parquet 或 ORC 等列式存储格式(Columnar Format)，它们将底层数据组织为行组(Row Group)或数据块(Data Block)。在单个数据块内部，各列(Column)的值采用 PAX（Partition Attributes Across，跨分区属性）布局进行连续存储。这与早期将每列独立存储为单一文件的列式数据库截然不同，PAX 布局有效保留了元组(Tuple)在存储空间上的局部性(Locality)，极大提升了批处理(Batch Processing)的执行效率。此类格式还将关键元数据(Metadata)——包括压缩编解码器(Compression Codec)、列偏移量(Column Offset)及数据概要(Sketches)——集中存储于文件尾部(File Footer)。执行引擎(Execution Engine)会优先读取尾部信息以快速解析文件结构，避免全量扫描远程对象，随后将提取的统计信息(Statistics)回传至目录(Catalog)，以指导查询规划器(Query Planner)生成最优执行计划。

## 极致的底层优化：Yellowbrick 案例研究
![关键帧](keyframes/part006_frame_00403966.jpg)
当 Yellowbrick 将其高性能的本地无共享一体机(Shared-Nothing Appliance)迁移至公有云环境时，直接调用标准 S3 SDK 与常规 TCP/IP 网络栈遭遇了严重的延迟瓶颈(Latency Bottleneck)。为突破对象存储的性能限制，工程团队实施了极为激进的底层系统优化。他们彻底绕过标准云网络库，转而采用基于 Intel DPDK（数据平面开发套件）实现内核旁路(Kernel Bypass)的自定义用户态网络栈(User-Space Networking)，从而消除了昂贵的上下文切换(Context Switch)与内存拷贝(Memory Copy)开销。此外，团队弃用了传统 TCP 协议，改用定制化的轻量级 UDP 传输协议，并自主开发了专用的 PCIe 驱动程序以最大化硬件通信效率。这些突破常规的工程实践，辅以计算节点上积极的本地缓存策略(Aggressive Local Caching)，生动诠释了现代数据库系统如何跨越网络协议与操作系统内核的边界，在云环境中维系极高的数据吞吐量(High Throughput)。

## 课程路线图与自底向上的实现策略
![关键帧](keyframes/part006_frame_00505133.jpg)
![关键帧](keyframes/part006_frame_00512933.jpg)
![关键帧](keyframes/part006_frame_00519300.jpg)
![关键帧](keyframes/part006_frame_00525466.jpg)
![关键帧](keyframes/part006_frame_00531433.jpg)
![关键帧](keyframes/part006_frame_00537800.jpg)
![关键帧](keyframes/part006_frame_00544266.jpg)
本课程将调整初始的教学大纲，摒弃最初自上而下(Top-Down)的概览模式，转而采用自底向上(Bottom-Up)的构建策略来深入剖析数据库内核(Database Kernel)。我们将从底层物理存储格式(Physical Storage Format)讲起，逐步向上构建并集成执行引擎(Execution Engine)、任务调度器(Task Scheduler)与查询优化器(Query Optimizer)。课程内容将高度聚焦于基于通用 CPU 的查询执行(CPU-based Query Execution)，刻意暂不涉及 GPU 或 FPGA 等硬件专用加速器(Hardware Accelerator)，以确保学生能够集中精力掌握核心算法(Core Algorithms)与系统架构基础(System Architecture Foundations)。下一节课将深入拆解 Parquet 与 ORC 等行业标准文件格式的底层内部机制，随后重点考察旨在突破当前性能与结构局限的新一代存储格式。通过研读这些奠基性学术论文(Foundational Research Papers)与剖析主流实现策略，学员将全方位理解现代 OLAP 系统如何在海量数据规模下实现极致的高性能(High Performance)查询处理。