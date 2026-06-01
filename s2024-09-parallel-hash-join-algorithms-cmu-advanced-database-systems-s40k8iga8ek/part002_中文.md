## 缓存局部性与顺序访问的预处理
![关键帧](keyframes/part002_frame_00000000.jpg)
通过对构建表(Build Table)与探测表(Probe Table)的数据进行预处理，并将其划分为离散的线程本地数据块(Thread-Local Data Chunks)，并行连接算法能够有效消除跨核心互联流量(Cross-core Interconnect Traffic)。该设计使每个工作线程(Worker Thread)能够在专用缓冲区上执行顺序扫描(Sequential Scan)，完全免去线程间通信(Inter-thread Communication)的开销。尽管分区步骤(Partitioning Phase)会引入额外的初始指令周期，但整体执行速度将获得显著提升，因为它成功规避了核心连接操作期间的硬件级瓶颈、缓存竞争(Cache Contention)与同步停顿(Synchronization Stalls)。

## 哈希连接在 OLAP 系统中的关键作用
哈希连接(Hash Join)依然是在线分析处理(Online Analytical Processing, OLAP)数据库系统中至关重要的核心算子(Operator)。尽管连接操作并非在所有查询(Query)中均占据绝对主导的执行成本，但在数据密集型工作负载(Data-Intensive Workloads)下，它通常是主要的性能瓶颈。工程优化的核心目标是在整个连接执行期间维持 CPU 核心的 100% 利用率，这需通过极力削减因数据从 DRAM 加载至 CPU 缓存(Cache)所引发的流水线停顿(Pipeline Stalls)来实现。优化此类内存传输(Memory Transfer)过程对于维持系统峰值吞吐量(Peak Throughput)至关重要。
![关键帧](keyframes/part002_frame_00034533.jpg)

## 高层架构：分区、构建与探测
从宏观架构(High-Level Architecture)来看，并行哈希连接(Parallel Hash Join)主要划分为三个阶段。首阶段为**分区(Partitioning)**，该阶段为可选步骤，主要依据连接键(Join Key)的哈希值，将内表/构建表(Inner/Build Table)与外表/探测表(Outer/Probe Table)拆分为多个独立分区(Partitions)。随后进入**构建(Build)**阶段，每个工作线程将被分配至特定分区，负责构建单一逻辑哈希表(Single Logical Hash Table)的本地片段(Local Fragment)。维护全局统一的逻辑视图对于规避哈希查找时的假阴性(False Negatives，即漏匹配)至关重要。最终在**探测(Probe)**阶段，工作线程顺序扫描其对应的外部分区，向哈希表发起查找以匹配记录，合并关联元组(Tuples)，并将结果集(Result Set)推送至下游执行流水线(Execution Pipeline)中。
![关键帧](keyframes/part002_frame_00066916.jpg)

## 被忽视的结果物化成本
学术界基准测试(Academic Benchmarks)往往严重低估最终结果物化(Result Materialization)步骤对整体性能的影响。诸多早期研究仅测量构建(Build)与探测(Probe)阶段的速率，却忽略了将匹配元组合并并复制至输出缓冲区(Output Buffer)所施加的 CPU 与缓存压力。在生产级系统中，此类近似内存拷贝(`memcpy`)的操作具有不可规避性。萨尔大学(Saarland University)等研究团队明确指出，在微基准测试(Micro-benchmarks)或单指令多数据流(SIMD)/向量化连接(Vectorized Joins)研究中省略物化环节，将导致性能评估严重失真。真实的数据库系统评估必须将此末端流水线阶段纳入考量，方能准确反映生产环境中的实际开销(Overhead)。
![关键帧](keyframes/part002_frame_00223799.jpg)

