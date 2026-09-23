---
title: "19｜并行探索：一题多解，择优录取"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/996240"
tags:
  - "#Agent/推理"
  - "#设计模式/并行探索"
  - "#推理/多视角"
  - "#推理/LensRegistry"
  - "#推理/聚合"
audio: true
duration: "19:30"
publish_date: "2026-07-13"
course_progress: "19/43"
pattern_position: "推理模式组·第 4 个"
related:
  - "[[lession16-推理模块导论]]"
  - "[[lession17-思维链]]"
  - "[[lession18-复杂度路由]]"
  - "[[lession20-迭代假设验证]]"
---

# 19｜并行探索：一题多解，择优录取

> **核心命题**：并行探索，是对同一个问题展开**多条独立的推理路径**，让每条路径带着自己的口径、证据和结论回来，再通过聚合、验证器和分歧裁决，把结果收束成自动判断、弱通过、人审或重新路由。**普通投票追求选出赢家，并行探索追求裁决分歧。选不出安全赢家时，verdict=None 本身就是一个正确结果。**

---

## 一、为什么单跑一条链不敢信

同一道难题，向你的 Agent 问两遍，两次的答案一样吗？

**答案是不一定**。LLM 的推理是概率生成，`temperature` 不为零的时候，每次解码走的路径都可能不同。

| 题目难度 | 单次采样的风险 |
|---------|--------------|
| **简单题** | 怎么采样都能采到对的答案 |
| **难题** | 答案的分布散开了，单次采样只是分布里的一个采样点 |

至于分布本身有多宽、这一次采样偏不偏，我们**完全不知道**。**单跑一条链，等于赌这一次采样刚好采到对的那条路径。**

并行探索同一道题同时跑几条路径，再把几条结果聚合起来择优。**它用多条样本去逼近那个分布，从而得到一个比单次采样更稳的结论。这个做法的本质，是用多样性换可靠性。**

> 机器学习里的**集成学习**，用一堆有差异的弱模型投票来决定最终的预测或分类结果，就是这个道理。到了 LLM 时代，**Self-Consistency** 把并行探索的思路引入了大模型的推理过程：不再只取一条 CoT，而是采样多条推理路径，再选出最一致的答案。

到执行型 Agent 里，**并行探索不能只停在"多采样几次"**，我们在强调"多条路径"的同时，也强调**证据、裁决和分歧**。

---

## 二、工程定义 + 双轴定位

> **并行探索，是对同一个问题展开多条独立推理路径，让每条路径带着自己的口径、证据和结论回来，再通过聚合、验证器和分歧裁决，把结果收束成自动判断、弱通过、人审或重新路由。**

在 [[lession4-双轴框架（下）\|双轴图谱]] 里，它落在**推理行 / 并行列**：

| 推理契约的提问 | 并行探索的答案 |
|--------------|--------------|
| **采用哪种推理拓扑？** | 并行 |
| **由什么验证？** | 聚合策略、验证器，以及**分歧裁决** |

---

## 三、三种形态：多条路径是怎么造出来的

### 3.1 形态一：采样投票（Self-Consistency）

同一个 prompt，采样 N 条 CoT 链，最后对答案投票。**Self-Consistency** 是代表。

- **适合**：数学题、分类、抽取、单一口径下的规则判断
- **特点**：简单、便宜、效果常常不错
- **前提**：几条路径的错误不要高度相关

### 3.2 形态二：结构化搜索（Tree of Thoughts）

**Tree of Thoughts** 把推理从一条链扩展成一棵树：

```text
每一步生成多个候选 thought
   ↓
模型或验证器对候选做评估
   ↓
好的继续展开，差的剪掉
   ↓
必要时回溯
```

- **适合**：需要前瞻、试错、规划和搜索的问题
- **特点**：强调多路径探索、自评估、前瞻和回溯
- **效果**：在 Game of 24 等任务上，相对普通 CoT 有明显提升

### 3.3 形态三：多视角并行（最值得重视）

