# 第 20 章　知识图谱和 Wiki 生成

[返回全书目录](../README.md) · [上一篇：图片、VLM 与多模态检索](02-image-vlm-multimodal-retrieval.md)

## 本章要解决的问题

普通 RAG 把知识组织为 Chunk，擅长回答“哪些片段与问题相似”。但有些任务要求跨文档组织：

- 某个产品与哪些组件、组织和概念相关？
- 两份文档对同一个实体分别说了什么？
- 新资料进来后，如何更新一页长期维护的主题说明？
- 删除源文档时，哪些派生知识应该撤回？
- 用户怎样浏览知识，而不必每次先提出一个精确问题？

知识图谱和 Wiki 都能让知识从“片段集合”变成“组织结构”，但它们不是同一种产物：

```text
知识图谱：机器可遍历的节点—边结构
Wiki：人可阅读、可编辑、可链接、可版本化的页面网络
```

WeKnora 同时实现了两条相关但独立的加工链。本章会把它们分别还原，再讨论怎样协作。

## 为什么 Chunk 不足以表达所有知识

假设三份文档分别说：

```text
文档 A：WeKnora 使用混合检索。
文档 B：混合检索组合关键词检索与向量检索。
文档 C：RRF 可以融合不同召回列表。
```

Chunk 检索可以分别找到这些句子。若要回答“WeKnora 的混合检索怎样工作”，模型还需自行识别：

- `WeKnora` 是系统实体；
- `混合检索` 是概念；
- `关键词检索`、`向量检索` 是组成能力；
- `RRF` 与混合检索存在融合方法关系；
- 三份证据应组合成一个主题说明。

知识图谱把关系显式化；Wiki 把跨来源内容持续归并成页面。它们都是预计算的知识加工，代价是增加抽取错误、合并冲突、更新和撤回问题。

## 先区分五种容易混淆的结构

### Chunk 关系

父子 Chunk、前后相邻和共同来源属于文档结构。它们说明片段怎样组织，不一定表达业务语义。

### 实体与关系图谱

例如：

```text
(WeKnora)-[USES]->(Hybrid Search)
(Hybrid Search)-[COMBINES]->(BM25)
```

节点和边用于遍历、扩展和关系查询。

### Wiki 链接图

Wiki 页面通过 `[[slug]]` 相互链接。边表示“一页引用另一页”，不一定等于 `USES`、`PART_OF` 等业务关系。

### 分类目录

文件夹、CategoryPath 和 ParentSlug 用于浏览层级。树结构要求一个主要位置，而知识关系通常是多对多图。

### 向量近邻

相似 Chunk 或页面在向量空间接近，这是统计关系，不是事实边。相似不等于“属于”“导致”或“依赖”。

把五种结构全部塞进一个 `relations` 字段，会导致语义和更新规则混乱。

## WeKnora 的 Graph RAG 主链

当知识库开启图抽取且 Neo4j 可用时，后处理会为每个文本类 Chunk 创建独立图抽取任务：

```mermaid
flowchart LR
    A["文本 Chunk"] --> B["LLM 结构化抽取"]
    B --> C["GraphData"]
    C --> D["Node: name/attributes/chunks"]
    C --> E["Relation: node1/type/node2"]
    D --> F["Neo4j / Graph Repository"]
    E --> F
    F --> G["查询实体与相邻关系"]
    G --> H["回到证据 Chunk"]
```

[`extract.go`](../../internal/application/service/extract.go) 中的 `ChunkExtractService`：

1. 检查任务是否被更新的解析 Attempt 淘汰；
2. 检查源 Knowledge 是否已取消或删除；
3. 读取 Chunk 和知识库抽取配置；
4. 使用配置的节点类型、关系类型、样例和自定义指令构造结构化 Prompt；
5. 调用 Chat Model 提取 `GraphData`；
6. 给节点写入当前 Chunk ID 作为来源；
7. 按知识库/文档 Namespace 写入图存储。

[`GraphData`](../../internal/types/extract_graph.go) 很精简：

```text
GraphNode(name, attributes, chunks)
GraphRelation(node1, node2, type)
NameSpace(knowledge_base, knowledge)
```

这足以支持实体查询和证据回溯，但不是完整本体系统。

