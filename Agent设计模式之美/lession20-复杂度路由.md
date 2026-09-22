---
title: "18｜复杂度路由：一件事该消耗多少推理"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/995335"
tags:
  - "#Agent/推理"
  - "#设计模式/复杂度路由"
  - "#推理/路由"
  - "#推理/RouteDecision"
  - "#推理/TalkerReasoner"
audio: true
duration: "23:12"
publish_date: "2026-07-09"
course_progress: "18/43"
pattern_position: "推理模式组·第 3 个"
related:
  - "[[lession18-推理模块导论]]"
  - "[[lession19-思维链]]"
  - "[[lession21-并行探索]]"
  - "[[lession22-迭代假设验证]]"
  - "[[lession9-上下文分诊]]"
---

# 18｜复杂度路由：一件事该消耗多少推理

> **核心命题**：复杂度路由是在任务进主链路之前，**用一次轻量判断**决定它该走哪条道、花多少推理、要不要人审。它是整个推理模块的前台。
>
> **先分诊，再看病。会推理的 Agent，先会分诊。**

---

## 一、引子：判断本身也要花成本

> 上一讲我们把 CoT 拆成了一条能对账的推理轨迹。但是**判断一件事该想多深，这个判断本身也要花成本**。如果每个请求都先认认真真分析一遍"你到底难不难"，那分析本身就成了新的浪费。

急诊室门口有分诊台。病人一进门，分诊护士先按病情轻重分流：

- 擦破皮的 → 普通门诊
- 胸痛的 → 直接进抢救室

护士的第一眼的判断**必须足够快、足够便宜**，不能说为了做个分诊先做一遍全面体检。

---

## 二、一句话里混着四类任务

> 帮我处理上海市场部 6 月薪资快照里的异常；能自动通过的直接过掉，不能过的列给我。

这句话表面只是一个请求，实际上是**一堆各种各样的任务**，至少混了四类判断：

| 类别 | 示例 | 复杂度 | 路由 |
|------|------|--------|------|
| **事实查询** | "这个月几号发薪" | 低风险、低复杂度 | 直接答 |
| **结构化判断** | "咖哥 6 月应发比 5 月高 18%，该不该自动放行" | 证据齐、规则多、主路径清楚 | [[lession19-思维链\|CoT]] |
| **多口径比较** | "小雪同时命中两版奖金政策，该按哪版算" | 几个口径都说得通 | [[lession21-并行探索\|并行探索]] |
| **迭代调查** | "月底总账差了 37 万，缺口从哪来" | 信息不足 | [[lession22-迭代假设验证\|迭代假设验证]] |

> **整句话如果直接丢给一个大模型自由发挥，风险很高**。这句话**不该直接得到一个答案**。它应该先被拆成任务原子，然后每个任务原子各自生成一份路由结果——**`RouteDecision`**。

```yaml
intent: resolve
domain: payroll_exception
evidence_state: missing       # 还没召回完整审批与考勤
mechanical_ready: false       # 关键机械状态待补
risk: write_sensitive
lane: 人审澄清                # 整句先停在澄清，拆开后各子任务再分流
reasoning_effort: high
human_gate: before_write
```

> 这一小段结构化信号，就决定了后面整个系统怎么走。**它不是给人看的注释，是驱动下游每一步的控制信号。**

---

## 三、复杂度路由模式的定义

### 3.1 在双轴图谱中的位置

在 [[lession4-双轴框架（下）\|双轴图谱]] 里，复杂度路由落在 **"推理 × 路由"** 的交点：

- **属于推理**：因为它决定一次任务要不要启动深度思考、采用哪种推理拓扑、投入多少预算
- **属于路由**：因为它不亲自解决问题，而是把问题送到合适的工作流、模型、推理模式和人审节点

### 3.2 有限理性的工程哲学

> **有限理性**告诉我们：现实中的决策者没有无限时间和无限算力，不会穷尽所有选项再找全局最优，而是在成本可接受的范围内找到一个足够好的方案。

Agent 也一样。**推理本身有成本**，所以系统在继续推理之前，要先多想一步——收益能不能抵过时间、token、工具调用和风险成本。

> 这听起来像套娃："先推理一下：这件事值不值得推理"。工程上也确实是套娃，所以**分诊台必须便宜**。为了节省推理成本而引入的路由器，不能自己变成新的成本黑洞。

### 3.3 思想谱系

| 来源 | 思路 |
|------|------|
| **级联分类器** | Viola-Jones (2001)：先用便宜特征筛掉简单样本，难样本才进入更贵的后续模型 |
| **MoE** | Adaptive Mixtures of Local Experts (1991)：用门控网络把输入分给不同专家 |
| **Enterprise Integration Patterns** | Content-Based Router：按消息内容把请求发到不同通道 |
| **FrugalGPT** | 把级联思想搬到大模型调用上：prompt adaptation / LLM approximation / LLM cascade |
| **RouteLLM** | 训练路由器，在强弱模型之间动态选择，优化成本与质量平衡 |

### 3.4 工程版定义

> **复杂度路由**：在执行前和执行中，对一个请求进行轻量级的结构化判断，决定它该走哪条工作流、用哪种推理拓扑、给多少预算、在哪一步插人审，输出**可执行、可审计、可版本化**的 `RouteDecision`。

