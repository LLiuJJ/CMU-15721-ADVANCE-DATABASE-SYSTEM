## 寄存器控制与性能优势
![关键帧](keyframes/part004_frame_00000000.jpg)
HyPer 基于 LLVM 的方案始终优于早期的 C++ 转译(Transpilation)系统，因为直接生成 LLVM 中间表示(Intermediate Representation, IR)能够对寄存器分配(Register Allocation)提供精确的底层控制。在转译模型中，系统需依赖外部 C++ 编译器以启发式方式(Heuristic Approach)决定数据的寄存器驻留策略，这通常难以最大化流水线执行效率。相比之下，LLVM 代码生成(Code Generation)允许数据库在整个推送型执行流水线(Push-based Execution Pipeline)中，显式地将元组数据驻留于 CPU 寄存器(CPU Register)内。即便是仅涉及聚合(Aggregation)与过滤(Filtering)（无复杂连接）的简单查询，此类激进的流水线处理与寄存器驻留技术亦能带来显著的性能跃升，使执行速度逼近硬件“裸机”(Bare-metal)极限。

## 编译开销与内存优化
![关键帧](keyframes/part004_frame_00081583.jpg)
转译方案的主要缺陷在于派生外部编译器进程所引发的巨大编译延迟(Compilation Latency)。HyPer 通过将 LLVM 编译器直接嵌入数据库的地址空间(Address Space)，并在系统启动时完成一次性初始化，从而彻底规避了此问题。编译任务在专用的后台线程中执行，消除了重复的进程创建(Process Creation)、动态库加载(Dynamic Library Loading)及编译器配置解析开销。此项架构优化将编译耗时大幅压缩至 20 毫秒以内，使其得以切实应用于分析型工作负载(Analytical Workload)。即便仅对比纯执行时间（剔除编译耗时），得益于其卓越的推送型模型(Push-based Model)与确定性的寄存器管理(Deterministic Register Management)，HyPer 生成的原生 LLVM 代码依然展现出更快的执行速度。

## 即席查询瓶颈
![关键帧](keyframes/part004_frame_00227349.jpg)
尽管联机事务处理(Online Transaction Processing, OLTP)系统可通过缓存预编译语句(Prepared Statement)轻松缓解编译开销，但联机分析处理(Online Analytical Processing, OLAP)环境常需执行高度不可预测的即席查询(Ad-hoc Query)，其编译成本往往随查询复杂度呈超线性增长(Super-linear Growth)。一个典型的现实案例是：当 HyPer 作为兼容 PostgreSQL 的加速插件集成至 Tableau 时，若通过 pgAdmin 连接数据库，系统会立即触发复杂的系统目录探查查询(System Catalog Probe Query)。尽管此类查询实际访问的数据量微乎其微，却会触发海量的编译任务，导致长达 20 秒的显著停顿(Stall)，令习惯于即时反馈(Instant Feedback)的用户误判系统处于无响应状态。这暴露出一个关键缺陷：对于高交互性或模式不可预测的工作负载，在执行前强制进行全量编译(Full Compilation)显然是不切实际的。

## 自适应执行：先解释后编译
![关键帧](keyframes/part004_frame_00448533.jpg)
为攻克编译延迟瓶颈，HyPer 团队于 2018 年引入了自适应执行模型(Adaptive Execution Model)。该系统不再阻塞查询执行以等待原生机器码(Native Machine Code)编译完成，而是立即启用自定义的轻量级字节码解释器(Bytecode Interpreter)对生成的 LLVM IR 进行解释执行，从而即时启动数据处理。与此同时，后台 LLVM 编译器异步运行(Asynchronously)，将相同的中间表示(IR)转换为深度优化的 x86 机器码。此类混合架构(Hybrid Architecture)既确保了查询的瞬间启动(Instant Startup)，又能充分释放即时编译(Just-In-Time, JIT)带来的长期性能红利。

## 多级编译流水线
![关键帧](keyframes/part004_frame_00461700.jpg)
![关键帧](keyframes/part004_frame_00483349.jpg)
该自适应流水线(Adaptive Pipeline)以并行阶段运行，并附有精确的性能基准。当查询优化器(Query Optimizer)完成解析（约 0.2 毫秒）且代码生成器(Code Generator)产出 LLVM IR（约 0.7 毫秒）后，系统将同步开辟三条执行路径：
1. **字节码编译（约 0.4 毫秒）：** 将 IR 降级为轻量级自定义字节码(Custom Bytecode)，以供解释器即时执行。
2. **快速本机编译（约 6 毫秒）：** LLVM 采用低开销优化标志编译 IR，以极速生成可执行的 x86 代码。
3. **深度优化编译（约 25 毫秒）：** LLVM 启用激进的优化传递(Optimization Pass)（如 `-O2`），旨在为长耗时查询榨取极致的执行速度。

## 在任务边界无缝热切换
![关键帧](keyframes/part004_frame_00534800.jpg)
![关键帧](keyframes/part004_frame_00555583.jpg)
![关键帧](keyframes/part004_frame_00563766.jpg)
执行引擎将在预定义的任务边界(Task Boundary)处，于不同编译层级之间实现无缝热切换(Seamless Hot Switching)。工作线程(Worker Thread)负责按数据块(Morsel)粒度处理数据。每处理完一个数据块，系统便会检查是否存在更优的查询代码版本可供切换。若快速本机代码(Fast Native Code)在 6 毫秒内编译就绪，解释器将在拉取下一数据块前被即时替换。针对运行时间更长的查询，系统将进一步无缝切换至耗时约 25 毫秒的深度优化本机代码。初始字节码并非 x86 机器码，而是一种轻量级、类 JVM 的中间表示。该格式专为数据库代码生成输出量身定制，确保在免去完整编译器前端(Compiler Frontend)开销的前提下，仍能实现极速解释执行。此种自适应策略(Adaptive Strategy)既为短耗时查询提供了毫秒级即时响应能力，又为长周期分析型工作负载解锁了巅峰性能。