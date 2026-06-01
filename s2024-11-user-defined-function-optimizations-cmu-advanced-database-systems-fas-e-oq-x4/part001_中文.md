## 课程概述：UDF 优化技术
本节首先概述了优化用户定义函数(User-Defined Function, UDF)的三种主要技术：内联技术(Inlining)（参考微软相关论文）、将 UDF 转换为结合横向连接(Lateral Join)的公共表表达式(Common Table Expression, CTE)，以及批处理(Batching)策略。讲座还将评估当前主流现代数据库系统对这些优化方法的支持情况。
![关键帧](keyframes/part001_frame_00000000.jpg)

## SQL 函数与外部语言 UDF
用户定义函数(UDF)通常接收输入参数、执行处理逻辑并返回输出结果，但它们可划分为两个截然不同的类别。第一类是纯 SQL 函数(Pure SQL Function)，其函数体完全由顺序执行的 SQL 语句构成，最终查询的结果集将直接作为函数返回值。此类函数既可直接调用，也可嵌入至 `FROM` 或 `WHERE` 子句中。从查询优化(Query Optimization)的角度来看，SQL 函数的行为类似于宏(Macro)：数据库引擎会将函数体内的 SQL 语句直接展开并内联至主查询中，形成嵌套查询结构，从而使查询优化器(Query Optimizer)能够对其进行全局分析与优化。正是由于这种透明性，SQL Server 等系统会严格限制 UDF 执行 `UPDATE` 等数据修改操作。若将数据变更语句混入展开后的查询中，极易打乱原有的执行顺序，进而引发不可预测的副作用(Side Effects)。
![关键帧](keyframes/part001_frame_00020416.jpg)
在随后的问答环节中对此作了进一步澄清：若允许在 SQL 函数内部执行数据更新操作，数据库通常会强制采用严格的顺序执行模型，即按顺序复制并逐一执行这些语句。该机制与 PostgreSQL 中的视图重写规则(View Rewriting Rule)颇为相似。
![关键帧](keyframes/part001_frame_00072416.jpg)

## 历史背景与过程语言标准
第二类涵盖使用外部过程语言(External Procedural Language)编写的 UDF。SQL 标准为此定义了持久存储模块(Persistent Stored Module, SQL/PSM)规范，但各大数据库厂商通常会根据自身需求实现特定的方言(Dialect)。这类语言在语法结构上与 Ada 或 Pascal 相似，通常要求显式声明变量。主流实现包括 Oracle 的 PL/SQL、PostgreSQL 的 PL/pgSQL 以及 IBM DB2 的 SQL PL。本次讲座将重点聚焦于 Transact-SQL(T-SQL)。T-SQL 由 Sybase 于 20 世纪 80 年代开发，是业界早期支持 UDF 的先驱之一。微软在 90 年代初获得 Sybase 代码库授权并推出 SQL Server，由此奠定了 T-SQL 中采用 `@` 前缀定义变量以及其鲜明的过程化编程(Procedural Programming)风格。尽管 Sybase 如今仅留存于部分传统企业环境中，但 T-SQL 依托 SQL Server 的广泛应用，至今仍保持着强劲的发展势头。
![关键帧](keyframes/part001_frame_00127400.jpg)

## 语言生态、安全性与实际优势
PostgreSQL 支持使用多种编程语言编写 UDF，涵盖 Tcl、Python、Perl 甚至 C 语言。然而，业界强烈不建议直接使用基于 C 语言编写的 UDF，因其存在显著的安全与稳定性风险：这类函数直接在数据库进程的地址空间(Address Space)中运行，若内存管理不当，极易导致整个数据库服务器崩溃(Crash)。Oracle 数据库通过将 C 语言代码转译为更安全的中间表示，并将其隔离在独立的进程(Independent Process)中运行，从而有效缓解了该风险。尽管引入外部语言 UDF 会带来一定的复杂性，但其优势依然显著：它们能够促进业务逻辑在不同应用层（如移动端与 Web 后端）之间的复用，彻底消除冗余的网络往返(Network Round-Trip)开销，并大幅简化复杂业务规则或分析逻辑的表达——若仅依赖纯 SQL 实现这些逻辑，往往显得冗长且难以维护。
![关键帧](keyframes/part001_frame_00345749.jpg)

## 优化挑战：不透明性与执行限制
制约 UDF 广泛采用的核心瓶颈在于查询优化器面临的黑盒不透明性(Black-Box Opacity)。当 UDF 采用外部语言编写时，查询优化器(Query Optimizer)无法探查其内部实现逻辑，因而难以准确估算执行成本(Execution Cost)或结果集的选择性(Selectivity)。例如，在 `WHERE` 子句中将数据列与 UDF 的返回值进行比较时，优化器因无法预知函数行为，往往只能生成次优甚至低效的执行计划(Execution Plan)。这种逻辑隔离也严重阻碍了查询的并行化(Parallelization)与向量化执行(Vectorized Execution)，常常迫使数据库引擎退化为低效的逐行处理模式（即隐式嵌套循环连接(Implicit Nested Loop Join)）。在极端场景下，UDF 甚至会在运行时动态拼接并执行 SQL 字符串，这使优化器完全丧失全局视野，彻底无法进行有效的全局查询规划(Global Query Planning)。
![关键帧](keyframes/part001_frame_00400666.jpg)
![关键帧](keyframes/part001_frame_00461749.jpg)