几个关键词解释：

- **"执行前和执行中"**：路由不是一次性动作。初始路由可能判断错，证据回来以后也可能改变。一个请求可以从 `Direct` 升级到 `CoT`，从 `CoT` 升级到 `Parallel`，也可以因为证据缺失而转人审
- **"可版本化"**：路由策略必须能追责
- **"工作流"和"推理拓扑"必须分开**：
  - 工作流决定有没有写权限、要不要证据、在哪插人审
  - 推理拓扑决定是直接答、链式推、并行比，还是循环查
  - 分别对应两个字段：`lane` 和 `reasoning_mode`

### 3.5 与推理契约的关系

> 推理模块导论的[[lession18-推理模块导论\|推理契约]]中有五问：是否启动慎思 / 哪种推理拓扑 / 投入多少预算 / 由什么验证 / 什么时候停。
>
> **复杂度路由回答"是否慎思、哪种拓扑、多少预算"**，它本身不解题，它**决定这道题该用多大力气解、由哪条流水线解**。
>
> 因此复杂度路由的判断属于"外部慎思过程"里的**元数据层（meta-reasoning）**。它的产物 `RouteDecision`，落进控制平面的决策审计记录部分。

### 3.6 与意图识别的关键区别

> 复杂度路由听起来像意图识别。的确，它做的事情也包含意图识别，但**远远比意图识别覆盖面广泛得多**。

| 老式意图识别 | 复杂度路由（`RouteDecision`）|
|------------|---------------------------|
| 一个标签："直接回答"/"查数据库"/"转人工" | 一份**可执行、可审计、可追责**的控制契约 |
| 分完就完了 | 驱动后续整个系统怎么跑 |
| 不解释 | 包含 `route_reason` + `blockers` + `escalated_from` |

> **普通分类器只分类，分完就完了。`RouteDecision` 这个数据结构则要驱动后续整个系统怎么跑。**

---

## 四、先把模糊请求拆成任务原子

复杂度路由的第一步，不是判断"这句话难不难"，而是**先把它拆成几个任务原子**。

### 4.1 任务原子（TaskAtom）

任务原子就是一个**可以被单独路由的最小任务单位**，最好只包含三件事：

- 它要处理什么**对象**
- 它要做什么**动作**
- 它依赖哪些**上游结果**

### 4.2 案例：4 个任务原子

```text
帮我处理上海市场部 6 月薪资快照里的异常；
能自动通过的直接过掉，不能过的列给我。
```

被拆成 4 个原子：

| ID | 任务 | 依赖 |
|----|------|------|
| **A1** | 列出上海市场部 6 月薪资快照里的异常 | — |
| **A2** | 判断哪些异常可以自动通过 | A1 |
| **A3** | 把 A2 判断为可自动通过的异常执行通过 | A2 |
| **A4** | 列出 A2 判断为不能自动通过的异常 | A2 |

> **A3 不能直接执行**。它依赖 A2 的结果。在 A2 做完之前，系统还不知道"哪些异常可以自动通过"。所以 A3 不是马上写库，而是进入一个**带前置条件的执行流程**。

> **关键设计**：不要让一整个模糊请求直接进入推理。**先拆成任务原子，每个原子单独分诊。**

---

## 五、四个路由信号：意图 / 证据 / 执行 / 风险

任务原子是下一步路由信号提取的起点。**系统到底怎么判断一件事难不难**？

> 在执行型 Agent 里，**难度和风险不是一回事**。
>
> - "把小冰社保基数改回上月"：这句话很短，推理也不深，但它要改敏感数据
> - "帮我统计全公司 5000 人过去 12 个月的薪资异常分布"：这句话很长，数据量很大，也可能需要不少工具调用，但它**只读**，不直接改业务状态
>
> 所以**复杂度路由不能只问"这题难不难"**，而要先拆成四个简单的信号。

### 5.1 第一个信号：任务意图（intent）

任务意图先回答：**用户到底要系统做什么**？

| 枚举值 | 含义 |
|--------|------|
| `chat` | 闲聊 |
| `information` | 查事实 |
| `analyze` | 做分析 |
| `draft` | 生成草稿 |
| `resolve` | 完成一个业务动作 |
| `unknown` | 判不准 |

> 这个枚举一开始不要做得太细。上来就做几十个意图分类就太乱了。**先粗粒度分一级，再在具体业务域里分二级**。

例：

- "小冰这个月社保基数怎么变了？" → `analyze`：要查规则、查记录、解释原因，但**不改数据**
- "把小冰社保基数改回上个月。" → `resolve`：要改变业务状态，而且是敏感数据

### 5.2 第二个信号：证据状态（evidence_state）

证据状态回答：**做这个判断所需的证据够不够**？

| 枚举值 | 含义 |
|--------|------|
| `ready` | 证据已召回，版本匹配 |
| `missing` | 关键证据缺失 |
| `stale` | 证据过期或版本不对 |
| `conflict` | 多个证据互相冲突 |
| `unknown` | 还没完成检索或核验 |

