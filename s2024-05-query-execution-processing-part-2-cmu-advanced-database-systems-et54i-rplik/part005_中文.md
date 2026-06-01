## 优化 Arrow 中的变长字符串存储
Apache Arrow 的基础设计通过在主列向量(Main Column Vector)中存储 12 字节的定宽条目（包含长度字段(Length Field)与指针/偏移量(Pointer/Offset)）来处理变长字符串(Variable-Length String)，而实际的字符数据则存放在独立的连续 Blob 缓冲区(Blob Buffer)中。尽管该方案可行，但会迫使执行引擎(Execution Engine)在每次访问字符串时执行指针解引用(Pointer Dereferencing)。为解决此瓶颈，该规范借鉴了德国数据库研究（特别是 Umbra 项目）中更高效的内存布局(Memory Layout)进行了扩展。优化后的方案为每个列条目分配 16 字节：包含一个头部(Header)、一个长度字段、4 字节的字符串前缀(String Prefix)，以及一个 8 字节的指针或偏移量。若字符串长度可容纳于该空间内，则采用内联存储(Inline Storage)并以零填充(Zero-padding)。对于较长字符串，4 字节前缀保持内联，而指针则指向外部 Blob 区域中的完整字符串。关键在于，完整字符串数据仍保留在外部，以避免在查询执行(Query Execution)期间重组零散数据所带来的额外开销。
![关键帧](keyframes/part005_frame_00000000.jpg)

## 前缀内联存储的性能优势与应用
这种 16 字节布局的核心优势在于大幅降低了内存访问中的指针解引用(Pointer Dereferencing)频率。对于诸多常见操作（如基于字符串前缀的过滤(Filter)或模式匹配(Pattern Matching)），执行引擎可直接基于定宽的内联数据评估谓词(Predicate)，无需访问外部 Blob 区域。该设计与高性能数据库工程中“极致压榨每一可用比特(Exploit Every Available Bit)”的趋势高度一致；例如，将布隆过滤器(Bloom Filter)嵌入 64 位 x86 指针的高位闲置位(High Unused Bits)中，以快速剪枝(Prune)哈希表查找(Hash Table Lookup)。DuckDB、Velox 和 Polars 等现代系统均已广泛采用此项技术。在实际应用中，它实现了高效的两阶段过滤(Two-Stage Filtering)：引擎首先扫描紧凑的前缀列以筛选候选匹配项(Candidate Match)，随后仅针对命中元组(Tuple)按需获取完整字符串。此外，该设计极大简化了去重(Deduplication)逻辑，因为共享相同数据的字符串可由同一指针表示，从而使系统最大限度地减少了冗余的内存访问(Memory Access)与计算开销。
![关键帧](keyframes/part005_frame_00144383.jpg)
![关键帧](keyframes/part005_frame_00158499.jpg)
![关键帧](keyframes/part005_frame_00197683.jpg)
![关键帧](keyframes/part005_frame_00212649.jpg)

## 使用 Substrate 标准化查询计划
除统一数据表示(Data Representation)外，该生态系统正逐步迈向查询执行计划(Query Execution Plan)的标准化。Substrate 是一项开源规范(Open Specification)，旨在以通用且系统无关(System-Agnostic)的格式表示关系代数查询计划(Relational Algebra Query Plan)。其核心目标是将查询优化层(Query Optimization Layer)与底层执行引擎解耦(Decouple)，使得独立的优化器(Optimizer)能够生成可被任意兼容执行引擎解析的 Substrate 计划。尽管在理念上与 Apache Arrow 标准化数据传输的方式相似，但 Substrate 致力于对查询逻辑(Query Logic)实现同等的标准化。然而，其当前实现多局限于简单查询场景，且仅由小型团队维护。相较于拥有广泛产业支持的 Apache Arrow 联盟，Substrate 在生态扩展性(Ecosystem Scalability)方面面临显著挑战。尽管其在解耦数据库组件(Database Component)方面颇具潜力，但目前 Substrate 仍属小众工具(Niche Tool)，尚未成为通用的行业标准。
![关键帧](keyframes/part005_frame_00473133.jpg)

## DataFusion：全面的基于 Arrow 的执行引擎
DataFusion 已成为基于 Apache Arrow 构建的执行引擎(Execution Engine)的领先实现。与严格聚焦于底层算子执行(Low-Level Operator Execution)的 Velox 不同，DataFusion 提供了更为完整的软件栈(Software Stack)，涵盖 SQL 解析前端(SQL Frontend)、查询优化器(Query Optimizer)及向量化执行运行时(Vectorized Execution Runtime)。这使得它对于那些希望快速构建新型数据库系统且不愿重复开发基础组件的开发者极具吸引力。目前，它已被众多现代平台采纳为核心执行引擎，其中最典型的案例是 InfluxDB 3.0，该平台近期完成了重大架构重构(Architectural Refactoring)以全面拥抱 SQL 与 Arrow 生态。其他如 CnosDB 以及 pg_analytics（PostgreSQL 的 DataFusion 扩展插件）等系统也深度集成了该引擎。随着在业界的广泛落地（包括 Snowflake 提供的原生支持），Arrow 与 DataFusion 技术栈实质上已确立为现代分析数据处理(Analytical Data Processing)的事实标准(De Facto Standard)。
![关键帧](keyframes/part005_frame_00573916.jpg)

## 表达式求值简介
查询执行(Query Execution)的核心环节之一涉及表达式求值(Expression Evaluation)，它决定了在表扫描(Table Scan)或聚合(Aggregation)操作期间，如何将谓词(Predicate)、过滤条件(Filter Condition)及连接条件(Join Condition)应用于底层数据。现代执行引擎摒弃了在运行时动态解释(Run-time Interpretation)每条条件的传统模式，转而将表达式树(Expression Tree)编译为高度优化的向量化代码(Vectorized Code)。该编译过程确保布尔校验(Boolean Check)、算术运算(Arithmetic Operation)与函数调用(Function Call)能够批量应用于整个数据批次(Data Batch)，从而充分释放 CPU 的 SIMD 指令集(SIMD Instruction Set)性能，并契合缓存友好型内存布局(Cache-Friendly Memory Layout)。深入理解表达式求值机制，是构建深度向量化执行引擎的基石，这也将在后续关于查询处理流水线(Query Processing Pipeline)的探讨中作为核心议题展开。
![关键帧](keyframes/part005_frame_00586683.jpg)