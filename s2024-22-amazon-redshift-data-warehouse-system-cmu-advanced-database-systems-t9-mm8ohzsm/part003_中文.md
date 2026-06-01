## 预编译原语与编译效率
![关键帧](keyframes/part003_frame_00000000.jpg)
为避免对每个查询代码片段执行完整的即时编译(Just-In-Time Compilation, JIT)，Redshift 刻意依赖精心优化的预编译基础算子(Pre-compiled Primitives)来处理表扫描(Table Scan)与数据过滤(Data Filtering)等底层操作。尽管运行时函数调用会带来 CPU 跳转与调用开销(CPU Jump/Call Overhead)，但向量化执行(Vectorized Execution)通过批量处理(Batch Processing)有效分摊了该成本，使其影响微乎其微。在亚马逊庞大的业务规模下，即使是对微小且重复的代码片段进行反复编译，累积起来也会造成巨大的计算资源浪费。亚马逊利用全系统遥测数据(System-wide Telemetry)与性能指标精准定位这些低效环节，并设计了一套专用缓存策略(Caching Strategy)，大幅降低了编译开销(Compilation Overhead)。此举不仅为客户提供了更低的查询延迟(Query Latency)，还为平台节省了数百万美元的基础设施成本(Infrastructure Costs)。
![关键帧](keyframes/part003_frame_00056333.jpg)

## 多版本缓存与硬件感知编译
为最大化预编译代码的利用率，Redshift 维护了一个精密的全局代码存储库(Global Code Repository)，其中同时保存了初始生成的源代码(Source Code)与多个已编译的机器码二进制文件(Machine Code Binaries)。鉴于 AWS 持续迭代推出新的实例类型(Instance Types)与底层 CPU 架构(Underlying CPU Architectures)，后台工作进程(Background Worker Processes)会自动检索缓存的源代码片段，并针对最新的硬件平台与 Redshift 软件版本进行重新编译(Recompilation)。这确保了在查询路由(Query Routing)阶段，系统能够精准匹配并调度专为当前执行硬件定制优化的二进制文件。这种云原生(Cloud-Native)能力使亚马逊能够在异构服务器集群(Heterogeneous Server Clusters)中持续维持峰值性能(Peak Performance)，同时免除了客户管理硬件相关编译状态(Hardware-Specific Compilation State)的负担。

## Aqua：基于 FPGA 的硬件加速
![关键帧](keyframes/part003_frame_00219233.jpg)
Aqua（高级查询加速器，Advanced Query Accelerator）于 2021 年左右正式推出，是一个专用的硬件加速层(Hardware Acceleration Layer)，架构上部署于计算工作节点(Compute Worker Nodes)与存储层(Storage Layer)之间。依托现场可编程门阵列(Field-Programmable Gate Array, FPGA)技术，Aqua 实现了深度的谓词下推(Predicate Pushdown)与聚合下推(Aggregation Pushdown)，使原始数据检索、条件过滤及初步聚合计算能够在数据返回至计算节点前高效完成。尤为关键的是，Aqua 作为全局多租户服务(Global Multi-Tenant Service)独立运行，与任何特定的 Redshift 集群解耦。这种架构分离(Architectural Decoupling)使用户能够弹性伸缩或调整计算节点规格，而无需担忧数据重分布(Data Redistribution)引发的性能瓶颈，因为该加速层在整个集群拓扑中始终保持全局可用。

## 向 Nitro 演进与存储层集成
尽管初期备受业界瞩目，Aqua 仍在 2022 年左右被悄然下线，且未发布正式的公开弃用公告(Public Deprecation Notice)。亚马逊将其核心的计算下推(Compute Pushdown)能力直接深度集成至 AWS Nitro 系统(AWS Nitro System)的定制芯片(Custom Silicon)与存储层基础设施(Storage Infrastructure)中。通过将硬件加速逻辑(Hardware Acceleration Logic)嵌入底层虚拟机监控程序(Hypervisor)及专用的网络与存储控制器卡(Dedicated Network and Storage Controller Cards)中，亚马逊彻底消除了对独立 FPGA 加速服务的依赖。该演进路径与 Amazon S3 Select 的设计理念一脉相承，即数据过滤与计算操作直接在对象存储层(Object Storage Layer)内部原生执行(Native Execution)，从而在保持甚至进一步提升下推性能(Pushdown Performance)的同时，大幅简化了整体系统架构。

## 查询优化器与统计信息管理
![关键帧](keyframes/part003_frame_00522666.jpg)
Redshift 的查询优化器(Query Optimizer)在底层架构上仍深度沿袭 PostgreSQL 的设计范式，首先应用启发式规则(Heuristic Rules)执行初始逻辑重写(Logical Rewriting)，随后通过基于成本的搜索算法(Cost-Based Search)确定最优的连接顺序(Join Ordering)。针对存储于 Redshift 托管存储(Redshift Managed Storage, RMS)中的数据，系统会自动化执行 `ANALYZE` 命令以采集详尽的列级统计信息(Column-Level Statistics)，并构建直接输入至基于成本的优化器(Cost-Based Optimizer)的统计模型(Statistical Models)。相对而言，针对直接查询原始 Amazon S3 数据的 Redshift Spectrum 查询，系统缺乏深度的表级统计信息(Table-Level Statistics)。为此，Redshift 转而采用从 Apache Parquet 文件头(Apache Parquet File Headers)中动态提取元数据(Metadata)的策略，并在计算层缓存区域映射(Zone Maps，即记录列的最小值与最大值范围)。该机制实现了高效的过滤器下推(Filter Pushdown)，有效规避了不必要的全文件扫描(Full File Scans)。