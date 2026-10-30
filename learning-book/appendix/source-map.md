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
| Agent 上下文与回合推进 | [`internal/agent/engine.go`](../../internal/agent/engine.go) |
| 长期记忆归并 | [`internal/application/service/memory/consolidate.go`](../../internal/application/service/memory/consolidate.go) |
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
| OpenAI 兼容 Chat 协议 | [`internal/models/api`](../../internal/models/api) |
| Chat Provider 差异与元数据 | [`internal/models/providers`](../../internal/models/providers) |
| Ollama Chat | [`internal/models/chat/ollama.go`](../../internal/models/chat/ollama.go) |
| Embedding 接口和工厂 | [`internal/models/embedding/embedder.go`](../../internal/models/embedding/embedder.go) |
| Embedding 批处理 | [`internal/models/embedding/batch.go`](../../internal/models/embedding/batch.go) |
| Rerank 接口和工厂 | [`internal/models/rerank/reranker.go`](../../internal/models/rerank/reranker.go) |
| VLM 接口和工厂 | [`internal/models/vlm/vlm.go`](../../internal/models/vlm/vlm.go) |
| ASR 接口和工厂 | [`internal/models/asr/asr.go`](../../internal/models/asr/asr.go) |
| Provider 元数据定义 | [`internal/models/providers/definition.go`](../../internal/models/providers/definition.go) |
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
| 知识检索工具 | [`internal/agent/tools/search_knowledge.go`](../../internal/agent/tools/search_knowledge.go) |
| Web 搜索工具 | [`internal/agent/tools/web_search.go`](../../internal/agent/tools/web_search.go) |
| 工具人工审批 | [`internal/agent/approval/gate.go`](../../internal/agent/approval/gate.go) |
| Skill 资源按需读取 | [`internal/agent/tools/read_file.go`](../../internal/agent/tools/read_file.go) |
| Skill 脚本执行 | [`internal/agent/tools/shell_exec.go`](../../internal/agent/tools/shell_exec.go) |

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
| 上游 OpenAI 兼容协议与流处理 | [`internal/models/api`](../../internal/models/api) |
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
| Worker 池 Queue 与 Handler 路由 | [`internal/router/task.go`](../../internal/router/task.go) |

## 9. 可观测性和治理

| 问题 | 入口 |
|---|---|
| Request ID、访问日志、延迟与脱敏 | [`internal/middleware/logger.go`](../../internal/middleware/logger.go) |
| Langfuse 配置、采样与批量 Export | [`internal/tracing/langfuse/config.go`](../../internal/tracing/langfuse/config.go) 与 [`manager.go`](../../internal/tracing/langfuse/manager.go) |
| HTTP Root Trace 与路由白名单 | [`internal/tracing/langfuse/middleware.go`](../../internal/tracing/langfuse/middleware.go) |
| Trace、Span、Generation 生命周期 | [`internal/tracing/langfuse/tracer.go`](../../internal/tracing/langfuse/tracer.go) |
| W3C traceparent 与 Asynq 传播 | [`internal/tracing/langfuse/asynq.go`](../../internal/tracing/langfuse/asynq.go) |
| Chat Token、Cache Usage 与流式 TTFT | [`internal/models/chat/langfuse_wrapper.go`](../../internal/models/chat/langfuse_wrapper.go) |
| Embedding/Rerank/VLM/ASR Usage | [`internal/models/embedding/langfuse_wrapper.go`](../../internal/models/embedding/langfuse_wrapper.go)、[`internal/models/rerank/langfuse_wrapper.go`](../../internal/models/rerank/langfuse_wrapper.go)、[`internal/models/vlm/langfuse_wrapper.go`](../../internal/models/vlm/langfuse_wrapper.go)、[`internal/models/asr/langfuse_wrapper.go`](../../internal/models/asr/langfuse_wrapper.go) |
| Agent Execute、Round 与 Tool Span | [`internal/agent/engine.go`](../../internal/agent/engine.go) 与 [`internal/agent/act.go`](../../internal/agent/act.go) |
| 文档处理持久业务 Span | [`internal/types/knowledge_span.go`](../../internal/types/knowledge_span.go) 与 [`internal/application/service/knowledge_span_tracker.go`](../../internal/application/service/knowledge_span_tracker.go) |
| 审计事件、Outcome 与资源范围 | [`internal/types/audit_log.go`](../../internal/types/audit_log.go) 与 [`internal/application/service/audit_log.go`](../../internal/application/service/audit_log.go) |
| Langfuse 配置与接入代码 | [`internal/tracing/langfuse`](../../internal/tracing/langfuse) |
| 限流 | [`internal/ratelimit`](../../internal/ratelimit) |

