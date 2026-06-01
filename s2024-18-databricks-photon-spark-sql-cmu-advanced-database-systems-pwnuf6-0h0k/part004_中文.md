## Java-C++ 边界(Java-C++ Boundary)及数据适配器(Adapter)
Photon 作为一款 C++ 引擎，通过 Java 原生接口(Java Native Interface, JNI) 被 Spark 的 JVM 调用。为了弥合底层架构差异，系统引入了“适配器”与“过渡算子(Transitional Operator)”，将传入的 Java 行式数据(Row-based Data) 转换为 Photon 内部的列式向量化格式(Columnar Vectorized Format)，并在计算完成后将结果转换回 Java 兼容的处理格式。尽管这些适配器在逻辑实现上较为轻量，但其底层涉及高昂的数据转置(Data Transposition)、内存复制与数据移动开销。因此，查询优化器的核心策略是尽可能使数据在 Photon 引擎内部闭环流转，以最大限度地减少跨语言边界的交互损耗。
![关键帧](keyframes/part004_frame_00000000.jpg)

## 宏观与微观自适应策略(Macro & Micro Adaptive Strategies)
为应对数据湖仓(Lakehouse)环境中不可预测的数据分布特征，Photon 实现了两种差异化的自适应执行范式。“宏观(Macro)”自适应作用于查询(Query)级别，通常在 Shuffle 操作之后触发。它会实时分析运行时统计信息(Runtime Statistics)来动态重构执行计划，例如调整并行 Worker 节点数量，或将 Shuffle Join(Shuffle Join) 降级切换为广播连接(Broadcast Join)。相反，“微观(Micro)”自适应则在单个算子内部的批次(Batch)级别运行。当数据流经元组批次或向量化数据块时，Photon 会依据实时数据特征自动切换至最优的算子底层实现。该过程对上层应用完全透明，无需查询优化器在运行时进行干预。
![关键帧](keyframes/part004_frame_00057000.jpg)

## 针对倾斜数据的动态分区管理
在宏观层面，Photon 采用一种主动式分区管理(Proactive Partition Management)策略，以缓解连接操作(Join)中的数据倾斜(Data Skew)与内存压力。与部分系统在数据溢出(Spill)时动态创建新分区的做法不同，Spark 会预先分配规模远超初始估算值的分区池(Partition Pool)。当某个 Worker 的活跃分区面临内存枯竭时，系统会自动识别出利用率较低的预分配分区，将溢出的数据迁移并整合至这些空闲分区中，同时释放已饱和分区的内存资源。尽管该过程同样涉及额外的数据移动(Data Movement)开销，但在缺乏准确先验统计信息(Prior Statistics)的复杂场景下，这种动态整合技术能有效降低内存碎片化风险，防止内存溢出(Out Of Memory, OOM)故障的发生。
![关键帧](keyframes/part004_frame_00215833.jpg)
![关键帧](keyframes/part004_frame_00256099.jpg)
![关键帧](keyframes/part004_frame_00269999.jpg)

## 基于模板的原语与批次级优化
微观自适应机制充分利用预编译的模板化原语(Templated Primitives)来消除运行时开销。例如，当检测到某字符串列仅包含 ASCII 字符时，Photon 会自动切换至专用处理路径，从而规避多字节 Unicode 编码处理中固有的分支预测失败(Branch Mispredictions)问题。此外，Photon 会对稀疏向量(Sparse Vectors)进行压缩编码，以优化哈希表探测(Hash Table Probe)阶段的缓存局部性(Cache Locality)。借助编译期布尔模板(Compile-time Boolean Templates)，代码生成器会静态派生出多个函数变体，预先确定是否跳过空值(Null)或非活跃行(Inactive Rows)的评估逻辑。在运行时，Photon 仅依据批次元数据(Batch Metadata)动态路由至最优变体，彻底剔除冗余的条件判断分支。该机制带来了显著的性能收益，例如在连接探测(Join Probes)场景下可实现高达 1.5 倍的执行加速。
![关键帧](keyframes/part004_frame_00332400.jpg)
![关键帧](keyframes/part004_frame_00413099.jpg)

## 基准测试结果与云定价模型
基于大规模 TPC-H 基准测试(TPC-H Benchmark)的评估结果显示，相较于标准 Spark SQL，Photon 展现出显著的性能优势，各类查询的执行耗时均实现大幅缩减。对于结构相对简单、以数据扫描为主的查询(Scan-Intensive Queries)，将计算算子完全卸载(Offload)至原生 C++ 引擎所能获得的性能收益最为显著。在商业化层面，Databricks 针对启用 Photon 加速的计算作业采用溢价定价(Premium Pricing)策略。该策略构建了一种双赢生态：用户端获得了极致的查询响应速度与大幅缩短的总计算时长；平台端尽管单位计算资源的定价略高，却通过缩短作业运行时间有效降低了集群占用周期，从而整体优化了资源利用率(Resource Utilization)。
![关键帧](keyframes/part004_frame_00512816.jpg)