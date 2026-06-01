## BetterBlocks 编码策略与采样权衡
课程首先阐明了 BetterBlocks 的贪心编码算法(Greedy Encoding Algorithm)的运作机制。尽管嵌套编码(Nested Encoding)的默认递归深度(Recursion Depth)设为三层，但系统完全依赖每个包含 64,000 个数据值的列块(Column Chunk)中抽取的 1% 代表性采样数据(Sample Data)来选择最优编码方案。学员提出疑问：若完整数据集(Data Set)的特征与采样数据不一致，算法是否会执行回溯(Backtracking)？讲师明确答复：不会。对整个数据集重新评估或回滚编码方案将带来难以承受的计算开销(Computational Overhead)。相反，该设计有意在数据加载(Data Ingestion)阶段引入约 2% 的可控前期开销，以此作为架构权衡(Architectural Trade-off)，从而确保运行时(Runtime)的查询执行(Query Execution)保持高度优化与高效。

![关键帧](keyframes/part004_frame_00000000.jpg)

## FSST：快速静态字符串文本压缩
在介绍完整数编码格式(Integer Encoding Format)后，讨论转向 FSST(Fast Static String Table)。这是一项 2020 年提出的创新技术，专为高效字符串压缩(String Compression)与快速随机访问(Random Access)而设计。与传统字典编码(Dictionary Encoding)将整个字符串映射为不透明标识符(Opaque Identifier)不同，FSST 能够识别频繁出现的子字符串(Substring)（最长 8 字节），并将其替换为单字节代码(Single-Byte Code)。该方法保留了直接在编码数据上执行局部查找(Partial Lookup)、前缀匹配(Prefix Matching)及其他查询优化(Query Optimization)操作的能力。尽管底层子字符串长度可变，但编码输出为每个代码维持了定长结构(Fixed-Length Structure)，因此需引入特定的终止标记(Terminator)来标识字符串结束，以防止尾部字节被错误解析。

![关键帧](keyframes/part004_frame_00081583.jpg)

## 针对 SIMD 优化的符号表生成
构建最优的 FSST 符号表(Symbol Table)属于 NP 完全问题(NP-Complete Problem)，因此作者采用了一种务实的启发式算法(Heuristic Algorithm)，而非穷举搜索(Exhaustive Search)或动态规划(Dynamic Programming)。系统在扫描数据时，采用直接覆盖策略(Direct Overwrite Strategy)将候选子字符串插入哈希表(Hash Table)：若目标哈希槽位(Hash Slot)已被占用，现有条目(Entry)将被立即驱逐(Evict)。这种看似激进的做法实为精心设计：真正能提升压缩率(Compression Ratio)的高价值符号自然会反复出现并被重新插入，从而使符号表随时间推移收敛(Converge)至一个高效的符号集合。其核心优势在于，消除线性探测(Linear Probing)与条件分支(Conditional Branch)后，整个符号表的生成与查找过程得以完全兼容 SIMD 单指令多数据(SIMD Vectorization)架构，有效规避了传统哈希表实现的性能陷阱(Performance Pitfall)。

![关键帧](keyframes/part004_frame_00104883.jpg)

## Roaring Bitmaps：自适应位图索引
本节最后探讨了 Roaring Bitmaps(Roaring 位图)，这是一种高度优化的数据结构(Data Structure)，用于管理位图索引(Bitmap Index)，并能动态适配数据密度(Data Density)。其将 64 位整数空间(64-bit Integer Space)划分为多个包含 $2^{16}$ 个值的容器(Container)。针对每个容器，系统根据其稀疏性(Sparsity)动态选择最高效的存储格式：若置位(Set Bit)数量少于 4,096 个，则采用紧凑的 16 位整数排序数组(Sorted Array of 16-bit Integers)；若密度超过该阈值，则切换为标准的位图阵列(Bitset Array)。查找操作通过计算目标键(Target Key)除以 $2^{16}$ 的商来定位对应容器，随后执行相应的数组搜索或位掩码操作(Bitwise Mask Operation)。这种混合存储策略(Hybrid Storage Strategy)使 Polaris 和 Futurebase 等系统在维持极低内存占用(Memory Footprint)的同时，能够高效支持集合成员资格检查(Set Membership Check)、空值追踪(Null Tracking)及位图遍历操作(Bitmap Traversal)。

![关键帧](keyframes/part004_frame_00495599.jpg)