## 图谱抽取不是让模型自由发挥

LLM 能从文本中抽实体和关系，但输出空间必须受约束：

- 允许哪些节点类型或标签；
- 允许哪些关系类型；
- 实体名称如何规范化；
- 是否允许文本没有明确支持的推断边；
- 一个关系至少需要哪些证据；
- 如何处理否定、时间和条件；
- 输出怎样通过 JSON Schema 和业务校验。

WeKnora 的 `ExtractConfig` 提供节点、关系、示例和自定义说明。它类似轻量 Ontology：告诉模型业务关心什么。

如果只提示“请抽取所有实体与关系”，很快会出现：

- 同一个实体的大小写、简称和全称形成多个节点；
- `使用`、`依赖`、`包含`被随机混用；
- 模型把上下文常识当成文档事实；
- 每个普通名词都变成节点；
- 图规模膨胀但查询价值很低。

图谱的价值来自稳定 Schema 和可追溯证据，不来自节点数量。

### 从一句文本到带证据的边，发生了什么

原文：

> 订单服务通过 Kafka 向库存服务发布库存预留事件。

若 Schema 只允许 `Service` 节点和 `PUBLISHES_TO` 关系，受控抽取可以产生：

```text
Node(order-service, type=Service)
Node(kafka, type=ServiceOrMiddleware)
Node(inventory-service, type=Service)
Relation(order-service, PUBLISHES_TO, kafka, source_chunk=C42)
Relation(kafka, DELIVERS_TO, inventory-service, source_chunk=C42)
```

这里最重要的字段不是漂亮的三元组文字，而是 `source_chunk=C42`。用户问“库存为什么会延迟更新”时，图可以帮助沿关系扩展候选，但最终回答仍应回到 C42 及其上下文确认“通过 Kafka”是否是当前版本架构、是否存在重试或失败条件。没有来源的图谱像没有引用的摘要，难以纠错、撤回和建立信任。

## 实体消歧是图谱质量的核心

以下名称可能指向同一实体：

```text
WeKnora / weknora / 腾讯 WeKnora
RAG / 检索增强生成 / Retrieval-Augmented Generation
```

也可能同名不同实体：

```text
Apple：公司或水果
Java：语言、岛屿或咖啡
```

实体归一化通常需要：

- canonical ID；
- 标准名称；
- aliases；
- 类型约束；
- 来源 Chunk；
- 合并置信度；
- 人工拆分/合并能力。

只拿标题字符串作全局唯一键很方便，但在多语言、多租户和专业领域会产生误合并。Namespace 可以防止跨知识库污染，却不能解决一个知识库内部的同名和别名。

## 图查询最终仍要回到原始证据

Graph RAG 的理想路径不是“把图中三元组当绝对事实直接回答”，而是：

```text
查询 → 识别实体 → 图遍历/关系扩展 → 找到来源 Chunk
    → 与文本检索结果融合 → Rerank → 基于证据回答
```

WeKnora 的图节点带 Chunk ID，图搜索结果最终映射回 SearchResult。普通 RAG 管线中的实体抽取与 `SearchNode` 可以获得图相关片段；Agent 的 `query_knowledge_graph` Tool 也会通过知识库搜索入口返回去重后的证据结果，并再次应用 SearchTargets 范围过滤。

这体现了一条重要原则：

> 图谱是证据路由器，不应成为失去来源的事实黑盒。

如果关系无法指回源文档和 Chunk，用户无法验证，删除源资料时也无法准确撤回。

## `graphBuilder` 与当前逐 Chunk 抽取路径

[`graph.go`](../../internal/application/service/graph.go) 还展示了一种批量构图思路：

1. 并发从每个 Chunk 抽实体；
2. 按一组 Chunk 批量抽关系；
3. 计算关系权重和实体度数；
4. 建立 Chunk 之间的关系图；
5. 可生成 Mermaid 视图。

当前生产后处理主线更直接：每个 Chunk 的 `ChunkExtractService` 生成 `GraphData` 并写入图仓库。阅读时不要因为仓库里存在 `graphBuilder`，就默认所有方法都位于当前在线主调用链。

这也是读开源项目的重要方法：区分“存在的能力实现”“被依赖注入的接口”和“当前业务入口实际调用的路径”。

