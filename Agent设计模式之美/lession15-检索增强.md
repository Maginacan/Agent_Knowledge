---
title: "13｜检索增强：Agent 的知识库和证据链"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/989200"
tags:
  - "#pattern/memory"
  - "#axis/memory"
  - "#axis/chain"
  - "#axis/loop"
  - "#topic/rag"
  - "#topic/evidence-chain"
  - "#topic/index-versioning"
  - "#topic/contextual-retrieval"
  - "#topic/session-state"
  - "#concept/blue-green"
  - "#concept/evidence-contract"
audio: true
duration: "27:49"
publish_date: "2026-06-23"
course_progress: 13
pattern_position: "记忆 × 链式 (Naive RAG) / 记忆 × 循环 (Agentic RAG)"
module: "记忆：沉淀之美（第 3 讲 / 共 5 讲）"
related:
  - "[[lession14-分层保留]]"
  - "[[lession16-进度追踪]]"
---

# 13｜检索增强：Agent 的知识库和证据链

> **本讲定位**：记忆模式组第二个具体模式 → 解决"货架太大以后，当前这一轮到底该取哪几件东西"。

| 属性 | 内容 |
|------|------|
| 课程模块 | 记忆：沉淀之美（第 3 讲 / 共 5 讲）|
| 课程编号 | 13 |
| 双轴坐标 | Naive RAG = **记忆 × 链式**；Agentic RAG = **记忆 × 循环** |
| 时长 | 27:49 |
| 发布日期 | 2026-06-23 |
| 前置讲 | [[lession14-分层保留]] — 货架怎么搭 |
| 后续讲 | [[lession16-进度追踪]] — "录" |

## 一句话核心

> **RAG（检索增强）= 在 Agent 时代，从"相似文本召回"升级成"可用、可信、可追溯的证据取回"。**
>
> 朴素 RAG 像一个**只管把相似的货堆过来的搬运工**。**生产级 RAG 是一条带批次、带凭证、带召回能力的供应链**。
>
> RAG 在记忆模块里解决的是 **"取"**。

---

## 一、RAG 为什么算记忆不算感知

### 1.1 一个容易混淆的问题

> 也曾思考过，RAG 最后不也是把几段材料塞进 context，让模型这一轮"看见"吗？为什么不把它归在感知模式？

### 1.2 感知 vs 记忆的边界

| 维度 | 感知 | 记忆 |
|------|------|------|
| 关心的是 | **这一次推理前**，哪些材料进入 context | 这些长期知识怎样**保存、索引、更新、过期、回滚** |
| 提问 | "这次能看到什么" | "知识库怎么管理" |

> **只看最后一步**，RAG 确实参与感知 —— 它把召回的片段送进当前上下文。
>
> 但**这一讲讨论的 RAG，重心在前面那半段**：写入侧（索引侧）。

### 1.3 RAG 的两条链路

```
              索引侧（离线）                          查询侧（在线）
┌─────────────────────────────┐    ┌──────────────────────────────┐
│ 知识源登记                       │    │ 当前任务约束                    │
│ 文档解析                         │    │       ↓                       │
│ 切块                             │    │ 过滤 → 召回 → 重排 → 引用       │
│ 向量化 + 关键词索引                │    │       ↓                       │
│ 索引版本管理                       │    │ 模型拿到带出处证据                 │
│ 权限范围管理                       │    └──────────────────────────────┘
└─────────────────────────────┘
```

> **RAG 是索引侧和查询侧两条链路的汇合**：
>
> - 离线把文档整理成**可发布、可回滚的索引资产**
> - 在线按当前任务约束**取回带出处的证据**
>
> **能进生产的 RAG，重点在于写入侧**。
>
> 要先把外部知识库做成一份**可发布、可追溯的软件资产**，然后才能被加工成可持续读取、可版本控制、可回滚、可审计的**记忆层**。

### 1.4 双轴图谱的位置

> 同一个 RAG，会随着工程成熟度，从**链式演进成循环**：

| 阶段 | 模式 | 双轴位置 |
|------|------|----------|
| **Naive RAG** | 文档进库 → 解析切块 → 建索引 → 召回 → 重排 → 注入 → 生成 | 记忆 × **链式** |
| **Agentic RAG** | 加回路："检索 → 评估文档 → 改写查询 → 再检索"（LangChain CRAG / Self-RAG）| 记忆 × **循环** |

> **同一模式可以存在多种拓扑结构，也说明双轴矩阵本来就是描述性的**，允许一个模式随成熟度跨格。

---

## 二、一个薪酬 SaaS 的真实问题

### 2.1 任务描述

> 用户发来要求：
>
> "帮我看一下**上海市场部 6 月薪资快照**里的异常。
>
> 咖哥缺 2 天考勤，小冰奖金比上月多 40%，小雪社保基数变了。
>
> 哪些可以自动通过，哪些要人审？"

### 2.2 任务里混着两类完全不同的信息

