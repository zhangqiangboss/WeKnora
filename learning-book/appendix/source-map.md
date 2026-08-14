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
| 短 Chunk 相邻扩展 | [`chat_pipeline/merge_expand.go`](../../internal/application/service/chat_pipeline/merge_expand.go) |
| 重叠识别与合并 | [`chat_pipeline/merge_overlap.go`](../../internal/application/service/chat_pipeline/merge_overlap.go) |
| Top-K 过滤 | [`chat_pipeline/filter_top_k.go`](../../internal/application/service/chat_pipeline/filter_top_k.go) |
| 构造模型消息 | [`chat_pipeline/into_chat_message.go`](../../internal/application/service/chat_pipeline/into_chat_message.go) |
| 模型引用句柄与来源映射 | [`chat_pipeline/references.go`](../../internal/application/service/chat_pipeline/references.go) |
| 流式生成 | [`chat_pipeline/chat_completion_stream.go`](../../internal/application/service/chat_pipeline/chat_completion_stream.go) |

## 3.1 Token 与上下文管理

| 问题 | 入口 |
|---|---|
| Chat 消息和生成参数 | [`internal/models/chat/chat.go`](../../internal/models/chat/chat.go) |
| 普通 RAG 历史加载 | [`chat_pipeline/load_history.go`](../../internal/application/service/chat_pipeline/load_history.go) |
| Agent Token 估算 | [`internal/agent/token/estimator.go`](../../internal/agent/token/estimator.go) |
| Agent 上下文压缩 | [`internal/agent/token/compress.go`](../../internal/agent/token/compress.go) |
| 旧历史总结 | [`internal/agent/memory/consolidator.go`](../../internal/agent/memory/consolidator.go) |
| 工具输出截断 | [`internal/agent/tools/truncate.go`](../../internal/agent/tools/truncate.go) |

## 4. 文档入库

| 问题 | 入口 |
|---|---|
| 创建知识 | [`internal/application/service/knowledge_create.go`](../../internal/application/service/knowledge_create.go) |
| 处理主干 | [`internal/application/service/knowledge_process.go`](../../internal/application/service/knowledge_process.go) |
| 后处理 | [`internal/application/service/knowledge_post_process.go`](../../internal/application/service/knowledge_post_process.go) |
| 文档解析基础设施 | [`internal/infrastructure/docparser`](../../internal/infrastructure/docparser) |
| Chunker | [`internal/infrastructure/chunker`](../../internal/infrastructure/chunker) |
| 自适应策略链 | [`internal/infrastructure/chunker/strategy.go`](../../internal/infrastructure/chunker/strategy.go) |
| 文档画像 | [`internal/infrastructure/chunker/profiler.go`](../../internal/infrastructure/chunker/profiler.go) |
| 切块结果校验 | [`internal/infrastructure/chunker/validator.go`](../../internal/infrastructure/chunker/validator.go) |
| 标题切分 | [`internal/infrastructure/chunker/heading_splitter.go`](../../internal/infrastructure/chunker/heading_splitter.go) |
| 启发式切分 | [`internal/infrastructure/chunker/heuristic_splitter.go`](../../internal/infrastructure/chunker/heuristic_splitter.go) |
| Chunk Service | [`internal/application/service/chunk.go`](../../internal/application/service/chunk.go) |
| Embedding 输入构造 | [`internal/application/service/knowledge_index_content.go`](../../internal/application/service/knowledge_index_content.go) |

### 4.1 文档解析、OCR 与图片加工