> **证据缺口不能用更大的模型弥补**。缺审批单，就去审批系统查；缺考勤记录，就去考勤系统查；政策版本冲突，就去政策注册中心核对。
>
> 只要 `evidence_state` 是 `missing` / `conflict` / `unknown`，任务就**不能直接进入自动决策，更不能进入自动写入**。

### 5.3 第三个信号：执行对象状态（mechanical_ready）

执行对象状态回答：**这次动作要作用到哪个真实对象上？对象有没有被程序确认**？

员工 ID、薪资批次 ID、审批单号、租户 ID、账号、税号——这些**都不应该让 LLM 在自然语言上下文里复制来复制去**。它们应该由程序维护，带数据来源、版本和校验。

| 检查项 | 确认方式 |
|--------|---------|
| `employee_id` | 是否来自员工主数据？ |
| `payroll_batch_id` | 是否来自当前批次？ |
| `approval_id` | 是否确认为同一租户下的审批单？ |
| `policy_version` | 是否覆盖当前结算月份？ |

> **只要关键执行对象状态不完整，就一票否决，不能进入写操作**。
>
> Agent 并不适合处理和传递数字类型的信息，Agent 需要确定信息的完整性，然后把数据对象交给**机械平面**中的工作流来执行。

### 5.4 第四个信号：动作风险（risk）

动作风险必须独立评估，按动作后果衡量：

| 风险等级 | 含义 |
|---------|------|
| `read_only` | 只读 |
| `draft` | 生成草稿，不自动提交 |
| `write_reversible` | 可回滚写入 |
| `write_sensitive` | 敏感写入，如薪资、权限、合同 |
| `external_irreversible` | 不可逆外部动作，如发薪、报税、支付 |

---

## 六、一个结构化结果：RouteDecision

四个信号提取完以后，路由器会做一次路由选择。`RouteDecision` 是这次决策动作的结构化结果。

```text
四个路由信号
    ↓
路由选择 routing
    ↓
RouteDecision
    ↓
lane / reasoning_mode / effort / budget / human_gate / model_policy
```

### 6.1 RouteDecision 实例（高风险执行型任务）

```python
RouteDecision(
    router_version="router_2026_06_01",
    policy_version="payroll_policy_2026_05",
    intent="resolve",
    evidence_state="ready",
    mechanical_ready=True,
    risk="write_sensitive",
    # lane 管业务边界：这件事属于计划执行类任务
    lane="plan_execute",
    # workflow_id 管具体流程：这次跑薪资异常放行流程
    workflow_id="payroll_exception_release_v3",
    # reasoning_mode 管推理形状：沿一条链推理
    reasoning_mode="cot",
    # reasoning_effort 管推理预算：中等强度
    reasoning_effort="medium",
    # human_gate 管责任边界：写入前必须人审
    human_gate="before_write",
    # model_policy 管模型策略：允许用推理模型
    model_policy="reasoning_model_allowed",
)
```

> 这就是一次高风险执行型任务的 `RouteDecision`：证据已齐、对象已确认、任务意图是完成业务动作，但因为风险是敏感写入，所以进入 `plan_execute` 车道，用 CoT 做中等强度推理，并且写入前必须人审。

### 6.2 完整 Python 数据结构

```python
from dataclasses import dataclass, field
from enum import Enum


class Intent(str, Enum):
    INFORMATION = "information"   # 查事实
    ANALYZE = "analyze"           # 只读分析
    DECIDE = "decide"             # 产出判断，但不写库
    EXECUTE = "execute"           # 执行业务动作
    UNKNOWN = "unknown"


class EvidenceState(str, Enum):
    READY = "ready"
    MISSING = "missing"
    CONFLICT = "conflict"
    UNKNOWN = "unknown"


class Risk(str, Enum):
    READ_ONLY = "read_only"
    DRAFT = "draft"
    WRITE_REVERSIBLE = "write_reversible"
    WRITE_SENSITIVE = "write_sensitive"
    EXTERNAL_IRREVERSIBLE = "external_irreversible"


class Lane(str, Enum):
    DIRECT_ANSWER = "direct_answer"
    READ_ONLY_ANALYSIS = "read_only_analysis"
    STRUCTURED_DECISION = "structured_decision"
    PLAN_EXECUTE = "plan_execute"
    CLARIFY_OR_REVIEW = "clarify_or_review"


class ReasoningMode(str, Enum):
    DIRECT = "direct"
    COT = "cot"
    PARALLEL = "parallel"
    ITERATIVE = "iterative"


class HumanGate(str, Enum):
    NONE = "none"
    BEFORE_WRITE = "before_write"
    ALWAYS = "always"


@dataclass
class TaskAtom:
    atom_id: str
    text: str
    depends_on: list[str] = field(default_factory=list)


@dataclass
class RouteSignals:
    intent: Intent
    evidence_state: EvidenceState
    target_ready: bool
    risk: Risk
    confidence: float = 1.0


@dataclass
class RouteDecision:
    router_version: str             # 路由策略版本，出事能追是哪版策略判错
    intent: str                     # chat / information / analyze / resolve / unknown
    evidence_state: str             # ready / missing / stale / conflict
    mechanical_ready: bool          # 关键机械状态齐不齐（接进度追踪的机械平面）
    risk: str                       # read_only / draft / write_reversible / write_sensitive / external_irreversible
    lane: str                       # 五条车道之一：决定下游工作流
    reasoning_mode: str             # DIRECT / COT / PARALLEL / ITERATIVE：决定推理拓扑，对应导论 ReasoningTrace 四出口
    reasoning_effort: str           # none / low / medium / high / xhigh
    human_gate: str | None = None   # 在哪一步插人审
    confidence: float = 0.0         # 路由本身的置信度，低了就降级到人审澄清
    escalated_from: str | None = None  # 从哪个更轻的 mode 升级而来，分诊时留空待回填
    signals: dict = field(default_factory=dict)

    def can_write(self) -> bool:
        # 要动写操作，机械状态必须齐、且落在计划执行车道
        return self.mechanical_ready and self.lane == "plan_execute"

    def needs_clarify(self) -> bool:
        # 路由器自己不确定，就降级人审澄清，不硬选一条道
        return self.intent == "unknown" or self.confidence < 0.5
```

