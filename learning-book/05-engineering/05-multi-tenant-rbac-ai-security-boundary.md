# 第 26 章　多租户、RBAC 与 AI 特有的安全边界

[返回全书目录](../README.md) · [上一篇：可观测性：Trace、Token、延迟与成本](04-observability-trace-token-latency-cost.md)

## 本章要解决的问题

传统后台系统通常先解决三个问题：

1. 你是谁；
2. 你有什么角色；
3. 这条数据是否属于你。

AI 应用在此基础上又增加了新的执行面：

- 模型可能生成越权的 Knowledge ID 或 Tool 参数；
- RAG 证据可能来自另一个租户；
- Agent 可以调用 MCP、Web、数据库、Shell 和 Sandbox；
- 用户可以配置 Model、Vector Store、MCP、Data Source 等外部 URL；
- Prompt、检索内容和 Tool Result 可能被发送给第三方模型；
- API Key 既代表机器，又可能代表外部终端用户；
- 共享 Agent 运行在所有者空间，但 Session 和 Sandbox 属于调用方；
- 异步 Worker 离开 HTTP 请求后仍要保留租户和发起人边界。

因此，多租户 AI 安全不能只靠一个 `tenant_id` 字段或一个 `@PreAuthorize`。

## 一次操作至少穿过七道边界

可以把授权判断写成一个交集：

```text
有效权限
  = 身份认证
  ∩ 当前租户
  ∩ Principal 类型
  ∩ Route Capability / Tenant Role
  ∩ 资源归属或共享权限
  ∩ 本次 Agent / @mention Scope
  ∩ Tool、Sandbox 与出站网络策略
```

任何一层拒绝，最终都应该拒绝。

这比简单的“角色越高权限越大”更接近真实系统：一个 Viewer 可以运行被允许的 Agent，但不能编辑 Model；一个带 `chat` Capability 的 API Key 可以发起会话，却不能管理历史或 Data Source；一个 Agent 即使能调用知识工具，也只能读取本次 SearchTargets 允许的文档。

## Authentication、Authorization 与 Scope 不同

### Authentication：你是谁

回答凭证是否有效，例如：

- Bearer JWT；
- Tenant API Key；
- Platform API Key；
- Embed Publish Token；
- 外部用户签名 Token。

### Authorization：这个主体能做什么

例如：

- Viewer 是否能读；
- Contributor 是否能编辑自己创建的 KB；
- Admin 是否能管理模型；
- API Key 是否拥有 `ingest` Capability。

### Scope：这次调用具体允许碰哪些对象

例如：

- 只能访问某三个 Knowledge Base；
- 本次 `@mention` 只选中了某份文档；
- Shared Agent 只开放配置中的 KB；
- Tool Approval 只批准了这一次调用。

认证成功只说明凭证是真的，不说明它可以操作任意对象。

## User 与 Principal 为什么要分开

[`internal/types/principal.go`](../../internal/types/principal.go) 定义了终端调用主体 `Principal`。它可能是：

- `web_user`；
- `api_tenant`；
- `api_platform`；
- `api_external_user`；
- `im_user`；
- `embed_channel`；
- `embed_session`；
- `embed_visitor`。

`UserID` 更接近 WeKnora 账号身份；`Principal` 表示本次请求真正需要隔离的终端主体。

例如一个 Tenant API Key 可能服务一万个外部用户。如果所有 Session、MCP OAuth Token 和聊天历史都只按这个 API Key 对应的系统 User 存储，一万个用户就会互相看到状态。

Principal 提供了更细的隔离键：

```text
api_external_user:<tenant_id>:<external_user_id>
embed_session:<tenant_id>:<channel_id>:<session_id>
embed_visitor:<tenant_id>:<channel_id>:<visitor_id>
```

一个关键原则是：

> 非 WeKnora 账号的 Principal 不能自动获得真人 RBAC 权利。

## Auth Middleware 怎样解析身份

[`internal/middleware/auth.go`](../../internal/middleware/auth.go) 的大致顺序是：

```text
OPTIONS / NoAuth Route
  → Bearer JWT
  → 若 JWT 无效，再尝试 X-API-Key
  → 都失败则 401
```

JWT 成功以后还没有结束，系统继续解析：

1. 目标 Tenant；
2. Tenant 是否存在；
3. 用户是否有 Active Membership；
4. 用户在该 Tenant 中的 Role；
5. 是否是跨租户系统运营主体。

目标 Tenant 的优先级是：

```text
X-Tenant-ID
  → JWT tenant claim / User home tenant
  → 第一条 active membership
```

