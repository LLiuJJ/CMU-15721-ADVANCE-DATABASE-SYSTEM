## 非抢占式调度与数据库协程
![关键帧](keyframes/part008_frame_00000000.jpg)
现代高性能数据库系统通常采用协作式非抢占调度模型(Cooperative Non-preemptive Scheduling Model)来管理并发(concurrency)。与允许操作系统内核强制中断线程并重新分配硬件资源的抢占式调度(Preemptive Scheduling)不同，数据库的非抢占式调度器依赖于线程主动让出控制权。这要求在数据库代码库中植入显式的让出(yield)指令，将执行权交还给内部调度器，以便其评估是否切换至其他任务。该调度范式由 SQL Server 于 2006 年通过其 SQL OS 抽象层率先引入，远早于 C++ 与 Go 等现代编程语言广泛支持内置协程(coroutines)的时代。

## 时间片与执行跟踪
![关键帧](keyframes/part008_frame_00077800.jpg)
![关键帧](keyframes/part008_frame_00089383.jpg)
为确保资源的公平分配，并防止长耗时操作导致其他任务陷入饥饿(starvation)状态，系统可为快照扫描(snapshot scan)等特定算子(operator)分配固定的执行时间片(time slice)（例如 4 毫秒）。算子在处理数据时会持续追踪已消耗的 CPU 时间。一旦执行时长超出预设阈值，线程便会触发让出操作，将控制权交还给调度器。尽管频繁查询系统时钟会带来较高的计算开销，但现代 CPU 提供的硬件时间戳计数器(Timestamp Counter, TSC)等指令集实现了高效且低开销的时间计量，使该机制在生产环境中具备高度的可行性。

## 针对锁争用的条件让出
内置的数据库调度器通过支持条件让出(conditional yield)机制，提供了远比宿主操作系统(Host OS)更为精细的控制粒度。例如，当线程尝试获取被其他事务持有的数据库锁(database lock)时，可向调度器附加特定条件交出控制权：“在该锁释放前，请勿重新调度本线程。”此时，数据库调度器既不会让线程陷入高开销的自旋等待(spin-waiting)以浪费 CPU 周期，也不会将控制权盲目交还给操作系统，而是将该线程挂起至等待队列，仅在锁资源满足条件时才高效地重新唤醒它。此举大幅降低了上下文切换(context switch)的开销，并有效提升了系统整体吞吐量(throughput)。

## 内部调度器的工业界实现
![关键帧](keyframes/part008_frame_00167399.jpg)
当前，众多数据库系统已采用自定义协程(coroutines)或调度框架，以规避操作系统的调度限制。例如，CeliaDB 实现了一套名为 C-star 的复杂框架，用于在工作线程池(worker thread pool)间管理协作式执行(cooperative execution)。FaunaDB 则采用了一种更为精简但高效的策略：在发起每次磁盘 I/O(disk I/O)请求前，主动将控制权让出给内部调度器。此外，西蒙菲莎大学(Simon Fraser University)研发的学术实验型系统（如 CorBase）已将线程局部协程(thread-local coroutines)直接集成至其核心架构中。这些工业界与学术界的实践印证了一个日益显著的共识：数据库内核级调度是实现可预测高吞吐量(predictable high throughput)性能的关键所在。

## 分布式调度与架构哲学
![关键帧](keyframes/part008_frame_00175833.jpg)
![关键帧](keyframes/part008_frame_00185683.jpg)
![关键帧](keyframes/part008_frame_00192833.jpg)
当工作负载扩展至多机集群时，分布式调度(distributed scheduling)不仅需继承单节点协调(single-node coordination)的全部复杂性，还需额外应对网络延迟(network latency)、跨节点通信(cross-node communication)及远程状态管理(remote state management)等挑战。以 Snowflake 为代表的云原生架构(cloud-native architectures)证明，现代系统已能够无缝融合本地协作式调度与分布式工作窃取(distributed work stealing)。本讲座传递的核心架构理念十分明确：为追求极致性能，数据库系统必须完全自主掌控线程管理，而非将其委托给通用操作系统。尽管构建类似 SQL OS 的自定义调度器会显著增加工程复杂度，但它彻底消除了不可预测的操作系统内核干扰，从而能够实现高度优化且具备工作负载感知(workload-aware)特性的执行流。

## 课程总结与后续主题
![关键帧](keyframes/part008_frame_00211199.jpg)
![关键帧](keyframes/part008_frame_00227350.jpg)
![关键帧](keyframes/part008_frame_00235283.jpg)
![关键帧](keyframes/part008_frame_00241649.jpg)
![关键帧](keyframes/part008_frame_00247816.jpg)
![关键帧](keyframes/part008_frame_00253783.jpg)
![关键帧](keyframes/part008_frame_00259950.jpg)
![关键帧](keyframes/part008_frame_00266416.jpg)
本模块至此结束了对查询调度(query scheduling)、协调机制(coordination mechanisms)及内部线程模型(internal thread model)的深入探讨。后续课程将聚焦于高级连接算法(advanced join algorithms)，首先详细讲解哈希连接(hash joins)，随后引入多路连接策略(multi-way join strategies)。此外，针对同学们先前提出的疑问，课程将增设关于硬件性能计数器(hardware performance counters)的专题概述。最后，下周的教学安排将转向项目进度汇报(project progress updates)，旨在将上述理论调度概念与实际动手开发系统紧密衔接。