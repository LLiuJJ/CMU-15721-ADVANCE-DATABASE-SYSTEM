## 编译策略：MLIR、LLVM 与代码生成
现代数据库研究正越来越多地利用先进的编译器基础设施(Compiler Infrastructure)来优化查询执行(Query Execution)。诸如 LingoDB 等系统会先将查询计划(Query Plan)转换为 MLIR（多级中间表示(Multi-Level Intermediate Representation)，一种专为编译器开发设计的框架），随后将其移交至 Clang 或 LLVM 等后端编译器进行最终优化(Optimization)。此方法与早期在运行时(Runtime)直接生成并编译高级 C++ 代码的技术形成鲜明对比。尽管生成原始 C++ 源码并交由系统编译器编译的过程，通常比直接生成 LLVM IR(LLVM Intermediate Representation)更为耗时，但它具备显著的调试(Debugging)优势：开发人员可直接审查生成的源代码以诊断程序崩溃(Crash)或定位性能瓶颈(Performance Bottleneck)。这两种技术路径均凸显了数据库执行引擎(Execution Engine)与现代编译器优化流水线(Compiler Optimization Pipeline)之间日益深度的融合趋势。
![关键帧](keyframes/part007_frame_00000000.jpg)

## 表达式优化：常量折叠与子树消除
在查询执行启动前，执行引擎会对表达式树(Expression Tree)应用经典的编译器优化技术(Compiler Optimization)，以削减运行时开销(Runtime Overhead)。常量折叠(Constant Folding)会在查询规划阶段(Query Planning Phase)对仅包含静态常量(Static Constant)的表达式（例如 `UPPER('Wang')`）进行单次求值(Evaluation)，并用预计算结果(Pre-computed Result)直接替换整个表达式子树，从而彻底避免在海量元组(Tuple)上进行重复计算。类似地，公共子表达式消除(Common Subexpression Elimination)会识别重复的表达式分支(Branch)，仅执行一次计算，并将后续算子重定向至缓存结果(Cached Result)。然而，此类优化的实现引发了关于组件职责边界(Responsibility Boundary)的架构探讨：这些优化究竟应由执行引擎负责，还是应交由独立的查询优化器(Query Optimizer)处理？在 MySQL 等紧耦合架构(Tightly Coupled Architecture)中，优化器可在规划期执行子查询(Subquery)以解析常量；而在解耦系统(Decoupled System)中，则往往需要跨组件复制函数实现，或依赖专用 API 共享求值结果。

## 自适应查询处理的挑战
传统查询优化器(Traditional Query Optimizer)高度依赖成本模型(Cost Model)与历史统计信息(Historical Statistics)来生成物理查询计划(Physical Query Plan)，但这些估算在数据规模庞大或分布复杂时往往失准，现代数据湖(Data Lake)环境尤甚。当文件未经统计信息收集即直接上传至 Amazon S3 等对象存储(Object Storage)时，优化器缺乏精准预测算子选择率(Operator Selectivity)与数据分布(Data Distribution)的关键元数据。此外，外部连接器(External Connector)通常无法透传联合数据库(Federated Database)内部的细粒度统计信息。一旦优化器选定了次优的连接顺序(Join Order)或投影策略(Projection Strategy)，即便底层采用高度优化的向量化执行(Vectorized Execution)，亦难以弥补架构层面的根本性低效。自适应查询处理(Adaptive Query Processing)应运而生，它允许执行引擎实时监测运行时数据特征(Runtime Data Characteristics)并动态调整执行计划。尽管激进的自适应策略可能涉及丢弃已计算的部分中间结果并向优化器请求全新计划，但主流商业系统普遍采用更为保守、渐进式(Progressive)的自适应机制，以期在控制额外开销的同时最大化查询性能。
![关键帧](keyframes/part007_frame_00186733.jpg)

## Velox 自适应技术：谓词重排序与列预取
为在不触发全局重新优化(Global Re-optimization)的前提下缓解计划次优问题，Velox 等执行引擎引入了多种运行时求值优化(Runtime Evaluation Optimization)。谓词重排序(Predicate Reordering)技术会根据实时观测到的选择率(Selectivity)与计算成本(Computational Cost)，动态调整 `WHERE` 子句中过滤器(Filter)的求值顺序。系统将高选择性(High Selectivity)的过滤条件优先前置，即使其单行计算成本较高，也能迅速缩减工作数据集(Working Dataset)，从而极大减少传递至后续算子的元组数量。此外，列预取(Column Prefetching)优化了算子内部的数据访问模式(Data Access Pattern)。当表达式需引用多个列时，引擎在处理首列数据的同时，会异步发起(Asynchronously Issue)针对后续列的 I/O 请求。待首列求值完毕时，目标数据已预先加载至内存，此举有效掩盖了底层存储延迟(Storage Latency)，确保持续填满 CPU 流水线(CPU Pipeline)，避免处理器空闲。
![关键帧](keyframes/part007_frame_00299733.jpg)

## 快速路径与特定类型优化
现代执行引擎通过在运行时动态检测数据属性(Data Attributes)，并无缝切换至专用的“快速路径(Fast Path)”实现，进一步压榨硬件性能。以“非空(Not-Null)快速路径”为例，引擎会实时监控算子间传递的空值位图(Null Bitmap)；若通过高效的硬件位计数指令(Hardware Popcount Instruction)判定当前数据批次(Data Batch)中不含空值，系统将直接跳过所有空值校验分支(Null Check Branch)。此举彻底消除了代价高昂的分支预测失败(Branch Misprediction)惩罚，使 CPU 得以沿精简的无分支代码路径(Branch-free Code Path)全速执行。同类优化亦广泛应用于字符串处理：当检测到某列数据纯由 ASCII 字符构成时，引擎将自动旁路昂贵的 UTF-8 验证与解码例程(Decoding Routine)。借助此类轻量级运行时检查(Lightweight Runtime Check)，系统可为当前数据负载动态匹配最高效的执行变体(Execution Variant)，在无需修改原始查询计划的前提下，大幅摊薄单条元组(Tuple)的处理开销。
![关键帧](keyframes/part007_frame_00423533.jpg)