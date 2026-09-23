---
title: "09｜渐进发现：信息的觅食循环"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/985303"
tags:
  - "#pattern/perception"
  - "#axis/perception"
  - "#axis/loop"
  - "#topic/discovery"
  - "#topic/agentic-search"
  - "#topic/rag-vs-search"
  - "#topic/code-exploration"
  - "#topic/foraging-theory"
  - "#concept/atomic-tools"
  - "#concept/information-gain"
  - "#concept/satisficing"
audio: true
duration: "24:49"
publish_date: "2026-06-10"
course_progress: 9
pattern_position: "感知 × 循环"
related:
  - "[[lession8-语义压缩]]"
  - "[[lession12-分层保留]]"
---

# 09｜渐进发现：信息的觅食循环

> **本讲定位**：感知模块第三个模式 → 解决"信息在哪里、怎么找到它"。

| 属性 | 内容 |
|------|------|
| 课程模块 | 感知：世界之美（第 4 讲 / 共 5 讲）|
| 课程编号 | 09 |
| 双轴坐标 | **感知 × 循环**（认知功能：感知 / 执行拓扑：循环）|
| 时长 | 24:49 |
| 发布日期 | 2026-06-10 |
| 前置讲 | [[lession8-语义压缩]] — 信息进来后怎么不压坏 |
| 后续讲 | [[lession12-分层保留]] — 记忆模块导论 |

## 一句话核心

> **渐进发现（Progressive Discovery）= 当 Agent 面对陌生信息空间时，通过广扫（Forage）→ 聚焦（Focus）→ 深挖（Deepen）三阶段循环定位所需信息，每轮带着前一轮的发现去做下一轮决策。**

---

## 一、从反例到动机：为什么 Agent 找不到 bug

### 1.1 上一讲的尾巴与本讲的开头

> 前两讲我们讲了**上下文分诊**和**语义压缩**。
>
> - 上下文分诊管**哪些信息能进 context**
> - 语义压缩管**已经进来的信息怎么压**
>
> **但它们有一个共同前提：你已经知道相关信息在哪。**

不过如果 Agent 面对的是：

| 陌生场景 | 不知道什么 |
|----------|-----------|
| 陌生代码库 | bug 藏在哪一个文件 |
| 没看过的合同 | 关键条款在哪一页 |
| 很长的事故日志 | 异常发生在哪一段时间 |

> **由此我们引入渐进发现（Progressive Discovery）模式，感知模块的第三个模式。**

### 1.2 开篇反例：电商订单邮件 bug

电商系统出 bug：**订单确认邮件偶尔会混入其他客户的订单条目**。

| 项目 | 数字 |
|------|------|
| 代码库规模 | 15000 文件 |
| 文档状态 | 稀疏 |
| 原作者 | 已离职 |
| 邮件 pipeline 路径 | 未知 |

**三种思路的对比**：

| 思路 | 做法 | 结果 |
|------|------|------|
| ① 全量喂入 | 把整个代码库喂给 Agent | ~800K token，超任何窗口；切 100 段并行，每段都说"没看到 bug"（看不到全局）|
| ② RAG 语义检索 | 嵌入向量数据库 | top-K 召回过多无关；bug 用的变量 `merge_user_state` 语义不命中 |
| ③ **grep + read** | Agent 自己探索 | 4 轮 ~18K token，3 分钟定位 |

**第③条路径的探索过程**：

1. `grep send.*confirm` → 30 个候选文件
2. 按文件名挑 5 个最可能 → `mailers/order_confirmed.rb` 跳出
3. 读完 5 个文件看到调用链：`MailerWorker → Cache.get_user → render`
4. 读 `Cache.get_user` → 发现 cache key 用 `order_id`，**但没用 `customer_id`**
5. **bug 找到了**

> **这个 bug 跟语义没关系，跟代码结构有关系**。只有 `grep + read + follow imports` 这种直接接触结构的方式，才能把它找出来。

### 1.3 引出 2025-2026 工业界争论

> 这一讲要讲的，就是**当你不知道相关信息在哪时，怎么定位它的工程细节**。
>
> 这也是 2025-2026 年工业界争论很激烈的一个战场：**Agentic Search vs RAG**。开篇这个故事，就是这场辩论的一个小切片。

---

## 二、什么是渐进发现模式

### 2.1 模式定义