| 问题 | 入口 |
|---|---|
| 统一读取请求、结果与图片引用 | [`internal/types/docparser.go`](../../internal/types/docparser.go) |
| DocReader 接口与远程引擎发现 | [`internal/types/interfaces/document_parser.go`](../../internal/types/interfaces/document_parser.go) |
| 简单格式、原始图片与音频 | [`internal/infrastructure/docparser/builtin_converter.go`](../../internal/infrastructure/docparser/builtin_converter.go) |
| 解析引擎注册、能力与可用性 | [`internal/infrastructure/docparser/engine_registry.go`](../../internal/infrastructure/docparser/engine_registry.go) |
| 文档转换、存图、切块与多模态编排 | [`internal/application/service/knowledge_process.go`](../../internal/application/service/knowledge_process.go) |
| Markdown 图片解析、持久化与引用重写 | [`internal/infrastructure/docparser/image_resolver.go`](../../internal/infrastructure/docparser/image_resolver.go) |
| OCR/VLM 输出清洗 | [`internal/application/service/ocr_sanitizer.go`](../../internal/application/service/ocr_sanitizer.go) |
| MinerU 适配器 | [`internal/infrastructure/docparser/mineru_converter.go`](../../internal/infrastructure/docparser/mineru_converter.go) |
| PaddleOCR-VL 适配器 | [`internal/infrastructure/docparser/paddleocr_vl_converter.go`](../../internal/infrastructure/docparser/paddleocr_vl_converter.go) |

### 4.2 图片、VLM 与多模态检索

| 问题 | 入口 |
|---|---|
| VLM 接口、配置与工厂 | [`internal/models/vlm/vlm.go`](../../internal/models/vlm/vlm.go) |
| 图片 OCR、Caption、子 Chunk 与索引 | [`internal/application/service/image_multimodal.go`](../../internal/application/service/image_multimodal.go) |
| 图片 Chunk 类型和 ImageInfo | [`internal/types/chunk.go`](../../internal/types/chunk.go) |
| 图片信息聚合、过滤与上下文渲染 | [`internal/searchutil/imageinfo.go`](../../internal/searchutil/imageinfo.go) |
| 图片命中后的父文本恢复 | [`internal/application/service/chat_pipeline/merge.go`](../../internal/application/service/chat_pipeline/merge.go) |
| 用户图片与视觉模型消息构造 | [`internal/application/service/chat_pipeline/common.go`](../../internal/application/service/chat_pipeline/common.go) |
| 图片参与查询理解与 Caption 持久化 | [`internal/application/service/chat_pipeline/query_understand.go`](../../internal/application/service/chat_pipeline/query_understand.go) |
| 文本 Embedding 统一接口 | [`internal/models/embedding/embedder.go`](../../internal/models/embedding/embedder.go) |

### 4.3 知识图谱与 Wiki

| 问题 | 入口 |
|---|---|
| 图节点、关系与 Namespace | [`internal/types/extract_graph.go`](../../internal/types/extract_graph.go) |
| 图仓库 Add/Del/Search 接口 | [`internal/types/interfaces/retriever_graph.go`](../../internal/types/interfaces/retriever_graph.go) |
| 逐 Chunk 图抽取与写入 | [`internal/application/service/extract.go`](../../internal/application/service/extract.go) |
| 在线实体图搜索 | [`internal/application/service/chat_pipeline/search_entity.go`](../../internal/application/service/chat_pipeline/search_entity.go) |
| Agent 图谱查询与范围校验 | [`internal/agent/tools/query_knowledge_graph.go`](../../internal/agent/tools/query_knowledge_graph.go) |
| Wiki 页面、来源、链接、版本与配置 | [`internal/types/wiki_page.go`](../../internal/types/wiki_page.go) |
| Wiki pending、任务、锁与核心数据结构 | [`internal/application/service/wiki_ingest.go`](../../internal/application/service/wiki_ingest.go) |
| Wiki Map/Reduce 与 Finalize | [`internal/application/service/wiki_ingest_batch.go`](../../internal/application/service/wiki_ingest_batch.go) |
| Chunk Citation | [`internal/application/service/wiki_ingest_cite.go`](../../internal/application/service/wiki_ingest_cite.go) |
| 页面候选去重 | [`internal/application/service/wiki_ingest_dedup.go`](../../internal/application/service/wiki_ingest_dedup.go) |
| 分类与目录计划 | [`internal/application/service/wiki_ingest_taxonomy.go`](../../internal/application/service/wiki_ingest_taxonomy.go) |
| 页面 CRUD、Revision、链接与目录 | [`internal/application/service/wiki_page.go`](../../internal/application/service/wiki_page.go) |
| Wiki 结构健康检查 | [`internal/application/service/wiki_lint.go`](../../internal/application/service/wiki_lint.go) |

