## 历史先驱：Kudu 与早期湖仓表
早在 Delta Lake 确立行业标准之前，Cloudera 开发的 Kudu（最初于 2015 年左右作为 Impala 的扩展组件推出）便已开创了类似的架构范式。Kudu 支持在分布式文件系统(Distributed File System)上执行高效的增量更新(Incremental Updates)，并在数据摄入(Data Ingestion)阶段自动采集列级统计信息。这些统计信息通过集中式元数据层(Metadata Layer)暴露给查询优化器(Query Optimizer)，使得 Impala 能够避免全量扫描原始文件，从而执行高效查询。市场主导权从昔日的大数据领军厂商 Cloudera 迅速向 Databricks 转移，这一趋势不仅凸显了数据库基础设施(Database Infrastructure)的迭代速度，也再次印证了构建灵活、开放架构在行业竞争中的核心价值。
![关键帧](keyframes/part006_frame_00000000.jpg)

## 现代表格式：Apache Hudi 与 Iceberg
Kudu 所确立的“数据摄入与目录管理(Catalog Management)”范式，最终演化为当今主流的开源表格式(Table Format)，例如 Apache Hudi（2016 年诞生于 Uber）与 Apache Iceberg（2017 年由 Netflix 开源）。这两套系统均构建了结构化的数据摄入层：通过事务日志(Transaction Log)路由数据变更，利用后台合并压缩(Compaction)将其转化为对象存储(Object Storage)中高度优化的列式文件(Columnar Files)，并持续以最新的统计信息与文件布局(File Layout)同步更新元数据目录(Metastore Catalog)。查询引擎随后直接消费这些目录元数据(Catalog Metadata)，以生成并规划出高效的执行计划(Execution Plan)。这种计算与存储解耦的架构(Decoupled Architecture)已成为现代数据平台的基石。如今，Snowflake 与 Amazon Redshift 等云数据仓库(Cloud Data Warehouse)均已原生集成对 Iceberg 的支持，允许直接查询外部数据湖，从而彻底打破了早期要求所有数据必须沉淀于专有托管存储(Proprietary Managed Storage)中的生态壁垒。
![关键帧](keyframes/part006_frame_00120416.jpg)

## 架构权衡：JVM 遗留与原生执行
Photon 引擎的研发历程，深刻揭示了在现代化遗留生态系统(Legacy Ecosystem)时所必须做出的务实工程权衡(Engineering Trade-offs)。Databricks 并未选择直接废弃已高度成熟、且支撑着海量创收级企业工作负载(Enterprise Workloads)的基于 JVM 的 Spark 运行时(JVM Runtime)，而是转而采用了一种细粒度的混合架构策略(Fine-grained Hybrid Architecture)。该策略通过谨慎注入原生 C++ 算子(Native C++ Operators)，并精细化管控 JVM 至原生代码(Native Code)的数据转换链路，在确保现有应用零侵入的前提下，实现了计算性能的极大化。然而，若以从零构建(Greenfield Development)全新架构的视角审视，JVM 平台已不再是底层执行引擎的最优解。尽管现代 JVM 配备了先进的垃圾回收器(Garbage Collector)以及适用于高频交易(High-Frequency Trading)等特定场景的专有运行时调优特性，但在超大规模分布式部署中，其固有的内存开销(Memory Overhead)与潜在的商业许可成本，促使 C++、Rust 或 Zig 等现代系统级语言(System-level Languages)成为新一代数据库引擎构建的首选基石。
![关键帧](keyframes/part006_frame_00409766.jpg)

## 总结与课程预告
本次探讨的尾声，将 Spark 的架构演进轨迹与下一代云原生数据平台(Cloud-Native Data Platform)的发展脉络进行了有机衔接。在即将展开的 Snowflake 专题课程中，我们将深入剖析其高层架构设计(High-level Design)如何与 Vectorwise 和 X100 等经典历史系统产生技术共鸣，这也深刻映射出其核心团队深厚的学术底蕴与工程积淀。与此同时，Snowflake 针对全托管式(Fully Managed)、服务导向的云部署环境对这些核心理念进行了深度适配。通过将自研的专有存储管理(Proprietary Storage Management)与开放表格式标准(Open Table Format Standards)相融合，Snowflake 能够高效承载现代复杂的分析型与事务型混合工作负载(Hybrid Workloads)。