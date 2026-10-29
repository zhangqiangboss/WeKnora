# 第 25 章　可观测性：Trace、Token、延迟与成本

[返回全书目录](../README.md) · [上一篇：多模型适配与依赖倒置](03-multi-model-adaptation-dependency-inversion.md) · [下一篇：多租户、RBAC 与 AI 特有的安全边界](05-multi-tenant-rbac-ai-security-boundary.md)

## 本章要解决的问题

传统 Web 服务出现问题时，我们经常先看：

- HTTP 状态码；
- 接口耗时；
- 数据库慢查询；
- 异常堆栈；
- CPU、内存和连接池。

AI 应用还多出了一组难题：

- 答案慢，是检索慢、Rerank 慢、模型首 Token 慢，还是输出太长？
- 一次用户提问为什么调用了五次模型？
- Agent 卡住，是模型重复思考，还是 Tool 一直超时？
- Token 突然翻倍，是历史太长、证据太多，还是缓存没有命中？
- 一份文档入库失败，究竟失败在解析、Embedding、VLM 还是后处理？
- 成本上涨来自在线聊天，还是后台 Summary、Question、Graph 任务？
- Trace 中记录的 Token 是供应商真实计量，还是本地估算？
- 打开观测以后，会不会把 Prompt、知识正文和密钥发送到第三方？

这一章沿 WeKnora 的实现，把日志、Trace、Span、Generation、Token Usage、延迟和成本放进一套统一诊断框架。

## 可观测性不是“多打印一些日志”

日志回答的是一个个局部事件：

```text
search started
rerank returned 8 results
chat completed
```

但排查一次 AI 请求时，我们还需要知道：

1. 这些事件是否属于同一次用户操作；
2. 它们的父子和先后关系；
3. 每一步花了多长时间；
4. 哪一步调用了哪个模型；
5. 输入、输出和 Token 用量；
6. 异步 Worker 是否延续了原请求；
7. 错误是业务降级，还是整条调用失败。

因此，AI 可观测性至少包含四种不同信号：

| 信号 | 主要回答 |
|---|---|
| Log | 某个时间点发生了什么 |
| Metric | 某类事件总体有多少、趋势怎样 |
| Trace | 一次请求经过了哪些步骤，父子关系是什么 |
| Business State | 用户对象当前到底处于什么状态 |

只保留 Log，会失去调用树；只保留 Trace，会缺少大盘趋势；只看 HTTP 200，又可能看不到后台任务最终失败。

## 先建立四层观测模型

WeKnora 中可以把观测面分成四层。

### 第一层：HTTP 请求日志

[`internal/middleware/logger.go`](../../internal/middleware/logger.go) 中的 `RequestID` Middleware 会：

- 接收或生成 `X-Request-ID`；
- 把 Request ID 同时写入 Gin Context 和标准 `context.Context`；
- 给请求 Logger 增加 `request_id` 字段；
- 把 Request ID 返回给客户端。

`Logger` Middleware 记录：

- Method 与 Path；
- Status Code；
- Response Size；
- 总 Latency；
- Client IP；
- 有限长度的请求和响应正文。

敏感字段会被替换成 `***`，查询参数中的 OAuth Code、Token、State 也会被清理；SSE 响应正文不会被完整缓存。

这一层适合回答：“哪个 HTTP 请求失败了？”

### 第二层：分布式 Trace

[`internal/tracing/langfuse`](../../internal/tracing/langfuse) 基于 OpenTelemetry SDK，把 Trace 发送到 Langfuse 或 LiteFuse。

它使用三种观察对象：

```text
Trace
  └─ Span
       └─ Generation
```

- `Trace`：一次顶层请求或独立后台任务；
- `Span`：检索阶段、Agent Round、Tool Call、Asynq Handler 等逻辑步骤；
- `Generation`：一次 Chat、Embedding、Rerank、VLM 或 ASR 模型调用。

这一层适合回答：“这次请求为什么慢、为什么调用了这些模型？”

### 第三层：持久业务处理 Span

[`internal/types/knowledge_span.go`](../../internal/types/knowledge_span.go) 定义了 `KnowledgeProcessingSpan`，把一份 Knowledge 的处理过程持久化成：

