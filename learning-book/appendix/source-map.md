# 附录 A　源码地图

[返回全书目录](../README.md)

这份地图不是完整目录说明，而是按学习问题提供稳定入口。遇到一个主题时，先从“编排/接口”进入，再选择一个具体实现，不要同时阅读所有供应商和后端。

## 0. 程序如何启动

| 问题 | 入口 |
|---|---|
| 服务进程入口 | [`cmd/server/main.go`](../../cmd/server/main.go) |
| 应用 Bootstrap | [`cmd/server/bootstrap.go`](../../cmd/server/bootstrap.go) |
| HTTP 监听与退出 | [`cmd/server/listen.go`](../../cmd/server/listen.go) |
| 依赖容器 | [`internal/container`](../../internal/container) |
| 路由总装配 | [`internal/router/router.go`](../../internal/router/router.go) |

## 1. 前端聊天与流

| 问题 | 入口 |
|---|---|
| Chat API | [`frontend/src/api/chat/index.ts`](../../frontend/src/api/chat/index.ts) |
| SSE 建立、中止与解析 | [`frontend/src/api/chat/streame.ts`](../../frontend/src/api/chat/streame.ts) |
| 流事件转 UI 状态 | [`frontend/src/composables/useChatStreamHandler.ts`](../../frontend/src/composables/useChatStreamHandler.ts) |
| 输入区与发送状态 | [`frontend/src/components/Input-field.vue`](../../frontend/src/components/Input-field.vue) |
| 引用抽屉 | [`frontend/src/composables/useChatReferencesDrawer.ts`](../../frontend/src/composables/useChatReferencesDrawer.ts) |
| 引用浮层 | [`frontend/src/composables/useChatCitationPopover.ts`](../../frontend/src/composables/useChatCitationPopover.ts) |

## 2. HTTP API 入口

| 问题 | 入口 |
|---|---|
| 普通问答、Agent、纯检索路由 | [`internal/router/routes_chat.go`](../../internal/router/routes_chat.go) |
| QA Handler | [`internal/handler/session/qa.go`](../../internal/handler/session/qa.go) |
| 会话流处理 | [`internal/handler/session/stream.go`](../../internal/handler/session/stream.go) |
| 知识上传和管理路由 | [`internal/router/routes_knowledge.go`](../../internal/router/routes_knowledge.go) |
| 知识 Handler | [`internal/handler/knowledge.go`](../../internal/handler/knowledge.go) |

## 3. 普通 RAG 编排

| 问题 | 入口 |
|---|---|
| RAG 总编排 | [`internal/application/service/session_knowledge_qa.go`](../../internal/application/service/session_knowledge_qa.go) |
| 管线状态 | [`internal/types/chat_manage.go`](../../internal/types/chat_manage.go) |
| QA 请求 | [`internal/types/qa_request.go`](../../internal/types/qa_request.go) |
| 管线插件目录 | [`internal/application/service/chat_pipeline`](../../internal/application/service/chat_pipeline) |
| 查询理解 | [`chat_pipeline/query_understand.go`](../../internal/application/service/chat_pipeline/query_understand.go) |
| 并行搜索 | [`chat_pipeline/search_parallel.go`](../../internal/application/service/chat_pipeline/search_parallel.go) |
| Rerank | [`chat_pipeline/rerank.go`](../../internal/application/service/chat_pipeline/rerank.go) |
| 合并 | [`chat_pipeline/merge.go`](../../internal/application/service/chat_pipeline/merge.go) |
| Top-K 过滤 | [`chat_pipeline/filter_top_k.go`](../../internal/application/service/chat_pipeline/filter_top_k.go) |
| 构造模型消息 | [`chat_pipeline/into_chat_message.go`](../../internal/application/service/chat_pipeline/into_chat_message.go) |
| 流式生成 | [`chat_pipeline/chat_completion_stream.go`](../../internal/application/service/chat_pipeline/chat_completion_stream.go) |

## 4. 文档入库

| 问题 | 入口 |
|---|---|
| 创建知识 | [`internal/application/service/knowledge_create.go`](../../internal/application/service/knowledge_create.go) |
| 处理主干 | [`internal/application/service/knowledge_process.go`](../../internal/application/service/knowledge_process.go) |
| 后处理 | [`internal/application/service/knowledge_post_process.go`](../../internal/application/service/knowledge_post_process.go) |
| 文档解析基础设施 | [`internal/infrastructure/docparser`](../../internal/infrastructure/docparser) |
| Chunker | [`internal/infrastructure/chunker`](../../internal/infrastructure/chunker) |
| Chunk Service | [`internal/application/service/chunk.go`](../../internal/application/service/chunk.go) |
| Embedding 输入构造 | [`internal/application/service/knowledge_index_content.go`](../../internal/application/service/knowledge_index_content.go) |

## 5. 检索

