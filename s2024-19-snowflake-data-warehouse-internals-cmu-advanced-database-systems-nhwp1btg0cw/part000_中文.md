## 课程介绍与安排
![关键帧](keyframes/part000_frame_00000000.jpg)
欢迎来到卡内基梅隆大学的高级数据库系统(Advanced Database Systems)课程。今天，我们将继续探索那些实现了本学期所探讨架构概念的现实世界数据系统(Real-world Data Systems)。在深入技术讲解之前，先同步几项课程事务通知，以便大家明确后续的时间安排。

## 期末项目与考试指南
![关键帧](keyframes/part000_frame_00029116.jpg)
期末项目(Final Project)展示将在原定的期末考试时段进行。尽管官方时间表标注为上午 8:30，但实际活动将于上午 9:00 正式开始，现场将提供甜甜圈和贝果。纸质期末考试将于 4 月 24 日发放。本次考试将摒弃简单的选择题与纯记忆类考点，转而要求学生综合运用本学期所学概念，并将其应用于全新的数据系统场景。此举旨在考察学生能否深入理解各类架构理念如何融入更广泛的数据处理生态系统(Data Processing Ecosystem)中。考试允许使用 ChatGPT 等人工智能(AI)工具辅助作答，但务必仔细核验输出结果。强烈建议切勿盲目照搬 AI 生成的文本，因为考题中可能故意设置了虚构前提，且 AI 极易产生“幻觉”(Hallucination)。课程组将通过匿名追踪表格监控 AI 工具的使用情况，并与授课教师的预判数据进行比对。

## 回顾：Databricks 与向量化执行
![关键帧](keyframes/part000_frame_00009966.jpg)
在上一讲中，我们探讨了 Databricks(Databricks) 的 Photon 引擎(Photon Engine)。Photon 并非独立的数据库系统，而是集成于 Apache Spark(Apache Spark) 中的扩展组件。它作为一个高度优化、高度向量化(Vectorized)的 C++ 引擎运行，并通过 Java 原生接口(Java Native Interface, JNI)由 Spark SQL 的 Java 运行时环境进行调用。通过将计算密集型(Compute-intensive)的查询操作从 Java 虚拟机(Java Virtual Machine, JVM)卸载至原生 C++ 代码，Photon 为分析型工作负载(Analytical Workloads)带来了显著的性能飞跃。

## 现代云数据仓库简介
今天，我们的讨论将从组件级扩展过渡至完整且具备生产级(Production-grade)能力的云数据仓库(Cloud Data Warehouse)，重点聚焦于 Snowflake 与 Amazon Redshift(Amazon Redshift)。这些系统均为成熟的独立架构，其设计原则与 Google Dremel(Google Dremel) 和 Yellowbrick 等系统存在诸多共通之处。为准确理解它们的架构背景，我们首先需回顾此类系统问世前的数据库发展脉络。

## OLAP 系统的历史格局
20 世纪 00 年代，专业的在线分析处理(Online Analytical Processing, OLAP)系统开始崭露头角，其核心主要围绕列式存储架构(Columnar Storage Architecture)与数据压缩技术构建。随后，Vectorwise 率先基于此类存储引擎开创了向量化执行(Vectorized Execution)技术。该领域的代表性厂商包括 Vertica、Greenplum、MonetDB、Vectorwise 与 ParAccel。除 MonetDB 和 Vectorwise 外，这些平台大多基于 PostgreSQL 衍生而来；工程师们剥离了传统的行式存储引擎并进行重构，从而专门针对分析型查询模式进行优化。与此同时，Apache Hadoop 与早期数据湖(Data Lake)范式逐渐兴起，Hive、Presto、Impala 及 Stinger 等系统开始为分布式文件系统提供结构化查询语言(SQL)接口。在此阶段，行业标准部署模式严格依赖于本地部署(On-Premise Deployment)：供应商销售软件许可证(Software Licenses)，客户则需自行提供硬件并运维本地基础设施。

## 向云原生架构的转型
数据库行业在 2011 至 2013 年左右开始向云原生架构(Cloud-Native Architecture)转型。Google 于 2011 年发表的 Dremel 论文证实，分析引擎可直接对存储于云对象存储(Cloud Object Storage)中的文件进行查询，从而有效实现了计算与存储的解耦(Compute-Storage Decoupling)。Facebook 于 2012 年启动了 Presto 项目的研发。与此同时，Amazon Web Services (AWS) 于 2011 年获得了 ParAccel 的源代码授权，并在 2013 年正式推出 Amazon Redshift，略微早于 Snowflake 面世。ParAccel 随后陷入财务困境并被拆分收购，其原始技术栈基本停滞，最终沦为“僵尸数据库”(Zombie Database)。

## Snowflake 的创立
![关键帧](keyframes/part000_frame_00511133.jpg)
在此次技术转型期，硅谷风险投资机构 Sutter Hill Ventures 决定从零开始投资一家全新的云原生数据库初创企业。该公司采用了一种独特的“精英集结式创业模式”(Curated Startup Model，常被戏称为“组建全明星团队”)，成功招募了多位资深前 Oracle 工程师，以及 Vectorwise 的核心开发者 Marcin Zukowski。在充裕的资金支持与风投机构的战略指导下，该团队全面负责 Snowflake 的架构设计与研发工作。该系统随后在商业市场上取得了巨大成功。创始团队对数据库底层工程(Database Engineering)的极致专注令人瞩目；例如，Marcin 曾将 Snowflake 的标志纹在腿部以纪念公司首次公开募股(Initial Public Offering, IPO)，这一轶事在业内广为流传。

## Snowflake 的架构与云优先理念
![关键帧](keyframes/part000_frame_00522216.jpg)
Snowflake 是一款完全托管(Fully-managed)、纯云原生的在线分析处理(OLAP)数据库系统，其底层代码完全采用 C++ 从零编写。在本地部署仍被视为行业标准的时代，工程团队曾面临客户要求提供本地可下载版本的巨大压力。但团队坚决拒绝了这一要求，全面押注于云服务的普及趋势——随着整个行业向云端大规模迁移，这一“云优先”(Cloud-First)决策被证明极具前瞻性。与早期系统不同，Snowflake 采用了多集群共享数据架构(Multi-Cluster, Shared-Data Architecture)，并通过在计算节点上实施激进的计算侧缓存(Aggressive Compute-Side Caching)策略，成功与 Dremel 和 Spark SQL 形成差异化竞争。通过自主研发每一个核心组件，该团队在分布式云环境中实现了对查询执行引擎(Query Execution Engine)、存储层集成(Storage Integration)以及性能调优(Performance Tuning)的完全掌控。