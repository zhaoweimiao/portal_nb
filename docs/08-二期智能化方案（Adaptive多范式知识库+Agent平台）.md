# 二期方案：moredoc 之上的 Adaptive 多范式 AI 知识库

> 文档版本：草案 v0.2
> 创建日期：2026-06-07
> 上游依赖：`docs/01`（需求规格说明书）、`docs/05`（一期最终方案）、`docs/07`（a30 部署存档）
> 编制对象：项目决策方 + 二期实施团队
> 文档定位：**一期 P3 项"AI 智能检索/问答"的详细方案**

---

## Context（为什么要写这份文档）

一期已基于 moredoc CE v3.4.0 跑通了"文件库"形态的知识库（详见 `docs/05` 与 `docs/07`），但 `docs/05` 第五节 P3 项"AI 智能检索/问答"只是一句话愿景，没有方案。

二期讨论中明确：

- **关系**：智能化是 moredoc 的**旁路 Copilot**，独立部署、只读 moredoc 数据、在 moredoc UI 注入 Agent 对话入口
- **思路偏好**：欣赏 Karpathy 编译型 Wiki / LLM + Obsidian / Graphify 这一脉"编译型 + markdown + 图"路线（用户提供 3 篇原文剪藏 md 作参考）
- **关键反馈**：那三种思路是**启发不是限定**——方案要"站在架构师角度，从多个角度，设计一个完善的、灵活的 agent 和知识库系统"
- **形态**：既是问答助手、也是 Agent 工作台（产 PPT/PDF/调研报告）
- **资源**：不限，聚焦技术方案

业界 2026 年的真共识也已经形成：**没有任何单一 RAG 范式能通吃企业知识库**。Microsoft GraphRAG 索引贵、LightRAG（EMNLP 2025 最佳论文）轻量但适用范围窄、LazyGraphRAG 把成本压到 0.1%、PageIndex/Vectorless 在结构化文档上 98.7% 准确率、Karpathy 编译型 Wiki 在百份级稳态最优、向量 RAG 仍是简单事实最佳——业界的方向是 **Adaptive RAG**：由 Query Router 按问题类型分流到不同检索路径。

本方案据此重塑：把"编译型 Wiki/Obsidian/Graphify"作为骨架里**可插拔的范式之一**，与业界其他成熟方案同台并存；同时把 moredoc MySQL 里的结构化资产当作**已经做完 60% 数据预处理的上游**深度复用。目标：搭一个能支撑未来 3–5 年演进的**灵活骨架**，而不是某一篇文章思路的工程化。

---

## 0. 立意与四条骨架性质

二期目标不是"在 moredoc 上接一个 RAG"，而是搭一个满足以下四条性质的骨架：

1. **Adaptive**——Query Router 按问题类型选检索路径，不绑死任何范式
2. **多范式可插拔**——Vector / Graph / Vectorless / Wiki 四条索引路径并存，按价值密度分级编译
3. **ACL 贯通**——moredoc 已建好的用户/组/权限模型必须穿透到 AI 层每一次召回
4. **可观测可评测**——每次回答可回放、可回归、可量化改进

Karpathy 编译型 Wiki、Obsidian 工作流、Graphify 都是骨架里**可插拔的一种范式**，不是骨架本身。

---

## A. 范式选型：从单一押注到 Adaptive 路由

七种主流范式的实战定位：

| 范式 | 强项场景 | 弱项 | 索引成本 | 查询延迟 | 二期采用阶段 |
| --- | --- | --- | --- | --- | --- |
| BM25/ES 关键词 | 精确名词、ID、代码、规章条款编号 | 语义近似 | 低（已有） | 极低 | P1 立刻 |
| Vector RAG | 单跳事实查询、FAQ、产品手册 | 跨文档推理、聚合分析 | 中 | 低 | P2 起 |
| Microsoft GraphRAG | 大语料跨文档综述 | 索引成本高、文档高频变更 | 极高 | 中 | P4 选型对比 |
| LazyGraphRAG (2025) | 想要 GraphRAG 效果但预算敏感 | 查询期 LLM 调用多 | 低 | 高 | **P4 优选** |
| LightRAG (EMNLP 2025) | 多跳推理、中等预算 | 真全局综合 | 中 | 中 | P4 备选 |
| PageIndex / Vectorless | 高度结构化文档（合同/规章/报告）实测 98.7% | 非结构化长文本 | 中 | 中 | P3 |
| Karpathy 编译型 Wiki | 稳态高价值知识、需要复利 | 高频更新、规模过千 | 极高（人+LLM） | 极低（查询变查表） | P5，仅高价值 Top 10% |