```text
root
  ├─ docreader
  ├─ chunking
  ├─ embedding
  ├─ multimodal
  └─ postprocess
```

每个阶段具有：

- `pending/running/done/failed/skipped/cancelled` 状态；
- 输入、输出和 Metadata；
- 错误码、公开错误信息与内部 Error Detail；
- StartedAt、FinishedAt、DurationMs；
- Attempt 与父子 Span ID。

这不是 OpenTelemetry Trace 的简单复制。它服务于产品页面和任务恢复，即使外部 Trace 被采样或过期，用户仍能看到文档处理到了哪一步。

### 第四层：审计日志与业务终态

Trace 可以告诉你“代码怎么走”，但不能替代：

- Knowledge 的 `processing/finalizing/completed/failed`；
- Task 的 queued/retry/archived；
- Audit Log 中的 accepted/success/failed/partial/canceled；
- Dead Letter 中保留的失败证据。

例如 HTTP 上传成功只表示请求被接受，不能证明文档最终完成了 Embedding。工程诊断必须把 Trace 与业务状态对齐。

## 一次在线请求怎样形成 Trace 树

[`internal/tracing/langfuse/middleware.go`](../../internal/tracing/langfuse/middleware.go) 的 Gin Middleware 位于认证之后，因此能够取得 User、Tenant 和 Session 信息。

对被选中的路径，它创建 Root Trace：

```text
POST /api/v1/agent-chat/:session_id
  └─ agent.execute
      ├─ agent.round.1
      │   ├─ chat.completion.stream
      │   └─ agent.tool.knowledge_search
      │       ├─ embedding.embed
      │       └─ rerank
      └─ agent.round.2
          └─ chat.completion.stream
```

Root Trace 通常携带：

- HTTP Method 与路由模板；
- Request ID；
- User ID；
- Session ID；
- Release 与 Environment；
- HTTP Status 和响应大小。

注意它使用 `c.FullPath()` 这样的路由模板，而不是把每个具体 UUID 都放进 Trace Name。否则高基数 ID 会把统计维度炸碎。

### 手工推演：回答变慢时先看哪一个 Span

某次请求总耗时 5.9 秒，首字节（TTFT）为 5.4 秒。Trace 展开后是：Query Rewrite 0.8 秒、Search 0.2 秒、Rerank 3.2 秒、构造 Prompt 0.1 秒、Chat 首字节 1.1 秒、后续输出 0.5 秒。此时“换一个更快的 Chat Model”最多影响 1.6 秒的模型阶段，解决不了主要的 3.2 秒 Rerank。

```text
request 5.9s
├─ rewrite 0.8s
├─ search  0.2s
├─ rerank  3.2s  <- 首要排查：候选数、网络、模型并发
└─ chat    1.6s (TTFT 1.1s + generation 0.5s)
```

同一条 Trace 的 Token 字段还要带来源标签：供应商返回的 `usage` 是实测，本地 tokenizer 是估算，按字符数折算则更粗。把三者直接相加或横向比较，会制造看似精确的成本结论；聚合报表至少应保留 `usage_source`。

## 不是所有 HTTP 路由都应该 Trace

`shouldTrace` 只选择可能触发模型或异步 AI 工作的路由，例如：

- Knowledge Chat、Agent Chat 与 Knowledge Search；
- Model 连通性与抽取测试；
- Evaluation；
- 文档、FAQ 和 Chunk 的处理入口；
- Wiki Auto-fix；
- 手工触发的数据源同步。

列表查询、静态资源、健康检查和轮询接口被排除。

这是一个非常实用的原则：

> Trace 的目标是保留因果链，不是复制全部 Access Log。

如果什么都 Trace，真正昂贵的模型调用会淹没在大量无关 GET 请求中。

但白名单也带来维护责任：新增一条会调用模型的路由时，若忘记更新 `shouldTrace`，模型 Generation 会自动创建浅层 Trace，失去完整 HTTP 上下文。

## 跨进程 Trace 怎样延续

HTTP 请求结束后，Asynq Worker 可能几分钟后才执行。普通 `context.Context` 无法跨 Redis 保存，因此必须传播稳定的 Trace Context。

WeKnora 的流程是：

