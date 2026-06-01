## 查询优化器：一项协作式例外
由于查询优化(Query Optimization)本身具有极高的复杂性，在一个学期内从零开始构建一个功能完整的优化器(Optimizer)是不太现实的。与项目其他分支中各小组独立开发相同组件并展开竞争的模式不同，负责查询优化器的团队将采用高度协作的工作模式。各小组之间不进行直接竞争，而是在同一个统一的代码库(Codebase)中紧密合作，共同开发不同的子系统(Subsystems)，以最大化地推进项目整体进度。
![关键帧](keyframes/part002_frame_00000000.jpg)

## 核心项目组件与架构
本学期的核心任务是构建新数据库系统的五个基础组件。**调度器(Scheduler)**将负责跨节点分配查询任务，并维持系统工作负载(Workload)的饱和度。**执行引擎(Execution Engine)**负责处理物理查询计划(Physical Query Plan)，为简化设计，该引擎默认在单节点上运行。**目录服务(Catalog Service)**将负责追踪数据库文件与数据模式(Schema)的元数据(Metadata)。**I/O 服务(IO Service)**将处理从本地磁盘(Local Disk)或云对象存储(Cloud Object Storage)中检索数据的任务，并实现一个统一的本地缓存(Unified Local Cache)，以尽可能降低高昂的远程数据获取开销。
![关键帧](keyframes/part002_frame_00022399.jpg)

## 优化器设计与开源灵感
查询优化器(Query Optimizer)将采用混合架构，结合基于规则的启发式算法(Rule-Based Heuristics)与基于代价的搜索(Cost-Based Search)，以生成最优执行计划(Execution Plan)。该项目将基于 OTB 进行开发，这是往届学生基于 Apache DataFusion 分支(Fork)而来的衍生项目。它保留了 DataFusion 的内部计划格式(Internal Plan Format)，但解耦了连接排序逻辑(Join Ordering Logic)。尽管鼓励学生从 Velox 和 DataFusion 等成熟的开源引擎中汲取架构灵感，但所有代码实现均须使用 Rust 语言从头编写，授课教师将针对合适的设计模式(Design Patterns)提供专业指导。
![关键帧](keyframes/part002_frame_00110666.jpg)
![关键帧](keyframes/part002_frame_00198416.jpg)

## 项目里程碑与渐进式反馈
为确保项目稳步推进并规避期末阶段的集成风险(Integration Risk)，整个项目周期围绕四个关键里程碑(Milestone)展开。项目将于 1 月 31 日以提交项目提案(Project Proposal)正式启动，随后安排两次月度课堂进度汇报。学期末将于 5 月举行最终演示与答辩(Final Presentation & Demo)，并强制要求在 GitHub 上提交完整代码。常规课堂时间将专门用于进度同步与问题排查(Debugging)，确保各小组开发步调一致，并清晰理解各自模块如何融入更广泛的系统架构(System Architecture)。
![关键帧](keyframes/part002_frame_00258049.jpg)

## 提案交付物与 API 标准化
初始提案需包含一份 1-2 页的 Markdown 格式设计文档(Design Document)、一次 5 分钟的课堂展示，以及一套全面的测试策略(Testing Strategy)。关键在于，各团队必须提交一份 **API 规范(API Specification)**，明确定义其组件将如何与其他模块进行交互，并将每个组件视为独立的内部微服务(Internal Microservice)。参与同一组件开发的竞争团队需各指派一名联络人负责跨组协调，确保双方在输入、输出、指令集及错误处理(Error Handling)标准上保持一致。这种对齐机制(Alignment Mechanism)保证了无论最终采纳哪一方的实现方案，都能无缝集成至最终系统中。
![关键帧](keyframes/part002_frame_00298483.jpg)

## API 复用与共享测试基础设施
为加速开发进程并与业界最佳实践接轨，强烈建议各团队直接采用成熟的开源 API 接口，而非自行设计专有接口(Proprietary Interface)。例如，目录服务应遵循 Apache Iceberg 标准，优化器需与 Apache Calcite 规范对齐，执行引擎则应兼容 Velox 或 DataFusion 的接口协议。对这些 API 进行标准化，不仅能显著降低初期的开发开销，还能使竞争双方复用同一套端到端测试框架(End-to-End Testing Framework)与验证脚本(Validation Scripts)。
![关键帧](keyframes/part002_frame_00492799.jpg)

## 进度跟踪与最终基准测试
在整个学期中，每次进度更新均需包含 5 分钟的演示环节，内容涵盖进度指标(Progress Metrics)、测试覆盖率(Test Coverage)数据、实时演示(Live Demo)以及持续迭代的设计文档。各团队联络人需提前协调并统一基准测试方法(Benchmarking Methodology)。在考试周期间，各小组将进行最终成果展示，并在 CMU 专用硬件集群上执行对比评估(Comparative Evaluation)。评估系统将严格测量各项指标的性能(Performance)、逻辑正确性(Correctness)以及 API 覆盖率(API Coverage)，以此决定哪一组的实现方案将被正式收录至本课程持续迭代的数据库系统中。
![关键帧](keyframes/part002_frame_00533000.jpg)
![关键帧](keyframes/part002_frame_00573133.jpg)