## Wiki 不是把摘要保存成 Markdown

WeKnora 的 [`WikiPage`](../../internal/types/wiki_page.go) 是长期知识产物，包含：

- `Slug` 与人类可读标题；
- 页面类型与状态；
- Markdown 正文和一句话摘要；
- aliases；
- FolderID、CategoryPath、WikiPath；
- `SourceRefs` 文档级来源；
- `ChunkRefs` 片段级证据；
- `InLinks` / `OutLinks`；
- Version、编辑来源和编辑者；
- 历史 Revision。

自动入库会生成四类基础页面：

| 页面类型 | 作用 |
|---|---|
| `summary` | 一份文档的入口摘要 |
| `entity` | 人、组织、产品、地点等实体页 |
| `concept` | 主题、方法或领域概念页 |
| `index` | 整个 Wiki 的入口与概览 |

`synthesis` 和 `comparison` 不由入库流水线自动创建，而由 Agent 在用户明确分析或比较时通过 Wiki 写工具生成。这避免批处理擅自制造高层推断页面。

## Wiki 生成为什么使用 Map/Reduce

一份文档可能贡献多个页面，多份文档又可能贡献同一页面。例如三份文档都讨论 RAG，应共同更新 `concept/rag`，而不是生成三页重复内容。

WeKnora 使用 Map/Reduce 思路：

```text
Map（按文档并行）
  文档 A → summary/A + entity/X 更新 + concept/RAG 更新
  文档 B → summary/B + concept/RAG 更新 + concept/RRF 更新

Shuffle（按 Slug 分组）
  concept/RAG → [A 的贡献, B 的贡献]

Reduce（按 Slug 合并）
  读取 concept/RAG 旧页面
  + 当前批次新贡献
  - 已删除文档的旧贡献
  → 新页面版本
```

这比“每个文档直接写页面”更能控制并发冲突和跨来源合并。

## Map 阶段：先找候选，再找证据

[`mapOneDocument`](../../internal/application/service/wiki_ingest_batch.go) 的主要步骤是：

1. 检查文档是否仍存在；
2. 读取并按顺序重建文本内容；
3. 对图像主导文档加入可用的图片语义；
4. 内容不足时拒绝 LLM 抽取，避免从文件名或图片标记幻觉；
5. Pass 0 抽取候选实体、概念与稳定 Slug；
6. 并行生成文档 Summary，并把候选分配到真正支持它的 Chunk；
7. 将 Chunk 引用合并回候选；
8. 输出一组按 Slug 聚合的 `SlugUpdate`。

候选抽取与 Chunk Citation 分开很关键。第一遍回答“有哪些主题值得建页”，第二遍回答“哪些具体片段支持每个页面”。若一步让模型同时写长页面和精确列出所有来源，Prompt 容易过载，引用也更难校验。

Citation 阶段使用短句柄映射 Chunk，模型输出后再解析回真实 Chunk UUID。未知句柄被拒绝，内部句柄不会泄漏到最终 Markdown。

## Summary 页面为什么只有文档级来源

实体/概念页可能只使用文档中的几个 Chunk，因此 `ChunkRefs` 很有意义。文档 Summary 是基于整份重建内容生成的概述，WeKnora 将其视为文档级派生产物，保留 `SourceRefs`，但不伪装成几条精确 Chunk 引用。

这是证据粒度诚实性：

```text
页面基于局部证据 → ChunkRefs
页面基于整份文档 → SourceRefs
```

不要为了 UI 都显示页码，给全篇摘要随便挂第一个 Chunk。

## Reduce 阶段：同一 Slug 的读—改—写

Map 结果按 Slug 分组后并行 Reduce。对一个页面，Reduce 需要同时处理：

- 新来源的增加；
- 已有来源的保留；
- 被删除来源的撤回；
- 页面内容、摘要、aliases 和引用更新；
- 新建、更新或删除页面；
- 用户已有分类和编辑结果的保护。

同一个 Slug 可能被多个并行批次更新，因此 WeKnora 使用按 Slug 的锁，而不是把整个知识库串行化。标准模式允许同一 KB 多个批次并发 claim 不同任务；Lite 模式则用进程内 KB 锁简化并发。