| 类型 | 例子 | 处理 | 来源 |
|------|------|------|------|
| **机械状态** | 员工 id、薪资批次 id、考勤记录 id、奖金审批单号、社保基数 id | **必须按位精确** | 工具返回 + SessionState，**程序确定性绑定，带 provenance** |
| **业务证据** | 6 月薪酬规则版本、上海市场部适用考勤扣款口径、奖金审批阈值、社保基数调整生效规则 | **适合 RAG** | 制度文档、审批规则、历史政策、内部知识库 |

> **LLM 不能凭印象复述一个 id，更不能从 RAG 里找一个"看起来像"的 id**。

### 2.3 两类信息的污染风险

| 污染类型 | 场景 | 后果 |
|----------|------|------|
| **机械状态被污染** | RAG 召回"上月奖金异常处理 FAQ"，模型把里面的 `payroll_batch_id` 当成本月批次 id | 错把上月 id 用于本月动作 |
| **证据缺失** | SessionState 里只有批次 id，却没有召回本月的规则版本 | Agent 基于**缺证据**的判断强行下结论 |

### 2.4 执行型 Agent 中的 RAG 边界

```
RAG:
  查政策、查规则、查历史口径，提供可引用证据。
Session State:
  管员工、批次、金额、账号、审批单号等机械真值。
Orchestrator:
  根据证据和状态编排下一步。
Verification Gate:
  在高风险动作前做一致性检查和人审。
```

> **RAG 管的是叙事证据和知识记忆**。它能告诉 Agent "这条奖金规则该怎么解释"，但它**不能**替 Agent 生成"对哪个薪资批次执行哪个动作"的关键参数。
>
> **证据和真值分属两个平面，最后在决策点合流**。

---

## 三、RAG 的来龙去脉：从"相关文档"到"可用证据"

### 3.1 学术源头

> **2020 年 5 月，Patrick Lewis 等人**在 NeurIPS 论文《Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks》里**第一次正式提出 RAG**。
>
> 核心想法：让生成模型在回答前，先去一个外部文档库里检索相关段落，把检索到的内容和问题一起送进模型。
>
> **模型不必把所有知识都背进参数，需要时去查就行**。

### 3.2 两条血缘

| 血缘 | 来源 | 关键技术 |
|------|------|----------|
| **信息检索（information retrieval）**| 上世纪七八十年代成熟 | 倒排索引、TF-IDF、**BM25**、向量空间模型 |
| **数据血缘（data provenance）**| 企业数据工程 | ETL 可复现、内容寻址 hash、**蓝绿发布** |

> RAG 的"取"这一半延续了几十年的信息检索老脉络。**稠密向量召回**则是把"相关性"从词面匹配换成了语义匹配。
>
> RAG 的"证据"这一半同样重要：索引要有版本、文档要有来源、答案要能追回出处。

### 3.3 朴素 RAG 为什么不够用

> 信息检索的目标是"**找到相关文档**"，它默认屏幕前有**一个人**，会自己判断哪条能用、哪条过期、哪条不适用。
>
> **但最原始的 Agent 系统中没有这个人**。
>
> Agent 拿到召回结果就直接往下推理、往下执行。
>
> 于是**"相关"远远不够**。Agent 要的是当前这个任务、这个租户、这个时间点、这个权限范围下，**真正适用且可引用的证据**。
>
> - **相关性**是给人看的排序
> - **证据**是给机器用的依据

### 3.4 2025 集体反思

| 来源 | 结论 |
|------|------|
| **Chroma Context Rot 研究（2025）**| 测试 18 个前沿模型，发现输入越长、待找信息和问题的语义相似度越低，**模型表现退化得越明显** |
| **ICML 2025 LaRA 基准（2326 个用例）**| RAG 和长上下文谁更好，**取决于模型规模、上下文长度、任务类型和召回片段的质量，没有一个方案通吃** |

> **朴素 RAG（Naive RAG）那种"查一次、塞进去、生成"的相似召回，在生产里不够用了**。

---

## 四、RAG 到底难在哪里

### 4.1 常见误解

> 很多团队第一次做 RAG，会把问题理解成"**召回率不够**"。
>
> 于是：
>
> - 换 embedding 模型
> - 调 chunk size
> - 换向量库
> - 把 top-K 从 5 调到 20
>
> **调完以后 demo 往往好看一点，生产问题却不一定能解决**。

### 4.2 五个失败环节

| 环节 | 失败表现 |
|------|----------|
| **知识源本身没治理** | 文档过期、互相矛盾、没有 owner、同一个概念有好几个名字 —— 矛盾 embed 后变成更难排查的矛盾 |
| **切块切坏** | 条款、表格、代码注释、PDF 页眉页脚被硬切开；模型召回一段文字时，看不出它属于哪一章、哪一版、哪一条 |
| **metadata 太薄** | 只存 `doc_id` 和 `text`，但真实业务检索需要：`effective_date` / `product_version` / `region` / `permission_scope` / `source_owner` / `document_status` |
| **召回只看语义相似** | 企业系统的产品代码、合同编号、错误码、条款号、地区编码同样关键 —— 只靠 embedding 经常漏掉 → **需要 BM25 和稠密向量混合检索** |
| **答案没有引用链** | 用户看到自然语言回答却不知道它来自哪份材料；工程师也分不清问题出在召回、重排、生成，还是知识源本身 |

