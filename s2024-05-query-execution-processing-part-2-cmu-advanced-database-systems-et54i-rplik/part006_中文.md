## 表达式树与朴素求值的代价
SQL 谓词(SQL Predicate)会被解析并转换为表达式树(Expression Tree)，其中每个节点代表一个操作符(Operator)（如 AND、OR 或比较运算符）或操作数(Operand)（常量、列引用或函数调用）。尽管该结构在概念上直观易懂，但采用朴素方法(Naive Approach)——即为每个元组(Tuple)递归遍历执行树——是极其低效的。传统的解释型执行(Interpreted Execution)严重依赖指针跳跃(Pointer Chasing)和虚函数表查找(Virtual Function Table Lookup)来动态决定下一步操作。这种间接控制流(Indirect Control Flow)会引发频繁的 CPU 缓存未命中(CPU Cache Miss)与分支预测失败(Branch Misprediction)，在处理数十亿级元组扫描时，将构成严重的性能瓶颈(Performance Bottleneck)。
![关键帧](keyframes/part006_frame_00000000.jpg)
![关键帧](keyframes/part006_frame_00089000.jpg)

## 即时编译与表达式扁平化
为消除解释执行的开销，现代系统将表达式树转换为可直接编译为原生机器码(Native Machine Code)的函数调用。诸如 Apache Arrow 的 Gandiva 等框架利用 LLVM 在运行时执行即时编译(JIT Compilation)，将历经数十年发展的编译器优化技术(Compiler Optimization Technique)直接赋能于查询执行(Query Execution)。作为替代方案，Velox 等引擎通过将表达式树扁平化(Flattening)为连续且顺序的操作数组(Operation Array)，从而规避了繁重的 JIT 编译开销。类似于 B+ 树(B+ Tree)通过连续存储节点以提升空间局部性(Spatial Locality)，这种扁平化布局使引擎能够以线性方式遍历操作指令，彻底摒弃了递归式的指针跳跃，进而显著提升了 CPU 缓存效率(Cache Efficiency)与指令流水线(Instruction Pipeline)的执行性能。
![关键帧](keyframes/part006_frame_00130233.jpg)
![关键帧](keyframes/part006_frame_00175883.jpg)

## 向量化与预编译原语
现代分析型数据库引擎(Analytical Database Engine)摒弃了针对单个元组(Tuple)的逐行求值模式，转而采用向量化执行(Vectorized Execution)技术批量处理整个数据批次(Data Batch)。系统通过引入预编译且针对特定数据类型优化的原语(Pre-compiled Primitives)（例如分别为 `int32` 或 `float64` 比较操作提供专属优化例程），彻底取代了通用的运行时求值，从而消除了动态类型检查(Dynamic Type Checking)的开销。当这些优化例程应用于包含数千个元素的列向量(Column Vector)时，函数调用成本被大规模分摊(Cost Amortization)，使得初始调用的开销几近于零。尽管将完整查询计划(Query Plan)编译为机器码有时会引发超过查询本身运行时间的编译延迟(Compilation Latency)，但仅针对表达式层(Expression Layer)实施编译或扁平化处理，则能在控制额外开销的前提下，带来持续且显著的性能收益。
![关键帧](keyframes/part006_frame_00397999.jpg)

## 高级优化与编译器集成
除结构扁平化之外，执行引擎还深度集成了诸如常量折叠(Constant Folding)等经典编译器优化技术(Compiler Optimization)。当表达式对静态常量(Static Constant)执行重复运算时，引擎仅需在规划阶段计算一次并缓存结果，从而彻底避免在数百万级元组上进行冗余求值(Redundant Evaluation)。此类编译器技术的引入，逐渐模糊了传统查询优化器(Query Optimizer)与代码生成器(Code Generator)之间的界限，引发了关于优化职责边界(Optimization Responsibility Boundary)的架构探讨。尽管部分系统仍选择保持各模块松散耦合，但诸如 LingoDB 等新兴研究平台已充分证明：将编译器阶段(Compiler Phase)深度嵌入数据库内核，能够实现针对关系型数据处理量身定制的、极具侵略性的上下文感知优化(Context-Aware Optimization)，从而释放极致性能。