### 4.4 外部数据源与持续同步

| 问题 | 入口 |
|---|---|
| DataSource、Resource、FetchedItem、Cursor 与 SyncLog | [`internal/types/datasource.go`](../../internal/types/datasource.go) |
| Connector、StreamingConnector 与注册表接口 | [`internal/datasource/connector.go`](../../internal/datasource/connector.go) |
| 运行时实际注册的 Connector | [`internal/container/container.go`](../../internal/container/container.go) 中的 `initConnectorRegistry` |
| 同步任务、流式处理、统一入库与结果状态 | [`internal/application/service/datasource_service.go`](../../internal/application/service/datasource_service.go) |
| 飞书 Wiki/Drive 共用同步引擎 | [`internal/datasource/connector/feishu/core/engine.go`](../../internal/datasource/connector/feishu/core/engine.go) |
| GitLab commit cursor、rename/delete 与回退全量 | [`internal/datasource/connector/gitlab/connector.go`](../../internal/datasource/connector/gitlab/connector.go) |
| RSS 指纹、部分失败与无删除语义 | [`internal/datasource/connector/rss/connector.go`](../../internal/datasource/connector/rss/connector.go) |
| Cron 与多实例任务去重 | [`internal/datasource/scheduler.go`](../../internal/datasource/scheduler.go) |

## 5. 检索

| 问题 | 入口 |
|---|---|
| 混合搜索主干 | [`internal/application/service/knowledgebase_search.go`](../../internal/application/service/knowledgebase_search.go) |
| 结果融合 | [`internal/application/service/knowledgebase_search_fusion.go`](../../internal/application/service/knowledgebase_search_fusion.go) |
| 多存储 fan-out | [`internal/application/service/knowledgebase_search_fanout.go`](../../internal/application/service/knowledgebase_search_fanout.go) |
| 分数归一化 | [`internal/application/service/retriever/normalizer.go`](../../internal/application/service/retriever/normalizer.go) |
| 检索参数与 RRF 默认值 | [`internal/types/retrieval_config.go`](../../internal/types/retrieval_config.go) |
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
| Chat Provider 差异适配 | [`internal/models/chat/provider.go`](../../internal/models/chat/provider.go) |
| Ollama Chat | [`internal/models/chat/ollama.go`](../../internal/models/chat/ollama.go) |
| Embedding 接口和工厂 | [`internal/models/embedding/embedder.go`](../../internal/models/embedding/embedder.go) |
| Embedding 批处理 | [`internal/models/embedding/batch.go`](../../internal/models/embedding/batch.go) |
| Rerank 接口和工厂 | [`internal/models/rerank/reranker.go`](../../internal/models/rerank/reranker.go) |
| VLM 接口和工厂 | [`internal/models/vlm/vlm.go`](../../internal/models/vlm/vlm.go) |
| ASR 接口和工厂 | [`internal/models/asr/asr.go`](../../internal/models/asr/asr.go) |
| Provider 元数据注册表 | [`internal/models/provider/provider.go`](../../internal/models/provider/provider.go) |
| 模型配置解析与客户端构造 | [`internal/application/service/model.go`](../../internal/application/service/model.go) |
| 模型实体、类型与参数 | [`internal/types/model.go`](../../internal/types/model.go) |
| 后台模型并发治理 | [`internal/models/limiter`](../../internal/models/limiter) |
| 模型 Debug 生产路径复用 | [`internal/handler/model.go`](../../internal/handler/model.go) |

## 7. Agent 与工具

