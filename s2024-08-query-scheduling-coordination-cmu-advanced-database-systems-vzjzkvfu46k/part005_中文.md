## 流水线亲和性与工作窃取权衡
为最大化吞吐量(Throughput)并最小化跨 CPU 插槽延迟(Cross-Socket Latency)，调度器通过实施流水线亲和性(Pipeline Affinity)策略，将消耗中间结果的下游任务分配至与生成该结果相同的物理核心(Physical Core)。当工作节点(Worker)耗尽本地任务队列(Local Task Queue)时，为避免核心闲置(Core Idling)，它可能会跨非统一内存访问(Non-Uniform Memory Access, NUMA)区域执行工作窃取(Work-Stealing)以获取远程数据。系统设计将 NUMA 互联延迟(NUMA Interconnect Latency)视为必要的权衡(Trade-off)，其核心逻辑在于：远程内存访问(Remote Memory Access)的代价仍低于 CPU 周期停滞(CPU Cycle Stalling)所带来的机会成本(Opportunity Cost)。尽管理论上可通过精确的启发式算法(Heuristic Algorithm)确定最佳窃取时机（以避免原任务在数毫秒内即完成的情况），但维护此类微观预测所需的状态开销(State Maintenance Overhead)往往难以承受。在以十万条元组(Tuple)为执行粒度(Execution Granularity)、操作在毫秒级完成的场景下，激进的工作窃取策略仍是一种更简单且更有效的近似方案(Approximation)。
![关键帧](keyframes/part005_frame_00000000.jpg)

## 缓冲区管理与交换算子同步
Morsel 驱动架构(Morsel-Driven Architecture)通过将每个任务严格分配至互不重叠的数据块(Morsel)来实现数据隔离(Data Isolation)，从而消除了对部分结果进行同步的需求。然而，管理中间输出带来了新的挑战。工作节点将结果写入核心本地缓冲区(Core-Local Buffer)以维持缓存效率(Cache Efficiency)，但激进的工作窃取可能导致缓冲区耗尽(Buffer Exhaustion)。若在执行期间动态重平衡(Dynamic Rebalancing)或刷新(Flush)这些缓冲区，将产生过高的协调开销(Coordination Overhead)。因此，系统通常假设内存充足，或选择暂时挂起处理直至当前流水线阶段完成。此外，流水线边界由串行执行的交换算子(Exchange Operators)进行同步控制。与可高度并行的扫描(Scan)或连接(Join)任务不同，此类算子无法并行化；系统必须由单一线程聚合所有上游子流水线的输入，合并中间数据，并在进入下一调度阶段前将最终输出物化(Materialization)。

## Hyper Morsel 调度器的局限性
尽管初代 Hyper 调度器已被广泛采用，但仍存在若干架构局限性(Architectural Limitations)。任务与数据块(Morsel)之间严格的映射关系，结合固定的“单核单工作线程”(One-Worker-Per-Core)模型，严重阻碍了工作负载的动态重平衡(Dynamic Rebalancing)。若单个拖尾任务(Straggler Task)因数据倾斜(Data Skew)或高计算代价谓词(High-Cost Predicates)而滞后，其余所有工作节点均被迫空闲等待。此外，该模型隐含假设了每条元组的处理代价是均匀的，这一假设在处理复杂逻辑或高选择性过滤(High-Selectivity Filtering)条件时往往失效。同时，依赖共享的全局哈希表(Global Hash Table)进行任务排队会引发锁争用瓶颈(Contention Bottleneck)，难以在现代多路服务器(Multi-Socket Hardware)上实现良好扩展。最关键的是，该调度器缺乏服务质量(Quality of Service, QoS)控制机制。它以一种尽最大努力(Best-Effort)的自由调度模式运行，允许单个长时运行的分析查询独占所有工作线程。这将导致新抵达的短查询(Short Query)面临资源匮乏，并严重恶化系统的尾部延迟(Tail Latency)与用户主观响应体验。
![关键帧](keyframes/part005_frame_00291549.jpg)

## Umbra 调度器：时间片与优先级衰减
为突破上述局限，原 Hyper 团队独立设计并开发了其继任系统 Umbra。在企业并购导致 Hyper 知识产权转移至 Tableau/Salesforce 后，Umbra 系统从零开始重构，并对其底层执行模型(Execution Model)进行了根本性的重新设计。Umbra 摒弃了僵化的任务与数据块绑定机制，转而采用基于时间片(Time Slice)的执行单元(Execution Quanta)。在每个时间片内，任务可连续处理多个数据块，直至分配的时间预算(Time Budget)耗尽，随后主动让出 CPU 控制权。该方法实现了细粒度资源共享(Fine-Grained Resource Sharing)，同时避免了同步部分任务完成状态所带来的复杂性。至关重要的是，Umbra 引入了自动指数级优先级衰减(Exponential Priority Decay)机制：随着查询运行时间的延长，其调度优先级将呈指数级系统性降低。该机制确保了短小、交互式的查询(Interactive Queries)能够被快速交错调度，并优先于长时运行的后台负载执行。从而在维持高整体吞吐量(Overall Throughput)的同时，有效根治了服务质量层面的资源饥饿(Resource Starvation)问题。
![关键帧](keyframes/part005_frame_00364150.jpg)