> 渐进发现模式的要义是：
>
> 当 Agent 面对**陌生信息空间**（陌生代码库 / 文档库 / 陌生数据源），通过**广扫、聚焦、深挖三阶段循环**定位自己所需要的信息。

| 阶段 | 中文 | 英文 | 动作 |
|------|------|------|------|
| 1 | 广扫 | Forage | grep / glob / find 低成本工具 |
| 2 | 聚焦 | Focus | 从候选挑 5-10 个完整读 |
| 3 | 深挖 | Deepen | 沿最强信号链追下去 |

### 2.2 在双轴图谱的位置

| 轴 | 类型 | 原因 |
|----|------|------|
| 认知功能 | **感知** | 决定 Agent 看到什么 |
| 执行拓扑 | **循环** | 一轮广扫、一轮聚焦、一轮深挖，**每轮带着前一轮的发现去做下一轮决策** |

**与前两讲的关键区别**：

| 模式 | 双轴 | 拓扑本质 |
|------|------|----------|
| 上下文分诊 [[lession7-上下文分诊]] | 感知 × 路由 | Router 单点路由 |
| 语义压缩 [[lession8-语义压缩]] | 感知 × 链式 | Chain 三层级联 |
| **渐进发现** | **感知 × 循环** | **迭代式，不是单次完成** |

### 2.3 工业数据支撑

> Anthropic 披露 **2026 Q1 有 78% 的 Claude Code 会话涉及多文件编辑**，而 2025 Q1 才 34%。

**这意味着 Agent 正越来越频繁地探索它不熟悉的区域**。

### 2.4 适用场景 vs 不适用场景

**适用**：

- 陌生代码库的 bug 调试
- 陌生客户的合同审阅
- 陌生事故的根因排查
- 渐进加载的信息可以是：**代码、文档、工具、Skills**

**不适用**（强行用反而画蛇添足）：

| 场景 | 原因 |
|------|------|
| 客服 Agent 接当前用户的工单上下文 | 已在 P1（[[lession7-上下文分诊]] 的工单分级）|
| 研究 Agent 看用户上传的某份文档 | 已在 P0 |

> 这种场景下 **Agent 不需要探索，因为它已经看到了**。

---

## 三、工程现场切片：Agentic Search vs 持久化索引

> 下面挑两个**貌似对立的工程切片**来讲：
>
> - 切片一：Claude Code 为什么**一度放弃 RAG**，转向 Agentic Search
> - 切片二：Augment Code 为什么**又走了持久化索引（persistent indexing）**路线
>
> 这两个切片合起来回答一个问题：**面对陌生代码库，Agent 到底应该"先建索引"，还是"现场探索"？**

### 切片一：Claude Code 的判断 — 代码探索优先 Agentic Search

Claude Code 早期用过 **RAG + local vector db**。

创始人 Boris Cherny 公开在 X 上的表态：

> *"Early versions of Claude Code used RAG + a local vector db, but we found pretty quickly that **agentic search generally works better**. It is also simpler and doesn't have the same issues around security, privacy, staleness, and reliability."*

> *"Internal benchmarks showed that **agentic search outperformed RAG by a lot**, which was surprising."*

**为什么这么判断**？

> RAG + local vector db 多了一层**长期存在的索引副本**；这层副本会带来一整套额外工程问题。
>
> Agentic Search 少了这层，所以问题面小很多。

**两条链路对比**：

| 链路 | RAG | Agentic Search |
|------|-----|-----------------|
| 流程 | 代码仓库 → chunk → embedding → 写入 vector db → top-K 召回 → 塞进 prompt | 当前任务 → grep / search / read file / follow imports → 读当前工作区里的真实文件 |
| **影子仓库** | **有**（vector db = 知识副本）| **无** |

**影子仓库带来的四大工程问题**：

| 问题 | 含义 |
|------|------|
| **Security（安全）** | 代码 / 配置 / 商业逻辑都多了一层暴露面 |
| **Privacy（隐私）** | 即使是私有部署，也带来额外数据治理问题 |
| **Staleness（过期）** | 索引一旦建好就会过时，commit 进来索引必须持续更新，慢一步 Agent 看到旧代码 |
| **Reliability（可靠）** | 召回依赖 embedding、查询表达、top-K、阈值，任何环节抖动结果就会变；grep/read 朴素但稳定、可解释、可复现 |

**黄佳的关键论断**：