| 问题 | 入口 |
|---|---|
| Agent QA 编排 | [`internal/application/service/session_agent_qa.go`](../../internal/application/service/session_agent_qa.go) |
| Agent Engine 与 ReAct 循环 | [`internal/agent/engine.go`](../../internal/agent/engine.go) |
| 流式模型调用 / Think | [`internal/agent/think.go`](../../internal/agent/think.go) |
| 工具执行 / Act | [`internal/agent/act.go`](../../internal/agent/act.go) |
| 终止判断与结果回填 / Observe | [`internal/agent/observe.go`](../../internal/agent/observe.go) |
| 达到上限后的答案收尾 | [`internal/agent/finalize.go`](../../internal/agent/finalize.go) |
| Agent、Tool 和状态类型 | [`internal/types/agent.go`](../../internal/types/agent.go) |
| 工具目录 | [`internal/agent/tools`](../../internal/agent/tools) |
| 工具注册、Schema 投影与执行 | [`internal/agent/tools/registry.go`](../../internal/agent/tools/registry.go) |
| 参数校验 | [`internal/agent/tools/param_validate.go`](../../internal/agent/tools/param_validate.go) |
| 内置工具白名单与名称 | [`internal/agent/tools/definitions.go`](../../internal/agent/tools/definitions.go) |
| 知识检索工具 | [`internal/agent/tools/knowledge_search.go`](../../internal/agent/tools/knowledge_search.go) |
| Web 搜索工具 | [`internal/agent/tools/web_search.go`](../../internal/agent/tools/web_search.go) |
| 工具人工审批 | [`internal/agent/approval/gate.go`](../../internal/agent/approval/gate.go) |
| Skill 按需读取 | [`internal/agent/tools/skill_read.go`](../../internal/agent/tools/skill_read.go) |
| Skill 脚本执行 | [`internal/agent/tools/skill_execute.go`](../../internal/agent/tools/skill_execute.go) |

建议先沿 `session_agent_qa.go → engine.go → think/act/observe.go → registry.go` 跟踪一个只调用一次读工具的请求，再进入 MCP、Skills 和沙箱。

## 7.1 MCP

| 问题 | 入口 |
|---|---|
| MCP Client 接口、握手、发现与调用 | [`internal/mcp/client.go`](../../internal/mcp/client.go) |
| MCP Capability 与内容类型 | [`internal/mcp/types.go`](../../internal/mcp/types.go) |
| Client 缓存与主体隔离 | [`internal/mcp/manager.go`](../../internal/mcp/manager.go) |
| 远程端点安全校验 | [`internal/mcp/security.go`](../../internal/mcp/security.go) |
| MCP Tool 到内部 Tool 的适配 | [`internal/agent/tools/mcp_tool.go`](../../internal/agent/tools/mcp_tool.go) |
| 对话内 OAuth 等待与重试 | [`internal/agent/tools/mcp_oauth.go`](../../internal/agent/tools/mcp_oauth.go) |
| MCP 服务业务配置 | [`internal/application/service/mcp_service.go`](../../internal/application/service/mcp_service.go) |
| 独立 WeKnora MCP Server | [`mcp-server`](../../mcp-server) |

## 7.2 Skills 与沙箱

| 问题 | 入口 |
|---|---|
| Skill 格式与元数据验证 | [`internal/agent/skills/skill.go`](../../internal/agent/skills/skill.go) |
| Skill 发现、加载与路径边界 | [`internal/agent/skills/loader.go`](../../internal/agent/skills/loader.go) |
| Skill 白名单、读取与执行编排 | [`internal/agent/skills/manager.go`](../../internal/agent/skills/manager.go) |
| 统一 Sandbox 接口和配置 | [`internal/sandbox/sandbox.go`](../../internal/sandbox/sandbox.go) |
| 会话 Shell 与文件能力 | [`internal/sandbox/capabilities.go`](../../internal/sandbox/capabilities.go) |
| 会话型远程沙箱管理 | [`internal/sandbox/session_manager.go`](../../internal/sandbox/session_manager.go) |
| Binding、恢复与销毁 | [`internal/sandbox/session_lifecycle.go`](../../internal/sandbox/session_lifecycle.go) |
| 脚本静态安全检查 | [`internal/sandbox/validator.go`](../../internal/sandbox/validator.go) |
| 沙箱端点 SSRF 防护 | [`internal/sandbox/url_guard.go`](../../internal/sandbox/url_guard.go) |

## 7.3 Agent 权限、预算与恢复