---

## 七、五条路由车道：Lane 负责流程

`lane` 字段回答：**这次请求应该进入哪条业务流水线**？它负责的是工作流、权限和边界，不管具体怎么推理。

| 车道 | 适用场景 | 写权限 | 证据要求 | 人审要求 |
|------|---------|--------|---------|---------|
| `direct_answer` | 简单事实、低风险问答 | 无 | 可选 | 无 |
| `read_only_analysis` | 解释原因、统计、查询、只读分析 | 无 | 必须引用证据 | 通常无 |
| `structured_reasoning` | 多证据、多规则，只产判断 | 无 | 必须绑定证据与验证器 | 高风险时需要 |
| `plan_execute` | 计划并执行业务动作 | 有，但受控 | 必须 `ready` | 写前或外部动作前必需 |
| `clarify_human_review` | 意图不明、证据冲突、对象不清、风险顶格 | 无 | 先补证 | 必需 |

> 这张表比"简单 / 中等 / 复杂"三分法实用，因为它**分流的是路径，不只是模型**。

**车道定义要点**：

- `direct_answer` 没有写权限
- `read_only_analysis` 必须引用证据，但不动业务状态
- `structured_reasoning` 只产判断，不执行写入
- `plan_execute` 才进入行动组，但写前必须过验证闸门和人审
- `clarify_human_review` 是兜底：意图不明、证据冲突、对象不清、风险顶格，就**停下来补证或交给人**

### 7.1 案例：四类子任务分到不同车道

薪酬请求"帮我处理上海市场部 6 月薪资快照里的异常"：

| 子任务 | 车道 |
|--------|------|
| "有哪些异常" | `read_only_analysis`（只读分析，引用证据，不写库） |
| "哪些能自动通过" | `structured_reasoning`（绑规则、绑证据，只产判断） |
| "直接过掉" | `plan_execute`（涉及写入，写前必须过验证闸门和人审） |
| "员工 ID 错位 / 审批单缺失 / 政策版本冲突" | `clarify_human_review`（暂停，补证或人工复核） |

> **分诊台先把它切成几个任务原子，再让它们各走各的车道。**

---

## 八、几种思考形状：ReasoningMode 负责拓扑

> `RouteDecision` 还有一个重要字段 `reasoning_mode`。**lane 管流程，reasoning_mode 管思考形状**。lane 和 reasoning_mode **不是一一对应关系**。一条车道里可以跑不同的 reasoning_mode。同一个 reasoning_mode 也可能出现在不同车道里。

| `ReasoningMode` | 适用问题 | 对应模式 |
|----------------|---------|---------|
| `DIRECT` | 简单、低风险、一步可答 | 直接思考 |
| `COT` | 主路径明确，中间步骤不能省 | [[lession19-思维链\|思维链]] |
| `PARALLEL` | 多个合理口径，需要并行比较 | [[lession21-并行探索\|并行探索]] |
| `ITERATIVE` | 信息不足，需要边查边修 | [[lession22-迭代假设验证\|迭代假设验证]] |

### 8.1 案例：同一车道不同推理拓扑

- 小雪同时命中两版奖金政策时 → 业务车道是 `structured_reasoning`（证据齐、只产决策、不写库）；但 `reasoning_mode` 是 `PARALLEL`（要并行算几个口径再择优）
- 月底总账差 37 万时 → 可能是只读分析或结构化推理车道；但 `reasoning_mode` 是 `ITERATIVE`（要边查边修假设）

| 例子 | `reasoning_mode` |
|------|-----------------|
| 这个月几号发薪 | `DIRECT` |
| 咖哥 18% 加薪该不该放行 | `COT` |
| 小雪命中双版政策 | `PARALLEL` |
| 总账 37 万缺口 | `ITERATIVE` |

> **Lane 与 ReasoningMode 是两个维度。**

---

## 九、路由规则的极简代码实现

### 9.1 选择车道

```python
def choose_lane(signals: RouteSignals) -> Lane:
    if signals.intent is Intent.INFORMATION:
        return Lane.DIRECT_ANSWER
    if signals.intent is Intent.ANALYZE:
        return Lane.READ_ONLY_ANALYSIS
    if signals.intent is Intent.DECIDE:
        return Lane.STRUCTURED_DECISION
    if signals.intent is Intent.EXECUTE:
        return Lane.PLAN_EXECUTE
    return Lane.CLARIFY_OR_REVIEW
```

