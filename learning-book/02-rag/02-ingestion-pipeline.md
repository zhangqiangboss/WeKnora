# 第 6 章　入库链路：文档如何变成向量索引

[返回全书目录](../README.md) · [上一篇：一张图理解 RAG](01-rag-overview.md) · [下一篇：从查询到流式答案](03-query-pipeline.md)

## 本章要解决的问题

用户看到的是“上传一个 PDF”，RAG 系统看到的却是一条可能持续很久、跨多个组件、需要幂等和失败恢复的数据加工任务。

本章沿 WeKnora 的实现理解：原始文档怎样变成数据库中的 Chunk 和检索引擎中的索引。

## 先区分四种数据形态

一份文档进入系统后，至少会出现四种形态：

1. **原始文件**：用于下载、预览、重新解析和审计。
2. **解析结果**：从 PDF/Word/HTML 等格式提取出的文本、图片和结构信息。
3. **Chunk 记录**：带内容、顺序、原文位置、父子关系和元数据的知识片段。
4. **检索索引**：向量、关键词字段以及指向 Chunk 的 ID。

不要把 Chunk 表与向量索引看成同一个存储。向量搜索返回的索引项最终必须能关联回可展示、可引用的 Chunk 和原始文件。

## 主链路

```mermaid
flowchart LR
    A["上传文件/URL/手工文本"] --> B["创建 Knowledge 记录"]
    B --> C["保存原始内容"]
    C --> D["提交解析任务"]
    D --> E["DocParser/DocReader 解析"]
    E --> F["Chunker 切分"]
    F --> G["保存 Chunk 到数据库"]
    G --> H["批量 Embedding"]
    H --> I["写入向量/关键词索引"]
    I --> J["更新处理状态与后续任务"]
```

真实代码还包含图片、多模态、图谱、Wiki、摘要、标签等后处理。本章第一遍只保留文本 RAG 主干。

## 第一步：创建知识记录，而不是同步处理完文件