```text
HTTP OTel Span Context
  → W3C traceparent
  → Task Payload.TracingContext
  → Redis / Asynq
  → Worker Extract
  → asynq.<task_type> Span
  → Model Generation
```

入队端调用 [`InjectTracing`](../../internal/tracing/langfuse/asynq.go)，把 W3C `traceparent`、User ID 和 Session ID 写入 Task Payload。

Worker 端的 `AsynqMiddleware`：

1. 从 Payload 提取 Trace Context；
2. 恢复远程父 Span；
3. 创建 `asynq.<task_type>` Span；
4. 记录 Task ID、Queue、Retry、MaxRetry 和 Payload Bytes；
5. 让 Handler 内的模型 Generation 自动成为子节点。

若定时任务没有上游 Trace，Worker 会创建一条独立 Trace，而不是产生孤儿 Span。

### Root 已经结束，子 Span 还能继续吗

可以。分布式 Trace 依靠 Trace ID 和 Parent Span ID 建立关系，不要求父进程一直存活。HTTP Root 可以先结束并导出，之后 Worker 仍可用同一 Trace ID 上报子 Span。

但这要求观测后端允许延迟到达，并且保留窗口足够长。

## Span 与 Generation 的职责不同

下面两者很容易混淆：

```text
Span: chat_pipeline.rerank
Generation: rerank
```

Generation 主要记录供应商调用：

- Model Name；
- 输入与输出；
- Model Parameters；
- Token 或其他 Usage；
- Provider 错误；
- 开始、结束和首 Token 时间。

业务 Span 记录应用决策：

- 候选数量；
- Threshold；
- 是否降级阈值；
- Rerank 后保留多少结果；
- 是否进入 Fallback；
- 整个阶段耗时。

如果只有 Generation，你能知道模型返回了什么，却不知道应用为何丢掉这些结果；如果只有业务 Span，又看不到真正的模型 Usage 和 Provider Error。

## Agent 为什么特别需要调用树

普通 RAG 一般只调用一到两次 Chat。Agent 每轮都可能重复：

```text
Think → Tool → Observe → Think
```

[`internal/agent/engine.go`](../../internal/agent/engine.go) 会为：

- 整次 Agent Execute；
- 每一轮 `agent.round.N`；
- 每个 Tool Call；

建立 Span。

Round Span 记录消息数、Tool 数、轮次、Token、Tool Call 数、Outcome 与 Duration。这样可以区分：

- 一次模型本身很慢；
- Tool 很慢；
- 单轮很快但循环次数太多；
- 同一回答反复出现，触发 stuck-loop 终止；
- Context Window 管理导致历史被压缩。

只看一次请求总耗时，无法做出这些判断。

## 延迟必须拆开看

“接口用了 12 秒”这个数字信息量很低。AI 请求至少要拆成：

| 指标 | 含义 |
|---|---|
| Queue Wait | 任务从入队到 Worker 开始执行 |
| Retrieval Latency | Query Rewrite、Embedding、检索、融合耗时 |
| Model Connect Latency | 发出请求到收到上游响应头 |
| TTFT | 调用开始到第一个有效 Token |
| Generation Duration | 模型完整输出耗时 |
| Stream Delivery | 服务端收到 Token 到客户端消费完成 |
| End-to-End Latency | 用户发送到完整结果可用 |

### TTFT 不等于总耗时

对流式 Chat，[`internal/models/chat/langfuse_wrapper.go`](../../internal/models/chat/langfuse_wrapper.go) 在收到第一条非空 `thinking` 或 `answer` Chunk 时调用 `MarkCompletionStart`。

所以：

```text
TTFT = first meaningful text chunk time - generation start time
```

它能揭示模型排队、网络连接和 Prompt Prefill 的等待，但不包含后续完整生成时长。

还要注意：如果模型先输出 Tool Call 而没有文字，这个实现不会把 Tool Call 当作“首 Token”。这里的 TTFT 更接近“首个用户可见文本延迟”。

### Asynq Span 不等于 Queue Wait

当前 Worker Span 从 Handler 开始执行时创建，能够测量执行时长，却不能直接得出 Task 在 Redis 中等待了多久。

Queue Wait 需要额外记录 `enqueued_at`，或从 Asynq Inspector/Queue 指标计算。不要把 Handler Duration 误当作端到端后台延迟。

