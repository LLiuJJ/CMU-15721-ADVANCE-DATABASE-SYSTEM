## 向量化查询执行(Vectorized Query Execution)简介
欢迎来到卡内基梅隆大学的高级数据库系统课程。今天的讲座重点介绍**向量化查询执行**（Vectorized Query Execution），这是现代 OLAP(Online Analytical Processing)系统实现高查询性能所依赖的一项基础技术。在之前的课程中，我们探讨了如何将查询计划(Query Plan)划分为多个流水线(Pipeline)以实现并行执行（任务并行化(Task Parallelism)），并讨论了查询编译(Query Compilation)与代码生成(Code Generation)的作用。我们还介绍了查询自适应技术(Adaptive Query Processing)，它允许优化器根据运行时观察到的性能动态调整执行策略。

![关键帧](keyframes/part000_frame_00000000.jpg)

![关键帧](keyframes/part000_frame_00009966.jpg)

基于这些概念，今天我们将深入探讨向量化技术，将其作为加速数据库查询处理的另一项关键优化层。

## 数据并行化(Data Parallelism)与性能扩展(Performance Scaling)
向量化技术将传统的标量算法(Scalar Algorithm)（每次处理一个元组(Tuple)或操作数(Operand)）转换为向量化格式，从而利用 CPU 提供的 SIMD(Single Instruction, Multiple Data)指令。这实现了**数据并行化**，使得单个算子(Operator)或表达式内的多个操作能够同时作用于不同的数据元素。其性能提升非常显著：正如将工作分配给多个线程或节点能带来线性加速一样，SIMD 引入了倍增效应。例如，在一台 32 核且具备理想扩展性能(Perfect Scaling)的机器上，任务并行可带来 32 倍的加速。如果 SIMD 每个周期能处理 4 个元组，那么单节点的理论最高加速比可达 128 倍。

![关键帧](keyframes/part000_frame_00036666.jpg)

尽管内存传输、磁盘 I/O(Disk I/O)和网络通信(Network Communication)等实际开销使我们无法达到理论峰值（实际收益通常限制在 1.4 倍左右），但即便是这种有限提升，对于数据库系统而言也是一项重大的优化成果。