这是一种细粒度并发控制：

```text
批次级并行提高吞吐
Slug 级串行保证同一页面读改写一致
```

锁获取失败或 Reduce 未落地时，贡献文档不能直接从 pending 队列删除，而要重新排队；否则“Map 成功但页面没更新”的贡献会永久丢失。

## 新增比删除容易，撤回才是难点

生成式知识库最大的坑不是第一次生成，而是源文档更新或删除。

若 `concept/rag` 同时来自 A、B、C 三份文档，删除 B 后不能：

- 整页删除，因为 A、C 仍然有效；
- 只删 `SourceRefs`，因为正文里可能仍含 B 独有事实；
- 完全重跑所有文档，成本可能很高。

WeKnora 为删除创建 `retract` 更新：

- 单一来源的过时页面可删除；
- 多来源页面由 Reduce 使用剩余来源与待删除内容重新整理；
- 文档重新解析后，对旧 Slug 集与新 Slug 集做差，产生 `retractStale`；
- 删除或重命名页面时清理死链和反向链接；
- Wiki pending 操作支持失败计数、重试和死信式终止。

这与物化视图维护非常相似：Wiki 是源文档的派生视图，必须支持增量更新和来源撤销。

## 去重、Slug 和别名

Wiki 页面需要稳定地址。候选实体/概念经过 Slug 规范化和去重，避免：

```text
concept/rag
concept/retrieval-augmented-generation
concept/检索增强生成
```

变成三个互不相干页面。

WeKnora 的批量去重会从已有实体/概念页中选候选，再让 LLM 判断合并；字符串/Gram 相似度用于缩小候选面，不单独决定事实身份。合并后保留 aliases。

稳定 Slug 的原则包括：

- Slug 是地址，不等于显示标题；
- 标题、语言和别名可变化，Slug 尽量稳定；
- 合并与拆分必须保留重定向或显式迁移；
- 不同页面类型的命名空间分开；
- LLM 建议 Slug 后仍由确定性代码规范化与校验。

## 分类树与链接图为何同时存在

用户浏览 Wiki 需要目录树，而真实知识关系是图。WeKnora 将 Folder 作为页面位置的事实源，并缓存 CategoryPath、WikiPath 和 Depth 供列表与排序；Map 完成后，对整个批次统一规划分类，再在 Reduce 前创建文件夹。

批量规划而不是每页独立分类，可以减少同义文件夹：

```text
AI / RAG
人工智能 / 检索增强
LLM 应用 / RAG 技术
```

分类只在页面还没有人工位置时应用，避免每次重新入库都打乱用户整理的目录。

页面之间还通过 `[[slug|display]]` 链接。自动 Linkify 会避开代码块、已有链接、图片和不安全的子串边界，并维护 InLinks/OutLinks。目录回答“放在哪里”，链接回答“与谁相关”。

## Finalize：批次之后的全局收敛

每个小批次结束都重建 Index、清理全库死链和注入交叉链接，会把一次大批量导入放大成大量重复工作。WeKnora 将这些 KB 级操作放入去抖的 `wiki:finalize`：

- 重建 Index 页介绍；
- 清理死链接；
- 注入跨页面链接；
- 修剪空目录；
- 合并一批变更后只运行一次全局收敛。

新页面正文在 Reduce 后先从 Draft 切到 Published，使用户尽快可见；全局导航稍后最终一致。这是对可见延迟与一致性成本的取舍。

## Wiki 是可编辑资产，不是一次性模型输出

页面可能由 Pipeline、Agent 或用户修改。WeKnora 记录：

```text
Version
LastEditSource = pipeline | agent | user | revert
LastEditorID
WikiPageRevision
```

只有用户可见内容改变时才增加版本；纯链接维护和元数据刷新不制造虚假版本。历史版本可以回退，并对自动流水线产生的 Revision 设置保留上限。

生成管线还应尊重人工编辑：

- 自动更新不要无条件覆盖人工整理；
- 分类规划不搬动已有人工页面；
- 内容合并要知道当前版本由谁编辑；
- 回退要形成新版本，而不是抹掉历史；
- Agent 写工具需要权限、范围和审计。