**核心决策：引入 Query Router 作为顶层调度。**

Router 是一个轻量分类器（初版规则 + prompt 工程，后用 `mnt_search_record` 微调），输入用户 query 与会话上下文，输出一个检索计划：

```python
plan = {
  "routes": ["es", "vector", "graph", "vectorless", "wiki"],   # 可多选
  "rerank": "rrf" | "llm",
  "budget": {"max_tokens": ..., "max_latency_ms": ..., "max_llm_calls": ...}
}
```

判定规则初版（可观测可调）：

| 信号 | 阈值 | 路由倾向 |
| --- | --- | --- |
| 命中 ID/编号/精确名词（正则） | regex 命中 | ES 优先 |
| 实体数 ≥ 2 且含关系词（"与"、"对比"、"影响"） | 实体抽取 | Graph + Vector |
| 含结构词（"第 X 条"、"附件 X"、"目录"） | 词典 | Vectorless（PageIndex 风格） |
| 查询长度 > 40 字 或 含"总结"、"综述"、"全景" | 启发式 | Wiki + LazyGraph |
| 历史 `search_record` 中相似 query 的 `spend_time` 长 | KNN 相似 | 升级路由（多路并发） |
| 默认 | — | ES + Vector + RRF |

这是骨架的灵魂：**任何新范式（明年出 X-RAG）只是给 Router 加一条 route 和一个判定规则**，不是重构系统。

---

## B. moredoc 数据资产的深度复用（差异化护城河）

一般 RAG 方案从 PDF 起步，我们从一张**已经预处理过的 MySQL** 起步。已确认的 moredoc 关键表：

| 表 | 关键字段 | 二期复用价值 |
| --- | --- | --- |
| `mnt_document` | id, title, keywords, description, pages, view_count, score, ext, language, source_url | 元数据 + 热度 + 多语言 |
| `mnt_attachment` | hash, type, path, name, size, ext | 文件物理路径 + hash 锚点 |
| `mnt_attachment_content` | **hash + content longtext + content_size** | **已抽取的全文，省 OCR/Office 解析** |
| `mnt_category` | parent_id, title, description, sort, doc_count | **多级分类树 + 描述**，天然本体 |
| `mnt_document_category` | document_id × category_id | 文档↔分类 多对多 |
| `mnt_document_relate` | document_id + related_document_id (text) | **预存的弱图谱种子边** |
| `mnt_search_record` | user_id, keywords, spend_time | **真实用户查询日志**（评测集 + Router 训练信号） |
| `mnt_document_score`、`mnt_favorite`、`mnt_comment` | document_id + score/count | **质量信号**（编译优先级） |
| `mnt_user` / `mnt_group` / `mnt_group_permission` / `mnt_user_group` | 完整 ACL 模型 | **必须穿透到 AI 层** |

### B.1 `mnt_attachment_content`：跳过 OCR/Office 解析

**收益**：节省至少 60% 的解析工程量与算力。
**代价**：`content` 是扁平 longtext，**丢失了页码锚点和版式结构**。

解决方案 — **双轨解析**：

- **快道**：直接消费 `mnt_attachment_content.content`，按字符滑窗切块（中文建议 chunk_size 400–600 字符 + 20% 重叠），用于 P1/P2 的 ES 增强与向量入库。块 ID 形如 `{hash}#offset_{start}-{end}`
- **慢道**：对**高价值文档**（score 高 / view_count 高 / 收藏多）的源文件路径 `mnt_attachment.path`，由二期独立 parser（MinerU、Docling 或 unstructured）重做带页码与结构的解析，用于 PageIndex 和精确引用
- **页码反推**：对快道文本按"换页符 / 章节特征 / 与 PDF 字符密度比对"估算页码作为弱锚点，标记 `cite_confidence: low|medium|high`，前端展示时区分置信度

### B.2 `mnt_category`：天然的初始本体

分类树是已经人工策划过的知识空间划分，二期当成三件事用：
1. **Wiki vault 初始目录骨架**：`vault/{category_path}/` 直接复用
2. **图谱根节点 + 域标签**：每节点继承所属 category 作为初始 community 标签，省 LLM 从零做社区检测
3. **ACL 默认继承单元**：检索结果按 category 聚合后套用 `mnt_group_permission` 检查

### B.3 `mnt_document_relate`：图谱冷启动种子边