`X-Tenant-ID` 不是“客户端想切就切”。服务端会验证用户对目标空间是否可达，非法、零值、不存在或无 Membership 都会拒绝。

## Context 是安全状态的传播载体

认证结果通过 [`internal/middleware/auth_context.go`](../../internal/middleware/auth_context.go) 同时写入：

- Gin `c.Keys`；
- `c.Request.Context()`。

其中包括：

- Tenant ID 与 Tenant Entity；
- User 与 User ID；
- Principal；
- Tenant Role；
- System Admin Flag；
- API Key Scope。

为什么要同时写两个位置？因为有的 Handler 读取 Gin Context，有的深层 Service、Repository、Model 和 Tool 只接收标准 Context。只写一边会出现危险的“不同行为路径看到不同身份”。

安全 Context 还有一个工程要求：

> 派生 Context 时只能缩小或保留授权，不能因 Clone、Detach 或异步化丢失租户边界。

## Tenant Role 是空间内的角色梯子

[`internal/types/tenant_member.go`](../../internal/types/tenant_member.go) 定义四个角色：

| Role | Level | 典型能力 |
|---|---:|---|
| Viewer | 10 | 读取、检索、运行允许的 Agent |
| Contributor | 20 | 创建资源，维护自己创建的 KB/Agent |
| Admin | 30 | 管理空间共享基础设施和他人资源 |
| Owner | 40 | Admin 能力加空间所有权与敏感生命周期操作 |

`HasPermission` 使用等级比较，高角色继承低角色能力。

当 Context 中没有有效 Role 时，`TenantRoleFromContext` 默认返回 Viewer，而不是 Admin。这是 Fail-closed 的体现。

## Role 不能替代资源归属

如果只规定 Contributor 可以修改 KB，就会产生：

```text
Contributor A 可以修改 Contributor B 的 KB
```

WeKnora 为 Knowledge Base、Custom Agent 等关键资源记录 `creator_id`，并使用：

```text
creator == current_user
OR role >= admin
```

这样的归属守卫。

[`internal/middleware/rbac.go`](../../internal/middleware/rbac.go) 中主要有：

- `RequireRole`；
- `RequireSystemAdmin`；
- `RequireOwnershipOrRole`；
- `EvaluateOwnershipOrRole`。

Role Guard 适合 Model、Vector Store、Storage、成员等租户基础设施；Ownership Guard 适合具体 KB、Agent 和子资源写操作。

### 子资源要沿归属链回溯

用户请求可能只携带 Chunk ID，授权不能只检查“这个 Chunk 存在”，而要解析：

```text
Chunk
  → Knowledge
  → Knowledge Base
  → Tenant + creator_id
```

否则容易产生 IDOR：攻击者猜到别人的 Chunk UUID，直接绕过 KB 权限。

## System Admin 与 Tenant Owner 不是同一个角色

Tenant Owner 只管理自己的空间；System Admin 管理平台范围的能力，例如全局设置或系统管理员生命周期。

两者必须是正交维度：

```text
TenantRoleOwner != IsSystemAdmin
```

`RequireSystemAdmin` 不受 Tenant RBAC 灰度开关影响。否则把租户 RBAC 切到观察模式，会意外把平台控制面开放给所有已认证用户。

跨租户访问也需要显式配置，并通常只授予目标 Tenant 的临时 Admin 能力，而不是 Owner 能力。

## API Key 不应该伪装成真人管理员

旧系统常把一个 API Key 映射成 Admin User，然后让它通过全部人类角色 Guard。这种方式很难限制“只能检索，不能删除”或“只能写某个 KB”。

当前 WeKnora 把机器授权作为独立权限体系：

```text
TenantAPIKeyScope
  ├─ ScopeType: tenant / platform
  ├─ FullAccess
  ├─ Capabilities[]
  └─ KnowledgeBaseIDs[]
```

Capability 包括 Retrieve、Chat、Ingest、Manage KB、Manage Agent、Manage Model、Manage MCP、Manage Data Source、Message History 等细分能力。

### Route Policy 默认拒绝

[`internal/middleware/api_key_gate.go`](../../internal/middleware/api_key_gate.go) 为每个允许 API Key 调用的 Route 注册 Policy。

判断顺序大致是：

1. 这条 `(Method, FullPath)` 是否声明 Policy；
2. 是否要求 Platform Key；
3. Tenant Key 是否 Full Access；
4. Scoped Key 是否具有任一允许 Capability；
5. 若都不满足则拒绝。

没有声明 Policy 的 Route 对 API Key 默认 Deny。这个设计比“记得给危险接口加 Deny Middleware”安全，因为漏配会拒绝而不是放行。

