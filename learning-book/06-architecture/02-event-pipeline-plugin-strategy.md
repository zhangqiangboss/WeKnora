# 第 29 章　事件管线、插件和策略模式

[返回全书目录](../README.md) · [上一篇：用 Java 开发者熟悉的方式重读 Go 分层](01-java-developer-read-go-layering.md)

## 本章要解决的问题

AI 应用需要频繁改变行为：换一种检索后端、接入一个数据源、按条件启用 Rerank、把模型 token 转成浏览器事件，或者为某个租户创建 Web Search Provider。若把这些分支堆在同一个 `if provider == ...`、`switch feature == ...` 中，核心调用链很快会被供应商和功能开关淹没。

WeKnora 使用了事件、管线插件、注册表、Factory 和 Strategy 等不同机制。但它们解决的问题不同：

- **事件**处理“一次过程已经发生了什么”；
- **管线插件**处理“一次请求按什么顺序经过哪些阶段”；
- **注册表 + Factory**处理“按类型选择并创建哪个实现”；
- **Strategy**处理“同一个稳定操作如何替换算法”；
- **真正的第三方插件**还需要隔离、版本、权限和生命周期，不能只靠一个 Go interface 宣称完成。

## 先区分四种扩展点

| 模式 | 决策发生时机 | 调用方期待什么 | 典型用途 |
| --- | --- | --- | --- |
| Event Bus | 过程运行中 | 订阅者观察或处理事件 | 流式 UI、进度、Trace、监听 |
| Pipeline Plugin | 处理请求时 | 有序阶段共同完成一个用例 | Query 理解、检索、Rerank、生成 |
| Registry + Factory | 配置/类型被解析时 | 根据 key 取实现或创建实例 | 数据源、IM、Web Search、检索引擎 |
| Strategy | 一个算法调用点 | 同一接口的可替换行为 | 不同 Driver、排序/检索实现 |

选择错误会制造隐性耦合。比如让“必须在检索前执行”的 Query Rewrite 订阅一个异步 Event Bus，无法表达顺序和失败传播；把每个纯旁路日志都塞进 Pipeline，又会让主路径背负不必要的依赖。

## Event Bus：过程内的事件，不是分布式消息队列

[`internal/event/event.go`](../../internal/event/event.go) 定义 `EventType`、`Event`、`EventHandler` 与 `EventBus`。它保存本进程内的 Handler，并提供 `On`、`Off`、`Emit` 和 `EmitAndWait`。

```text
Session / Agent / Chat Pipeline
        │ Emit(EventChatStream, EventAgentToolCall, ...)
        ▼
     EventBus
     ├── HTTP/SSE 适配器 → 浏览器事件
     ├── IM 适配器       → 聊天平台消息
     └── 其他本地订阅者  → 进度或观测
```

默认 EventBus 同步、顺序调用 Handler；Handler 返回错误会使 `Emit` 失败。`NewAsyncEventBus` 则以 goroutine fire-and-forget，panic 会被恢复且错误不会回传。`EmitAndWait` 会并发执行并等待全部订阅者。三者的选择是业务语义，而不是性能微调：

- UI 流的关键适配失败是否应终止当前请求？使用同步路径时要明确；
- 旁路遥测可以异步，但必须接受事件丢失、乱序和背压不可见；

### 手工推演：同一事件为何不能同时承担两种承诺

`answer.delta` 发生后，SSE 订阅者负责把文字显示给用户，遥测订阅者负责记录耗时。若把两者都放在同步 `Emit`，一次遥测服务超时就会拖慢甚至中断回答；若都改成异步，SSE 的顺序、错误和断线也变成不可知。较合理的划分是：回答协议要求的路径有明确顺序与失败策略，旁路观测允许丢失并单独设超时。

更重要的是，这个 Event 不能顺便承担“稍后一定给文档做 Embedding”的承诺。进程崩溃后 EventBus 内没有可重放记录；必须可靠完成的长工作应写入业务状态并进入持久 Queue。事件描述已经发生的过程，队列描述仍需完成的工作，二者看似都在传消息，恢复语义却完全不同。
- 需要多个独立操作全部完成时，用等待语义并设置 Context deadline。

它不是 Kafka、Redis Stream 或 Asynq。它没有跨进程持久化、offset、消费者组、重放或崩溃恢复。第 22、23、27 章中的 Redis Stream 和 Task Queue 才负责那些跨进程的可靠性边界。