## 9.1 多租户、RBAC 与 AI 安全

| 问题 | 入口 |
|---|---|
| User 与终端 Principal | [`internal/types/principal.go`](../../internal/types/principal.go) |
| JWT、API Key、目标 Tenant 与外部用户身份 | [`internal/middleware/auth.go`](../../internal/middleware/auth.go) |
| 认证状态写入 Gin/标准 Context | [`internal/middleware/auth_context.go`](../../internal/middleware/auth_context.go) |
| Tenant Role 与 Membership | [`internal/types/tenant_member.go`](../../internal/types/tenant_member.go) |
| Role、System Admin 与 Ownership Guard | [`internal/middleware/rbac.go`](../../internal/middleware/rbac.go) |
| API Key Capability 与 KB Allow-list | [`internal/types/tenant_api_key.go`](../../internal/types/tenant_api_key.go) 与 [`internal/middleware/api_key_gate.go`](../../internal/middleware/api_key_gate.go) |
| Own/Organization Share/Shared Agent 的 KB Access | [`internal/middleware/kb_access.go`](../../internal/middleware/kb_access.go) |
| Pipeline、Session 与 Sandbox Tenant Context | [`internal/types/const.go`](../../internal/types/const.go) 与 [`internal/types/context_helpers.go`](../../internal/types/context_helpers.go) |
| Agent Knowledge/Chunk/Tag Scope 再授权 | [`internal/agent/tools/scope_authorization.go`](../../internal/agent/tools/scope_authorization.go) |
| Tool 参数验证与 Human Approval | [`internal/agent/tools/registry.go`](../../internal/agent/tools/registry.go) 与 [`internal/agent/approval/gate.go`](../../internal/agent/approval/gate.go) |
| AES-GCM Secret 与密钥缺失策略 | [`internal/utils/crypto.go`](../../internal/utils/crypto.go) |
| 通用 SSRF、MCP 与 Sandbox URL Guard | [`internal/utils/security.go`](../../internal/utils/security.go)、[`internal/mcp/security.go`](../../internal/mcp/security.go)、[`internal/sandbox/url_guard.go`](../../internal/sandbox/url_guard.go) |
| 前端角色镜像（仅 UX） | [`frontend/src/stores/auth.ts`](../../frontend/src/stores/auth.ts) |
| RBAC 角色与共享实现（API Key 以当前源码为准） | [`internal/middleware/rbac.go`](../../internal/middleware/rbac.go) |
| 沙箱实现 | [`internal/sandbox`](../../internal/sandbox) |

## 9.2 部署形态与基础设施

| 问题 | 入口 |
|---|---|
| Lite 与标准版的能力边界 | [`docs/LITE.md`](../../docs/LITE.md) |
| 桌面 Lite 的启动、edition 和本地运行时 | [`cmd/desktop/main.go`](../../cmd/desktop/main.go) |
| 无 Redis 时的内存 Stream 与本进程任务执行 | [`internal/stream/factory.go`](../../internal/stream/factory.go) 与 [`internal/router/sync_task.go`](../../internal/router/sync_task.go) |
| Redis 缺席时的依赖装配 | [`internal/container/container.go`](../../internal/container/container.go) 中的 `initRedisClient` |
| 后端能力快照与 Lite Gate | [`internal/handler/deployment_capabilities.go`](../../internal/handler/deployment_capabilities.go) |
| 标准 Compose 服务、环境变量与可选 Profile | [`docker-compose.yml`](../../docker-compose.yml) |
| Helm 参数、App 部署与 PVC | [`helm/values.yaml`](../../helm/values.yaml)、[`helm/templates/app.yaml`](../../helm/templates/app.yaml) 与 [`helm/templates/pvc.yaml`](../../helm/templates/pvc.yaml) |
| RWO/RWX、多副本与升级策略 | [`helm/README.md`](../../helm/README.md) |

