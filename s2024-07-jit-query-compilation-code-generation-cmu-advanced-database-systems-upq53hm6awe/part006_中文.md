## 历史渊源：20 世纪 70 年代的代码生成
![关键帧](keyframes/part006_frame_00000000.jpg)
最早的查询编译(Query Compilation)实现可追溯至 20 世纪 70 年代 IBM 开创性的 System R 项目。继 PRTV(Potted Relational Test Vehicle) 原型之后，工程师直接基于优化后的查询计划(Query Plan)生成 IBM System/370 汇编代码(Assembly Code)，以高效执行扫描(Scan)与连接(Join)操作。在 CPU 时钟频率(Clock Frequency)极低且硬件性能羸弱的年代，动态解释(Dynamic Interpretation)查询计划将引发难以忍受的延迟。采用代码生成是一项关键的工程决策，旨在从当时极度受限的计算资源中榨取极限性能。

## 工程权衡与向解释执行的转变
尽管代码生成(Code Generation)具备显著的性能优势，但由于维护开销(Maintenance Overhead)过高，IBM 最终在后续的 SQL/DS 与 DB2 等商业版本中弃用了该技术。彼时数据库系统尚处萌芽期，内部应用程序接口(Internal API)频繁变更，致使汇编代码生成逻辑(Code Generation Logic)不断失效，需持续重构。此外，调试(Debugging)工作极为艰巨；一旦查询执行失败，工程师缺乏有效手段将运行时汇编错误(Run-time Assembly Error)追溯至最初生成的源码。在硬件性能逐步提升的背景下，维护与调试高度特化(Highly Specialized)汇编代码的工程负担，已远超其带来的运行时性能收益(Run-time Performance Benefit)。

## Vectorwise：构建时预生成的原语
![关键帧](keyframes/part006_frame_00088566.jpg)
Vectorwise 系统并未采用运行时的即时编译(Just-In-Time, JIT Compilation)，而是转向了构建时特化(Build-time Specialization)方案。该系统利用脚本预生成数百个高度优化的 C++ 函数，穷尽了算子(Operator)、谓词(Predicate)与数据类型(Data Type)的各类组合（例如 `INT32 < constant`、`STRING = constant`）。 
![关键帧](keyframes/part006_frame_00287349.jpg)
![关键帧](keyframes/part006_frame_00296133.jpg)
这些预编译原语(Pre-compiled Primitives)在部署前会被静态链接(Statically Linked)至数据库二进制文件(Database Binary)中。在运行时(Run-time)执行查询时，引擎通过组装指向对应预生成原语的函数指针数组(Array of Function Pointers)，动态构建执行流水线(Execution Pipeline)。 
![关键帧](keyframes/part006_frame_00304549.jpg)
尽管函数指针跳转(Function Pointer Dispatch)对现代 CPU 而言通常代价高昂，但 Vectorwise 采用大型向量批次(Large Vector Batches)处理元组(Tuple)的策略有效缓解了该问题。每次间接调用的开销在整批数据中被大幅摊薄(Amortized)，从而在规避运行时编译延迟(Run-time Compilation Latency)的同时，提供了高度可预测且迅捷的执行性能。

## 运行时编译器选择与优化
![关键帧](keyframes/part006_frame_00341949.jpg)
支撑该方向的学术研究进一步挖掘了预编译(Pre-compilation)的潜力。在实验环境中，研究人员利用 GCC、ICC、Clang 等多种编译器，配合不同的优化标志(Optimization Flag)对所有原语函数进行交叉编译。在运行时，系统内置的轻量级性能分析机制(Lightweight Profiling Mechanism)会评估目标硬件特征，并动态切换(Dynamic Switching)至针对该特定 CPU 架构(CPU Architecture)表现最优的机器码(Machine Code)实现。尽管这种多编译器运行时选择(Multi-compiler Runtime Selection)机制展现出可观的理论收益，但因系统架构过于复杂，最终被判定不具备商业发行(Commercial Release)的可行性。

## Amazon Redshift：全局代码缓存架构
![关键帧](keyframes/part006_frame_00375916.jpg)
Amazon Redshift 采用了类似 HiQ 的转译(Transpilation)技术，将查询片段(Query Fragment)转换为模板化 C++ 代码，深度融合了推送型执行(Push-based Execution)与向量化(Vectorization)技术。为彻底规避为每个查询频繁派生 GCC 进程所带来的巨大延迟开销(Latency Overhead)，Redshift 依托其云原生架构(Cloud-native Architecture)维护了一个庞大的全局编译代码缓存(Global Compiled Code Cache)。系统摒弃了针对独立数据库实例的冷编译(Cold Compilation)模式，转而查询集中式缓存(Centralized Cache)，该缓存汇聚了跨所有 Redshift 租户(Redshift Tenant)编译的查询代码片段。此举实现了高达 99.95% 的全局缓存命中率(Global Cache Hit Rate)。由于生成的代码仅封装特定操作逻辑与常量(Constant)，绝不触碰专有用户数据(Proprietary User Data)，因此跨租户共享缓存(Cross-tenant Cache Sharing)不会引发任何安全或隐私合规风险。每逢系统版本迭代，后台服务会自动执行缓存预热(Cache Warm-up)与重编译(Recompilation)，确保代码获取延迟始终远低于即时 GCC 编译时间。
![关键帧](keyframes/part006_frame_00570200.jpg)

## Oracle：存储过程的转译
尽管 Oracle 并未对标准 SQL 查询实施即时编译(Just-In-Time, JIT Compilation)或谓词特化(Predicate Specialization)，但其成功将转译技术(Transpilation)应用于用户自定义逻辑。由 PL/SQL 编写的存储过程(Stored Procedure)与用户定义函数(User-Defined Function, UDF)会被自动转换为 Pro*C——一种 Oracle 专有的受限 C 语言方言(Restricted C Dialect)。随后，这些生成的 C 代码将编译为原生机器码(Native Machine Code)，彻底消除了过程式数据库逻辑(Procedural Database Logic)的解释开销(Interpretation Overhead)。这一实践充分证明，即便不直接集成于核心查询执行引擎(Core Query Execution Engine)，转译技术在优化应用层数据库函数(Application-level Database Function)方面依然具备极高的工程价值。