### Route Capability 还不够

带 `retrieve` Capability 的 Key 还可能被限制到：

```text
KnowledgeBaseIDs = [kb-a, kb-b]
```

所以最终权限是：

```text
Route Capability ∩ KB Allow-list
```

`ingest` 也不等于 `manage_kbs`：允许往既有 KB 写内容，不代表允许创建、删除或重新配置 KB。

### Tenant Key 与 Platform Key

- Tenant Key 绑定一个 Tenant，不能借 `X-Tenant-ID` 切到其他空间；
- Platform Key 对多数业务路由必须显式提供目标 `X-Tenant-ID`；
- Platform Key 的能力也由 Platform Capability 限制，不是天然无限权限。

### 源码与说明文档的版本差异

当前 [`docs/RBAC说明.md`](../../docs/RBAC说明.md) 的角色例外部分仍保留了“API Key 在所属空间固定为 Admin”的早期描述；但当前运行代码已经把 API Key 从真人 Role Ladder 中短路出去，改由 `APIKeyRouteAuthorizer`、Capability 与 KB Allow-list 授权。

遇到这种冲突时，应以当前 Route 注册、Middleware、类型定义和测试共同形成的运行契约为准。文档仍适合学习四级角色、Ownership 与共享空间，但 API Key 部分要回到源码核对。这也是“把开源项目当书读”时必须掌握的方法：概念文档给方向，执行路径给事实。

## 外部用户身份为什么还需要签名

一个 Tenant API Key 常被集成服务器共享。如果直接相信客户端传来的：

```text
X-External-User-ID: alice
```

任何拿到 API Key 的调用方都可以冒充 Alice。

[`resolveAPIPrincipal`](../../internal/middleware/auth.go) 支持：

- Tenant 级统一 Principal；
- Direct Header 模式；
- Signed Token 模式。

Signed Token 会验证：

- HS256；
- Audience；
- Expiration；
- 最大 TTL；
- Not Before；
- Tenant ID；
- Subject。

直接 Header 适合网络边界已经可信的后端集成；面向不可信客户端时应优先使用短期签名 Token。

## 跨租户共享不是改写资源 Tenant

Knowledge Base 被共享以后，资源仍属于源 Tenant。共享关系只授予接收方一条访问路径，不会修改：

- `kb.tenant_id`；
- `creator_id`；
- Model 与 Vector Store 所属空间；
- 源空间的审计与配置。

[`internal/middleware/kb_access.go`](../../internal/middleware/kb_access.go) 按顺序解析：

```text
同 Tenant 自有 KB
  → Organization Shared KB
  → Shared Agent 携带的只读 KB
  → 拒绝
```

共享权限与调用者在 Organization 中的角色一起决定有效权限；Shared Agent 路径只提供 Read。

### Effective Tenant Rewrite

当接收方访问源 Tenant 的 Shared KB 时，下游检索必须使用源 Tenant 的 Embedding、索引和存储配置。因此 KB Access Guard 会把 Context 中的 Effective Tenant 重写为 Source Tenant。

这很强大，也很危险：如果同时把 Session、Sandbox 和 OAuth 的所有权都切到源 Tenant，就可能造成跨租户状态串用。

## 一次共享 Agent 请求可能有三个 Tenant 视角

[`internal/types/const.go`](../../internal/types/const.go) 明确区分：

| Context Key | 作用 |
|---|---|
| `TenantIDContextKey` | 当前 Pipeline 使用哪个空间的模型、KB 和配置 |
| `SessionTenantIDContextKey` | Session/Message 属于哪个调用方空间 |
| `SandboxTenantIDContextKey` | Session 到持久 Sandbox 的 Binding 属于谁 |

典型共享 Agent 场景：

```text
调用者 Tenant A
  ├─ Session 属于 A
  ├─ Sandbox Binding 属于 A
  └─ 借用 Tenant B 的 Agent
       ├─ Model 来自 B
       ├─ KB 来自 B
       └─ MCP 配置来自 B
```

若把所有东西只用一个 `tenant_id` 表示，要么读不到 B 的 Agent 资源，要么把 A 的会话和 Sandbox 错存到 B。

这是多租户复杂业务中很值得学习的设计：

> “当前执行空间”和“状态所有者空间”可能不是同一个概念。

## Agent Scope 是比 Tenant 更窄的授权

用户可以访问整个 Tenant，不等于一次 Agent 调用应该搜索整个 Tenant。

一次对话可能只选择：

- 某个 KB；
- 某几份 Knowledge；
- 某些 Tag；
- `@mention` 的一个对象。