> **这才是执行型 Agent 里最值得重视的一种**。它不是同一个 prompt 多采样几次，而是让几条路径**各按一个不同的业务口径去推同一件事**。

| 案例 | 路径一 | 路径二 | 路径三 |
|------|--------|--------|--------|
| **小雪的双版政策奖金** | 旧版政策有效期口径 | 新版政策发放时点口径 | 审批日期切换口径 |
| **跨月年终奖** | 实际发放月口径 | 税法所属期口径 | 地方社保规则口径 |

> 这种问题靠采样投票是**无法解决**的。你把同一个 prompt 采样 10 次，如果它们都默认用了"发放月口径"，那 10 条链只是同一个前提下的 10 次复读。真正需要的是**业务口径的多样性，而不是随机性的多样性**。

### 3.4 三种形态对比

| 形态 | 多样性来源 | 适合 | 局限 |
|------|----------|------|------|
| **采样投票** | 随机采样 | 有标准答案 | 错误高度相关时无效 |
| **结构化搜索** | 自评+剪枝+回溯 | 需要前瞻试错 | 计算开销大 |
| **多视角并行** | 业务口径 | 制度分歧、规则多版本 | 依赖口径注册 |

---

## 四、工程实现：五层结构

从 `RouteDecision` 到 `ReasoningTrace` 的完整数据流：

```text
RouteDecision
   ↓
LensRegistry（口径注册表）
   ↓
N 条相互隔离的路径各自计算
   ↓
聚合分级（不只数票）
   ↓
口径分歧被完整记录到 ParallelTrace
```

### 第一层：接住路由结果

并行探索不是任务自己走进来的，是 [[lession20-复杂度路由\|分诊台]] 送进来的。`RouteDecision` 至少要带：

```yaml
lane: structured_reasoning
reasoning_mode: PARALLEL
max_paths: 3
human_gate: before_write   # 或 before_decision
budget:
  token: ...
  latency: ...
  tool_calls: ...
```

> **最重要的是 `lane` 和 `reasoning_mode` 继续分开**。`reasoning_mode = PARALLEL` 只说明这次怎么推理。**它不说明能不能写库、能不能自动放行、能不能跳过人审**。能不能写，仍然由 `lane`、`risk`、`mechanical_ready` 和 `human_gate` 决定。

### 第二层：口径注册表（LensRegistry）

多视角并行的"视角"从哪里来？prompt 里写一句"请从多个角度分析"是**没什么技术含量**的。因为模型现场编出来的角度**没有出处**。后面验证器不知道怎么查，审计时也说不清为什么当时按这个口径算。

更稳的做法是建一个**口径注册表 `LensRegistry`**。每个口径都应该登记：

| 字段 | 含义 |
|------|------|
| `lens_id` | 口径唯一 ID |
| 口径名称 | 人能读的名字 |
| 政策出处 | 这个口径来自哪个政策/制度 |
| 适用条件 | 什么时候用这条口径 |
| 需要哪些证据 | 哪些 `evidence_refs` 必须就位 |
| 默认验证器 | 由谁来验证它 |
| 风险等级 | 风险标签 |

### 小雪奖金案例：三条 lens

```yaml
- lens_id: "old_policy_effective_period"
  出处: "旧版奖金政策 + 考核周期条款"
  适用: "奖金对应考核周期落在旧版有效期内"

- lens_id: "new_policy_payout_date"
  出处: "新版奖金政策 + 发放时点条款"
  适用: "发放动作发生在新版上线之后"

- lens_id: "approval_date_switch"
  出处: "历史审批惯例 + 审批单日期"
  适用: "政策切换期内，以审批通过日期确定适用版本"
```

> **模型可以提议新口径，但新口径不能直接进入自动判断**。它必须说明来源，经过人审确认，再进入注册表。**这些口径也是业务资产的一部分**。

### 第三层：路径隔离

多条路径要独立，工程上的落点就是**上下文隔离**：

- 三条路径各拿自己的口径、证据要求和验证器
- 各算各的结论
- **中途不共享中间结果**

