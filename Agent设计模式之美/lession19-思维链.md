---
title: "17｜思维链：给 Agent 的判断留下一条可检查的路径"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/994287"
tags:
  - "#Agent/推理"
  - "#设计模式/CoT"
  - "#推理/可验证"
  - "#推理/ClaimStep"
  - "#推理/Validator"
audio: true
duration: "20:46"
publish_date: "2026-07-07"
course_progress: "17/43"
pattern_position: "推理模式组·第 2 个"
related:
  - "[[lession18-推理模块导论]]"
  - "[[lession20-复杂度路由]]"
  - "[[lession17-失败日记]]"
  - "[[lession13-记忆模块导论]]"
---

# 17｜思维链：给 Agent 的判断留下一条可检查的路径

> **核心命题**：CoT 在推理契约里的工程定位，是用一条主路径，把复杂判断拆成若干可验证命题，为最终 Validator 放行做准备。**完全不是"写出思考过程"的提示词技巧，而是一个生产系统里的推理结构。**

---

## 一、看起来像推理 vs 真的能用

沿用薪酬 SaaS 示例：

> 用户：帮我看一下上海市场部 6 月薪资快照里的异常。小冰缺 2 天考勤，咖哥奖金比上月多 18%，小雪社保基数变了。哪些可以自动通过，哪些要人审？

感知和记忆组件通过 RAG 等模式把咖哥那笔 18% 薪资异常相关的证据取回来了：薪资快照、奖金政策、审批记录、政策版本、历史工资等等。看起来事情已经差不多了。

给一个普通模型加一句"请逐步思考"，它会输出一大段看起来很完整的分析：

> 第一步看小冰，第二步看咖哥，第三步判断小雪。读起来是在推理。

不过我们还要追问：

- 小冰缺考勤，它查的是哪张考勤表？
- 咖哥奖金突增，有奖金审批单吗？
- 小雪基数变化的生效月份，系统是否做了确认？

> **CoT 在 Agent 中每一步都要能落回证据，最后形成一个能对账的判断和一个可验证的结构**：先确认异常事实 → 再拆工资组成 → 再核对适用规则 → 再查审批记录 → 最后决定要放行、拦截，还是转人工。

---

## 二、写出解题过程的重要性

### 2.1 思想谱系

| 时期 | 关键贡献 | 核心思想 |
|------|---------|---------|
| 1945 | Polya《How to Solve It》 | 理解问题 → 拟定计划 → 执行计划 → 回顾检验 |
| 1950s | GPS | 目标差距拆成子目标，形成问题求解框架 |
| 2021 | Scratchpad | 模型给出最终答案前显式写出中间步骤 |
| 2022 | CoT / Zero-shot-CoT | "Let's think step by step" |
| 2023+ | Self-Consistency / Tree of Thoughts / ReAct | 单链扩展成多路径、树状搜索、"思考-行动-观察"交替 |
| 推理模型时代 | o1 / DeepSeek-R1 | 推理链不再只是提示词技巧，而是模型推理时计算中主动使用的能力 |

> 这条推理思想谱系的共同点是**把复杂问题拆成中间状态**。过去依赖明确状态和手写规则，今天更多依赖概率生成、运行时预算和外部验证。
>
> 也正因为今天的推理变成了概率生成，工程师才更需要 **Trace、证据和验证器**。**模型越会想，系统越要会记账。**

---

## 三、CoT 在推理契约中的位置

上一讲我们立的推理契约（Reasoning Contract），每次任务进来要回答五个问题。**CoT 主要回答其中两个**：

| 回答的问题 | CoT 的位置 |
|----------|----------|
| **采用哪种推理拓扑？** | CoT 选择的是**一条主路径**：一步接一步往前走，前一步的输出成为后一步的输入，最后把结论收束到一个可以验证的决定上 |
| **由什么验证？** | 每一步都绑定可被工具、数据库、规则引擎或人工复核独立检查的对象 |

### CoT 不做什么

- **不会同时维护很多条候选路径再投票** → 那是 [[lession21-并行探索\|并行探索]]
- **不是持续观察环境、不断重写假设的迭代闭环** → 那是 [[lession22-迭代假设验证\|迭代假设验证]]

### CoT 的任务画像

任务主线基本清楚，但中间步骤不能省，必须沿着证据链逐步推进。

