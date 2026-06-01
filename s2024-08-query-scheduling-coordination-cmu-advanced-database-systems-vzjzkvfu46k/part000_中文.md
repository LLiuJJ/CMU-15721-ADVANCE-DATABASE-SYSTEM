## 简介与课程背景
欢迎来到卡内基梅隆大学高级数据库系统(Advanced Database Systems)课程。本期讲座在演播室现场观众面前录制。在简要回顾此前涵盖的杂项话题及行为面试(Behavioral Interviews)相关内容后，我们将把焦点直接转向数据库内核。
![关键帧](keyframes/part000_frame_00000000.jpg)
今天的讲座将探讨如何将此前生成的优化查询计划(Optimized Query Plan)实际部署到系统中并启动执行。
![关键帧](keyframes/part000_frame_00009766.jpg)

## 查询执行：编译与向量化
近期的讲座涵盖了查询优化(Query Optimization)，特别探讨了并行哈希连接(Parallel Hash Join)等连接策略。在执行这些查询计划时，主要存在两种编译路径：转译(Transpilation，即源码到源码的编译，例如生成 C++ 代码)，以及借助 LLVM 等编译器框架，通过底层中间表示(Intermediate Representation, IR)进行即时编译(Just-In-Time Compilation, JIT)。
![关键帧](keyframes/part000_frame_00042183.jpg)
然而，过去十年间业界的普遍趋势更倾向于采用基于单指令多数据流(Single Instruction, Multiple Data, SIMD) 的向量化(Vectorization)技术，通常还会结合自动向量化(Auto-vectorization)与转译(Transpilation)技术。构建和维护 JIT 编译扩展的工程开销极高。正如 Databricks Photon 论文所指出的，与其依赖少数专家攻克复杂的 JIT 开发路径，更高效的常规做法是投入更广泛的工程资源来优化 SIMD 实现，从而获得与传统编译方式相媲美的性能。

## 查询调度中的核心术语
今天的讨论聚焦于调度(Scheduling)，即确定如何将查询计划分配至系统中的不同工作节点(Worker)。为确保表述清晰，我们必须精确定义以下核心术语：**查询计划(Query Plan)** 是由关系算子(Relational Operators)构成的有向无环图(Directed Acyclic Graph, DAG)。**算子实例(Operator Instance)** 表示将特定算子应用于某数据块（例如扫描单个行组(Row Group)）的执行实例。**任务(Task)** 是包含多个算子实例的计算单元，通常位于同一流水线(Pipeline)内，随后将被分发至工作节点执行。**任务集(Task Set)**（或工作集(Work Set)）指执行给定查询所需的全部任务集合。
![关键帧](keyframes/part000_frame_00141333.jpg)
通过在查询规划阶段识别流水线阻断点(Pipeline Breakers)，我们可以将连续的流水线转化为离散的任务，进而实现高效的分发与执行。

## 单节点调度与数据放置
调度(Scheduling)的核心挑战在于如何将任务分配给工作节点(Worker，广义上可指 CPU 核心(Core)、线程(Thread)、进程(Process)或物理节点(Node))，同时精确追踪数据流向：即输入数据的来源，以及中间结果应缓存于何处或路由至何处。尽管 PostgreSQL 等传统系统通常将调度委托给操作系统(Operating System)，但现代高性能数据库均会在内核层面自行实现调度管理。
![关键帧](keyframes/part000_frame_00199983.jpg)
我们将首先聚焦于单节点调度(Single-Node Scheduling)，因为无论系统架构如何，其核心问题始终一致：决定任务的执行位置及输出数据的处理方式。在分布式环境中，BigQuery/Dremel 等系统虽会在流水线阻断点后引入数据洗牌(Shuffle)阶段以重新平衡工作节点负载，但其底层调度逻辑与单节点实现一脉相承。

## 调度目标：吞吐量、公平性与延迟
构建高性能查询调度器(Query Scheduler)需围绕几个核心目标展开。首先，必须**最大化吞吐量(Throughput)**，确保系统持续消费输入并产出结果，避免出现 CPU 或 I/O 空闲。其次，需维持**公平性(Fairness)**，确保没有任何查询会因资源被长期剥夺而发生饥饿(Starvation)。即使长时间运行的查询(Long-running Queries)被分配了较低优先级，它最终也必须能够顺利完成。
![关键帧](keyframes/part000_frame_00231083.jpg)
第三，系统必须通过最小化尾部延迟(Tail Latency，如 P999)来保持**高响应性(Responsiveness)**。这对短查询(Short Queries)尤为关键：若 100 毫秒的查询退化为 1000 毫秒，用户会立刻察觉；而 10 分钟的查询延迟 20 秒则往往不易被感知。优先调度短查询能显著提升用户对系统响应速度的主观体验。最后，调度器本身必须具备**低开销(Low Overhead)**。耗费过多计算时间去求解理论上“最优”的调度方案无异于本末倒置；系统应将绝大部分计算周期留给实际的查询执行。

## 背景概念与后续主题
在深入具体实现之前，我们将简要回顾一些基础概念：明确**工作节点(Worker)**的界定与部署位置、理解**数据分区(Data Partitioning)**，以及管理非统一内存访问(Non-Uniform Memory Access, NUMA) 架构下的**本地内存(Local Memory)与远程内存(Remote Memory)**。正如 Morsel-Driven 论文所强调的，调度的核心目标是确保工作节点始终处理本地数据，从而避免代价高昂的远程内存访问或跨节点网络传输。随后，我们将剖析三种不同的调度实现方案：Hyper 论文中提出的静态碎块驱动(Static Morsel-Driven)方法、Umbra 数据库中更为动态且复杂的调度机制，以及 SAP HANA 团队采用的替代策略。这一递进式的探讨将为理解现代查询执行协调(Query Execution Coordination)机制奠定坚实的基础。