> 模型在分析第二个口径时，不应该看见第一个口径的结论；分析第三个口径时，也不能被前两个结论影响。**并行探索的可靠性，来自路径之间的差异和独立。如果路径互相污染，后面的投票和聚合就会产生虚假的置信度，最后得到的判断就不是独立的。**

### 第四层：每条路径本身是一条 mini 可验证链

每条路径沿着自己的口径推进时，同样要：

- 观察事实、推导中间结果、核对证据
- 每一步都要绑定证据

路径的产出是一个结构化的 **`PathResult`**，里面记录：

| 字段 | 含义 |
|------|------|
| `lens` | 用的哪个口径 |
| `claim` | 结论是什么（自然语言） |
| `normalized_claim` | 归一化的结论（用于投票） |
| `evidence_refs` | 凭什么 |
| `validator` | 谁来验证 |
| `validation_status` | 验证结果 |
| `confidence` | 自评置信度 |
| `latency` / `tool_calls` / `reasoning_tokens` | 资源消耗 |

> **也就是说广度诊室里跑的，其实是 N 条 CoT 可验证链。**
>
> 这里也要强调：**模型自评置信度只是弱信号**。薪酬、税务、权限、合同这类任务，**不能靠模型给出的置信度来放行**。真正能放行的，是**证据和验证器**。

### 第五层：聚合分级，分歧落盘

多条 `PathResult` 回来之后，系统要聚合。**最省事的做法当然是多数投票**。但多数投票有一个危险本能：**它总能投出赢家，这对执行型 Agent 很危险**。因为有些分歧不是噪声，而是真实制度分歧。把它硬投成赢家，就是把不确定性盖住了。

生产里的聚合应该**分级**：

| 聚合场景 | 处理方式 |
|---------|---------|
| **全部一致** | 自动产出判断，置信较高 |
| **多数一致，但存在异议** | 只读分析可以弱通过并标注异议；高风险写入**不能直接自动放行**，必须看 `human_gate` |
| **平票、三方分裂、关键路径失败** | 转人审或重新路由 |
| **证据缺失、验证失败、对象状态不完整** | 不聚合，先补证 |

聚合完成后，整趟并行探索要写进 **`ReasoningTrace`**：

```text
mode = PARALLEL
path_specs
path_results
aggregation_decision
disagreement_level
winning_path_ids
dissenting_path_ids
failed_path_ids
stop_reason
```

> **口径分歧要格外注意**，因为它本身就是审计事件和重要信息。是 Agent 系统向人类汇报成果的一部分，**并不一定非要选出一个"最正确的结果"**。

---

## 六、最小生产级代码骨架

### 6.1 数据结构