> 佳哥绝不想说 RAG 没用，而是**代码探索不能只靠语义召回**。
>
> **代码库不同于普通文档库**。bug 往往藏在**变量名、调用链、缓存 key、配置文件、测试路径**这些**结构关系**里。
>
> - RAG 看的是**语义相似**
> - grep + read + follow imports 看的是**代码结构**
>
> 前者可能召回一堆"语义相关"的文档，后者能顺着**真实调用链**往下追踪。

**工程结论**：

> 在中小型代码库里，如果目标是 bug 定位、调用链追踪、陌生模块理解，**agentic search 往往更直接**。
>
> **先别急着上复杂 RAG，grep + read 应该能解决 70% 的探索问题。**

### 切片二：Augment Code — 超大代码库需要持久化的索引

**但是我们也不能因为 Claude Code 这样说，就认定 grep + read 是渐进加载的全部内容了。Indexing 仍然有它的价值**。

Augment Code 把 **Context Engine** 做成一个工业级索引系统：

| 特性 | 指标 |
|------|------|
| 索引规模 | **400K+ 文件** |
| 全量索引 | 较快 |
| **增量更新** | **45 秒**（commit 后不到一分钟，Agent 就能看到新代码）|
| 跨 repo 依赖 | 支持 |
| 暴露方式 | 通过 MCP 暴露给 Claude Code 和 Cursor 使用 |

**要解决的核心问题**：

| 维度 | 问题 | 解决 |
|------|------|------|
| **新鲜度** | 代码每天都在变，commit 一进来索引如果跟不上，Agent 看到的就是旧代码 | **45 秒增量更新** |
| **规模** | 400K+ 文件不能指望 Agent 每次都从零 grep | 持久化索引 = 提前维护大规模代码库结构 |

**为什么还需要持久化索引**：

> 一个 400K 文件的代码库，如果 Agent 每次都现场 grep，跑几轮探索就可能变成**分钟级延迟**。
>
> 持久化索引的价值，就是**提前维护好大规模代码库的结构**，让 Agent 查询时不需要每次扫整片森林，而是在一张实时更新的地图上探索。

**和 Boris 判断的关系**：

> **这和 Boris 的判断并不冲突**。Boris 说的是 "agentic search generally works better"，**generally** 这个词就意味着这个判断并不是绝对的。
>
> 一个 400K 文件的代码库，如果 Agent 每次都现场 grep，跑几轮探索就可能变成分钟级延迟。**这个时候，持久化索引的价值就出来了**：它把一部分探索成本提前支付掉，用实时增量和跨 repo 索引换取交互时的速度。

### Agentic Search vs Indexing 选型表

| 场景 | 推荐 | 理由 |
|------|------|------|
| 中小型代码库 | **Agentic Search** | 现场 grep+read 即可，70% 探索问题可解 |
| 超大规模 / 跨 repo / 强依赖追踪 | **持久化索引**（Augment 路线）| 持久化地图 + 45s 增量更新 |
| 隐私敏感场景 | **Agentic Search 优先** | 不留影子仓库 |
| 灰色地带 | **混合** | Cursor 式折中：能现场探索时现场探索，必须提前建图时提前建图 |

> 持久化索引是 **Agentic Search 的规模化补丁**。小中型代码库可以现场探索，超大代码库和跨 repo 场景需要提前维护一张代码地图。

---

## 四、三阶段循环：广扫 → 聚焦 → 深挖

> Agentic search 真正落地时，**绝对不能让 Agent 碰运气乱搜**。它真正的工程骨架，是**广扫、聚焦、深挖（forage-focus-deepen）三阶段循环**。

> 三阶段循环 — **Pirolli & Card 1999 信息觅食理论**在 LLM Agent 上的复刻

### 4.1 第一阶段：广扫（Forage）

| 维度 | 内容 |
|------|------|
| 工具 | `grep` / `glob` / `find` 低成本工具 |
| 目标 | 拿到 **30-50 个候选** |
| 关注点 | 文件名、路径、匹配行、周边上下文（**不读完整文件**）|
| 代价 | **几千 token 级别** |

### 4.2 第二阶段：聚焦（Focus）

| 维度 | 内容 |
|------|------|
| 工具 | `read` 完整读 |
| 目标 | 从候选里挑 **5-10 个最可能**完整读 |
| 关注点 | 谁调用谁、关键函数在哪、哪个文件可能处在主路径上 |
| 案例 | 开头 `mailers/order_confirmed.rb` 就是在 Focus 阶段跳出来的 |

### 4.3 第三阶段：深挖（Deepen）

