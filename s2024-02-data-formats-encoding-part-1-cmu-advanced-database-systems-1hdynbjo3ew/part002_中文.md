## 元组标识与定长优化
![关键帧](keyframes/part002_frame_00000000.jpg)
尽管部分混合或遗留系统会在列值中嵌入显式元组标识符(Tuple ID)并依赖辅助哈希表(Auxiliary Hash Table)进行关联查找，但该机制会引入显著的额外开销。现代列式存储的行业标准倾向于采用定长编码(Fixed-Length Encoding)。通过确保每个数据值占用恒定的字节宽度，查询引擎仅需基于当前扫描偏移量(Scan Offset)，即可通过简单且可预测的算术运算精准重建完整元组(Tuple Reconstruction)。此举彻底免除了对复杂指针追踪(Pointer Tracing)或额外索引查找的依赖。即便引入了游程编码(Run-Length Encoding)等压缩算法，系统仍可通过遍历压缩块(Compressed Block)精准定位目标行的确切字节偏移量(Byte Offset)，从而完好保留定长寻址的性能优势。

## 字典编码与半结构化数据处理
![关键帧](keyframes/part002_frame_00038583.jpg)
为在维持定长处理性能的同时兼容字符串等变长数据类型(Variable-Length Data Type)，列式系统广泛采用字典编码(Dictionary Encoding)。该技术将基数庞大(高唯一值域)的原始数据映射为紧凑的定宽整数编码。以 Parquet 为代表的现代文件格式会积极地将字典编码应用于各类数据类型（甚至包括整型），因为该策略能显著压缩数据值域(Cardinality)，并生成高度同质化的数据分布，从而为后续的二级压缩(Secondary Compression)创造极佳条件。针对 JSON 等半结构化数据(Semi-Structured Data)，系统会避免在查询运行时进行耗时的文本解析。取而代之的是，在数据摄入(Data Ingestion)阶段将嵌套的层次结构扁平化(Flattening)为独立的关系列。这些源自嵌套字段的字符串值随后将被统一施以字典编码，使得查询引擎能够沿用标准关系列的高吞吐量定长扫描机制进行处理。

## 多列查询挑战与 PAX 模型
![关键帧](keyframes/part002_frame_00276533.jpg)
分析型工作负载极少孤立地扫描单一列；此类查询通常需要联合多个属性以执行过滤(Filter)、分组(Group By)或聚合(Aggregation)操作。纯粹的分解存储模型(Decomposition Storage Model, DSM)将各属性分散存储于完全独立的物理文件中，这迫使执行引擎为重建完整元组而在不同文件间频繁执行高成本的 I/O 寻址。此种物理碎片化严重破坏了数据访问的空间局部性(Spatial Locality)，进而导致 CPU 缓存效率(Cache Efficiency)大幅下降。为打破此性能权衡，PAX(Partition Attributes Across, 跨属性分区)模型应运而生。PAX 模型既保留了列式存储的核心优势（如高压缩率与向量化执行(Vectorized Execution)），又通过将相关属性值在物理上聚合至更大的内存页或磁盘块中，成功恢复了行式存储所具备的空间局部性。

## 行组、列块与基于元数据的数据跳过
![关键帧](keyframes/part002_frame_00334066.jpg)
PAX 通过数据表的水平切分来实现其混合架构，切分单元被称为“行组”(Row Group)，其体积通常维持在数十兆字节量级。该尺寸设定在元数据管理开销(Metadata Overhead)与细粒度数据跳过(Fine-Grained Data Skipping)的需求之间达成了最佳平衡。在单个行组内部，数据以“列块”(Column Chunk，亦称条带(Stripe))为单位进行顺序排布：即先将某一属性的全部值连续写入，随后再切换至下一属性的存储布局。上述所有结构最终均封装于单一物理文件中，且文件头与文件尾均附带了详尽的元数据信息。文件尾部集中存储了各自行组的区域映射(Zone Map，即最小/最大值等统计摘要)，使得查询规划器(Query Planner)能够仅通过解析元数据即可完成谓词(Predicate)评估。若某行组的统计极值明确超出查询过滤范围，执行引擎将直接跳过对该数据块的整体读取。此举在避免依赖海量零碎小文件的同时，亦未牺牲列式处理的固有优势，从而实现了 I/O 开销的大幅削减。