```python
from __future__ import annotations
import asyncio
import time
from collections import Counter
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Awaitable, Callable


class PathStatus(str, Enum):
    SUCCESS = "success"
    FAILED = "failed"
    TIMEOUT = "timeout"


class ValidationStatus(str, Enum):
    NOT_RUN = "not_run"
    PASSED = "passed"
    FAILED = "failed"
    NEEDS_REVIEW = "needs_review"


class AggregationRoute(str, Enum):
    AUTO_DECISION = "auto_decision"
    AUTO_WITH_DISSENT = "auto_with_dissent"
    HUMAN_REVIEW = "human_review"
    REROUTE = "reroute"


class DisagreementLevel(str, Enum):
    NONE = "none"
    MINOR_DISSENT = "minor_dissent"
    SPLIT = "split"
    INSUFFICIENT = "insufficient"


@dataclass(frozen=True)
class EvidenceRef:
    source_id: str
    source_type: str
    version: str | None = None
    effective_at: str | None = None
    content_hash: str | None = None


@dataclass(frozen=True)
class PathSpec:
    path_id: str
    lens: str
    instruction: str
    required_evidence: list[str] = field(default_factory=list)
    validator: str | None = None
    timeout_ms: int = 12_000


@dataclass
class PathResult:
    path_id: str
    lens: str
    claim: str | None = None
    normalized_claim: str | None = None
    evidence_refs: list[EvidenceRef] = field(default_factory=list)
    rationale_summary: str | None = None
    confidence: float | None = None
    validator: str | None = None
    validation_status: ValidationStatus = ValidationStatus.NOT_RUN
    status: PathStatus = PathStatus.SUCCESS
    error: str | None = None
    latency_ms: int = 0
    tool_calls: int = 0
    reasoning_tokens: int = 0

    def can_vote(self, require_validation: bool = True) -> bool:
        if self.status is not PathStatus.SUCCESS:
            return False
        if not self.normalized_claim:
            return False
        if require_validation and self.validation_status is not ValidationStatus.PASSED:
            return False
        return True


@dataclass(frozen=True)
class AggregationPolicy:
    require_validated_paths: bool = True
    require_unanimous_for_auto: bool = True
    allow_auto_with_dissent: bool = False
    min_agreement_for_dissent: float = 2 / 3


@dataclass
class AggregationDecision:
    verdict: str | None
    route: AggregationRoute
    disagreement: DisagreementLevel
    agreement_ratio: float
    winning_path_ids: list[str] = field(default_factory=list)
    dissenting_path_ids: list[str] = field(default_factory=list)
    failed_path_ids: list[str] = field(default_factory=list)
    note: str = ""


@dataclass
class ParallelTrace:
    trace_id: str
    task_id: str
    route_decision_id: str | None
    path_specs: list[PathSpec]
    path_results: list[PathResult] = field(default_factory=list)
    policy: AggregationPolicy = field(default_factory=AggregationPolicy)
    aggregation_decision: AggregationDecision | None = None

    def aggregate(self) -> AggregationDecision:
        usable = [
            result for result in self.path_results
            if result.can_vote(self.policy.require_validated_paths)
        ]
        failed = [
            result.path_id for result in self.path_results
            if result.status is not PathStatus.SUCCESS
        ]
        if len(usable) < 2:
            return AggregationDecision(
                verdict=None,
                route=AggregationRoute.HUMAN_REVIEW,
                disagreement=DisagreementLevel.INSUFFICIENT,
                agreement_ratio=0.0,
                failed_path_ids=failed,
                note="有效路径不足，不能自动聚合",
            )
        votes = Counter(result.normalized_claim for result in usable)
        top_count = max(votes.values())
        top_claims = [claim for claim, count in votes.items() if count == top_count]
        if len(top_claims) > 1:
            return AggregationDecision(
                verdict=None,
                route=AggregationRoute.HUMAN_REVIEW,
                disagreement=DisagreementLevel.SPLIT,
                agreement_ratio=top_count / len(usable),
                failed_path_ids=failed,
                note="多条路径平票或分裂，转人审",
            )
        top_claim = top_claims[0]
        winners = [r for r in usable if r.normalized_claim == top_claim]
        dissenters = [r for r in usable if r.normalized_claim != top_claim]
        agreement_ratio = len(winners) / len(usable)
        if agreement_ratio == 1.0:
            return AggregationDecision(
                verdict=top_claim,
                route=AggregationRoute.AUTO_DECISION,
                disagreement=DisagreementLevel.NONE,
                agreement_ratio=agreement_ratio,
                winning_path_ids=[r.path_id for r in winners],
                failed_path_ids=failed,
                note="所有有效路径一致",
            )
        if (
            not self.policy.require_unanimous_for_auto
            and self.policy.allow_auto_with_dissent
            and agreement_ratio >= self.policy.min_agreement_for_dissent
        ):
            return AggregationDecision(
                verdict=top_claim,
                route=AggregationRoute.AUTO_WITH_DISSENT,
                disagreement=DisagreementLevel.MINOR_DISSENT,
                agreement_ratio=agreement_ratio,
                winning_path_ids=[r.path_id for r in winners],
                dissenting_path_ids=[r.path_id for r in dissenters],
                failed_path_ids=failed,
                note="多数一致，但存在异议，必须留痕",
            )
        return AggregationDecision(
            verdict=None,
            route=AggregationRoute.HUMAN_REVIEW,
            disagreement=DisagreementLevel.MINOR_DISSENT,
            agreement_ratio=agreement_ratio,
            winning_path_ids=[r.path_id for r in winners],
            dissenting_path_ids=[r.path_id for r in dissenters],
            failed_path_ids=failed,
            note="存在分歧，当前策略不允许自动通过",
        )
```