moredoc 已经有"相关文档"关系。二期不丢，作为图的**初始边集**，置信度 `0.5`。LLM 抽取的边初始置信度 `0.3`。两者重叠的边升到 `0.8`。用户在 UI 上点击/采纳过的引用进一步升到 `0.95`。这是一个**逐步收敛的弱图**，避免一上来就跑 GraphRAG 烧钱建强图。

### B.4 `mnt_search_record`：路由器训练信号 + 评测集生成器

`user_id + keywords + spend_time` 是天然的难度标签：
- 短 keywords + 短 spend_time → 简单事实查询样本
- 长 keywords / 多次连续相关查询 / 长 spend_time → 复杂综合查询样本
- 反复出现的 query → 高频，优先编译进 Wiki

**二期第一周**就应该把这张表做成 eval set 生成器（抽样 + LLM 生成参考答案 + 人工审 10%）。**没有 eval set，所谓"灵活演进"就是凭感觉换方案。**

### B.5 质量信号

入库优先级权重 = `α·score + β·log(view_count) + γ·favorite_count`。Wiki 编译白名单从这里出。Reranker 后融合时作为先验。

---

## C. 系统架构：十层分离

不画一张大图。十层模块化，每层职责单一、独立演进、独立替换。

| 层 | 职责 | P1 起步实现 | 可演进选项 |
| --- | --- | --- | --- |
| **L1 Source** | 数据源 | moredoc MySQL（只读副本 / CDC） | + Confluence / 飞书 / GitLab / SharePoint |
| **L2 Sync** | 增量同步 | `moredoc-syncer`：基于 `updated_at` 轮询；按 hash/category/ACL 增量 | Debezium CDC + Kafka |
| **L3 Parse** | 文本 + 结构化 | 快道：直读 `mnt_attachment_content`；慢道：高价值文档重解析 | MinerU、Docling、unstructured |
| **L4 Index** | 多路索引并存 | ES（复用 moredoc 自带） | + 向量库 + 图库 + PageIndex 树 + Wiki vault |
| **L5 Compile** | LLM 编译产物 | 无（P1 跳过） | wiki 页、社区摘要、PageIndex 树摘要 |
| **L6 Retrieval** | Query Router + 多路检索 + 融合 | Router v0（规则）+ ES + RRF | Router v1（分类器微调）+ Reranker（bge-reranker-v2-m3） |
| **L7 Agent** | MCP Server 暴露能力 | QA Agent + 基础工具集 | Synthesis / Report / Slide / Lint / Heal / Compile |
| **L8 Application** | Chat API + 任务编排 + 引用解析 | FastAPI + 简单编排 | LangGraph 工作流 + Temporal 长任务 |
| **L9 UI** | 嵌入入口 + 工作台 + Vault 浏览 | moredoc 内嵌 Copilot 抽屉 | + Next.js 工作台 + Quartz vault |
| **L10 Ops** | 评测/监控/审计/Lint/Heal/备份 | 日志 + RAGAS 雏形 | Phoenix（OSS）/ LangSmith + 自动 Heal |

**关键解耦原则**：

- L4 五种索引**互不依赖**，任何一种都能单独关停回滚
- L6 Router 与具体索引解耦，新增范式只需新增 route handler
- L7 Agent 通过 MCP 调用 L4–L6 能力，业务逻辑与基础设施隔离
- **ACL 检查在 L6 召回后、L7 进 Agent 前**强制套一道，任何 route 都绕不过——**不是工具，是不可绕过的中间件**

### 端口分配（沿用一期 a30 公网 34440-34450 ↔ 内网 18000-18010 映射规则）

一期占 18005（公网 34445）。二期建议：

| 服务 | 内网 | 公网 | 备注 |
| --- | --- | --- | --- |
| Agent 工作台 Web (Next.js) | 18006 | 34446 | 用户主入口 |
| Chat API (FastAPI) | 18007 | 34447 | 嵌入抽屉调用 |
| Quartz vault 静态站 | 18008 | 34448 | 只读浏览 |
| Phoenix Trace UI | 18009 | 34449 | 评测/调试 |
| Gitea (内网 only) | 18010 | — | vault PR review |
| 余量 | — | 34450 | 留给本地 vLLM 推理 |

---

## D. Agent 系统设计

### D.1 Agent 矩阵