如果没有这些规则，“自动 Wiki”会成为不断重写用户工作的内容机器人。

## Wiki 的 Agent 工具面

WeKnora 为具备 Wiki Capability 的 Agent 提供：

- 搜索与读页；
- 精读源文档；
- 创建/覆盖和局部替换；
- 重命名、删除；
- 标记、读取和处理页面问题。

Agent 可以在用户请求下创建 `synthesis` 或 `comparison` 页面，但工具只在 Wiki KB 范围内注册，写工具还受到共享 Agent 只读策略和对象级范围约束。

页面写入不是普通字符串更新：重命名要改入链，删除要处理死链，局部替换要基于当前内容，失败时需要补偿已修改的关联页面。这正是第 17 章所讲的副作用治理。

## Wiki Lint 与健康度

随着页面增加，Wiki 会出现：

- 孤儿页；
- 断链；
- 过时来源；
- 缺少交叉引用；
- 空内容；
- 重复 Slug。

[`wiki_lint.go`](../../internal/application/service/wiki_lint.go) 分页检查页面、链接和来源，并输出问题严重度、统计与健康分。部分机械问题可自动修复，例如清理断链或处理空页面；事实冲突和错误合并应进入人工/Agent 问题工作流。

生成式内容质量不能只在生成时评估，还要持续维护。Wiki Lint 类似代码静态检查：它不能证明事实正确，但能发现结构性腐化。

## 图谱与 Wiki 怎样协作

两者可以互补：

| 维度 | 知识图谱 | Wiki |
|---|---|---|
| 基本单元 | 节点与边 | 页面与链接 |
| 强项 | 关系遍历、多跳扩展、实体邻居 | 阅读、编辑、主题归并、浏览 |
| 结构 | 类型化图 | Markdown 内容 + 链接图 + 分类树 |
| 来源 | 节点/边关联 Chunk | SourceRefs / ChunkRefs |
| 更新 | 图节点边增删合并 | Map/Reduce 页面重写与 Revision |
| 查询 | Entity/Path/Graph RAG | 搜索、读页、索引浏览 |
| 主要风险 | 实体误合并、关系幻觉、图爆炸 | 内容覆盖、来源漂移、死链、陈旧页面 |

可行的组合方式是：

- 图谱帮助 Wiki 推荐相关页面和交叉链接；
- Wiki 页面提供图节点的人类可读解释；
- Graph RAG 与 Wiki Search 作为独立检索通道参与 Agent；
- 两者最终都回到原始 Chunk 证据；
- 不把 Wiki 链接自动解释成业务关系边。

## 一致性不是“强一致或最终一致”的单选题

应按数据类型选择：

| 数据 | 推荐一致性 |
|---|---|
| 同一 Slug 页面正文读改写 | 需要串行/乐观锁保护 |
| SourceRefs/ChunkRefs 与正文提交 | 尽量原子或同一业务事务 |
| 页面发布状态 | Reduce 成功后立即可见 |
| Index 页与全库交叉链接 | 可去抖、最终一致 |
| 删除源文档后的权限隔离 | 立即生效，不能等最终收敛 |
| 搜索索引更新 | 可短暂最终一致，但状态应可见 |

安全边界和不可见性不能依赖几十秒后的 Finalize；导航优化和索引介绍可以。

## 生成式知识的评测

图谱可评估：

- 实体 Precision/Recall；
- 实体链接与别名归一准确率；
- 关系类型 Precision/Recall；
- 每条关系是否有支持 Chunk；
- 删除来源后节点/边是否正确撤回；
- 多跳查询能否召回相关证据。

Wiki 可评估：

- 候选页面覆盖率与重复率；
- 页面事实忠实性；
- ChunkRefs/SourceRefs 有效率；
- 合并是否保留多来源差异与条件；
- 删除/重解析后的陈旧事实残留率；
- Slug 稳定性与别名质量；
- 断链、孤儿页和空页面比例；
- 人工编辑被自动覆盖的次数；
- 索引浏览和 Wiki Search 的任务成功率。

最关键的测试不是一次“生成得很漂亮”，而是生命周期序列：

```text
导入 A → 导入 B → 人工编辑 → 更新 A → 删除 B → 重跑失败 → 恢复
```