## Pipeline Plugin：用有序阶段组织一个用例

聊天不是一个 Handler 调一次模型，而是一串可能启用、跳过或失败的阶段。容器在 [`internal/container/container.go`](../../internal/container/container.go) 中集中注册 Chat Pipeline 的 Event Manager 及 Search、Rerank、Web Fetch、Merge、Data Analysis、最终消息、Chat Completion、Filter Top K、Query Understand、History、Memory Recall、Entity 等插件。

这体现了一个重要设计：**注册发生在 composition root，执行顺序与条件由管线管理器定义**。插件不是任意时刻都可插入的 callback；它应声明自己消费/产生什么数据、处于哪个阶段、失败是否中断，以及是否受配置开关控制。

一个理想的阶段模型可以这样阅读：

```text
Query / Session Context
  → 历史与记忆加载
  → 意图识别、改写、实体抽取
  → 检索 / Web Fetch
  → 合并、Rerank、Top-K 过滤
  → Prompt/上下文构造
  → Chat Completion（流式 Event）
  → 最终消息持久化
```

管线的收益是局部演进：加入一个只读的质量检查阶段，不必让 Session Handler 知道所有实现细节。但管线也不是万能抽象。阶段太多、共享可变上下文没有明确类型、或 plugin 能随意重排时，调试会退化为“这次为什么少跑了一步”。对每个阶段至少应定义输入、输出、不变量、开关、超时、可重试性和 Trace 名称。

## Registry + Factory：按类型延迟选择实现

数据源的 [`ConnectorRegistry`](../../internal/datasource/connector.go) 是最直接的例子。它按 `Connector.Type()` 登记 Connector，再由业务按配置 key 查找。Connector 的稳定契约包含连接校验、资源列举、全量与增量拉取；`StreamingConnector` 则作为可选能力，使支持者能进行边拉取边入库和 Checkpoint。

这比将 Feishu、Notion、Confluence 等判断散落在同步服务中更可维护：

```text
DataSource.type
  → ConnectorRegistry.Get(type)
  → Connector / optional StreamingConnector
  → 统一的同步、检查点、删除收敛与审计流程
```

Web Search 使用略不同的 `Registry`。[`internal/infrastructure/web_search/registry.go`](../../internal/infrastructure/web_search/registry.go) 保存的是 `ProviderFactory` 而不是 Provider 实例。原因是 Provider 往往依赖当前租户的密钥、endpoint 和参数，应在实际调用时创建；单例全局 Provider 会错误地共享租户配置。

这给出一个经验法则：

- 无状态、启动时唯一的实现，可直接注册实例；
- 依赖请求/租户配置或昂贵生命周期的实现，注册 Factory，并明确谁负责 close、缓存和隔离；
- 可选能力用小接口表达，而不是让基础接口膨胀到所有实现都要造空方法。

## Strategy 与 Adapter：替换算法，但不泄漏供应商协议

检索引擎、文件存储和模型 Provider 都符合“上层面向能力、下层适配具体协议”的结构。第 9–11、24 章已经展示：同一个 `RetrieveEngine` 或模型能力 Port 可以被 PostgreSQL、SQLite、Qdrant、Milvus 或不同模型供应商实现。

这里要区分两个词：

- **Strategy**：上层在同一抽象下选择不同算法/行为，例如按配置选择 Retriever；
- **Adapter**：下层把供应商 SDK、HTTP/GRPC、认证和错误格式翻译成该抽象。

若上层开始判断 `if qdrant`、`if openai` 并处理各自错误码，Adapter 边界已经泄漏。若 Adapter 反过来决定业务 Tenant Scope、RAG 阶段顺序或 HTTP 状态码，也同样越界。

## 注册表不是自动发现机制

WeKnora 的多处扩展都是**编译时随应用发布**的：`BuildContainer` 中显式调用注册函数，例如 IM Adapter Factory、数据源 Connector、Chat Pipeline 插件。这种方式的优点是可审计、可测试、可随主程序统一升级；缺点是新增实现需要改代码、重新编译和发布。

真正的运行时插件系统还需要回答：

1. 插件来自哪个可信来源，签名和版本如何校验？
2. 它在进程内、子进程、容器还是远程服务运行？
3. 它可读取哪些 Secret、文件、网络和租户数据？
4. 升级、禁用、崩溃和超时如何不拖垮宿主？
5. 插件 API 如何做兼容性、迁移和审计？