| 维度 | 内容 |
|------|------|
| 工具 | `read` / `git log` / `follow imports` |
| 目标 | 沿 Focus 阶段发现的可疑链继续追下去 |
| 关注点 | 被调用函数、配置文件、测试用例、历史 commit |
| 边界 | **不能再铺开，只追一两条最有信号的链** |
| 案例 | 追到 `Cache.get_user`，发现 cache key 缺少 `customer_id` |

### 4.4 DiscoveryTrace：渐进发现的"行车记录仪"

> 左边的 **12 个字段作为 DiscoveryTrace**，是渐进发现过程里的"行车记录仪"。
>
> Agent 每走一步探索，都要留下这类 trace：

```python
@dataclass
class DiscoveryEvent:
    phase: str             # FORAGE / FOCUS / DEEPEN
    query: str             # 这一轮用的搜索 query
    result_count: int      # 拿到多少候选
    selected_count: int    # 选中了几个
    reason: str            # 为什么选
    tokens_used: int       # 花多少 token
    cost: float            # 成本
    next_phase: str        # 下一步准备进入哪个阶段
```

**示例**：

```
phase = BROAD_SEARCH
query = "k8s pod oom"
result_count = 47
selected_count = 5
next_phase = FOCUS
```

> 意思是 Agent 现在还在广扫阶段，用低成本搜索拿到了 47 个候选，但**只挑出 5 个进入下一阶段**。它不是一看到结果就全读，而是在**控制探索成本**。

### 4.5 Pirolli & Card 三判断指标

| 指标 | 含义 | 阶段 |
|------|------|------|
| **`information_gain`** | 单位成本拿到多少信息 | Forage |
| **`patch_quality`** | 当前信息斑块值不值得继续挖 | Focus |
| **`marginal_value`** | 继续深挖的边际收益 | Deepen |

**`information_gain`**：

> 一次 grep、glob、find 如果很便宜，却能暴露一批高相关候选，那 information gain 就高，**值得继续扫**。
>
> 反过来，如果搜了很多次都只是噪声，就该**换 query 或换方向**。

**`patch_quality`**：

> patch 可以理解成"**信息斑块**"，比如一个目录、一个模块、一组日志时间段、一个合同章节。
>
> 聚焦阶段会判断：这批候选里，**哪个 patch 的平均相关性最高？哪个最值得精读？**
>
> 所以 Agent **不再全局乱搜，而是开始收敛到少数高质量区域**。

**`marginal_value`**：

> Agent 沿着调用链、配置、测试、commit 往下追时，要不断问：**继续读下去还赚钱吗**？
>
> 如果每多花 1000 token 都能拿到新线索，就继续 deepen；
>
> 如果读到后面全是重复信息，**边际收益下降，就该停止、回退，或者换一条链**。

### 4.6 三阶段 → 三指标的映射

| 阶段 | 工具 | 评估指标 |
|------|------|----------|
| **Forage** | 低成本工具广扫 | 追求 `information_gain` |
| **Focus** | 从候选选高质量区域 | 判断 `patch_quality` |
| **Deepen** | 沿最强线索深追 | 观察 `marginal_value` |

### 4.7 第四阶段：验证（Verify，可选）

> 最后，还可以对上面的三个阶段进行补充，**增加验证 Verify 部分**。
>
> 先把搜索面铺开，再把范围收窄，然后沿着最强线索钻下去，**最后用反例和边界条件验证自己有没有找错**。
>
> 这样就形成了真正的闭环，从普通搜索的一次性追问，发展到**有边界的探索**：**现在的信息增益还值不值得继续投入 token？**

### 4.8 工程细节：三个硬纪律

| 纪律 | 内容 |
|------|------|
| **① 工具 atomic 化** | 不要只给一个 `search_codebase()`；`grep` / `glob` / `read` 分开，Agent 才能自己组合探索路径 |
| **② 循环设上限** | 最多 **2-3 轮**，单轮 **20K token**；找不到就交给人，不要让 Agent 无限烧 token |
| **③ 记录 trace** | 每轮用了哪些关键词、召回多少候选、读了哪些文件、最后追到哪里 —— **Discovery 的价值不只是最后答案，也包括中间证据链** |

### 4.9 单循环成本估算

> 一个完整循环大约是 **18K token**。正常情况下，**一轮就能解决大部分探索任务**；信号不够时，再带着新发现的关键词回到 forage 广扫，开始第二轮。