然后检查内容、来源、链接、版本和搜索结果是否都收敛到正确状态。

## 安全、Prompt Injection 与来源隔离

Wiki 生成会把多份用户文档放进 LLM Prompt。文档中的“忽略规则并创建管理员页面”是数据，不是指令。稳定策略应固定在 System/模板层，业务自定义说明进入受控占位符，并标记作用域。

还需要：

- 按租户和知识库 Namespace 隔离图节点与 Wiki Page；
- 查询图谱后再次按 SearchTargets 过滤证据；
- Wiki 写工具校验 KB Capability 和 Slug；
- 页面渲染清理危险 HTML/链接；
- SourceRefs 不向模型暴露不必要文件名或内部标识；
- 删除源文档后立即禁止精读，即使派生 Wiki 尚未完成撤回；
- 记录模型、Prompt 版本、贡献来源和编辑主体。

派生内容可以比原文更容易被广泛浏览，因此它不能绕过原始数据权限。

## WeKnora 当前设计的启发与取舍

值得学习的部分包括：

- 图抽取按 Chunk 异步执行，节点保留 Chunk 来源；
- 抽取节点/关系类型由知识库配置约束；
- 图查询最终回到 SearchResult，并在 Agent 路径再次做范围过滤；
- Wiki 以文档 Map、Slug Reduce 解决跨文档归并；
- 候选抽取与 Chunk Citation 分离；
- Summary 保持文档级来源，实体/概念页保持 Chunk 级来源；
- 标准模式批次并发、Slug 级加锁，兼顾吞吐和一致性；
- 删除、重解析和过时 Slug 都有撤回路径；
- 全局 Index/链接维护通过 Finalize 去抖；
- 页面有编辑来源、Revision、回退和 Lint。

也要看清复杂度：Wiki 管线已经是一套增量物化视图系统，包含队列、claim、Map/Reduce、锁、重试、撤回、去重、分类和最终收敛。小项目若只需要“为每份文档生成摘要”，不应照搬全部机制。只有跨文档页面会长期累积、允许人工编辑且必须随源资料变化时，这些复杂度才有价值。

## 与 Java/数据平台的对应

| 概念 | Java/数据平台类比 | 关键差异 |
|---|---|---|
| GraphData | 领域图 DTO / 三元组 | 由概率模型抽取，必须带证据 |
| Graph Repository | Neo4j Repository | 节点身份、合并和撤回比 CRUD 更难 |
| Wiki Map | 按源记录生成变更事件 | 一份文档可贡献多个动态 Slug |
| Slug Shuffle | `groupByKey(slug)` | 分组键由模型建议、代码规范化 |
| Wiki Reduce | 物化视图增量聚合 | 合并正文也由 LLM参与，非纯函数 |
| SourceRefs/ChunkRefs | Data Lineage | 同时用于展示、撤回和权限校验 |
| Wiki Finalize | Debounced Batch Compaction | 收敛索引、链接和目录，不阻塞每批发布 |
| Revision | Audit Log / Version Table | 要区分 Pipeline、Agent 和人工编辑 |
| Wiki Lint | 静态检查/数据质量规则 | 结构问题可确定检查，事实问题仍需证据评估 |

可以把知识图谱看成“带来源的语义索引”，把 Wiki 看成“由 LLM 辅助维护的增量物化知识视图”。

## 推荐源码阅读顺序

### Graph RAG

1. [`extract_graph.go`](../../internal/types/extract_graph.go)：先看节点、关系和 Namespace 的最小模型。
2. [`retriever_graph.go`](../../internal/types/interfaces/retriever_graph.go)：理解 Add/Del/Search 的图仓库边界。
3. [`extract.go`](../../internal/application/service/extract.go)：跟踪逐 Chunk 抽取、来源绑定和异步状态。
4. [`search_entity.go`](../../internal/application/service/chat_pipeline/search_entity.go)：看在线问题怎样查询图节点。
5. [`query_knowledge_graph.go`](../../internal/agent/tools/query_knowledge_graph.go)：看 Agent 图查询、KB 限制与结果范围过滤。
6. [`graph.go`](../../internal/application/service/graph.go)：最后阅读批量构图、权重和 Chunk 关系算法，区分它与当前主线。

