## TPC 基准测试与性能认证的演进
事务处理性能委员会(Transaction Processing Performance Council, TPC)由 Jim Gray 等数据库先驱于 20 世纪 80 年代创立，旨在标准化联机分析处理(Online Analytical Processing, OLAP)基准测试，防止厂商发布不可复现且仅服务于自身利益的性能营销宣传。作为 TPC-H 的现代继任者，TPC-DS(TPC Decision Support Benchmark)是数据仓库工作负载的官方权威审计基准测试。2021 年，Databricks 获得了官方的 TPC-DS 认证，在 100TB 规模因子(Scale Factor)的测试中斩获榜首，超越了以往的领先者。该认证不仅严格评估原始查询吞吐量(Query Throughput)，还全面衡量云服务模型下的性价比(Price-Performance)，且必须经过严格的第三方审计(Third-party Audit)。
![关键帧](keyframes/part005_frame_00000000.jpg)
![关键帧](keyframes/part005_frame_00021949.jpg)
![关键帧](keyframes/part005_frame_00087833.jpg)
尽管具备显著的营销影响力，但官方 TPC 排名在现代企业采购决策(Enterprise Procurement Decisions)中的话语权已十分有限。虽然 TPC-H 和 TPC-DS 等基准测试在技术评估方面仍具参考价值，但严格且昂贵的审计流程对新兴初创公司而言往往不切实际。此外，在现代云环境中进行真正的“同等条件(Apple-to-Apple Comparison)”对比也愈发困难。云厂商高度抽象了底层硬件（例如虚拟化计算资源与特定 EC2 实例型号之间的差异），使得几乎无法验证不同云提供商是否采用了完全相同的硬件基线(Hardware Baseline)。因此，尽管 TPC 认证标志着深厚的工程成熟度(Engineering Maturity)，但在托管云数据平台(Managed Cloud Data Platform)时代，其历史上的市场主导地位已有所减弱。
![关键帧](keyframes/part005_frame_00099600.jpg)

## 开源 Spark 加速替代方案
与开源的 Apache Spark 本身不同，Databricks 的 Photon 运行时(Runtime)是一款专有的闭源引擎(Closed-source Engine)，仅通过付费订阅模式提供。然而，开源生态已孕育出多种替代性加速器(Accelerators)，能够实现相当的性能提升。知名项目包括 Apache Gluten（前身为 Gazelle），它支持 Spark 将完整的查询计划(Query Plan)卸载(Offload)至 Apache Arrow、FPGA、ClickHouse 或 Velox 等原生执行后端(Native Backend)。其他专用加速器则聚焦于特定硬件或执行框架，例如面向 GPU 加速数据科学工作负载(GPU Workloads)的 NVIDIA RAPIDS、基于 DataFusion 构建的 Blaze，以及近期推出的完全基于 DataFusion 实现的 Apache Arrow Comet。
![关键帧](keyframes/part005_frame_00290266.jpg)
与 Photon 的细粒度架构(Fine-grained Architecture)（即在算子级别(Operator Level)动态替换单个算子并精细管理 JVM 到 C++ 的数据转换）不同，这些开源替代方案通常采用更为粗粒度的策略。它们会拦截由 Spark Catalyst 优化器生成的完整物理查询计划(Physical Query Plan)，完全绕过 Spark 原生执行运行时(Native Execution Runtime)，并将计算工作负载直接路由(Routing)至外部原生引擎。用户虽仍通过标准的 Spark SQL 接口进行交互，但底层执行过程已被整体委托。目前，DataFusion 已成为诸多此类加速器的统一基石，作为 Spark 生态系统中高性能、可插拔的执行后端(Pluggable Execution Backend)，在业界获得了广泛采用。
![关键帧](keyframes/part005_frame_00359916.jpg)
![关键帧](keyframes/part005_frame_00405816.jpg)
![关键帧](keyframes/part005_frame_00425566.jpg)

## Delta Lake：解决湖仓统计信息难题
湖仓架构(Lakehouse Architecture)面临的一项根本性挑战是长期缺乏可靠的数据统计信息(Data Statistics)。当查询引擎直接在对象存储(Object Storage)上处理原始文件时，通常无法预先获知数据分布(Data Distribution)或列数据选择性(Column Selectivity)。虽然运行时自适应(Runtime Adaptivity)机制有助于缓解次优执行计划带来的问题，但唯有将自适应执行与准确的预置统计信息相结合，才能释放最佳性能。Delta Lake 通过在原始对象存储与计算引擎之间引入结构化的数据摄入(Data Ingestion)与事务层(Transaction Layer)，成功填补了这一空白。它摒弃了用户将任意文件直接随意写入对象存储（如 S3）的传统做法，转而通过严格的事务日志(Transaction Log)来统一管控数据摄入流程。
在后台，异步的文件合并(File Compaction)进程会将增量数据合并并转换为经过优化的 Parquet 文件。关键在于，在此列式转换阶段系统会计算详尽的统计信息，并将其持久化(Persistence)至集中式的元数据目录(Metastore Catalog)中。这使得查询优化器能够精准获取元数据(Metadata)（如列的最小值/最大值、空值计数及数据分布），从而真正实现基于成本的优化(Cost-Based Optimization, CBO)。通过在数据摄入阶段主动采集与维护统计信息，Delta Lake 成功将传统的非结构化数据湖转化为高度可查询且富含统计特征的数据环境。该理念继承了 Cloudera Impala 等早期系统的行业实践，并为现代湖仓架构的演进奠定了坚实基础。