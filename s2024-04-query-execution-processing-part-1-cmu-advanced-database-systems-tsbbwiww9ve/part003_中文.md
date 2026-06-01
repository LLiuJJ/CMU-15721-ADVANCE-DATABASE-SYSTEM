## 过滤谓词的无分支执行
传统的过滤操作(Filter Operation)依赖于条件 `if` 语句，这迫使中央处理器(CPU)猜测执行路径，并在分支预测错误(Branch Misprediction)时遭受严重的性能惩罚(Performance Penalty)。对于现代硬件而言，更高效的方法是无分支执行(Branchless Execution)。系统在复制数据前不再显式检查谓词(Predicate)，而是无条件地将每个元组(Tuple)写入输出缓冲区(Output Buffer)。随后，系统利用三元运算符(Ternary Operator)根据谓词是否满足来计算写入偏移量(Write Offset)（0 或 1）。若某元组未通过过滤，后续迭代只需简单地覆盖此前写入的元组。尽管从软件工程角度而言，这种持续的数据复制(Data Copy)看似计算冗余，但它会被转化为线性、确定性的汇编指令(Assembly Instructions)，使 CPU 能够在零分支预测开销的情况下高速执行。
![关键帧](keyframes/part003_frame_00000000.jpg)
![关键帧](keyframes/part003_frame_00009116.jpg)

Vectorwise 的性能基准测试(Performance Benchmark)表明，尽管传统分支方法在极低选择率(Selectivity, <5%)时表现更优，但当选择率接近 50% 时，由于频繁的流水线冲刷(Pipeline Flush)，其性能会急剧下降。相比之下，无分支执行(Branchless Execution)无论数据分布(Data Distribution)如何，均能保持平稳且可预测的执行成本，这印证了面向硬件的优化(Hardware-Oriented Optimization)往往优于直观的软件逻辑。
![关键帧](keyframes/part003_frame_00143616.jpg)

## 代码特化与消除运行时分发
为进一步最小化 CPU 周期(CPU Cycles)，数据库系统应采用代码特化(Code Specialization)技术，以消除运行时类型检查(Runtime Type Checking)与动态分发(Dynamic Dispatch)机制。传统实现通常依赖冗长的 `switch` 语句或表达式树遍历(Expression Tree Traversal)来处理不同的数据类型(Data Types)与操作符，PostgreSQL 的 `numeric` 加法函数便是典型示例。此类结构迫使 CPU 执行大量跳转指令(Jump Instructions)与间接函数调用(Indirect Function Calls)，严重破坏了指令流水线(Instruction Pipeline)与缓存局部性(Cache Locality)。尽管 Rust 等现代编程语言或高级编译器(Advanced Compiler)可能优化掉部分逻辑，但间接跳转(Indirect Branch)对于现代超标量架构(Superscalar Architecture)而言本质上依然代价高昂。
![关键帧](keyframes/part003_frame_00237766.jpg)
![关键帧](keyframes/part003_frame_00305249.jpg)

现代联机分析处理(Online Analytical Processing, OLAP)系统通过在查询规划(Query Planning)阶段利用系统目录元数据(System Catalog Metadata)预先确定精确的数据类型与操作，从而规避了此问题。通过提前编译(Ahead-Of-Time, AOT)或即时编译(Just-In-Time, JIT)生成高度特化且具备类型感知(Type-Aware)的执行原语(Execution Primitives)，引擎为 CPU 提供了一段专为该查询量身定制的连续指令序列，从而在执行期彻底绕过了条件逻辑与动态类型解析(Dynamic Type Resolution)。

## 查询执行模型：控制流与数据流
在确立面向硬件的优化策略后，讨论重点转向决定算子(Operators)交互方式的架构级执行模型(Architectural Execution Model)。每种查询处理模型(Query Processing Model)均由两条独立路径定义：控制流(Control Flow)与数据流(Data Flow)。控制流决定调度顺序(Scheduling Order)，规定算子或其算子实例(Operator Instances)何时被激活调用。数据流则指定数据传递路径，明确每个算子的输入来源(Input Source)与输出目的地(Output Destination)。在 OLAP 环境中，数据流通常针对列的子集(Column Subsets)进行处理，而非完整的行元组(Row Tuples)，以此优化缓存命中率(Cache Hit Rate)。这两条路径构成了评估不同处理范式(Processing Paradigms)的基础，最终指引系统设计向能够最大化分析型工作负载(Analytical Workloads)硬件利用率的方向演进。
![关键帧](keyframes/part003_frame_00421349.jpg)

## 迭代器（火山）模型
历史上最为流行的执行范式是迭代器模型(Iterator Model)，亦称为火山模型(Volcano Model)。在 MonetDB/X100 取得突破之前，该架构主导了数据库系统设计，且至今仍是众多行式存储(Row Store)与联机事务处理(Online Transaction Processing, OLTP)系统的标准实现。在此模型中，每个算子均需实现标准的 `next()` 接口。执行流程沿控制流自上而下展开：父算子(Parent Operator)通过调用子算子(Child Operator)的 `next()` 接口请求数据，子算子继而递归调用其下属算子的 `next()`，从而将单个元组沿查询计划(Query Plan)自底向上拉取(Pull)，直至传递至根节点。
![关键帧](keyframes/part003_frame_00532350.jpg)

尽管这种基于拉取(Pull-Based)的方法具备良好的模块化(Modularity)与代码简洁性，但会引入显著的函数调用开销(Function Call Overhead)，并阻碍高效的数据批处理(Data Batching)。每次 `next()` 调用均代表一次控制流转移，现代 CPU 难以对此类调用进行高效的流水线化(Pipelining)处理。这使得传统迭代器模型从根本上不适用于需要沿执行计划持续处理大数据块的高吞吐量 OLAP 工作负载。
![关键帧](keyframes/part003_frame_00554250.jpg)