### Wiki

1. [`wiki_page.go`](../../internal/types/wiki_page.go)：先掌握页面、来源、链接、版本与配置。
2. [`wiki_ingest.go`](../../internal/application/service/wiki_ingest.go)：看 pending 操作、任务触发、锁、重试和核心数据结构。
3. [`wiki_ingest_batch.go`](../../internal/application/service/wiki_ingest_batch.go)：跟踪 Map、按 Slug 分组、Reduce 与 Finalize。
4. [`wiki_ingest_cite.go`](../../internal/application/service/wiki_ingest_cite.go)：理解候选页面怎样绑定真实 Chunk。
5. [`wiki_ingest_dedup.go`](../../internal/application/service/wiki_ingest_dedup.go)：看候选去重和已有页面匹配。
6. [`wiki_ingest_taxonomy.go`](../../internal/application/service/wiki_ingest_taxonomy.go)：看批量分类与文件夹计划。
7. [`wiki_page.go`](../../internal/application/service/wiki_page.go)：看版本、链接、搜索、Graph View 和目录维护。
8. [`wiki_lint.go`](../../internal/application/service/wiki_lint.go)：最后看持续质量检查与自动修复边界。

第一次阅读建议只跟踪两个文档共同贡献 `concept/rag` 的路径：两次 Map 怎样产生相同 Slug，Reduce 怎样合并，SourceRefs/ChunkRefs 怎样形成。再删除其中一份文档，观察 retract 如何保留另一来源。

## 常见误区

1. **知识图谱就是 Wiki 页面之间的链接。** 一个是类型化事实关系，一个是引用关系。
2. **向量相似度可以直接生成事实边。** 相似只表示接近，不等于业务关系成立。
3. **图节点越多，知识越丰富。** 无约束抽取会制造噪声、别名重复和图爆炸。
4. **图查询可以直接回答，不必回原文。** 抽取关系也可能幻觉，必须保留证据 Chunk。
5. **Wiki 就是每篇文档生成一份摘要。** 真正难点是跨文档同页归并、链接和生命周期维护。
6. **每个文档并行写页面即可。** 相同 Slug 会发生丢失更新，需要分组 Reduce 和并发控制。
7. **删除文档只需删除 SourceRef。** 正文里的独有事实也必须撤回或重写。
8. **模型生成的 Slug 可直接作为地址。** 需要规范化、校验、去重和稳定迁移策略。
9. **自动生成可以覆盖人工页面。** 必须建模编辑来源、Revision 和保护策略。
10. **一次生成质量高就足够。** 更新、删除、失败恢复和长期结构健康更能决定系统价值。

## 本章小结

- Chunk、实体关系图、Wiki 链接图、分类树和向量近邻表达不同关系。
- Graph RAG 将文本抽成节点/边，但查询结果应回到来源 Chunk 才能验证和授权。
- 图谱质量依赖受控 Schema、实体消歧、证据绑定和删除撤回，不依赖节点数量。
- Wiki 是可阅读、可编辑、可版本化的长期派生知识资产，不是一次性摘要。
- WeKnora 使用文档级 Map、Slug 分组和页面级 Reduce 归并跨文档贡献。
- 候选抽取与 Chunk Citation 分离，使页面组织和证据定位分别可控。
- Summary 页使用文档级来源，实体/概念页使用片段级证据，来源粒度应诚实。
- Slug 级锁保护同页更新，全局链接和 Index 通过去抖 Finalize 最终收敛。
- 源文档更新和删除必须触发 retract；派生知识的撤回比首次生成更难。
- Revision、编辑来源、Lint 和 Agent 写工具共同把 Wiki 从模型输出变成可治理产品。

### 可选实践

选择三份讨论同一技术主题的文档，手工设计 `summary/*`、`entity/*`、`concept/*` 候选与 `SourceRefs/ChunkRefs`。画出 Map 输出、按 Slug 分组和 Reduce 后页面。然后模拟删除其中一份文档，逐项写出哪些页面删除、哪些页面重写、哪些链接和版本变化。最后再判断其中哪些关系值得进入类型化知识图谱。

下一篇：[外部数据源与持续同步](04-external-data-source-sync.md)
