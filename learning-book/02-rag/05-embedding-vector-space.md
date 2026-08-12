# 第 9 章　Embedding 与向量空间

[返回全书目录](../README.md) · [上一篇：Chunking](04-chunking.md) · [下一篇：关键词、向量与混合检索](06-hybrid-retrieval.md)

## 本章要解决的问题

Embedding 常被描述成“把文本转成一串数字”。这个说法没有错，却无法解释：为什么问题能找到措辞不同的答案、为什么不同模型的向量不能混用、为什么维度影响存储，以及为什么换模型要重建索引。

## 从字符串匹配到语义位置

关键词检索比较词项。Embedding 模型则把输入映射成固定维度向量：

```text
"差旅住宿报销上限" → [0.018, -0.227, ..., 0.091]
```

单个维度通常没有可直接解释的业务含义。重要的是整个向量在模型空间中的相对位置。

如果模型训练良好：

```text
"费用上限" 与 "最高报销标准" 距离较近
"费用上限" 与 "数据库连接池" 距离较远
```

向量检索因此能发现没有共享精确词语、但语义相近的片段。

## 相似度是怎样计算的

常见指标包括：

### Cosine Similarity

比较两个向量方向的夹角：

```text
cos(q, d) = (q · d) / (|q| |d|)
```

方向越相近，Cosine 越高。

### Dot Product

直接计算点积 `q · d`。当向量已经归一化时，它与 Cosine 排序等价；未归一化时还受向量长度影响。

### Euclidean Distance

比较几何距离。距离越小越相似。

不同向量数据库可能返回“相似度”或“距离”，范围与方向也不同。上层不能假设所有 `Score=0.8` 含义相同。

WeKnora 在多检索后端 fan-out 时使用引擎感知的 Score Normalizer，入口见 [`internal/application/service/retriever/normalizer.go`](../../internal/application/service/retriever/normalizer.go)。

## 文档向量与查询向量必须来自同一空间

Embedding 空间由模型、版本、维度和部分输入约定共同决定。使用模型 A 索引文档，却用模型 B 计算问题向量，相当于拿两套不同坐标系中的点直接算距离，结果没有可靠语义。

因此必须保证：

- 文档和查询使用同一 Embedding 模型或明确兼容模型；
- 维度一致；
- 输入前缀/任务类型约定一致；
- 模型升级后重建旧索引。