---

## 五、8 框架横切：每家怎么做 Discovery

> 把主流 Agent 框架横向看一遍，会发现渐进发现这个模式是 **Coding Agent 正在收敛出来的工程共识**。

### 5.1 横切的三个判断

| # | 判断 | 含义 |
|---|------|------|
| 1 | **Agentic Search 已经是 Coding Agent 的主流共识** | 代码里的关键信息经常藏在**结构关系**里，而非藏在**语义相似度**里 |
| 2 | **持久化索引是 Agentic Search 的规模化补丁** | 小中型代码库可以现场探索，超大代码库和跨 repo 场景需要提前维护一张代码地图 |
| 3 | **深挖时不应该污染主 Agent 的上下文** | 更成熟的做法是把这个深入探索过程**隔离出去**：让 Sub-Agent 或独立 search worker 负责广扫和初筛，**主 Agent 只拿回压缩后的发现、证据链和候选文件** |

> 这一点非常重要。**长期运行的 Agent 最怕把探索过程中的中间垃圾全部塞进主上下文**，最后真正有用的信息反而被淹没。

### 5.2 Cursor 的折中形态

> Cursor 则更像**混合路线**：
>
> - **短 session、局部修改、当前文件附近的问题**，可以用 agentic search
> - **跨 repo、跨模块、长期项目上下文**，就需要索引托底
>
> 这其实是未来很多 Coding Agent 会采用的折中形态，**能现场探索时现场探索，必须提前建图时提前建图**。

### 5.3 收敛方向

> 不同框架的实现方式不同，但方向正在收敛。**渐进发现的关键是一套纪律**：
>
> - 用**原子级别**的工具探索
> - 用**代码索引**控制规模
> - 用 **Sub-Agent 隔离噪声**

---

## 六、工业级实现：从骨架到生产

> 这一节的完整代码可以参考代码库中的具体实现。
>
> 渐进发现落到工程里，可以分成**三层**：**最小骨架、业务装配、生产观测**。

### 6.1 第一层：最小骨架

> 它回答的问题是：**广扫、聚焦、深挖这三阶段，代码上怎么组织**。

**最小骨架只需要五个核心对象**：

| 对象 | 作用 |
|------|------|
| `Phase` | 当前阶段枚举：`FORAGE` / `FOCUS` / `DEEPEN` |
| `Candidate` | 候选目标（一个文件、一条日志、一段 trace） |
| `DiscoveryEvent` | 一次探索动作的记录 |
| `DiscoverySession` | 一次完整探索的记录 |
| `ProgressiveDiscoverer` | 执行器，把 grep / read / scorer 三个 atomic tools 组合跑三阶段循环 |

**`Candidate` 至少要包含**：

```python
@dataclass
class Candidate:
    path: str        # 文件路径
    snippet: str     # 摘要片段
    score: float     # 评分
    reason: str      # 入选理由
```

**`DiscoveryEvent` 至少要包含**：

```python
@dataclass
class DiscoveryEvent:
    query: str
    input_count: int
    output_count: int
    files_read: int
    tokens_used: int
    duration_ms: int
```

**`DiscoverySession` 至少要包含**：

```python
@dataclass
class DiscoverySession:
    task: str
    cycles: int
    final_files: list[str]
    success: bool
    total_tokens: int
```

**最关键的三条工程纪律**：

| # | 纪律 | 含义 |
|---|------|------|
| 1 | **工具 atomic 化** | grep、glob、read、scorer **分开注入**，Agent 才能自己决定先广扫、再精读、再深追。同一套骨架可接本地文件系统 / MCP server / Augment Context Engine |
| 2 | **循环明确上限** | `max_cycles = 3`。三轮还找不到 → 关键词错了 / 任务描述太泛 / 需要人介入，继续循环是白烧 token |
| 3 | **单轮预算** | `budget_per_cycle = 20K token`。forage 阶段如果 grep 出太多个候选，必须**立刻截断** |

### 6.2 第二层：业务装配

> 它回答另一个问题：**这套骨架放到真实业务里，应该加什么字段**。

**以运维事故响应 Agent 为例**：接到告警后，Agent 先生成一个 **IncidentContext**：

```python
@dataclass
class IncidentContext:
    incident_id: str
    severity: str              # P0 / P1 / P2
    alert_metric: str          # latency_p99 / error_rate / memory / cpu
    affected_service: str
    timestamp_iso: str
    sla_minutes_remaining: int
```