### 9.2 选择推理拓扑

```python
def choose_reasoning_mode(signals: RouteSignals, lane: Lane) -> ReasoningMode:
    if lane is Lane.DIRECT_ANSWER:
        return ReasoningMode.DIRECT
    if signals.evidence_state is EvidenceState.CONFLICT:
        return ReasoningMode.PARALLEL
    if signals.evidence_state in {EvidenceState.MISSING, EvidenceState.UNKNOWN}:
        return ReasoningMode.ITERATIVE
    if lane in {Lane.STRUCTURED_DECISION, Lane.READ_ONLY_ANALYSIS}:
        return ReasoningMode.COT
    if lane is Lane.PLAN_EXECUTE:
        return ReasoningMode.DIRECT
    return ReasoningMode.DIRECT
```

### 9.3 选择人审节点

```python
def choose_human_gate(signals: RouteSignals) -> HumanGate:
    if signals.risk is Risk.EXTERNAL_IRREVERSIBLE:
        return HumanGate.ALWAYS
    if signals.risk is Risk.WRITE_SENSITIVE:
        return HumanGate.BEFORE_WRITE
    return HumanGate.NONE
```

### 9.4 生成 RouteDecision

```python
def route(atom: TaskAtom, signals: RouteSignals) -> RouteDecision:
    lane = choose_lane(signals)
    reasoning_mode = choose_reasoning_mode(signals, lane)
    human_gate = choose_human_gate(signals)
    blockers: list[str] = []
    if signals.intent is Intent.UNKNOWN:
        blockers.append("intent_unknown")
    if signals.evidence_state is EvidenceState.MISSING:
        blockers.append("evidence_missing")
    if signals.evidence_state is EvidenceState.CONFLICT:
        blockers.append("evidence_conflict")
    if signals.intent is Intent.EXECUTE and not signals.target_ready:
        blockers.append("target_not_ready")
    if signals.confidence < 0.6:
        blockers.append("low_route_confidence")
    route_reason = (
        f"intent={signals.intent.value}, "
        f"evidence_state={signals.evidence_state.value}, "
        f"target_ready={signals.target_ready}, "
        f"risk={signals.risk.value}"
    )
    return RouteDecision(
        atom_id=atom.atom_id,
        signals=signals,
        lane=lane,
        reasoning_mode=reasoning_mode,
        human_gate=human_gate,
        blockers=blockers,
        route_reason=route_reason,
    )
```

---

## 十、案例回放：薪酬请求的四条 RouteDecision

### A1：列出异常

```python
signals_a1 = RouteSignals(
    intent=Intent.ANALYZE,
    evidence_state=EvidenceState.READY,
    target_ready=True,
    risk=Risk.READ_ONLY,
    confidence=0.9,
)
decision_a1 = route(atoms[0], signals_a1)
```

| 字段 | 值 |
|------|-----|
| `lane` | `read_only_analysis` |
| `reasoning_mode` | `cot` |
| `human_gate` | `none` |
| `blockers` | `[]` |

→ 进入**只读分析流程**。

### A2：判断哪些可自动通过

```python
signals_a2 = RouteSignals(
    intent=Intent.DECIDE,
    evidence_state=EvidenceState.READY,
    target_ready=True,
    risk=Risk.READ_ONLY,
    confidence=0.85,
)
```

| 字段 | 值 |
|------|-----|
| `lane` | `structured_decision` |
| `reasoning_mode` | `cot` |
| `human_gate` | `none` |
| `blockers` | `[]` |

→ 进入**结构化决策流程**，也就是 [[lession19-思维链\|CoT Trace]]：一步步拆出 claim、evidence、validator，最后产出"可自动通过清单"和"不可自动通过清单"。

### A3：执行通过（敏感写操作）

```python
signals_a3 = RouteSignals(
    intent=Intent.EXECUTE,
    evidence_state=EvidenceState.READY,
    target_ready=False,           # ⚠️ 关键：还不知道哪些可通过
    risk=Risk.WRITE_SENSITIVE,
    confidence=0.8,
)
```

| 字段 | 值 |
|------|-----|
| `lane` | `plan_execute` |
| `reasoning_mode` | `direct` |
| `human_gate` | `before_write` |
| `blockers` | `["target_not_ready"]` |

> **注意**：A3 确实应该进入 `plan_execute`，因为用户要求"直接过掉"。但它现在**还不能写库**，因为 `target_ready=False`。它必须等 A2 产出明确的、验证通过的目标清单后，才能**解除 blocker**。即使 blocker 解除，因为风险是 `write_sensitive`，也仍然要在写前经过 `human_gate=before_write`。
>
> **这就是路由的意义所在**——系统承认这是执行任务，但并**不会让它立刻执行**。需要等待后续步骤统一完成。

### A4：列出不能通过的异常

```python
signals_a4 = RouteSignals(
    intent=Intent.ANALYZE,
    evidence_state=EvidenceState.READY,
    target_ready=True,
    risk=Risk.READ_ONLY,
    confidence=0.9,
)
```

