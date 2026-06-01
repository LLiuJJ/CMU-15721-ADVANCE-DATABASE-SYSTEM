## 元数据架构与格式对比
本节首先对比了 Parquet、ORC 以及较新的 *Better Blocks* 等现代列式格式(Columnar Format)。一个关键的设计分歧在于元数据(Metadata)的存储位置：传统格式将数据模式(Data Schema)和编码信息直接嵌入文件页脚(File Footer)，而 *Better Blocks* 则提议将元数据存储在独立的外部管理服务中。支持者认为，这种架构分离允许系统通过字节范围请求(Byte-Range Request)快速扫描元数据（例如 Z-Order 映射(Z-Order Mapping)），从而精准判断是否真正需要读取底层的数据文件(Data File)。然而，讲师指出，像 Amazon S3 这样的云对象存储(Cloud Object Storage)已原生支持高效的字节范围读取，因此内嵌式与外部化元数据在实现查询剪枝(Query Pruning)方面的实际效能是等效的。最终的技术选型，本质上归结于对文件自包含性(Self-Containment)带来的可移植性，与解耦存储架构(Decoupled Storage Architecture)之间的理念权衡(Trade-off)。

![关键帧](keyframes/part003_frame_00000000.jpg)

## 轻量级编码方案分类
*Better Blocks* 引入了一套全面且对中央处理器(CPU)友好的编码策略(Encoding Strategy)，旨在规避 Delta 编码(Delta Encoding)的顺序依赖(Sequential Dependency)以及重量级块压缩(Heavyweight Block Compression)带来的不透明性。该策略涵盖了针对高度重复数据的游程编码(Run-Length Encoding, RLE)，以及源自 IBM DB2 的频率编码(Frequency Encoding)。频率编码利用位图(Bitmap)隔离出高频常见值，并将低频异常值(Outlier)进行单独存储。其他方案包括结合位打包(Bit Packing)的参考系编码(Frame of Reference, FOR)（即相对于全局最小值存储增量，而非依赖相邻值差值）、标准字典编码(Dictionary Encoding)，以及用于浮点数转换的定点小数编码(Fixed-Point Encoding)。在字符串处理方面，系统采用 FSST 算法(Fast Static Symbol Table, FSST)，在字节或子字符串(Sub-string)级别进行压缩映射，而非针对完整字符串进行字典映射。这一特性使其极其适用于 URL 或日志路径(Log Path)等数据。此外，系统利用 roaring 位图(Roaring Bitmaps)高效追踪空值(Null Values)与异常值的出现频率，从而有效避免存储空间的无谓膨胀(Storage Bloat)。

![关键帧](keyframes/part003_frame_00090000.jpg)

## 用于编码优化的空间采样
与 ORC 依赖小型前瞻缓冲区(Look-ahead Buffer)动态预测编码方案不同，*Better Blocks* 在数据加载(Data Ingestion)阶段执行了更为精细的编码选择流程。为规避纯随机采样(Random Sampling)（易破坏 RLE 的数据连续性）或纯顺序采样(Sequential Sampling)（可能无法准确表征列块整体数据分布(Data Distribution)）的缺陷，该算法在包含 64,000 个数据值的列块(Column Chunk)中，跳转至 10 个均匀间隔的起始位置，并在每个位置提取连续的 64 个值。这种混合采样策略(Hybrid Sampling Strategy)既保留了全局数据的随机代表性，又维持了局部数值的连续性，从而使算法能够精准评估游程编码、字典编码或位打包中哪一种为最优解。一旦针对采样数据确定了最佳编码方案，该方案将被直接推广并应用于整个目标数据块。

![关键帧](keyframes/part003_frame_00270000.jpg)

## 贪心选择与计算权衡
该编码选择过程(Encoding Selection Process)对采样数据采用贪心算法(Greedy Algorithm)进行穷举评估，仅在数据加载阶段引入约 2% 的额外计算开销(Computational Overhead)。系统最多会在生成的衍生列(Derived Column)上递归应用(Recursively Apply)所选编码器三次（例如，对 RLE 产出的长度数组再次执行位打包）。尽管有学员质疑为何不采用回溯算法(Backtracking Algorithm)以搜索全局最优的编码路径(Global Optimal Path)，但该设计实为有意规避。在数 TB 级(Terabyte-Scale)的数据集上运行回溯或穷举搜索(Exhaustive Search)将引发难以承受的计算成本，并严重拖慢数据加载吞吐量。当前采用的 1% 采样率(Sampling Rate)代表了一个务实的最佳平衡点，在编码压缩精度与可接受的计算开销之间实现了有效权衡。

![关键帧](keyframes/part003_frame_00390000.jpg)
![关键帧](keyframes/part003_frame_00420000.jpg)
![关键帧](keyframes/part003_frame_00450000.jpg)
![关键帧](keyframes/part003_frame_00480000.jpg)

## 结论：优先保障查询性能
驱动 *Better Blocks* 及同类现代文件格式的核心理念十分明确：分析型数据库(Analytical Database)的绝大多数业务场景均属于读密集型工作负载(Read-Intensive Workload)。通过在数据加载阶段承担可控的前期计算成本(Upfront Computational Cost)，以智能化地进行采样、评估并应用轻量级编码(Lightweight Encoding)，系统确保了最高频的操作场景——即查询执行(Query Execution)与全表数据扫描(Data Scanning)——能够获得显著的性能加速。这种架构权衡(Architectural Trade-off)确保了数据以一种高度契合 SIMD 向量化指令(SIMD Vectorization)且运行时解码开销(Decoding Overhead)极低的结构进行持久化存储。最终，系统在无需应用开发者修改现有 SQL 工作流(SQL Workflow)的前提下，提供了更具成本效益且高效的分析处理能力。

![关键帧](keyframes/part003_frame_00570000.jpg)