这些选择形成服务端拥有的 `SearchTargets`。

模型随后可能生成：

```json
{"knowledge_id": "some-other-document"}
```

不能因为参数来自模型就相信它。模型会幻觉，也可能受到 Prompt Injection 影响。

[`internal/agent/tools/scope_authorization.go`](../../internal/agent/tools/scope_authorization.go) 会：

- 把 Knowledge ID 解析回所属 KB；
- 验证 KB 是否在本次 SearchTargets 中；
- 验证文档或 Tag Scope；
- 把 Chunk 解析回 Knowledge 再验证；
- 过滤 Graph Search 返回的越界结果；
- 用服务端数据重建 Wiki Source Ref，而不是相信模型提供的标题。

核心原则是：

> 模型输出是待校验的用户输入，不是授权证明。

## Prompt Injection 为什么不能只靠 Prompt 防御

你可以在 System Prompt 中写：

```text
不要访问用户未授权的数据。
```

但模型可能：

- 忘记指令；
- 被文档中的恶意文本诱导；
- 生成错误 ID；
- 选择不恰当的 Tool；
- 把敏感 Tool Result 复制到答案。

所以 Prompt 只能表达行为意图，真正边界必须由代码执行：

```text
Tool Allow-list
  + JSON Schema Validation
  + Object Scope Authorization
  + Human Approval
  + Timeout / Output Limit
  + Sandbox / Network Policy
```

这和 SQL 注入防御很像：告诉用户“不要输入恶意 SQL”不能替代参数化查询。

## Tool Schema 不是权限系统

Tool Schema 能验证：

- 必填字段；
- 类型；
- Enum；
- 长度或数值约束。

它不能验证：

- 这个 Knowledge 是否属于当前 Scope；
- 当前 Principal 是否能删除；
- URL 是否指向内网；
- 这次写操作是否得到审批；
- Tool Result 是否包含别人的数据。

[`internal/agent/tools/registry.go`](../../internal/agent/tools/registry.go) 在执行前做 Schema Validation，但具体 Tool 仍必须执行对象级授权。

## 高风险 Tool 需要 Human-in-the-loop

[`internal/agent/approval/gate.go`](../../internal/agent/approval/gate.go) 为危险 MCP Tool 提供审批 Gate。

审批请求至少绑定：

- Tenant；
- User；
- Pending ID；
- Service 与 Tool；
- 参数；
- Timeout。

跨实例通过 Redis Pub/Sub 解析审批结果，且验证 Tenant/User 是否匹配。

Checker 失败时当前默认 Fail-close：要求审批；只有显式配置才退化为 Fail-open。

Human Approval 也不是永久授权。它只应该批准这次明确动作，不能让模型在后续 Round 中无限复用。

## Sandbox 是执行边界，不是一个布尔开关

Agent 执行代码时需要控制：

- 文件系统；
- 进程；
- CPU、内存与时间；
- 网络；
- 环境变量与 Secret；
- Session 生命周期；
- 多租户 Binding。

不同 Sandbox Backend 的隔离强度不同。本地进程、Docker、远程 MicroVM 不能因为都实现同一个接口就被视为同等安全。

还要防止 Session 删除后残留远程 Sandbox 持续运行和计费，所以 Session Owner Tenant 与 Sandbox Binding 必须保持一致，并有 Orphan Reaper。

## 用户可配置 URL 会把服务器变成 SSRF 代理

AI 平台允许配置大量出站地址：

- Model Base URL；
- MCP Endpoint 与 OAuth Metadata；
- Vector Store；
- Storage Backend；
- Data Source；
- Web Fetch；
- Remote Sandbox Control Plane。

如果服务端直接请求任意 URL，攻击者可以探测：

- `127.0.0.1`；
- RFC1918 内网；
- Kubernetes Service；
- Redis/PostgreSQL 等非 HTTP 端口；
- 云元数据 `169.254.169.254`；
- DNS Rebinding 后的内部地址。

## SSRF 防御必须做两次

WeKnora 的安全 Transport 体现了两层检查：

### 保存或构造时验证

检查：

- Scheme 只允许 HTTP/HTTPS；
- Hostname 与受限后缀；
- 直接 IP 和混淆 IP；
- DNS 解析结果；
- 私网、Loopback、Link-local 与 Metadata；
- 危险端口；
- 显式 Whitelist。

这能尽早给配置者可读错误。

### 实际 Dial 时再验证

DNS 结果可能在保存后变化，Redirect 也可能跳到新 Host。因此真正连接时还要校验目标 IP。

