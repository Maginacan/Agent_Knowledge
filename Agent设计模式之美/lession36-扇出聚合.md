---
title: "33｜扇出聚合：让多路证据保留差异再汇成结论"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/1012847"
tags:
  - "#Agent/协作"
  - "#设计模式/扇出聚合"
  - "#协作/EvidenceCard"
  - "#协作/GatherReport"
  - "#协作/聚合五步"
  - "#协作/独立证据率"
audio: true
duration: "23:05"
publish_date: "2026-09-03"
course_progress: "33/43"
pattern_position: "协作模式组·第 3 个"
related:
  - "[[lession35-层级委派]]"
  - "[[lession34-协作模块导论]]"
  - "[[lession19-并行探索]]"
  - "[[lession28-技能包]]"
  - "[[lession33-自愈循环]]"
---

# 33｜扇出聚合：让多路证据保留差异再汇成结论

> **核心命题**：扇出扩大搜索面，聚合器负责**盘点缺席、对齐口径、保留冲突和追踪来源**，再把材料交给业务裁决。
>
> **一句话**：聚合不是把几段文字拼在一起，也不是看哪边票多。**好的聚合器不是尽快消灭差异，而是先判断哪些差异可以合并，哪些差异必须保留**。
>
> 真实聚合的判定方法——**并行返回 ≠ 可以直接比较**；**材料齐全 ≠ 结论为真**。

---

## 一、释题与开篇问题

上一讲把一页技术简报拆成三个不同责任：

- Atlas 研究协作拓扑。
- Birch 研究状态管理。
- Comet 研究跨 Agent 交接。

结果收回来，主编问："DeepSeek Harness 的子 Agent 是隔离的吗？"三个人的回答是：

- **Atlas**：会话上下文是隔离的。
- **Birch**：进程内 Worker 仍可能看到同一个工作区。
- **Comet**：工具列表可以受限。

这三句话聚焦在不同维度：会话边界 / 文件系统边界 / 工具权限边界。如果主编不保留这些维度，把三张资料卡统一压缩成"隔离"或"不隔离"，其中两张没有出现 `shared`，系统会按 **2 票对 1 票**给出结论：

> "DeepSeek Harness 的子 Agent 默认完全隔离。"

> **聚合阶段最容易出现的问题**：各种带限定条件的证据，在聚合过程中被压缩成片面的、不能区分其推论背景的选票。

---

## 二、模拟聚合失败的一个案例

配套实验：`collaboration/light_labs/editorial_gather_lab.py`
配套工作台：`collaboration/light_labs/web_app.py`

```bash
git clone https://github.com/huangjia2019/agent-design-patterns.git
cd agent-design-patterns
python3 collaboration/light_labs/editorial_gather_lab.py
```

第一次输出，聚合器采用最省事的办法——凡是没有明确写 `shared` 的答案，都算支持隔离：

```text
== flatten and vote ==
answer=isolated support=2/3
lost=workspace:shared
```

三张投票卡中确实有两张被归到了 `isolated` 一侧。问题在于，三位研究员并没有围绕同一个字段投票：

- **session**：彼此是否看到相同的会话上下文？
- **workspace**：彼此是否访问相同的工作目录和文件？
- **tools**：彼此是否拥有相同的工具集合？

把三个字段折叠成一个布尔值以后，**多数票失去了统计意义**。

第二次输出，聚合器不急着选择"隔离"或"不隔离"，先保留每张卡的维度和来源，按维度收回每一张证据卡：

```text
== evidence-aware gather ==
verdict=qualified admitted=true
session   boundary=isolated   source=dsh subagent docs
workspace boundary=shared     source=dsh spawn provider
tools     boundary=restricted source=dsh tool filter
risk=workspace_shared
```

最终的准确表述：

> 进程内 spawn Worker 拥有独立会话，工具面可以受到限制；**文件系统是否隔离，则取决于 provider、沙箱和运行策略**。

`session=isolated` 与 `workspace=shared` 并不冲突，描述的是两种不同边界。**前一个实验中聚合器的错误主要在于删掉了证据所属的维度**。

可以启动网页版工作台进行观察：

```bash
python3 collaboration/light_labs/web_app.py
```

浏览器打开 `http://127.0.0.1:8098`，左侧选择"33 扇出聚合"，点击"运行对照实验"。工作台顺序执行 `flatten_and_vote()` 与 `gather()`：

- 左侧：2/3 错在把三个不同问题当成同一张选票。
- 右侧：把 session、workspace 和 tools 三张卡按原维度收回，结论是 `qualified`，并把 `workspace_shared` 信息交给下游判断。