### 4.3 终极视角

> 把这些问题放在一起看，**RAG 其实是一条知识供应链**。
>
> 供应链做的从来不是简单搬运。它要把**正确的东西，在正确的时间，以正确的包装，送到正确的人手里**。
>
> RAG 要做的，是把**正确的证据，按当前任务的约束，带着可追溯的出处，送到模型面前**。

---

## 五、企业 RAG 的第一件事：知识源和数据治理

### 5.1 先后顺序

> 很多企业 RAG 项目开会时，**第一张 PPT 会写技术栈**：
>
> 向量数据库、embedding 模型、reranker、LLM、LangChain 或 LlamaIndex。
>
> **技术栈当然要选，但它不该是第一件事**。
>
> **真实的 RAG 项目，第一件事应该是看知识源**。

### 5.2 证据表法

> 拿一批真实问题，做一张证据表。

| 用户问题 | 正确证据 | 证据来源 | 证据 owner | 版本 / 生效期 | 权限范围 |
|----------|----------|----------|------------|--------------|----------|
| ... | ... | ... | ... | ... | ... |

> **这张表比一开始就建索引更重要**，因为它会暴露几个基础问题：
>
> - 哪些知识**没有 owner**
> - 哪些文档**已经过期**
> - 哪些口径**互相冲突**
> - 哪些材料**不能给当前 Agent 看**
> - 哪些答案**根本没有可引用证据**

> **如果这张表都填不出来，RAG 项目很容易变成"把企业知识的混乱自动化"**。
>
> Agent 看起来更快了，**错误传播也更快了**。

### 5.3 企业 RAG 落地顺序

| 步骤 | 动作 |
|------|------|
| **1** | 先收集真实问题和正确证据，再给知识源补 **owner、版本、生效日期和权限** |
| **2** | 然后定义 **chunk schema、citation schema 和 index manifest** |
| **3** | 接着建 **candidate index**，用 golden questions 做回归 |
| **4** | 通过以后再切 alias 或 current pointer |
| **5** | 最后接入**答案引用、RetrievalTrace 和索引版本监控** |

> **等这些基础打稳，再考虑 Agentic RAG、多轮检索和 LLM Wiki**。

---

## 六、向量索引库怎么建：先有 manifest，再有 vector DB

### 6.1 索引 = 软件资产

> 一套 RAG 索引应该说得清楚：
>
> - 它来自哪些源文档
> - 用哪个解析器、哪个切块策略、哪个 embedding 模型生成
> - 当前在线的是哪个版本，上一版在哪里
> - 出问题以后能不能回滚
> - 某个答案到底用了哪一版索引里的哪个 chunk

### 6.2 六步索引建设流程

| 步骤 | 名称 | 关键产出 |
|------|------|----------|
| **1** | **知识源登记** | 每份文档 `doc_id` / `source_uri` / `owner` / `permission_scope` / `effective_from` / `effective_to` / `source_hash` / `document_status` |
| **2** | **可复现的切块** | 记录 `parser_version` / `chunker_version` / `chunk_index` / `text_hash`，让 `chunk_id` 稳定生成 |
| **3** | **Ingestion manifest** | 索引构建参数：corpus_version / parser / chunker / embedding_model / collection / source_count / chunk_count |
| **4** | **候选索引 + 灰度发布** | 先生成 candidate 索引，用 **golden questions** 做回归测试 |
| **5** | **蓝绿切换** | 应用访问稳定 `alias`（如 `payroll_rag_current`），底层动态切换 collection |
| **6** | **删除、退休、备份、回滚** | 准备上一版在哪 / 当前版构建参数 / 能否快速切回 / 误入库能否定位清掉 |

### 6.3 chunk_id 生成公式

```python
chunk_id = hash(
    doc_id
    + parser_version
    + chunker_version
    + str(chunk_index)
    + text_hash
)
```

### 6.4 index_manifest 完整示例

```yaml
raw_docs:
  - payroll-bonus-policy-2026-v2.pdf

index_manifest:
  corpus_version: payroll-policy-2026-06-14
  parser_version: pdf-parser-2.4
  chunker_version: clause-aware-v2
  embedding_model: bge-m3
  embedding_dim: 1024
  hybrid_index_version: bm25-cn-v1+dense-v3
  collection: payroll_rag_20260614_candidate
  alias_after_release: payroll_rag_current
  source_count: 1842
  chunk_count: 53218
  golden_set_passed: false   # C 没过回归不许上线
```

### 6.5 蓝绿发布与回滚