[`internal/models/chat/transport.go`](../../internal/models/chat/transport.go) 的共享 HTTP Transport 使用 `SSRFSafeDialContext`；[`internal/mcp/security.go`](../../internal/mcp/security.go) 在保存与建 Client 前检查 MCP 及 OAuth URL；[`internal/sandbox/url_guard.go`](../../internal/sandbox/url_guard.go) 还提供按 Tenant 配置的 Private Endpoint Policy。

Sandbox 即使允许 Private Endpoint，Link-local/Metadata 仍保持禁止。

## Custom Header 也可能成为越权通道

允许用户给 Model 请求添加自定义 Header 很方便，但若能覆盖：

- `Authorization`；
- `Host`；
- Provider 签名 Header；
- Trace 或代理控制 Header；

就可能绕过 Factory 生成的认证或把凭证发送到错误目的地。

因此自定义 Header 必须有保留名单、规范化和最终覆盖顺序。Header 配置本身也应视为 Secret-bearing Data。

## Secret 加密的真实边界

[`internal/utils/crypto.go`](../../internal/utils/crypto.go) 使用 `SYSTEM_AES_KEY` 提供 AES-256-GCM，并用 `enc:v1:` 标记密文。

它保护的配置包括 Model Credential、MCP Auth、Data Source Credential、API Key、Web Search、Sandbox 环境变量等。

需要理解四个边界：

### 数据库加密不等于端到端加密

应用进程仍要解密并调用 Provider。拥有进程内存、环境变量或调试权限的管理员仍可能看到 Secret。

### Legacy Plaintext 兼容仍然存在

没有 `enc:v1:` 前缀的旧值会被视为 Plaintext。迁移期必须计划重写，而不能因为新代码支持加密就假设历史行都已加密。

### Key 丢失与轮换会影响全部凭证

密文存在但 `SYSTEM_AES_KEY` 缺失或变化时无法恢复。需要备份、受控轮换与启动检查。

### 不同读取路径有不同失败策略

- 真正使用 Secret 的 Hot Path 应 Fail Loudly；
- List/Scan 路径可以把无法解密的字段置空并警告，让 UI 仍能展示其他资源；
- 某些高风险配置在没有 Key 时直接拒绝保存 Plaintext。

“统一吞掉解密错误”会让系统把密文当 API Key 发出去；“任何一行坏密文都让列表崩溃”又会降低可恢复性。

## 日志与 Trace 也是数据外泄面

安全不只保护业务数据库。日志和 Trace 可能包含：

- Authorization Header；
- API Key；
- OAuth Code；
- Prompt 和聊天历史；
- 检索证据；
- Tool 参数与结果；
- 外部 URL；
- Error Stack；
- 用户、租户和 Session 标识。

[`internal/middleware/logger.go`](../../internal/middleware/logger.go) 会清理常见敏感字段、限制 Body 大小并跳过 SSE Body；Langfuse Wrapper 对图片、批量文本和 Payload 也做了预览限制。

但完整 Chat Generation 仍可能记录 Prompt 与 Response。应把观测后端纳入同一安全域：

- Access Control；
- Tenant/项目隔离；
- Retention；
- Encryption；
- Export Audit；
- 自建或合规 Region。

## Audit Log 与普通 Log 的区别

普通 Log 主要服务排障，可能轮转、采样或格式变化。Audit Log 服务于“谁对什么做了什么”。

[`internal/types/audit_log.go`](../../internal/types/audit_log.go) 的记录包含：

- Tenant；
- Actor User 与 Role；
- Action；
- Scope Type/ID；
- Target Type/ID/User；
- Request Path；
- Outcome；
- 有界 Details；
- CreatedAt。

记录采用 Append-only，没有 Update 与 Soft Delete 作为正常生命周期。

权限拒绝会写 `rbac.access_denied`，并对重复探测做短时间去重，避免攻击者刷爆审计表。异步操作还应区分 `accepted` 和最终 `success/failed/partial/canceled`。

审计 Details 不应保存正文、Credential、完整 URL 或原始 Stack。审计证据需要足够定位，但不应制造第二份敏感数据仓库。

## 异步 Worker 如何保留安全归属

HTTP Handler 入队以后，Worker 已经没有 JWT。Task Payload 至少要保留：

- Tenant ID；
- 业务对象 ID；
- Task Initiator；
- 必需的 Scope 或版本信息；
- Trace Context。

[`TaskInitiator`](../../internal/types/context_helpers.go) 会快照真人 User ID 与 Role，并在 Worker Context 中恢复，使审计记录仍能归因到发起者。