> CoT 把复杂判断拆成若干可验证命题。这串命题沿着一条主路径向前推进：**看到异常 → 拆解问题 → 核验证据 → 确认解决**。同时，系统也不断追问三个问题：结果是否满足目标，证据是否充分可靠，推理是否符合规则。

---

## 四、一步只表达一个可验证命题

### 4.1 反例：过载的步骤

> "综合分析员工 P 的工资、奖金、政策和历史记录，判断该薪资变化是否合理。"

看起来很全面，实际上**什么无法指导核查工作**。它把好几块内容全塞进同一步里：

- 观察事实（工资变化）
- 拆解问题（拆成几块）
- 计算差异
- 核对规则
- 查证审批
- 最终判断

系统既不知道这一步的核心是什么，也不知道应该用哪个工具验证，更不知道如果这一步错了，下游哪些结论就不成立了。

### 4.2 三个自检问题

每一步都应该能用以下三个问题检查。如果答不上来，这一步就不应该进入正式 Trace：

1. **这一步有没有一个明确的命题（立论）？**
2. **这个命题能不能被工具、规则、数据库或人独立检查？**
3. **如果这个命题错了，下游哪些步骤会被影响到？**

### 4.3 正例：员工 P 的 18% 薪资异常

```text
S1：P 的 6 月应发工资比 5 月高 18%。
S2：18% 的增量主要来自季度奖金。
S3：该租户规则要求应发涨幅超过 15% 时核查审批。
S4：季度奖金审批单存在，且金额、审批人、生效月份均匹配。
S5：在 S2、S3、S4 均通过验证的前提下，本次可以自动放行。
```

---

## 五、CoT 五步法：观察 → 拆解 → 推导 → 验证 → 决策

| 类型 | 英文 | 职责 | 示例 |
|------|------|------|------|
| **观察事实** | Observe | 把外部事实读进来 | "P 的 6 月应发工资比 5 月高 18%" |
| **拆解问题** | Decompose | 把大问题拆成可验证的小 claim | "增量来自哪里 / 是否触发审批规则 / 审批是否存在且匹配" |
| **计算或推导** | Derive | 基于已有事实推导或计算中间结果 | "18% 的增量主要来自季度奖金" |
| **验证** | Verify | 对照规则、审批、合同、政策或数据库进行核查 | "超过 15% 需要审批" "审批单金额和月份匹配" |
| **决策** | Decide | 在前面步骤都通过的前提下，做一个局部或最终决定 | "本次可以自动放行" |

### 5.1 结构化轨迹示例

```text
S1 OBSERVE
  主张：P 的 6 月应发工资比 5 月高 18%。
  证据：payroll_snapshot:2026-05:v3
  证据：payroll_snapshot:2026-06:v3

S2 DECOMPOSE
  主张：该异常需要拆成三个待验证子问题：增量来源、规则触发、审批匹配。
  依赖：S1
  输出：Q1 增量来自哪一项？
  输出：Q2 是否触发租户审批规则？
  输出：Q3 是否存在匹配的审批单？

S3 DERIVE
  主张：18% 的增量主要来自季度奖金，而不是基本工资或补贴。
  依赖：S1、S2.Q1
  工具：payroll_delta_calculator
  观察：basic_salary_delta = 0
  观察：allowance_delta = 0
  观察：bonus_delta = 8,400

S4 VERIFY
  主张：该租户规则要求"单月应发涨幅超过 15% 时核查审批"。
  依赖：S2.Q2
  证据：policy:payroll_anomaly:v7
  有效期：2026-05-01 起

S5 VERIFY
  主张：季度奖金审批单存在，金额、审批人、生效月份均匹配。
  依赖：S3、S4、S2.Q3
  证据：approval:BONUS-18472
  验证器：approval_validator
  结果：passed

S6 DECIDE
  主张：本次 18% 异常属于有审批支撑的季度奖金，可自动放行。
  依赖：S3、S4、S5
  验证器：payroll_release_gate
  结果：passed
```

> **这条链和普通"逐步分析"最大的不同，是它每一步都能回到外部对象**。薪资快照、政策版本、审批单都不是模型编的，验证器结果也不是模型自评。它们都能被系统重新读取、重新计算、重新验证。

### 5.2 普通 CoT vs 生产 CoT