### 6.2 关键设计点

| 对象 | 职责 |
|------|------|
| **`PathSpec`** | 执行前的契约：规定这条路径采用什么口径、使用什么指令、需要哪些证据、由哪个验证器检查。没有它，并行探索容易退化成同一个 prompt 多采样几次 |
| **`PathResult`** | 路径执行后的回执。不只保存答案，还保存归一化答案、证据引用、验证状态、失败原因、工具调用和推理 token |
| **`claim` vs `normalized_claim`** | 自然语言不能直接投票。"9600 元""9,600""旧版口径金额 9600"语义相同，**聚合前必须归一化** |
| **`can_vote()`** | 路径成功、有规范化结论，并且在策略要求时通过验证器，才**有资格参与聚合** |
| **`aggregate()`** | 不是简单地 `most_common()`，而是先判断有效路径是否足够，再判断是否存在唯一赢家 |

> **对薪酬、税务、权限这类高风险场景，默认策略应该是 `require_unanimous_for_auto=True`，也就是不允许用 2:1 多数直接自动写入。**

### 6.3 执行层

执行层要解决三个问题：

1. 每条路径要在自己的 `PathSpec` 下独立运行，**不能互相污染**
2. 每条路径都要有**超时和异常处理**，一条路径失败不能拖垮整次并行探索
3. 所有路径完成以后，结果必须**回填到 `ParallelTrace`**，再交给聚合器裁决

```python
RunPath = Callable[[str, PathSpec], Awaitable[PathResult]]


async def _run_one_path(
    question: str,
    spec: PathSpec,
    run_path: RunPath,
) -> PathResult:
    start = time.perf_counter()
    try:
        result = await asyncio.wait_for(
            run_path(question, spec),
            timeout=spec.timeout_ms / 1000,
        )
        result.path_id = spec.path_id
        result.lens = spec.lens
        result.validator = result.validator or spec.validator
        result.latency_ms = int((time.perf_counter() - start) * 1000)
        if result.claim and not result.normalized_claim:
            result.normalized_claim = result.claim.strip().lower()
        return result
    except asyncio.TimeoutError:
        return PathResult(
            path_id=spec.path_id,
            lens=spec.lens,
            status=PathStatus.TIMEOUT,
            error="path_timeout",
            latency_ms=int((time.perf_counter() - start) * 1000),
        )
    except Exception as exc:
        return PathResult(
            path_id=spec.path_id,
            lens=spec.lens,
            status=PathStatus.FAILED,
            error=f"{type(exc).__name__}: {exc}",
            latency_ms=int((time.perf_counter() - start) * 1000),
        )


async def parallel_explore(
    trace_id: str,
    task_id: str,
    question: str,
    path_specs: list[PathSpec],
    run_path: RunPath,
    policy: AggregationPolicy,
    route_decision_id: str | None = None,
) -> ParallelTrace:
    tasks = [
        asyncio.create_task(_run_one_path(question, spec, run_path))
        for spec in path_specs
    ]
    results = await asyncio.gather(*tasks)
    trace = ParallelTrace(
        trace_id=trace_id,
        task_id=task_id,
        route_decision_id=route_decision_id,
        path_specs=path_specs,
        path_results=results,
        policy=policy,
    )
    trace.aggregation_decision = trace.aggregate()
    return trace
```