| 问题 | 入口 |
|---|---|
| 混合搜索主干 | [`internal/application/service/knowledgebase_search.go`](../../internal/application/service/knowledgebase_search.go) |
| 结果融合 | [`internal/application/service/knowledgebase_search_fusion.go`](../../internal/application/service/knowledgebase_search_fusion.go) |
| 多存储 fan-out | [`internal/application/service/knowledgebase_search_fanout.go`](../../internal/application/service/knowledgebase_search_fanout.go) |
| 检索接口 | [`internal/types/interfaces/retriever.go`](../../internal/types/interfaces/retriever.go) |
| 检索引擎工厂 | [`internal/application/service/retriever/factory.go`](../../internal/application/service/retriever/factory.go) |
| 组合检索引擎 | [`internal/application/service/retriever/composite.go`](../../internal/application/service/retriever/composite.go) |
| PostgreSQL 实现 | [`internal/application/repository/retriever/postgres`](../../internal/application/repository/retriever/postgres) |
| SQLite 实现 | [`internal/application/repository/retriever/sqlite`](../../internal/application/repository/retriever/sqlite) |

阅读具体后端时只选一个。第一次优先 PostgreSQL 或 SQLite。

## 6. 模型能力

| 问题 | 入口 |
|---|---|
| Chat 接口和工厂 | [`internal/models/chat/chat.go`](../../internal/models/chat/chat.go) |
| OpenAI 兼容 Chat | [`internal/models/chat/remote_api.go`](../../internal/models/chat/remote_api.go) |
| Provider 适配 | [`internal/models/chat/provider.go`](../../internal/models/chat/provider.go) |
| Ollama Chat | [`internal/models/chat/ollama.go`](../../internal/models/chat/ollama.go) |
| Embedding 接口和工厂 | [`internal/models/embedding/embedder.go`](../../internal/models/embedding/embedder.go) |
| Embedding 批处理 | [`internal/models/embedding/batch.go`](../../internal/models/embedding/batch.go) |
| Rerank 接口和工厂 | [`internal/models/rerank/reranker.go`](../../internal/models/rerank/reranker.go) |
| 模型业务配置 | [`internal/application/service/model.go`](../../internal/application/service/model.go) |

## 7. Agent 与工具

| 问题 | 入口 |
|---|---|
| Agent QA 编排 | [`internal/application/service/session_agent_qa.go`](../../internal/application/service/session_agent_qa.go) |
| Agent 核心 | [`internal/agent`](../../internal/agent) |
| 工具目录 | [`internal/agent/tools`](../../internal/agent/tools) |
| 知识检索工具 | [`internal/agent/tools/knowledge_search.go`](../../internal/agent/tools/knowledge_search.go) |
| Web 搜索工具 | [`internal/agent/tools/web_search.go`](../../internal/agent/tools/web_search.go) |
| Skill 执行 | [`internal/agent/tools/skill_execute.go`](../../internal/agent/tools/skill_execute.go) |
| MCP | [`internal/mcp`](../../internal/mcp) |
| 独立 MCP Server | [`mcp-server`](../../mcp-server) |

建议完成第二篇 RAG 后再进入此区域。

## 8. 事件、流与异步任务

| 问题 | 入口 |
|---|---|
| 应用事件 | [`internal/event`](../../internal/event) |
| 流管理 | [`internal/stream`](../../internal/stream) |
| 任务路由和检查 | [`internal/router/task.go`](../../internal/router/task.go) |
| Runtime | [`internal/runtime`](../../internal/runtime) |
| Worker 池说明 | [`docs/worker-pool-governance.md`](../../docs/worker-pool-governance.md) |

## 9. 可观测性和治理

| 问题 | 入口 |
|---|---|
| Tracing | [`internal/tracing`](../../internal/tracing) |
| Langfuse 文档 | [`docs/Langfuse集成.md`](../../docs/Langfuse集成.md) |
| RBAC | [`internal/router/rbac.go`](../../internal/router/rbac.go) |
| RBAC 说明 | [`docs/RBAC说明.md`](../../docs/RBAC说明.md) |
| 中间件 | [`internal/middleware`](../../internal/middleware) |
| 限流 | [`internal/ratelimit`](../../internal/ratelimit) |
| 沙箱 | [`internal/sandbox`](../../internal/sandbox) |

## 10. 推荐的第一次调用链

只追普通 RAG：

```text
frontend/src/api/chat/streame.ts
  → internal/router/routes_chat.go
  → internal/handler/session/qa.go::KnowledgeQA
  → internal/application/service/session_knowledge_qa.go::KnowledgeQA
  → internal/application/service/chat_pipeline/*
  → internal/application/service/knowledgebase_search.go::HybridSearch
  → internal/models/chat/chat.go::ChatStream
  → EventBus / SSE
  → frontend/src/composables/useChatStreamHandler.ts
```

第一次达到这里就停止，不要继续追所有 Repository 和供应商实现。