## SIMD 架构(Architecture)与历史背景
SIMD 的基础可追溯到 20 世纪 60 年代的弗林分类法(Flynn's Taxonomy)，该分类法对计算机体系结构(Computer Architecture)的指令类型进行了划分。SIMD 允许处理器使用专用的宽寄存器(Wide Register)，同时对多个数据元素执行相同的操作。数据库引擎设计中的一个关键优化原则是：尽可能让数据驻留在这些 SIMD 寄存器中，在将结果写回 CPU 缓存(CPU Cache)或主存(Main Memory)之前完成最大量的计算。

![关键帧](keyframes/part000_frame_00262716.jpg)

尽管多种指令集架构(Instruction Set Architecture, ISA)都支持 SIMD 的变体，但本讲座将重点关注 Intel 的 **AVX-512**(Advanced Vector Extensions 512)。虽然由于遗留软件依赖，PowerPC 等传统架构仍在某些特定的企业环境中存在，但现代云基础设施和数据库优化已高度依赖 x86 架构(x86 Architecture)的演进。与早期版本相比，AVX-512 专门引入的指令集增强功能对数据库工作负载(Workload)具有显著优势。

## 实际示例：标量与 SIMD 矩阵加法(Matrix Addition)
为了说明执行上的差异，我们考虑一个简单的矩阵加法操作 `X + Y = Z`。在传统的标量方法中，循环会顺序遍历每个元素，每对值执行一条加法指令和一条存储指令。即使编译器进行了循环展开(Loop Unrolling)，根本瓶颈依然存在：一条指令恰好只能处理一个数据元素。

![关键帧](keyframes/part000_frame_00457283.jpg)

![关键帧](keyframes/part000_frame_00467266.jpg)

借助 SIMD，我们可以将数值向量直接加载到宽寄存器中（例如 128 位或 512 位）。假设使用 32 位整数，一条 SIMD 指令即可同时完成两个完整寄存器中对应位置元素的相加，并输出一个向量结果。过去需要八条独立的加法和存储指令才能完成的操作，现在仅需两条指令即可完成，这大幅降低了指令获取/解码(Instruction Fetch/Decode)的开销，并显著提升了列式数据库(Column-oriented Database)扫描的吞吐量(Throughput)。

![关键帧](keyframes/part000_frame_00507066.jpg)

## 水平向量化(Horizontal Vectorization)与垂直向量化(Vertical Vectorization)
数据库系统主要通过两种范式(Paradigm)来实现向量化。**水平向量化**（Horizontal Vectorization）指的是对单个 SIMD 寄存器内的所有元素进行操作，以产生一个标量结果，例如对一个包含四个数据元素的寄存器进行求和。虽然早期 CPU 缺乏对这些规约操作(Reduction Operation)的原生支持，但现代架构（AVX2 和 AVX-512）已包含专用的指令。然而，水平向量化在核心关系型查询处理(Relational Query Processing)中通常不占主导地位。

![关键帧](keyframes/part000_frame_00552066.jpg)

数据库引擎的主要关注点是**垂直向量化**（Vertical Vectorization），即以批处理(Batch Processing)方式将操作应用于多个数据向量的对应元素。这种模型与列式存储格式(Columnar Storage Format)天然契合，能够在数十亿个元组上实现高效且适合流水线(Pipeline)的执行，构成了现代高性能 OLAP 查询引擎的骨干。

---

## 数据库中垂直向量化与水平向量化
![关键帧](keyframes/part001_frame_00000000.jpg)
数据库查询执行领域演进的关键技术是**垂直向量化(Vertical Vectorization)**。在该模型中，SIMD(Single Instruction, Multiple Data) 寄存器按数据通道(Lanes)对齐，单条指令对来自多个通道的固定宽度数据元素进行并行操作，以生成新的输出向量。这种方法与列式数据处理(Columnar Data Processing)完美契合。虽然**水平向量化(Horizontal Vectorization)**在核心关系运算(Relational Operations)中不占主导地位，但它在特定聚合操作(Aggregation Operations)中仍然非常有用。例如，ClickHouse 等系统利用水平向量化将 SIMD 寄存器内的所有元素求和为一个标量值(Scalar Value)。该功能在 AVX2(Advanced Vector Extensions 2) 及更新的架构中已得到广泛支持，但在早期的 CPU 代际(CPU Generations)中并不存在。

## Intel SIMD 与 AVX-512 的演进
![关键帧](keyframes/part001_frame_00031783.jpg)
回顾 Intel SIMD 扩展(SIMD Extensions)的历史，**AVX-512**(Advanced Vector Extensions 512)（约于 2017 年推出）是现代数据库系统的一个重要里程碑。它引入了 512 位宽寄存器(512-bit Wide Registers)，能够同时处理整数、单精度浮点数(Single-Precision Floating-Point)和双精度浮点数(Double-Precision Floating-Point)。AVX-512 中最具影响力的新增功能是原生支持**谓词掩码(Predicate Masks)和置换指令(Permutation Instructions)**。这些硬件级特性(Hardware-Level Features)允许开发者精确指定哪些数据通道应参与运算，从而免去了早期 SIMD 实现中必需的手动位掩码(Bitmask)模拟操作。

## 硬件碎片化与运行时自适应
![关键帧](keyframes/part001_frame_00128100.jpg)
与早期“全有或全无”模式的 SIMD 扩展不同，AVX-512 引入了一个碎片化的生态系统(Fragmented Ecosystem)。Intel 将 AVX-512 划分为多个可选的指令子集(Instruction Subsets)，这意味着宣称支持 AVX-512 的 CPU 可能仅实现了特定的指令组。因此，数据库引擎不能假设所有硬件都具备完整的 AVX-512 支持。像 ClickHouse 这样的系统会实施运行时 CPU 标志位检查(Runtime CPU Feature Flag Checking)和条件代码路径(Conditional Code Paths)，以动态适应底层处理器的能力。开发者必须仔细验证实际可用的指令子集，因为使用不当有时会导致性能不佳，甚至由于旧微架构上的降频惩罚(Frequency Downclocking Penalty)而导致执行变慢。

## 在数据库代码中实现 SIMD 的策略
![关键帧](keyframes/part001_frame_00205399.jpg)
![关键帧](keyframes/part001_frame_00222099.jpg)
数据库开发者通常从以下三种方法中选择一种来利用 SIMD：
1. **编译器自动向量化(Compiler Auto-Vectorization)**：编译器自动将结构紧凑的数组循环重写为向量化指令。这是最简便的方法，但高度依赖于代码结构的规范性以及不存在内存依赖关系(Memory Dependencies)。
2. **编译器提示(Compiler Hints)**：开发者通过 Pragma 或关键字提供明确指导，以鼓励编译器进行向量化，而无需编写底层内部函数(Intrinsics)。
3. **手动 SIMD 内部函数(Manual SIMD Intrinsics)**：开发者直接调用特定的 SIMD 指令，在增加复杂度和可移植性开销的代价下提供最大程度的控制。
现代编译器（GCC、Clang、ICC）在自动向量化方面已有显著改进，但成功与否仍取决于能否将数据库代码组织成结构清晰、利于循环优化(Loop Optimization)的代码块。此外，编译环境(Compilation Environment)决定了输出结果：在没有 AVX-512 的机器上编译的代码，即使部署到支持它的企业级服务器上，也不会利用该特性。

## 指针别名与编译器的保守性
![关键帧](keyframes/part001_frame_00273966.jpg)
阻碍自动向量化的主要障碍是**指针别名(Pointer Aliasing)**。当函数接收多个指针（例如输入数组 `x`、`y` 和输出数组 `z`）时，编译器无法在编译时(Compile Time)确定这些内存区域是否重叠。如果 `z` 与 `x` 重叠，向量化循环可能导致后续的迭代读取到过期或被覆盖的数据，从而产生与标量执行(Scalar Execution)不同的错误结果。为了保证数学正确性，编译器会采取高度保守的立场，拒绝自动向量化那些无法证明指针独立性(Pointer Independence)的循环。

## 使用 `restrict` 和 Pragmas 启用自动向量化
![关键帧](keyframes/part001_frame_00397249.jpg)
![关键帧](keyframes/part001_frame_00528233.jpg)
![关键帧](keyframes/part001_frame_00588633.jpg)
为了克服别名障碍，开发者可以使用 `restrict` 关键字（最初源自 C99 标准，但已广泛被 C++ 编译器支持）。通过将指针标注为 `restrict`，程序员向编译器明确保证内存区域在循环执行期间不会重叠，从而开启安全的自动向量化。或者，开发者可以采用强力的编译器 Pragma(Compiler Pragma)来禁用依赖检查(Dependency Checking)并强制向量化。虽然这如同“不系安全带驾驶”般存在风险，且要求开发者绝对确保内存安全(Memory Safety)，但在自动优化失效时，它提供了一种强有力的后备手段。理解这些编译器行为对于构建高性能的向量化数据库算子(Database Operators)至关重要。

---

## 使用 `restrict` 实现安全的自动向量化(Auto-Vectorization)
![关键帧](keyframes/part002_frame_00000000.jpg)
![关键帧](keyframes/part002_frame_00023800.jpg)
C/C++ 中的 `restrict` 关键字是一个关键的编译器提示(Compiler Hint)，用于保证指针参数在函数生命周期内不会发生内存别名(Memory Aliasing)。通过显式声明数组引用的是独立且不重叠的内存区域，开发者能够使编译器安全地将标量循环(Scalar Loop)重写为向量化的 SIMD(Single Instruction, Multiple Data) 指令，而无需担心数据覆盖或损坏。这项技术在 DuckDB 等现代数据库引擎中至关重要，这些引擎大量使用 `__restrict` 注解来标记算子函数(Operator Functions)，从而解锁激进的自动向量化优化。系统通常会将此技术与条件位掩码(Conditional Bitmask)优化结合使用：如果有效性掩码(Validity Mask)确认整个数据批次(Data Batch)中不包含空值(Null)，引擎便会绕过逐行的空值检查逻辑，从而大幅降低分支预测开销(Branch Overhead)。这一策略与 Velox 和 ClickHouse 所采用的优化方案高度一致。

## 使用编译时宏(Compile-Time Macros)处理硬件碎片化(Hardware Fragmentation)
![关键帧](keyframes/part002_frame_00071033.jpg)
由于 AVX-512(Advanced Vector Extensions 512) 被划分为多个可选的指令子集(Instruction Subsets)，而非单一的整体特性集，数据库二进制文件(Database Binaries)必须能够适配不同的 CPU 能力(CPU Capabilities)。ClickHouse 等引擎通过大量使用编译时宏（如 `#if defined`）来解决这一问题，从而精确检测目标处理器所支持的具体 AVX-512 指令组。这种条件编译(Conditional Compilation)机制在单一代码库(Codebase)中衍生出多条专用代码路径(Code Paths)，确保聚合运算(Aggregation Operations)与数学原语(Mathematical Primitives)能够充分调用当前硬件确切支持的指令。尽管这提升了源代码的复杂度，但它彻底消除了运行时的特性检查(Runtime Feature Checking)开销，并确保了代码在各种 Intel 微架构(Intel Microarchitectures)上均能实现最优的执行性能。

## 编译器 Pragma 与显式 SIMD 内部函数(SIMD Intrinsics)
![关键帧](keyframes/part002_frame_00156466.jpg)
![关键帧](keyframes/part002_frame_00188199.jpg)
当自动向量化因编译器保守的别名分析(Alias Analysis)而受阻时，开发者可以使用 `#pragma ivdep` 或 OpenMP 的 `#pragma simd` 等编译器指令(Compiler Pragmas)来强制启用优化。这些指令会指示编译器忽略潜在的内存依赖(Memory Dependencies)并强制进行向量化，从而将保证代码正确性的责任完全交由程序员承担。为了获得对执行流程的绝对控制——尤其是在自主定义硬件栈(Hardware Stack)的托管云数据库(Managed Cloud Databases)（如 BigQuery、Redshift）中——工程师通常会完全绕过自动向量化，转而直接编写**显式 SIMD 内部函数(Explicit SIMD Intrinsics)**。这些内部函数在语法上表现为标准的 C/C++ 函数调用，但在编译时会直接映射为特定的 CPU 指令，从而实现对 SIMD 寄存器(SIMD Registers)的精准操控。为保持跨指令集架构(Instruction Set Architecture, ISA)的可移植性(Portability)，开发者通常会引入 Google Highway 或 `libsimd` 等硬件抽象层(Hardware Abstraction Layer)，以便在目标环境不支持 AVX-512 时，自动回退(Fallback)至更窄的寄存器或标量实现。

## 在代码中实现显式向量化(Explicit Vectorization)
![关键帧](keyframes/part002_frame_00310449.jpg)
编写显式向量化代码需要将传统的标量循环重构为基于 SIMD 寄存器的批量操作。开发者需将常规的数组指针转换为宽向量类型(Wide Vector Types)（例如 512 位整型向量），将连续内存中的数据批量加载(Load)至 SIMD 寄存器中，并执行并行的算术或比较指令。在此模式下，循环的迭代次数将按比例缩减（除以向量通道宽度/Vector Lane Width），这意味着单次迭代即可并行处理 4、8 或 16 个数据元素，彻底摒弃了逐一顺序处理的方式。完成 SIMD 运算后，计算结果将被批量存储(Store)回输出缓冲区(Output Buffer)。尽管该方法要求开发者严谨处理循环边界(Loop Boundaries)与尾部剩余元素(Tail Handling)，但它能提供高度稳定且极高的吞吐量(Throughput)，这对于列式存储数据库(Columnar Storage Database)中扫描数十亿元组(Tuples)的场景至关重要。

## 评估向量化策略(Vectorization Strategies)：性能基准测试(Performance Benchmarking)
![关键帧](keyframes/part002_frame_00377466.jpg)
![关键帧](keyframes/part002_frame_00493199.jpg)
针对不同向量化方法的学术基准测试(Academic Benchmarks)表明，混合策略(Hybrid Strategy)始终能带来最佳性能。在利用 Clang、GCC 和 Intel ICC 编译器，针对 TPC-H 基准测试中的核心数据库原语(Database Primitives)（如哈希连接、选择过滤、投影）进行的受控实验(Controlled Experiments)中，结果证实：仅依赖**自动向量化**虽能提供一个可靠的性能基线(Performance Baseline)，但仍有大量优化潜力未被挖掘；反之，**完全手动编写内部函数**往往会引入过高的代码复杂度(Code Complexity)，反而抵消了预期的性能收益。最优实践是：首先依赖编译器对结构清晰的循环进行自动向量化，随后通过性能分析工具(Profiling)精准定位瓶颈，再手动重写这些特定的热点代码(Hotspot Code)。这种有针对性的混合方案实现了极高的指令执行效率，充分证明了针对底层算子原语进行精细化优化，仍是加速向量化查询执行(Vectorized Query Execution)的最有效路径。

---

## 评估向量化策略(Vectorization Strategies)：基准测试(Benchmarking)与权衡
学术基准测试对比不同向量化策略的结果表明，并不存在一种适用于所有工作负载(Workloads)的“通用最优”方法。在针对 TPC-H 查询进行测试时，完全手动编写的 SIMD 内部函数(SIMD Intrinsics)有时能生成更精简的汇编指令(Assembly Instructions)，或比编译器自动生成的代码实现更快的执行时间。然而，这些性能收益伴随着巨大的工程成本，因为编写内部函数几乎等同于手写汇编代码。相反，纯粹的自动向量化(Auto-Vectorization)虽然能大幅提升开发者的生产效率，但也留下了未被充分挖掘的优化空间。理论上最优的策略是一种**混合方法(Hybrid Approach)**：依赖编译器自动向量化绝大多数数据库原语(Database Primitives)，随后对生成的汇编代码进行性能剖析(Profiling)以定位瓶颈，并仅选择性地使用手动内部函数重写这些热点代码(Hotspot Code)。尽管这种方法能实现极高的指令执行效率，但它要求开发者具备深厚的底层系统专业知识，且在面对不可预测的查询工作负载时难以大规模推广。

![关键帧](keyframes/part003_frame_00000000.jpg)

## AVX-512 降频现象(Frequency Throttling)
尽管 AVX-512(Advanced Vector Extensions 512) 提供了更宽的寄存器，但受限于硬件层面的功耗(Power Consumption)和散热(Thermal)约束，它并不能在所有场景下都保证更优的性能。密集执行 AVX-512 指令流可能会触发 **CPU 降频(CPU Downclocking)** 机制，即 Intel 处理器为控制散热与功耗，会自动降低其基础时钟频率(Base Clock Frequency)。这种架构层面的权衡(Trade-off)导致部分编译器在历史上曾默认采用 AVX2 指令集，以优先保障持续稳定的时钟频率，而非追求更宽的数据通道(Data Lanes)。其影响之大，以至于 Intel 最终在部分消费级芯片(Consumer Chips)上移除了对 AVX-512 的支持，以避免用户感知到性能倒退。尽管新一代微架构已缓解了该问题，但数据库工程师仍需意识到：在处理混合工作负载时，较窄的 SIMD 位宽有时反而优于 AVX-512，因为它们有效规避了降频惩罚(Throttling Penalty)带来的性能损耗。

![关键帧](keyframes/part003_frame_00149699.jpg)
![关键帧](keyframes/part003_frame_00188849.jpg)

## 谓词掩码(Predicate Masks)：条件化 SIMD 执行的核心
AVX-512 引入的一项变革性特性是专用的**谓词掩码寄存器(Predicate Mask Registers)**（最多支持 32 个）。与早期 SIMD 架构需占用宝贵的数据寄存器来模拟条件逻辑不同，AVX-512 允许直接通过位掩码(Bitmask)来门控(Gate)指令的执行。掩码中的每一位精确对应一个 SIMD 数据通道(Lane)：位值为 `1` 表示启用该通道的运算，为 `0` 则表示跳过。开发者可在**合并掩码(Merge Masking)**与**零掩码(Zero Masking)**之间进行选择：合并掩码会保留非活动通道(Inactive Lanes)在目标寄存器中的原有值，而零掩码则会将非活动通道的结果自动置零。这一特性是现代向量化数据库的基石，它使得 `WHERE` 子句求值、`NULL` 值处理以及复杂过滤链(Filter Chains)得以高效执行，无需依赖代价高昂的分支跳转(Branch Jumps)或会破坏流水线连续性的分散写入(Scatter Operations)。

![关键帧](keyframes/part003_frame_00279533.jpg)
![关键帧](keyframes/part003_frame_00476116.jpg)

## 使用置换原语(Permutation Primitives)构建查询算子
在高效处理谓词过滤后，向量化引擎进一步依赖数据重排原语(Data Rearrangement Primitives)来构建更复杂的查询算子。**置换(Permute)**操作作为核心基础模块，允许 CPU 根据指定的索引(Indices)或控制向量，有选择性地将一个或多个输入向量中的元素提取并重新排序至输出向量。借助宽寄存器批量重排元组(Tuples)，置换操作彻底消除了传统逐行数据移动带来的标量开销(Scalar Overhead)。当与掩码运算(Mask Arithmetic)和比较指令(Comparison Instructions)结合使用时，置换原语能够高效支撑连接(Joins)、排序(Sorting)及哈希表查找(Hash Table Lookups)等复杂操作，构成了现代 OLAP 查询执行引擎(OLAP Query Execution Engine)的底层基石。

![关键帧](keyframes/part003_frame_00597500.jpg)

---

## 高效的基于寄存器的数据操作
在 AVX-512 问世之前，向量重排(Vector Permutation) 通常需要将数据从寄存器移出至内存，稍后再读回，这一过程会导致缓存污染(Cache Pollution) 并引发性能下降。AVX-512 通过单条高效指令直接在寄存器内完成所有排列操作，从而消除了这一瓶颈。借助索引向量(Index Vector)，CPU 无需经过中间内存步骤，即可将输入值精准映射到目标位置。
![关键帧](keyframes/part004_frame_00000000.jpg)

## 选择性加载与存储操作
掩码操作(Masked Operations) 利用位掩码(Bitmask) 有条件地控制内存与向量寄存器之间的数据流动。选择性加载(Masked Load) 仅在掩码指示为 `1` 的位置从内存中读取数据，跳过掩码为 `0` 的位置，从而保留向量寄存器中的现有内容并避免不必要的覆盖。相反，选择性存储(Masked Store) 遵循完全相同的掩码逻辑，将向量元素写回内存，确保只有指定的通道(Lane) 被更新。这两个方向的操作均可作为单条指令执行。
![关键帧](keyframes/part004_frame_00058400.jpg)

## 压缩与扩展变换
压缩(Compress) 操作会将活跃元素（即掩码为 `1` 的位置）紧凑地打包到目标向量的起始处，当目标空间用尽时，剩余通道将被清零或忽略。其反向操作为扩展(Expand)，它根据掩码将紧凑的数据序列重新分布到完整向量中，并将未使用的通道填充为零。这些互补指令使得完全在寄存器内进行高效的、由掩码驱动的数据重塑(Data Reshaping) 成为可能。
![关键帧](keyframes/part004_frame_00107650.jpg)
![关键帧](keyframes/part004_frame_00132999.jpg)
![关键帧](keyframes/part004_frame_00170783.jpg)

## 散布与收集内存访问
散布(Scatter) 和收集(Gather) 指令专门用于处理非连续的内存访问模式。收集(Gather) 指令使用索引向量从分散的内存地址获取数据，并按指定顺序将其组装到连续的向量寄存器中。散布(Scatter) 指令则执行相反的操作，将向量通道中的数据写入由索引指定的不同内存地址。尽管硬件会自动处理内存对齐问题，但为了匹配 L1 缓存(L1 Cache) 的加载/存储吞吐量限制并避免消耗过多时钟周期，这些操作的性能优化通常以缓存行(Cache Line) 为单位进行。
![关键帧](keyframes/part004_frame_00195199.jpg)
![关键帧](keyframes/part004_frame_00233016.jpg)

## 数据库中的纵向 SIMD 向量化
将这些架构特性应用于数据库系统时，优化重点从横向 SIMD 向量化(Horizontal SIMD Vectorization) 转向了纵向 SIMD 向量化(Vertical SIMD Vectorization)，以最大化通道利用率。通过在独立的通道中并行处理多个元组(Tuple)，系统可以避免在已被过滤或谓词评估结果为假的数据上浪费时钟周期。尽管 2015 年的基础研究基于 32 位指针和数据常驻 L3 缓存(L3 Cache) 的假设已略显过时，但现代实现已针对现实中的大规模工作负载，对这些位掩码和索引向量进行了充分适配与优化。
![关键帧](keyframes/part004_frame_00298733.jpg)

## 无分支扫描实现
在实现无分支扫描(Branchless Scan) 时，系统采用位运算替代传统的 `if-else` 条件分支逻辑，以维持 SIMD 流水线(SIMD Pipeline) 的执行效率。系统将键值加载到 SIMD 寄存器中，并针对查询的范围边界执行并行比较操作。每次比较都会生成一个独立的比较掩码，随后通过 SIMD 逻辑与(AND) 操作将这些掩码合并，生成最终的谓词掩码(Predicate Mask)，从而精准识别出满足条件的元组。
![关键帧](keyframes/part004_frame_00418333.jpg)
![关键帧](keyframes/part004_frame_00453833.jpg)

## 位掩码压缩与优化实践
逐步演示展示了系统如何评估八个单字符键值与范围边界的匹配关系。系统首先生成两个比较掩码，将其进行逻辑组合后，结合一个包含 `0-7` 顺序索引的偏移向量(Offset Vector)，交由压缩指令进行处理。该过程将符合条件元组的精确索引提取至一个紧凑的寄存器中。额外的硬件优化指令（例如 `` `rank` `` 指令）可快速统计最终掩码中置位（值为 `1`）的比特数，以便在未找到匹配项时触发提前退出(Early Exit) 机制，从而节省不必要的计算周期。
![关键帧](keyframes/part004_frame_00518516.jpg)

---

## 将向量化集成到查询原语中
在实际数据库系统中实现单指令多数据流(Single Instruction, Multiple Data, SIMD) 通常依赖于调用显式向量化函数(Explicit Vectorization Functions) 或依赖编译器自动向量化(Compiler Auto-Vectorization)。在处理 `WHERE` 子句(WHERE Clause) 时，系统会将列值(Column Values) 与常量进行比较并生成结果。一个关键的设计抉择在于：是立即生成匹配偏移量列表(Match Offset List)，还是保留中间位掩码(Intermediate Bitmask)。位掩码(Bitmask) 极具价值，因为它们可以在最终筛选(Final Selection) 之前进行逻辑组合（例如通过 SIMD 逻辑与(SIMD AND) 操作），从而高效处理复杂的谓词逻辑(Predicate Logic)。为每一种可能的查询变体(Query Variant) 生成最优的底层原语序列(Primitive Sequence) 并非易事，通常需要超出标准自动向量化能力范围的人工优化(Manual Tuning)。
![关键帧](keyframes/part005_frame_00000000.jpg)

## AVX-512 基准测试与系统开销
针对哈希计算(Hash Computation)、内存收集(Gather) 和连接探测(Join Probing) 等核心操作的 AVX-512 微基准测试(Microbenchmarking) 展现了显著的加速效果，相比标量执行(Scalar Execution) 最高可实现 2.3 倍的性能提升。然而，当这些向量化原语(Vectorized Primitives) 被集成到完整的查询引擎(Query Engine) 中时，整体性能提升通常会缩减至 10% 左右。这一差异可以通过阿姆达尔定律(Amdahl's Law) 来解释：数据物化(Data Materialization)、算子间通信(Inter-Operator Communication) 以及内存移动(Memory Movement) 的开销稀释了底层计算带来的收益。尽管如此，这些优化效果具有累积性，最大化每一执行层级的处理速度对于实现系统级数量级(System-Level Order-of-Magnitude) 的性能跃升至关重要。
![关键帧](keyframes/part005_frame_00165899.jpg)
![关键帧](keyframes/part005_frame_00217433.jpg)
![关键帧](keyframes/part005_frame_00246449.jpg)

## SIMD 通道利用率难题
向量化查询执行(Vectorized Query Execution) 中一个长期存在的难题是由“无效元组”(Filtered-Out Tuples) 导致的向量通道(Lane) 利用率低下。当应用过滤谓词(Filter Predicate)（例如 `age > 20`）时，SIMD 寄存器(SIMD Register) 中的部分通道将包含不满足条件的数据。在标量执行(Scalar Execution) 中，这些数据会被直接跳过。但在向量化流水线(Vectorized Pipeline) 中，失效的元组仍会滞留在寄存器内，并被不必要地传递至聚合(Aggregation) 或连接(Join) 等后续阶段，从而浪费计算资源与缓存带宽(Cache Bandwidth)。为了维持高吞吐量(High Throughput)，系统必须阻止无效数据在执行流水线中持续向下传播。
![关键帧](keyframes/part005_frame_00346349.jpg)
![关键帧](keyframes/part005_frame_00376249.jpg)
![关键帧](keyframes/part005_frame_00381650.jpg)

## 人造流水线断点与宽松算子融合
为了消除无效元组的传播问题，现代向量化引擎(Vectorized Engine) 引入了人造流水线断点(Synthetic Pipeline Breakers)。系统不再将所有算子(Operators) 融合到一个不间断的单一循环中，而是插入中间暂存缓冲区(Intermediate Staging Buffer)。执行引擎会扫描并过滤输入向量，仅将符合条件的元组写入紧凑缓冲区(Compact Buffer) 中。一旦缓冲区达到容量上限，流水线便会安全地过渡到下一阶段（例如聚合阶段），从而保证 100% 的通道利用率(Lane Utilization)。这种“宽松算子融合(Relaxed Operator Fusion)”架构在融合流水线(Fused Pipeline) 的高效性与数据物化点(Data Materialization Points) 的灵活性之间取得了精妙平衡。
![关键帧](keyframes/part005_frame_00418999.jpg)
![关键帧](keyframes/part005_frame_00428733.jpg)
![关键帧](keyframes/part005_frame_00468566.jpg)
![关键帧](keyframes/part005_frame_00506699.jpg)

## 分阶段缓冲区实现与预取技术
其具体实现遵循一个紧凑循环(Tight Loop)：扫描输入元组，通过 SIMD 比较(SIMD Comparison) 进行过滤，并将有效结果逐步打包至暂存缓冲区(Staging Buffer) 中。若缓冲区未满，循环将继续提取更多数据；一旦填满，控制权即移交至向量化聚合阶段(Vectorized Aggregation Phase)。这种分阶段架构(Phased Architecture) 不仅确保了清晰的数据边界(Data Boundaries)，还为软件预取(Software Prefetching) 创造了理想条件。尽管 CPU 能够自动处理顺序访问的硬件预取(Hardware Prefetching)，但开发人员仍可在该紧凑的暂存循环中注入显式的预取指令(Prefetch Instructions)，主动将后续的内存块提前加载至缓存中，从而显著降低下一次数据迭代过程中的内存延迟(Memory Latency)。
![关键帧](keyframes/part005_frame_00529850.jpg)
![关键帧](keyframes/part005_frame_00542466.jpg)
![关键帧](keyframes/part005_frame_00573016.jpg)

---

## 软件预取与整体查询编译
现代 x86 架构(x86 Architecture) 允许开发者向 CPU 发送显式的软件预取提示(Software Prefetch Hint)，告知处理器哪些内存区域即将被访问。尽管硬件并未强制必须遵循这些提示，但当它们被集成到紧凑的执行循环(Tight Execution Loop) 中时，仍能带来显著的性能提升。当该技术结合整体查询编译(Whole-Query Compilation) 与宽松算子融合(Relaxed Operator Fusion) 时，预取机制能够创建自然的流水线边界(Pipeline Boundary)，从而大幅降低缓存未命中(Cache Miss) 率。性能基准测试(Performance Benchmark) 表明，从朴素的解释执行模型(Naïve Interpretation Model) 迁移至完全编译、向量化且分阶段的执行流水线，可带来惊人的性能改进。尽管最大理论收益（例如约 97%）反映的是从未经优化的基线代码(Unoptimized Baseline Code) 升级而来的跨度，但一致的结论表明：引入暂存缓冲区(Staging Buffer) 与激进的向量化技术(Aggressive Vectorization)，能够为各类查询工作负载(Query Workload) 提供稳定且渐进的性能加速。
![关键帧](keyframes/part006_frame_00000000.jpg)
![关键帧](keyframes/part006_frame_00028466.jpg)
![关键帧](keyframes/part006_frame_00073900.jpg)

## 高级向量回填策略
SIMD 执行(SIMD Execution) 中长期存在的一个挑战是：在过滤操作(Filter Operation) 丢弃无效数据后，如何高效处理部分填充的寄存器。为了在避免频繁物化结果(Frequent Result Materialization) 的同时最大化向量通道(Lane) 利用率，先进的系统采用了动态向量回填策略(Dynamic Vector Backfilling Strategy)。**缓冲策略(Buffering Strategy)** 使执行过程保持在同一算子(Operator) 内部，利用额外的寄存器临时暂存有效元组。一旦主寄存器填满，SIMD 压缩指令(SIMD Compress Instruction) 会将暂存数据与当前批次数据合并，随后传递给下游算子。**部分回填策略(Partial Backfill Strategy)** 则更为激进：它会将控制权交回子算子(Child Operator) 以拉取新的输入元组，试图以更细的粒度填充空闲通道。尽管该策略在理论上具有更高的执行效率，但跨算子跟踪写入位置所需的状态维护开销(State Maintenance Overhead) 极为复杂。因此，大多数生产级数据库(Production-Grade Database) 为简化实现，通常选择直接将无效元组沿流水线传递至后续阶段。
![关键帧](keyframes/part006_frame_00140533.jpg)

## 哈希表的横向 SIMD 方法
由于指针追逐(Pointer Chasing) 特性和非连续内存布局(Non-Contiguous Memory Layout)，哈希表(Hash Table) 众所周知地对 SIMD 极不友好。为了解决这一难题，**横向向量化(Horizontal Vectorization)** 策略直接修改了哈希表的物理结构：每个哈希桶(Hash Bucket) 存储四个键(Key) 和四个对应的值(Value)。在查找(Lookup) 阶段，系统对单个探测键(Probe Key) 进行哈希计算以定位目标桶，随后将该探测键广播(Broadcast) 至 SIMD 寄存器的各个通道中。接着，单条并行比较指令(Parallel Compare Instruction) 将探测键与桶内存储的四个键同时进行比对，生成标识匹配项的位掩码(Bitmask)。尽管该设计颇具巧思，但面临通道利用率不可预测的挑战。空闲槽位(Empty Slot) 或负载稀疏的桶意味着比较操作经常在无效或空数据上执行，从而难以充分发挥硬件的峰值吞吐量(Peak Throughput)。
![关键帧](keyframes/part006_frame_00333383.jpg)
![关键帧](keyframes/part006_frame_00351066.jpg)

## 纵向哈希探测与输出排序挑战
**纵向向量化(Vertical Vectorization)** 方法保留了传统的哈希表结构（每个槽位存储一个键），但改为并行处理多个探测键。借助 SIMD 哈希计算与收集指令(Gather Instruction)，执行引擎将多个分散的桶条目提取至单个向量寄存器(Vector Register) 中进行同步比较。当发生哈希冲突(Hash Collision) 或特定通道提前完成匹配时，这些通道会动态从输入流中回填新的探测键，从而在多轮迭代中保持所有 SIMD 单元处于高负载状态。尽管纵向哈希探测(Vertical Hash Probing) 的性能始终优于横向方法，但它引入了一个关键的正确性挑战：输出确定性(Output Determinism)。由于各通道的回填与处理速率存在差异，最终输出的元组顺序可能不再严格匹配原始输入顺序。若查询执行(Query Execution) 要求严格的结果顺序一致性，系统则需引入额外的排序或标记机制来保证语义正确性。
![关键帧](keyframes/part006_frame_00438100.jpg)
![关键帧](keyframes/part006_frame_00521200.jpg)

---

## 关系输出排序与调试权衡
哈希连接(Hash Join) 中的纵向向量化(Vertical Vectorization) 会并行处理多个探测键(Probe Key)，这本质上会导致结果解析与输出顺序的非确定性。尽管标准关系代数(Standard Relational Algebra) 并不强制要求严格的物理顺序，但这种非确定性在排查流水线异常(Pipeline Anomalies) 时，可能会增加调试(Debugging) 与结果复现(Result Reproducibility) 的难度。开发者必须认识到这一权衡(Trade-off)：正确的查询结果在不同次执行中可能会以不同的顺序呈现。
![关键帧](keyframes/part007_frame_00000000.jpg)

## 缓存驻留与现代处理器架构
SIMD 加速哈希连接(Hash Join) 的性能优势在很大程度上受限于内存层级(Memory Hierarchy) 的边界。当哈希表(Hash Table) 大小超出 CPU 缓存容量(CPU Cache Capacity) 时，受内存延迟(Memory Latency) 增加的影响，其相较于标量执行(Scalar Execution) 的吞吐量优势会迅速衰减。然而，现代处理器架构通过配备大容量三级缓存(L3 Cache) 显著缓解了该瓶颈，部分 AMD 芯片在单插槽(Single-Socket) 配置下即可提供高达 800MB 的缓存。通过策略性地将较小的数据集指定为连接的构建端(Build Side)，系统可使哈希结构完全驻留于缓存(Cache-Resident) 中，从而在避免内存访问性能损耗的前提下，充分释放 SIMD 的吞吐潜力。
![关键帧](keyframes/part007_frame_00045000.jpg)
![关键帧](keyframes/part007_frame_00084566.jpg)

## 无冲突 SIMD 直方图构建
在并行构建直方图(Parallel Histogram Construction) 时，若多个 SIMD 通道(SIMD Lane) 将不同的输入键哈希至同一桶(Bucket) 中，便会引发写入冲突(Write Conflict)。为避免使用代价高昂的原子操作(Atomic Operation) 或引入串行化(Serialization) 开销，系统会跨 SIMD 通道复制直方图数组(Histogram Array)，形成列式布局(Columnar Layout)，使每个通道仅在专属的内存区域进行累加。待所有通道完成本地计数(Local Counting) 后，系统通过一次横向归约操作(Horizontal Reduction) 将各列求和，最终合并为精确的全局直方图。该技术彻底消除了竞态条件(Race Condition)，同时保持了满额的向量通道利用率(Vector Lane Utilization)。

## AVX-512 降频现象
更宽的向量指令(Wider Vector Instructions) 并不自动等同于更快的执行速度。在许多 Intel 架构(Intel Architecture) 上，执行高负载的 AVX-512 工作负载会触发功耗与散热管理(Power and Thermal Management) 机制，导致 CPU 核心频率发生激进降频(Aggressive Downclocking)。这种动态频率调节(Dynamic Frequency Scaling) 往往会抵消 512 位寄存器带来的理论吞吐量收益(Theoretical Throughput Gain)，有时反而使 AVX-2 成为性能更优且更稳定的选择。AMD 处理器通常采用将 512 位操作拆分为成对的 256 位执行单元(256-bit Execution Units) 进行处理的方式，从而有效规避了该降频惩罚(Downclocking Penalty)，维持了时钟频率的稳定性。因此，GCC 和 Clang 等现代编译器(Compiler) 通常默认采用 AVX-2 内建函数(AVX-2 Intrinsics)。数据库工程师必须严格审查第三方库(Third-Party Libraries)，以防在生产环境负载(Production Workload) 中意外触发降频。
![关键帧](keyframes/part007_frame_00199999.jpg)
![关键帧](keyframes/part007_frame_00208999.jpg)
![关键帧](keyframes/part007_frame_00229183.jpg)
![关键帧](keyframes/part007_frame_00242749.jpg)

## 结论与查询执行的未来方向
尽管 SIMD 向量化(SIMD Vectorization) 是现代数据库引擎(Database Engine) 的核心优化技术，但唯有将其融入更宏观的执行策略(Execution Strategy) 中，方能释放最大效能。为最大化硬件效率(Hardware Efficiency)，系统需将核内并行(In-Core Parallelism) 与优化的数据布局(Optimized Data Layout) 紧密结合；这通常依托编译器自动向量化(Compiler Auto-Vectorization) 实现，并辅以定向内建函数(Targeted Intrinsics) 或硬件无关库(Hardware-Agnostic Libraries)。未来的演进方向聚焦于整体查询编译(Whole-Query Compilation)，该技术能够启用推送式(Push-Based)、以数据为中心(Data-Centric) 的执行模型。此架构允许对寄存器分配(Register Allocation)、缓存数据流转(Cache Data Movement) 及算子融合(Operator Fusion) 实施细粒度控制(Fine-Grained Control)，从而为下一代高度优化的查询处理器(Query Processor) 奠定坚实基础。
![关键帧](keyframes/part007_frame_00377133.jpg)