| Agent | 触发方式 | 主要工具 | 输出 | 典型耗时 |
| --- | --- | --- | --- | --- |
| **QA Agent** | 用户在 moredoc 内嵌抽屉提问 | search_es / search_vector / cite_resolve | Markdown 答 + 引用 | 3–10s |
| **Synthesis Agent** | 工作台"综述"按钮 | + query_graph / search_wiki | 多段对比报告 | 30s–2min |
| **Report Agent** | 工作台"长报告"按钮 | + render_pdf | PDF | 3–10min |
| **Slide Agent** | 工作台"PPT"按钮 | + render_pptx | PPTX | 3–8min |
| **Lint Agent** | 后台 cron 每日 | search_wiki / query_graph | 健康报告 | 后台 |
| **Heal Agent** | Lint 触发或人工 | + compile_wiki | 修复 PR | 后台 |
| **Compile Agent** | 入库后或 cron | parse / chunk / extract_entities | 索引产物 | 后台 |

### D.2 协作模式

- **用户即时型**：QA / Synthesis / Report / Slide —— 用户在 UI 触发，同步或半异步
- **后台编译型**：Compile —— 文档入库 webhook 触发，按价值优先级队列
- **后台守护型**：Lint → Heal —— Lint 每日扫描产出 issue，Heal 自动修小问题、生成 PR 给大问题
- **Agent 链**：Report Agent 内部触发若干 Synthesis Agent 并行收集子主题素材

### D.3 共享 MCP 工具集（按层归类）

```
# 检索类
search_es(query, filters, k)
search_vector(query, k, filters)
query_graph(entities, hops, edge_types)
search_pageindex(doc_id, semantic_query)
search_wiki(topic)

# moredoc 桥接
moredoc_doc(hash) -> metadata
moredoc_attachment_content(hash) -> text
moredoc_acl_check(user_id, doc_id) -> bool
moredoc_category_tree() -> tree

# 引用与渲染
cite_resolve(chunk_id) -> {doc, page, url, snippet}
render_pdf(markdown, template)         # Typst 实现
render_pptx(outline, template)         # Marp 实现

# 编译类（仅 Compile/Heal Agent 可见）
extract_entities(text)
extract_relations(text)
write_wiki_page(path, content)
```

**ACL 哨兵**：`moredoc_acl_check` 不是工具，是一个**强制中间件**——所有 `search_*` 工具返回前都过一遍，按 `user_id × doc_id` 过滤，未授权文档**连标题都不能出现在 Agent 上下文里**。

### D.4 Memory / Context 管理

- **Session memory**：单次会话上下文，Redis，TTL 1 小时
- **User memory**：用户偏好（习惯术语、关注 category），可被用户编辑
- **Team memory**：团队常用查询模板、术语表、缩略语词典，由管理员维护，注入 system prompt

### D.5 可观测与评测

- **Trace**：每次请求记录 `query → route → retrieved chunks → llm calls → answer → citations`，落 Phoenix（OSS 自部署）
- **Eval set 自动生成**：从 `mnt_search_record` 抽样 + LLM 生成参考答案 + 人工审 10%
- **RAGAS 四指标**：faithfulness / answer_relevancy / context_precision / context_recall，每周回归
- **回放**：每条 trace 可一键重跑，对比新旧 prompt/模型/路由策略
- **A/B 灰度**：Router 新版只对 10% 流量生效，指标好再放量

---

## E. 演进路线图（六阶段）

| 阶段 | 周期 | 目标 | 引入范式 | 关键产出 | 退出门槛 | 依赖 |
| --- | --- | --- | --- | --- | --- | --- |
| **P1 最小可用** | 2–3 周 | 接通 moredoc + LLM 直答 | ES（复用）+ LLM | moredoc 嵌入抽屉、Chat API、**ACL 中间件**、Trace 基础设施、eval set 生成器 | 内部 20 人试用，回答率 ≥ 70%，0 越权 | — |
| **P2 向量增强** | 3–4 周 | 提升语义召回 | + Vector RAG | 向量库（pg_vector 起步，超 200 万 chunk 切 Qdrant）、syncer 增量、Reranker | RAGAS context_recall ≥ 0.75 | P1 |
| **P3 结构化文档** | 3–4 周 | 合同/规章/报告精准引用 | + Vectorless（PageIndex 风格） | 高价值文档重解析、文档树索引、精确页码引用 | 结构化文档问答准确率 ≥ 90% | P1 |
| **P4 图谱与综述** | 4–6 周 | 跨文档关系与综述 | + LazyGraphRAG（备 LightRAG） | 弱图（复用 `mnt_document_relate` 种子）、社区摘要、Synthesis Agent | 综述类查询满意度 ≥ 70% | P2 |
| **P5 编译型 Wiki** | 持续 | 高价值知识复利 | + Karpathy Wiki + Obsidian 约定 + Quartz | vault 仓库、Compile/Lint/Heal Agent、Quartz 浏览、Graphify 选装做跨域关联视图 | 高频 query 命中 Wiki ≥ 40%，首屏延迟 < 1s | P2 + P4 |
| **P6 Agent 工作台** | 4–6 周 | 长任务产 PPT/PDF/报告 | — | 工作台 Web、Report/Slide Agent、任务队列（Dramatiq → Temporal） | 周报告产出 ≥ 10 份，人工修改率 < 30% | P2 起 |

