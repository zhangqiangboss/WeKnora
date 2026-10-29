# 第 3 章　Token、上下文、消息与 Prompt

[返回全书目录](../README.md) · [上一篇：把大语言模型看成一种不确定的计算组件](02-llm-as-a-component.md) · [下一篇：模型能力、成本、速度与选择](04-model-selection.md)

## 本章要解决的问题

调用 LLM 时，我们表面上发送的是文字，模型实际处理的是 Token 序列。系统所说的“记忆”“参考资料”和“规则”，最终都必须进入一次请求的有限上下文。

理解 Token 和上下文预算，是理解多轮对话、RAG、Agent、成本和长文本失败的共同基础。

## Token 不是字，也不是单词

Tokenizer 会把字符串编码成模型词表中的离散单元。一个 Token 可能是：

- 一个常见英文单词；
- 单词的一部分；
- 一个汉字或几个字符；
- 标点、空格或格式符号；
- 代码中的常见片段。

不同模型族可能使用不同 Tokenizer，所以不存在“一个汉字固定等于一个 Token”或“一个 Token 固定四个字符”的普遍公式。

工程中应区分：

- **精确计数**：用目标模型对应的 Tokenizer，或读取供应商响应中的 Usage；
- **近似估算**：在调用前快速判断预算，必须留安全余量。

WeKnora 的 Agent Token 估算器使用 `cl100k_base`，并明确将它视为近似值；完整调用后的供应商 Usage 才是主要事实来源。实现见 [`internal/agent/token/estimator.go`](../../internal/agent/token/estimator.go)。

切块器为了不引入完整 Tokenizer，则按语言使用保守字符比例估算，见 [`internal/infrastructure/chunker/tokens.go`](../../internal/infrastructure/chunker/tokens.go)。二者服务的场景不同：前者管理对话上下文，后者保护 Embedding 模型的输入上限。

## 上下文窗口是一份总预算

模型的上下文窗口不是只留给用户问题。一次请求的总预算大致是：

```text
系统消息
+ 对话历史
+ 当前问题
+ RAG 检索证据
+ 工具定义
+ 工具调用与结果
+ 图片等多模态占用
+ 期望输出
≤ 模型上下文上限
```

如果模型支持 32K 上下文，不表示可以放入 32K 输入后仍生成长答案。应用必须为输出保留空间。

可以用一个简单预算式：

```text
可用于动态上下文的预算
= 模型窗口
- 最大输出预算
- 固定系统提示
- 工具 Schema
- 安全余量
```

动态上下文再在历史、检索证据和附件之间分配。

这和 JVM 内存不同：超过预算通常不是变慢，而是请求被拒绝、前文被隐式截断，或应用主动丢弃内容。

### 手工推演：16K 窗口为什么不能装进 16K 的资料

假设模型窗口为 16,000 Token，产品要求最多生成 2,000 Token；固定 System Prompt 与工具 Schema 占 1,500 Token，安全余量取 500 Token。真正可分给历史、当前问题和检索证据的只有 `16,000 - 2,000 - 1,500 - 500 = 12,000` Token。

若历史为 5,000、当前问题为 300、候选证据为 10,000，合计 15,300，必须主动处理 3,300 Token，而不是碰运气交给供应商裁剪。一个可解释的策略可以先去重/压缩候选证据，再缩减低价值历史，同时始终保留当前问题、系统规则和本轮完整的 Tool Call—Tool Result 组。预算表让“为什么这段资料没被带入”成为可诊断的产品行为。

## 长上下文不等于可以不管理上下文

即使模型窗口很大，也不能把所有内容无差别塞进去：

- 输入 Token 仍有费用和延迟；
- 过多无关内容会稀释关键证据；
- 旧历史可能与当前任务冲突；
- 工具输出可能包含重复日志或不可信指令；
- 关键信息在长上下文中不一定总能被稳定利用。

因此，上下文工程不是“怎样填满窗口”，而是“怎样让有限注意力集中在当前任务最需要的信息上”。

## 消息比字符串多一层语义