| 普通 CoT | 生产 CoT |
|---------|---------|
| "我认为它合理。" | "S6 依赖 S3、S4、S5；S3 来自工资差异计算器；S4 来自政策版本 v7；S5 由审批验证器通过；所以 S6 可以放行。" |

> 这就是从 CoT 到**可核验轨迹**的关键变化。

---

## 六、可验证 CoT 的工程实现

答案不是靠模型"自觉遵守"，而是靠**结构化模式和验证闸门**。模型默认很容易输出一段散文式思维链，但工程系统不能直接相信这段散文。

### 6.1 整体流程

```text
LLM 生成 StepDraft（仅类型 + 命题草稿）
       ↓
程序完成工程化处理
       ↓
- 补充 evidence_refs（RAG/数据库/工具/规则/人审）
- 调用工具（Derive 步骤）
- 绑定 validator（从注册表）
- 检查 depends_on（依赖图）
- 原子性闸门
       ↓
合格的 → ClaimStep 进入正式 Trace
不合格 → 留在 scratchpad，或升级为人工复核
```

> **LLM 的职责是：把复杂任务拆成候选命题。程序的职责是：把候选命题编译成可验证节点。**

---

## 七、六层实现细节

### 第一层：Step Schema

每一步都是一个定义好的对象。`kind` 最关键——不分类型，模型很容易把"观察事实、解释原因、应用规则、做决定"塞进同一个步骤。类型一分开，程序就可以对不同步骤施加不同约束。

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Any


class StepKind(str, Enum):
    OBSERVE = "observe"        # 观察事实
    DECOMPOSE = "decompose"    # 拆解问题
    DERIVE = "derive"          # 计算或推导
    VERIFY = "verify"          # 核对规则或证据
    DECIDE = "decide"          # 做出决定


class StepStatus(str, Enum):
    DRAFT = "draft"
    PASSED = "passed"
    FAILED = "failed"
    NEEDS_REVIEW = "needs_review"


@dataclass(frozen=True)
class EvidenceRef:
    source_id: str
    source_type: str
    version: str | None = None
    effective_at: str | None = None
    content_hash: str | None = None


@dataclass
class ClaimStep:
    step_id: str
    kind: StepKind
    claim_text: str                          # 给人看的简短命题
    subject: str                             # 给程序看的结构化命题
    predicate: str
    object: Any | None = None
    depends_on: list[str] = field(default_factory=list)
    evidence_refs: list[EvidenceRef] = field(default_factory=list)
    action: str | None = None
    observation: Any | None = None
    validator: str | None = None
    status: StepStatus = StepStatus.DRAFT
```

### 第二层：原子性闸门（Atomicity Gate）

模型仍然可能把多个判断塞进命题文本，所以加一个**原子性闸门**做两类检查。

**第一类：静态检查**

```python
AMBIGUOUS_CONNECTORS = [
    "并且", "同时", "以及", "因此", "所以", "从而",
    "说明", "证明", "可以判断", "综合来看",
]

def looks_too_composite(claim_text: str) -> bool:
    hits = [word for word in AMBIGUOUS_CONNECTORS if word in claim_text]
    return len(hits) >= 2
```

> 这类检查不能完美判断语义，但能拦掉一大批明显承载过多的步骤，因为我们希望保证命题的原子性，**一次处理一件事儿**。

**第二类：类型约束**

```python
def validate_step_shape(step: ClaimStep) -> list[str]:
    errors: list[str] = []
    if not step.claim_text.strip():
        errors.append("claim_text is required")
    if not step.subject or not step.predicate:
        errors.append("structured claim requires subject and predicate")
    if looks_too_composite(step.claim_text):
        errors.append("claim_text seems to contain multiple claims")

    if step.kind is StepKind.OBSERVE:
        if not step.evidence_refs:
            errors.append("OBSERVE step requires evidence_refs")
        if step.depends_on:
            errors.append("OBSERVE step should not depend on derived steps")

    if step.kind is StepKind.DECOMPOSE:
        if not step.depends_on:
            errors.append("DECOMPOSE step requires an observed problem")
        if step.evidence_refs:
            errors.append("DECOMPOSE should output questions, not claim external evidence")

    if step.kind is StepKind.DERIVE:
        if not step.depends_on:
            errors.append("DERIVE step requires upstream dependencies")
        if not step.action:
            errors.append("DERIVE step requires a tool or computation action")

    if step.kind is StepKind.VERIFY:
        if not step.evidence_refs:
            errors.append("VERIFY step requires evidence_refs")
        if not step.validator:
            errors.append("VERIFY step requires validator")

    if step.kind is StepKind.DECIDE:
        if not step.depends_on:
            errors.append("DECIDE step requires dependencies")
        if not step.validator:
            errors.append("DECIDE step requires decision gate validator")

    return errors