**P3、P4、P6 可与 P2 并行**，不严格串行。**P5 必须在 P4 之后**，否则没有图谱与社区摘要支撑 Wiki 编译质量。

**P1 的关键纪律**：**不引入向量库、不编译 Wiki、不建图**。第一阶段交付的应该是"基础设施 + ACL 贯通 + 可观测"——让团队**先看到能用**。这与 Karpathy"按需建库"的精神一致，但落地优先级是 **trace、ACL、syncer、嵌入入口** 这些没人讨论但决定成败的东西。

---

## F. 风险与对策

| 风险 | 对策 |
| --- | --- |
| **幻觉传染**（LLM 编译产物入库后被再次检索） | 编译产物标 `source: compiled` 元数据；引用解析层对 compiled 内容**强制回溯到原始 chunk**；RAGAS faithfulness 监控；关键页强制 Claude Sonnet 编译 + 2 人 review |
| **编译成本失控** | 按 `score × view_count × favorite_count` 分级，**Top 10% 才编译 Wiki**；LazyGraphRAG 替代完整 GraphRAG；`content_hash` 增量触发不重跑 |
| **规模上限** | L4 各索引独立分片；向量库从 pg_vector 切 Qdrant 的迁移脚本预先备好；INDEX.md 分层（按分类树）；Graph 社区摘要作为二级 index |
| **增量更新一致性** | `hash` 作为唯一锚点；syncer 用 outbox 模式 + 至少一次投递；编译产物带 `source_hash`，源变则失效；moredoc 删除时不立即删 wiki，标 `archived: true` 由人决定 |
| **多人审计** | Wiki 走 Gitea PR；图谱/向量变更落审计日志；所有 Agent trace 7 天热存 + 90 天冷存；每个一级分类指定 owner |
| **来源追溯** | `cite_resolve` 强制返回 `{doc_hash, page, snippet, url}`；前端引用可点击跳转 `http://183.56.181.9:34445/document/{id}?page={N}` 高亮 |
| **权限隔离（4 范式贯通）** | ACL 中间件在 L6 召回后强制套用；图谱按 `doc_id` 集合裁剪子图；Wiki 页有 `acl_groups` frontmatter；向量库 metadata 带 `doc_id` 过滤；越权问答返回"该话题需更高权限"而非伪装不知道 |
| **与 moredoc ES 关系** | **增强不替换**——P1 直接用 moredoc ES；P2 起 syncer 把 chunk 级数据写到独立 ES index，moredoc 文档级 ES 保留 |
| **Router 误判** | Router 输出**带 confidence**；低置信度回落到默认多路 + RRF；Router 决策入 trace，每周用真实 query 反推回归；人工标注 1% 样本持续微调 |
| **评测体系** | RAGAS 周回归 + `mnt_search_record` 抽样 + 人工评 1% + A/B 灰度 |
| **技术栈复杂度** | L4 各索引**默认关闭**，按阶段启用；统一 docker-compose + 备 Helm；单一 trace 后端（Phoenix）；运维 SLO 文档化 |

---

## G. 选型矩阵（横向对比）