> **这个小实验揭示**：聚合过程中收回来，不等于拼起来。

---

## 三、扇出聚合的工程化定义

扇出聚合并不是大模型时代才出现的结构。

**MapReduce**（Dean & Ghemawat, 2004）：Map 阶段处理输入分片，Reduce 阶段按键合并中间结果，运行时负责切分、调度、通信和故障恢复。

今天许多 Agent 并行架构仍沿用这副骨架：

```text
输入分片 → 并行 Worker → 中间工件 → 聚合函数
```

Agent 系统比数值计算多了一层**语义问题**：两个整数可以直接求和，两份调查结果却可能讨论不同对象、不同时间、不同单位和不同来源。即使文本看起来相似，也未必能放进同一个 Reduce。

**MapReduce 留给 Agent 工程最重要的启发**：把**执行函数与收口函数分开设计**——Map 负责局部计算，Reduce 必须事先说明怎样对齐、怎样组合、怎样处理缺席和异常。

**Bagging**（Breiman, 1996）提供了另一条参考：多个基学习器在不同样本上训练，再通过平均或投票降低方差。它能发挥作用的前提是**基学习器之间存在差异**。

> 如果多个 Worker 使用同一模型、同一提示、同一资料和同一搜索顺序，它们很可能共享同一盲区。**输出数量增加了，独立证据没有增加**。三份同源判断，不会因为来自三个会话就自动成为三份独立证据。

### 双轴坐标

扇出聚合位于"**协作 × 并行**"坐标。一次完整运行有三段：

1. 把能够独立推进的工作扇给多位 Worker。
2. 每位 Worker 在相对隔离的上下文中形成小工件。
3. 聚合器按明确规则对齐、去重、保留冲突，再形成总结果。

### 工程化定义

> **扇出聚合**把一个具有真实并行面的任务分发给多个独立执行者，再由聚合器依据**覆盖、口径、冲突和来源**规则，把局部工件收成一份**可以继续裁决**的结果。

"可以继续裁决"几个字很重要——**聚合器不一定替业务做最终决定**。它首先要让上层看清：哪些证据到齐了、哪些相互冲突、哪些来源缺席、最后那句结论从哪几张卡推出来。

---

## 四、为什么值得用扇出聚合

### 三个收益

1. **缩短墙钟时间**。三个调查方向分别需要 12 / 8 / 10 分钟。串行 ≈ 30 分钟；并行后主体耗时接近最慢的一路，再加调度与聚合开销。对**网页检索、代码扫描、合同初筛、区域数据核验**等长等待、低依赖任务，时间收益非常明显。
2. **扩大证据面**。不同 Worker 可以访问不同来源、使用不同工具或从不同专业角度检查同一个对象——安全 Agent 查看漏洞库，代码 Agent 读取调用路径，运维 Agent 检查运行配置。三路合起来比单一路径更容易暴露盲区。**但证据面扩大 ≠ 简单增加票数**，聚合器需要同时保留结论和来源血缘。
3. **把局部失败显式化**。八位 Worker 分别处理八百份合同的八个批次，最终只回来七份，聚合器应报告**缺少哪一批**，而不是把七份高质量报告拼成一份看起来完整的总结。**类型化聚合**让缺席、超时、坏格式和冲突保留在报告中，系统才有机会选择等待、重派、降级或交给人工。

### 伴随成本

并行会增加 token、连接数、速率限制、调度、重试和聚合开销。

- 任务没有真实并行面时，扇出不会带来收益。
- 结果没有统一工件结构时，**聚合器会成为新的上下文瓶颈**。

### 区分三种相近模式

| 模式 | 关注点 | 检查什么 |
| --- | --- | --- |
| **层级委派** | 责任划分 | 主管检查三块责任是否全部完成 |
| **扇出聚合** | 多路结果收回 | 聚合器保证三张卡保持原来的维度，不把不同口径压成选票 |
| **并行探索** | 候选解法竞争 | 同一道修复题同时尝试三条补丁路径，依据测试选择继续推进 |

真实系统经常组合这些模式：Lead Agent 先按地区做层级委派 → 每个地区 Worker 对多个资料源进行扇出聚合 → 聚合过程遇到未决问题启动并行探索比较多种解释。

---

## 五、聚合以前如何比较结果

代码里让 Worker 交回 `EvidenceCard`：

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class EvidenceCard:
    """One worker's finding about one precisely named boundary."""
    worker_id: str
    dimension: Dimension
    boundary: Boundary
    claim: str
    source_id: str
    source_url: str