> 这和传统软件里的**蓝绿部署**很像：
>
> - 旧索引是 **blue**
> - 新索引是 **green**
>
> **新索引先在旁边构建、验证、灰度**，通过以后再把 alias 或 current pointer 指过去。
>
> - **Milvus**：collection alias —— 应用访问稳定别名，底层 collection 可以动态切换
> - **LanceDB**：版本化写入和 time-travel，索引可以**回到历史版本**

> **这样的索引不再是一坨"建好就放那儿"的向量**，而是一份可以发布、灰度、切换、回滚的**软件资产**。

---

## 七、工程现场切片

### 切片对比总览

| 切片 | 来源 | 核心问题 |
|------|------|----------|
| 切片一 | **Anthropic Contextual Retrieval** | chunk 失去上下文怎么办 |
| 切片二 | **LlamaIndex → Agentic Document Processing** | 文档进入系统前要先干净 |
| 切片三 | **执行型 Agent 中的 RAG** | RAG 不能接管执行 |

### 切片一：Anthropic Contextual Retrieval（2024-09）

> 解决：**文档一旦被切成 chunk，chunk 很容易失去上下文**。
>
> chunk 一旦脱离它的来源语境，召回的结果就**撑不起可追溯的结论**。

**反例**：

```
本责任在等待期后生效。
```

> 这句话单独拿出来几乎没法用。
>
> - 它属于哪款产品？
> - 哪一版条款？
> - 等待期是 90 天还是 180 天？
> - 这段文字是在"重大疾病保险金"下面，还是在"轻症疾病保险金"下面？

**Contextual Retrieval 的做法**：在每个 chunk 前面加一段短上下文，再一起进入 embedding 和 BM25 索引。

```
[Context]
这是 ACME 重疾险 2022 版条款中"重大疾病保险金"一节。
本节说明等待期后的赔付条件，适用于 2022-01-01 至 2023-12-31 生效保单。

[Chunk]
本责任在等待期后生效。
```

**Anthropic 测试结果（top-20 chunk 检索失败率）**：

| 策略 | 失败率 |
|------|--------|
| 朴素检索 | 5.7% |
| Contextual Embeddings | 3.7% |
| + Contextual BM25 | 2.9% |
| + Reranker | **1.9%**（相对降低 67%）|

**最佳适用场景**：

| 适合 | 不太适合 |
|------|----------|
| 条款、合同、法规等强章节结构文档 | 产品卡片、短 FAQ、单条知识条目 |
| 财报、研报、手册等上下文依赖强的 PDF | —— |
| 企业内部制度、FAQ、历史工单（版本和适用范围很重要）| —— |

**同期对比方案：Jina late chunking（2024-09）**

> 同样解决切片时上下文丢失，但**做法不同**：
>
> 先用长上下文模型把整篇文档的所有 token 嵌好，**再在 pooling 前切块**，让每个 chunk 天生带着跨段语义。
>
> BEIR 多个数据集测试：**稳定优于朴素切块，而且文档越长，增益越大**。

### 切片二：LlamaIndex —— RAG 前面还有文档工程

> LlamaIndex 早期几乎就是 RAG 框架的代表之一。
>
> 但 **2026 年它不再只把自己定位成 RAG framework**，而是转向 **智能文档处理（agentic document processing）**，也就是**面向 Agent 的文档处理基础引擎**。

**为什么转型**：

> 他们观察到很多 RAG 失败，**在文档进入系统之前就埋下了**。

**常见 PDF 失败**：

- 表格被 OCR 读乱
- 页眉页脚混进正文
- 双栏论文阅读顺序错了
- 合同里的附件和主条款断开
- 财报图表只剩一句"图 4.2"

> **这些问题一旦进入索引，后面再怎么换向量库都很难救回来**。

**LlamaIndex 的诊断问题清单**：

> 抽 50 条真实用户问题，再把对应应该命中的原始材料找出来，人工检查材料进入系统后的形态：
>
> - 原始文档有没有被正确解析？
> - 表格有没有保留行列关系？
> - 图表里的关键数字有没有进入可检索文本？
> - chunk 能不能追到页码、章节、条款号？
> - 引用链能不能回到原始文件？
>
> **如果这些问题答不上来，先别急着调 top-K。你的 RAG 还没到 retrieval 优化阶段**。

### 切片三：执行型 Agent 中的 RAG

**三类用户输入**：

| 类型 | 处理 | RAG 角色 |
|------|------|----------|
| **闲聊** | 直接结束 | 不需要 |
| **分析型任务** | 查知识库后回答 | **RAG 主力** |
| **执行型任务** | resolve 流程：推理 → 编排 → 工具调用 → 状态绑定 → 人审 → 验证 | RAG **辅助**，不能接管 |

**反例对比**：

| 输入 | 类型 | 处理 |
|------|------|------|
| "为什么上海市场部 6 月奖金异常这么多？" | **Analyze** | RAG 检索奖金政策、历史异常解释、部门规则，给出分析 |
| "把这些异常都处理掉，能自动通过的就过，不能的提交人审" | **Execute** | RAG 提供"哪些异常要人审"的业务证据，**员工 id / 批次 id / 审批单号 / 金额 / 状态流转** 走 SessionState + 工具 |