| 维度 | 候选 | 二期建议 | 理由 |
| --- | --- | --- | --- |
| **知识图谱** | MS GraphRAG / LightRAG / LazyGraphRAG / Graphify / Neo4j LLM Graph Builder / Cognee | **P4 主选 LazyGraphRAG，备选 LightRAG** | 索引成本 0.1% vs 完整 GraphRAG；Graphify 太重浏览侧；MS GraphRAG 索引太贵 |
| **向量库** | Milvus / Qdrant / Weaviate / pg_vector / 复用 ES | **P2 起 pg_vector，> 200 万 chunk 切 Qdrant** | 初期省运维（一套 PG 服务）；Qdrant 生态成熟、过滤强 |
| **Vectorless** | PageIndex / 自研 | **自研轻量树（基于 mnt_category + 章节解析）+ 借鉴 PageIndex 思路** | 结合 moredoc 既有结构 |
| **Agent 框架** | LangGraph / CrewAI / AutoGen / Pydantic AI | **LangGraph（编排）+ Pydantic AI（单 Agent）** | LangGraph 状态图可观测、可回放；Pydantic AI 类型安全 |
| **Wiki 浏览** | Quartz / Obsidian Publish / MkDocs / Docusaurus | **Quartz v4** | Obsidian 原生兼容、静态、双链、图谱视图、免费、内网部署 |
| **评测** | RAGAS / Phoenix / LangSmith / 自研 | **RAGAS 指标 + Phoenix trace（OSS 自部署）** | 不锁厂商、内网可部署、可灰度 |
| **LLM 路由** | LiteLLM / OpenRouter / Portkey | **LiteLLM 自部署** | a30 上跑、统一计费、便于切本地 GPU |
| **MCP Server** | Python SDK / fastmcp | **fastmcp** | 工具集生态最全 |
| **任务队列** | Celery / Dramatiq / RQ / Temporal | **Dramatiq（中量）→ Temporal（长任务）** | Report Agent 跑 10 分钟需要 Temporal 的可恢复性 |
| **PPT 渲染** | Marp / python-pptx / Reveal.js | **Marp** | markdown→pptx 一行命令，公司模板靠 theme CSS |
| **PDF 渲染** | Typst / Pandoc / weasyprint | **Typst** | 中文友好、增量编译快、长表格分页稳 |
| **Office/PDF 解析（慢道）** | MinerU / Docling / unstructured / Marker | **MinerU 为主，Marker 兜底扫描件** | 中文表格识别佳；快道直接复用 moredoc 数据，慢道才需要 |
| **LLM** | DeepSeek-V3 / Claude Sonnet 4.6 / Kimi / 本地 Qwen2.5-32B | **DeepSeek-V3 默认 + Claude Sonnet 关键页 + P6 阶段切本地 Qwen** | DeepSeek 输入 ~$0.27/M token，编译千份级月成本可控 |

### Karpathy / Obsidian / Graphify 的具体落位

| 思路 | 在本骨架中的位置 | 何时启用 |
| --- | --- | --- |
| **Karpathy 编译型 Wiki** | L5（Compile）+ L4（Wiki 索引）的"按需"分支 | P5：仅对 Top 10% 高价值知识编译 |
| **Obsidian 工作流约定** | L4 Wiki vault 的目录/链接/frontmatter 规范；Obsidian 客户端是少数本地编辑者可选工具 | P5 起；不强制团队装 Obsidian |
| **Graphify** | L4 图库的一种实现选项；同时为 L9 提供 Obsidian-like 浏览视图 | P5 选装（如 LazyGraphRAG 满足不了跨域关联视图需求时） |
| **`maomaomonkey/ai-wiki-compiler`**（已开源 Skill） | L5 编译器底座 fork 起点（六大工作流 + 三脚本：wiki-search/wiki-lint/wiki-index） | P5 启动时 fork |

---

## H. 用户交互流程（五个核心场景）

### A. 阅读 PDF 时呼起 Copilot（P1 即可）

1. 用户在 moredoc 阅读 `/document/12345`
2. Nginx `sub_filter` 已注入嵌入抽屉，右下角"AI"按钮常驻
3. 用户问"年假可以跨年结转吗"
4. 抽屉向 `/api/chat` 发起 SSE，带 `current_doc_id=12345`
5. Chat API → MCP → `moredoc_acl_check` → Query Router 判定（短问、有"可以吗"→ 简单事实查询）→ ES + Vector 多路召回 → ACL 中间件过滤 → QA Agent 流式答 + 引用
6. 抽屉渲染答案，脚注可点跳 `?page=18` 高亮

### B. 跨文档综述：项目 X 历史决策（P2+P4）

1. 用户在工作台输 "总结项目 A 从立项到当前的所有关键决策"
2. Router 判定（长问 + "总结"）→ Graph + Wiki + Vector 多路
3. LangGraph Synthesis Agent 编排：查 `mnt_document_relate` 种子 → LazyGraph 社区摘要 → 关联 Wiki 页 → LLM 综合
4. 中间状态推到 UI（"正在读 7 个相关页…"）
5. 输出 markdown 综述，每个决策点带 `[^doc:xxx#pY]`
6. 用户可点"保存为 wiki/syntheses/" → 进入 `_drafts/` → Gitea PR

