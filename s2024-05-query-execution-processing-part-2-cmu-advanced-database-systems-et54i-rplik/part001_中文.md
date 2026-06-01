## 并行数据扫描与独立执行
当对分区数据(Partitioned Data)执行查询（例如扫描多个 Parquet 文件）时，系统会将每个数据分区(Partition)分配给一个独立的算子实例(Operator Instance)。这些实例在并发执行期间无需协调中间结果(Intermediate Result)。然而，在后续阶段，系统必须合并这些结果，以便传递给下游算子(Downstream Operator)或作为最终的应用输出。这种协调工作由交换算子(Exchange Operator)处理，这一基础概念可追溯至 20 世纪 90 年代，至今仍是现代分布式与并行数据库架构的核心。
![关键帧](keyframes/part001_frame_00000000.jpg)

## 流水线构建与哈希连接构建阶段
将逻辑查询计划(Logical Query Plan)转换为物理计划(Physical Plan)时，需要将数据分配至可用的工作节点(Worker Node)。例如，扫描表 A 的三个分区会生成三个并行的算子实例。扫描完成后，各流水线(Pipeline)会独立且立即地应用过滤(Filter)与投影(Projection)操作。随后，结果被送入哈希连接(Hash Join)的构建侧(Build Side)。在此阶段，交换算子充当流水线断点(Pipeline Breaker)，确保所有并行实例完成工作后，系统才进入下一阶段。根据具体实现，执行引擎可能构建一个共享哈希表(Shared Hash Table)，或多个局部哈希表(Local Hash Table)。采用“先独立构建局部表，再合并”的策略往往更高效，因为它避免了向共享结构并发写入时产生的复杂闩锁(Latch)开销。
![关键帧](keyframes/part001_frame_00065400.jpg)

## 探测、聚合与交换机制
在查询计划的探测侧(Probe Side)，多个工作节点独立扫描表 B 的分区，应用过滤器，并对已构建的哈希表执行探测(Probe)操作。由于这些工作节点会产生并行的输出流(Output Stream)，系统需要引入第二个交换算子来合并结果，再将最终输出交付给用户。交换算子本身既可以仅作为等待所有输入就绪的同步屏障(Synchronization Barrier)，也可以主动将多个数据缓冲区(Data Buffer)合并为统一的输出流。具体实现取决于输出缓冲区如何在线程间暂存，以及是否涉及物理数据移动(Physical Data Movement)。
![关键帧](keyframes/part001_frame_00079850.jpg)
![关键帧](keyframes/part001_frame_00170316.jpg)

## 交换算子的线程调度与屏障逻辑
交换算子如何在线程或 CPU 核心上调度是一个常见问题。交换算子通常作为专用实例(Dedicated Instance)运行，负责追踪来自多个流水线的输入。在某些情况下，它不执行繁重的计算，仅作为屏障(Barrier)发出信号，通知下游阶段可以开始执行。在其他场景中，它会主动合并结果：既可以通过物理移动数据缓冲区来生成最终输出，也可以通过策略性地暂存缓冲区，允许多个线程并发写入。具体采用哪种方式，取决于系统的内存管理(Memory Management)机制与并发模型(Concurrency Model)。
![关键帧](keyframes/part001_frame_00222849.jpg)

## 交换算子分类：收集模式
最常见的交换模式是收集算子(Gather Operator)，它将多个工作节点的结果整合为单一的输出流(Output Stream)。该术语在 2006 年 SQL Server 的一篇技术文献发表后被广泛采用，为理解数据移动(Data Movement)提供了清晰的框架。收集算子仅负责聚合并将并行输出转发至查询计划的下一阶段。作为合并分区结果的默认机制，它不会改变底层的数据分发逻辑(Data Distribution Logic)。
![关键帧](keyframes/part001_frame_00286066.jpg)

## 分发与重分区交换策略
除了收集模式外，交换算子还支持更高级的数据路由模式(Data Routing Pattern)。分发交换(Distribute Exchange)接收单一输入流(Input Stream)，并将其扇出(Fan Out)至多个下游工作节点。当需要在查询有向无环图(Directed Acyclic Graph, DAG)的不同分支间重用中间结果时（例如计算出的子查询被多次引用），该模式尤为实用。重分区交换(Repartition Exchange，通常称为混洗 Shuffle)接收多个输入，并将其重新分发至多个输出通道(Output Channel)。这使得系统能够在流水线阶段之间动态调整执行工作节点的数量。当查询优化器(Query Optimizer)估算不准确或数据规模发生意外波动时，这种动态调整能力至关重要。
![关键帧](keyframes/part001_frame_00332416.jpg)

## 执行语义与自适应扩缩容
交换算子的设计引发了关于数据语义(Data Semantics)与计算开销(Computational Overhead)的重要考量。尽管收集或重分区操作在逻辑上通常维持元组(Tuple)数量的一对一映射，但当数据需路由至多个下游算子时，系统在物理层面可能会进行数据复制。这种复制虽不影响逻辑正确性(Logical Correctness)，但会改变物理资源(Physical Resource)的消耗。此外，交换算子天然充当数据暂存点(Staging Point)，能够对数据进行缓冲(Buffering)，从而使执行引擎能够自适应地管理内存、并发性以及工作节点的扩缩容(Worker Scaling)。这种灵活性确保了查询计划在不同工作负载(Workload)、硬件配置及执行估算(Execution Estimation)下均能保持稳健(Robust)。
![关键帧](keyframes/part001_frame_00370849.jpg)