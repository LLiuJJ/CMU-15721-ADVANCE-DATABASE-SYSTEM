## 哈希表扩容与构建-探测权衡
![关键帧](keyframes/part006_frame_00000000.jpg)
在开放寻址哈希表(Open Addressing Hash Table)中，标准的扩容策略(Resizing Strategy)通常是初始分配约为预期条目数(Entries)两倍的容量，并在哈希表趋于饱和时再次将容量翻倍。然而，传统的线性探测(Linear Probing)存在一个关键的性能瓶颈：随着哈希表逐渐填满，键值(Key)会被不断推离其理想哈希位置(Ideal Hash Position)，导致查找操作(Lookup Operation)中产生漫长且不可预测的顺序扫描(Sequential Scan)。数据库系统通过利用哈希连接(Hash Join)独特的两阶段结构——构建阶段(Build Phase)与探测阶段(Probe Phase)——来缓解这一问题。通过主动承担构建阶段额外的计算开销(Computational Overhead)，系统可提前重组哈希表结构，以最大限度地缩短探测距离(Probe Distance)。此项优化策略有意以更高的写入/构建成本(Write/Build Cost)为代价，换取显著更快且更具可预测性的读取/探测操作(Read/Probe Operation)。

## 罗宾汉哈希的概念与现实争议
![关键帧](keyframes/part006_frame_00139649.jpg)
![关键帧](keyframes/part006_frame_00165599.jpg)
![关键帧](keyframes/part006_frame_00182383.jpg)
罗宾汉哈希(Robin Hood Hashing)于 20 世纪 80 年代提出，通过“劫富济贫”的位移策略(Displacement Strategy)有效解决了线性探测的效率低下问题。在插入操作(Insertion)中，若新键的理想槽位已被占用，算法会对比现有键的当前探测距离(Probe Distance，即跳数(Hop Count))与待插入键的预期距离。若待插入键距离其理想位置更远，算法将执行位置交换，将现有键向后推移(Displace Backward)。尽管近年来的学术文献(Academic Literature)普遍认为其优于标准线性探测，但在实际工程实现中的表现却存在争议。例如，QuestDB 最初基于早期基准测试(Benchmark)的乐观结果采用了罗宾汉哈希，但最终发现其在生产工作负载(Production Workload)中性能不升反降，遂撤销了该变更。InfluxDB 也报告了类似经历，这凸显了理论优势(Theoretical Advantage)并不总能普遍转化为不同数据分布与硬件配置下的实际性能提升(Actual Performance Gain)。

## 插入机制与探测保证
![关键帧](keyframes/part006_frame_00215900.jpg)
该核心机制要求在每个键值旁额外存储一个“跳数”(Hop Count)，用于记录其偏离原始哈希位置(Original Hash Position)的距离。执行插入时，算法需动态评估是继续顺序插入还是触发键值交换。例如，若键 `E` 距离其理想位置 2 步，并遭遇了键 `D`（距离理想位置仅 1 步），`E` 将抢占(Displace)该槽位，而 `D` 被向后推移，随后系统递归地重新评估(Recursive Re-evaluation)后续位置，直至为 `D` 找到合适槽位。该过程高度依赖条件分支判断(Conditional Branching)与内存复制操作(Memory Copy)，导致插入操作的计算开销显著增加。然而，查找过程(Lookup Process)保持严格的确定性：探测将持续进行，直至命中目标键或遭遇空槽(Empty Slot)，从而保证零假阴性(False Negative)（即绝不会漏报已存在的键值）。得益于其位移逻辑(Displacement Logic)，算法可确保绝不会存在有效键逻辑上位于空槽之后的情况，因此一旦遇到空槽即可安全终止查找。

## 方差降低与工作负载敏感性
在罗宾汉哈希表中，所有条目的跳数(Hop Count)总和在数学层面与标准线性探测等效；该算法并未减少数据位移的绝对次数。相反，它大幅降低了这些距离的方差(Variance)。通过平滑键值的分布，它有效规避了最坏情况(Worst-Case Scenario)，即少数键被推离极远，而多数键仍滞留于理想位置。这使得探测时间(Probe Time)高度可预测。对于存在数据倾斜的工作负载，其表现通常更优，因为高频访问的键会被优先拉近至理想槽位。尽管可叠加布隆过滤器(Bloom Filter)等辅助结构以快速剔除(Fast Reject)不存在的键，但罗宾汉哈希的核心性能增益仍高度依赖于具体的访问模式(Access Pattern)、查询倾斜度(Query Skewness)，以及系统对构建开销(Build Overhead)与探测延迟(Probe Latency)之间权衡的容忍度。

## 跳房子哈希：有界邻域
![关键帧](keyframes/part006_frame_00531983.jpg)
![关键帧](keyframes/part006_frame_00586566.jpg)
![关键帧](keyframes/part006_frame_00596383.jpg)
跳房子哈希(Hopscotch Hashing)通过引入严格的边界约束(Boundary Constraint)扩展了罗宾汉哈希的理念：键值仅允许在其理想哈希位置周围预定义的“邻域”(Neighborhood)内进行位移。该邻域的大小通常经过精心设计，以严格对齐 CPU 缓存行(CPU Cache Line)的容量（例如覆盖连续的 3-4 个槽位）。通过限制位移范围，算法可确保任意有效键或空槽(Empty Slot)均驻留于同一缓存行内，从而保证无论键在邻域内的具体位置如何，内存访问延迟(Memory Access Latency)均保持一致。若在尝试受限交换(Restricted Swap)后，目标邻域内仍无可用空间，哈希表将立即触发扩容操作。该方法有效限制了最大探测长度(Maximum Probe Length)，彻底消除了异常冗长扫描的风险，且相较于无界线性探测(Unbounded Linear Probing)或罗宾汉探测，能更高效地利用现代 CPU 缓存层次结构(CPU Cache Hierarchy)。