## Token Usage 有三种可信度

### 第一类：供应商真实返回

Chat 通常返回：

```text
PromptTokens
CompletionTokens
TotalTokens
```

WeKnora 还归一化：

- Cache Read Tokens；
- Cache Write Tokens；
- Cache Miss Tokens；
- Cache Status。

这是最接近账单口径的数据，但仍应以供应商最终账单为财务结算依据。

### 第二类：本地 Tokenizer 估算

Agent 在 Context Window 管理中使用 Token Estimator。它用于：

- 判断是否需要压缩历史；
- 限制 Tool Result；
- 为新追加消息估计增量。

这种估算服务于容量控制，不一定与某个供应商的精确计费 Tokenizer 相同。

### 第三类：字符比例估算

某些模型接口不返回 Usage。当前 Langfuse Wrapper 会近似计算：

| 能力 | 当前策略 |
|---|---|
| Embedding | 文本 rune 数约除以 4，再加少量边界值 |
| Rerank | Query 与所有 Document rune 数约除以 4 |
| VLM | Prompt 与结果按 rune 数估算 |
| ASR | 不用 Token，以最后一个 Segment 的结束时间估算秒数 |

这些值适合趋势和相对比较，不适合精确结算。

## Cache Token 为什么要单独观察

只看 Prompt Tokens，可能会得出错误结论：

```text
Prompt Tokens 很大 → 成本一定很高
```

实际上，同样的 Prompt Tokens 中可能有很大部分命中了 Provider Prompt Cache。缓存读、缓存写和普通输入往往具有不同单价。

WeKnora 通过 `call_purpose` 区分调用用途，例如 Agent Round 或 Wiki 修改；通过 `prompt_prefix_fingerprint` 记录不可逆短哈希，用来观察同一稳定前缀的缓存效果。

这里有一个重要边界：

> 前缀指纹不泄露原始前缀，但 Generation Input 本身仍可能记录完整 Prompt。

指纹安全不等于整条 Trace 已经脱敏。

## 成本不是 Token 乘一个固定单价

更接近现实的估算是：

```text
Chat Cost
  = uncached_input × input_price
  + cache_read × cache_read_price
  + cache_write × cache_write_price
  + output × output_price

Embedding Cost
  = input_tokens × embedding_price

ASR Cost
  = audio_seconds × price_per_second
```

还可能存在：

- 按图片或像素计费；
- 按 Rerank Document 数计费；
- Batch 折扣；
- Region 与服务等级差异；
- 免费额度；
- Provider 返回 Usage 与最终账单修正。

当前 WeKnora 把 Usage 发送到 Langfuse，由 Langfuse 的 Model Pricing 配置计算金额。代码中没有一套完整的租户账本、实时预算扣减或硬性 Cost Gate。

因此当前能力更接近：

```text
成本观察与归因
```

而不是：

```text
财务级结算与预算阻断
```

## 一次请求的成本应该怎样归因

最有用的成本维度通常不是“今天用了多少 Token”，而是：

- Tenant；
- User 或 Principal；
- Session；
- Model 与 Provider；
- `call_purpose`；
- 在线请求或后台任务；
- Knowledge Base；
- Agent；
- 成功、降级或失败；
- Cache Hit/Miss。

例如后台自动 Question Generation 消耗上涨，优化方向可能是减少 fan-out；Agent Round 变多，则应检查 Tool Schema、停止条件和 Prompt，而不是直接换便宜模型。

## 观测本身也需要可靠性设计

Langfuse 集成是可选的。关闭时，公开入口退化为 No-op，模型 Factory 也不会安装对应 Wrapper。

启用后使用 OpenTelemetry Batch Span Processor：

- 达到 `FlushAt` 批量发送；
- 超过 `FlushInterval` 定时刷新；
- `QueueSize` 限制内存缓冲；
- `RequestTimeout` 限制一次上报；
- `SampleRate` 控制采样；
- Shutdown 时刷新剩余 Span。

观测系统的基本原则是：

> 观测失败不应让用户的问答和后台任务失败。

因此队列必须有界，上报错误不能覆盖业务错误。但代价是高峰期可能丢 Trace，所以“控制台没看到”不等于“业务没有执行”。

