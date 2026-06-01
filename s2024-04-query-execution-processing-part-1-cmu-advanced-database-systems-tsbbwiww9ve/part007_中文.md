## 推送式输出控制的局限性
尽管推送式(Push-based)（自下而上）执行模型在最小化函数调用开销(Function Call Overhead)和最大化 CPU 流水线(CPU Pipeline)利用率方面表现出色，但其在管理输出数据量(Output Data Volume)时引入了特定的挑战。在传统的拉取式(Pull-based)模型中，根算子(Root Operator)保留着显式控制权：一旦满足 `LIMIT` 条件，它便会直接停止发起 `next()` 调用，从而立即终止下游(Downstream)的执行流程。然而，在推送式架构中，流水线(Pipeline)的触发由外部调度器(External Scheduler)主导。即使所需的输出结果已达成，引擎仍可能继续生成并向下游推送完整的批次数据(Batch Data)，从而导致对非必要数据进行冗余处理。这一权衡凸显出：拉取式控制流(Control Flow)能够提供更简洁的提前终止(Early Termination)机制，而推送式模型则更侧重于维持连续且不间断的数据流。
![关键帧](keyframes/part007_frame_00000000.jpg)

## 执行方向与数据粒度的正交性
推送(Push-based)与拉取(Pull-based)执行模式的选择，与数据的分批方式(Data Batching)是相互独立（即正交(Orthogonal)）的。即使在推送式模型中，算子(Operator)仍可被设计为处理单个元组(Tuple)、完全物化(Fully Materialized)的结果集(Result Set)，或固定大小的向量(Vector)。例如，一个推送流水线可遍历一批元组，并一次性对整个批次调用向量化谓词评估(Vectorized Predicate Evaluation)函数。这表明执行方向(Execution Direction)（即由谁发起处理）与数据粒度(Data Granularity)（元组与向量的对比）是两个独立的架构维度(Architectural Dimension)，可根据特定工作负载(Workload)的特征进行灵活组合。
![关键帧](keyframes/part007_frame_00080616.jpg)

## 部分过滤批次的处理挑战
在经典的迭代器模型(Iterator Model)中，若某个元组未通过谓词检查(Predicate Check)，它会在传递至父算子(Parent Operator)前被直接丢弃，从而确保仅有有效数据流经查询计划(Query Plan)。向量化执行(Vectorized Execution)从根本上改变了这一机制。由于算子处理的是固定大小的批次，单个输入向量(Input Vector)在经过过滤后，往往会混合包含有效与无效的元组。核心的工程挑战随之而来：系统应如何表示这种部分过滤的批次，同时避免产生高昂的内存分配(Memory Allocation)与数据拷贝(Data Copy)开销？若在每一步算子处理中都将有效元组物理压缩(Physical Compaction)至新的缓冲区(Buffer)，过度的内存管理开销将严重拖累整体性能。
![关键帧](keyframes/part007_frame_00116283.jpg)

## 选择向量（位置列表）
解决该问题的高效方案是**选择向量**（Selection Vector），亦称**位置列表**（Position List）。该元数据结构(Metadata Structure)是一个紧密排列的整数偏移量(Integer Offsets)数组，专门用于索引原始数据向量(Raw Data Vector)中的有效元组。例如，若一个包含五个元组的批次仅在索引 1、3 和 4 处匹配成功，选择向量只需存储 `[1, 3, 4]`。该紧凑列表将被传递至下游，使后续算子(Subsequent Operator)仅处理相关元组，并安全地跳过“无效”数据。包括 Databricks Photon 引擎相关研究在内的现代文献表明，选择向量通常是实现通用过滤(General Filtering)的最快方法，因为它彻底消除了中间内存(Intermediate Memory)的重新分配开销。
![关键帧](keyframes/part007_frame_00193283.jpg)

## 位图与硬件加速掩码
另一种主流表示方法是**位图**（Bitmap），亦称**有效性掩码**（Validity Mask）。它为批次中的每个元组分配一个二进制位(Binary Bit)，用于标记其状态为有效（`1`）或已过滤（`0`）。该方法在硬件加速(Hardware Acceleration)环境中表现尤为突出。AVX-512 等现代 SIMD（单指令多数据流）指令集原生支持掩码操作(Mask Operations)，允许 CPU 直接在寄存器级别跳过对无效数据通道(Data Lane)的处理，从而避免分支跳转(Branching)。通过将位图加载至 SIMD 执行掩码(SIMD Execution Mask)，向量化引擎可在多个数据通道上并行执行谓词评估(Predicate Evaluation)或算术运算(Arithmetic Operations)，并在硬件层面自动屏蔽被过滤的结果。
![关键帧](keyframes/part007_frame_00242749.jpg)

## 零拷贝理念与拉取模型的状态跟踪开销
选择向量与位图背后的核心理念是**零拷贝执行**（Zero-Copy Execution）。传递未经修改的数据缓冲区(Data Buffer)并附带轻量级元数据，其计算成本远低于频繁调整数组大小与执行紧凑化操作(Compaction)。若下游算子检查位图或选择向量后发现其为空，即可立即中止对该批次的全部处理。相反，若在拉取式模型中尝试部分填充输出缓冲区，则需引入复杂的状态管理(State Management)与记录机制，以便跨算子边界(Operator Boundary)跟踪剩余的可用槽位(Slot)。此类间接处理方式将抵消向量化带来的性能优势。实践表明：保持数据原地驻留，仅移动有效性元数据(Validity Metadata)，才是现代查询引擎(Query Engine)的最优策略。
![关键帧](keyframes/part007_frame_00552116.jpg)