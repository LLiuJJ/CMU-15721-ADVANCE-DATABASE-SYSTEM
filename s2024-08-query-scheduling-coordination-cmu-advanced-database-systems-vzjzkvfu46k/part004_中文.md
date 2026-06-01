## 静态调度的局限性
静态调度(Static Scheduling)涉及将任务预先刚性分配给工作节点(Worker)或 CPU 核心(CPU Core)，通常以优化初始数据局部性(Initial Data Locality)为目标。虽然实现简单，但该方法完全缺乏运行时(Runtime)适应性。其根本缺陷在于“拖尾(Straggler)问题”：当数据分布倾斜(Data Skew)，或某些查询谓词(Query Predicates)具有高度选择性但计算代价高昂时，某个工作节点处理其分配数据块的速度可能会显著慢于其他节点。由于下游流水线阶段(Downstream Pipeline Stages)通常依赖于上游任务的完成，这单个缓慢的工作节点会导致所有其他核心陷入停滞，从而形成严重的性能瓶颈，彻底抵消并行执行(Parallel Execution)带来的收益。
![关键帧](keyframes/part004_frame_00000000.jpg)

## 引入 Morsel 驱动并行
为缓解运行时差异和拖尾问题，Morsel 驱动并行(Morsel-Driven Parallelism)模型引入了动态的即时任务分配机制。在该架构中，数据表被水平切分为易于管理的数据块，称为“Morsel”（概念上等同于现代数据库中的行组(Row Group)）。系统采用拉式(Pull-Based)分配模型，每个核心运行一个专用的工作线程(Worker Thread)，自主从中央队列中拉取任务。工作线程会优先处理位于其本地非统一内存访问(NUMA)区域内的 Morsel 任务，以最小化跨 CPU 插槽互联(Cross-Socket Interconnect)的延迟。然而，若本地无可用任务，工作线程将主动执行全局队列中的下一个可用任务，而不受数据局部性约束，从而确保核心保持忙碌状态，并有效掩盖缓慢拖尾任务(Straggler Tasks)带来的影响。这一极具影响力的设计已被业界广泛采用，尤为显著的是，它构成了 DuckDB 查询执行(Query Execution)的基石。
![关键帧](keyframes/part004_frame_00099016.jpg)

## 协作式调度与扩展性挑战
Morsel 驱动执行依赖于协作式调度(Cooperative Scheduling)范式：系统中不设专用的调度分发线程。相反，确定下一个最佳任务的逻辑被分散至所有工作节点中，它们通过轮询方式共同访问一个全局任务队列(Global Task Queue)。虽然这简化了系统架构并分散了决策压力，但原始研究对保护共享全局哈希表(Shared Global Hash Table)或队列所需的同步开销(Synchronization Overhead)并未深入探讨。随着现代多路服务器(Multi-Socket Servers)的核心数量扩展至数十甚至数百个，对该中心化数据结构的访问争用(Access Contention)成为了关键的性能瓶颈。面向大规模硬件设计的系统明确指出了这一局限性，证明若不引入过高的锁开销(Lock Overhead)或缓存行颠簸(Cache-Line Bouncing)代价，单个全局队列便无法高效扩展至高核心数环境。
![关键帧](keyframes/part004_frame_00209416.jpg)
![关键帧](keyframes/part004_frame_00237383.jpg)

## 优化 Morsel 粒度以提升性能
确定 Morsel 的最佳粒度(Morsel Granularity)是一项关键的系统工程权衡(System Engineering Trade-off)。若 Morsel 过小，工作节点将耗费过多 CPU 周期在任务队列争用上，导致调度开销(Scheduling Overhead)反客为主成为主要瓶颈。反之，若 Morsel 过大，系统实际上将退化回静态调度模式，单个过大的数据块极易制造出拖尾任务，进而导致整个流水线停滞。奠基性的 Hyper 论文指出，每个 Morsel 包含约 10 万条元组(Tuple)是该架构的最佳平衡点，有效兼顾了并行度(Parallelism)与队列争用。Peloton 和 NoisePage 等其他研究型系统则尝试了不同的粒度策略，例如固定 10MB 的数据块或仅含 1000 条元组的微块，以优化内存布局(Memory Layout)、指针运算(Pointer Arithmetic)及缓存行为(Cache Behavior)。
![关键帧](keyframes/part004_frame_00379633.jpg)

## 执行流程、局部性与流水线依赖
在执行期间，每个工作节点利用本地内存(Local Memory)处理其分配的 Morsel，并将所有中间结果写入核心本地缓冲区(Core-Local Buffer)。该设计严格避免向远程 NUMA 内存(Remote NUMA Memory)写入数据，从而保护了关键内存带宽，并最小化了跨 CPU 互联延迟。然而，严格的流水线依赖(Pipeline Dependencies)仍会强制引入同步屏障(Synchronization Barrier)：当下游阶段需要上游结果时，若前置流水线尚未完全物化中间结果，工作节点必须进入空闲状态。试图通过切换至其他并发查询(Concurrent Queries)来缓解此类空闲时间会引入新的复杂性，因为共享缓冲池(Shared Buffer Pool)可能面临严重争用，或需执行代价高昂的数据驱逐(Data Eviction)与上下文交换。归根结底，尽管 Morsel 驱动执行最大化了硬件利用率(Hardware Utilization)并缓解了数据倾斜问题，但其性能根本上仍受制于复杂查询计划中固有的顺序依赖关系(Sequential Dependencies)。
![关键帧](keyframes/part004_frame_00452883.jpg)
![关键帧](keyframes/part004_frame_00593333.jpg)