## 9.3 RAG 评测

| 问题 | 入口 |
|---|---|
| QA 数据样本 | [`internal/types/dataset.go`](../../internal/types/dataset.go) |
| 评测任务与指标结构 | [`internal/types/evaluation.go`](../../internal/types/evaluation.go) |
| 评测任务编排 | [`internal/application/service/evaluation.go`](../../internal/application/service/evaluation.go) |
| 搜索、Rerank 与生成结果 Hook | [`internal/application/service/metric_hook.go`](../../internal/application/service/metric_hook.go) |
| 指标算法实现 | [`internal/application/service/metric`](../../internal/application/service/metric) |
| 评测 API | [`internal/handler/evaluation.go`](../../internal/handler/evaluation.go) |

## 9.4 Go 分层与依赖装配

| 问题 | 入口 |
|---|---|
| 服务进程启动与生命周期 | [`cmd/server/main.go`](../../cmd/server/main.go) |
| 依赖图 composition root | [`internal/container/container.go`](../../internal/container/container.go) |
| Gin Engine、全局 Middleware 与能力 Route 注册 | [`internal/router/router.go`](../../internal/router/router.go) |
| 知识库 HTTP Route | [`internal/router/routes_knowledge.go`](../../internal/router/routes_knowledge.go) |
| Handler 的请求/响应边界 | [`internal/handler/knowledgebase.go`](../../internal/handler/knowledgebase.go) |
| Knowledge Base Service/Repository 契约 | [`internal/types/interfaces/knowledgebase.go`](../../internal/types/interfaces/knowledgebase.go) |
| 用例编排与 GORM 持久化 | [`internal/application/service/knowledgebase.go`](../../internal/application/service/knowledgebase.go) 与 [`internal/application/repository/knowledgebase.go`](../../internal/application/repository/knowledgebase.go) |
| Lite / Redis 两种模式下的依赖图验证 | [`internal/container/router_wiring_test.go`](../../internal/container/router_wiring_test.go) |

## 9.5 事件、管线与扩展点

| 问题 | 入口 |
|---|---|
| 进程内 Event Bus 与同步/异步语义 | [`internal/event/event.go`](../../internal/event/event.go) |
| Event 输出适配与用法示例 | [`internal/event/adapter.go`](../../internal/event/adapter.go) 与 [`internal/event/usage_example.md`](../../internal/event/usage_example.md) |
| Chat Pipeline 插件装配 | [`internal/container/container.go`](../../internal/container/container.go) 中的 `NewPlugin...` 注册 |
| 数据源 Connector 契约、可选流式能力与 Registry | [`internal/datasource/connector.go`](../../internal/datasource/connector.go) |
| 租户配置驱动的 Web Search Provider Factory | [`internal/infrastructure/web_search/registry.go`](../../internal/infrastructure/web_search/registry.go) |
| IM Adapter Factory 与通道生命周期 | [`internal/im/service.go`](../../internal/im/service.go) |
| 检索引擎 Factory/Adapter | [`internal/container/engine_factory.go`](../../internal/container/engine_factory.go) |

## 9.6 架构取舍与演进

| 问题 | 入口 |
|---|---|
| Lite/Redis 下 Task Enqueuer 与 Inspector 的装配差异 | [`internal/container/container.go`](../../internal/container/container.go) 与 [`internal/router/task_inspector.go`](../../internal/router/task_inspector.go) |
| Memory/Redis Stream 的显式选择 | [`internal/stream/factory.go`](../../internal/stream/factory.go) |
| 稳定资源引用与物理文件服务的装饰 | [`internal/application/service/file/resource_catalog.go`](../../internal/application/service/file/resource_catalog.go) |
| 启动迁移、资源清理与依赖生命周期 | [`internal/container/container.go`](../../internal/container/container.go) 与 [`internal/container/cleanup.go`](../../internal/container/cleanup.go) |

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