---

## 八、知识检索技术选型卡

> **RAG 不是万能入口**。把它放回 Agent 系统，它要和另外几条路科学分工。

| 技术路径 | 适用场景 | 与 RAG 关系 |
|----------|----------|--------------|
| **结构化查询** | "这个批次现在什么状态" / "这张审批单过没过" / "员工本月社保基数" | 优先查业务系统，**别绕到 RAG**。让 LLM 从召回文字里"读"一个金额或 id 是执行型 Agent 最常见的事故源 |
| **RAG** | 文档问答、政策检索、案例查找 | **承接 analyze**，辅助 execute |
| **LLM Wiki**（Karpathy 2026-04）| 知识"**编译一次、持续更新**" | **补上 RAG 不擅长的一类需求**：长期知识策展和复利积累。不是 RAG 的下一代，而是**互补** |
| **记忆框架**（Mem0 / Zep / Letta）| 跨会话的用户状态演化 | **和 RAG 互补，不要混为一谈，更不要在一个系统里同时上两套** |

### LLM Wiki 的特别说明

> **2026 年 4 月，Karpathy 提出**与其每次提问都从原始文档重新召回、重新综合，不如让 LLM 在入库时就把原始资料**一次性编译成一组结构化、可交叉引用的 markdown 页面**，知识"**编译一次、持续更新**"，好答案本身又变成新的 wiki 页。

> **我不建议把它讲成"RAG 的下一代"**。更准确的说法是，它补上了 RAG 不擅长的一类需求，**长期的知识策展和复利积累**。
>
> 到了大型企业知识库，还要再补权限、审计、多人协作和审核流程。

---

## 九、证据契约：RAG 到底应该返回什么

> 既然 RAG 要送的是证据，我们就先给"证据"下一个工程定义 —— **证据契约（Evidence Contract）**。
>
> **一条可用的证据，至少要有四件套**：

| # | 字段 | 含义 |
|---|------|------|
| 1 | **source** | 来源：原始文档、系统、表、API、文件路径 |
| 2 | **version** | 版本：文档版本、索引版本、生效时间、废止时间 |
| 3 | **scope** | 范围：适用地区、租户、角色、产品、流程阶段、权限 |
| 4 | **citation** | 引用：页码、章节、条款号、`chunk_id` / `source_hash` / `index_version` |

### 9.1 Naive RAG vs 企业级 RAG 的对比

| Naive RAG | 企业级 RAG |
|-----------|------------|
| 奖金增长 40% 自动通过 人审 | `semantic_query: 奖金增长异常 自动通过 人审 审批阈值`<br>`filters: tenant_id=acme, region=上海, payroll_month=2026-06, employee_group=市场部, rule_status=active, permission_scope=payroll_operator`<br>`required_citation: [rule_id, effective_from, source_owner, section, index_version]` |

> 这样一来，RAG 的任务就从"**找相关文字**"，变成"**按当前业务上下文取回可引用的证据**"。
>
> **这一步看着只是多加了几个过滤字段，它其实是 naive RAG 和企业级 RAG 的分水岭**。

### 9.2 完整 Schema 代码

```python
from dataclasses import dataclass, field
from datetime import date
from enum import Enum
from typing import Any

class DocumentStatus(Enum):
    ACTIVE = "active"
    SUPERSEDED = "superseded"
    DRAFT = "draft"
    RETIRED = "retired"

@dataclass
class IndexManifest:                 # A 一版索引怎么构建、发布到哪里
    corpus_version: str
    collection_name: str
    alias: str                       # B 应用只认别名，底层版本随时可切
    parser_version: str
    chunker_version: str
    embedding_model: str
    embedding_dim: int
    built_at: str
    source_count: int
    chunk_count: int
    golden_set_passed: bool = False  # C 没过回归不许上线

@dataclass
class EvidenceChunk:                 # D 不只存文本，还存版本、来源、权限、生效期
    chunk_id: str
    doc_id: str
    index_version: str
    source_hash: str
    chunk_hash: str
    text: str
    context: str                     # E Contextual Retrieval 补进来的上下文
    source_uri: str
    page: int | None = None
    section: str | None = None
    effective_from: date | None = None
    effective_to: date | None = None
    region: str | None = None
    tenant_id: str | None = None
    permission_scope: str = "internal"
    status: DocumentStatus = DocumentStatus.ACTIVE
    metadata: dict[str, Any] = field(default_factory=dict)

    def is_usable_on(self, day: date) -> bool:   # F 召回到不等于能用，先过期检查
        if self.status != DocumentStatus.ACTIVE:
            return False
        if self.effective_from and day < self.effective_from:
            return False
        if self.effective_to and day > self.effective_to:
            return False
        return True

@dataclass
class EvidenceRequest:               # G 语义查询 + 业务过滤 + 必带引用
    semantic_query: str
    filters: dict[str, Any]
    required_citations: list[str]
    mechanical_state_refs: list[str] = field(default_factory=list)  # H 依赖哪些机械状态，但不交给 RAG 生成

@dataclass
class RetrievalTrace:                # I 一次检索的完整留痕
    request: EvidenceRequest
    index_version: str
    candidates: list[str]
    reranked: list[str]
    used_in_answer: list[str]
    missing_reason: str | None = None
```