API Key 对应的 Synthetic User 不被伪装成真人创建者，通常归因成系统工作。

但不能把入队时的 Role 永久视为授权。对于延迟很久或高风险的任务，应考虑：

- 执行时重新验证 Membership；
- 检查资源 Version/Tenant；
- 用户已被移除时取消任务；
- Secret 在执行时重新加载；
- 不把可撤销授权完整固化进 Payload。

## 前端 RBAC 只负责体验

[`frontend/src/stores/auth.ts`](../../frontend/src/stores/auth.ts) 计算：

- `currentTenantRole`；
- `hasRole()`；
- `isSystemAdmin`；
- 当前 Effective Tenant。

前端可以隐藏没有权限的按钮，避免用户点击后才看到 403。

但这些值可能来自 Local Storage，用户可以在 DevTools 修改。源码注释明确指出：它们只能控制 UI，真正授权必须由后端 Middleware、Service 和 Repository 完成。

```text
Frontend Gate = UX
Backend Gate = Security
```

二者需要一致，但不能互相替代。

## Repository 为什么仍要 Tenant Scope

Route Middleware 可以被未来的新入口绕开：

- Worker；
- Internal Service；
- IM Adapter；
- MCP Server；
- 管理脚本；
- 新增但漏挂 Guard 的 Route。

因此 Repository Query 仍应包含 Tenant 或已验证的 Effective Tenant 条件：

```sql
WHERE tenant_id = ? AND id = ?
```

安全成熟度更高时，还可以使用数据库 Row-Level Security。但即使有 RLS，应用层仍要处理共享、角色和业务错误语义。

## 用最小权限交集理解一次请求

以一个 Scoped API Key 调用 Agent Chat 为例：

```text
API Key 有效
  ∩ ScopeType 允许当前 Tenant
  ∩ Route Policy 允许 chat
  ∩ Agent 对调用方可运行
  ∩ KnowledgeBaseIDs 包含 Agent 使用的 KB
  ∩ SearchTargets 允许本次文档/Tag
  ∩ Tool 在注册表和 Agent 配置中可用
  ∩ Tool 参数 Schema 合法
  ∩ 对象级授权通过
  ∩ 高风险 Tool 已审批
  ∩ Sandbox/Outbound Policy 允许执行
```

不要把其中任何一步替换成：

```text
模型认为可以
```

## Threat Modeling：按数据流找边界

可以用下面的表检查一个新 AI 功能：

| 数据流 | 主要威胁 | 典型控制 |
|---|---|---|
| Browser → API | 凭证伪造、Tenant 切换 | JWT/API Key 验证、Membership、X-Tenant-ID 校验 |
| API → Repository | IDOR、漏加 Tenant | Ownership Guard、Tenant Query、404/403 语义 |
| Prompt → Model | 敏感数据外发 | 数据分级、最小上下文、Provider 合规、Trace Redaction |
| Model → Tool | Prompt Injection、幻觉参数 | Allow-list、Schema、Scope Authorization、Approval |
| Tool → Network | SSRF、数据外泄 | URL Validation、Safe Dial、Redirect Policy、Egress Allow-list |
| Tool → Sandbox | 任意代码、资源耗尽 | 隔离、Quota、Timeout、文件/网络限制 |
| HTTP → Queue | 身份丢失、陈旧授权 | Tenant/Initiator Payload、执行时复核、Version Check |
| Worker → Provider | Secret 泄露、无限重试 | Secret Store、Redaction、Retry Budget、Concurrency Gate |
| Trace/Log → Backend | 二次数据泄露 | Redaction、Sampling、Retention、Viewer RBAC |

## 与 Java / Spring 的对应关系

| WeKnora / Go | Java / Spring 常见对应 |
|---|---|
| Gin Auth Middleware | Spring Security Authentication Filter |
| `Principal` | `Authentication.getPrincipal()` 的领域扩展 |
| Tenant Context | TenantContext / AbstractRoutingDataSource Context |
| `RequireRole` | `@PreAuthorize("hasRole(...)")` |
| Ownership Lookup | PermissionEvaluator / Domain ACL |
| APIKeyRoutePolicy | OAuth Scope / 自定义 AuthorizationManager |
| KB Access Guard | Method Security + Resource Authorization Service |
| `context.Context` | SecurityContext + Reactor Context |
| TaskInitiator Payload | Message Header 中的 Actor/Tenant Snapshot |
| AES-GCM Secret | KMS/Vault 封装或 JPA AttributeConverter |
| Safe HTTP Transport | 受控 `WebClient` Connector / DNS 与 Egress Policy |
| Tool Approval Gate | Workflow Human Task / Approval Service |