```

### 第三层：验证器注册表（ValidatorRegistry）

"一步可验证"还有一个隐藏条件：**系统得知道谁来验证它**。所以程序里要有一个验证器注册表，把某类谓词（predicate）绑定到确定性验证器。

```python
from collections.abc import Callable

ValidatorResult = tuple[StepStatus, Any]
Validator = Callable[[ClaimStep], ValidatorResult]


class ValidatorRegistry:
    def __init__(self) -> None:
        self.validators: dict[str, Validator] = {}

    def register(self, name: str, validator: Validator) -> None:
        self.validators[name] = validator

    def run(self, step: ClaimStep) -> ValidatorResult:
        if not step.validator:
            return StepStatus.NEEDS_REVIEW, {"reason": "missing validator"}
        validator = self.validators.get(step.validator)
        if validator is None:
            return StepStatus.NEEDS_REVIEW, {
                "reason": f"unknown validator: {step.validator}"
            }
        return validator(step)
```

> **Validator 不一定是另一个 LLM**。能确定性验证的，尽量用数据库、规则引擎、权限系统、计算器或测试框架。
>
> - 金额是否一致 → 交给计算器
> - 政策是否生效 → 交给规则引擎
> - 审批人是否具备权限 → 交给权限系统
> - 工资项是否重复发放 → 交给数据库查询
>
> LLM 适合拆解问题、生成检查计划、解释结果，但**不适合独自承担所有判断**。

### 第四层：依赖图防止错误一路传下去

一步一个命题以后，下一件事是把步骤连成 **DAG**。最终决定不能直接依赖自然语言上下文，而要依赖**已经通过验证的上游节点**。

```python
@dataclass
class ClaimTrace:
    trace_id: str
    steps: list[ClaimStep] = field(default_factory=list)
    final_decision: str | None = None
    stop_reason: str | None = None

    def index(self) -> dict[str, ClaimStep]:
        return {step.step_id: step for step in self.steps}

    def dependencies_passed(self, step: ClaimStep) -> bool:
        by_id = self.index()
        for dep_id in step.depends_on:
            dep = by_id.get(dep_id)
            if dep is None:
                return False
            if dep.status is not StepStatus.PASSED:
                return False
        return True
```

> **失效传播**：如果 S4 的政策版本验证失败，S5 和 S6 就不能继续当作事实使用。它们不是"低置信度"，而是**依赖已经断了**。系统必须知道，哪一步错了，然后哪些下游结论必须一起作废。

### 第五层：追踪运行器（Trace Runner）

```python
class TraceRunner:
    def __init__(self, registry: ValidatorRegistry) -> None:
        self.registry = registry

    def run(self, trace: ClaimTrace) -> ClaimTrace:
        for step in trace.steps:
            shape_errors = validate_step_shape(step)
            if shape_errors:
                step.status = StepStatus.NEEDS_REVIEW
                step.observation = {
                    "reason": "invalid_step_shape",
                    "errors": shape_errors,
                }
                trace.stop_reason = "invalid_step_shape"
                return trace

            if not trace.dependencies_passed(step):
                step.status = StepStatus.NEEDS_REVIEW
                step.observation = {
                    "reason": "dependencies_not_passed",
                    "depends_on": step.depends_on,
                }
                trace.stop_reason = "dependency_not_passed"
                return trace

            if step.kind in {StepKind.VERIFY, StepKind.DECIDE}:
                status, observation = self.registry.run(step)
                step.status = status
                step.observation = observation
                if status is not StepStatus.PASSED:
                    trace.stop_reason = f"step_failed:{step.step_id}"
                    return trace
            else:
                # OBSERVE / DECOMPOSE / DERIVE 通常由证据读取或工具执行填充状态
                step.status = StepStatus.PASSED

        decisions = [
            step for step in trace.steps
            if step.kind is StepKind.DECIDE
        ]
        if decisions and all(step.status is StepStatus.PASSED for step in decisions):
            trace.final_decision = str(decisions[-1].object)
            trace.stop_reason = "all_required_claims_verified"
        else:
            trace.stop_reason = "no_decision_step"
        return trace