| 问题 | 入口 |
|---|---|
| LLM、Tool 与重试默认预算 | [`internal/agent/const.go`](../../internal/agent/const.go) |
| 模型调用超时、重试与证据降级 | [`internal/agent/think.go`](../../internal/agent/think.go) |
| 工具结果预算、上下文压缩与终止 | [`internal/agent/observe.go`](../../internal/agent/observe.go) |
| 请求级 Skill/MCP/检索范围收窄 | [`internal/application/service/session_agent_qa.go`](../../internal/application/service/session_agent_qa.go) |
| 知识对象级范围授权 | [`internal/agent/tools/scope_authorization.go`](../../internal/agent/tools/scope_authorization.go) |
| 工具审批 Gate | [`internal/agent/approval/gate.go`](../../internal/agent/approval/gate.go) |
| Agent 历史持久化重建 | [`internal/application/service/agent_history.go`](../../internal/application/service/agent_history.go) |

## 8. 事件、流与异步任务

| 问题 | 入口 |
|---|---|
| 请求内应用事件与 EventBus | [`internal/event/event.go`](../../internal/event/event.go) |
| StreamEvent 与 StreamManager 接口 | [`internal/types/interfaces/stream_manager.go`](../../internal/types/interfaces/stream_manager.go) |
| Memory/Redis 事件回放实现 | [`internal/stream`](../../internal/stream) |
| EventBus 到流事件的投影 | [`internal/handler/session/agent_stream_handler.go`](../../internal/handler/session/agent_stream_handler.go) |
| SSE 消费、ContinueStream 与 StopSession | [`internal/handler/session/stream.go`](../../internal/handler/session/stream.go) |
| SSE Header、终止约定与独立 Stop Watcher | [`internal/handler/session/helpers.go`](../../internal/handler/session/helpers.go) |
| 上游 OpenAI 兼容 SSE 读取 | [`internal/models/chat/sse_reader.go`](../../internal/models/chat/sse_reader.go) |
| Task 类型、Queue 拓扑与默认参数 | [`internal/types/task.go`](../../internal/types/task.go) |
| 六组 Asynq Server 与 Handler 路由 | [`internal/router/task.go`](../../internal/router/task.go) |
| 业务 Task 的 Queue、Timeout 与 Retry | [`internal/application/service/knowledge_task_options.go`](../../internal/application/service/knowledge_task_options.go) |
| 文档后处理 fan-out 与入队补偿 | [`internal/application/service/knowledge_post_process.go`](../../internal/application/service/knowledge_post_process.go) |
| 子任务原子计数与最终状态晋升 | [`internal/application/repository/knowledge.go`](../../internal/application/repository/knowledge.go) 中的 `FinalizeSubtask` |
| queued/active/retry 任务取消与竞态收敛 | [`internal/router/task_inspector.go`](../../internal/router/task_inspector.go) |
| Dead Letter 中间件与记录 | [`internal/middleware/asynqdl/asynqdl.go`](../../internal/middleware/asynqdl/asynqdl.go) |
| Durable Pending Op 与 ClaimBatch | [`internal/application/repository/task_queue.go`](../../internal/application/repository/task_queue.go) |
| Lite 同步执行器 | [`internal/router/sync_task.go`](../../internal/router/sync_task.go) |
| Runtime | [`internal/runtime`](../../internal/runtime) |
| Worker 池容量与治理说明 | [`docs/worker-pool-governance.md`](../../docs/worker-pool-governance.md) |

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

## 9.1 RAG 评测

| 问题 | 入口 |
|---|---|
| QA 数据样本 | [`internal/types/dataset.go`](../../internal/types/dataset.go) |
| 评测任务与指标结构 | [`internal/types/evaluation.go`](../../internal/types/evaluation.go) |
| 评测任务编排 | [`internal/application/service/evaluation.go`](../../internal/application/service/evaluation.go) |
| 搜索、Rerank 与生成结果 Hook | [`internal/application/service/metric_hook.go`](../../internal/application/service/metric_hook.go) |
| 指标算法实现 | [`internal/application/service/metric`](../../internal/application/service/metric) |
| 评测 API | [`internal/handler/evaluation.go`](../../internal/handler/evaluation.go) |

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