### C. 长任务：30 分钟培训 PPT（P6）

1. 用户选"任务模板：培训 PPT"，输入主题"采购流程合规要点"
2. LangGraph + Temporal 状态机：outline → expand → verify → render
3. 工作台显示步骤进度 + 中间产物预览
4. 完成下载 .pptx，附带 sources.md（引用清单 + moredoc 链接）

### D. 管理员：批量新规章入库（P5+P6）

1. 管理员上传 50 份新规章到 moredoc
2. syncer 增量发现新 doc_id 批次
3. Compile Agent 并行跑解析 → 向量入库 → PageIndex 重建 → LazyGraph 局部重算 → 高价值的进 Wiki `_drafts/`
4. Lint 报告："47 篇通过 / 3 篇有矛盾，待 review"
5. 管理员在 Gitea 看 3 个 PR，merge 或退回；merge 后 Quartz 自动重建

### E. 管理员：wiki/索引健康度检查（P5）

1. 触发 Lint flow
2. 报告四类：矛盾 / 孤立 / 断链 / 过期
3. 报告进 Gitea Issue，逐条认领；可触发 Heal Agent 自动修小问题

---

## I. 与一期文档的衔接

- **本文档落位**：`docs/08-二期智能化方案（Adaptive多范式知识库+Agent平台）.md`
- **引用关系**：本文档 → `docs/01`（业务目标）+ `docs/05`（一期边界、第五节 P3 项被本文档完整细化）+ `docs/07`（部署环境）
- **后续可拆分子文档**（实施阶段按需）：
  - `docs/09-syncer-spec.md`：L2 同步层详细规约
  - `docs/10-mcp-tools-spec.md`：L7 工具集接口契约
  - `docs/11-eval-protocol.md`：评测协议与 baseline
  - `docs/12-acl-bridge.md`：ACL 跨范式贯通方案

**二期仓库结构建议**（同主仓 `portal_nb` 下新增）：

```
portal_nb/
├── docs/                          # 既有文档 + 新增 08
├── apps/
│   ├── moredoc/                   # 一期，不动
│   ├── syncer/                    # L2
│   ├── parser/                    # L3 慢道
│   ├── retrieval-api/             # L6（Router + 多路检索 + RRF）
│   ├── mcp-server/                # L7（fastmcp + ACL 中间件）
│   ├── chat-api/                  # L8（FastAPI + SSE）
│   ├── orchestrator/              # L8（LangGraph + Temporal）
│   ├── workbench-web/             # L9 工作台 (Next.js)
│   └── copilot-widget/            # L9 moredoc 嵌入抽屉（JS + Nginx subfilter）
├── infra/
│   ├── compose/                   # 本地 docker-compose.kb.yml
│   └── helm/                      # 生产 Helm（可选）
├── vault/                         # L5 Wiki vault（Quartz 源、Gitea 托管 PR）
├── eval/                          # RAGAS + eval set 生成器
└── ops/                           # Lint/Heal 脚本、备份、监控
```

`vault/` 是**运行时数据卷**（独立 git repo `kb-vault.git` 由 Gitea 托管），与代码仓 `portal_nb` 分离，避免代码 PR 与知识 PR 混在一起。

---

## 架构师核心判断（5 条）

**1. moredoc 的 MySQL 是被严重低估的资产，必须把"复用 mnt_attachment_content + mnt_category + mnt_document_relate + mnt_search_record"作为差异化护城河。**
几乎所有公开 RAG 方案是从零开始的；我们从 60% 完成态起步，应该把这个优势放大，而不是绕开。尤其 `mnt_attachment_content` 让我们跳过 OCR/Office 解析；`mnt_search_record` 让我们有天然的评测集——这是别的方案花钱也买不来的。

**2. Adaptive Router 是骨架的灵魂，不是某一种范式。**
Karpathy/Obsidian/Graphify/GraphRAG/Vectorless 都是**可插拔的 route**。任何"押注单一范式"的方案在 1–2 年内都会被打脸。骨架要做的，是让明年出现的新范式可以**只新增一个 route handler 就接入**。

**3. P1 必须克制，跑 直读 + ES + LLM 即可。**
不要在第一阶段堆向量库、知识图谱、Wiki 编译。第一阶段交付的应该是"基础设施 + ACL 贯通 + 可观测"——让团队**先看到能用**。这个克制与 Karpathy"按需建库"的精神是一致的，但落地优先级是 **trace、ACL、syncer、嵌入入口** 这些**没人讨论但决定成败**的东西。