### 9.3 Schema 五个设计点

| # | 设计点 | 含义 |
|---|--------|------|
| A | **`IndexManifest`** | 记录一版索引怎么构建、发布到哪里 |
| B | **`alias`** | 应用只认别名，底层版本随时可切 |
| C | **`golden_set_passed`** | 没过回归**不许上线** |
| D | **`EvidenceChunk`** | 不只存文本，还存**版本、来源、权限、生效期和 hash** |
| E | **`context`** | Contextual Retrieval 补进来的上下文 |
| F | **`is_usable_on()`** | 召回到**不等于能用**，先过期检查 |
| G | **`EvidenceRequest`** | 语义查询 + 业务过滤 + 必带引用 |
| H | **`mechanical_state_refs`** | 依赖哪些机械状态，但**绝不把这些机械值交给 RAG 生成** |

---

## 十、RetrievalTrace：给检索装一块仪表盘

### 10.1 为什么需要

> 最终呈现给用户的是一段自然语言答案，**看不出 Agent 当时手里到底有没有正确证据**。
>
> 一次错误回答，可能错在四个完全不同的方面：

| 错误来源 | 问题归属 |
|----------|----------|
| **没召回到正确证据** | 召回问题（query、索引、过滤的锅）|
| **召回到了但被重排压下去** | 重排问题 |
| **召回到了也用了，但材料是过期版本** | 知识源问题 |
| **材料没问题，模型却没采纳** | 推理或规划问题 |

> **没有检索追踪，这四种错误靠日志也排查不出问题，你只能靠猜**。

### 10.2 RetrievalTrace 字段

每次检索都留下：

| 字段 | 含义 |
|------|------|
| `request` | 这次发的证据请求 |
| `index_version` | 用的哪一版索引 |
| `candidates` | 召回了哪些 candidate |
| `reranked` | 重排后留下谁 |
| `used_in_answer` | 最后哪几条真的进了答案 |
| `missing_reason` | 没召回到时的原因 |

> **能从一次错误回答，反查到具体是哪一版索引里的哪个 chunk 出了问题**，这也是企业 RAG 和玩具 RAG 的一个重要分界线。

---

## 十一、总结一下

### 11.1 RAG 的本质

> 虽然检索增强模式在记忆模式组的关键字是"**取**"，但它**绝不只是 Agent 的"搜索框"**，而是 **Agent 的知识供应链**。
>
> - **朴素 RAG** 像一个只管把相似的货堆过来的搬运工
> - **生产级 RAG** 是一条带批次、带凭证、带召回能力的**供应链**

### 11.2 供应链五要素映射

| 供应链概念 | RAG 映射 |
|------------|----------|
| **货从哪来** | `source` 来源 |
| **是不是这一批的版本** | `version` 版本 |
| **能不能发到这个区域** | `scope` 范围 |
| **运输途中有没有凭证可追** | `citation` 引用 |
| **出了质量问题能不能召回** | `RetrievalTrace` |

### 11.3 长程 Agent 的真实痛点

> 长程 Agent 的真实痛点，是**每一步看起来都合理，合在一起却慢慢跑偏**。
>
> RAG 在这里的责任，是**减少"凭印象判断"**，让 Agent 在下结论前先拿到**当前任务适用的证据**，**而不是先拿到一段读着很顺的相似文本**。

### 11.4 三件事合在一起

> 分层保留解决"**哪些记忆常驻、哪些按需加载**"，
> RAG 解决"**外部大库里的哪一点证据该被取回来**"。
>
> 在执行型 Agent 中：
>
> - **RAG 管业务证据**
> - **SessionState 管机械真值**
>
> **两者都带着信息来源，最后在决策点合流**。

> 越是把**知识源、索引、引用、权限、机械状态边界和 trace**放在一起看，**RAG 越接近生产**。**这几样就越不能分开**。

---

## 十二、精选留言

### 留言 1：梓威 — RAG 与 Bitter Lesson

> 想和咖哥讨论一下，我之前研究生是做 NLP 的（那时候大模型还没出来），我感觉**现在的 RAG 很像当初 LLM 没出来时的 NLP**，各种在细节上调优，但是后来被 LLM 无情的碾压了（**Bitter Lesson**：任何依赖人类先验知识、手动设计的启发式规则最终都会被"通用算法 + 算力规模"碾压）。
>
> **RAG 有没有可能遇到类似的问题**？

