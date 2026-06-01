## UDF(User-Defined Function) 的不透明性与 RBAR(Row By Agonizing Row) 问题
用户定义函数(User-Defined Function)对数据库优化器(Database Optimizer)构成了重大挑战，因为其内部的 SQL 逻辑或查询在运行前完全不可见。这种不透明性迫使执行引擎(Execution Engine)陷入微软所称的“逐行痛苦处理”(Row By Agonizing Row)模式。系统未能利用基于集合的处理(Set-based Processing)方式，而是逐行遍历外层查询，并为每一个元组(Tuple)单独调用 UDF。由于优化器无法洞察函数内部逻辑，系统无法识别将操作合并为高效连接(Join)或缓存重复查询结果的机会。尽管静态代码分析(Static Code Analysis)表明该模式在代码库中的出现率不足 5%，但其对运行时性能(Runtime Performance)的影响却极为显著。
![关键帧](keyframes/part002_frame_00000000.jpg)

## 业界共识与严重的性能退化(Performance Degradation)
标量 UDF(Scalar UDF)对查询性能的负面影响已在数据库行业中得到广泛记录与认可。微软在 2006 年的一篇知名博文中直言不讳地将 T-SQL 标量函数称为“魔鬼”，并展示了一个典型案例：仅将业务逻辑封装在 UDF 中，执行时间便从 2.6 秒激增至 38 秒。这并非 SQL Server 独有的问题；所有主流关系型数据库管理系统(RDBMS)都面临同样的瓶颈。截至 2008 年，微软虽发布了关于编译型 UDF(Compiled UDF)的更新文档，但仍严厉批评 RBAR 执行模式，将标量 UDF 描述为事实上的“性能杀手”。业界的共识始终明确：尽管 UDF 显著提升了开发者的生产力与代码模块化程度，但其固有的性能开销要求进行系统性的优化。
![关键帧](keyframes/part002_frame_00087183.jpg)
![关键帧](keyframes/part002_frame_00127283.jpg)

## 案例分析：TPC-H 基准测试(TPC-H Benchmark) 查询 12 与 13 小时的执行耗时骤增
为阐明 UDF 不透明性带来的极端后果，Floyd 的论文以 TPC-H 查询 12 为例，提供了一个极具说服力的案例。原本用于校验客户键(Customer Key)是否为空的简单 `WHERE` 子句，被替换为一个在客户表上执行验证查找(Validation Lookup)的 UDF。在未使用 UDF 的情况下，该查询仅需 0.8 秒即可完成。然而，引入这种过程式包装(Procedural Wrapping)后，执行时间竟暴涨至 13 小时。由于优化器无法识别该函数的简单逻辑，默认采用了逐行评估(Row-by-Row Evaluation)策略，从而产生了巨大的函数调用开销。值得注意的是，包含变量声明或使用显式 `RETURN` 子句的 UDF 会阻碍简单的内联优化，而纯 SQL UDF(Pure SQL UDF，不含过程式元素)则更易于优化。“Freud”项目等先进的内联研究(Inlining Research)表明，将此类函数转换回声明式(Declarative)形式后，执行时间可从 13 小时大幅缩减至约 900 毫秒。
![关键帧](keyframes/part002_frame_00167233.jpg)
![关键帧](keyframes/part002_frame_00185399.jpg)

## UDF 优化的四种核心方法
数据库厂商历来采用四种核心策略来缓解 UDF 带来的性能瓶颈。首先是**编译(Compilation)**，即将解释执行(Interpreted Execution)的 UDF 代码转换为本机机器指令(Native Machine Instructions)。尽管此举降低了单次调用开销（Oracle 与 SQL Server 2016 及以上版本已实现该特性），但对查询优化器而言，该函数依然是一个黑盒(Black Box)。其次是**语言扩展与编译提示(Language Extensions & Pragmas)**，例如 SingleStore 的 MPL 语言，允许开发者嵌入并行化提示(Parallelization Hints)，但优化器的可见性依然有限。第三种也是当前研究的前沿重点是**内联(Inlining)**，它将 UDF 转换为声明式关系代数表达式(Declarative Relational Algebra Expressions)，使其能够无缝融入主查询计划(Query Plan)中进行全局优化。第四种是**批量执行(Batch Execution)**，它将 UDF 重写为可同时处理多个元组的集合查询，从而彻底消除了逐行调用函数的开销。
![关键帧](keyframes/part002_frame_00308033.jpg)

## 内联机制与子查询转换(Subquery Transformation)
现代 UDF 优化高度依赖于内联技术（在学术研究中常以“Freud”项目为代表，在 SQL Server 中则称为“UDF 内联”(UDF Inlining)）。该技术在查询进入基于成本的优化器(Cost-Based Optimizer, CBO)阶段之前，应用静态转换规则将 UDF 逻辑转化为关系代数表达式。内联能否成功，关键取决于数据库处理与重写嵌套子查询(Nested Subquery)的能力。理想情况下，优化器会对这些嵌套查询进行去相关化(Decorrelation)处理，并将其展平(Flattening)为标准的连接(Join)操作。若无法去相关，系统则可能将子查询结果物化(Materialization)至临时表中，以供后续连接操作使用。为保留原始过程式 UDF(Procedural UDF)所要求的严格执行顺序，系统会引入横向连接(Lateral Join)以将各操作按序串联。尽管该技术对声明式 SQL UDF 极为有效，但若底层过程式逻辑过于复杂，此转换流程仍可能失败或导致性能退化。
![关键帧](keyframes/part002_frame_00427533.jpg)
![关键帧](keyframes/part002_frame_00511433.jpg)