> 这段代码的灵魂不是 `asyncio.gather`。gather 只是让路径真的并发，**生产级并行探索真正要解决的，是路径定义、证据验证、失败路径、分歧裁决和 Trace 留痕**。真正的可靠性来自前面的 `PathSpec`、`PathResult`、验证器状态和聚合策略。

---

## 七、案例：小雪这笔奖金完整跑一遍

### 7.1 分诊台 RouteDecision

```yaml
lane: structured_reasoning
reasoning_mode: PARALLEL
max_paths: 3
human_gate: before_write
```

→ 任务可以做结构化判断，但**不能自动写入薪资系统**。并行探索只是产出建议和证据，最终写入还要过人审或业务闸门。

### 7.2 三条路径

| 路径 | 口径 | 金额 | 证据 |
|------|------|------|------|
| **1** | 旧版有效期口径 | 9,600 元 | 旧版奖金政策第 4.2 条 + 考核周期记录 |
| **2** | 新版发放时点口径 | 8,200 元 | 新版政策里的发放时点条款 + 发放批次记录 |
| **3** | 审批日期切换口径 | 9,600 元 | 审批单日期 + 两条历史同类案例 |

### 7.3 聚合

```text
9,600：2 条路径
8,200：1 条路径
```

**如果是只读分析**，可以给出"多数口径支持 9,600，但存在新版发放时点口径异议"的结论，并把异议原样写进 Trace。

**但如果下一步要写入薪资系统**，就不能因为 2:1 多数直接自动改金额。这里要回到 `RouteDecision`：`human_gate = before_write`。系统可以把建议金额、异议口径和证据链交给 HR 或薪酬管理员确认，**但不能绕过写入闸门**。

> **这就是 `lane` 和 `reasoning_mode` 分开的价值**。PARALLEL 让判断更稳；lane 决定能不能动手。

### 7.4 与单链对比

| 单跑一条链的风险 | 并行探索的优势 |
|---------------|--------------|
| 链的第一步就要选口径。假设选了新版口径，后面每步都严密、有证据，会**非常自信地放行 8,200 元** | 把**口径本身存在分歧**这件事，**摆到了台面上**来，供下游（人或 Agent）进一步判断 |
| 错误在轨迹上**完全看不出来**，因为整条链形式上无懈可击，**错的只是最开始那个没有人确认过的口径选择** | 异议路径被原样记录，作为审计事件和决策依据 |

---

## 八、三个口径都合法：年终奖跨月案例

> 如果更难办的情况呢：公司的年终奖跨 5、6 两个月分两次发放，那么**社保缴费基数和个税应该算到哪个月**？

这道题至少有**三个口径**：

| 口径 | 规则 |
|------|------|
| **制度口径** | 按实际发放月归集，钱哪个月到账就算哪个月 |
| **税法口径** | 按所属期摊分，这笔奖金对应哪个考核期，就摊到哪个期间 |
| **地方社保口径** | 对跨月发放的奖金基数又另有专门规定 |

**三个口径背后是三套并行有效的法规体系**。也就是说，这道题**根本不存在唯一正确的答案**，只存在这家公司这一次按哪个口径入账的选择。三条路径跑完，很可能是三个月份、三个基数，各有各的法规支撑，很难取舍。

> **正确的动作是把三个口径连同各自的法规依据摊开，转给 HR 和财务去敲定**。表面上看，这像是 Agent 没能给出答案，实际上它是在如实报告：**这件事的不确定性是真实存在的，不应该由一次采样投票来假装消化掉**。

---

## 九、聚合分级的三个出口

```text
三条线路结果一致         → 自动放行
两条一致一条不同         → 弱通过并留痕检查
三条线路各说各话         → 转人工审核
                              ↑
              因为分歧本身就是该转人审的证据
```

---

## 十、并行探索的指标

并行探索这个模式对于生产指标来说是**一抬一耗**：

| 维度 | 影响 |
|------|------|
| **抬高** | **验证通过率**。几个口径同时摆开后，系统要么择优选出证据最足的那条，要么把分歧顶到人审，无论走哪条路，最终落库的结论质量都更高 |
| **消耗** | **单位验证成功成本**。因为每一次单独推理都消耗 token，有几条链路，就多几倍的 token |

