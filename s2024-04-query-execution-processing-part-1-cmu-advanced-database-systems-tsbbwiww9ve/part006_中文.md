## 拉取式（自上而下）执行模型
传统的查询执行依赖于拉取式(Pull-based)、自上而下(Top-down)的架构。处理流程从根算子(Root Operator)开始，并沿查询计划(Query Plan)向下级联调用。执行过程由重复的 `next()` 调用驱动：根算子向其子算子请求数据，子算子继续向下请求，直至从叶节点(Leaf Node)的扫描算子(Scan Operator)处逐层向上拉取数据，最终生成查询结果。这种迭代器模型(Iterator Model)构成了大多数经典数据库系统的基础设计，并天然支持流水线边界(Pipeline Boundary)，用于在特定阶段暂存中间结果(Intermediate Results)。
![关键帧](keyframes/part006_frame_00000000.jpg)

## 函数调用开销与 CPU 流水线中断
尽管实现简单且应用广泛，但拉取式模型引入了显著的运行时开销(Runtime Overhead)。每次 `next()` 调用通常涉及虚函数调用(Virtual Function Call)与指针间接寻址(Pointer Indirection)，这在现代超标量 CPU(Superscalar CPU) 上会转化为代价高昂的跳转指令(Jump Instruction)。频繁的控制流切换会破坏指令预取(Instruction Prefetch)与分支预测(Branch Prediction)，进而阻碍处理器的并行执行能力。随着查询复杂度的提升，数百万次逐元组(Tuple-at-a-time)或逐批次(Batch-at-a-time)的函数调用所累积的开销，往往成为系统性能的主要瓶颈。
![关键帧](keyframes/part006_frame_00031099.jpg)

## 推送式（自下而上）替代方案
为消除 `next()` 调用带来的开销，推送式模型(Push-based Model)通过从叶节点启动并将数据逐层向上传递，彻底反转了执行方向。外部调度器(External Scheduler)或执行控制器(Execution Controller)采用自下而上的方式启动流水线(Pipeline)，使算子能够在生成数据后，立即将其推送(Push)给父算子。该架构因 Hyper 和 Umbra 等数据库系统而广为人知，这些系统率先在查询执行中引入了即时编译(JIT Compilation)技术。通过将控制流(Control Flow)与数据移动(Data Movement)解耦，推送式模型实现了更具可预测性且更契合底层硬件的执行模式。
![关键帧](keyframes/part006_frame_00142199.jpg)

## 算子融合与紧凑的流水线循环
在推送式架构中，整个流水线被融合(Operator Fusion)为连续的代码块，而非依赖离散独立的算子对象。例如，哈希连接(Hash Join)可被编译为两个紧凑的 `for` 循环：一个负责扫描构建侧(Build Side)并构建哈希表(Hash Table)；另一个负责扫描探测侧(Probe Side)、评估谓词(Predicate)、探测哈希表并输出结果。这种融合机制确保每个元组或向量批次(Vector Batch)在引擎处理下一批次之前，都能完成从数据源到目的地的完整处理流程，从而大幅削减了上下文切换(Context Switch)与中间物化(Intermediate Materialization)的开销。
![关键帧](keyframes/part006_frame_00155216.jpg)

## 实现策略：JIT 编译与代码特化
若要在不硬编码(Hardcoding)所有可能查询计划的前提下实现算子融合，必须依赖先进的编译技术。目前主要有两种实现路径：即时编译(JIT Compilation)与代码特化(Code Specialization)。以 Hyper 为代表的 JIT 系统会在运行时动态生成 LLVM IR（中间表示(Intermediate Representation)）甚至原生 x86 汇编代码，将其编译为共享对象(Shared Object)，并通过动态链接(Dynamic Linking)加载执行。另一种方案如 Vectorwise 系统，则采用代码特化技术，将算子逻辑表示为预编译函数指针数组。在处理向量化批次(Vectorized Batch)时，函数指针调用的开销可被均摊至上千个元组，从而在无需完整运行时编译的情况下，依然保持高吞吐量(Throughput)。

## 集中式调度与数据路由
推送式执行依赖集中式调度器(Centralized Scheduler)来统一管理控制流，精确指定每个流水线的触发时机及其输出数据的流向。调度器负责预先分配输出缓冲区(Output Buffer)，在数据需分发至多个下游算子(Downstream Operator)时处理分支逻辑(Branch Logic)，并在分布式架构中将数据分发工作委托给 Shuffle 服务。这种显式编排(Explicit Orchestration)彻底取代了传统 `next()` 调用链中的隐式控制流(Implicit Control Flow)，使数据库引擎能够对跨查询计划的执行顺序、资源分配与数据路由(Data Routing)实现完全掌控。

## 性能权衡与架构复杂性
尽管推送式 JIT 模型通过将数据紧密驻留于 L1 缓存(L1 Cache)与 CPU 寄存器(CPU Register)中，提供了卓越的性能表现，但其也引入了极高的工程复杂度。即便对经验丰富的数据库研发团队而言，构建并维护一个健壮的 JIT 编译器(JIT Compiler)与调度器依然是一项艰巨挑战。相比之下，拉取式模型虽存在虚函数调用开销，却为早期终止(Early Termination)（如 `LIMIT` 操作）提供了更直观的控制流，且更易于调试与排查。最终，在架构模型之间的抉择，实质上是在硬件资源利用率(Hardware Resource Utilization)与开发复杂度、系统可维护性(System Maintainability)之间寻求最佳平衡。
![关键帧](keyframes/part006_frame_00523750.jpg)