```

> Trace Runner 的职责不是"替模型思考"，而是**执行结构化步骤、检查依赖、运行验证器、决定是否停止**。

### 第六层：LLM 输出 StepDraft，程序编译 ClaimStep

```python
@dataclass
class StepDraft:
    kind: StepKind
    claim_text: str
    suggested_subject: str | None = None
    suggested_predicate: str | None = None
    suggested_object: Any | None = None
    suggested_evidence_query: str | None = None


class ClaimCompiler:
    def compile(
        self,
        draft: StepDraft,
        step_id: str,
        evidence_refs: list[EvidenceRef],
        validator: str | None,
        depends_on: list[str],
    ) -> ClaimStep:
        if not draft.suggested_subject or not draft.suggested_predicate:
            raise ValueError("Draft lacks structured claim fields")
        return ClaimStep(
            step_id=step_id,
            kind=draft.kind,
            claim_text=draft.claim_text,
            subject=draft.suggested_subject,
            predicate=draft.suggested_predicate,
            object=draft.suggested_object,
            depends_on=depends_on,
            evidence_refs=evidence_refs,
            validator=validator,
        )
```

> **为什么多这一层？** LLM 很适合拆问题，但不应该让它自己决定"我已经有证据了"。
>
> - 证据应该来自 RAG、数据库、工具、规则系统或人工确认
> - validator 应该来自系统注册表
>
> 程序设计的完整落地过程：**LLM 提出步骤 → schema 约束 → 原子性检查 → 证据绑定 → 验证器绑定 → 依赖检查**。能通过，才进入推理追踪链路；不能通过，就停在草稿区（scratchpad），或者升级为人工复核（human review）。

---

## 八、三道闸门：证据 / 确定性验证 / 升级退出

在上面的流程中，LLM 只负责提出候选步骤。真正决定这一步能不能进入正式轨迹的，是**闸门**。一条 CoT 进入生产，至少要过三道闸门：

### 8.1 第一道：证据闸门

关键判断必须绑定证据，没有证据的判断，不能进入最终决定。

> 比如"P 的奖金应该是正常发放"，如果没有审批单、政策版本、工资明细支撑，只能算模型猜测。它可以留在草稿区里，不能进入决策。

**证据闸门要检查**：

- 证据是否存在
- 来源是否权威
- 版本是否正确
- 有效期是否覆盖当前场景
- 证据是否属于当前租户和当前员工

> 企业系统里，**跨租户误用证据，是比推理错误更危险的事故**。

### 8.2 第二道：确定性验证闸门

能用确定性程序验证的，**不要交给模型自评**。

| 验证什么 | 交给谁 |
|---------|-------|
| 金额是否一致 | 计算器 |
| 政策是否生效 | 规则引擎 |
| 审批人是否具备权限 | 权限系统 |
| 工资项是否重复发放 | 数据库查询 |

LLM 可以解释验证结果，但**不应该替代验证器本身**。

### 8.3 第三道：升级退出闸门

CoT 适合一条主路径可以走通的问题，但如果它走不通，**就要退出**。以下情况应该触发升级：

- 证据缺失、证据冲突
- 规则版本不唯一
- 验证器失败
- 关键依赖无法确认
- 模型连续两次改写同一事实
- 工具结果和模型解释不一致

> 此时不要让 CoT 继续硬编，而应该升级到**并行探索**、**迭代假设验证**，或者**人工复核**。
>
> **这些闸门就是一次 CoT 的停止条件**。当所有关键依赖均通过验证，或者出现无法通过单链解决的冲突并升级，CoT 就结束，并进入下一步工作流程。

---

## 九、供应侧状态（Provider State）vs 应用层轨迹（Application Trace）

### 9.1 为什么需要区分

Agent 不是一次性问答。真实任务会**多轮调用工具**，可能中途失败，可能切换模型，也可能跨会话继续执行。工程上必须区分两种状态：

| 状态 | 位置 | 职责 | 类比 |
|------|------|------|------|
| **Provider Reasoning State** | 模型供应商 / API 层 | 让模型下一轮接着想 | 模型续接用的**内部状态包** |
| **Application Audit Trace** | 应用侧 | 业务审计、复盘、出错定位 | 结构化的**责任链** |

### 9.2 供应侧推理状态

在跨轮工具调用时，应用可以通过：

- `previous_response_id`（上一轮响应 ID）
- 手动传回上一轮的推理项

帮助模型接着上一轮继续推理。在无状态（stateless）或零数据保留（zero data retention）场景中，也可以通过**加密推理项**在请求之间携带必要的推理上下文。

### 9.3 应用侧审计轨迹

应用侧审计轨迹要记录：

- 任务 ID、推理模式、每一步主张
- 证据引用、工具动作、观察结果
- 验证器、步骤状态、最终决定、停止原因

> 供应商侧推理状态**不是业务审计记录**。你不能指望一个加密推理项告诉审核员：为什么某人的工资被放行。
>
> **供应商侧推理状态负责"续接思考"，应用侧审计轨迹负责"留下责任链"。**

### 9.4 跨轮设计原则

- 模型内部状态 → 由 Provider state 续接
- 应用审计轨迹 → 由 Application trace 结构化保存
- 长期经验复用 → 经过验证、脱敏、蒸馏后再写入 [[lession13-记忆模块导论\|记忆层]]

> **不要只保存最终答案，也不要把原始长篇 CoT 全量长期保存。**

---

## 十、总结

一句话总结：

> **CoT 在推理契约里的工程定位，是用一条主路径，把复杂判断拆成若干可验证命题，为最终 Validator 放行做准备。**

我们这里讲的 CoT：

- ❌ 不是"写出思考过程"的提示词技巧
- ✅ 而是**生产系统里的推理结构**

任务进来了：

1. 先经过推理契约判断是否适合单链
2. 如果适合，就沿着主路径推进
3. 每一步只表达一个可验证命题
4. 每个命题绑定证据、依赖和验证器
5. 最后由 Decision Gate 检查整条链是否可以放行

> **提醒：这么复杂的 CoT 设计流程，该用才用，可别到处都用**。简单任务被硬塞推理，又慢又贵又啰嗦，何必呢。这节课我们讲的是**设计思想**，可不是万能法则哈。

---

## 十一、思考题

1. 你的 Agent 现在是不是所有请求都默认"逐步思考"？挑几个最简单的查询看一眼，它有没有为了配合提示，硬编一段没必要的推理？如果按推理边界分四档（不开思考、浅想、结构化推理、高风险推理），有多少请求其实可以不开思考？
2. 你系统里最近一次有争议的 Agent 决策，能不能把它的推理拆成观察、规则核对、查证、决策，每一步都挂上证据？哪一步挂不上？如果有挂不上的地方，恰恰是 `is_accountable()` 会拦下的地方。
3. 你现在保存推理轨迹吗？今天就给 trace 加上 `claim` 和 `evidence_refs`，找一个跨了三轮工具调用的任务，观察一下。

---

## 十二、下一讲预告

下一讲我们讲第二个推理模式：**复杂度路由（Complexity-Based Routing）**。

这一讲我们一直在说"该想才想、想多深"，但判断该想多深这件事本身要花成本。**复杂度路由就是把这件事做成分诊台**。

---

## 精选留言摘录

> **Geek_936fa6**：
> 每天来追更，感觉学了不少东西，期待后面出一个实操课程把这些东西应用起来。
>
> **作者回复**：那必须滴。

> **天下霸唱**：
> 老师，我用 Claude，是不是这个 harness 都实现这些功能了？
>
> **作者回复**：当然了。**我们 Agent 设计的思想基础就是——不是所有的场景都直接上大 Harness！总有需要上设计的地方**。这就如同在学习 Linux 和 K8S 的设计思想是一样的。越往后，这些设计就越被封装起来了，而这些设计思想仍然是珍贵的。

> **元气🍣 🇨🇳**：
> 目的就说把对的东西一直走下去，就是"过过脑子，检讨检讨"在 Agent 中的应用，不过不能照搬这个思路。
>
> **作者回复**：这个比喻很形象。真正落地时要把"过过脑子"变成**可检查的工件**：用了哪些事实、做了哪些验证、为何选这个动作、还剩什么不确定性。这样才能复核，不能只要求模型写一段看起来很深的自我检讨。

> **街角·陌路△**：
> 对这段内容的理解是，**执行型 Agent 里的 CoT，并不是简单地让 Agent 把自己的思考过程全部写出来，而是把人经过实践和验证得到的判断经验，拆成一套显性的、可验证的推理流程，再交给 Agent 去执行**。
>
> 它其实和上次提到的 [[lession17-失败日记\|失败日记]] 有点像。二者的方向都是希望让 Agent 的执行更加工程化，而不是简单地换一个能力更强的 Agent，然后让它在没有约束的情况下自由发挥。
>
> **作者回复**：理解准确。**工程上要外显的是假设、证据、检查结果、选择和不确定性，不需要公开模型的全部隐藏思维**。这样既能复核，又不会把"写得像推理"误当成"真的受过验证"。

> **Geek_7f0e61**：
> CoT 感觉把纵轴的感知、行动、思考、反思，都囊括了。学习下来感觉内容挺多，工程实施上也比较繁重。
>
> **作者回复**：**不能照搬照抄实施**。我讲解的时候，是一股脑把概念和思考全部堆出来，不然讲解不完整。但是实际场景中是**按需使用**。

> **花魂**：
> 是不是可以理解成，当一个任务可以链式思考，但复杂度比较高，审计要求比较高的时候，适合使用这种结构化推理模式？如果比较简单，就靠大模型自己的 CoT 推理就可以了。
>
> 另外，在实现的时候，是先使用 prompt 引导模型按照这个思想思考，定义步骤类型、验证类型、闸门类型等，然后代码定义这些内容的验证和串联，再把 prompt 和代码作为上下文的一部分传递给 LLM，让大模型自己按照要求做推理执行验证。如果有步骤不通过，就中止给人类判断，如果都通过，就输出我们平时看到的 LLM 给出的任务执行步骤？
>
> **作者回复**：什么时候用结构化 CoT vs 靠大模型自己的 CoT。
>
> 你的直觉基本正确，但两个维度（复杂度 + 审计）只是完整判断尺子的两条。结构化 CoT 是否值得上，**看的是五个维度的任一维度是否高，不用五个都高**：
>
> | 维度 | 低 → 大模型自己的 CoT | 高 → 上结构化 CoT |
> |------|--------------------|-----------------|
> | 失败代价 | … | … |
> | 审计要求 | … | … |
> | 一致性要求 | … | … |
> | 回溯需求 | … | … |
> | 可验证性 | … | … |
>
> 任何一维高，就该考虑要不要上结构化 CoT。**触发点是"错了怎么办"，不是"任务本身有多难"**。
>
> 关于验证：**最好不要让用一个模型自己验证自己的 CoT**。

---

## 参考资料

- George Pólya. *How to Solve It*. Princeton University Press, 1945.
- Newell, Shaw, Simon. *Report on a General Problem-Solving Program*. IFIP, 1959
- Nye et al. *Show Your Work: Scratchpads for Intermediate Computation with Language Models*. arXiv:2112.00114, 2021.
- Wei et al. *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. arXiv:2201.11903, NeurIPS 2022.
- Kojima et al. *Large Language Models are Zero-Shot Reasoners*. arXiv:2205.11916, NeurIPS 2022.
- Wang et al. *Self-Consistency Improves Chain of Thought Reasoning*. arXiv:2203.11171, ICLR 2023.
- Yao et al. *Tree of Thoughts: Deliberate Problem Solving with Large Language Models*. arXiv:2305.10601, NeurIPS 2023.
- Yao et al. *ReAct: Synergizing Reasoning and Acting in Language Models*. arXiv:2210.03629, ICLR 2023.
- OpenAI. *Reasoning models*.
- DeepSeek-AI. *DeepSeek-R1 incentivizes reasoning in LLMs through reinforcement learning*. Nature 645:633-638, 2025.
- Anthropic. *The "think" tool: Enabling Claude to stop and think*. 2025.
- OpenAI / Frontier Model Forum. *Chain-of-Thought Monitorability 评估*. 2025.
- Anthropic. *Reasoning Models Don't Always Say What They Think*. arXiv:2505.05410, 2025.