**`alert_metric` 决定 keyword 推导**：

| alert_metric | 关键词集 |
|--------------|----------|
| `latency_p99` | `timeout`、`slow query`、`circuit breaker`、`connection pool` |
| `error_rate` | `Exception`、`5xx`、`FAILED` |
| `memory` | `OOM`、`memory leak`、`GC` |

> **keyword 推导其实是领域知识沉淀**。

**领域 keyword 推导示例**：

| 业务 | 关键词 |
|------|--------|
| 金融 | `amount mismatch`、`FX rate`、`settlement failure` |
| 医疗 | `abnormal vitals`、`contraindication` |
| 电商 | `payment timeout`、`inventory mismatch`、`cart abandoned` |

**空间裁剪（重要前置步骤）**：

> 运维日志每天可能有 **10GB**，不可能全量 grep。事故响应里最自然的裁剪方式是**时间窗口**，只看告警前后 15 分钟。

| 业务 | 裁剪维度 |
|------|----------|
| 运维事故 | 时间窗口（告警前后 15 分钟）|
| 合同审阅 | 章节 |
| 研究综述 | 论文类型 |
| 代码库 | 目录 / 模块 |

> **先裁剪，再 forage，否则 token 很快爆掉**。

**业务装配的最终产物 — IncidentEvidence**：

```python
@dataclass
class IncidentEvidence:
    suspect_logs: list[str]          # 可疑日志
    suspect_services: list[str]      # 可疑服务
    recent_deploys: list[str]        # 最近部署
    config_files: list[str]          # 相关配置
    correlated_traces: list[str]     # 相关 trace
```

> 即使 Agent 没有直接找到根因，**这些证据对值班工程师也有用**。
>
> 生产里的 Agent **不一定要自己解决问题，更常见的价值是先把路探出来，再把人带到正确的位置**。

### 6.3 第三层：生产观测

> 它回答最后一个问题：**上线之后，怎么知道 Discovery 系统是不是健康**。

| 指标 | 含义 | 健康区间 | 异常信号 |
|------|------|----------|----------|
| **`cycles_to_success_p50`** | 找到答案需要几轮 | **接近 1** | p50 涨到 2 或 3 → keyword 推导变差 / 业务场景变了 |
| **`forage_to_focus_ratio`** | 广扫阶段 token / 聚焦阶段 token | **0.3 - 0.5** | 太高：关键词太宽（候选太多）；太低：关键词太窄（候选太少）|
| **`zero_signal_rate`** | 三阶段跑完后，完全没拿到有效信号的比例 | **< 5%** | 突然升高 → grep 没查到 / read 权限错 / scorer 排序坏 / 索引已过时 |

> 所以，**渐进发现的生产实现不仅仅是一段"会搜索"的代码，而是一套小系统**：
>
> - 前面有 **atomic tools**，负责接触真实世界
> - 中间有**三阶段循环**，负责控制探索节奏
> - 旁边有**业务上下文**，负责把搜索变成诊断
> - 后面有 **trace 和指标**，负责让整个过程可观察、可调优

---

## 七、Discovery 在生产里的常见卡点

> 这个模式在落地时，**最常见的坑有四个**。

### 坑 1：Forage 关键词太宽

**症状**：用户说"登录变慢了"，Agent 直接 `grep login` 和 `slow`，可能一下返回 **5000 个候选**，Forage 阶段 token 直接爆掉。

**应对**：

- 让关键词更窄：`LoginController`、`auth_timeout`、`session_create` 通常比 `login`、`slow` 更有用
- 生产里可以先用**一个轻量模型**，把用户描述翻译成 **5-8 个精确关键词**，再开始 grep

### 坑 2：Focus 挑错文件

**症状**：Forage 拿到 30 个候选，Focus 本应挑 top-8 精读。但 scorer 把测试文件排在生产文件前面 → 读一堆 `test/spec`，真正的 `services/auth.rb` 没读到。

**解法**：**给 scorer 加业务权重**：

| 优先级 | 文件类型 |
|--------|----------|
| 高 | 生产文件 |
| 中 | 核心目录 |
| 低 | 测试文件 / 边缘目录 / 长期没人碰的文件 |

> **生产文件优先于测试文件，核心目录优先于边缘目录，最近修改优先于长期没人碰的文件**。

### 坑 3：Deepen 追进死胡同

**症状**：Agent 看到一个依赖就一路追下去，最后追到**第三方库源码**，读了几百行也没有任何信号。