```

> 这里最容易忽略的是 `dimension`。没有它，"会话隔离"与"文件共享"看上去像两个打架的答案；**有了它，系统才知道两张卡谈的是不同边界**。

### 生产可比性还应包括

| 维度 | 要回答的问题 | 示例陷阱 |
| --- | --- | --- |
| 对象一致性 | 所有 Worker 是否在讨论同一个对象？ | "Claude Managed Agents" / "Claude Code subagent" / 自建 wrapper 名称接近但运行边界不同——对象应绑定产品、provider、部署方式或实体 ID |
| 时间一致性 | 证据是否来自允许比较的时间范围？ | 2026-08 文档 vs 2025 旧版实现，结论不同可能只是版本变化 |
| 单位和口径一致 | 比例、币种、统计周期和分母是否一致？ | 38,444 与 384.44 可能相差 100 倍，也可能只是分与元换算；10% failure rate 可按调用/任务不同 |
| 来源可追溯 | 每张卡能否回到具体文档、查询、代码位置或数据快照？ | "根据官方资料"不是稳定引用 |
| 置信语义一致 | 不同 Worker 返回的 0.8 是否表示同一种东西？ | 模型主观把握 / 分类器概率 / 规则命中率 / 校准后统计概率，字段名相同不代表数值语义相同 |

生产版证据卡通常会扩展为：

```python
@dataclass(frozen=True)
class EvidenceCard:
    worker_id: str
    target_ref: str
    target_version: str
    dimension: str
    value: str
    claim: str
    source_id: str
    source_version: str
    source_lineage: tuple[str, ...]
    observed_at: str
    confidence: float | None
    confidence_semantics: str | None
```

**聚合器只能处理已经获得身份和口径的数据**，没有这些上下文，聚合器会把几份互不相干的结果编成一段流畅解释。

---

## 六、聚合器依次做五件事

"聚合"听上去像一个动作，但工程上按顺序拆为五步：

### 第一步：盘点缺席

聚合开始时，先检查应该返回的工件是否全部出现。

- 本讲要求 session、workspace、tools 三个维度各有一张证据卡。缺少任一维度，报告就应进入 `insufficient`，而不是让主 Agent 依据常识补写一段。
- 合同 / 简历 / 数据核验中，缺席通常通过**分片 ID** 识别：

```python
expected_shards = {"A", "B", "C", "D"}
received_shards = {"A", "B", "D"}
missing_shards  = {"C"}
```

缺失分片必须进入结果状态、监控和重派流程。**成功结果的数量不能掩盖未处理输入**。

### 第二步：归一口径

结果到齐后，确认它们能否放进同一张表。

- `workspace` / `working directory` / `repo root` 可能指向同一对象，也可能分别指容器目录、进程工作目录和仓库根目录。**归一化不能只靠字符串相似度，而要依赖术语表、实体映射或业务主数据**。
- 薪酬场景：E0007 与 0007 是否同一员工，要查员工主数据；USD 100 与 SGD 100 不能直接相加；按月统计与按结算周期统计不能混在一起。

**先统一身份和单位，后面的去重与冲突判断才有意义**。

### 第三步：去重

不同 Worker 可能返回相同结论，需要区分两种情况：

- **独立来源支持同一结论**。
- **同一来源被多个 Worker 重复引用**。

三张卡引用同一篇官方文档，可以说明多位 Worker 独立找到了同一资料，**却不能直接当成三份相互独立的证据**。

去重至少要看：

1. 结论是否重复。
2. 来源是否重复。
3. 来源之间是否存在上游依赖。

> 两篇文章来自不同网站，也可能都在转述同一份公告。文本不同，不代表证据独立。向量相似度可辅助发现近似表述，但**是否源于同一主文档还需要其它证据进一步分析**。

### 第四步：处理冲突

聚合器发现不同值后，不应立刻投票。需要先判断冲突属于哪一种：

- 本讲中两张卡属于**不同维度限定**，不是需要消灭的冲突。
- 只有同时满足以下条件，才采用投票策略：
  1. 回答同一个命题。
  2. 讨论同一个对象和版本。
  3. 采用同一种取值语义。
  4. 各路判断具有足够独立性。
  5. 投票规则与业务风险相匹配。

### 第五步：检查接缝

单项结果全部通过后，还要检查它们合在一起是否产生新的问题：

- 每个部门的付款都没有超过部门预算，所有部门相加仍可能越过公司资金线。
- 每个微服务都满足局部延迟目标，完整调用链仍可能超出端到端 SLA。
- 每份合同单独没有高风险条款，多份合同组合后却可能形成集中度风险。

> 第 32 讲的**组合验收**先检查责任覆盖；本讲进一步检查**跨分片不变量**：
> - 总额是否越界？
> - 同一对象是否重复出现？
> - 不同工件是否引用了互不兼容的版本？
> - 局部结论组合后是否产生新的业务风险？
>
> **接缝问题通常不属于任何一个 Worker，它只能在聚合层被发现**。

---

## 七、关键代码：报告要把分歧留下来

当前实验中的 `gather()` 不直接返回一段自然语言总结，而是返回**类型化报告**：

```python
from typing import Literal