Java 开发者应特别注意 ThreadLocal TenantContext：异步线程、WebFlux、消息队列中很容易丢失或串租户。Context 必须显式传播，并在任务结束时释放。

## 当前实现值得学习的设计

### Principal 与 User 分离

机器、外部用户、IM 用户和嵌入访客不会因为共享一个 Tenant 凭证就共享所有状态。

### 人类 RBAC 与机器 Capability 分离

API Key 不依赖伪造 Admin Role；Route Policy 默认拒绝，Capability 与 KB Allow-list 取交集。

### Role 与 Ownership 分离

Contributor 可以管理自己的资源，不能横向修改同空间其他 Contributor 的资源。

### 共享访问显式重写 Effective Tenant

下游能使用源空间的模型和索引，同时用独立 Context Key 保留 Session/Sandbox 所有者。

### 模型参数再次经过对象 Scope 校验

Agent 不信任模型生成的 UUID、Title 或 Graph Result，服务端根据 SearchTargets 和真实数据重建授权。

### SSRF 在保存与 Dial 两层防护

既改善配置体验，又封住 DNS Rebinding 与 Redirect 的运行时窗口。

### Secret 解密区分展示路径和使用路径

列表尽量可恢复，真正调用时明确失败，避免密文被当成 Credential。

## 当前边界与可演进方向

### 从固定角色到 Policy Engine

当规则扩展到环境、资源标签、时间和风险等级时，可以考虑 ABAC 或 Policy Engine。但固定四级 Role + Ownership 更容易理解和审计，不应过早复杂化。

### 数据库级 Tenant 防线

为核心表增加 PostgreSQL RLS、Tenant-aware DB Session 或静态 Query Lint，降低 Repository 漏写 Tenant 条件的风险。

### API Key Rotation 与短期 Credential

支持 Key Version、双 Key 轮换、LastUsed、精细 Expiration、按 Principal 的速率限制与异常行为告警。

### Secret Manager / KMS

用 Vault、KMS Envelope Encryption 或云 Secret Manager 替代单一进程环境变量，提供轮换、审计和分权。

### Egress Proxy

把所有 Model/MCP/Web/Sandbox 出站集中到受控 Proxy，按 Tenant/Provider 建 Allow-list、速率限制与流量审计。

### Prompt 与 Tool 数据防泄漏

引入敏感字段分类、PII 检测、Tool Result Label、回答前 DLP 和不同 Provider 的数据区域策略。

### 异步执行时重新授权

对延迟任务在执行点检查 Membership、资源版本、共享关系和 Secret 状态，防止撤权后旧任务继续执行。

### 安全测试矩阵

按 `(Principal Type, Tenant, Role/Capability, Resource Owner, Share Permission, Agent Scope)` 生成表驱动测试，尤其覆盖：

- 同租户横向越权；
- 跨租户 UUID 猜测；
- API Key 漏注册 Route；
- Shared Agent 的 Session/Sandbox 隔离；
- Model 生成越界 Tool 参数；
- SSRF Redirect 与 DNS Rebinding；
- Secret Key 轮换失败。

## 推荐的源码阅读顺序

1. [`internal/types/principal.go`](../../internal/types/principal.go)：先区分 User、Principal 与 Session Owner。
2. [`internal/middleware/auth.go`](../../internal/middleware/auth.go)：跟踪 JWT、API Key、目标 Tenant 和外部用户 Principal。
3. [`internal/middleware/auth_context.go`](../../internal/middleware/auth_context.go) 与 [`internal/types/context_helpers.go`](../../internal/types/context_helpers.go)：看安全状态如何进入 Context。
4. [`internal/types/tenant_member.go`](../../internal/types/tenant_member.go) 和 [`internal/middleware/rbac.go`](../../internal/middleware/rbac.go)：理解 Role 与 Ownership。
5. [`internal/types/tenant_api_key.go`](../../internal/types/tenant_api_key.go) 和 [`internal/middleware/api_key_gate.go`](../../internal/middleware/api_key_gate.go)：理解机器 Capability 与默认拒绝。
6. [`internal/middleware/kb_access.go`](../../internal/middleware/kb_access.go)：跟踪 Own、Organization Share 与 Shared Agent 三条路径。
7. [`internal/types/const.go`](../../internal/types/const.go)：重点读 Session/Sandbox Tenant 的注释。
8. [`internal/agent/tools/scope_authorization.go`](../../internal/agent/tools/scope_authorization.go)：看模型参数怎样被服务端 Scope 再授权。
9. [`internal/agent/tools/registry.go`](../../internal/agent/tools/registry.go) 与 [`internal/agent/approval/gate.go`](../../internal/agent/approval/gate.go)：看 Schema、执行和审批边界。
10. [`internal/utils/crypto.go`](../../internal/utils/crypto.go)：理解 Secret 的严格与宽松读取。
11. [`internal/utils/security.go`](../../internal/utils/security.go)、[`internal/mcp/security.go`](../../internal/mcp/security.go) 与 [`internal/sandbox/url_guard.go`](../../internal/sandbox/url_guard.go)：理解出站网络防护。
12. [`frontend/src/stores/auth.ts`](../../frontend/src/stores/auth.ts)：最后看前端如何镜像 Role，同时明确它不承担真实授权。
13. 配合 [`docs/RBAC说明.md`](../../docs/RBAC说明.md) 阅读角色矩阵、共享空间和上线策略。