**解法**：**给 Deepen 设边界**：

| 规则 | 含义 |
|------|------|
| 不追第三方库 | 除非 bug 报告明确点名 |
| 不追超过 2 跳的依赖 | A → B → C 之后就要停下来判断是否还有价值 |

### 坑 4：Discovery 和 RAG 撞车

**症状**：同一个 Agent 同时跑 Discovery 和 RAG，RAG 召回一批文档，Discovery 又探出另一批文件。**两边结果不一致时，Agent 不知道信谁**。

**解法**：**先定主路径而非简单的混用**。

| 场景 | 主路径 |
|------|--------|
| 小中型代码库 / 隐私敏感 | **优先 Discovery** |
| 超大代码库 / 跨 repo 强依赖 | **优先持久化索引** |
| 灰色地带 | 混合，但必须定义清楚：**什么时候用现场探索，什么时候查索引，冲突时谁优先** |

---

## 八、总结一下

### 8.1 渐进发现的本质

> **渐进发现的本质是会找路的 Agent**。它是**信息觅食理论在 LLM Agent 上的新形态**。

### 8.2 Pirolli & Card 1999 的启示

> Pirolli 和 Card 1999 年提出这个理论，讲的是**人和动物怎么在复杂环境里寻找有价值的信息**。
>
> 觅食者不会把整片森林翻一遍，也不会闭着眼乱走。它会：
>
> 1. 先找**可能有食物的斑块**
> 2. 发现有价值就**深挖**
> 3. **边际收益下降就换地方**

### 8.3 Satisficing 哲学

> 这就是**三阶段循环的本质**。它追求"**足够强的证据**"，找到够用的线索，就停下来；信号不够，再换一组关键词继续探。
>
> 这也是 **Herbert Simon 讲的 satisficing**：不是寻找理论上的最优解，而是在**有限时间、有限 token、有限上下文里**，找到一个**足够好的解**。

### 8.4 工程决策一致性

| 工程决策 | 背后逻辑 |
|----------|----------|
| 为什么要有 `max_cycles`？ | 探索不能无限循环 |
| 为什么要有 token budget？ | Discovery 的目标是**有纪律地看**（不是越多越好）|
| 为什么 atomic tools 比 `search_codebase()` 更重要？ | grep / glob / read 分开，Agent 才能根据上一轮发现**重组下一轮动作** |
| 为什么 agentic search 在很多代码探索任务里比 RAG 效果更好？ | 它**不假设索引已经覆盖一切**，而是让 Agent **带着当前线索逐步逼近答案** |

### 8.5 终极洞见

> 所以设计承担探索任务的 Agent 时，**除了上下文窗口容量外，更要关注怎么强化它的探索能力**，具体就是：
>
> - **广扫的广度**
> - **聚焦的判断**
> - **深挖的克制**
>
> **记住，Discovery 是侦探**。
>
> **给 Agent 原子工具，给它好的 keyword 推导，给它 satisficing 的纪律，比把整个代码库一股脑塞给它更重要**。

> 大家都写出会找路的 Agent。

---

## 九、精选留言

### 留言 1：邋遢的流浪剑客 — codebase-memory-mcp 混合路线