这一抬一耗，决定了**并行探索不应该是默认开关**，而应该由 [[lession20-复杂度路由\|分诊台]] 挑出来在适当情景下单独使用。客服分类这种又便宜又有标准答案的任务开并行何必呢。**值不值得付这 N 倍的 token，需要看这类判断错一次的代价有多大**。

另外，**分歧率本身是一个值得盯的观测信号**：

> 如果某类任务的分歧率突然升高，往往说明这类业务的口径正在变得不稳定，比如**新政策刚刚上线、新旧规则正处在交接期**。因此并行探索也是**业务问题的一个预警器**。

---

## 十一、总结

并行探索，是对同一个问题展开多条独立的推理路径，再通过**聚合、验证器和分歧裁决**把它们收成一个更稳的结论。

### 11.1 六个反模式（避坑指南）

| # | 反模式 | 后果 |
|---|--------|------|
| 1 | 把同一个 prompt 多采样几次，就以为有了多视角 | 只是简单采样多样性，**并不是业务口径多样性** |
| 2 | 永远强行投出一个赢家 | 聚合器要可以接受 `verdict = None`，**不要把真实分歧包装成伪确定性** |
| 3 | 模型自己的置信度替代验证器 | 高风险任务需要单独的验证器进行校准 |
| 4 | 不记录异议路径 | 需要把多个视角的分析原因如实记录 |
| 5 | 并行探索绕过分诊台 | **不要所有的问题都并行探索**，没有必要 |
| 6 | 口径没有注册表 | 探索口径靠模型现场编，今天一个说法，明天另一个说法，**不可审计，也不可复用** |

> **并行探索的结论中，不要害怕分歧的出现**，因为我们用多了 Claude Code / Codex 这些 Agent 的人都知道，它们通常都会装得很确定，**并行的探索应该让它们更诚实地暴露不确定性**。

---

## 十二、思考题

1. 你的 Agent 现在对高风险判断是**单跑一次就出结论**，还是会跑几条路径交叉验证？如果跑多条，遇到分歧的时候如何处理？
2. 挑一类你系统里"**跑两次结果可能不一样**"的判断，哪种情况用采样投票就够了，哪种情况值得建口径注册表？
3. 你现在有没有把多路径分歧当成一个**可观测信号**记进 trace？如果某类任务的分歧率最近悄悄升高了，你能不能第一时间发现不稳定的原因？

---

## 十三、下一讲预告

广度诊室处理的是**口径多**的问题，一次铺开看清楚。但是深水区还有另一种病人：**口径不多，根因藏得深**。

那是深水区的另一间诊室，也是推理模块的最后一个模式——**迭代假设验证（Iterative Hypothesis Testing）**。

---

## 精选留言摘录

> **牙叔**：
> 个人还比较困惑的点是，即使前面有了分诊台把不同问题场景路由到了不同执行通道，但依旧很多场景会落到 PARALLEL 策略上。但像"小雪奖金"只是其中一个场景，这个场景的解决路径"取口径、取政策、取金额值、设计验证链、聚合评判"似乎是为奖金场景特殊定制的？需要明确的 LensRegistry 口径注册表，怎么承接不可预测的问题呢？
>
> **作者回复**：并行探索不是必须提前注册所有口径。**LensRegistry 只是成熟业务域里的加速器，不是并行探索的前提**。并行探索真正需要的，是几条相互独立、能够带证据回来的路径。
>
> 路径可以来自三种地方：
> - 第一种是**采样**，同一个问题多跑几条链，用来对抗模型方差
> - 第二种是**通用探索视角**，比如按时间线查、按 source of record 查、按最近变更查、按权限动作查、按反例查
> - 第三种才是**领域注册口径**，比如薪酬、税务、社保这些稳定规则里的正式 lens
>
> 对不可预测的问题，可以临时生成 **Candidate Lens**，用这些候选口径去打开搜索空间、拉取证据、发现分歧。但候选口径只能用于探索，**不能直接用于高风险自动裁决**。只有经过证据验证、人工确认、并沉淀进注册表的 lens，才有资格在后续任务里参与自动聚合。
>
> **咖哥发言**：**已知问题走注册口径，未知问题走候选口径；注册口径可以裁决，候选口径只能探路。探路探多了，才有资格沉淀成新的注册口径。**

