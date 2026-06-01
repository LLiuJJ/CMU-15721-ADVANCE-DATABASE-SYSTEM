## 缓存优化的布隆过滤器(Cache-Optimized Bloom Filter)
![关键帧](keyframes/part007_frame_00000000.jpg)
现代分析型文件格式针对硬件效率，对布隆过滤器(Bloom Filter)等概率数据结构(Probabilistic Data Structures)进行了深度优化。分块布隆过滤器(Blocked Bloom Filter)通过限制哈希映射范围，使其仅针对能够恰好装入单个 CPU 缓存行(CPU Cache Line)的位组进行操作。通过确保整个过滤器结构常驻于 L1 缓存中，系统显著降低了昂贵的缓存未命中(Cache Miss)开销，并在谓词过滤(Predicate Filtering)期间实现超高速的成员资格检测(Membership Testing)。

## 嵌套数据结构的记录拆解(Record Shredding for Nested Data)
![关键帧](keyframes/part007_frame_00023466.jpg)
![关键帧](keyframes/part007_frame_00078633.jpg)
高效处理半结构化数据(Semi-Structured Data)（如 Protobuf 或 JSON）要求突破将完整文档存储为不透明二进制块(Opaque Binary Blobs)的传统范式。由 Google Dremel 系统首创的**记录拆解(Record Shredding)**技术，能够将层次化文档(Hierarchical Documents)扁平化(Flatten)为离散的列式字段。该技术摒弃了显式的父子指针(Parent-Child Pointers)，转而依赖辅助的 `repetition`（重复级别(Repetition Level)）与 `definition`（定义级别(Definition Level)）列。定义级别用于标识当前值在模式层次结构(Schema Hierarchy)中的嵌套深度，而重复级别则用于标记新记录的起始或重复字段的复位。此类元数据使查询引擎得以直接扫描各嵌套字段，并在运行时动态重建文档层级关系(Document Hierarchy)，彻底免除了对完整半结构化数据负载(Semi-Structured Data Payload)的运行时解析开销。

## 存在长度编码(Length-of-Presence Encoding)
作为记录拆解(Record Shredding)的轻量化替代方案，存在长度编码(Length-of-Presence Encoding)通过为记录中缺失的可选属性(Optional Attributes)插入占位符或空值(Null Values)，以维持严格的列对齐(Column Alignment)。该机制确保每一列与文档偏移量(Document Offset)之间保持精确的一对一位置映射，使系统仅需借助简单的存在标志(Presence Flag)与字节偏移量即可高效重建元组。尽管处理稀疏字段(Sparse Fields)时可能略微增加存储开销，但该方法能与标准的字典编码(Dictionary Encoding)及块压缩(Block Compression)方案无缝集成，从而有效规避了由重复/定义级别机制引发的复杂元数据遍历(Metadata Traversal)开销。

## Parquet 与 ORC 的真实世界评估(Real-World Evaluation)
![关键帧](keyframes/part007_frame_00258249.jpg)
为突破合成基准测试(Synthetic Benchmarks)的局限性，研究人员采用源自 Hugging Face 与 GitHub 等平台的真实生产数据集(Real-World Datasets)，对 Parquet 与 ORC 进行了深度评估。对比分析面临的核心挑战在于各语言特定实现(Language-Specific Implementations)间的行为差异。为确保公平且高性能的横向对比，本研究统一采用 Apache Arrow 的 C++ 库(Apache Arrow C++ Library)作为底层读写引擎，从而规避了传统 Java 实现中常见的 SIMD 指令集支持不统一，或对可选规范特性（如页级区域映射(Page-Level Zone Map)）处理不一致等问题。该实验配置提供了真正意义上的公平对标(Fair Comparison)，精准量化了两种格式在现代硬件优化型查询执行引擎(Hardware-Optimized Query Execution Engine)下的实际性能表现。

## 压缩效率与查询性能的权衡(Compression Efficiency vs. Query Performance Trade-offs)
![关键帧](keyframes/part007_frame_00369533.jpg)
![关键帧](keyframes/part007_frame_00448233.jpg)
尽管 Parquet 与 ORC 在整体文件大小(Overall File Size)上差异微乎其微，但其底层压缩策略会因目标工作负载(Workload)的不同而呈现出截然不同的性能特征(Performance Characteristics)：
- **存储效率(Storage Efficiency)：** 在以浮点数(Floating-Point Numbers)为主导的机器学习(Machine Learning)与日志记录(Logging)工作负载中，Parquet 凭借对浮点数据激进应用字典编码(Dictionary Encoding)，实现了更优的压缩收益（此类数据在特定分布下往往具备意外的可压缩性）。而在富含字符串(String-Rich)的传统业务与地理空间(Geospatial)工作负载中，ORC 则表现更为出色，这得益于其采用了更为丰富的字典后编码(Post-Dictionary Encoding)策略。
- **执行速度与向量化(Execution Speed & Vectorization)：** Parquet 始终展现出更快的顺序扫描(Sequential Scan)性能。其对直接位打包(Direct Bit-Packing)的依赖，为实现高度优化的线性无分支(Branch-Free) SIMD 向量化(SIMD Vectorization)创造了条件。相比之下，ORC 倾向于激进采用游程编码(Run-Length Encoding, RLE)；该策略虽能在连续重复数据上频繁生效，但往往会打断向量化流水线，迫使执行引擎回退至标量 SISD(Single Instruction, Single Data)处理模式。
- **分支预测错误代价(Branch Misprediction Penalty)：** ORC 依赖启发式规则(Heuristic-Based Rules)进行动态编码选择，这会在运行时解码(Decoding)阶段引入大量条件分支。在现代超标量 CPU(Superscalar CPU)架构中，此类不可预测分支极易引发流水线冲刷(Pipeline Flush)与执行停顿(Stall)。Parquet 则凭借更简洁、确定性更强的编码路径(Encoding Path)，彻底规避了此类性能损耗。
- **设计理念(Design Philosophy)：** 评估结果深刻揭示，在现代硬件架构下，设计的简洁性(Architectural Simplicity)与可预测的内存访问模式(Predictable Memory Access Patterns)通常优于复杂的自适应压缩(Complex Adaptive Compression)算法。相较于追求微乎其微的存储空间节省，最小化分支预测错误(Branch Misprediction)并最大化连续、可向量化的数据流(Continuous Vectorizable Data Stream)具有更高的性能价值。这一核心原则也正持续驱动着现代查询编译(Query Compilation)与代码特化(Code Specialization)技术的演进。