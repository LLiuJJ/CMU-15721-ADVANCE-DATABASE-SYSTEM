## 字典编码(Dictionary Encoding)与二级压缩(Secondary Compression)
![关键帧](keyframes/part006_frame_00000000.jpg)
![关键帧](keyframes/part006_frame_00019883.jpg)
字典编码将重复或变长数据映射为紧凑的定长代码(Fixed-Length Codes)，从而实现可预测的内存访问(Predictable Memory Access)与高效的向量化扫描(Vectorized Scan)。Apache Arrow 等系统通过预排序字典条目(Pre-sorted Dictionary Entries)优化该过程，允许引擎直接通过字节偏移量(Byte Offset)进行定位，无需在运行时动态维护哈希表(Hash Table)。一旦原始值被转换为字典代码，生成的列数据流(Column Data Stream)通常会产生长序列的连续重复整数。针对此类序列，系统可应用游程编码(Run-Length Encoding, RLE)、差分编码(Delta Encoding)或基准值编码(Frame-of-Reference, FOR)等二级编码技术(Secondary Encoding Techniques)实现深度压缩。此外，若按字母序或词频对字典条目进行重排，相同代码将在物理上高度聚集，从而在触发后续重量级块级压缩(Heavyweight Block Compression)前，最大化轻量级编码方案的压缩增益。

## 处理高基数(High Cardinality)与压缩范围(Compression Scope)
![关键帧](keyframes/part006_frame_00084700.jpg)
![关键帧](keyframes/part006_frame_00109450.jpg)
当目标列呈现极高基数或包含极长唯一字符串时，盲目应用字典编码可能适得其反，甚至导致存储开销(Storage Overhead)不降反升。在此类场景下，现代文件格式会触发回退机制(Fallback Mechanism)：原始长字符串被剥离并存放于独立的二进制大对象(BLOB, Binary Large Object)存储区，而主列数据流仅保留指向目标字符串的定长字节偏移量(Fixed-Length Byte Offsets)。除回退策略外，各格式对字典编码的适用类型界定亦存在分歧。ORC 传统上将其严格限定于字符串类型，认为数值列的数据分布过于随机(Random Distribution)而难以获益。相反，Parquet 采取普适策略，对包括整型与浮点型(Floating-Point Types)在内的所有数据类型均默认启用字典编码。实证基准测试(Empirical Benchmarks)表明，Parquet 的激进策略能达成更优的整体压缩率，因为有效降低全量列的数据基数，能为后续编码层创造显著更高的压缩密度。

## 块压缩(Block Compression)与现代硬件权衡(Modern Hardware Trade-offs)
![关键帧](keyframes/part006_frame_00279583.jpg)
完成基础编码后，列式格式通常会对行组(Row Group)应用 Snappy 或 Zstandard 等通用块级压缩算法(General-Purpose Block Compression Algorithms)。在传统架构中，该策略的权衡逻辑明确偏向于“以 CPU 周期换取 I/O 带宽”，旨在缓解缓慢磁盘与网络链路的压力。然而，在现代云原生环境与 NVMe 等高性能存储架构中，CPU 算力已取代 I/O 成为核心瓶颈。Zstandard 等不透明压缩方案(Opaque Compression Schemes)要求查询引擎在执行任何过滤或谓词评估前，必须全量解压(Decompress Entire Block)目标数据块，这直接扼杀了向量化执行(Vectorized Execution)与谓词下推(Predicate Pushdown)的性能优势。尽管此类高压缩比算法在冷数据归档(Cold Data Archiving)中仍具价值，但现代分析型引擎正日益转向 CPU 友好型轻量级编码(CPU-Friendly Lightweight Encodings)，旨在支持直接在压缩比特流(Compressed Bitstream)上执行算子，彻底免除强制性的全块解压开销。

## 字典暴露(Dictionary Exposure)与谓词下推(Predicate Pushdown)
![关键帧](keyframes/part006_frame_00311666.jpg)
传统 Parquet 与 ORC 库的核心局限在于，字典结构通常被严密封装于内部迭代器(Internal Iterator)底层。当查询引擎请求列数据时，获取的已是全量解压后的原始值，这从根本上阻断了将过滤谓词(Filter Predicate)下推至存储层的可能。为突破此瓶颈，Google 的 Prasella（基于 Artists 格式）等新兴架构选择将字典显式暴露(Explicitly Expose)给查询规划器(Query Planner)。该设计允许引擎将语义谓词（如 `WHERE name = 'Andy'`）动态编译为对应的字典代码(Dictionary Codes)，直接在编码列数据流(Encoded Column Stream)上执行过滤运算，并跳过所有非匹配数据块的解码流程。此种“压缩态评估”架构在海量数据扫描期间，实现了内存带宽消耗(Memory Bandwidth Consumption)与 CPU 开销的断崖式下降。

## 元数据过滤器(Metadata Filters)：区域映射(Zone Maps)与布隆过滤器(Bloom Filters)
![关键帧](keyframes/part006_frame_00368299.jpg)
为在不读取原始数据载荷(Raw Data Payload)的前提下加速查询规划(Query Planning)，现代格式会在行组层级嵌入轻量级元数据过滤器。区域映射(Zone Maps)为每个列块(Column Chunk)维护确定性的最小/最大值统计(Min/Max Statistics)；若查询谓词的目标范围与该统计区间无交集，引擎即可安全跳过(Skip)整个数据块。然而，区域映射仅能界定数据的潜在存在范围，对于通过过滤的块仍需执行全量顺序扫描。针对等值匹配或点查询(Point Queries)，布隆过滤器(Bloom Filters)提供了高效的概率型替代方案。该结构严格保证零假阴性(Zero False Negatives)，仅容忍极低的假阳性率(False Positive Rate)，使引擎能够迅速剔除绝对不含目标值的冗余数据块。值得注意的是，受限于哈希算法特性，布隆过滤器对数据插入顺序并不敏感；而区域映射则能从物理数据聚类(Physical Data Clustering)中显著获益。通过对底层数据进行预排序或分区(Partitioning)，可有效收窄极值统计范围，将原本宽泛低效的元数据转化为高选择性的数据跳过机制(Data Skipping Mechanism)，从而实现 I/O 开销的指数级优化。