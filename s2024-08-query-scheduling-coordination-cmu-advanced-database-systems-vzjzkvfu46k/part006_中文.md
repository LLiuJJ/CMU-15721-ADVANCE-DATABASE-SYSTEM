## 动态 Morsel 大小调整与执行时间估算
调度器无法预先预测任务的执行时间，因此依赖于持续的运行时监控(Runtime Monitoring)来自适应地管理工作负载的粒度(Workload Granularity)。系统并非在内存中物理复制或调整数据大小，而是根据观测到的性能表现，动态调整*后续* Morsel 的逻辑边界(Logical Boundaries)。若任务持续以快于约 1 毫秒的目标阈值完成，调度器会以指数方式(Exponential Manner)增大后续分配任务的 Morsel 尺寸。反之，若执行时间过长，则会收紧边界（即减小数据分块大小）。这种持续校准实现了一项关键平衡：它在最大限度降低频繁查询全局队列带来的状态维护开销(State Maintenance Overhead)的同时，有效避免了因任务块过大或分布不均而引发的拖尾问题(Straggler Problem)。
![关键帧](keyframes/part006_frame_00000000.jpg)
![关键帧](keyframes/part006_frame_00030000.jpg)

## 优先级衰减与基于水位线的公平调度
为防止长时间运行的分析查询(Analytical Query)独占系统资源并降低短小交互式工作负载(Interactive Workload)的响应速度，调度器实现了自动指数级优先级衰减(Exponential Priority Decay)机制。查询在初始阶段享有高优先级，但随着执行时间的延长，其调度优先级会系统性地降低。为确保公平性(Fairness)并防止长时运行任务陷入饥饿(Starvation)，系统采用了结合全局水位线(Global Watermark)（或称“通行证”(Pass)）计数器的步长调度算法(Stride Scheduling Algorithm)。新抵达的短查询会被分配特定的水位线值，使其能够在执行队列中获得优先调度；而长时间运行的查询则会通过累积调度信用(Scheduling Credit)，逐步提升其水位线。一旦其水位线超过全局阈值(Global Threshold)，系统便会为其分配 CPU 时间片(CPU Time Slice)。该机制确保了长查询最终得以完成，同时不会阻塞新进入的交互式查询。

## 去中心化通知与基于槽位的任务队列
Umbra 调度器的一项重大架构转变是摒弃了集中式且依赖重度锁机制(Heavyweight Locking)的全局任务队列(Global Task Queue)。取而代之的是，系统采用了一个包含任务元数据指针(Metadata Pointer)的全局槽位(Slot)数组，并结合每个工作节点(Worker)上的线程局部存储(Thread-Local Storage, TLS)来跟踪本地状态。工作节点通过维护特定的位掩码(Bitmask)（如 `active`、`change` 和 `return` 掩码），即可在本地监控队列状态，而无需执行持续的同步操作(Synchronization Operations)。
![关键帧](keyframes/part006_frame_00420000.jpg)
当一个任务集(Task Set)完成且新任务集入队时，系统通过在系统互联总线(Interconnect Bus)上执行原子比较并交换(Compare-And-Swap, CAS)操作，翻转其他工作节点返回掩码(`return` Bitmask)中的特定位。此举有效避免了昂贵的推式协调开销(Push-Based Coordination Overhead)。
![关键帧](keyframes/part006_frame_00450000.jpg)
该机制的作用类似于轻量级的消息公告板(Message Bulletin Board)：它仅广播队列状态已发生变更，而无需传输实际的任务数据(Task Data)，亦无需依赖重量级的同步闩锁(Synchronization Latches)。
![关键帧](keyframes/part006_frame_00480000.jpg)
当工作线程(Worker Thread)下一次进入调度例程(Scheduling Routine)时，它会检查本地掩码(Local Bitmask)；一旦检测到变更通知，便自主从全局槽位数组中拉取(Pull)更新后的任务信息。
![关键帧](keyframes/part006_frame_00510000.jpg)
通过限制活跃槽位(Active Slots)的数量（例如上限为 128 个），并依赖原子位翻转操作(Atomic Bit-Flip Operations)替代完整的数据传输或锁争用(Lock Contention)，该架构在多核系统(Multi-Core Systems)上实现了极高的可扩展性(Scalability)。
![关键帧](keyframes/part006_frame_00540000.jpg)
这种去中心化的拉式方法(Decentralized Pull-Based Approach)确保了工作节点始终维持高利用率(High Utilization)，能够动态适应(Dynamically Adapt)不断变化的工作负载，并彻底规避了传统集中式调度器(Centralized Scheduler)固有的性能瓶颈。
![关键帧](keyframes/part006_frame_00570000.jpg)
![关键帧](keyframes/part006_frame_00600000.jpg)