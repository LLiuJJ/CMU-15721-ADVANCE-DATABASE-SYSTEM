## 关系代数(Relational Algebra)基础与组（Group）的定义
![关键帧](keyframes/frame_00000000.jpg)
![关键帧](keyframes/frame_00007033.jpg)
在 Cascades 等优化框架中，查询优化严格遵循关系代数(Relational Algebra)的等价变换规则(Equivalence Rules)，利用交换律(Commutativity)等数学属性安全地重排执行计划，同时确保查询语义不变。该架构的核心在于**组（Group）**这一概念。一个组封装了所有逻辑等价(Logically Equivalent)且产生相同预期子查询结果(Subquery Results)的逻辑表达式(Logical Expressions)与物理表达式(Physical Expressions)。例如，表示三表连接(A JOIN B JOIN C)的组，将囊括所有可行的逻辑连接顺序(Join Orders)及其衍生的所有对应物理实现(Physical Implementations)。此外，每个组还会追踪其成员必须满足或强加的特定物理属性(Physical Properties)（如排序顺序(Sort Order)）。

## 作为惰性占位符(Lazy Placeholders)的多表达式（Multi-Expressions）
![关键帧](keyframes/frame_00082700.jpg)
为应对自顶向下搜索(Top-Down Search)中潜在的**组合爆炸(Combinatorial Explosion)**问题，Cascades 引入了**多表达式（Multi-Expressions）**作为计划树中的惰性占位符。多表达式仅声明当前层级需执行的操作，而将具体连接顺序(Join Order)、访问路径(Access Paths)及物理算法(Physical Algorithms)的确定明确推迟。优化器避免了急切展开(Eagerly Expanding)所有潜在子计划(Subplans)所导致的内存膨胀，转而通过这些占位符抽象表示“下层待解析的结构”。此举使搜索引擎(Search Engine)能够在计划树高层级先行做出结构决策，无需即刻下沉至基础表(Base Tables)。仅当优化器判定必要时，才会依据预设的优先级策略对这些占位符进行增量式展开(Incremental Expansion)。

## 转换规则(Transformation Rules)与实现规则(Implementation Rules)
![关键帧](keyframes/frame_00204483.jpg)
![关键帧](keyframes/frame_00346099.jpg)
优化器依赖两类模式匹配规则(Pattern Matching Rules)对计划树进行展开与重构：**转换规则（Transformation Rules）**负责将逻辑表达式(Logical Expressions)重写为等价的其他逻辑形式（例如，将左深连接树(Left-Deep Join Tree)转换为右深连接树(Right-Deep Join Tree)）；**实现规则（Implementation Rules）**则负责将逻辑表达式映射为具体的物理操作符(Physical Operators)（例如，将逻辑等值连接(Logical Equi-Join)实现为哈希连接(Hash Join)或归并连接(Sort-Merge Join)）。这些规则均由目标模式(Target Pattern)与替换模板(Replacement Template)精确定义。在逻辑到逻辑(Logical-to-Logical)的转换中，一个核心挑战是防范无限循环(Infinite Loop)，即规则可能在两种等价形态间反复切换。为此，优化器必须严格追踪已应用的转换历史以阻断循环并抑制内存膨胀。特别是针对那些绝对收益的优化规则，系统通常直接覆盖(Overwrite)旧状态，而非保留冗余的历史快照。

## 备忘录表(Memo Table)与代价缓存机制
![关键帧](keyframes/frame_00368249.jpg)
**备忘录表（Memo Table）**是化解无限循环风险与消除自顶向下搜索冗余的核心缓存机制。针对搜索过程中遇到的每个多表达式(Multi-Expression)，备忘录表会持久化记录当前发现的**最低代价物理表达式(Lowest-Cost Physical Expression)**及其对应的代价估算(Cost Estimation)。在实际架构中，该信息通常不作为独立数据结构游离存在，而是直接内嵌于组(Group)的定义中。依托此缓存机制，优化器彻底规避了对相同子树(Subtree)的重复遍历。当某条规则生成的新计划匹配到已评估过的多表达式时，优化器仅需直接从备忘录表中检索已知的最优代价，即可跳过繁琐的递归展开(Recursive Expansion)与二次代价计算。

## 增量代价评估(Incremental Cost Evaluation)与搜索工作流(Search Workflow)
![关键帧](keyframes/frame_00436499.jpg)
![关键帧](keyframes/frame_00475233.jpg)
![关键帧](keyframes/frame_00580266.jpg)
搜索工作流(Search Workflow)以自顶向下的方式增量推进。在评估诸如 `A ⋈ B` 的多表达式(Multi-Expression)时，优化器首先对叶节点(Leaf Nodes) `A` 和 `B` 应用实现规则(Implementation Rules)，对比顺序扫描(Sequential Scan)与索引探测(Index Probe)等备选方案。各表成本最低的物理访问路径(Physical Access Paths)将被即时写入备忘录表（例如，`Cost(A)=10`, `Cost(B)=20`）。随后，当系统探索替代的逻辑连接顺序(Logical Join Order)（如 `B ⋈ A`）时，优化器会直接查询备忘录表以复用已缓存的叶节点代价，从而大幅削减冗余遍历。在穷尽所有逻辑排列(Logical Permutations)后，实现规则将实例化具体的物理连接算法(Physical Join Algorithms)。物理连接的总代价等于连接操作符(Join Operator)自身开销与其子节点已缓存代价之和（例如，`Cost(HashJoin(A,B)) + 10 + 20 = 80`）。最终，该最优物理计划及其聚合代价将被回写至备忘录表。至此，Cascades 框架“自顶向下探索结构、自底向上累加代价”的核心优化循环得以闭环。