| 字段 | 值 |
|------|-----|
| `lane` | `read_only_analysis` |
| `reasoning_mode` | `cot` |
| `human_gate` | `none` |
| `blockers` | `[]` |

→ 进入**只读分析流程**，输出"不能自动通过的异常清单"和原因。

### 整体执行图

```text
用户请求
   ↓
任务原子拆分
   ↓
每个原子提取四个信号
   ↓
四个信号生成 RouteDecision
   ↓
RouteDecision 决定 lane / reasoning_mode / human_gate / blockers
   ↓
不同原子进入不同流程
```

---

## 十一、三层路由复杂度选择：workflow / model / effort

上面聚焦的是**工作流路由（workflow routing）**，决定直接回答 / 只读分析 / 结构化推理 / 规划与执行 / 还是人工审核。**这一层永远在模型外面**。因为它处理的是业务权限、证据缺口、写入边界、人审责任。模型不能自己决定"我可以改薪资"。

### 11.1 第二层：模型路由（model routing）

决定便宜模型、强模型、专用模型、推理模型之间怎么选。[[lession18-推理模块导论\|FrugalGPT 和 RouteLLM]] 主要在这一层。

| 系统 | 思路 |
|------|------|
| **FrugalGPT** | 讨论 LLM cascade，用不同模型组合降低成本 |
| **RouteLLM** | 学习在强弱模型之间路由，以平衡成本和质量 |

### 11.2 第三层：推理强度路由（effort routing）

> 这里要补充一个最近的关键变化。也就是**第三层推理强度路由（effort routing）**。它不是换模型，而是在同一个模型或同一模型族里调"想多深"。现在 Claude、GPT-5、Gemini 都把"想多深"做成了模型内部的一个 effort 档位，于是 effort 路由开始部分取代模型路由。

OpenAI 的推理模型文档也明确把**推理强度 reasoning effort**（以及 reasoning tokens 和跨轮推理状态 reasoning state）作为 API 里的工程概念，做成了可选参数。

### 11.3 三层路由对比

| 层级 | 关心什么 | 例子 |
|------|---------|------|
| **第一层：工作流路由** | 业务权限、证据、写入边界、人审 | `Lane` / `HumanGate` |
| **第二层：模型路由** | 选哪个模型 | FrugalGPT / RouteLLM |
| **第三层：推理强度路由** | 在同一个模型里想多深 | `reasoning_effort` |

---

## 十二、把路由推到极致：Talker-Reasoner 双模

前面讲的都是"**同一个 Agent 选不同档位**"。把这个思想推到极致，就是**干脆把档位拆成两个独立的 Agent 同时跑**——这就是 2024 年 Google DeepMind 在 NeurIPS 2024 Open-World Agents workshop 上提出的 **Talker-Reasoner 架构**。

### 12.1 架构

| 角色 | 负责 | 模型 |
|------|------|------|
| **Talker** | 嘴（System 1） | 快模型，维持连续低延迟对话 |
| **Reasoner** | 脑（System 2） | 慢模型，后台深想 |

### 12.2 工作流

```text
用户说话
   ↓
Talker 用快模型先接住、维持节奏
   ↓
遇到真需要深想的问题
   ↓
异步抛给 Reasoner
   ↓
Reasoner 在后台慢慢算、把结论写进共享信念库
   ↓
Talker 随时读最新的结论继续聊
```

### 12.3 适用场景

> 这种设计的好处在**实时交互场景**特别明显。你跟一个语音助手说话，它不能让你干等二十秒它"深度思考"。Talker 用快模型先把对话接住、维持节奏，遇到真需要深想的问题，异步抛给 Reasoner。

**一个管嘴、一个管脑，这其实是一种按时间维度做的路由**，把"现在就得回"和"可以慢慢想"这两类需求，分给了两个跑在不同节奏上的 Agent。

### 12.4 代价

不过它不是免费的：

- 两个 Agent、两套上下文、两套 trace、一套同步协议
- 工程复杂度明显上去了

> 我的判断是：**单 Agent 的后台批处理任务不值得拆，但语音助手、实时陪聊这类"边对话边深想"的场景几乎必须拆**。
>
> **Talker-Reasoner 是把路由推到极致的双模架构，它不是第五个推理模式，它是复杂度路由在"按时间分流"这个方向上的一种极端实现。**

---

## 十三、路由关键指标：首次路由命中率

> 导论中，我们给推理系统立了四个指标：**验证通过率、首次路由命中率、单位验证成功成本、推理漂移率**。这四个指标里，负责衡量复杂度路由的指标是**首次路由命中率**。

### 13.1 指标定义

**首次路由命中率**衡量的是**第一次选的推理模式是不是足够解决问题**。读这个指标，关键看两种失配方向：

| 失配类型 | 表现 | 系统问题 |
|---------|------|---------|
| **系统性低估难度** | 大量本以为是直接推理的请求，最后升级到迭代才搞定 | 分诊台把难题当简单题放进了快路。省下的那点钱，全赔在反复升级、重试、推翻重来上 |
| **系统性过度推理** | 大量低风险的查询，一上来就被塞进高成本并行探索或迭代验证 | 分诊台过于谨慎，把简单题当难题供着，每一笔都在烧没必要的算力 |

### 13.2 对症下药