> **作者回复**：当然有可能。但是我们学过的每一个理论知识都不白学。
>
> **现在也没有任何人工作时手工开平方，手工求导**。
>
> - 求导、开平方、傅里叶变换，学的是**思想**
> - 求导、开平方、傅里叶变换，工具以**集成**

### 留言 2：街角·陌路△ — RAG 当成记忆很合适 + 卢向东老师的企业 RAG 观点

> 我觉得 **RAG 当成记忆很合适**，因为 RAG 也相当于是记忆的一种分层的体现。我们没有办法把所有内容都存入到大模型。但是 RAG 相当于**外挂的记忆**。就好比我们人脑没办法把所有内容都存到大脑里，但是我们可以记一些**关键的信息索引**，需要的时候去对应的笔记或者书本中查看就好。
>
> 所以记忆中是否就应该做一些引导，把**知识库本身去进行分层**，比如不同业务用到什么知识库，把它放到记忆里，然后再让大模型调用工具去调用不同的知识库，而不是一股脑把所有内容都放到同一个知识库里。
>
> 还有一个问题，就是我们说到把 RAG 归到链式或者循环式，那基本都说的是传统 RAG。但如果现在的 **LLM Wiki** 是不是可以把它归成层级式的，**gbrain** 可以归成编排式的，但是又感觉它归类起来很模糊，这个问题老师是怎么看待的？

> **作者回复**：相当精辟的分享。是的，**RAG 是一个企业至今还广泛应用的记忆机制**。它和 LLM Wiki 是互补的，并非谁取代谁。
>
> 这个问题我和专门做企业知识库的**卢向东老师**探讨了一下：
>
> ———
>
> **他认为**：不能说 rag 被彻底颠覆，如果企业文件数量是指数级的，向量搜索的优势依然存在，**llmwiki 在数据量不大的情况下**让大模型把知识读取一遍摘要构造成文件树结构确实是一种 **agentic rag 渐进式搜索的方向**，这里面要平衡性能和成本。llmwiki 会读所有入库的文件，递归总结摘要。构建索引，这个里面的耗时和成本需要做取舍。
>
> 我们现在是**两者都用**，比如像银行里面的很多银行，在**上层会先做域的构建**，其实也是帮助系统去理解用户的问题是针对哪个一级、二级域的，然后再使用 RAG 来检索事实证据。比如手机银行里面，有人问昨天我卡里面有一笔 123 元的消费扣款，是怎么回事。系统会先**根据意图找到这是属于信用卡域的消费记录域**，然后再通过 **NL2SQL 和向量/BM25 混合检索**去找证据。最后再让大模型整理之后输出。
>
> 也就是说，**如果仅仅是个人知识库，小体量的，我觉得 LLM Wiki 应该是成立的**。但是碰到企业级的海量数据，目前看来 **RAG 还是无法避免的**。
>
> ———
>
> 我认为他这些是来自于企业落地过程中的真实感悟，在此分享给大家。
>
> **至于 RAG 的各种拓扑结构，我同意你的看法，是模糊的，不必纠结，管用就行**。：）

### 留言 3：Geek_7f0e61 — 文档治理第一步

> 我们项目组中架构/方案设计文档很多，当前使用 Rag，但是缺少 owner，版本，权限等信息，也缺少检索评估。**感觉路还很长**。但佳哥建议方案文档使用 llmwiki，应该如何选择？

> **作者回复**：**第一步应该把文档版本和相互之间的关系整理清楚**。
>
> 即使使用 RAG，**也应该保留好元数据翻译文档之间的关联**。

---

## 十三、思考题

> 这一讲我们把 RAG 放回记忆层，定位成一种**证据取回机制**。现在回到你自己的系统，做一次小审计。

### 题 1：证据源审计

> 挑 **20 条真实用户问题**，逐条问：
>
> - 正确答案依赖的证据，**真的存在于知识源里**吗？
> - 这份证据有没有 **owner、版本、生效日期和权限范围**？

### 题 2：召回质量审计

> 你的 RAG 现在召回的，**是当前任务适用的证据，还是只是语义相似的材料**？
>
> - 能不能从一段回答，**追回它用的原文页码、条款号、索引版本或文件路径**？

### 题 3：边界审计

> 在你的系统里，**哪些信息该走 RAG，哪些必须走结构化查询或 SessionState**？
>
> - 有没有**机械状态**（id、金额、批次号）正在被 RAG 或 LLM 的自由文本**悄悄污染**？

> **这三个问题的目的，是帮你判断你遇到的问题，到底该靠优化 RAG 解决，还是根本就不该交给 RAG**。

---

## 十四、下一讲预告：进度追踪

> 下一讲我们讲**记忆模块的第三个模式——进度追踪（Progress Tracking）**。
>
> - RAG 解决的是"**从大库里取回该用的证据**"
> - 进度追踪解决的是另一个问题：**一个长任务跑到一半，Agent 怎么记住自己已经做了什么、为什么这么做、下一步该怎么接**
>
> 导论里那个 **auth.py 重构事故，真正对症的药就在下一讲**。