**4. ACL 不是一个模块，是一道横向中间件，必须在 L6 召回后、Agent 上下文构造前强制套用，绝不能依赖 Agent 自觉。**
这是企业 AI 知识库和个人 Obsidian 工作流的根本区别。任何一种检索范式（Vector / Graph / Vectorless / Wiki）只要漏一次权限检查，整个项目的信任成本就要重建。建议把 `moredoc_acl_check` 实现成**不可绕过的 retrieval middleware**，而不是可选 MCP 工具。

**5. 可评测优先于可演进。没有 RAGAS + 真实 query 回归 + Trace 回放，所谓"灵活演进"就是凭感觉换方案。**
`mnt_search_record` 是天上掉下来的训练 + 评测信号源，二期第一周就应该把它做成 eval set 生成器。否则 P4 引入图谱、P5 编译 Wiki 时，**无法量化是不是真的变好了**，演进会退化成"换工具的安慰剂"。这是综合方案最容易被忽视、却决定长期价值的一条。

---

## Critical Files

待二期立项后涉及的关键路径（绝对路径或主仓建议路径）：

| 路径 | 用途 |
| --- | --- |
| `docs/05-最终实施方案（需求分解+技术选型+实施计划）.md` | 一期 P3 锚点（末尾追加一行"P3 详见 docs/08"） |
| `docs/07-moredoc-a30实际部署存档与验证手册.md` | 端口规则、工作目录、共存栈基础 |
| `docs/08-二期智能化方案（Adaptive多范式知识库+Agent平台）.md` | **本方案落位** |
| `apps/syncer/` | L2 同步层（基于 mnt_updated_at 增量） |
| `apps/mcp-server/` | L7 fastmcp + **ACL 中间件**（不可绕过） |
| `apps/retrieval-api/` | L6 Router + 多路检索 + RRF/Reranker |
| `eval/eval_set_from_search_record.py` | 从 `mnt_search_record` 生成评测集 |
| `vault/schema/CLAUDE.md` | 编译器人格（P5 启用） |

## 关键外部资源

- Karpathy 原始 Gist：<https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f>
- ai-wiki-compiler（P5 阶段 fork 起点）：<https://github.com/maomaomonkey/ai-wiki-compiler>
- Microsoft GraphRAG：<https://github.com/microsoft/graphrag>
- LightRAG（EMNLP 2025 最佳论文）：<https://github.com/HKUDS/LightRAG>
- LazyGraphRAG：Microsoft 2025 论文与官方仓库
- PageIndex（Vectorless RAG）：<https://github.com/VectifyAI/PageIndex>
- Graphify v5：<https://github.com/safishamsi/graphify>
- Quartz 4：<https://quartz.jzhao.xyz/>
- RAGAS：<https://github.com/explodinggradients/ragas>
- Phoenix（OSS Trace）：<https://github.com/Arize-ai/phoenix>
- LiteLLM：<https://github.com/BerriAI/litellm>
- MinerU（中文 PDF 解析）：<https://github.com/opendatalab/MinerU>

---

## Verification

本方案是**设计文档**而非代码改动，验证方式：

1. **文档完整性**：将本方案作为 `docs/08-...md` 落盘后，对照 `docs/05` 第五节 P3，确认 P3 被完整细化（覆盖问答 / 综述 / 生成 / 检索 / 审计五类需求）
2. **架构自洽**：按"五个核心用户场景"端到端走一遍，每个能在十层架构上找到完整数据流路径
3. **资源约束**：核对端口分配与 a30 部署存档中"公网 34440-34450 ↔ 内网 18000-18010"端口规则一致
4. **P1 可启动判定**：阶段 1 的产出（嵌入抽屉 + Nginx subfilter + Chat API + ACL 中间件 + Trace + Eval set 生成器）应能在 2-3 周内验证，即可宣布方案可落地
5. **范式开放性自检**：随便挑一个"明年才出现"的假想新范式（如 X-RAG），按方案能否只新增一个 route handler + 一条判定规则就接入——能则方案灵活度合格
6. **用户复核**：用户阅读后能确认：(a) 没有绑死 Karpathy/Obsidian/Graphify，三者作为可插拔范式存在；(b) moredoc 的数据资产被深度复用而不是绕开；(c) ACL/可评测/可观测三件事被当成第一优先级
