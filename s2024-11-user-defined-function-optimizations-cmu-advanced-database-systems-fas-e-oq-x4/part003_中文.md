## 查询重写与子查询展平
![关键帧](keyframes/part003_frame_00000000.jpg)
在优化用户定义函数(User-Defined Function, UDF)之前，理解查询优化器(Query Optimizer)如何处理嵌套子查询(Nested Subquery)至关重要。在执行计划(Execution Plan)不支持有向无环图(Directed Acyclic Graph, DAG)的数据库系统中，优化器必须对这些嵌套结构进行重写。理想的优化策略是去相关化(Decorrelation)与展平(Flattening)，即将嵌套查询转换为标准的连接操作(Join Operation)。例如，当查询试图将 `order` 表与 `user` 表进行连接时，高级优化器可能会识别出所需的所有数据实际上已完整存在于 `order` 表中。在此最佳情况下，优化器能够完全消除对 `user` 表的访问，避免逐行评估(Row-by-Row Evaluation)，从而获得显著的性能提升。若无法实现展平，优化器则会回退至物化策略(Materialization)，将子查询结果存储至临时表(Temporary Table)中，以便后续执行连接操作。
![关键帧](keyframes/part003_frame_00015449.jpg)
![关键帧](keyframes/part003_frame_00023416.jpg)

## 横向连接在执行顺序中的作用
![关键帧](keyframes/part003_frame_00058000.jpg)
横向连接(Lateral Join)是实现 UDF 内联(Inlining)的一项关键机制。与标准连接(Standard Join)不同（标准连接中 `FROM` 子句内的子查询相互隔离且无法交叉引用），横向连接允许子查询直接引用同一嵌套层级中前置表(Preceding Table)的列或属性。这一特性对于保留过程式 UDF(Procedural UDF)逻辑所要求的严格执行顺序至关重要。从概念上讲，其工作原理类似于嵌套循环(Nested Loop)：针对前置表中的每一个元组(Tuple)，横向子查询都会独立执行，并能完全访问外部上下文(Outer Context)。这确保了变量间的依赖关系能够按序解析，从而精准复现原始函数的控制流(Control Flow)。
![关键帧](keyframes/part003_frame_00119316.jpg)

## “Freud”方法：五步内联概述
![关键帧](keyframes/part003_frame_00166399.jpg)
“Freud”内联技术通过结构化的五个步骤，将过程式 UDF 转换为声明式关系代数(Declarative Relational Algebra)表达式。首先，将过程式语句(Procedural Statements)（如 T-SQL、PL/pgSQL）转换为等效的 SQL 查询。其次，依据数据依赖关系(Data Dependency)将 UDF 划分为多个逻辑区域(Logical Regions)。第三步，对每个区域内的表达式进行合并优化。第四步，利用横向连接将这些逻辑区域串联，以严格保留原有的执行顺序。最后，在查询进入基于成本的优化器(Cost-Based Optimizer, CBO)之前，将生成的 SQL 结构直接嵌入至调用查询中。该方法依赖于静态转换规则(Static Transformation Rules)，使得标准优化器无需依赖专门的成本模型(Cost Model)即可高效处理生成的子查询。

## 第一步：将过程式逻辑转换为 SQL
![关键帧](keyframes/part003_frame_00257649.jpg)
初始转换步骤将 UDF 语句直接映射(Mapping)为标准的 SQL 结构。变量声明与赋值操作会被转换为不含 `FROM` 子句的 `SELECT` 语句，用于对常量值或表达式进行投影(Projection)。过程式赋值操作（例如将聚合结果(Aggregation Result)存储至变量中）则会被转换为嵌套的 `SELECT` 查询，并为输出结果设置别名。
![关键帧](keyframes/part003_frame_00307366.jpg)
关键在于，`IF` 等控制流结构(Control Flow Structures)会被替换为符合 SQL 标准的 `CASE WHEN` 表达式。例如，针对 `total` 变量的条件检查会被转化为一个 `CASE` 代码块，依据预设阈值输出 `"platinum"` 或 `NULL`，随后将其投影至最终结果集中。尽管此示例展示了一对一的映射关系，但在处理复杂逻辑时，系统可能需要合并多条语句，或将单条语句拆分至多个 SQL 查询中。
![关键帧](keyframes/part003_frame_00322033.jpg)

## 第二步与第三步：区域划分与临时表生成
![关键帧](keyframes/part003_frame_00390199.jpg)
完成语法转换后，优化器将 UDF 划分为离散的区域(Discrete Regions)以隔离依赖关系。每个区域转换后的 SQL 代码会被封装至子查询中，并物化(Materialization)为瞬时的内存临时表(In-Memory Temporary Table)（例如 `ER1`、`ER2`）。这些表充当仅在查询执行期间存活的中间结果集(Intermediate Result Set)。通过此种代码划分方式，系统能够显式追踪 `total` 或 `level` 等变量如何在区域间传递，确保下游区域在不违反作用域规则(Scope Rules)的前提下，正确引用上游区域的计算结果。
![关键帧](keyframes/part003_frame_00395599.jpg)

## 第四步与第五步：使用横向连接进行链接及处理返回
![关键帧](keyframes/part003_frame_00459499.jpg)
为串联这些已划分的逻辑区域，系统采用横向连接（在 SQL Server 中称为 `CROSS APPLY`，在 PostgreSQL/SQLite/Oracle 中称为 `LATERAL`）。该机制使得每个后续区域的子查询均能访问前序区域生成的临时表，从而精准满足区域间的依赖关系（如条件 `total > 1000000`）。
![关键帧](keyframes/part003_frame_00502166.jpg)
UDF 末尾的 `RETURN` 语句仅被视为该组装查询结构的最终投影操作(Final Projection)。通过将所有逻辑封装于一个引用最终临时表的单一 `SELECT` 语句中，过程式函数被完全内联(Inlined)为一个庞大的声明式查询(Declarative Query)。
![关键帧](keyframes/part003_frame_00533933.jpg)
随后，该最终结构被嵌入至原始调用查询中，使基于成本的优化器得以应用标准的连接重排序(Join Reordering)与谓词下推(Predicate Pushdown)等优化策略，从而将执行时间从数小时大幅压缩至毫秒级。

## 处理复杂控制流（问答环节）
![关键帧](keyframes/part003_frame_00541933.jpg)
在转换过程中，常会涉及控制流划分的细节问题，例如为何 `ELSE` 代码块可能被视为独立的区域。实际上，这主要取决于内联算法(Inlining Algorithm)的严格程度与设计策略。尽管从逻辑层面而言，`IF/ELSE` 结构可直接映射为单个 `CASE WHEN` 表达式，但面对复杂的过程式逻辑时，通常需将不同分支拆分为独立区域，以便精准追踪变量状态与执行路径。优化器的核心目标是在查询简洁性与语义正确性之间取得平衡，确保在将最终查询移交至标准优化流程前，所有过程式依赖均能通过横向连接或条件投影(Conditional Projection)得到显式解析。