> **元气🍣 🇨🇳**：
> 和随机森林的剪枝算法及投票机制有什么区别？
>
> **作者回复**：相似点是都有多路候选和聚合，差别在于**随机森林的基学习器和投票规则训练后相对固定；Agent 路径会动态选工具、改变状态、消耗不同成本，而且错误高度相关**。因此还要额外管理独立性、证据、预算和副作用。

> **街角·陌路△**：
> 我感觉它有点像主 Agent 同时给多个子 Agent 派活。每一条并行路径背后，其实都还是一个具有一定自主性的 Agent。这样一来，我们之前对 Agent 提到的那些要求，似乎同样也适用于这些并行路径，比如要追踪它们的执行进度，需要的时候主动检索信息，以及尽量把判断所依据的证据保留下来，而不只是让几个 Agent 各自输出一个答案。
>
> 我目前会更倾向于把并行探索理解成一种类似交叉验证的方式。它或许可以通过不同路径之间的相互印证和分歧，帮助我们判断一个结果到底有多可信，但**多个 Agent 得到的结果，也不一定天然就优于单个 Agent**。虽然常说"三个臭皮匠顶个诸葛亮"，但也可能三个臭皮匠沿着同一个错误方向越走越远，最后还不如其中一个臭皮匠。所以我觉得，**并行探索可能不只是在增加 Agent 的数量，更重要的还是这些路径能不能相对独立地展开，并把各自的证据和分歧保留下来**，让 Agent 最终判断和执行所依赖的证据链更加充分。
>
> **作者回复**：理解得很准确。**并行探索的价值来自路径差异、证据独立和分歧可见，数量本身不构成可靠性**。每条路径仍要有进度、证据和资源边界，最终报告也应保留少数意见。

> **有学识的兔子**：
> 1. 我会使用多个不同的 model，来跑同一个高风险判断，并人工审查结果的对比情况；我觉得也算是另一种的多路径的形式。
>
> 挑一类你系统里"跑两次结果可能不一样"的判断，哪种情况用采样投票就够了，哪种情况值得建口径注册表。
> - 采样：适合问题简单，有标准答案，且结果对系统影响小
> - 相对复杂的任务，需要不同的视角来佐证结果是否可靠
>
> 你现在有没有把多路径分歧当成一个可观测信号记进 trace？如果某类任务的分歧率最近悄悄升高了，你能不能第一时间发现不稳定的原因？
> - 没有作为 trace，而是 publish 出来，让用户知道它们的分歧点
> - 不知道，因此 trace 还是很重要的，这样就可以记录过程，查看是哪个环节开始偏离
>
> **作者回复**：你的做法成立，**但多模型不天然等于独立**。低风险、答案同口径时可以投票；高风险或证据不一致时要保留分歧，交给外部事实、独立裁决器或人工。分歧率、证据来源和首次偏离步骤都值得进 trace。

---

## 参考资料

- Francis Galton. *Vox Populi*. Nature, 1907.
- Wang et al. *Self-Consistency Improves Chain of Thought Reasoning in Language Models*. arXiv:2203.11171, ICLR 2023（采样投票）.
- Yao et al. *Tree of Thoughts: Deliberate Problem Solving with Large Language Models*. arXiv:2305.10601, NeurIPS 2023（结构化搜索）.
- ParaThinker: *Native Parallel Thinking*. arXiv:2509.04475, 2025-09（模型原生并行思考）.
- GenPRM: *Generative Process Reward Models*. arXiv:2504.00891, 2025（验证器引导的 best-of-N 聚合）.