- 前者 → **放松"判简单"的门槛**，或**加强升级前的预判**
- 后者 → **收紧"判复杂"的触发条件**

> 之所以能精准地对症下药，前提是你**把每次 `RouteDecision` 和它最终的实际归宿都记进了 trace**，才能算出"判了什么、最后实际走了什么"。这也是前面 `RouteDecision` 一定要进入审计记录的原因。

### 13.3 派生指标：单位验证成功成本

> **单位验证成功成本**衡量的是，拿到一个通过验证的正确结果，平均花了多少钱和多少延迟。因为升级、重试、推翻重来的成本全摊进了分母里。命中率下降，单位成本当然就涨。
>
> 因此**会算账的 Agent，先要把分诊台调准**。

---

## 十四、总结

> 这一讲，我们把"该花多少推理"做成了一座**工程化分诊台**。

### 14.1 复杂度路由不是分类器

**复杂度路由不是简单给 Agent 加一个分类器**。它的核心，是在任务进入主链路之前，先判断这件事该不该深想、该走哪条车道、用哪种推理拓扑、给多少预算、哪里必须人审。

### 14.2 它看四个信号

| 信号 | 问什么 |
|------|-------|
| **任务意图** | 用户是查事实、做分析，还是要完成动作？ |
| **证据状态** | 证据是否 ready，有没有缺失、过期或冲突？ |
| **机械状态** | 员工 ID、批次 ID、审批单号等执行对象是否确认？ |
| **动作风险** | 这是只读、草稿、可回滚写入，还是敏感 / 不可逆动作？ |

### 14.3 它输出 RouteDecision

复杂度路由的输出**不是一个"简单 / 中等 / 复杂"的标签**，而是一份 `RouteDecision`。它是一张**控制单**，告诉系统：

| 字段 | 含义 |
|------|------|
| `lane` | 走哪条业务车道 |
| `reasoning_mode` | 用哪种推理拓扑 |
| `reasoning_effort` / `budget` | 花多少推理 |
| `human_gate` | 哪里必须人审 |
| `blockers` | 当前有没有阻塞项 |

> **这几个字段不要混**：
> - `Lane` 管**业务边界**
> - `ReasoningMode` 管**思考形状**
> - `HumanGate` 管**风险刹车**
> - `Blockers` 管**当前能不能继续**

### 14.4 推理组总图

推理这一组可以理解成 **"一座分诊台加三间诊室"**：

- **复杂度路由** = 分诊台
- **[[lession19-思维链|CoT]]** = 对账诊室（一条链）
- **[[lession21-并行探索|并行探索]]** = 择优诊室（多路径）
- **[[lession22-迭代假设验证|迭代假设验证]]** = 逼近根因诊室（多轮）

> CoT 负责对账，并行探索负责择优，迭代假设验证负责逼近根因。

### 14.5 推理经济学总开关

> **复杂度路由也是整套推理经济学的总开关**。大部分请求其实不需要深想，应该走快路；深想要留给真正复杂、昂贵、高风险的那一小部分任务。盯紧它的本命指标：首次路由命中率。路由准了，成本、延迟、人审和重试都会一起下降。
>
> 最后记住一句话：**先分诊，再看病。会推理的 Agent，先会分诊。**

---

## 十五、思考题

1. 你的 Agent 现在按什么决定一个请求用多大的模型、想多深？是按 query 长度或工具数量，还是按任务意图、证据状态、机械状态、动作风险这四个信号？这四个里，哪个信号你现在完全没看？
2. 找一条你系统里"**很短但很危险**"的请求，比如一句话改个金额或权限。现在的路由会把它放进哪条车道？它的 `risk` 是独立估的，还是搭了复杂度的便车？写操作前有没有验证闸门和人审？
3. 你的路由决策是一段自然语言，还是一个**带版本号的结构化对象**？如果上周发生过一次误路由，你能追到是哪一版策略、按哪个信号判错的吗？再看一眼，你分得清 `lane` 和 `reasoning_mode` 这两个输出吗，你的系统里有没有把它们焊成了一对、结果在某个场景卡住？

---

## 十六、下一讲预告

下一讲我们讲第三个推理模式：**并行探索（Parallel Exploration）**。

复杂度路由把判断分了诊。当一件事真分到深水区、而且一条路想不稳，要不要同时想几条？**并行探索是深水区的广度诊室**。

---

## 精选留言摘录

> **天下霸唱**：
> 老师为什么不把这一节放在 cot 前面呢？
>
> **作者回复**：导论中说了：本模块中的四种推理模式对应四种不同的计算拓扑，我们的学习顺序是"基础、路由、广度、深度"。不过，**生产运行时要先路由**。Agent 应该先决定是否需要深想，再决定采用 CoT、并行探索还是迭代验证。CoT 是一般推理学习的基础。不过现在想想，从实现的视角，学习的顺序来说，**复杂度路由都可以先上**。是的！

> **charles**：
> 脑补不出来这个 reasoning_mode 为 COT、PARALLEL、ITERATIVE，lane 为 read_only_analysis、structured_reasoning 时，后续的推理流程是怎么做的。
>
> **作者回复**：我的意思后续的推理流程就是进入了不同的模式，而各个具体的模式是怎么做的呢，要参考其他模式各讲的具体设计。**的确，这一讲应该放在 CoT 之前，因为路由在前**，然后才进入其他具体模式。