@dataclass(frozen=True)
class GatherReport:
    """A typed result that keeps coverage, conflicts, and attribution."""
    verdict: Literal["isolated", "qualified", "insufficient"]
    accepted: bool
    missing_dimensions: tuple[Dimension, ...]
    conflicting_dimensions: tuple[Dimension, ...]
    cards: tuple[EvidenceCard, ...]
```

各字段职责：

- `cards`：保留原始证据卡和来源，使聚合结论可以回溯。
- `missing_dimensions`：说明哪些要求的证据尚未返回。
- `conflicting_dimensions`：说明哪些维度内部仍有未解决冲突。
- `accepted`：表示这份聚合报告是否满足结构准入要求。
- `verdict`：表示当前证据支持哪一种业务表述。

轻量实现核心判断：

```python
missing = tuple(
    dimension for dimension, matches in by_dimension.items() if not matches
)
conflicts = tuple(
    dimension
    for dimension, matches in by_dimension.items()
    if len({card.boundary for card in matches}) > 1
)
sources_present = all(
    card.source_id and card.source_url.startswith("http") for card in cards
)
accepted = not missing and not conflicts and sources_present
```

这里必须分清两个概念：

- `accepted` 表示这份证据报告**结构完整、内部没有未处理冲突，可以交给上层 Agent 处理**。
- 它不等于"所有边界都隔离"。所以最终业务判断还保留 `qualified`。

> **"材料齐全"与"结论为真"是两种不同判断**。将二者放进同一个 `success` 字段，会让上层 Agent 很难区分到底通过了什么。**状态与结论分开**，是更严谨的做法。

---

## 八、2026 年的主流 Harness 如何实现扇出聚合模式

现代 Harness 能较方便地启动并行 Agent，但它们主要解决**运行问题**，不能替代**证据语义**。

### Anthropic 多 Agent 研究系统

- Lead Agent 创建多个 Subagent，并行探索不同方向，再把压缩结果送回主上下文。
- 适合 **breadth-first 查询**：问题需要同时搜索多个来源和方向，单一上下文容易被检索过程占满。
- Lead Agent 仍负责**分配范围、判断结果是否重叠、识别缺口**并形成最终答案。
- Subagent 数量扩大的是搜索容量，**不能替代聚合规则**。

### OpenAI Agents SDK

- 编排指南展示了用 `asyncio.gather` 并行运行多个 Agent，`output_type` 可为每路结果指定结构。
- 运行时帮助同时发起调用、等待多路返回、收集成功与异常、获得结构化输出。
- **不会自动知道**：哪些分片应该到齐、哪些来源属于同一血缘、两个不同结论是真冲突还是维度不同。
- **这些规则仍属于业务聚合器**。`asyncio.gather` 能把结果放进同一个列表，但不能替业务定义这个列表怎样变成可信结论——这些业务规则就是需要具体实现的部分。

### DeepSeek Harness

- 本讲使用的版本通过 `ctx.subagents` 提供统一子 Agent 接缝，可以挂接 spawn、fork、ACP、Codex、Claude Code 与 dsh SDK 等 provider。
- 对扇出聚合而言：不同 Worker 可采用不同执行方式，而上层继续要求它们返回**同一种 `EvidenceCard`**。
- 生产实现可以由父编排器并发启动，或启用后台运行，再把返回工件送入确定性聚合器：

```text
provider    →  决定任务怎样执行
EvidenceCard →  规定局部结果怎样表达
GatherReport →  规定多路结果怎样收回
```

> **Harness 管并发、会话与生命周期，业务代码则负责覆盖、口径、冲突、溯源和接缝约束**。Harness 负责运行时实现，我们负责补全业务语义——这个基本思路适合所有模式实现。

---

## 九、上线之后的观测指标

生产系统对扇出聚合过程的选择性监控指标：

| 指标 | 含义 |
| --- | --- |
| **分片覆盖率** | 预期分片中有多少返回了合格工件？缺失分片是否都被披露？ |
| **独立证据率** | 去除同源重复后，真正独立的来源还剩多少？ |
| **冲突保留率** | 已知冲突有多少被保留并送入裁决，多少被错误扁平化？ |
| **增量收益** | 新增一个 Worker 后带来的新覆盖或独立证据，是否抵得上额外的调用与聚合成本？ |

> 四项指标中，**独立证据率最容易被高估**——三位 Worker 返回三张卡不代表三份证据。聚合器应如实记录去掉同源转述后的来源数量。超时、拒收率和收口耗时则作为运行指标持续观察。

**聚合器本身也需要版本和回归测试**：词表变化、身份映射错误或冲突规则更新都可能改变最终结论。因此，**Worker 和聚合器都应该被监控**。

---

## 十、什么时候适合使用扇出聚合模式

### 适合

- **输入能天然分片**的任务：按合同、客户、地区、代码目录或时间窗口拆分处理。
- **需要多源核验的调查**：同一结论需要官方文档、代码实现、运行配置和第三方资料共同支撑。
- **业务窗口很短、等待成本高于调用成本**的场景：事故排查、风险扫描、临时决策支持，墙钟时间比 token 成本更重要。

### 前置条件

- 每路输出能够压成**结构化工件**。
- 聚合规则能够说清楚。

### 不适合直接扇出

- 后一项必须读取前一项的完整中间过程。
- 多位 Worker 需要同时修改同一份状态，却没有隔离工作区和合并机制。
- 各路结果没有共同对象、单位和时间口径。
- 最终只能把几段长文本交给一个更大的模型"凭感觉总结"。
- 单 Agent 基线已经足够快，并行成本无法换来业务收益。

### 规模上限

Worker 数量不是越多越好。分片过细后，调度、超时、重试、去重和聚合的固定成本迅速增加。聚合器最终仍要消费所有局部工件，**它可能成为新的上下文瓶颈**。

规模较大时采用**分层聚合**：

```text
局部 Worker
  ↓
