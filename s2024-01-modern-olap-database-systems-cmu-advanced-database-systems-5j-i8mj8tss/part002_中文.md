## 现代数据湖基础设施与目录系统
![关键帧](keyframes/part002_frame_00000000.jpg)
数据通常以 Parquet 等格式存储在数据湖(Data Lake)中，或托管于 Databricks 的 Delta Lake 等专业系统内。这些系统接收 Parquet 文件并将其写入对象存储(Object Storage)，同时更新目录(Catalog)以追踪数据的最新版本。它们能够自动处理模式(Schema)变更，并提供额外的基础设施，确保对象存储层能够维护精确的内容元数据(Metadata)。业内知名的实现包括 Delta Lake、Uber 开源的 Hudi、Netflix 开源的 Iceberg、Snowflake 的混合表(Hybrid Tables)，以及谷歌内部正在逐步对外公开的 Napa 系统。

## 课程重点与核心设计原则
尽管目录与增量更新(Delta-Update)架构在工业界至关重要，但本学期课程不会对其进行深入探讨。课程的核心重心仍将聚焦于查询执行(Query Execution)的性能优化，增量更新机制将在后续章节中展开。不过，近期研究论文中提出的一些基础性洞见将贯穿始终，用以指导本课程中的系统设计决策。牢记这些设计原则，不仅对构建数据库内核(Database Kernel)大有裨益，也能在未来的工程实践中，帮助大家为特定场景甄选出最合适的联机分析处理(OLAP)系统。
![关键帧](keyframes/part002_frame_00030000.jpg)
![关键帧](keyframes/part002_frame_00090000.jpg)

## 趋势一：超越传统 SQL 的多样化工作负载
![关键帧](keyframes/part002_frame_00120000.jpg)
现代企业越来越多地需要处理传统结构化查询语言(SQL)之外的多样化工作负载。尽管 SQL 仍是数据查询的首选语言，但在实际生产环境中，数据科学家(Data Scientist)频繁在交互式 Notebook 环境中，依托 PyTorch 与 TensorFlow 等框架运行机器学习(Machine Learning)任务。与标准的 OLAP 查询相比，这类机器学习工作负载呈现出截然不同的数据访问模式(Data Access Pattern)。例如，相较于传统的 JDBC/ODBC 驱动进行结果集转换，工程师们更倾向于采用 Apache Arrow 等内存原生格式(Memory-Native Format)进行批量数据导出。尽管机器学习与 SQL 工作负载在后端查询规划器(Query Planner)与执行引擎(Execution Engine)层面存在共性，但前端 API 与数据访问接口的设计必须具备足够的灵活性，以适配这些日益多样化的需求。
![关键帧](keyframes/part002_frame_00150000.jpg)

## 趋势二：共享磁盘架构与目录治理
向共享磁盘(Shared-Disk)架构的演进（即将数据直接存放于 Amazon S3 等对象存储中）削弱了对数据摄入(Data Ingestion)与访问的强管控。由于数据可直接写入云存储，而无需经由传统的数据库前端网关，过去严格的管控流程已不再是硬性约束。然而，维护一个健壮的目录(Catalog)系统对于追踪模式(Schema)演进、管理文件版本以及清晰掌握底层存储中的实际数据资产依然至关重要。这种以目录为中心(Catalog-Centric)的治理范式，在系统灵活性与必要的数据管控之间取得了良好平衡。
![关键帧](keyframes/part002_frame_00210000.jpg)

## 趋势三：处理非结构化与半结构化数据
![关键帧](keyframes/part002_frame_00270000.jpg)
现代企业数据中有相当大比例属于非结构化数据(Unstructured Data)（如图像、视频文件）或半结构化数据(Semi-structured Data)（如 JSON 日志、包含文本嵌入 Embedding 的混合元组）。虽然非结构化数据通常需借助 Transformer 等外部机器学习(Machine Learning)框架来提取结构化特征，但半结构化数据则对数据库系统提出了直接的技术挑战。即便数据已存储在 Parquet 或 ORC 等经过高度优化的文件格式中，系统仍须高效地解析并查询其中的嵌套(Nested)或不规则结构。各大厂商的应对策略各有侧重：Snowflake 通常倾向于在数据摄入(Data Ingestion)阶段进行显式的列展开与物化，而 Databricks 则可能采用动态(On-the-fly)解析技术。本课程后续将对这些核心策略展开深入探讨。
![关键帧](keyframes/part002_frame_00300000.jpg)

## 从单体架构向可组合数据库服务的转变
![关键帧](keyframes/part002_frame_00330000.jpg)
在过去十年间，行业趋势已逐渐从单体(Monolithic)数据库系统，转向由独立服务或组件拼装而成的可组合(Composable)架构。只要各组件均能提供稳定且定义清晰的应用程序接口(API)，它们便可实现独立开发、无缝替换或采用成熟的第三方方案。这与 PostgreSQL、DuckDB 或 SQLite 等传统数据库形成鲜明对比，后者的核心模块（如查询解析器(Parser)、优化器(Optimizer)、目录(Catalog)等）通常由单一团队紧密耦合、从零构建。可组合架构虽然带来了更高的系统灵活性与更短的开发迭代周期，但同时也引入了全新的工程权衡(Engineering Trade-offs)。
![关键帧](keyframes/part002_frame_00360000.jpg)

## 组件集成与抽象层成本的挑战
![关键帧](keyframes/part002_frame_00390000.jpg)
集成独立组件必然因引入额外的抽象层(Abstraction Layer)而产生性能开销。当系统的全局状态与优化知识被割裂至不同服务时，整体性能优化往往只能妥协于各组件间兼容性的最低标准。构建或集成独立的查询优化器(Query Optimizer)等复杂组件（如 Apache Calcite、ORCA）向来以技术门槛极高著称，因为它们内部携带的隐式假设(Implicit Assumptions)极少能与外部系统完美契合。如何使这些服务实现高效通信——突破基础的 RESTful 或 HTTP 协议限制——并在此过程中规避协议不匹配与性能瓶颈，依然是业界一项重大且极具挑战性的系统工程难题。

## 中间表示与数据类型同步
可组合数据库设计的一个核心环节，是明确定义用于组件间信息交换的中间表示(Intermediate Representation, IR)。例如，当查询优化器(Query Optimizer)将生成的执行计划(Execution Plan)传递给任务调度器(Scheduler)时，双方必须对查询结构与操作符的内部表示达成严格一致。此外，数据类型(Data Type)定义必须在服务边界处进行强制同步；若独立开发的组件在整数位宽或类型定义上出现偏差，极易引发静默失败(Silent Failure)或严重的性能衰退。建立统一的中间表示规范与严格的类型契约(Type Contract)，是利用模块化组件构建连贯、高性能数据库系统的基石。
![关键帧](keyframes/part002_frame_00570000.jpg)