MCP 和 Skill 是向外部能力扩展的一种更受协议与沙箱约束的路径，但也不自动解决授权与供应链问题。不要把 Registry 误称为安全的动态插件平台。

## 失败模式与观测点

| 失败 | 通常根因 | 应记录的证据 |
| --- | --- | --- |
| 新 Connector 在 UI 可见但同步失败 | 注册了元数据却未注册实现，或凭据/能力不匹配 | connector type、注册状态、配置版本、错误分类 |
| Rerank 偶尔未执行 | 管线开关、前置结果为空、阶段超时或条件跳过 | pipeline stage、skip reason、候选数、耗时 |
| UI 显示断流但模型仍在跑 | Event Bus/HTTP 适配失败与计算生命周期未统一 | request/session/event ID、取消传播、最终状态 |
| 多租户 Web Search 串密钥 | 把 provider 实例做成全局单例 | tenant/provider identity、Factory 输入的脱敏摘要 |
| 异步订阅者拖慢系统 | 无背压的 goroutine 积压 | handler latency、并发量、丢弃/超时计数 |

每个扩展点都应拥有稳定的 type/key、配置版本、可观测标签和测试替身。没有这些，注册表只是更难搜索的 `switch`。

## 推荐的源码阅读顺序

1. [`internal/event/event.go`](../../internal/event/event.go)：理解同步、异步与等待三种事件语义。
2. [`internal/event/adapter.go`](../../internal/event/adapter.go) 和 [`internal/event/usage_example.md`](../../internal/event/usage_example.md)：观察事件如何接入不同输出端。
3. [`internal/container/container.go`](../../internal/container/container.go)：搜索 `NewPlugin`、`RegisterAdapterFactory` 与 `initConnectorRegistry`，确认集中注册点。
4. [`internal/datasource/connector.go`](../../internal/datasource/connector.go)：阅读 Connector、可选 StreamingConnector 和 Registry。
5. [`internal/infrastructure/web_search/registry.go`](../../internal/infrastructure/web_search/registry.go)：比较实例注册与 Factory 注册。
6. [`internal/im/service.go`](../../internal/im/service.go)：查看 IM `AdapterFactory` 的创建与生命周期管理。
7. 回看 [`internal/container/engine_factory.go`](../../internal/container/engine_factory.go)：将检索后端的选择放回 Strategy/Adapter 边界理解。

## 常见误区

### 误区一：Event Bus 就是可靠消息队列

进程内事件没有持久化和重放能力；跨进程、可恢复工作需要 Stream/Queue。

### 误区二：所有插件都应异步

有依赖顺序和错误传播的业务阶段需要有序管线；异步只适合允许旁路和丢失的工作。

### 误区三：注册了接口就完成插件化

接口只定义调用形状。动态插件还需要信任、隔离、生命周期、兼容性和权限模型。

### 误区四：Factory 天然比单例更好

Factory 解决按租户/请求构造，不解决缓存、关闭、密钥泄漏或连接池上限；无状态实现反而应避免重复创建。

### 误区五：插件顺序只是实现细节

Query Rewrite 在 Retrieval 之后、或 Rerank 在最终上下文截断之后，都会改变 RAG 质量和成本。顺序本身是产品语义。

## 本章小结

- Event、Pipeline、Registry/Factory 和 Strategy 是不同层次的扩展机制。
- WeKnora 的 EventBus 服务于过程内事件；它不替代 Redis Stream 或任务队列。
- Chat Pipeline 通过有序插件组织一次问答的阶段，必须显式管理顺序、条件、错误和观测。
- Connector Registry 与 Web Search Factory 分别展示了实例登记和按租户延迟创建两种模式。
- Strategy/Adapter 让供应商协议不渗入业务用例；注册点集中在 container 以便审计。
- 运行时第三方插件系统还需要信任、隔离、权限、生命周期和版本治理。

### 可选实践

设计一个“公司内部搜索”Connector：列出稳定 `type`、最小 Connector 契约、全量/增量 Cursor、删除证据、凭据边界和注册位置。再决定其进度是 Event、Pipeline Stage 还是队列任务，并说明失败是否需要阻断主流程、如何重试及 Trace 上应出现哪些字段。

下一章：[哪些设计值得学习，哪些复杂度不必照搬](03-what-to-learn-what-not-to-copy.md)