分组聚合器
  ↓
区域或主题报告
  ↓
上层综合
```

> 每一层都需要保留覆盖、冲突和来源，**不应该在中间层过早抹平**。

---

## 十一、总结

扇出解决多路工作怎样同时推进，聚合解决这些结果怎样被收成一份不掩盖证据状态的报告。

真正的聚合不是把几段文字拼在一起，也不是看哪一边票数更多。它先检查**分片是否到齐**，再统一**对象、时间和单位**，随后识别**同源重复**，区分**不同类型的冲突**，最后检查**局部结果组合后是否违反全局约束**。

本讲实验中，三位研究员分别回答会话、工作区和工具边界：

- **扁平化投票**把三种维度压成一个布尔值，得到"完全隔离"的错误标题。
- **类型化聚合**保留 `session=isolated`、`workspace=shared` 和 `tools=restricted`，最终形成 `qualified` 结论，并把共享工作区风险一并交给下游。

> 本讲最值得记住的：**能够并行返回的结果，不一定能够直接比较**。

---

## 十二、思考题

1. 你的多路结果真的在回答同一个字段，还是只因为都使用了"风险""通过""隔离"这些相似词，就被放进了同一轮投票？
2. 聚合报告能否明确列出**预期分片、已返回分片、缺席分片、未决冲突和每条结论的来源**？
3. 三位 Worker 引用同一篇资料时，系统记录的是三票支持，还是**一份证据被三次发现**？
4. `accepted=true` 在你的系统里表示**材料齐全、结论正确，还是业务已经获准执行**？这几种状态是否被错误地压成了同一个成功字段？
5. 增加一位 Worker 以后，它带来的新覆盖或独立证据，是否足以抵消额外的调用、去重和聚合成本？

---

## 十三、下一讲预告

下一讲进入**对抗评审**。技术简报已经聚合完成，独立评审员却发现标题仍然说得过满。它提交了一条有来源的异议，发布流程依然继续向前。**那么，什么机制能挡住发布按钮**？我们下一讲见。

---

## 十四、参考资料

1. Dean & Ghemawat, *MapReduce: Simplified Data Processing on Large Clusters*, 2004.
2. Breiman, *Bagging Predictors*, 1996.
3. Anthropic, *How we built our multi-agent research system*, 2025-06-13.
4. OpenAI Agents SDK: Agent orchestration，并行 Agent 与代码编排。
6. ADPS, *C2 Fan-out / Gather*，聚合五环节与适用边界。
7. DeepSeek Harness: Subagent subsystem，provider 与子代理生命周期。
8. DeepSeek Harness README，Developer Preview 与插件架构。