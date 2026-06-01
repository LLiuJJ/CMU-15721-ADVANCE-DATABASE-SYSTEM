## 课程简介与注意事项
欢迎来到卡内基梅隆大学高级数据库系统(Advanced Database Systems)课程，本次课程在演播室现场观众面前录制。在深入技术内容之前，先说明一项课程安排：我们将于本周三结束关于项目提案(Project Proposal)汇报的讨论，因此请将相关问题留到届时再提出。

![关键帧](keyframes/part000_frame_00000000.jpg)
![关键帧](keyframes/part000_frame_00009966.jpg)

## 存储模型与文件元数据回顾
今天的课程承接上次内容，重点讲解数据文件的物理布局(Physical Layout)。回顾一下，现代数据库系统通常不再采用纯粹的 N元组存储模型(N-ary Storage Model, NSM)或分解存储模型(Decomposition Storage Model, DSM)，而是转向混合列式架构(Hybrid Columnar Architecture)。数据表会被划分为水平分区或行组(Row Group)，在每个组内，各列的数据会连续排列，处理完一列后再跳转到下一列。这种设计既吸收了列式存储的高压缩率优势，又保留了行式存储的空间局部性(Spatial Locality)。我们还探讨了文件格式的底层细节，包括存储在文件页脚(Footer)中的元数据(Metadata)、行组跳转偏移量、数据布局（行式与列式），以及类型系统(Type System)中的嵌套结构。

![关键帧](keyframes/part000_frame_00040916.jpg)

## 压缩技术与拆分简介
在应用编码方案(Encoding Scheme)之后，标准的数据处理流水线通常会对编码后的数据应用通用的块压缩算法(Block Compression Algorithm)，例如 Snappy 或 Zstandard（请注意，GZip 在现代分析型数据库(Analytical Database)中已很少使用）。我们之前简要介绍了“拆分(Shredding)”的概念，该技术源自 Google BigQuery 的底层引擎 Dremel 项目。今天，在过渡到本学期重点研读的论文之前，我将更详细地讲解拆分技术，因为这一核心概念将在后续课程中反复出现。

## 半结构化数据处理与隐式模式
现实世界的数据集中充斥着 JSON 文档或 Protocol Buffers 数据。将其存储为单一的 `varchar` 或 `blob` 列虽能实现基础的数据提取，但会丧失列式存储的核心性能优势，例如 PAX模型(Partition Attributes Across) 和向量化执行(Vectorized Execution)。取而代之的是，系统会对每个文档进行“拆分(Shred)”或展开操作，将每个 JSON 路径(JSON Path)存储为独立的列。尽管 NoSQL 数据库以“无模式(Schema-less)”为卖点（即无需编写显式的 `CREATE TABLE` 语句），但隐式模式(Implicit Schema)始终存在。应用程序极少生成完全随机的文档；它们之间存在足够的结构重叠(Structural Overlap)，足以将数据可靠地分解为具有明确数据类型的列。

![关键帧](keyframes/part000_frame_00216549.jpg)

## 记录拆分的核心机制
记录拆分(Record Shredding)机制将每个文档路径存储为独立的列，并使用两个辅助整型列来追踪嵌套上下文(Nested Context)：**定义级别(Definition Level)**和**重复级别(Repetition Level)**。定义级别记录了从根节点到达当前值的路径上，实际存在多少个有效的非空步骤。重复级别则用于追踪特定层级上重复组(Repetition Group)的重复次数。与传统的空值填充策略不同，拆分技术避免了为缺失字段存储显式空值标记(Explicit Null Marker)。尽管该方法为每个属性引入了额外的元数据列，但这些整型数组具有极高的压缩率(Compression Ratio)，能在保留优异查询性能的同时，最大限度地降低存储开销。

![关键帧](keyframes/part000_frame_00294683.jpg)

## 逐步示例演示与问答
为了具体说明，假设存在一个类似 Protocol Buffer 或具有明确 JSON Schema 的文档。对于顶层字段 `document ID`，其重复级别(Repetition Level)和定义级别(Definition Level)的初始值均为零。当进入包含重复 `language` 子组的嵌套 `name` 结构时，第一个 `code` 条目的重复级别为零（表示首次出现），定义级别为二（表示路径深入了两层）。当添加 `country` 字段（值为 "US"）时，重复级别设为零，定义级别设为三，以反映其更深的嵌套路径。在处理第二个 `language` 组时，重复级别递增为一。若在某次重复迭代中 `country` 等可选字段缺失，系统仍会记录一条带有相应重复级别的条目，但其定义级别会降低，以反映实际存在的最浅有效路径。这种方法能在不增加额外存储负担的情况下，高效地表示字段缺失。在示例演示环节，关于深度计算规则得到了进一步澄清：定义级别仅严格统计路径中实际存在的非空节点步骤。若某个可选节点缺失，则不计入深度累加，这使查询引擎(Query Engine)能够在执行时准确重建原始的嵌套数据结构。