上传入口位于知识库范围内，例如路由 [`routes_knowledge.go`](../../internal/router/routes_knowledge.go#L67) 注册：

- 文件上传；
- URL 导入；
- 手工知识。

Handler 负责接收请求，Application Service 创建 Knowledge 元数据、保存文件并安排后续处理。这样的 API 通常先返回任务/知识标识，而不是让一个 HTTP 请求阻塞到所有 Embedding 完成。

这是传统后台任务模式在 AI 数据加工中的应用。文档解析和向量化可能受到文件大小、外部解析器和模型限流影响，不适合绑定在短生命周期请求中。

## 第二步：解析不是简单读取字符串

不同文档格式的“正文”并不天然存在：

- PDF 保存的可能是页面坐标上的字符和绘制命令；
- 扫描件需要 OCR；
- Word 有标题层级、表格、页眉和图片；
- HTML 有导航、广告、脚本和正文；
- Excel 的行列关系无法直接等价成连续段落。

解析器需要输出统一的中间形态，同时尽量保留：

- 文本；
- 段落/标题结构；
- 页面或字符位置；
- 图片与说明；
- 表格或版面信息；
- 原始文件引用。

WeKnora 将文档解析基础设施放在 [`internal/infrastructure/docparser`](../../internal/infrastructure/docparser)，标准版还可连接独立 DocReader 服务。Lite 模式则强调内置 Simple 解析和较少外部依赖，差异见 [`docs/LITE.md`](../../docs/LITE.md)。

## 第三步：Chunking 建立可检索边界

解析结果还不是最终 Chunk。Chunker 根据知识库配置选择策略、大小、重叠和父子模式。

WeKnora 的切块基础设施位于 [`internal/infrastructure/chunker`](../../internal/infrastructure/chunker)。当前设计支持：

- 标题感知切分；
- PDF/章节启发式切分；
- 递归分隔符切分；
- 自动策略选择与失败回退；
- 父子 Chunk；
- 上下文标题 breadcrumb。

具体策略和默认参数可先看 [`docs/CHUNKING.md`](../../docs/CHUNKING.md)。

### 父子 Chunk 的存储语义

在父子模式中：

- 子 Chunk 较小，用于生成向量和匹配；
- 父 Chunk 较大，保存在数据库，用于命中后的上下文扩展；
- 父 Chunk 通常不需要单独写向量索引。

这在 [`knowledge_process.go`](../../internal/application/service/knowledge_process.go#L400) 附近表现得很清楚：父 Chunk 先加入数据库写入集合；真正用于 `textChunks` 和后续索引的是子 Chunk，或未启用父子模式时的普通 Chunk。

## 第四步：先保存 Chunk，再建立索引

WeKnora 在 [`knowledge_process.go`](../../internal/application/service/knowledge_process.go#L476) 中明确指出：无论是否启用向量/关键词索引，Chunk 都会保存到数据库，因为 Wiki、图谱和摘要等功能也需要它们。

这体现了一个重要边界：

```text
Chunk 是知识加工后的业务数据
向量/关键词条目是针对某种检索能力建立的派生索引
```

派生索引可以重建，Chunk 和原始文件则承载更稳定的业务含义。

每个文本 Chunk 包含的典型信息包括：

- `ID`；
- 租户、知识库和知识 ID；
- 文本内容；
- 标题上下文；
- 顺序和原文起止位置；
- 父 Chunk ID；
- 前后 Chunk 关系；
- 是否启用。

这些字段让后续检索结果能够扩展上下文、恢复顺序并生成引用。

## 第五步：构造真正用于 Embedding 的文本

向量化的输入不一定只是 Chunk 原文。WeKnora 在建立 `IndexInfo` 时会组合：

- 文档标题；
- Chunk 的标题 breadcrumb；
- Chunk 内容；
- 部分文档级元数据。

入口位于 [`knowledge_process.go`](../../internal/application/service/knowledge_process.go#L500) 和 [`knowledge_index_content.go`](../../internal/application/service/knowledge_index_content.go)。

这样做的原因是，用户问题往往包含文档或章节语境，而一个孤立片段可能只写“上限为 500 元”。把“差旅制度 > 住宿标准 > 北京”等上下文放入 Embedding 输入，有助于语义对齐。

代价是增加 Embedding Token，并可能让重复标题过度影响相似度。它是检索效果与成本之间的选择，不是无条件规则。

## 第六步：批量 Embedding 与写索引

Embedding 调用通常支持批量输入。逐 Chunk 发 HTTP 请求会造成严重网络开销和限流压力，因此 WeKnora 通过 `BatchEmbed` 和池化/并发控制批量处理。

模型抽象见 [`Embedder`](../../internal/models/embedding/embedder.go#L16)，批处理辅助逻辑见 [`internal/models/embedding/batch.go`](../../internal/models/embedding/batch.go)。

在处理服务中，`retrieveEngine.BatchIndex` 负责结合 Embedding 模型和索引信息完成批量写入，调用位置在 [`knowledge_process.go`](../../internal/application/service/knowledge_process.go#L569)。

“检索引擎”而不是业务 Service 负责不同后端的具体写法，使 PostgreSQL/pgvector、SQLite、Elasticsearch、Milvus、Qdrant 等实现可以共享上层流程。

## 幂等为什么是入库链路的核心

用户可能：

- 对同一文档重新解析；
- 修改 Chunking 参数后重建；
- 在处理途中取消；
- 删除文档；
- 因外部限流而重试任务。

如果系统只会“追加新 Chunk 和新向量”，很快会出现重复召回和脏索引。

WeKnora 在重新处理前清理旧 Chunk 和旧索引；写入前后多次检查知识是否已被删除或取消；失败时清理已经写入的部分数据。相关主干从 [`knowledge_process.go`](../../internal/application/service/knowledge_process.go#L275) 开始。

这部分代码看起来不像 AI，却直接决定检索结果是否可信。AI 工程里许多质量问题最终仍是经典数据一致性问题。

## 状态与可观测性

文档处理不是单一的“成功/失败”，而是解析、切块、Embedding、多模态、后处理等阶段。WeKnora 会记录阶段开始、结束、失败和统计信息，例如：

- 计划/实际写入的 Chunk 数；
- 总字符数；
- Embedding 模型和维度；
- 写入的向量数量；
- 存储大小；
- 稳定错误码。

这些信息既服务于 UI 进度，也服务于排障和重试。对 AI 数据管线而言，“在哪个阶段失败”通常比一条最终异常消息更有价值。

## 源码阅读顺序

建议只沿主干读：

1. [`internal/router/routes_knowledge.go`](../../internal/router/routes_knowledge.go#L67)：找到三类知识创建入口。
2. [`internal/handler/knowledge.go`](../../internal/handler/knowledge.go)：观察请求如何转入 Service。
3. [`internal/application/service/knowledge_create.go`](../../internal/application/service/knowledge_create.go)：创建记录、保存内容和提交任务。
4. [`internal/application/service/knowledge_process.go`](../../internal/application/service/knowledge_process.go#L243)：Chunk 处理和索引主干。
5. [`internal/infrastructure/chunker`](../../internal/infrastructure/chunker)：切块策略。
6. [`internal/models/embedding/embedder.go`](../../internal/models/embedding/embedder.go#L16)：Embedding 模型抽象。
7. [`internal/application/service/retriever`](../../internal/application/service/retriever)：检索引擎的工厂与组合。

### 第一遍主动略过

- 图片和视频的多模态后处理；
- 知识图谱和 Wiki 任务；
- 跨租户共享；
- 所有向量数据库的具体实现；
- 删除过程中的每个补偿分支。

先确认主干，再选择一个主题深入。

## 设计取舍

### 同步处理还是异步处理

同步实现简单，但大文件、外部模型和失败重试会拖垮请求生命周期。异步处理增加队列、状态和幂等复杂度，却更适合生产负载。

### Chunk 存数据库还是只存向量库

只存向量库看似简单，但不利于编辑、版本、引用、父子关系和重建。WeKnora 把 Chunk 当业务数据，向量作为派生索引。

### 一个固定切块器还是多策略

固定切块便于理解和调试；多策略能利用 Markdown 标题、PDF 页和章节结构，但增加配置与回退逻辑。WeKnora 面向多格式知识平台，因此选择后者。

## 本章小结

- 上传文档只是入库任务的起点，不是终点。
- 原始文件、解析结果、Chunk 和检索索引是不同数据形态。
- Chunk 是业务知识单元，向量索引是可以重建的派生数据。
- 父子 Chunk 用小片段匹配、大片段提供上下文。
- Embedding 输入可以加入标题和结构信息，不必等于原文。
- 重解析、取消、重试和删除要求整条链路具备幂等与阶段状态。

### 可选实践

从项目 UI 的 Chunk 预览功能选择同一段 Markdown，分别使用标题切分和递归切分。只比较输出片段及其标题上下文，不需要继续调用模型。

下一章：[问答链路：从查询到带引用的流式答案](03-query-pipeline.md)
