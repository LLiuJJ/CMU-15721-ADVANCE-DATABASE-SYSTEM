## 跳房子哈希：邻域约束与插入
![关键帧](keyframes/part007_frame_00000000.jpg)
跳房子哈希(Hopscotch Hashing)通过定义固定大小（例如三个连续槽位）且相互重叠的“邻域”(Neighborhood)，为探测长度(Probe Length)引入了严格的边界约束。执行插入操作时，新键必须驻留于其目标邻域内。例如，键 `A` 经哈希映射至邻域 3，因哈希表当前为空，故占据该邻域的首个槽位。键 `B` 映射至邻域 1 并占据对应槽位。后续键 `C` 与 `D` 的插入虽遵循线性探测(Linear Probing)模式，但被严格限制在各自的目标邻域范围内。 
![关键帧](keyframes/part007_frame_00016699.jpg)
![关键帧](keyframes/part007_frame_00023433.jpg)
当键 `E` 尝试插入邻域 3 时，若发现该邻域已满，算法不会进行无限制的向前扫描，而是遍历现有条目，甄别出哪些条目在逻辑上合法属于邻域 3。 
![关键帧](keyframes/part007_frame_00036366.jpg)
算法识别到 `D` 的原始哈希位置实为邻域 4。鉴于邻域 4 内存在空闲槽位，`D` 被向后推移一个槽位，迁移至其另一有效位置。 
![关键帧](keyframes/part007_frame_00045300.jpg)
此项定向交换(Directional Swap)成功释放了邻域 3 中的一个槽位，使 `E` 得以顺利插入。该机制确保所有键与其理想哈希位置(Ideal Hash Position)之间的偏移量始终维持在可预测且严格对齐缓存行(Cache-Line Aligned)的范围内。

## 跳房子哈希的复杂度、交换机制与扩容触发条件
![关键帧](keyframes/part007_frame_00065316.jpg)
邻域交换逻辑(Neighborhood Swap Logic)相较于标准线性探测或罗宾汉哈希(Robin Hood Hashing)更为复杂，在构建阶段(Build Phase)会引入更高的计算开销(Computational Overhead)。然而，该成本是严格有界且可均摊的(Amortized Cost)。关键的性能权衡在于，现代 CPU 架构极度偏好简单指令流；在诸多实际基准测试(Benchmark)中，跳房子哈希复杂的条件分支(Conditional Branching)与交换操作反而可能导致性能逊于更简单的方案。
![关键帧](keyframes/part007_frame_00093200.jpg)
跳房子哈希的一项根本限制在于其严格的扩容触发条件(Resizing Trigger Condition)：若某邻域已满，且其中任意条目均无法合法迁移(Migrate)至相邻的空闲槽位，算法将强制终止当前插入并触发全表扩容(Resize Operation)，即便表中其他区域仍存在空闲空间。此特性使扩容过程对数据分布(Data Distribution)与物化策略(Materialization Strategy)高度敏感。采用延迟物化(Late Materialization)的系统尚可承受此开销，但存储完整元组（即早期物化 Early Materialization）的系统在扩容时将面临高昂的数据复制成本。为缓解该问题，QuestDB 等数据库系统采用元组外置存储策略，将完整元组存放于独立堆区(Heap)，哈希表中仅保留行偏移量(Row Offset)，从而将扩容降级为轻量级的指针重分配(Pointer Reallocation)操作。

## 布谷鸟哈希：多函数驱逐与有界查找
![关键帧](keyframes/part007_frame_00201716.jpg)
布谷鸟哈希(Cuckoo Hashing，注：学术上易与双重哈希 Double Hashing 区分)采用固定的确定性驱逐机制(Deterministic Eviction Mechanism)，取代了无限制的线性扫描。该算法摒弃单一哈希函数，转而使用多个哈希函数（通常为两个，且采用不同种子 Seed），为每个键精确生成两个候选槽位(Candidate Slot)。执行插入时，若任一候选槽位为空，键将直接写入。若两槽位均被占用，算法会随机“踢出”(Evict)一个现有键以腾出空间，随后利用备用哈希函数尝试重新插入(Re-insert)被驱逐的键。
![关键帧](keyframes/part007_frame_00214949.jpg)
此驱逐链(Eviction Chain)将持续执行，直至定位到空闲槽位。为防范无限循环(Infinite Loop)，算法会记录已访问路径，一旦检测到环路(Cycle)即中止操作并强制触发扩容。该架构带来显著的性能收益：查找操作(Lookup Operation)被严格限定为 O(1) 时间复杂度，最多仅需两次哈希运算与两次内存访问(Memory Access)。IBM DB2 等系统正是利用此特性，实现了高度可预测的探测端延迟(Probe Latency)。

