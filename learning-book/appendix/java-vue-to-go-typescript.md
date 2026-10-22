# 附录 C　Java/Vue 到 Go/TypeScript 的概念对照

[返回全书目录](../README.md)

本附录用于降低阅读 WeKnora 源码时的语言切换成本。它描述近似职责，不表示语法或运行时完全等价。

| Java / Spring | Go / WeKnora | TypeScript / Vue | 阅读提示 |
| --- | --- | --- | --- |
| `main` + Spring Boot | `cmd/server/main.go` | Vite 入口 | Go 的可执行入口可有多个，例如 desktop |
| `@Configuration` / Bean | `internal/container.BuildContainer` + `dig.Provide` | 应用级 provider / composable 装配 | 构造函数参数就是依赖清单 |
| `@RestController` | `internal/handler` | API client / page action | Handler 处理协议，不应承载核心用例 |
| `@RequestMapping` | `internal/router/Register...Routes` | Vue Router | Route Guard 与 middleware 共同决定访问边界 |
| Servlet Filter / Interceptor | Gin middleware | Router guard / Axios interceptor | 后端仍是最终安全边界 |
| `@Service` | `internal/application/service` | composable/store 的业务编排 | Service 接收 `context.Context` 而非 `gin.Context` |
| `@Repository` / JPA DAO | `internal/application/repository` + GORM | 无直接对应 | 注意某些 Repository 查询不自动加租户条件 |
| interface + implementation | 消费方定义的小 interface + 构造函数 | TypeScript interface/type | 不必给每个 struct 创建接口 |
| `ThreadLocal` / Request Scope | `context.Context` | provide/inject 不等同请求上下文 | 异步任务需重建 Context 与授权 |
| `CompletableFuture` / MQ consumer | goroutine + Asynq/SyncTaskExecutor | Promise / Web Worker | goroutine 本身不提供持久化或重试 |
| `@ControllerAdvice` | `AppError` + ErrorHandler middleware | 全局错误处理 | 保持业务错误与基础设施错误的语义 |
| Jackson DTO | `struct` + JSON tag | interface/type + schema | Go 零值与 omitted 字段需特别留意 |
| JUnit/Mockito | `testing` + fake interface | Vitest | 依赖小接口通常比 mock 大对象简单 |

## 三个最常见的迁移误解

### “Go 没有注解，所以没有生命周期”

生命周期集中在构造、启动、`ResourceCleaner` 和 `context` 取消中。阅读 `cmd/server/main.go` 与 `internal/container/container.go`，比在全局搜索注解更有效。

### “TypeScript 类型等于运行时校验”

前端类型只在编译期存在。所有来自浏览器、模型、MCP 和外部连接器的值仍要在 Go Handler/Service 边界验证。

### “Vue 隐藏按钮就等于后端拒绝”

前端 capability 和 RBAC 状态用于体验；Router、middleware、Service Scope 和 Repository 查询才构成真实授权。

## 推荐的最短阅读路线

1. 从 [`cmd/server/main.go`](../../cmd/server/main.go) 到 [`internal/container/container.go`](../../internal/container/container.go)；
2. 选一个 [`internal/router`](../../internal/router) Route；
3. 跟进对应 Handler、Service、interface 和 Repository；
4. 最后从 [`frontend/src`](../../frontend/src) 查看页面如何调用并渲染状态。

不要同时转换所有 Go 语法。先沿一条业务链确认职责，再按需理解 `context`、interface、goroutine 和 error 返回值。
