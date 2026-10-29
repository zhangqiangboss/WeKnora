# 第 28 章　用 Java 开发者熟悉的方式重读 Go 分层

[返回全书目录](../README.md) · [上一篇：部署形态：Lite、标准版与可替换基础设施](../05-engineering/06-deployment-lite-standard-replaceable-infrastructure.md)

## 本章要解决的问题

第一次进入 Go 项目，Java 开发者很容易产生两个错觉：没有 Spring 注解就没有依赖注入；文件夹不叫 `controller/service/repository` 就没有分层。WeKnora 恰好能用来拆掉这两个错觉。

它仍然有熟悉的职责：HTTP 入口、认证与授权、用例编排、持久化、外部系统适配和启动装配。不同的是，Go 倾向把依赖关系、构造函数和接口显式写在代码里，而不是交给反射扫描。读懂这种差异后，Go 项目会从“很多包和函数”重新变成你熟悉的 Web 应用。

## 不要按目录名翻译，要按职责翻译

可以先建立一张近似对照表：

| Java/Spring 中常见概念 | WeKnora 中的主要位置 | 主要责任 |
| --- | --- | --- |
| `@RestController` | `internal/handler` | HTTP 解析、响应和错误映射 |
| `@RequestMapping` / Filter | `internal/router`、`internal/middleware` | 路由、路由组、认证、RBAC、Trace |
| `@Service` / application service | `internal/application/service` | 一个业务用例的编排、事务与跨资源协调 |
| `@Repository` | `internal/application/repository` | GORM 查询、持久化细节 |
| Port / interface | `internal/types/interfaces` | 上层依赖的能力契约 |
| Adapter / client | `internal/models`、`internal/storage`、`internal/infrastructure` 等 | 模型、对象存储、解析器、外部 API |
| `@Configuration` + Bean 装配 | `internal/container` | 显式构造并注册依赖图 |
| `main` | `cmd/server/main.go` | 启动 HTTP 服务、信号与生命周期 |

这不是严格的一一映射。比如一个 Handler 可能为 SSE、文件重写或 Agent 协议承担更多边界工作；一个 Service 也可能协调 Repository、模型、队列、审计和存储。但用“职责”而不是“包名”阅读，能避免把 Go 的组织方式误判为没有架构。

## 一次创建知识库请求如何穿过分层

以创建 Knowledge Base 为例，主链大致是：

```text
POST /api/v1/knowledge-bases
  → Router 注册 Route 与 Guard
  → Middleware 写入 Request ID、Principal、Tenant、Role 等 Context
  → KnowledgeBaseHandler.CreateKnowledgeBase
  → KnowledgeBaseService.CreateKnowledgeBase
  → KnowledgeBaseRepository / Store Resolver / Audit / Task Enqueuer
  → GORM / PostgreSQL 或 SQLite / 外部基础设施
  → Handler 将结果或 AppError 映射回 HTTP
```

[`internal/router/router.go`](../../internal/router/router.go) 创建 Gin Engine、装配全局中间件，并把 `/api/v1` 下的能力交给若干 `Register...Routes` 函数。具体知识库 Route 在 [`internal/router/routes_knowledge.go`](../../internal/router/routes_knowledge.go)。这相当于 Spring MVC 的 Mapping 层，但授权不是散落在每个 Handler 的注解里：Route Group 与 middleware 共同组成可复用的 Guard。

[`KnowledgeBaseHandler.CreateKnowledgeBase`](../../internal/handler/knowledgebase.go) 负责读取 JSON、填补请求中被省略的默认字段、做输入形状校验，然后调用 `interfaces.KnowledgeBaseService`。它应知道 HTTP 状态码和 `AppError`，但不应自己拼 GORM 查询或决定向量库绑定是否属于当前租户。

真正的业务决策在 [`NewKnowledgeBaseService`](../../internal/application/service/knowledgebase.go) 返回的 service 中：创建 UUID、从 Context 取得 Tenant/User、应用默认存储、校验向量库绑定、协调 Repository、检索引擎、任务和审计。它既不依赖 `*gin.Context`，也不应该把 HTTP JSON 作为核心输入；因此同一用例还能被后台任务、CLI 或其他入口复用。

### 手工推演：改错层会留下什么后患

假设“创建知识库时必须校验向量库属于当前 Tenant”。若把这条规则只写在 Handler，Web 请求会被挡住，但未来的导入 CLI、后台同步或测试直接调用 Service 时可以绕过它；若把 GORM 查询与 HTTP 状态码塞进 Service，复用者又被迫理解 `400/403`。