## Sampling 会改变你能回答的问题

低采样率适合：

- 总体延迟分布；
- 常见调用链；
- Token 与成本趋势。

但不适合：

- 追查某个特定用户的一次偶发失败；
- 完整审计；
- 精确成本账本；
- 统计极少发生的安全事件。

当前 Manager 使用 Parent-based Ratio Sampling。上游已采样的 Trace 会保持整条链一致，避免 HTTP Root 被采样而 Worker 子 Span被随机丢弃。

真正需要更强排障能力时，可以演进为：

- 错误 Trace 强制保留；
- 慢 Trace Tail Sampling；
- 重要租户不同采样率；
- 普通请求低采样、Evaluation 全采样。

## 数据最小化比 Trace 完整更重要

模型 Trace 天然可能包含：

- 用户问题；
- 对话历史；
- 检索证据；
- Tool 参数与结果；
- 模型回答；
- 文件正文或摘要；
- User ID、Tenant ID 与 Session ID。

WeKnora 已做了一些限制：

- Asynq Payload 只预览约 1 KB；
- Batch Embedding 只预览有限条文本；
- VLM 不上传原始图片字节；
- Embedding 输出只记录维度和少量向量预览；
- Rerank 输出限制数量；
- Access Log 对常见密钥字段脱敏。

但启用外部 Langfuse 仍然意味着 Prompt 与 Response 可能离开 WeKnora 数据平面。生产环境必须明确：

1. 数据是否允许发送到该观测后端；
2. 是否需要自建 Langfuse；
3. 保留期多长；
4. 哪些租户可以启用；
5. 是否应进一步 Redact 或 Hash；
6. 谁能查看 Trace。

“它只是日志”不是合规豁免理由。

## 三种 Span 不要混用

WeKnora 中至少存在三种常被叫作 Span 的对象：

| 名称 | 存储 | 目的 |
|---|---|---|
| OTel Span | Langfuse/LiteFuse | 分布式调用链与性能分析 |
| KnowledgeProcessingSpan | 业务数据库 | 文档处理时间线、错误与恢复 |
| 文本 Span/Offset | 文档内容结构 | 字符或版面位置 |

本章前两类都属于观测，但生命周期不同。不要为了“统一名词”强行把业务状态只存在外部 Trace 中。

## 诊断案例：回答为什么慢

可以按下面顺序排查：

1. 用 Request ID 找到 Access Log，确认端到端耗时和 HTTP 状态；
2. 打开同一次 Session 的 Trace；
3. 看检索、Rerank、Chat 各自 Span Duration；
4. 比较 Chat TTFT 与 Generation 总时长；
5. 检查 Prompt Tokens、Output Tokens 和 Cache Read；
6. 若是 Agent，检查 Round 数与最慢 Tool；
7. 若有异步任务，检查 Queue Wait、Retry 和 Worker Span；
8. 回到业务状态确认结果是否真正完成。

常见判断：

| 现象 | 更可能的原因 |
|---|---|
| TTFT 高、Output 短 | Provider 排队、网络、Prompt Prefill |
| TTFT 低、总时长高 | 输出太长或模型生成速率低 |
| Chat 快、总请求慢 | 检索、Rerank、Tool 或客户端消费慢 |
| 单轮快、Agent 总体慢 | Round 太多或重复调用 Tool |
| Handler 快、用户等待久 | Queue Wait 或前端轮询间隔 |
| Token 高但成本未同比增长 | Prompt Cache 命中或价格配置差异 |

## 诊断案例：成本为什么突然上涨

先把金额拆成 Usage，再把 Usage 拆成调用：

```text
总成本上涨
  ├─ 请求数上涨？
  ├─ 单请求调用次数上涨？
  ├─ 单调用输入 Token 上涨？
  ├─ 输出 Token 上涨？
  ├─ Cache Hit 下降？
  ├─ 更贵模型占比上涨？
  └─ 后台 fan-out 数上涨？
```

然后按 `call_purpose`、Model、Tenant、在线/后台和成功状态分组。

失败调用也可能产生账单。只统计成功响应会低估真实成本。

## 指标、Trace 与评测的关系

三个概念不能互相替代：

- 可观测性告诉你系统实际上怎么运行；
- 评测告诉你答案质量是否达到目标；
- 业务指标告诉你用户是否完成任务。

