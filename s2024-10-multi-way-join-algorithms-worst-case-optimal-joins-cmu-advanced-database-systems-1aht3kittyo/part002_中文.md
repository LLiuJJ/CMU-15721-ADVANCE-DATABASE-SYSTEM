## 继续跳跃连接迭代
![关键帧](keyframes/part002_frame_00000000.jpg)
跳跃机制通过同步多张表的迭代器(Iterator)位置得以持续推进。若某迭代器指向的值严格小于其他迭代器指向的最大值，系统即可判定该位置无法产生匹配。此时，迭代器无需执行低效的线性扫描(Linear Scan)，而是直接“跳跃”(Skip)至大于或等于该最大值的下一个有效值。该过程循环往复，直至所有迭代器精确对齐至同一数值，从而确认匹配项。尽管其概念直观清晰，但要高效执行此类跳跃操作，仍需依赖专门设计的底层数据结构(Underlying Data Structure)。

## 针对数据库的 Trie 树适配
![关键帧](keyframes/part002_frame_00066250.jpg)
为支持快速跳跃与多维匹配(Multi-Dimensional Matching)，该算法核心依赖于 **Trie树(Trie Tree)**（发音为 "try"），该术语最初由卡内基梅隆大学教授 Edward Fredkin 提出。有别于传统字符串Trie树将单词逐层拆分为单一字符或数字，数据库系统对此结构进行了针对性改造：每个节点直接存储完整的属性值(Attribute Value)。Trie树的每一层级严格对应连接键(Join Key)中的特定属性。系统会剔除给定属性的重复值，并通过指针(Pointer)进行分支，以映射后续属性的唯一子值(Unique Child Values)。
![关键帧](keyframes/part002_frame_00076650.jpg)

## Trie 树构建与属性映射
![关键帧](keyframes/part002_frame_00123916.jpg)
为便于可视化(Visualization)，Trie树通常采用水平布局进行展示。从根节点(Root Node)出发，第一层节点由初始属性的唯一值（如属性 A）填充。随后，从每个 A 节点延伸出的边(Edge)将用于存储第二属性（如属性 B）的对应唯一值。此种层次化模式(Hierarchical Pattern)将逐级延伸，直至连接键中的所有属性均被映射完毕。
![关键帧](keyframes/part002_frame_00137866.jpg)
核心在于，共享同一父节点的所有值在叶子层级(Leaf Level)必须保持严格的排序顺序。该有序特性对于在连接执行阶段实现高效的顺序求交(Sequential Intersection)至关重要。

## 执行多路连接示例
![关键帧](keyframes/part002_frame_00189366.jpg)
现以对三张表（R、S、T）执行连接为例，连接条件为 `R.a = T.a`、`R.b = S.b` 及 `S.c = T.c`。查询优化器(Query Optimizer)首先确立全局属性求值顺序(Global Attribute Evaluation Order)（A → B → C）。执行流程自 R 表的 Trie 树根节点起步，定位首个 A 值（例如 0）。该值随即用于探测(Probe) T 表的 Trie 树；若存在匹配的 `T.a = 0`，算法便继续推进。随后，流程下探至 B 层级。R 表的 B 值（例如 0）将探测 S 表的 Trie 树以寻找匹配的 `S.b`。一旦在目标表间确认了 A 与 B 的匹配关系，系统即抵达最终层级（C）。此时，迭代器将遍历 S 与 T 表中的 C 值并计算其交集(Intersection)，进而结合已知的遍历路径（A=0, B=0 及匹配的 C）重构结果元组(Result Tuple)。该过程将系统性地遍历特定 A 值下的所有 B 值，随后回溯(Backtrack)以处理下一个 A 值，循环往复直至穷尽所有有效组合。

## 属性求值顺序的限制
在此架构下，一项严格的约束在于：Trie树的层级结构必须与查询优化器所确定的全局求值顺序(Global Evaluation Order)精确对齐。即便某张表缺失部分连接属性，其 Trie 树亦不可任意重排层级（例如，若优化器指定顺序为 A → B → C，则绝不可将 C 置于 B 之前）。在多路探测(Multi-Way Probing)与求交循环中，强制所有表专属的 Trie 树遵循统一的遍历顺序(Traversal Order)，是维持各迭代器同步的强制性要求。

## 性能权衡：预计算与动态构建
![关键帧](keyframes/part002_frame_00568516.jpg)
该策略的落地引入了显著的架构权衡(Architectural Trade-off)。以 EmptyHeaded 论文提出的方案为例，其主张在查询执行前预先计算并物化(Materialize)所有潜在连接顺序的 Trie 树。然而，正如 Umbra 论文所指出，当面对海量数据集(Large-Scale Dataset)（含数十亿元组）或需支持增量更新(Incremental Update)的系统时，此做法将变得极为低效且脱离实际。无论是从磁盘加载预物化的结构，还是为每次查询全盘重建，均会产生难以承受的性能开销。因此，现代数据库系统更倾向于采用动态构建策略，结合惰性求值(Lazy Evaluation)与惰性物化(Lazy Materialization)技术，仅在连接执行过程中按需动态生成数据结构中必需的部分。