## 工作负载权衡：针对读密集型访问的优化
尽管罗宾汉哈希、跳房子哈希与布谷鸟哈希显著提升了构建阶段(Build Phase)的算法复杂度与 CPU 周期消耗(CPU Cycle Consumption)，但其工程设计明确针对读取密集型工作负载(Read-Intensive Workload)进行了优化。在数据库系统中，哈希连接(Hash Join)通常被视为“一次写入，多次读取”(Write-Once, Read-Many)操作：哈希表仅在构建阶段实例化一次，随后在探测阶段(Probe Phase)需对大规模关系表(Relation)执行数百万次探测查询。以构建速度换取有界(Bounded)、缓存友好(Cache-Friendly)且分支可预测(Branch-Predictable)的查找性能，对于探测吞吐量(Probe Throughput)主导整体查询执行时间(Query Execution Time)的分析型(OLAP)与事务型(OLTP)数据库而言，是一项具备严密数学依据的架构权衡(Architectural Trade-off)。

## 哈希表存储布局与键表示
![关键帧](keyframes/part007_frame_00401133.jpg)
实现开放寻址哈希(Open Addressing Hashing)需审慎决策哈希表槽位(Slot)内实际存储的物理数据形态。对于变长数据(Variable-Length Data)，直接存储完整元组将引发兼容性问题，因开放寻址强制要求固定槽位尺寸以保障内存寻址的可预测性。业界通用的解决方案是存储指向独立元组堆(Tuple Heap)的 64 位偏移量(Offset)或指针。该设计虽最小化了槽位体积并简化了扩容逻辑，却在连接操作(Join Operation)期间引入了指针解引用(Pointer Dereferencing)的额外开销。此外，系统还需抉择存储原始连接键(Join Key)抑或预计算哈希值(Precomputed Hash Value)。存储哈希值可在冲突解决(Collision Resolution)时启用高效的整数比较，规避高昂的字符串或多列比对成本，但会额外增加单槽内存占用。数据库架构师(Database Architect)必须依据工作负载特征与硬件缓存容量(Hardware Cache Capacity)，在空间效率(Space Efficiency)与 CPU 消耗(CPU Consumption)之间持续寻求最优平衡。

## 探测阶段优化：横向信息传递
![关键帧](keyframes/part007_frame_00515016.jpg)
探测阶段(Probe Phase)本质上是顺序执行的，需遍历外表(Outer Relation)并频繁查询哈希表。然而，引入布隆过滤器(Bloom Filter)可大幅提升该阶段性能，此类优化在查询优化领域被称为“横向信息传递”(Sideways Information Passing, SIP)。在构建阶段，系统会并行构建一个紧凑的布隆过滤器。在发起哈希表探测前，查询引擎优先检索该过滤器。得益于布隆过滤器零假阴性(Zero False Negative)的数学特性，若判定目标键不存在，即可确证其不在数据集中，系统从而直接跳过(Skip)代价高昂的哈希表查找操作。该技术在 Hyper 与 Umbra 等现代数据库中得到广泛部署，部分实现还会叠加桶级布隆过滤器(Bucket-Level Bloom Filter)，以彻底规避不必要的链式遍历(Chained Traversal)。

## 过渡到性能基准测试
![关键帧](keyframes/part007_frame_00576383.jpg)
理论探讨随后转向实证性能分析(Empirical Performance Analysis)，在受控工作负载(Controlled Workload)下对各类分区与哈希策略进行横向对比。基准测试(Benchmark)不仅将无分区基线(Non-Partitioned Baseline)与基数分区变体(Radix Partitioning Variants)进行对照，还评估了专为 IBM DB2 研发、后被 ClickHouse 等系统采纳用于处理字符串密集型工作负载(String-Intensive Workload)的专用数据结构——如简洁哈希表(Concise Hash Table)。这些实证评估量化了算法复杂度(Algorithmic Complexity)、缓存局部性(Cache Locality)及硬件感知优化(Hardware-Aware Optimization)如何切实转化为现代并行数据库引擎(Modern Parallel Database Engine)中可度量的吞吐量(Throughput)提升。