## 深入分区阶段
分区阶段(Partitioning Phase)的核心宗旨在于优化数据局部性(Data Locality)。通过将输入关系(Input Relations)重新路由至各个 CPU 核心，使每个工作线程仅处理专属的数据子集，线程即可在零跨 NUMA 节点内存访问(Cross-NUMA Memory Access)的前提下执行缓冲区顺序扫描。执行分区指令的额外开销，始终能被缓存命中率(Cache Hit Rate)的提升与内存访问延迟(Memory Access Latency)的降低所完全抵消。若输入数据在加载时已预先按连接键完成分区（在现代数据湖(Data Lake)架构中极为罕见），则可安全跳过此步骤。否则，系统将采用内存内分区(In-Memory Partitioning)策略，将传统的基于磁盘的 Grace 哈希连接(Disk-based Grace Hash Join)算法针对现代硬件特性进行深度适配。
![关键帧](keyframes/part002_frame_00332849.jpg)

## 非阻塞与阻塞分区策略
实现分区阶段主要存在两种架构范式。**非阻塞分区(Non-blocking Partitioning)**允许所有工作线程并发执行并同步填充分区缓冲区。该方案实现相对直观，但必须依赖闩锁(Latches)等同步原语来防止并发写入导致的数据覆盖(Data Corruption)。相比之下，**阻塞式/基数分区(Blocking/Radix Partitioning)**的实现更为复杂：它通常需对数据进行多轮遍历(Multi-pass Traversal)，但通过在填充阶段前精确预计算写入偏移量(Write Offsets)，彻底消除了线程间同步(Inter-thread Synchronization)的开销。
![关键帧](keyframes/part002_frame_00398700.jpg)

## 共享全局分区与线程局部存储
在非阻塞分区架构中，系统需在共享缓冲区模型与私有缓冲区模型间进行权衡。采用**共享全局分区(Shared Global Partitions)**意味着各线程可随时向任意目标分区写入数据，因此必须在目标哈希桶(Target Hash Buckets)上频繁加闩锁以规避数据损坏。替代方案为**线程局部存储(Thread-Local Storage)**模型，该系统为每个 CPU 核心分配 `N` 个微型分区(Mini-Partitions)。由于各核心在其私有缓冲区上以单线程模式(Single-threaded Mode)运行，该方案彻底清除了初始写入阶段的所有同步开销。然而，其代价是在流程末端引入了强制性的合并阶段(Merge Phase)，需将这些微型分区聚合重组至最终的全局分区结构中。
![关键帧](keyframes/part002_frame_00448766.jpg)

## 共享分区的执行机制
在共享非阻塞模型(Shared Non-blocking Model)中，数据表在逻辑上被划分为 `N` 个分区缓冲区。在处理数据分块(Morsels)时，工作线程对连接键执行哈希运算以定位目标分区，并将元组追加至对应的共享缓冲区，其底层机制类似于链式哈希表(Chained Hash Table)。鉴于并发写入(Concurrent Writes)的不可预测性，数据插入前必须在对应哈希桶上获取闩锁。该方案的核心权衡极为明确：算法在缓冲区填充阶段需承担高昂的同步开销，但所有数据已直接落位至最终的全局分区，使得后续清理阶段(Cleanup Phase)实现零额外开销(Zero Overhead)。
![关键帧](keyframes/part002_frame_00483433.jpg)

## 线程局部分区与合并开销
线程局部方案(Thread-Local Approach)则将上述权衡关系完全反转。各核心独立维护 `N` 个微型分区，允许在初始数据遍历(Initial Data Pass)期间执行完全无锁的高速写入(Lock-free Writes)。性能瓶颈随之转移至末端合并阶段，系统需在此协调数据聚合操作。通常，系统会指派特定核心负责收集归属于同一全局分区 ID(Global Partition ID)的所有微型分区，并将其拼接至统一的全局缓冲区。尽管该设计彻底消除了运行时的闩锁竞争(Latch Contention)，但最终的合并遍历(Merge Pass)极易演变为执行流水线的阻塞点(Pipeline Stall Point)；若缺乏精细优化，将引入显著的端到端延迟(End-to-End Latency)。
![关键帧](keyframes/part002_frame_00543833.jpg)
![关键帧](keyframes/part002_frame_00582433.jpg)