更稳定的切法是：Handler 负责把 JSON 变成用例输入、把 `AppError` 变成 HTTP；Service 依据 Context 中已认证的身份做业务与归属判断；Repository 只按给定条件存取数据。它不是教条式“每层只能调下一层”，而是保证同一条业务不变量无论从 HTTP、任务还是 CLI 进入，都在同一个位置成立。

最后，[`knowledgeBaseRepository`](../../internal/application/repository/knowledgebase.go) 通过 GORM 执行持久化。注意 `GetKnowledgeBaseByID` 的注释明确说明它不带 Tenant 过滤；带租户隔离的调用要使用 `GetKnowledgeBaseByIDAndTenant`，或由上层 Access Guard 处理共享路径。这是一个重要提醒：Repository 的接口名和 Service/Middleware 的调用链合起来才是安全边界，不能只看某一条 SQL。

## Go 的 Interface 通常放在消费者一侧

Java 项目里，接口常与实现放在同一业务包，或者先建一个抽象再生成实现。Go 的常见习惯更强调小接口和消费方依赖：谁需要什么方法，谁就拥有描述这组方法的契约。

[`internal/types/interfaces/knowledgebase.go`](../../internal/types/interfaces/knowledgebase.go) 中的 `KnowledgeBaseService`、`KnowledgeBaseRepository` 使 Handler 可以依赖“创建、查询、搜索”等能力，而不是 `knowledgeBaseService` 这个私有结构体。构造函数 `NewKnowledgeBaseService` 的返回类型也正是这个接口。

这带来三个直接好处：

1. Handler 测试可注入很小的 fake，而无需启动真实数据库；
2. Lite 与标准版可以用同一上层调用方式替换 Task、Stream 或 Retriever 实现；
3. 依赖从函数签名可见，不需要猜测类字段何时由框架反射注入。

但不要把“每个 struct 都抽 interface”当作 Go 教条。稳定边界、外部依赖、需要替身测试的协作点最值得抽象；只在一个包内使用的纯数据处理，直接使用具体类型往往更清晰。接口过大同样会让实现和测试脆弱。

## 依赖注入没有消失，只是由启动时显式装配

[`internal/container/container.go`](../../internal/container/container.go) 的 `BuildContainer` 是全局 composition root。它使用 `go.uber.org/dig` 注册配置、数据库、Redis、文件服务、Repository、Service、Handler 和 `router.NewRouter`。换成 Spring 的语言，它就是集中且显式的 `@Configuration`。

```text
BuildContainer
  → Provide(initDatabase / initRedisClient / initFileService)
  → Provide(Repository constructors)
  → Provide(Service constructors)
  → Provide(Handler constructors)
  → Provide(router.NewRouter)
  → Invoke(main 的 HTTP Server 启动函数)
```

`dig.In` 让 [`router.RouterParams`](../../internal/router/router.go) 声明自己需要的所有依赖；构造函数参数也同样是依赖清单。少了 Provider、构造函数返回的类型不匹配或出现循环，应用会在装配/启动时失败，而不是在第一条请求才隐式得到 `null`。

依赖图也应测试。[`internal/container/router_wiring_test.go`](../../internal/container/router_wiring_test.go) 用 `dig.New(dig.DryRun(true))` 分别验证 Lite（无 Redis）与 Redis 模式能否解析出 Gin Router，而不真的建立外部连接。这相当于 Spring Context 启动测试，但更聚焦于依赖图本身。

## Context 不是 ThreadLocal

Java/Spring 中，安全上下文和 Trace 常借助 `ThreadLocal`、Servlet Request Scope 或 Reactor Context 传播。Go 中请求信息通常放在 `context.Context`：HTTP 请求的 `c.Request.Context()` 会随调用显式传给 Service、Repository、模型和任务。

这使依赖可见，也更容易在异步边界暴露问题。启动 goroutine、入队或在后台重试时，不能机械地复用原请求 Context：它可能已取消，也可能携带不该长期保存的请求对象。应只提取必要的 Tenant、Principal、Trace 或任务元数据，并按新生命周期创建带超时的 Context。第 23、26 章的异步任务和授权复核，正是这种显式传播的后果。

## 错误也应跨层保持语义

Handler 不该把所有错误都转换为 500。`CreateKnowledgeBase` 会识别 `AppError`，让 middleware 按既定的 HTTP 状态和业务码渲染；只有未分类的基础设施错误才收敛为内部错误。这与 Java 的 `@ControllerAdvice` 很相似，只是错误类型和显式 `c.Error(...)` 更容易在调用点看见。

