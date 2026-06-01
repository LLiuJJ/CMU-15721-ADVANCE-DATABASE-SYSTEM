## 优化器生成器(Optimizer Generator)与搜索策略
![关键帧](keyframes/frame_00000000.jpg)
现代查询优化器(Query Optimizer)通常借助优化器生成器构建。该架构允许开发者使用高级领域特定语言(Domain-Specific Language, DSL)定义模式匹配(Pattern Matching)条件与转换规则(Transformation Rules)，从而避免硬编码(Hard-coding)复杂的过程式逻辑(Procedural Logic)。这种关注点分离(Architectural Separation)显著提升了优化器的可维护性，并支持不同团队独立开发搜索引擎(Search Engine)与规则集(Rule Sets)。规则的执行通常遵循两种主要范式(Paradigms)之一：分层搜索(Layered Search)或统一搜索(Unified Search)。在分层搜索中，优化器首先独立于代价模型(Cost Model)，应用一组固定的启发式规则(Heuristic Rules)（如谓词下推(Predicate Pushdown)）进行逻辑重写，随后才进入基于代价的搜索(Cost-Based Search)阶段。相反，统一搜索将逻辑转换(Logical Transformations)、物理实现选择(Physical Implementation Selection)与代价评估(Cost Estimation)融合至单一的、综合的探索过程(Exploration Process)中。尽管如 Cascades 等框架在理论上支持纯粹的统一搜索，但许多生产级数据库(Production Databases)（如 Microsoft SQL Server）在实际中采用混合策略(Hybrid Strategy)：优先应用严格的启发式规则进行初步剪枝，随后再过渡至统一的基于代价的搜索空间探索。

## 分层搜索(Layered Search)方法与 Starburst 的遗产
分层搜索架构由 IBM 的 Starburst 项目首创，其设计理念至今仍在深刻影响着 Apache Calcite 等现代优化框架(Optimization Frameworks)。该过程在严格划分的阶段(Phases)中执行：优化器首先对逻辑计划(Logical Plan)进行重写(Rewriting)，应用那些具备普适收益或理论必要性的转换规则，此过程完全独立于代价模型。待逻辑转换收敛后，系统进入第二阶段，采用类似于经典 System R 优化器的自底向上动态规划(Bottom-Up Dynamic Programming)算法，对物理实现(Physical Implementations)进行探索。这种两阶段架构(Two-Phase Architecture)使数据库系统能够强制执行必要的逻辑优化，并在早期阶段大幅剪枝(Prune)搜索空间，从而避免基于代价的搜索浪费资源去评估本质上低效的逻辑结构。

## IBM 系统中的关系演算(Relational Calculus)与逻辑重写
![关键帧](keyframes/frame_00292666.jpg)
原始 Starburst 实现的一个显著特征在于，其初始重写阶段高度依赖关系演算。优化器并非直接操作标准的关系代数操作符(Relational Algebra Operators)，而是先将查询计划转换为更为抽象的关系演算数学表达形式。此举为那些在传统关系代数中难以精确表述的复杂逻辑转换提供了更强的表达能力(Expressive Power)。应用启发式规则后，系统会将优化后的演算表达式还原为逻辑计划（通常称为查询图模型(Query Graph Model, QGM)），随后才执行基于代价的优化。尽管该方案在理论上极为优雅，但在实际工程实现与调试(Debugging)方面却极具挑战。除 IBM 的早期研究与部分 DB2 特定版本外，现代数据库系统已基本弃用关系演算，转而直接采用关系代数操作。这主要是因为关系演算抽象层级过高，与底层可执行代码之间存在难以逾越的抽象鸿沟(Abstract Gap)。

## DB2 的复杂历史与优化器的演进
![关键帧](keyframes/frame_00409983.jpg)
查询优化器架构的演进历程与企业级数据库工程的历史紧密交织。例如，DB2 for Linux, Unix, and Windows (DB2 LUW) 实际上起源于 20 世纪 80 年代末至 90 年代初的 OS/2 Database Manager。随着 Microsoft Windows 逐渐占据操作系统市场主导地位，IBM OS/2 的发展举步维艰。为保持竞争力，数据库团队决定引入 IBM 研究院先进的 Starburst 查询优化器，以推动其系统架构的现代化(Modernization)。此次集成标志着分层搜索模型正式迈入生产级环境(Production Environments)。然而，在多个完全独立的 DB2 代码库（如 z/OS、LUW 等）之间同步维护此类复杂的优化器框架，历来是一项艰巨的工程挑战。工程师们常常需要应对与关系演算绑定的自定义 DSL 所带来的僵化性(Rigidity)与陡峭的学习曲线(Steep Learning Curve)。这一现象深刻揭示了理论上的优化纯粹性与可持续软件工程(Sustainable Software Engineering)之间必须做出的现实权衡(Trade-off)。

## 统一搜索(Unified Search)、Cascades 与备忘录技术(Memoization)
![关键帧](keyframes/frame_00509799.jpg)
为突破严格分阶段方法的局限性，Cascades 框架率先引入了统一搜索策略。在该模型下，逻辑转换、物理操作符选择(Physical Operator Selection)与代价评估被统一纳入单一、集成的搜索空间(Search Space)中协同进行。该策略面临的核心挑战在于潜在查询计划数量的组合爆炸(Combinatorial Explosion)，极易导致计算资源耗尽而变得不可行(Computationally Intractable)。为应对这一挑战，Cascades 深度依赖备忘录技术。优化器通过维护一张结构化的内存表(Memory Table)，记录已探索的子计划(Subplans)、其逻辑等价变体(Logically Equivalent Variants)及相关代价，从而有效避免了重复计算(Redundant Computations)。当搜索引擎遭遇已评估过的逻辑等价子计划时，将直接检索缓存结果，而非重新生成计划或重复计算代价。该机制使得统一搜索能够在彻底探索复杂转换空间(Transformation Space)的同时，将优化耗时(Optimization Time)严格控制在实际可行的阈值内。