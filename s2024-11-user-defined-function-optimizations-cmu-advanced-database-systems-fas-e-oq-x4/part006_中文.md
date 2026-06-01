## 从 SSA 到递归 CTE
在编译流水线(Compilation Pipeline)中，用户定义函数(User-Defined Function, UDF)首先被转换为静态单赋值(Static Single Assignment, SSA)形式。这是传统编译器(Compiler)中的一种标准技术，其核心原则是每个变量在整个执行流程中仅被赋值一次。随后，该SSA表示形式被进一步转换为管理范式(Administrative Normal Form, ANF)。此阶段对控制逻辑进行结构化重组，以支持尾递归(Tail Recursion)的生成。在该范式下，函数可递归调用自身或其他函数，形成能够高效表示循环结构的调用链。所有的计算操作均发生在递归尾调用之前，当满足终止条件时，最后一次迭代将跳出循环并返回最终结果。为进一步提升简化程度，相互递归(Mutual Recursion)会被转换为直接递归(Direct Recursion)。这确保了递归调用仅作为函数执行的最后一步，且控制流保持单向传递。这一特性保证了最内层递归调用的输出能够准确映射至最外层的 `SELECT` 语句，从而允许将完整的过程式逻辑无缝嵌入嵌套的SQL查询中。

![关键帧](keyframes/part006_frame_00000000.jpg)

将直接递归形式转换为SQL的过程，核心在于将控制流与变量状态精确映射至递归公共表表达式(Recursive Common Table Expression, Recursive CTE)。编译器会构建嵌套查询以完成变量初始化，转换 `IF/ELSE` 条件分支，并将核心递归逻辑直接注入SQL语句。借助这一系统化的映射机制，复杂的过程式循环与分支结构被成功转化为标准的、可供查询优化器深度优化的关系型SQL。

![关键帧](keyframes/part006_frame_00023766.jpg)

![关键帧](keyframes/part006_frame_00090566.jpg)

## 性能对比与优化器限制
在评估该方法时，研究人员将其与 PostgreSQL(Postgres) 等系统中的原生UDF执行机制进行了对比。此类系统通常缺乏类似微软Froid的内置内联(Inlining)功能。对于逻辑极为简单的函数，由于解析复杂CTE会引入额外开销，直接调用原生UDF的实际执行速度反而可能更快。然而，当UDF作用于超大规模数据表时，基于集合的CTE(Set-based CTE)方案便开始展现出显著的性能优势。尽管存在优势，但其性能提升幅度仍不及在 SQL Server 中显著，这主要归因于 PostgreSQL 的查询优化器(Query Optimizer)在激进优化方面相对保守。该优化器尚无法执行同等深度的优化策略（例如常量折叠(Constant Folding)、死代码消除(Dead Code Elimination)或高级连接重排序(Join Reordering)），从而限制了此编译转换方法的理论性能上限。

![关键帧](keyframes/part006_frame_00137499.jpg)

![关键帧](keyframes/part006_frame_00168699.jpg)

## 批处理方法：状态表与顺序更新
作为递归CTE的替代方案，“批处理”(Batch Processing)技术最初由学术研究探索提出，随后其演进方向与FEL项目及早期Froid论文的研究路径逐渐趋同。与Froid架构不同，批处理方案无需对底层查询优化器进行任何修改。取而代之的是，它将UDF逻辑转换为一系列作用于临时状态表(Temporary State Table)的 `UPDATE` 语句。该状态表实质上是UDF过程式变量(Procedural Variables)的关系型映射载体。在调用UDF时，系统会动态创建一个临时表，其列结构严格对应函数内部定义的各个局部变量。随后，系统通过执行一系列 `UPDATE` 查询来模拟过程式步骤，以批处理方式集中更新状态表，彻底摒弃了传统的逐行(Row-by-Row)执行模式。

![关键帧](keyframes/part006_frame_00238233.jpg)

## 执行机制与广泛的泛化能力
状态表中设有一个特殊的布尔型返回列，用于标记哪些元组(Tuple)应当作为最终结果集返回。表中的每一行均精确对应一个传入UDF的输入元组。批处理机制摒弃了单条记录的独立处理模式，转而针对所有输入元组并发更新整个状态表。这种基于集合的执行(Set-based Execution)模式使得成千上万个UDF调用得以并行处理，每一行数据均通过 `UPDATE` 语句独立维护并演进其状态。迭代计算通常借助 `generate_series` 函数结合横向连接(Lateral Join)来驱动控制。

![关键帧](keyframes/part006_frame_00385999.jpg)

该架构具备极强的泛化能力，甚至有潜力支持那些会破坏传统内联机制的特性，例如异常处理(Exception Handling)与动态SQL(Dynamic SQL)。其中，异常状态可通过追踪表内的状态变迁来进行管理；而动态查询则可通过将拼接生成的SQL字符串暂存为状态变量来妥善处理。这使得批处理方案成为一种更为灵活、且与优化器无关(Optimizer-Agnostic)的过程式执行替代方案。

![关键帧](keyframes/part006_frame_00405033.jpg)

![关键帧](keyframes/part006_frame_00566166.jpg)

## ProcBench 与实际 UDF 分类
为科学评估上述技术，Froid团队开发了 `ProcBench` 基准测试(Benchmark)。该基准旨在高度还原企业生产环境中观察到的真实UDF工作负载(Workload)特征。`ProcBench` 依据结构特征将UDF划分为多个类别，以便针对性地验证不同的优化路径。其中的首要大类为无输入参数(Parameter-less)的UDF。例如，形如 `SELECT max_return_reason_web()` 的函数无需传入参数，通常仅执行单次调用即可返回静态值或聚合结果。深入理解此类分类体系，有助于数据库研究人员针对生产系统中最常见的主流过程模式，量身定制高效的编译优化策略。

![关键帧](keyframes/part006_frame_00590983.jpg)