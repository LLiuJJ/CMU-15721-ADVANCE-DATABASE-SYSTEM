## 无共享架构的局限与共享磁盘架构的崛起
传统的无共享架构(Shared-Nothing Architecture)数据仓库面临显著的扩展瓶颈(Scalability Bottleneck)，因为扩容(Scaling Out)不仅需要增加计算节点，还会触发复杂且成本高昂的数据重分布(Data Redistribution)。20世纪末至21世纪初，行业开始转向共享磁盘架构(Shared-Disk Architecture)以突破这一限制。通过将计算与存储解耦(Compute-Storage Decoupling)，数据库系统将数据持久化(Data Persistence)任务卸载(Offload)至外部服务（如云对象存储(Cloud Object Storage)，例如 AWS S3）。这种架构分离使得计算层能够独立扩展(Scale Independently)并实现高度优化，而无需直接管理底层存储机制。
![关键帧](keyframes/part001_frame_00000000.jpg)
![关键帧](keyframes/part001_frame_00012466.jpg)

## 从专有数据仓库到开放湖仓系统
由 Snowflake 开创的第一代共享磁盘系统采用专有格式(Proprietary Format)将数据存储在对象存储中，导致这些数据仅能通过其特定的查询引擎(Query Engine)进行访问。现代演进版本则被冠以“湖仓一体”(Lakehouse)架构之名，全面转向开放标准(Open Standards)。用户不再受限于必须通过数据库专用接口摄入数据，而是可以直接将原始文件写入对象存储，并在中央目录(Central Catalog)中进行元数据注册。随后，查询引擎便能无缝读取这些文件，从而将数据湖(Data Lake)的灵活性与低成本优势，与传统数据仓库的数据治理能力、高性能及模式(Schema)控制能力完美结合。

## 市场繁荣与系统差异化
分析型数据库(Analytical Database)领域竞争异常激烈，这主要源于数据基础设施(Data Infrastructure)巨大的商业价值，以及云厂商生态系统带来的供应商锁定效应(Vendor Lock-in)。许多知名系统最初仅为科技巨头内部使用的数据平台（例如 Facebook 的 Presto、LinkedIn 的 Pinot），随后才剥离为独立的开源项目(Open Source Projects)。尽管底层的存储引擎(Storage Engine)与执行引擎(Execution Engine)在技术上已趋于同质化，成为基础标配，但真正的产品差异化(Differentiation)如今主要体现在用户体验、云原生集成能力，以及最关键的一点——查询优化器(Query Optimizer)的先进程度。尽管终端用户主要与顶层应用接口交互，但深入掌握这些底层内部机制对于系统开发者而言依然至关重要。
![关键帧](keyframes/part001_frame_00256383.jpg)

## 数据湖与湖仓一体：架构澄清
“数据湖”(Data Lake)本质上仅指原始的对象存储层，文件在此以无强制结构的方式存放，这将模式发现(Schema Discovery)与数据解析的负担全部推给了下游的查询端（即“读时模式” Schema-on-Read）。湖仓一体架构在此基础上引入了自动化目录管理、模式强制约束(Schema Enforcement)以及统一的管理控制平面。湖仓系统无需在不同服务间手动同步元数据，而是能够自动追踪文件位置、处理模式演进(Schema Evolution)，并基于底层原始文件提供无缝的、类传统数据库的查询体验。

## 核心 OLAP 架构与课程重点
在现代分析型数据管道(Analytical Data Pipeline)中，业务操作型(OLTP)数据库的数据通过 ETL 流程或流式中间件，被持续路由(Routing)至集中的对象存储层。目录服务(Catalog Service)持续追踪文件位置与数据模式，使解耦的查询引擎能够实时获取元数据(Metadata)，并直接下推查询至存储层执行分析操作。本学期的课程将重点聚焦于该查询引擎的设计与性能优化——它是连接元数据目录与底层存储的核心 OLAP 组件。这一架构范式(Architectural Paradigm)由 Google Dremel 等先驱系统开创，后经 Snowflake 成功商业化，已成为过去十年来业界的主流架构标准。
![关键帧](keyframes/part001_frame_00466633.jpg)

## 管理数据新鲜度与事务性更新
不可变对象存储(Immutable Object Storage)面临的一大核心挑战是如何处理数据的持续更新，同时避免产生数据过期(Stale Data)或存储碎片化问题。现代湖仓系统通过在底层文件存储之上实现符合 ACID 标准的事务处理能力(Transactional Capabilities)（涵盖插入、更新与删除操作）来解决此难题。具体而言，系统通常采用追加轻量级事务日志(Transaction Log)的方式来追踪数据变更，随后在后台异步运行压缩合并(Compaction)任务。该任务负责合并增量更新、清理过期记录，并重写优化后的列式文件格式(Columnar File Format)（如 Parquet）。这种日志结构合并(Log-Structured Merge)方法在保障查询数据新鲜度(Data Freshness)的同时，完整保留了现代分析型列存格式的高吞吐读取性能优势。
![关键帧](keyframes/part001_frame_00550533.jpg)