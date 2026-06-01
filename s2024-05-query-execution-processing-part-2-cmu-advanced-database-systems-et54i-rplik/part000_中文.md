## 查询处理模型简介与回顾
我们的高等数据库系统课程能否像在有现场观众的演播室中录制那样进行呢？今天，我们将承接上节课的内容，探讨查询的实际执行方式。在上一讲中，我们重点探讨了查询处理模型(Query Processing Model)，即如何确定查询计划(Query Plan)中各个算子(Operator)之间传递数据的形态与体量。我们还研究了用于触发算子执行的控制机制，重点区分了推式(Push-based)与拉式(Pull-based)模型。
![关键帧](keyframes/part000_frame_00000000.jpg)

尽管 BusTub 采用拉式(Pull-based)架构，但本课程主要聚焦于推式(Push-based)系统。我们还讲解了如何表示谓词(Predicate)和过滤器(Filter)的输出，例如使用选择向量(Selection Vector)、位置列表(Position List)或位图(Bitmap)来指示哪些元组(Tuple)满足特定条件。
![关键帧](keyframes/part000_frame_00009766.jpg)

## 数据表示与推式系统架构
在本学期的后续内容中，我们将假设所讨论的概念系统采用向量化(Vectorized)与推式(Push-based)架构。如今，向量化已成为几乎所有现代数据库系统的标准配置。尽管许多系统仍依赖拉式(Pull-based)执行，但推式模型具有显著优势：集中式调度器(Centralized Scheduler)能够完全掌控任务的执行时机。这种集中式方法实现了细粒度(Fine-grained)管理，例如在内存受限时，只需停止任务调度即可暂停执行。同样，若某个算子需要发起阻塞式网络调用(Blocking Network Call)，调度器可暂时挂起该任务，待资源可用时再恢复执行，从而提供比拉式方案更强的控制力。
![关键帧](keyframes/part000_frame_00027899.jpg)

## 课程路线图与后续主题
今天的课程将继续探讨查询的并行执行(Parallel Execution)，探索不同类型的并行机制以及允许多个算子实例(Operator Instance)并发运行的系统架构。从下周开始，我们将深入探讨查询计划(Query Plan)中单个算子的并行化，研究算子的输出格式，并讨论诸如延迟物化(Late Materialization)等数据访问策略。我们将讲解算子之间的数据流动方式，并以 Apache Arrow 作为系统的标准数据格式。此外，我们还将预览表达式求值(Expression Evaluation)与自适应查询执行(Adaptive Query Execution)，重点介绍基于 Veloc 论文的谓词求值自适应性。在本学期后半段，我们将探讨查询优化(Query Optimization)以及查询自适应性的复杂性，这类特性必须从零开始进行架构设计，而非事后修补。
![关键帧](keyframes/part000_frame_00143833.jpg)

## 现代硬件下并行执行的动机
并行执行(Parallel Execution)的核心思想非常直观：我们希望数据库系统能够通过同时运行多个操作来充分利用可用硬件。现代系统早已不再依赖传统的单核机器。即便是单路 CPU(Single-socket CPU)如今也拥有数十个核心，而多路配置(Multi-socket Configuration)在功能上已等效于分布式系统(Distributed System)。在这种环境下，我们必须设计能够将单个查询分发至多台机器以并行化各项操作的系统。无论系统采用多线程、多进程还是多节点架构，高层级的并行概念始终保持一致。尽管网络通信会引入延迟，但现代机架内网络(Intra-rack Network)已高度优化，且并行抽象在不同部署规模下保持一致。
![关键帧](keyframes/part000_frame_00226666.jpg)

## 查询间并行与调度策略
从高层级来看，并行主要分为两种类型：查询间并行(Inter-query Parallelism)与查询内并行(Intra-query Parallelism)。查询间并行侧重于在系统内并发执行多个查询，由协调器(Coordinator)或调度器(Scheduler)决定执行时机与资源分配。许多现有系统采用基础的先来先服务策略(First-Come, First-Served, FCFS)，根据查询到达时间和资源需求分配优先级，后续查询则排队等待可用资源。企业级系统通常会实施更复杂的调度策略，例如为特定用户或连接标识(Connection String)设置优先级。无论采用何种策略，其核心目标都是保持所有系统资源处于活跃状态。例如，若某个查询需要执行单节点交换操作(Single-node Exchange Operation)，调度器可将其他空闲核心分配给不同任务。我们将在本学期后期更详细地探讨调度算法(Scheduling Algorithm)与数据分发策略(Data Distribution Strategy)。
![关键帧](keyframes/part000_frame_00356999.jpg)

## 查询内并行与声明式 SQL 的优势
查询内并行(Intra-query Parallelism)涉及将单个查询分发至多个工作节点(Worker Node)与计算资源。这充分凸显了 SQL 等声明式语言(Declarative Language)的优势：用户无需了解底层架构，无论查询实际跨越单个节点还是上百个节点。相同的 SQL 查询可被系统自动编译为不同的查询计划片段(Query Plan Fragment)并进行分发，无需用户手动重写或针对本地环境进行调试。查询内并行主要有两种实现方式：水平并行(Horizontal Parallelism)与垂直/异步 I/O 并行(Vertical/Asynchronous I/O Parallelism)。尽管两者可结合使用，但在现代执行引擎(Execution Engine)中，水平并行更为普遍。
![关键帧](keyframes/part000_frame_00471700.jpg)

## 水平（算子内）并行
水平并行(Horizontal Parallelism)，亦称算子内并行(Intra-operator Parallelism)，其核心是将查询计划中的单个算子拆分为多个独立实例。每个实例对各自的输入执行完全相同的计算逻辑，但处理的是数据的不同分区(Data Partition)。根据具体实现，算子可能显式感知自身的并行状态，也可能对此无感知。当算子对并行无感知时，系统通常会在执行计划中引入交换算子(Exchange Operator)，以收集并合并各并行任务的结果。每个标准算子均配有对应的并行化版本，交换机制确保了结果的无缝整合，使下游算子(Downstream Operator)无需额外管理分布式执行状态。
![关键帧](keyframes/part000_frame_00576550.jpg)