一个模型可以更快、更便宜，却让 Recall 或 Groundedness 下降；也可以离线评测很好，但生产中因 Provider 限流导致大量超时。

成熟的 AI Release 应同时比较：

```text
质量 + 延迟 + Token + 成本 + 失败率
```

## 与 Java / Spring 的对应关系

| WeKnora / Go | Java / Spring 常见对应 |
|---|---|
| `context.Context` 传播 Trace | OTel Context / Reactor Context |
| Request ID Logger Field | SLF4J MDC |
| Gin Middleware | Servlet Filter / HandlerInterceptor |
| Model Wrapper | Decorator / Spring AOP |
| OTel Span | OpenTelemetry Span |
| Asynq Trace Carrier | Kafka/Rabbit Header Propagation |
| BatchSpanProcessor | OTel BatchSpanProcessor |
| KnowledgeProcessingSpan | 业务 Job Step 表 / Spring Batch Execution |
| Langfuse Generation | LLM Observation / 自定义 Micrometer Observation |

Java 开发者最容易犯的错误，是只给 Controller 和 Repository 加 AOP，却漏掉异步消息与模型 Stream。真正的端到端 Trace 必须让 Header/Carrier 穿过线程池、消息队列和 Provider Wrapper。

## 当前实现值得学习的设计

### 观测是 Decorator，不污染业务接口

Chat、Embedding、Rerank、VLM 和 ASR 都通过 Wrapper 增加 Generation，业务调用方无需知道 Langfuse。

### 在线与异步使用统一 W3C Context

`traceparent` 是跨服务标准，而不是 Langfuse 私有 UUID。这样未来可以接入其他 OTel 后端。

### 业务 Span 与技术 Trace 并存

外部 Trace 服务于诊断，数据库 Span 服务于产品状态和恢复，避免让外部观测系统成为业务事实源。

### Usage 明确标注估算边界

不同模型能力没有伪装成同一种精确 Token 口径，ASR 甚至使用 Seconds 作为单位。

### 观测失败不阻塞业务

No-op、Batch、有限 Queue 与上报 Timeout 让观测保持旁路性质。

## 当前边界与可演进方向

### 统一 Metrics 与 SLO

当前源码主要具备结构化日志、Trace 与业务状态，尚未形成完整的 Prometheus/Micrometer 式指标面。可以补充：

- 请求 P50/P95/P99；
- TTFT 与 Generation Duration；
- Queue Wait 与 Backlog Age；
- 各 Provider 429/5xx/Timeout；
- Token/请求与 Cost/请求；
- Cache Hit Ratio；
- Agent Round/请求；
- Dead Letter Rate。

### 建立成本账本

若要做租户配额与预算阻断，需要持久化：

- Usage Event ID；
- Provider Request ID；
- Tenant/Principal/Model；
- 真实或估算标记；
- 价格版本与币种；
- Retry 与重复上报幂等键；
- 最终账单校准。

### 增加 Queue Wait

在 Payload 或任务元数据中记录 EnqueuedAt，Worker 开始时计算 Queue Delay，与 Handler Duration 分离。

### 更细的数据治理

为 Trace 增加字段级 Redaction、租户级开关、独立保留期、敏感 Tool 输出默认不采集和管理员访问审计。

### 错误与慢请求优先采样

使用 Tail Sampling 保留失败和高延迟 Trace，普通成功请求按比例采样。

## 推荐的源码阅读顺序