WeKnora 的 [`chat.Message`](../../internal/models/chat/chat.go#L47) 包含角色、文本、多模态内容、工具调用、工具结果以及推理内容等字段。

角色决定内容在对话协议中的意义：

| 角色 | 工程含义 |
|---|---|
| `system` | 应用给模型的目标、规则和边界 |
| `user` | 用户输入或当前任务 |
| `assistant` | 模型历史回答，或包含工具调用意图的消息 |
| `tool` | 应用执行工具后返回给模型的观察结果 |

工具调用与工具结果必须正确配对。压缩历史时如果保留了 `tool` 结果，却删除对应的 assistant tool call，某些供应商会拒绝请求，模型也失去动作语境。

WeKnora 的 Agent 压缩逻辑会把 assistant tool call 与随后 tool result 组成不可拆分的逻辑组，见 [`internal/agent/token/compress.go`](../../internal/agent/token/compress.go)。

## Prompt 的四个层次

把 Prompt 理解成一段字符串模板太窄。可以把上下文构造分成四层：

### 1. 行为层

定义模型扮演什么角色、遵守什么边界、如何处理无答案和引用。

### 2. 任务层

描述当前要完成的具体任务，例如“根据证据回答问题”或“提取结构化字段”。

### 3. 证据层

放入 RAG 片段、附件、网页、图片描述和工具结果。这部分通常是动态的，也是 Prompt Injection 风险最高的部分。

### 4. 输出层

规定语言、格式、JSON Schema、引用形式和长度。

系统的最终消息可能由多份模板和运行状态拼装，而不是由一个 `prompt.txt` 决定。

## RAG 中的上下文构造

WeKnora 在 `INTO_CHAT_MESSAGE` 阶段将以下内容组合为模型消息：

- System Prompt 与摘要策略；
- 当前查询和可能的查询改写；
- 多轮历史；
- 合并、过滤后的知识片段；
- 文档标题、位置和来源元数据；
- 引用内容、临时附件和图片描述；
- 无匹配时的回退策略。

入口见 [`internal/application/service/chat_pipeline/into_chat_message.go`](../../internal/application/service/chat_pipeline/into_chat_message.go)。

这解释了为什么调试时必须查看“最终渲染的上下文”：搜索命中了正确 Chunk，不代表它最终进入了模型；进入了模型，也不代表来源边界表达清楚。

## 多轮对话不是无限追加历史

普通 RAG 管线使用 `MaxRounds` 控制加载历史轮数。`LOAD_HISTORY` 插件按轮次读取消息，见 [`load_history.go`](../../internal/application/service/chat_pipeline/load_history.go)。

Agent 的对话更长，还会累积工具定义和工具输出，因此需要进一步治理：

1. 使用供应商 Usage 获取已消耗 Token；
2. 对新追加消息做增量估算；
3. 达到窗口一定比例时移除较旧历史；
4. 保留 System Prompt、当前轮以及完整工具调用组；
5. 更复杂场景可将旧历史总结为记忆。

当前 Agent 压缩在估算超过上下文窗口 80% 时触发，具体值是 WeKnora 的实现选择，不是所有应用都应采用的标准答案。

## 截断、删除与总结不是同一件事

### 截断

保留一段内容的头部/尾部，丢弃中间或超长部分。适合日志和工具输出，但可能丢失关键证据。

### 删除

按轮次或优先级完全移除旧消息。简单、便宜，但会失去长期信息。

### 总结

用模型将旧历史压缩为更短记忆。保留语义更好，但增加调用成本，并可能把错误写进长期记忆。

WeKnora 三种方式都有涉及：工具输出限制、上下文裁剪，以及 Agent memory consolidator。阅读时要辨认它们各自改变了什么信息。

## 上下文预算的优先级

一个实用的优先顺序是：

1. 保留安全规则和任务目标；
2. 保留当前用户问题；
3. 保留直接支持答案的证据；
4. 保留当前轮工具调用的完整因果链；
5. 保留必要的近期历史；
6. 压缩或删除重复、陈旧和低价值内容。

这不是固定算法，但比“从最早消息开始机械删除”更能表达业务价值。

## Prompt Injection 的上下文边界

RAG 文档和网页属于数据，不应自动升级为系统指令。一份被检索到的文档可能写着“忽略之前所有规则”，应用必须通过消息角色、模板边界、工具权限和输出校验降低它对系统行为的影响。

重要原则是：

```text
可信指令与不可信证据必须在结构和权限上区分
```

仅仅在 Prompt 中说“不要被攻击”并不足以替代工具授权和数据隔离。

## 源码观察点

1. [`internal/models/chat/chat.go`](../../internal/models/chat/chat.go#L20)：消息、工具和生成参数。
2. [`internal/application/service/chat_pipeline/load_history.go`](../../internal/application/service/chat_pipeline/load_history.go)：普通问答如何限制历史轮次。
3. [`internal/application/service/chat_pipeline/into_chat_message.go`](../../internal/application/service/chat_pipeline/into_chat_message.go)：RAG 上下文如何进入消息。
4. [`internal/agent/token/estimator.go`](../../internal/agent/token/estimator.go)：估算与供应商 Usage 的关系。
5. [`internal/agent/token/compress.go`](../../internal/agent/token/compress.go)：压缩时保留哪些语义边界。
6. [`internal/agent/memory/consolidator.go`](../../internal/agent/memory/consolidator.go)：历史总结的实现入口。

## 常见误区

1. **上下文越长越好。** 无关信息会增加成本并干扰模型。
2. **字符数可以精确换算 Token。** 只能近似，且依赖模型与语言。
3. **把数据库聊天记录全部发给模型。** 持久化历史与本次模型上下文不是同一个集合。
4. **RAG 命中多少就塞多少。** 仍需合并、预算与优先级。
5. **系统 Prompt 可以代替权限控制。** 模型规则不是安全边界。

## 本章小结

- Token 是模型处理和计费单位，不等于固定数量的字或单词。
- 上下文窗口是系统消息、历史、证据、工具和输出共享的总预算。
- Prompt 工程在成熟系统中更接近上下文工程。
- 多轮历史、RAG 证据和工具结果必须按价值治理。
- 压缩时要保护消息协议和工具调用的因果完整性。
- Usage 是调用后的主要事实，调用前的 Tokenizer 或字符比例只是估算。

### 可选实践

手工列出一次知识问答可能进入上下文的全部组成，并给它们分配一个假想的 16K Token 预算。重点解释优先级，不要求精确计算。

下一章：[模型能力、成本、速度与选择](04-model-selection.md)