WeKnora 的跨知识库 `HybridSearch` 在检索前执行 Embedding 模型一致性检查，见 [`knowledgebase_search.go`](../../internal/application/service/knowledgebase_search.go#L133)。共享知识库场景还会使用知识库所属租户的 Embedding 配置生成查询向量，见 [`model.go`](../../internal/application/service/model.go#L439)。

## 维度意味着什么

维度是向量中浮点数的数量，例如 768、1024 或 1536。更高维度可能容纳更丰富表示，但不自动等于更高业务质量。

维度会影响：

- 每条向量的存储大小；
- 内存和索引体积；
- 距离计算成本；
- 数据库索引结构；
- 网络传输；
- 模型是否支持维度裁剪。

粗略估算，`float32` 每维 4 字节：

```text
100 万条 × 1536 维 × 4 字节 ≈ 5.7 GiB 原始向量
```

还未包括索引、元数据、对齐和副本。因此 Chunk 数与维度会共同放大存储成本。

WeKnora 在 [`EmbeddingParameters`](../../internal/types/model.go#L58) 保存维度和是否支持覆盖，并在入库前估算存储需求。

## Embedding 输入不是原文的机械复制

高质量检索需要让查询表达与文档表达在语义上对齐。WeKnora 建立索引时可能在 Chunk 内容外增加：

- 文档标题；
- 标题 breadcrumb；
- 文档级元数据。

具体入口是 [`buildKnowledgeIndexContent`](../../internal/application/service/knowledge_index_content.go) 和 [`knowledge_process.go`](../../internal/application/service/knowledge_process.go#L514)。

这样，孤立的“上限为 500 元”会带上“差旅制度 > 北京住宿标准”的语境。

某些 Embedding 模型还要求查询与文档使用不同前缀，例如“query:”和“passage:”。这属于模型契约，必须由适配层或配置统一处理，不能只在一侧添加。

## 批量向量化

文档入库可能包含成千上万个 Chunk。逐条调用远程 API 会产生大量网络开销，正确做法通常是：

1. 按模型批次上限分组；
2. 使用 `BatchEmbed`；
3. 控制并发；
4. 校验返回数量与顺序；
5. 将向量与 Chunk ID 对齐写入索引；
6. 对部分失败进行明确重试或整批失败处理。

WeKnora 的接口同时提供 `Embed` 与 `BatchEmbed`，批处理辅助逻辑见 [`internal/models/embedding/batch.go`](../../internal/models/embedding/batch.go)。真正索引时由 `retrieveEngine.BatchIndex` 协调模型和后端。

## 查询时只计算一次向量

当一次请求跨多个知识库或向量存储，但它们共享同一 Embedding 模型时，问题向量没有必要重复计算。

WeKnora 的 `HybridSearch` 在 fan-out 前预先计算 `QueryEmbedding`，再传给各存储分组。相关代码见 [`knowledgebase_search.go`](../../internal/application/service/knowledgebase_search.go#L156)。

这是一项简单但重要的优化：假设一次查询跨 N 个存储，预计算可以把 N 次相同模型请求降低为 1 次，同时减少成本和延迟波动。

## 向量索引不是普通 B-Tree

高维空间中的精确全量距离计算成本很高。向量数据库通常使用近似最近邻索引，例如 HNSW，在召回质量、内存、构建时间和查询速度之间取舍。

“近似”意味着：

- 理论上最相似的向量可能偶尔未被找到；
- 索引参数会影响召回率和速度；
- Top-K 越大、搜索越深通常成本越高；
- 数据量和维度会改变最佳参数。

WeKnora 把这部分封装在不同 Retriever Repository 中。第一次阅读只需选 PostgreSQL 或 SQLite 实现：

- [`retriever/postgres`](../../internal/application/repository/retriever/postgres)
- [`retriever/sqlite`](../../internal/application/repository/retriever/sqlite)

## 阈值和 Top-K 是两类不同过滤

### Top-K

无论分数如何，取排名最前的 K 条。它控制候选数量。

### Threshold

只保留分数达到最低要求的结果。它控制最低相关性。

二者一起使用时：

```text
先按阈值排除明显不相关 → 再取 Top-K
```

但分数分布依模型、后端和语料而异，不能照搬别人系统中的 0.7。WeKnora 的租户检索配置给出默认 VectorThreshold 与 EmbeddingTopK，定义见 [`retrieval_config.go`](../../internal/types/retrieval_config.go)。

阈值必须通过真实问题集校准，并观察无答案问题的误召回。

## 为什么向量检索会失败

### 1. Chunk 中没有完整答案

这是知识边界问题，不是向量问题。

### 2. 问题与文档表达差异超出模型能力

可尝试更适合领域/语言的模型、查询改写或加入标题上下文。

### 3. 精确标识符被语义稀释

错误码、型号和人名通常需要关键词检索补充。

### 4. 阈值过高或 Top-K 太小

正确片段被搜索策略截断。

### 5. 模型空间不一致

文档和查询使用不同模型、维度或前缀。

### 6. 近似索引参数不足

大规模索引搜索深度不够，候选未被访问。

## 模型升级为什么需要迁移计划

更换 Chat 模型后，旧消息仍可继续使用；更换 Embedding 模型后，已有文档向量通常必须全量重建。

迁移需要考虑：

- 新旧索引是否双写；
- 重建期间查询走哪一套；
- 存储是否暂时翻倍；
- 如何回滚；
- 所有知识库是否使用相同版本；
- 重建后如何验证召回没有退化。

Embedding 选择因此是数据架构决策，而不仅是一个 API 配置。

## 源码观察点

1. [`internal/models/embedding/embedder.go`](../../internal/models/embedding/embedder.go#L16)：统一接口、配置与供应商工厂。
2. [`internal/models/embedding/batch.go`](../../internal/models/embedding/batch.go)：批量处理与结果对齐。
3. [`internal/application/service/knowledge_process.go`](../../internal/application/service/knowledge_process.go#L500)：文档 Chunk 如何生成索引项并批量写入。
4. [`internal/application/service/knowledgebase_search.go`](../../internal/application/service/knowledgebase_search.go#L15)：查询向量怎样生成和复用。
5. [`internal/application/service/retriever/normalizer.go`](../../internal/application/service/retriever/normalizer.go)：不同后端分数的工程处理。
6. [`internal/types/interfaces/retriever.go`](../../internal/types/interfaces/retriever.go)：检索引擎抽象。

## 本章小结

- Embedding 将内容映射到可比较的语义空间，不是摘要或答案。
- 文档与查询必须使用一致的模型空间、维度和输入约定。
- 维度、Chunk 数和索引结构共同决定存储与查询成本。
- Embedding 输入可以加入标题和结构信息以改善语义对齐。
- 批处理、并发和查询向量复用是关键工程优化。
- Top-K 控制数量，Threshold 控制最低相关性，两者都需按语料校准。
- 更换 Embedding 通常需要全量重建索引和迁移方案。

### 可选实践

选择十个短句，用同一个 Embedding 模型生成向量并计算两两 Cosine 相似度。先写下你预期最相似的句子对，再比较实际结果。

下一章：[关键词、向量与混合检索](06-hybrid-retrieval.md)
