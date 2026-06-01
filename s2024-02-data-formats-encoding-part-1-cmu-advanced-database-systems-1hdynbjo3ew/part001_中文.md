## 物化视图(Materialized View)与增量维护(Incremental Maintenance)
![关键帧](keyframes/part001_frame_00000000.jpg)
当向支持物化视图的基表(Base Table)插入新数据时，必须刷新(Refresh)视图以维持数据一致性。尽管部分数据库系统支持自动增量刷新，但如 PostgreSQL 等系统仍需手动触发。增量维护的核心挑战在于效率：当插入单行数据时，经过优化的系统无需从头重新计算整个聚合查询(Aggregation Query)，而是精准识别数据变更（例如递增运行总计(Running Total)），并相应地更新已持久化的结果。从功能上看，物化视图充当一种持久化的辅助数据结构，类似于系统重启后依然可存取的临时表(Temporary Table)或索引(Index)。

## 课程重点：持久化数据格式(Persistent Data Format)与编码(Data Encoding)
![关键帧](keyframes/part001_frame_00092666.jpg)
本讲座重点探讨数据编码如何直接支撑数据并行化(Data Parallelism)与向量化(Vectorization)等高级执行技术，相关内容将在后续查询处理(Query Processing)章节中深入展开。讨论将严格聚焦于持久化数据格式——即实际落盘至磁盘或对象存储(Object Storage)中的底层字节与文件结构，而非内存中的临时中间格式。透彻理解这些文件的物理布局(Physical Layout)是设计存储模型(Storage Model)的基石，其核心目标是为分析型工作负载(Analytical Workload)最大化 I/O 效率与计算吞吐量(Computational Throughput)。

## 面向 OLTP 工作负载的 N 元存储模型(N-ary Storage Model, NSM)（行式存储(Row-Oriented Storage)）
![关键帧](keyframes/part001_frame_00119850.jpg)
N 元存储模型(N-ary Storage Model, NSM)，即行式存储，是 PostgreSQL、MySQL 与 SQLite 等系统的默认架构。该模型将单个元组(Tuple)的所有属性(Attribute)连续存储于固定大小的数据页(Data Page)中，页大小通常介于 4KB 至 16KB 之间。此布局针对 OLTP(联机事务处理) 环境进行了深度优化，以高效支撑事务频繁检索、插入或更新单条记录的访问模式。由于整行数据通常可完整容纳于单个数据页内，事务提交(Commit)或刷盘(Flush)所需的 I/O 操作被降至最低。然而，该模型依赖 Volcano 迭代器模型(Volcano Iterator Model)进行逐元组处理，在执行分析型查询时效率低下。在 OLAP(联机分析处理) 场景中，当查询仅需访问众多列中的少数几列时，行式存储会迫使系统读取并解析全部无关属性，从而严重浪费内存带宽与 CPU 缓存(Cache)空间。

## 面向 OLAP 的分解存储模型(Decomposition Storage Model, DSM)（列式存储(Column-Oriented Storage)）
![关键帧](keyframes/part001_frame_00191449.jpg)
为突破行式存储在分析处理中的瓶颈，分解存储模型(Decomposition Storage Model, DSM)，即列式存储，将元组按属性进行垂直拆分。同一列的所有值被连续存储于独立的文件或存储区域中。该架构彻底消除了读取未使用列的 I/O 开销，并生成高度同质的数据分布(Data Distribution)，极为契合高级压缩算法的应用。鉴于 OLAP 系统主要执行只读的批量扫描(Batch Scan)，而非频繁的定点更新(Point Update)，列式存储有效避免了因修改单行数据而需同步更新数十个独立列文件所引发的事务开销(Transaction Overhead)。此类物理文件通常体积庞大（可达数百兆字节），但在逻辑上会被划分为更小、易于管理的块(Block)或行组(Row Group)，从而支持精准的数据跳过(Data Skipping)机制。

## 列式文件的物理布局(Physical Layout)
![关键帧](keyframes/part001_frame_00360783.jpg)
![关键帧](keyframes/part001_frame_00443716.jpg)
在纯列式布局(Pure Columnar Layout)中，每个属性均写入其专属的物理文件中。文件起始处设有元数据头部(Metadata Header)，其中记录系统版本、统计摘要(Statistical Summary)（如用于数据跳过的区域映射(Zone Map)），以及用于追踪缺失值的空值位图(Null Bitmap)。为优化扫描性能并简化内存寻址，列式存储架构极度偏好固定长度数据类型(Fixed-Length Data Type)。当列内每个值占用一致的字节数时，存储引擎即可执行高速且可预测的顺序扫描(Sequential Scan)，无需解析复杂的变长分隔符或长度前缀(Length Prefix)。

## 元组重建(Tuple Reconstruction)与变长数据处理(Variable-Length Data Handling)
![关键帧](keyframes/part001_frame_00529183.jpg)
当查询请求多个属性时，系统必须将垂直拆分的列数据重新“拼接”(Reassemble)。行业标准做法采用隐式偏移量计算(Implicit Offset Calculation)：鉴于执行引擎明确掌握当前处理位置及各列的固定字节宽度(Fixed Byte Width)，可瞬时推算出其他列文件中对应值在内存或磁盘上的精确偏移量(Offset)。替代方案是为每个值嵌入显式元组 ID(Explicit Tuple ID)，并依赖辅助哈希表(Hash Table)或范围查找(Range Lookup)来定位匹配属性，但这会引入显著的额外开销。为在保留定长处理性能优势的同时，原生兼容字符串等常见变长数据类型，列式存储广泛采用字典编码(Dictionary Encoding)技术，将变长值映射为紧凑的定宽整数编码(Fixed-Width Integer Codes)。该机制确保列式引擎在异构数据类型处理中，仍能维持其高吞吐量的扫描性能。