> 其实还是看针对的场景吧，我们的一种场景是在**产品定方案期间会和研发了解目前的一些系统的现状及逻辑**，这种场景就没办法用本地的 grep 这种方案。
>
> 目前是把 [https://github.com/DeusData/codebase-memory-mcp](https://github.com/DeusData/codebase-memory-mcp) 做了改造，然后通过知识库来维护站点的信息及代码仓库的名称配合 codebase 的 mcp 回复产品的问题，目前的方案是这样定的。

> **作者回复**：这套方案是合理的，而且是"**现场搜索 + 持久化索引**"的混合路线。
>
> 产品人员问的通常是"这个站点的退款逻辑怎么走"，他不知道代码在哪，也没有本地仓库可供 grep。此时可以分三层处理：
>
> 1. **站点知识库**先把业务语言映射到相关系统和代码仓库
> 2. **codebase-memory-mcp** 查询函数、类、调用链、接口和跨服务关系
> 3. 遇到关键结论或索引未覆盖的部分，再回到当前分支的源码核对

### 留言 2：学习 — keyword 是怎么推导的？

> 下面这段话有个疑问，**怎么跑**是提前录入的么？Agent 怎么知道内存对应的内容是：OOM、memory leak、GC？
>
> 这些字段决定 Discovery 怎么跑。如果 alert_metric 是 latency_p99，关键词可以从 timeout、slow query、circuit breaker、connection pool 开始。
>
> 如果 alert_metric 是 error_rate，关键词可以从 Exception、5xx、FAILED 开始。如果 alert_metric 是 memory，关键词可以从 OOM、memory leak、GC 开始。

> **作者回复**：这些关键词**不是单一来源**——既不是纯预录入，也不是纯 LLM 现编。**生产里几乎都是三层叠加**：
>
> - **LLM 常识先验**
> - **组织专属字典**
> - **历史 incident 检索**
>
> 只靠第一层（LLM 常识先验）会有适用性问题，后两层补的是组织专属和历史教训。留言中讲："关键词"被截断，未完待续。

---

## 十、思考题

### 题 1：keyword 推导逻辑

> 观察一下你 Agent 的 keyword 推导逻辑，它是怎么把用户描述翻译成 grep 关键词的？描述越宽泛，越容易让它踩到**广扫关键词太宽**的坑。基于今天所学，你将如何调整你的描述？

**提示**：

- 先故意给 Agent 一个泛化任务（"看看代码哪里有问题"），统计它的**广扫阶段 token 消耗**
- 再给同一 Agent 一个精确任务（"看看 LoginController 的 timeout 处理"）
- 对比两次的 token 消耗和信息获取质量

### 题 2：代码库规模 × commit 频率选型

> 算一下你团队代码库的规模 + commit 频率。
>
> - 你会考虑 **Agentic Search、persistent indexing、还是混合**？
> - 如果是混合，Discovery 触发条件怎么设？
> - 某些任务用 grep（隐私敏感）、某些任务用 indexing（跨 repo），**怎么让 Agent 自己挑**？

### 题 3：Discovery + Memory 协同

> 设计一个 **Discovery + Memory 协同的场景**。
>
> Agent 这次解决了 bug，把 `final_files` + 关键证据沉淀到 **procedural memory**。下次类似任务进来时：
>
> - **怎么先查 memory？**
> - **memory 命中和未命中分别走什么路径？**

---

## 十一、下一讲预告：多模态融合

> 下一讲我们学习**感知模块的最后一个模式**。我们将会探讨**如何处理非文本的输入**：
>
> - 一张**架构图**
> - 一段**服务运行 5 小时的日志（500MB）**
> - 一份 **200 页的 PDF 合同**
>
> 这些东西怎么进 Agent？

---

## 附：本讲核心要点速查

| 关键词 | 含义 |
|--------|------|
| **Progressive Discovery** | 广扫→聚焦→深挖三阶段循环定位陌生信息空间中的关键信息 |
| **双轴定位** | 感知 × 循环（迭代式，非单次完成）|
| **Agentic Search** | grep + read + follow imports 现场探索，少维护一份影子仓库 |
| **Persistent Indexing** | Augment 路线，400K+ 文件 / 45s 增量更新 |
| **Forage / Focus / Deepen** | 广扫（30-50 候选）→ 聚焦（5-10 完整读）→ 深挖（追可疑链）|
| **信息觅食理论** | Pirolli & Card 1999，斑块 + 边际收益 |
| **`information_gain`** | 单位成本拿到多少信息（Forage 评估）|
| **`patch_quality`** | 信息斑块质量（Focus 评估）|
| **`marginal_value`** | 深挖边际收益（Deepen 评估）|
| **DiscoveryTrace** | 12 字段行车记录仪：phase/query/result_count/selected/next_phase 等 |
| **Satisficing** | Herbert Simon：有限时间内找足够好的解 |
| **Atomic tools** | grep / glob / read / scorer 分开，不封装 search_codebase() |
| **`max_cycles = 3`** | 循环上限，三轮找不到就交人 |
| **`budget_per_cycle = 20K`** | 单轮 token 预算 |
| **三大指标** | `cycles_to_success_p50` / `forage_to_focus_ratio (0.3-0.5)` / `zero_signal_rate (<5%)` |
| **三个隔离** | Sub-Agent / search worker 隔离噪声，不污染主上下文 |
| **四个卡点** | Forage 太宽 / Focus 挑错文件 / Deepen 追死胡同 / Discovery × RAG 撞车 |
| **74% → 78%** | Anthropic 2025 Q1 → 2026 Q1 多文件编辑会话占比 |