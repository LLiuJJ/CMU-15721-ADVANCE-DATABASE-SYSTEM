## Volcano 的自顶向下搜索(Top-Down Search)与分支定界剪枝(Branch and Bound Pruning)
![关键帧](keyframes/frame_00000000.jpg)
Volcano 优化器生成器(Volcano Optimizer Generator)以首创迭代器执行模型(Iterator Execution Model)和并行交换操作符(Parallel Exchange Operators)而闻名，并率先引入了查询计划的自顶向下生成方法之一。该搜索策略并非从基础表(Base Tables)开始自底向上构建，而是从期望的输出结果出发进行逆向推导，逐步组装生成该结果所需的逻辑操作符(Logical Operators)与物理操作符(Physical Operators)。在优化器向下遍历计划树(Plan Tree)的过程中，系统会应用转换规则(Transformation Rules)生成替代实现方案，并自叶节点向根节点逐步累积代价估算(Cost Estimation)。该方法的核心特征之一是分支定界剪枝。若某个物理操作符无法满足必需的属性要求（例如父节点归并连接(Sort-Merge Join)所依赖的特定排序顺序(Sort Order)），则整个搜索分支将被立即剪除。同理，若部分构建路径的累积代价已超过当前已知的最优代价(Current Best Cost)，搜索过程将直接截断该分支，从而避免对次优计划(Suboptimal Plans)进行冗余探索。

![关键帧](keyframes/frame_00042483.jpg)

## Cascades 框架与基于占位符(Placeholders)的惰性展开(Lazy Expansion)
Cascades 框架建立在 Volcano 自顶向下及逆向链推理(Backward Chaining)的基础之上，但有效解决了一个关键的可扩展性瓶颈：在高度复杂查询中极易引发的内存开销爆炸(Memory Explosion)问题。与在每个节点急切生成(Eager Generation)所有可能转换组合的做法不同，Cascades 采用了惰性展开策略。该框架在计划树中插入占位符，用于声明所需的数据属性(Data Properties)与逻辑结构(Logical Structures)，而无需预先绑定具体的物理执行方法。这些占位符仅在搜索过程明确需要时才会被实例化，并依据可配置的优先级系统(Priority System)逐步细化。这种延迟求值机制(Lazy Evaluation Mechanism)有效防止了搜索空间(Search Space)的过早膨胀，确保计算资源被集中投入到最具潜力的转换路径上。

![关键帧](keyframes/frame_00206899.jpg)

## 自包含优化任务(Self-Contained Optimization Tasks)与动态优先级(Dynamic Priorities)
Cascades 围绕模块化、自包含的任务对象构建其优化逻辑。每个任务封装了四个核心组件：待匹配的计划模式(Patterns)、对应的转换规则、定义或强制物理属性(Physical Properties)的元数据(Metadata)，以及执行优先级(Execution Priority)。此架构实现了高度上下文感知(Context-Aware)的优化过程。随着搜索的推进，优化器可依据当前计划状态动态调整转换规则的评估优先级。例如，若上游节点选定索引嵌套循环连接(Index Nested Loop Join)，系统将优先探索索引探测(Index Probe)路径，而非“顺序扫描(Sequential Scan) + 显式排序(Explicit Sort)”的组合，因为前者天然满足所需的排序要求。尽管 Microsoft SQL Server 等商业数据库通常将此类优先级硬编码(Hard-coded)，但 Cascades 框架在理论上支持随搜索树的演进动态重算(Dynamic Recalculation)优先级。

![关键帧](keyframes/frame_00321650.jpg)

## 操作符与表达式的统一优化(Unified Optimization of Operators and Expressions)
Cascades 的一项核心架构优势在于，它能够在单一规则引擎(Rule Engine)内统一处理多个优化领域(Optimization Domains)。该框架将 SQL 谓词(Predicates)与 WHERE 子句表达式视为树状结构(Tree Structures)，其底层抽象与关系操作符树(Relational Operator Trees)完全一致。因此，用于确定连接顺序(Join Ordering)和选择物理操作符的同一套模式匹配(Pattern Matching)与转换机制，可直接复用于表达式简化(Expression Simplification)。诸如将 `1 = 2` 化简为 `FALSE` 的基础常量折叠(Constant Folding)或逻辑化简，能够与复杂的物理计划生成无缝协同处理。这消除了对相互独立、分阶段特定优化遍次(Optimization Passes)的依赖，构建了一个内聚且精简的编译流水线(Compilation Pipeline)，使逻辑优化、物理优化与谓词优化(Predicate Optimization)得以统一执行。

![关键帧](keyframes/frame_00416650.jpg)

## 术语澄清与关系代数(Relational Algebra)基础
在 Cascades 框架中，“表达式”(Expression)一词的内涵已超越传统 SQL 谓词的范畴。它泛指查询计划(Query Plan)中的任意计算节点或子树(Subtree)。逻辑表达式(Logical Expression)可能仅定义一个抽象的三表连接序列，而其对应的物理表达式(Physical Expression)则明确指定具体的执行算法，例如基于顺序扫描的哈希连接(Hash Join)或基于索引探测的嵌套循环连接(Nested Loop Join)。应用于这些表达式的每一次转换均严格遵循关系代数等价规则(Relational Algebra Equivalence Rules)。通过确保所有重写操作均保持语义等价性(Semantic Equivalence)（例如利用 `A JOIN B ≡ B JOIN A` 的交换律(Commutativity)），优化器得以安全地排列、探索并计算替代执行路径的代价，从根本上杜绝了查询结果出错的风险。