1. [`internal/middleware/logger.go`](../../internal/middleware/logger.go)：先理解 Request ID、Latency 与日志脱敏。
2. [`internal/tracing/langfuse/config.go`](../../internal/tracing/langfuse/config.go) 和 [`manager.go`](../../internal/tracing/langfuse/manager.go)：看启用、采样、批量上报与 No-op。
3. [`internal/tracing/langfuse/middleware.go`](../../internal/tracing/langfuse/middleware.go)：看 Root Trace 与路由白名单。
4. [`internal/tracing/langfuse/tracer.go`](../../internal/tracing/langfuse/tracer.go)：看 Trace、Span、Generation 的父子关系和 Finish。
5. [`internal/tracing/langfuse/asynq.go`](../../internal/tracing/langfuse/asynq.go)：理解 `traceparent` 怎样穿过 Queue。
6. [`internal/models/chat/langfuse_wrapper.go`](../../internal/models/chat/langfuse_wrapper.go)：跟踪流式 TTFT、完整输出和 Token Usage。
7. 再对比 [`internal/models/embedding/langfuse_wrapper.go`](../../internal/models/embedding/langfuse_wrapper.go)、[`internal/models/rerank/langfuse_wrapper.go`](../../internal/models/rerank/langfuse_wrapper.go)、[`internal/models/vlm/langfuse_wrapper.go`](../../internal/models/vlm/langfuse_wrapper.go) 与 [`internal/models/asr/langfuse_wrapper.go`](../../internal/models/asr/langfuse_wrapper.go) 的估算策略。
8. [`internal/agent/engine.go`](../../internal/agent/engine.go)：搜索 `agent.round`，看 Agent 调用树。
9. [`internal/types/knowledge_span.go`](../../internal/types/knowledge_span.go) 与 [`internal/application/service/knowledge_span_tracker.go`](../../internal/application/service/knowledge_span_tracker.go)：区分持久业务 Span。
10. 最后读 [`docs/Langfuse集成.md`](../../docs/Langfuse集成.md)，把配置和运维建议补齐。

## 常见误区

### 误区一：HTTP 200 就代表 AI 任务成功

200 可能只表示入队成功；必须继续看 Worker、业务状态和 Dead Letter。

### 误区二：总接口耗时就是模型耗时

它还包含检索、Rerank、Tool、Queue、序列化和客户端消费。

### 误区三：所有 Token 都是供应商精确返回

Embedding、Rerank 与 VLM 常是字符估算；本地 Tokenizer 也可能和计费 Tokenizer 不同。

### 误区四：Token 相同，成本就相同

输入、输出、缓存读写、模型、Region 和服务等级价格都可能不同。

### 误区五：Trace 能替代业务状态表

Trace 可以采样、丢失和过期，不能作为 Knowledge 或 Task 的事实源。

### 误区六：Trace 越完整越好

完整 Prompt、证据和 Tool 输出可能包含敏感数据。先定义数据分类和保留策略。

### 误区七：上报失败应该让业务失败

观测应旁路运行。反过来，也不能因为上报是旁路就不监控观测丢失率。

### 误区八：一个 Trace ID 足以定位一切

还需要 Request ID、Session ID、Task ID、Knowledge ID、Tenant ID 与 Provider Request ID 等不同身份。

## 本章小结

- AI 可观测性需要 Log、Metric、Trace 和业务状态共同回答问题。
- WeKnora 用 Request ID 串联 Access Log，用 OTel Trace/Span/Generation 表达调用树。
- HTTP 与 Asynq 通过 W3C `traceparent` 传播，定时任务没有父级时创建独立 Trace。
- Generation 记录模型输入输出和 Usage，业务 Span 记录应用选择、降级和结果数量。
- TTFT、完整生成、Queue Wait 和端到端延迟是不同指标。
- Chat Usage 通常来自供应商；Embedding、Rerank、VLM 多为估算；ASR 以秒计量。
- Token 只是成本输入，还要考虑缓存、模型价格、调用次数和失败请求。
- 当前能力偏向成本观察与归因，不是财务级账本或硬预算 Gate。
- KnowledgeProcessingSpan 是持久业务时间线，不能被外部 Trace 替代。
- Trace 可能包含 Prompt、证据与 Tool 输出，启用外部后端前必须完成数据治理。
- 观测系统应有界、可采样、失败不阻塞业务，同时自身也要被监控。

### 可选实践

选择一次“Agent 查询知识库并调用一个 Tool”的请求，设计完整观测树。至少标出 HTTP Root、Agent Execute、两个 Round、两个 Chat Generation、一次 Tool Span、Embedding、Retrieval 和 Rerank。为每个节点填写开始时间、结束时间、TTFT、Token Usage、Tenant、Session、Model、Outcome 和错误字段。然后模拟三种异常：Provider 首 Token 变慢、Tool 超时后重试、Langfuse Queue 满导致 Trace 丢失，分别说明用户业务状态、日志、Trace 和成本数据还能保留什么。
