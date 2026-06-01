## SIMD 约束与传统格式的限制
现代 SIMD(Single Instruction, Multiple Data) 架构要求硬件寄存器通道(Register Lane)内的所有数据元素必须具有固定且统一的大小。因此，变长编码(Variable-Length Encoding)在加载至寄存器前必须进行定长化处理，这会带来显著的计算开销。此外，Parquet 和 ORC(Optimized Row Columnar) 等传统列式格式(Columnar Format)通常会在读取时直接解压数据，从而剥离了宝贵的底层结构信息。这种做法隐藏了底层的字典编码(Dictionary Encoding)，并使经过块压缩(Block Compression)的数据（例如通过 Snappy 或 Zstandard 压缩的数据）对数据库引擎而言成为完全不透明的黑盒(Opaque Black Box)。其结果是，系统无法直接在编码流上执行查找或原生操作，导致在查询执行前必须进行完整的数据物化(Data Materialization)。

![关键帧](keyframes/part002_frame_00000000.jpg)

## 数值间依赖与 SIMD 并行性
某些轻量级编码方案（如 Delta 编码(Delta Encoding) 和游程编码(Run-Length Encoding, RLE)）会在相邻数据值之间引入顺序依赖(Sequential Dependency)。在 SIMD 寄存器中，硬件操作旨在同时处理多个相互独立的数据元素。当当前值的计算依赖于前序值（例如，需通过将增量与前一个值累加以重建当前值）时，中央处理器(CPU) 便无法高效地执行并行算术运算。相反，系统必须执行代价高昂的寄存器重排(Register Shuffle)和顺序移位操作，从而完全抵消了向量化执行(Vectorized Execution)的性能优势。这种根本性的架构不兼容性，使得传统的编码策略已不再适配现代的硬件加速(Hardware Acceleration)环境。

![关键帧](keyframes/part002_frame_00214816.jpg)

## 硬件多样性与可移植性挑战
ARM、x86（含 AVX-512(Advanced Vector Extensions 512-bit)）、图形处理器(GPU) 和张量处理器(TPU) 等多样化硬件架构的迅速普及，使得底层向量化代码的实现变得日益复杂。每种指令集架构(Instruction Set Architecture, ISA) 都提供不同的 SIMD 能力与底层内建函数(Intrinsics)。尽管存在抽象库可根据底层 CPU 特性动态分派(Dynamic Dispatch)操作，但主流数据库系统为追求极致性能，通常避免使用此类抽象层，转而直接采用手动调优的内建函数。然而，这种硬编码方法牺牲了软件可移植性(Portability)，并迫使执行引擎去适配那些本质上会阻碍编译器自动向量化(Compiler Auto-Vectorization)的僵化文件格式设计。

## 下一代文件格式：FastLanes 与 Better Blocks
为突破上述传统瓶颈，近期学术界引入了全新的存储范式(Storage Paradigm)。*Better Blocks*（源自慕尼黑工业大学）可视为“Parquet++”的演进版本，它利用贪心算法(Greedy Algorithm)对列块(Column Chunk)进行采样，并自动甄选最优的轻量级编码方案(Lightweight Encoding Scheme)。该系统会递归地将最匹配的编码器应用于派生列(Derived Column)，在不依赖不透明块压缩(Opaque Block Compression)的前提下实现高压缩率(Compression Ratio)。*FastLanes* 则采用了截然不同的策略，其刻意以特定的非连续布局(Discontinuous Layout)存储数据，旨在最大限度减少条件分支(Conditional Branch)并最大化运行时解码速度。这两种设计均致力于保持数据结构对中央处理器(CPU)友好且透明，使查询引擎能够在维持高吞吐量(High Throughput)的同时，规避完整数据解压(Data Decompression)的步骤。

![关键帧](keyframes/part002_frame_00268733.jpg)

## 外部元数据与设计权衡
*Better Blocks* 一项显著的架构决策是将数据模式(Data Schema)与文件元数据(File Metadata)存储于外部元数据管理服务中，而非嵌入数据文件内部。支持者认为此举实现了底层存储与格式定义的解耦(Decoupling)，但批评者指出，这在文件的自包含性(Self-Containment)与可移植性(Portability)方面做出了重大妥协。将元数据内嵌至文件可确保其在离线或孤立状态下仍能被完整解析，而外部存储方案则引入了额外的依赖风险并增加了数据共享的复杂度。在现代云基础设施中，高度冗余的对象存储(Object Storage)已大幅降低数据永久损坏的风险，因此这种架构分离更多是基于特定设计理念的取舍，而非绝对的技术刚需。最终，这些新一代文件格式优先考量执行效率(Execution Efficiency)与硬件对齐(Hardware Alignment)，在抽象底层复杂性的同时，为应用开发者带来了显著的性能跃升。

![关键帧](keyframes/part002_frame_00364466.jpg)