→ [[lession16-进度追踪]]

---

## 十五、参考资料

> | # | 来源 |
> |---|------|
> | 1 | Patrick Lewis et al. **Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks**. arXiv:2005.11401, NeurIPS 2020. |
> | 2 | LangChain (Ankush Gola). **Self-Reflective RAG with LangGraph**. 2024-02-07. |
> | 3 | Anthropic. **Introducing Contextual Retrieval**. 2024-09-19. |
> | 4 | Jina AI. **Late Chunking in Long-Context Embedding Models**. arXiv:2409.04701, 2024-09. |
> | 5 | Singh et al. **Agentic Retrieval-Augmented Generation: A Survey on Agentic RAG**. arXiv:2501.09136, 2025-01 (v4 2026-04). |
> | 6 | Chroma Research. **Context Rot: How Increasing Input Tokens Impacts LLM Performance**. 2025-07-14. |
> | 7 | LaRA: **Benchmarking Retrieval-Augmented Generation and Long-Context LLMs**. arXiv:2502.09977, ICML 2025. |
> | 8 | Anthropic. **Effective Context Engineering for AI Agents**. 2025-09. |
> | 9 | LightOn (Amélie Chatelain). **RAG is Dead, Long Live RAG: Retrieval in the Age of Agents**. 2025-11-12. |
> | 10 | **A-RAG: Scaling Agentic Retrieval-Augmented Generation via Hierarchical Retrieval Interfaces**. arXiv:2602.03442, 2026-02. |
> | 11 | LlamaIndex. **LlamaIndex is more than a RAG Framework. It is Agentic Document Processing**. 2026-03-03. |
> | 12 | Andrej Karpathy. **LLM Wiki**. gist, 2026-04-04. |
> | 13 | Milvus Docs. **Manage Aliases**. |
> | 14 | Pinecone Docs. **Backups overview**（备份 / 恢复 2026-03 GA）. |
> | 15 | Qdrant Docs. **Snapshots**. |
> | 16 | LanceDB Docs. **Versioning and Reproducibility**. |

---

## 附：本讲核心要点速查

| 关键词 | 含义 |
|--------|------|
| **RAG 的双轴** | Naive RAG = 记忆 × 链式；Agentic RAG = 记忆 × 循环（加"评估 → 改写 → 再检索"回路）|
| **两类信息** | 机械状态（SessionState 管）/ 业务证据（RAG 管）|
| **RAG 演进** | 2020 Lewis 创立 → 2023 大火 → 2024 泛滥 → 2025 反思（Chroma Context Rot / LaRA）→ 2026 重新定位 |
| **RAG ≠ 相似文本召回** | 是当前任务适用且可引用的证据 |
| **知识供应链** | source / version / scope / citation / trace 五要素 |
| **证据契约四件套** | source 来源 / version 版本 / scope 范围 / citation 引用 |
| **企业 RAG 落地顺序** | 收集真实问题 → 知识源补 owner/版本/权限 → chunk schema → candidate index → golden questions 回归 → 别名切换 → trace 监控 |
| **索引六步流程** | 知识源登记 → 可复现切块 → ingestion manifest → 候选索引 → 蓝绿切换 → 删除退休 |
| **chunk_id 公式** | `hash(doc_id + parser_version + chunker_version + str(chunk_index) + text_hash)` |
| **Contextual Retrieval** | top-20 失败率 5.7% → 3.7% → 2.9% → 1.9%（相对降低 67%）|
| **Late chunking** | 长上下文先 embed 再切块，文档越长增益越大 |
| **LlamaIndex 转型** | agentic document processing（PDF OCR / 页眉页脚 / 双栏阅读顺序 / 图表占位）|
| **执行型 RAG 边界** | RAG analyze / SessionState execute / Orchestrator 编排 / Gate 一致性检查 |
| **结构化查询优先** | "批次状态 / 审批单 / 员工社保基数"别绕到 RAG |
| **LLM Wiki** | Karpathy 2026-04：编译一次、持续更新、好答案变新 wiki |
| **记忆框架 ≠ RAG** | Mem0/Zep/Letta 管跨会话状态演化；不要同时上两套 |
| **Schema 9 字段** | chunk_id / doc_id / index_version / source_hash / chunk_hash / text / context / source_uri / page / section / effective_from / effective_to / region / tenant_id / permission_scope / status |
| **`is_usable_on()`** | 召回到不等于能用，先过期检查 |
| **`mechanical_state_refs`** | 依赖哪些机械状态，但不交给 RAG 生成 |
| **RetrievalTrace 四类错误** | 没召回到 / 重排压下 / 过期版本 / 没采纳 |
| **Naive vs 企业 RAG** | 朴素召回 vs 带 filters + required_citations |
| **三种 RAG 对比** | 朴素 RAG（搬运工）/ Agentic RAG（带回路）/ LLM Wiki（编译型）|