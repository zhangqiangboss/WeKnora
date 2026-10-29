# 第 2 章　把大语言模型看成一种不确定的计算组件

[返回全书目录](../README.md) · [上一篇：AI 应用不是一个更聪明的 CRUD](01-ai-application-engineering.md) · [下一篇：Token、上下文、消息与 Prompt](03-token-context-message-prompt.md)

## 本章要解决的问题

面对聊天模型，人很容易使用人格化语言：“它知道”“它理解”“它决定”。这些说法便于交流，但不利于工程设计。

在代码中，更有用的视角是：LLM 是一个接受结构化消息和参数、返回文本或工具调用、可能流式输出、具有概率性和外部依赖特征的计算组件。

## 模型调用的基本契约

一个聊天模型调用可以抽象为：

```text
Chat(messages, options) → response
ChatStream(messages, options) → stream of chunks
```

其中 `messages` 不只是一段字符串，而是一组带角色的消息：

- `system`：系统目标、边界和行为约束；
- `user`：用户输入；
- `assistant`：模型之前的回答；
- `tool`：工具执行结果。

`options` 可能包含温度、最大输出 Token、思考模式、工具定义、工具选择策略和结构化输出格式。

WeKnora 直接用接口表达了这个契约：[`Chat`](../../internal/models/chat/chat.go#L87)。接口同时提供非流式 `Chat` 和流式 `ChatStream`。

## 它像什么，又不像什么

### 像外部微服务

LLM 调用具有典型远程依赖特征：

- 网络延迟和超时；
- 鉴权与额度；
- 限流与并发上限；
- 供应商协议差异；
- 可观测性和成本统计；
- 暂时性错误与重试。

所以你可以使用熟悉的 HTTP Client、适配器、熔断、限流和 Trace 思维。

### 不像普通微服务

普通微服务的返回结构通常由双方严格约定。模型即使被要求输出 JSON，也可能出现缺字段、额外解释、格式错误或语义不一致。

普通微服务的功能说明通常不占请求长度；模型的规则、历史和参考资料都要进入上下文并消耗 Token。

普通微服务升级应维持契约；模型版本变化可能在 API 不变时改变回答风格、工具调用倾向和推理效果。

因此，模型接口外仍需输入控制、输出校验、失败回退和评测。

## 三类容易混淆的模型能力

### Chat Model

输入消息，生成回答或工具调用。它负责语言理解和生成，是用户最直观看到的模型。

WeKnora 的抽象位于 [`internal/models/chat`](../../internal/models/chat)。

### Embedding Model

输入文本，输出一组浮点数向量。它不回答问题，而是把文本映射到一个可计算相似度的空间。

WeKnora 的核心接口是 [`Embedder`](../../internal/models/embedding/embedder.go#L16)：

- `Embed`：单条文本向量化；
- `BatchEmbed`：批量向量化；
- `GetDimensions`：获取向量维度；
- `GetModelID`：标识模型。

Embedding 既用于文档入库，也用于查询时把问题映射进同一个向量空间。

### Rerank Model

输入一个查询和一组候选文本，输出更精细的相关性排序。它通常比向量召回慢，但对较小候选集的语义判断更准确。

其角色不是“找到整个知识库中的一切”，而是“把第一阶段召回的候选重新排好”。WeKnora 的实现位于 [`internal/models/rerank`](../../internal/models/rerank)。

三者可以记成：

```text
Embedding：把内容放进可搜索空间
Retrieval：快速找到候选
Rerank：精排候选
Chat：基于最终上下文生成回答
```

## 多供应商适配：AI 时代的策略模式

WeKnora 支持本地 Ollama 和多种远程供应商。上层不应该为每个供应商写一套 RAG 流程，因此模型层采用统一接口和工厂选择。

在 [`NewChat`](../../internal/models/chat/chat.go#L142) 中，系统先按本地或远程来源选择实现；远程实现再根据 provider 选择 Anthropic 独立协议或 OpenAI 兼容协议。

这可以类比 Java 中：

```java
interface ChatModel {
    ChatResponse chat(List<Message> messages, ChatOptions options);
    Flux<StreamResponse> stream(List<Message> messages, ChatOptions options);
}
```

然后由 Factory 根据配置创建 `OllamaChatModel`、`AnthropicChatModel` 或 `OpenAICompatibleChatModel`。

### 为什么“兼容 OpenAI API”仍不等于完全相同

不同供应商即使使用相近端点，也可能在以下位置存在差异：

- 是否支持 `reasoning_content`；
- 工具调用字段格式；
- 思考模式参数；
- 流式结束信号；
- Token 参数名称；
- 多模态消息格式；
- Base URL 和鉴权头。

所以兼容协议减少了适配成本，但不能取消 provider 层。WeKnora 在 `providerAdapter` 与不同远程实现中吸收这些差异。

## 装饰器：在模型调用外叠加工程能力

创建模型实例以后，WeKnora 会在真实实现外包装调试日志、Langfuse Trace 和并发控制。例如 [`NewChat`](../../internal/models/chat/chat.go#L142) 的返回路径依次安装包装器。

可以把它想象成：

```text
业务代码
  → 并发限制包装器
    → Trace 包装器
      → Debug 包装器
        → 真实供应商客户端
```

具体包裹顺序会影响“等待限流的时间是否算入模型耗时”等观测语义。代码注释明确说明了为什么并发包装器位于最外层或特定位置。这是成熟工程中值得关注的细节：装饰器不只是语法技巧，它决定指标代表什么。

Embedding 也采用类似方式，在 [`NewEmbedder`](../../internal/models/embedding/embedder.go#L84) 附近叠加并发、调试和 Langfuse 能力。

## 流式生成不是把完整字符串切碎

流式输出的价值是降低用户感知延迟。模型可能几十秒后才完成回答，但几百毫秒或几秒内就能产生第一个片段。

一条完整流需要处理：

- 建立连接；
- 多个文本增量；
- 思考内容或工具事件；
- 引用和阶段进度；
- 完成信号；
- 服务端错误；
- 用户主动中止；
- 页面重连或恢复。

WeKnora 前端通过 [`fetchEventSource`](../../frontend/src/api/chat/streame.ts#L151) 接收 SSE，后端模型接口以 `<-chan types.StreamResponse` 暴露流式结果。二者之间还有应用级 EventBus 和消息持久化，不是简单地把供应商字节原样代理给浏览器。

这与 Java Reactor/`Flux` 的心智模型比较接近：数据在时间上逐个到达，完成、错误和取消都是协议的一部分。

## 为什么 Prompt 不等于一个字符串模板

在一个成熟 RAG 系统中，最终输入模型的内容可能由以下部分组合：

- System Prompt；
- 当前问题；
- 多轮历史；
- 检索到的知识片段；
- 每段知识的来源元数据；
- 用户引用内容；
- 临时附件；
- 图片或图片描述；
- 输出约束；
- Agent 工具定义。

因此更准确的术语是“上下文构造”或“上下文工程”。Prompt 模板只是其中一部分。

WeKnora 在 RAG 管线的 `INTO_CHAT_MESSAGE` 阶段统一组装模型消息，具体实现见 [`into_chat_message.go`](../../internal/application/service/chat_pipeline/into_chat_message.go)。

## 工程上必须假设模型会失败

调用模型时至少要考虑四类失败：

1. **传输失败**：网络、超时、限流、服务不可用。
2. **协议失败**：流中断、字段不兼容、非法响应。
3. **格式失败**：没有遵守 JSON Schema 或指定格式。
4. **语义失败**：回答流畅但错误、漏答、误用证据。

前两类可以主要靠传统可靠性工程处理；第三类需要解析、校验和重试；第四类必须依赖评测、证据约束和产品策略。

这也是 AI 工程与传统接口开发最显著的差别之一：HTTP 200 只说明调用成功，不说明业务结果正确。

### 手工推演：四种“成功”其实不是一回事

用户要求把合同提取为 JSON。供应商返回 HTTP 200，流也正常结束，但模型额外包了一层 JSON 代码围栏并漏掉 `effective_date`；这说明传输成功、协议大致成功，却格式失败。即使 JSON Schema 校验通过，若把甲方和乙方颠倒，仍是语义失败。

因此处理链应分层：网络/429 等传输错误按短暂失败重试；无法解析的格式错误先做清洗、校验或有限次数重试；语义错误保留输入、模型版本与评测样本，交给证据约束、人工复核或离线评测改进。不能用“HTTP 200”代替业务成功，也不能对已经产生外部副作用的 Tool Call 盲目重试。

## 源码观察点

1. [`internal/models/chat/chat.go`](../../internal/models/chat/chat.go#L20)：消息、工具、参数和模型接口。
2. [`internal/models/chat/remote_api.go`](../../internal/models/chat/remote_api.go)：OpenAI 兼容远程模型的调用与流处理。
3. [`internal/models/chat/provider.go`](../../internal/models/chat/provider.go)：供应商差异的适配边界。
4. [`internal/models/embedding/embedder.go`](../../internal/models/embedding/embedder.go#L16)：Embedding 接口与工厂。
5. [`internal/models/rerank/reranker.go`](../../internal/models/rerank/reranker.go)：Rerank 抽象。
6. [`frontend/src/api/chat/streame.ts`](../../frontend/src/api/chat/streame.ts#L24)：浏览器端如何管理流的开始、数据、错误、关闭和中止。

### 阅读时回答这四个问题

- 哪些字段属于所有 Chat 模型的公共契约？
- 哪些差异被限制在 provider/remote 实现内部？
- `Chat` 与 `ChatStream` 的调用者分别得到什么？
- 并发和 Trace 是修改每个供应商实现，还是通过包装器统一加入？

## 本章小结

- LLM 是概率性、远程、昂贵且可能流式返回的计算组件。
- Chat、Embedding 和 Rerank 是三类不同能力，不能混为“一个大模型”。
- 统一接口、工厂和 provider 适配器隔离多供应商差异。
- 模型调用的工程契约包括消息、参数、流、错误、取消和观测。
- Prompt 的本质是上下文构造，而不只是一段模板字符串。
- HTTP 调用成功不等于模型语义正确，效果必须单独评测。

### 可选实践

选择一个你熟悉的第三方支付或短信接口，对比它与 LLM API 在契约、重试、幂等、结果校验和监控上的异同。只写一页对比，不需要编码。

下一章：[Token、上下文、消息与 Prompt](03-token-context-message-prompt.md)
