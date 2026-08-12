# 第 7 章　问答链路：从查询到带引用的流式答案

[返回全书目录](../README.md) · [上一篇：文档如何变成向量索引](02-ingestion-pipeline.md) · [下一篇：Chunking](04-chunking.md)

## 本章要解决的问题

知识已经被处理为 Chunk 和索引。现在用户提交一个问题，WeKnora 怎样确定搜索范围、召回证据、组织消息，并把回答逐步返回浏览器？

本章追踪普通知识问答，不进入 Agent 循环。

## 从浏览器开始

前端不是等待一个完整 JSON，而是通过 `fetchEventSource` 发起可携带 POST Body 的 SSE 请求。主要代码位于 [`frontend/src/api/chat/streame.ts`](../../frontend/src/api/chat/streame.ts#L24)。

前端流管理需要维护：

- `isStreaming`：当前是否仍在接收；
- `isLoading`：是否还在等待初始响应；
- `AbortController`：用户停止时取消请求；
- 数据缓冲；
- chunk handler：把不同事件交给聊天状态处理；
- generation 标识：避免旧流关闭时污染新请求。

这里已经体现出 AI 交互与普通表单提交的差异：用户会看到“查询理解、检索、思考、生成”的时间过程，而不只是最终结果。

## HTTP 入口只选择模式

普通知识问答路由是：

```text
POST /api/v1/knowledge-chat/:session_id
```

路由定义见 [`routes_chat.go`](../../internal/router/routes_chat.go#L74)，Handler 见 [`KnowledgeQA`](../../internal/handler/session/qa.go#L791)。

Handler 做两件主要事情：

1. 解析和校验会话、问题、知识范围、附件、模型等请求上下文；
2. 选择普通 QA 模式并交给统一执行函数。

它没有直接编写向量查询或供应商 HTTP 请求。和良好 Java 分层一样，Controller/Handler 不应该成为 AI 逻辑的堆放处。

## 创建一次问答的运行状态

真正的普通 RAG 编排从 [`sessionService.KnowledgeQA`](../../internal/application/service/session_knowledge_qa.go#L23) 开始。

它先解析四类关键输入：

1. **检索范围**：知识库、指定文档、标签或共享知识范围；
2. **模型**：本次使用的 Chat、Embedding 和可能的 Rerank 模型；
3. **策略参数**：召回 Top-K、阈值、改写、回退、历史轮数；
4. **交互上下文**：会话、图片、附件、引用内容、语言和 EventBus。

这些内容被放入 `ChatManage`：

- `PipelineRequest`：本次管线的输入和配置；
- `PipelineState`：各阶段逐步产生的中间状态；
- `PipelineContext`：事件总线、消息 ID 等运行时依赖。

可以把它类比为一个显式的 Workflow Context。不同阶段读取前序结果并写入新状态，避免通过大量散乱参数传递。

## 先确定检索边界，再谈相关性

搜索不是“拿用户问题查整个向量库”。系统必须先确定用户允许搜索什么。

WeKnora 的 `buildSearchTargets` 会把知识库 ID、文档 ID 和标签范围归一成统一的 `SearchTarget`，同时处理租户与共享权限。入口见 [`session_knowledge_qa.go`](../../internal/application/service/session_knowledge_qa.go#L432)。

这表达了一个重要原则：

```text
安全过滤是检索定义的一部分，不是拿到结果后的附加过滤。
```

如果先跨所有租户召回，再在应用层删掉无权限结果，既有泄漏风险，也会让无权限候选占据 Top-K，降低合法结果召回率。

## 纯聊天与 RAG 的分流

系统并非每次都执行检索。如果没有知识范围，也没有启用网页搜索，就可以直接构造用户消息并流式调用 Chat 模型。

当存在知识检索范围或网页搜索时，才动态组装 RAG 管线。分流逻辑见 [`session_knowledge_qa.go`](../../internal/application/service/session_knowledge_qa.go#L156)。

这避免无意义的 Embedding、搜索和 Rerank 延迟，也说明“聊天”和“知识问答”共享模型能力，但不是同一条执行路径。

## 管线阶段逐步展开

### 1. LOAD_HISTORY

在启用多轮对话时加载历史消息。历史不是越多越好：它占用上下文，也可能把旧问题中的检索范围和错误假设带入当前轮次。

因此系统通常按轮数、Token 或摘要策略限制历史。

### 2. QUERY_UNDERSTAND

用户当前问题可能依赖上文，例如：

> 那北京呢？

单独搜索这句话几乎没有意义。查询理解阶段可以结合历史进行改写或扩展，得到更独立的搜索表达。

这一阶段可能提升召回，也可能因改写偏离原意而伤害结果，所以需要保留原问题并可观测改写结果。实现位于 [`query_understand.go`](../../internal/application/service/chat_pipeline/query_understand.go)。

### 3. CHUNK_SEARCH_PARALLEL

搜索阶段对多个知识范围以及可能的网络搜索并行处理。知识库搜索进一步进行向量与关键词召回。

并行版本见 [`search_parallel.go`](../../internal/application/service/chat_pipeline/search_parallel.go)，核心知识搜索插件见 [`search.go`](../../internal/application/service/chat_pipeline/search.go)。

底层 `HybridSearch` 的主干位于 [`knowledgebase_search.go`](../../internal/application/service/knowledgebase_search.go#L87)：

1. 检查知识库和 Embedding 空间；
2. 计算或复用查询向量；
3. 按实际检索后端分组；
4. 执行向量和关键词检索；
5. 归一化、融合或去重结果；
6. 恢复 Chunk 上下文。

### 4. CHUNK_RERANK

候选召回完成后，Rerank 模型根据查询重新计算候选相关性，并应用阈值与数量限制。

实现位于 [`rerank.go`](../../internal/application/service/chat_pipeline/rerank.go)。如果没有配置 Rerank 模型，系统需要有明确的降级行为，而不能假装已经精排。

### 5. WEB_FETCH（可选）

如果启用网页搜索，搜索结果可能只有摘要或链接。Web Fetch 会获取部分页面正文，替换过短的 snippet，使后续模型获得更完整证据。

### 6. CHUNK_MERGE

多个结果可能来自相邻 Chunk、重叠片段、父子片段或不同检索器。合并阶段需要：

- 去重；
- 合并相邻/重叠内容；
- 解析父 Chunk；
- 维护引用来源；
- 控制合并后的文本大小。

入口位于 [`merge.go`](../../internal/application/service/chat_pipeline/merge.go)，相关逻辑拆分在同目录多个 `merge_*.go` 文件中。

### 7. FILTER_TOP_K

即使 Rerank 后仍可能有过多内容。最终过滤阶段结合 Top-K、阈值与预算选择交给模型的证据。

它解决的是“模型真正能看到什么”，而不是“搜索引擎曾经找到什么”。实现见 [`filter_top_k.go`](../../internal/application/service/chat_pipeline/filter_top_k.go)。

### 8. INTO_CHAT_MESSAGE

这一阶段把问题、历史、检索结果、附件、图片信息、引用元数据和 Prompt 组合成模型消息。

实现见 [`into_chat_message.go`](../../internal/application/service/chat_pipeline/into_chat_message.go)。这是“检索系统”与“生成模型”的边界，也是调试答案错误时必须观察的位置。

### 9. CHAT_COMPLETION_STREAM

最后通过统一 `Chat` 模型接口发起流式生成，将增量答案转换为应用事件，并处理完成、错误、Token 和消息状态。

实现位于 [`chat_completion_stream.go`](../../internal/application/service/chat_pipeline/chat_completion_stream.go)。

## 不经过 LLM 的知识搜索为什么重要

WeKnora 还提供纯知识搜索接口：

```text
POST /api/v1/knowledge-search
```

Handler 的 [`SearchKnowledge`](../../internal/handler/session/qa.go#L690) 明确说明“不使用 LLM 总结”。Service 只运行：

```text
CHUNK_SEARCH → CHUNK_RERANK → CHUNK_MERGE → FILTER_TOP_K
```

这条接口在学习和排障中非常重要。它把“检索是否正确”与“模型是否正确使用证据”分离开：

- 纯搜索已经没有正确片段：问题在入库或检索；
- 纯搜索正确，但回答错误：重点检查上下文构造、Prompt 或 Chat 模型；
- 搜索与回答都正确，但引用错误：重点检查元数据和流事件映射。

这种分阶段诊断比反复调整 Prompt 更可靠。

## SSE 事件比文本增量更丰富

前端收到的不只是 `"你"`、`"好"` 这样的文本块。应用还可能发送：

- 阶段开始和完成；
- 查询理解结果；
- 检索进度；
- 引用列表；
- 思考内容；
- 工具调用（Agent 模式）；
- 错误；
- 最终完成信号。

前端的 [`useChatStreamHandler`](../../frontend/src/composables/useChatStreamHandler.ts) 将这些事件重建成聊天 UI 中的消息、步骤和引用状态。

因此 SSE 在这里是应用级事件协议，而不只是模型文本的传输格式。

## 中止与后台生命周期

用户关闭页面或点击停止，不等于所有后台工作自动正确结束。系统要协调：

- 浏览器连接取消；
- 供应商模型流取消；
- EventBus 停止；
- 会话消息状态收尾；
- 已生成内容是否保留；
- 异步标题生成是否继续；
- 重连时是否允许继续接收活跃流。

Handler 在建立流时会设置 stop watcher 和 stream handler。相关代码集中在 [`internal/handler/session/qa.go`](../../internal/handler/session/qa.go) 与 [`stream.go`](../../internal/handler/session/stream.go)。

这是典型生产复杂度：能流式显示只是第一步，取消、断线和一致性才决定体验是否可靠。

## 调试 RAG 的正确顺序

遇到错误答案时，建议从前到后检查：

1. **范围**：实际搜索了哪些知识库、文件和标签？
2. **查询**：改写后的搜索问题是什么？
3. **粗召回**：向量和关键词分别召回了什么？
4. **精排**：正确片段是否被 Rerank 降低或阈值过滤？
5. **合并**：父子或相邻片段是否恢复了必要上下文？
6. **最终上下文**：真正进入模型的文本是什么？
7. **生成**：模型是否严格依据证据？
8. **展示**：SSE 和前端是否正确合并事件与引用？

这也是管线化和 Trace 的价值：每一步都应能被观察，而不是只留下最终回答。

## 源码阅读顺序

1. [`frontend/src/api/chat/streame.ts`](../../frontend/src/api/chat/streame.ts#L39)
2. [`internal/router/routes_chat.go`](../../internal/router/routes_chat.go)
3. [`internal/handler/session/qa.go`](../../internal/handler/session/qa.go#L791)
4. [`internal/application/service/session_knowledge_qa.go`](../../internal/application/service/session_knowledge_qa.go#L23)
5. [`internal/application/service/chat_pipeline`](../../internal/application/service/chat_pipeline)
6. [`internal/application/service/knowledgebase_search.go`](../../internal/application/service/knowledgebase_search.go#L87)
7. [`internal/models/chat/chat.go`](../../internal/models/chat/chat.go#L87)

第一遍沿上述顺序只找输入、输出和阶段状态。第二遍再进入某个具体插件。

## 设计取舍

### 事件管线还是固定函数调用

固定调用链更直接；事件管线增加抽象，却便于动态阶段、插件替换、阶段 Trace 和测试。WeKnora 功能组合多，因此选择显式管线。

### 先流式返回还是等待全部完成

流式显著改善感知延迟，并允许展示检索进度；代价是前后端都要处理增量、完成、错误、取消和恢复。

### 查询改写是否默认开启

改写能解决多轮省略和口语问题，也可能改变用户意图。可靠系统需要记录原始问题与改写问题，并用评测决定策略。

## 本章小结

- 在线问答先确定安全检索范围，再执行相关性搜索。
- 普通聊天与 RAG 共享模型接口，但走不同管线。
- WeKnora 将查询理解、搜索、Rerank、合并、过滤、消息构造和生成显式拆成阶段。
- 纯知识搜索接口可以隔离检索质量与生成质量。
- SSE 承载的是应用事件协议，不只是文本字符流。
- 正确的 RAG 调试应逐阶段检查，而不是直接修改 Prompt。

### 可选实践

选择一次实际问答，在浏览器 Network 中查看 SSE 事件顺序，再与本章的管线阶段逐项对应。只观察，不要求修改代码。

下一章：[Chunking：知识边界决定检索上限](04-chunking.md)