> **元气🍣 🇨🇳**：
> 分拆是 LLM 分，还是用正则分？
>
> **作者回复**：**两者都用**。格式稳定的边界可用规则、Schema 或正则切，语义复杂的部分让模型提议分解，再由运行时检查每个子任务的输入、输出、验收和副作用。**不要让正则承担语义判断，也别让模型独自决定拆到哪里**。

> **街角·陌路△**：
> 因为我们大部分做的并不是 OpenClaw 或者 Claude Code 这种通用型智能体，所以在进行复杂度路由时，是不是应该优先由人来制定一些明确的规范，而不是完全交给智能体自主判断？
>
> 对于大部分已经比较确定的业务场景，可以先通过规则或者固定流程完成路由。只有遇到一些规则覆盖不到的特殊情况时，再增加一个特殊分支，让大模型进行自主判断。
>
> 同时，这种自主判断可能也不应该全部交给一个智能体来完成。对于比较复杂的任务，可以先进行子任务拆分，再根据每个子任务的复杂度，决定它应该走固定流程、交给单个智能体，还是继续进入更复杂的推理流程。
>
> 不过任务也不能无限拆分下去。在拆分过程中，应该设置一个明确的评判标准，用来判断当前任务是否已经达到了原子化，是否已经可以交给一个工具、流程或者智能体独立完成。
>
> 也就是说，整体上应该是**人工规则负责大部分稳定场景，大模型负责少量不确定场景**；复杂任务可以继续对子任务进行复杂度路由，但必须设置拆分的终止条件。这种方案是否可行？
>
> **作者回复**：**这个方案可行，也是企业里更稳的起点**：规则覆盖稳定的大头，模型只接不确定的尾部。拆分停止条件可以看五项：**一个负责人、一份输入、一份可验工件、一个副作用边界、一个有限预算**；继续拆不再提高可验证性时就停。

> **Geek_c8c9fa**：
> 佳哥，任务原子化一般都怎么做？LLM 做？还是有其他更好的方案？
>
> **作者回复**：通常采用**混合方案**：规则先切开已有业务边界，模型补充不确定的分解，运行时再按输入、输出、验收条件和副作用边界校验。**正则只适合格式稳定的切分，不能承担语义分解**。

> **Geek_7f0e61**：
> 佳哥，如何拆分任务原子？文中给出了 A1，A2，A3，A4 四个子任务，也给出了后续如何路由选择 decision，但是如何拆分得到？
>
> **作者回复**：A1 到 A4 先由模型或规则提出，再由确定性契约验收。一个子任务达到"**单一负责人、明确输入、可验输出、独立副作用边界、有限重试**"时，才算够原子；继续拆只会增加协调成本时就该停。

> **PatrickL**：
> 我制作了「复杂度路由」的流程图，评论区没法发图片，直接上 mermaid 代码（**Layer 1-6 完整架构图**：TaskAtom → Signal Extractor → RouteDecision → Lane/ReasoningMode → 推理诊室 → Validator → 路由日志/成本账单/质量评分）。
>
> **作者回复**：谢谢！

> **George**：
> 学习这门课的操作疑问：这些代码属于伪代码，只是为了更方便的介绍流程，还是在可以实际操作落地的代码？
>
> **作者回复**：**伪代码，仅方便理解设计而用**。我的 Repo 里面也有一些简单的能跑通的模式代码，课程案例内容太长了，我没有贴哪些代码。可以去 Repo 里面瞅瞅。未来我有时间了，会制作更完整代码和案例，以及相关训练营！

---

## 参考资料

- Dominique Jean Larrey. *Mémoires de chirurgie militaire et campagnes*. 1812-1817.
- Herbert A. Simon. *Administrative Behavior*. Macmillan, 1947.
- Eric J. Horvitz. *Reasoning about Beliefs and Actions under Computational Resource Constraints*. UAI, 1987.
- Thomas Dean, Mark Boddy. *An Analysis of Time-Dependent Planning*. AAAI, 1988.
- Stuart Russell, Eric Wefald. *Do the Right Thing: Studies in Limited Rationality*. MIT Press, 1991.
- Jacobs, Jordan, Nowlan, Hinton. *Adaptive Mixtures of Local Experts*. Neural Computation, 1991.
- Paul Viola, Michael Jones. *Rapid Object Detection using a Boosted Cascade of Simple Features*. CVPR, 2001.
- Hohpe, Woolf. *Enterprise Integration Patterns*. Addison-Wesley, 2003.
- Chen, Zaharia, Zou. *FrugalGPT: How to Use Large Language Models While Reducing Cost and Improving Performance*. 2023.
- Ong et al. *RouteLLM: Learning to Route LLMs with Preference Data*. arXiv:2406.18665, ICLR 2025.
- Christakopoulou et al. *Agents Thinking Fast and Slow: A Talker-Reasoner Architecture*. arXiv:2410.08328, Google DeepMind, NeurIPS 2024 Workshop on Open-World Agents.
- OpenAI. *Reasoning models*（reasoning effort / Structured Outputs）（API 文档）. / *Introducing GPT-5*. 2025-08-07.