好的边界是：

- Repository 返回“未找到、唯一约束、连接失败”等底层事实；
- Service 将可预期业务条件转为领域/应用错误，并决定是否重试、审计或补偿；
- Handler 将可安全暴露的应用错误映射为协议响应；
- Middleware 负责统一格式、Request ID 与兜底恢复。

把 HTTP 状态码写进 Repository，或在 Handler 里根据字符串判断数据库错误，都会让边界倒置。

## 分层不是单向瀑布

经典图常画成 `Controller → Service → Repository`。AI 应用还存在模型、流、文件、队列和工具执行等横向能力，因此更接近下图：

```text
                HTTP / MCP / IM / CLI
                         │
                   Handler / Router
                         │
              Application Service / Pipeline
             ┌───────┬────┴─────┬────────┐
             │       │          │        │
        Repository  Model     Task     Storage/Retriever
             │       │          │        │
             └─────── Infrastructure / external systems ─┘
```

这里的“Application Service”应掌握用例顺序和业务不变量；Adapter 应掌握供应商协议；Repository 应掌握数据读写；Router/Handler 应掌握协议和认证入口。一个文件位于哪个目录不如它是否遵守这条职责边界重要。

## 推荐的源码阅读顺序

1. [`cmd/server/main.go`](../../cmd/server/main.go)：从进程入口、生命周期和 `BuildContainer` 开始。
2. [`internal/container/container.go`](../../internal/container/container.go)：只追一条依赖链，不要试图记住所有 `Provide`。
3. [`internal/router/router.go`](../../internal/router/router.go) 与 [`internal/router/routes_knowledge.go`](../../internal/router/routes_knowledge.go)：看 Route、Middleware 与 Guard 如何组合。
4. [`internal/handler/knowledgebase.go`](../../internal/handler/knowledgebase.go)：定位 `CreateKnowledgeBase`，区分协议校验和业务调用。
5. [`internal/types/interfaces/knowledgebase.go`](../../internal/types/interfaces/knowledgebase.go)：阅读消费者真正需要的服务/仓储契约。
6. [`internal/application/service/knowledgebase.go`](../../internal/application/service/knowledgebase.go)：观察用例如何协调多个 Port。
7. [`internal/application/repository/knowledgebase.go`](../../internal/application/repository/knowledgebase.go)：最后查看 GORM 和 Tenant 条件如何落地。
8. [`internal/container/router_wiring_test.go`](../../internal/container/router_wiring_test.go)：用测试回看依赖图如何在两种部署模式下被验证。

## 常见误区

### 误区一：没有注解就是没有框架边界

Go 把路由、构造函数和注册写得更显式；边界依然存在，而且通常更容易沿调用链追踪。

### 误区二：Handler 就是业务层

Handler 负责协议适配。跨存储、模型、队列、授权和审计的业务规则应留在 Service/Pipeline 或专门的访问层。

### 误区三：Repository 自动保证租户安全

无租户过滤的底层查询也有合理用途；安全来自正确的 scoped 方法、Access Guard 和调用路径的共同约束。

### 误区四：Context 可以任意传进后台 goroutine

请求取消、截止时间和身份生命周期都可能不适合后台任务。异步边界必须重新定义上下文与授权。

### 误区五：接口越多越符合依赖倒置

只为稳定协作点抽象。为每个实现预先复制一份接口，会增加认知负担而非可测试性。

## 本章小结

- WeKnora 的 Go 分层可以用 Java 熟悉的入口、应用用例、仓储、Adapter 和 composition root 来理解。
- `handler`、`router`、`middleware` 管理协议与入口；`application/service` 编排业务；`repository` 封装持久化；`container` 组装依赖图。
- Go 的构造函数与接口让依赖显式，`dig` 在启动时验证装配，而不是依赖注解扫描。
- Interface 应表达消费者所需的少量能力，而不必覆盖每个具体类型。
- Context 是显式传播的请求/任务元数据，不是可以无条件继承到后台的 ThreadLocal。
- 租户隔离、错误语义和异步生命周期都必须沿完整调用链验证。

### 可选实践

选择“创建知识库”或“发起一轮问答”中的一条调用链，画出 Route、Middleware、Handler、Service、Repository/Adapter、数据库/模型/队列各自的输入和输出。然后假设要新增一个 CLI 入口：指出哪些 Handler 逻辑不能复用、哪些 Service 接口可以直接复用，以及 CLI 如何安全地构造 Tenant、Principal、超时和错误输出。

下一章：[事件管线、插件和策略模式](02-event-pipeline-plugin-strategy.md)