## 常见误区

### 误区一：所有表都有 tenant_id 就安全了

Query 漏条件、共享路径、异步 Context 丢失和 IDOR 仍会越权。

### 误区二：认证成功就是 Admin

真人要看 Tenant Role 与 Ownership，API Key 要看 Route Capability 和 KB Scope。

### 误区三：前端隐藏按钮就是权限控制

Local Storage 和前端状态都可篡改，后端必须独立验证。

### 误区四：Tool Schema 能阻止越权

Schema 只能验证形状，不能证明对象属于本次 Scope。

### 误区五：模型不会故意访问越界对象

模型会幻觉，也会被 Prompt Injection 诱导。所有模型输出都必须当作不可信输入。

### 误区六：URL 保存时校验一次就够了

DNS Rebinding、Redirect 和配置导入要求在真正 Dial 时再次验证。

### 误区七：数据库密文意味着 Secret 已经安全

应用仍能解密；日志、Trace、内存、环境变量和错误响应都是新的泄漏面。

### 误区八：共享就是把 tenant_id 改成接收方

共享只增加访问关系，资源归属与源空间配置不能改变。

### 误区九：一个 Tenant Context 足以描述共享 Agent

Pipeline Tenant、Session Tenant 和 Sandbox Tenant 可能不同。

### 误区十：入队时有权限，Worker 永远有权限

Membership、Share 和 Secret 都可撤销；高风险延迟任务应在执行时复核。

## 本章小结

- 多租户 AI 安全是身份、租户、主体类型、角色/能力、资源、Agent Scope 和执行策略的交集。
- User 表示账号，Principal 表示需要隔离的终端调用主体；二者不能混为一谈。
- JWT 认证后仍要解析目标 Tenant、Membership 与当前 Role。
- Viewer、Contributor、Admin、Owner 构成空间角色梯子，资源写入还要叠加 Creator Ownership。
- System Admin 与 Tenant Owner 是两个正交权限维度。
- API Key 使用独立 Route Capability 与 KB Allow-list；未声明 Policy 的 Route 默认拒绝。
- 跨租户共享不改变资源归属，KB Guard 会解析 Effective Tenant。
- Shared Agent 需要区分 Pipeline、Session 与 Sandbox 的 Tenant 所有权。
- SearchTargets 是比 Tenant 更窄的本次授权，模型生成的 Tool 参数必须再次校验。
- Prompt、Tool Schema 和模型自律都不能替代服务端对象授权与 Human Approval。
- 用户可配置的 Model/MCP/Vector/Sandbox URL 必须在保存与 Dial 两层防 SSRF。
- AES-GCM 保护数据库 Secret，但仍需处理 Legacy Plaintext、Key 轮换和日志/Trace 泄漏。
- Audit Log 保存安全事实，普通 Log 和 Trace 不能替代它。
- 前端 RBAC 只改善体验，后端 Guard 与 Tenant-scoped Repository 才是安全边界。

### 可选实践

设计一条“Tenant A 的 Scoped API Key 代表外部用户 Alice，运行 Tenant B 分享过来的 Agent；该 Agent 读取 B 的两个 KB，并调用一个需要审批的 MCP 写工具”的授权链。为每一步写出 Principal、Pipeline Tenant、Session Tenant、Sandbox Tenant、Route Capability、KB Allow-list、Organization Permission、SearchTargets、Tool Schema、对象授权和 Approval。然后分别模拟 Alice 伪造 Bob、API Key 缺少 `chat`、模型生成第三个 KB 的 UUID、MCP URL 重定向到云元数据、用户在任务入队后被撤权，说明哪一道 Gate 应拒绝以及需要记录什么审计证据。

下一章计划：[部署形态：Lite、标准版与可替换基础设施](../README.md)
