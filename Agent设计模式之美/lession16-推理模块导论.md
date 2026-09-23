---
title: "16｜推理模块导论：让 Agent 想得清楚，也想得起来"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/993323"
tags:
  - "#Agent/推理"
  - "#设计模式/导论"
  - "#推理/契约"
  - "#推理/预算"
  - "#推理/追踪"
audio: true
duration: "14:21"
publish_date: "2026-07-02"
course_progress: "16/43"
pattern_position: "推理模式组·导论（第 1 个）"
related:
  - "[[lession11-记忆模块导论]]"
  - "[[lession15-失败日记]]"
  - "[[lession17-思维链]]"
  - "[[lession18-复杂度路由]]"
  - "[[lession19-并行探索]]"
  - "[[lession20-迭代假设验证]]"
---

# 16｜推理模块导论：让 Agent 想得清楚，也想得起来

> **核心命题**：感知让 Agent 看见此刻，记忆让 Agent 留住过去，**推理要做的则是：面对眼前的信息和过去的经验，接下来该相信什么、该选择什么、该做什么**。推理工程不是一味制造更长的思维链，而是调度一条**最短、充分、可验证**的决策路径。

---

## 一、三大模式组的递进关系

| 模块 | 解决的"动词" | 关心什么 |
|------|------------|---------|
| [[lession6-感知模块导论\|感知]] | 看见 | 此刻发生了什么 |
| [[lession13-记忆模块导论\|记忆]] | 留住 | 过去留下了什么 |
| **推理** | **决策** | **基于这些信息，现在应该相信什么、选择什么、做什么** |

### 1.1 引子：18% 涨薪的故事

Agent 通过渐进发现、RAG 等感知和记忆模式把"咖哥涨薪 18%"的证据取回来了：调薪审批单、奖金计算规则、政策版本、历史工资，每一条都带着出处。

**可证据到手，事情并没有结束**。这 18% 到底是一次正常的年度调薪，还是混进了一次性奖金？是数据重复算了两遍，还是用错了规则版本？该自动放行，该转人工，还是该当场拦截？**这些都是推理要负责的内容。**

---

## 二、推理模式的来龙去脉

### 2.1 从生存经验到三段论

人类的推理，最早不是从公式开始的，而是从生存经验开始的：

- 看到乌云 → 想到下雨
- 发现脚印 → 判断猎物方向
- 把一次次观察总结成"如果……那么……"的经验

后来，亚里士多德把这种思考整理成**三段论**：

```text
大前提：所有 A 都是 B
小前提：C 是 A
结  论：因此 C 是 B
```

### 2.2 从规则到 LLM：两代推理范式

| 时代 | 代表系统 | 核心思路 | 局限 |
|------|---------|---------|------|
| **传统 AI** | Logic Theorist (1956) / GPS (1957) / STRIPS (1971) / MYCIN / Hearsay-II | 规则清楚、过程可解释；输入相同 → 输出相同 | 规则需人工编写，复杂世界维护成本高 |
| **LLM 时代** | o1 / DeepSeek-R1 / GPT-5 | 概率生成；权重里压进模式与启发式 | 概率性、不透明性、不稳定性 |

```python
# 程序推理：规则即代码
if raise_ratio > 0.15 and not has_approval:
    block_payment()
```

> 给 LLM 多写几条业务规则，并不能自动得到一个可靠的推理系统。

### 2.3 必须分清的三个东西

| 类别 | 含义 | 是否可审计 |
|------|------|-----------|
| **模型内部推理** | 发生在模型内部，可能很长 | 通常不暴露 |
| **外部慎思过程** | 系统组织出来的：提出假设、查工具、比较证据、调验证器 | 可观察 |
| **决策记录** | 工程师审计用：看过哪些证据、执行过哪些动作、为何升级 | 可追溯 |

**这三者不能混为一谈。** 不能把一段看起来有条理的自然语言，直接当成模型真实计算过程的证明。早期 OpenAI 的推理模型特意不向用户暴露原始思维链；Anthropic 的研究也发现，**模型生成的 CoT 并不总是忠实披露实际影响答案的信息**。

### 2.4 缝合两个时代

> 今天的 Agent 推理工程，本质上是在缝合这两个时代的方案：保留 LLM 的表达能力和泛化能力，同时把传统 AI 里的状态、目标、证据、搜索、验证和停止条件重新装回来。
>
> **LLM 给了我们一个很会猜的推理器，Harness 工程系统要做的，是给这个推理器装上边界、仪表盘和刹车系统。**

---

## 三、推理不等于把思维链写得更长

### 3.1 2022 vs 2026 的回答

| 时期 | 关心问题 | 标准做法 |
|------|---------|---------|
| **2022** | 怎么让大模型推理？ | Chain-of-Thought：在 prompt 里加一句"Let's think step by step" |
| **2026** | 怎么让 Agent **判断**如何推理？ | reasoning_effort、GPT-5 实时 router、adaptive thinking |

> 推理从模型的一种能力，进化成了一种需要被**分配、约束、验证以及停止**的计算资源。

### 3.2 两股改写力量

- **训练侧**：o1、DeepSeek-R1 靠 RL 把"会想"压进权重，涌现出长链推理与自我验证
- **产品侧**：reasoning_effort / adaptive thinking 把"想多深"变成调用时设定的参数

### 3.3 新的核心问题

过去侧重"怎么让 Agent 多想几步"，现在的难点是：

- 这道题**值不值得深想**？
- 该沿一条路想到底，还是**同时试几条**？
- 给它多少 token、多少时间、多少次工具调用？
- 什么时候证据够了**该停**？
- 推不出来的时候，什么时候**升级**、什么时候**交给人**？

---

## 四、四种推理场景案例

6 月的薪酬结算中出现了 4 个具体问题，需要采用四种不同的推理策略：

| 问题 | 性质 | 推理策略 |
|------|------|---------|
| **Q1**: 小冰这个月几号发薪？ | 查日历和发薪规则 | **直接回答**（不值得长推理） |
| **Q2**: 咖哥应发比上月高 18%，该不该自动放行？ | 拆工资组成、核审批、核奖金规则、生效时间 | **链式分解 CoT** |
| **Q3**: 小雪同时命中两版奖金政策，该按哪版算？ | 多个都说得通的口径，比较证据 | **并行探索** |
| **Q4**: 月底总账差了 37 万，缺口从哪来？ | 一次推理不够，提假设、查数据、排除、缩小 | **迭代假设验证** |

> **要建的，不是一条无限延长的思维链，而是一座推理调度台。**
>
> 一个判断进来，先判断它需不需要深想；需要再决定走哪条计算路径、投入多少预算、由谁验证、在什么条件下停止。如果证据不足会继续查，如果成本失控要知道换路。

---

## 五、思考快与慢 + 推理契约

### 5.1 System 1 / System 2

最简单的推理调度是 Kahneman 的快慢思考系统：

| 系统 | 特点 | 适用 | 工具 |
|------|------|------|------|
| **System 1** | 快、自动、低耗 | 熟悉、直接的问题 | 直接生成、小模型、缓存、规则、简单工具查询 |
| **System 2** | 慢、刻意、高耗 | 多步计算、证据比较、复杂决策 | 结构化分解、并行搜索、反复验证、更强模型 |

但这只能告诉我们思考"有快有慢"，还不够全面。

### 5.2 推理契约（Reasoning Contract）

设计生产系统的推理模块，要考虑下面**五个问题**：

```text
1. 是否启动深入思考？    → 这个请求直接回答是否已经足够？
2. 采用哪种推理拓扑？    → 沿一条链、并行尝试，还是循环调查？
3. 投入多少预算？        → token、时间、模型调用、工具调用的上限
4. 由什么验证？          → 单元测试、业务规则、外部数据、人审、还是另一个模型？
5. 什么时候停止？        → 达到什么证据标准后输出，什么情况后升级或放弃？
```

这五项合在一起，称为一次任务的**推理契约**。同一个任务，现在有一整排旋钮可拧：让它走更长的推理路径、跑多条候选再投票、用验证器筛选、根据中间观察修订计划、之后再决定要不要换一个更强的模型。

### 5.3 计算预算控制

> 推理时投入的计算（test-time compute）其实是可以被显式管理的维度。我们要根据不同请求**自适应分配计算**，通过一个**预算包**控制整体思考量。

```python
from dataclasses import dataclass


@dataclass
class ReasoningBudget:
    max_thinking_tokens: int = 8000    # 内部思考 token 上限
    max_latency_ms: int = 12000        # 端到端延迟上限
    max_model_calls: int = 4           # 模型调用次数上限
    max_tool_calls: int = 8            # 工具调用次数上限
    max_parallel_paths: int = 3        # 并行路径数上限
```

> **当答案已经稳定之后，就别再继续消耗推理所用的 Token 数量。**

---

## 六、四种推理模式：分诊台 + 三个诊室

本模块中的四种推理模式对应四种不同的计算拓扑，学习顺序是"**基础、路由、广度、深度**"。不过，**生产运行时要先路由**。

| 模式 | 角色 | 解决问题 |
|------|------|---------|
| **CoT 思维链** | 诊室 1 | 要想清楚、能对账 |
| **复杂度路由** | 分诊台 | 一个判断走快路还是进深想，进了深想又归哪个诊室 |
| **并行探索** | 诊室 2 | 要在几个口径之间择优 |
| **迭代假设验证** | 诊室 3 | 要顺着一个根因往深里挖 |

四个模式组成一条**控制链**：

```text
先路由 → 再推理 → 推理之后要验证 → 验证不通过就换路 → 达到停止条件才输出
```

### 6.1 CoT 思维链（[[lession19-思维链\|lession19]]）

把复杂的判断拆解成一条可检查的链。咖哥 18% 的加薪，拆成：

- 基本工资变化、奖金变化、补贴变化
- 政策生效时间、审批状态

中间产物：子问题、计算项、证据引用、判断依据等。

### 6.2 复杂度路由（[[lession20-复杂度路由\|lession20]]）

按任务意图、证据状态、机械状态、动作风险四个信号，在以下车道之间分流：

- 直接回答 / 证据分析 / 结构化推理 / 计划执行 / 人工审批

对成本和推理质量做权衡。

### 6.3 并行探索（[[lession21-并行探索\|lession21]]）

让 Agent **广着想**。当一个问题有多条都说得通的路径时，并行探索同时生成不同的假设、计划、解释，再由验证器比较。

### 6.4 迭代假设验证（[[lession22-迭代假设验证\|lession22]]）

让 Agent **深着查**。有些问题在信息不足时根本不能一次解决。每一次观察，都改变下一轮的假设，让模型在行动获得的新信息上更新计划。

---

## 七、推理追踪：记录决定，不记录脑内独白

### 7.1 为什么需要 Reasoning Trace

我们之前说过感知追踪和记忆追踪，推理模块自然也需要自己的仪表盘——**推理追踪（Reasoning Trace）**。

没有它，推理系统就是个**黑盒**：

- 不知道 Agent 在每个任务上想了多久
- 不知道花了多少钱
- 不知道想得对不对

> 说到推理的追踪，你可能会想到把大模型返回的 `thinking` 字段直接塞进记录。**这是模型生成的全部"内心戏"整体保存**。更好的方法其实是记录**外部可核验的决策事件**。

### 7.2 Python 实现

```python
from dataclasses import dataclass, field
from enum import Enum


class ReasoningMode(str, Enum):
    DIRECT = "direct"         # System 1，直接答
    COT = "cot"               # 链式分解
    PARALLEL = "parallel"     # 并行探索
    ITERATIVE = "iterative"   # 迭代假设验证


@dataclass
class ReasoningStep:
    step_id: int
    hypothesis: str            # 这一步要验证的命题，不是模型的私密思维
    evidence_refs: list[str]   # 这一步实际引用了哪些可追踪证据
    action: str                # 查库 / 运行代码 / 调 API / 询问审批器
    observation: str           # 外部环境返回了什么
    decision: str              # 根据证据做出的局部决定
    confidence: float = 0.0    # 自评置信度，要做 calibration
    model_calls: int = 0
    tool_calls: int = 0
    thinking_tokens: int = 0
    latency_ms: int = 0


@dataclass
class ReasoningTrace:
    task_id: str
    mode: ReasoningMode
    route_reason: str                       # 为什么分到这个 mode
    budget: ReasoningBudget
    steps: list[ReasoningStep] = field(default_factory=list)
    final_decision: str = ""
    validator: str = ""                     # 由什么验证器放行
    validation_passed: bool = False
    stop_reason: str = ""                   # 为什么停
    escalated_from: str = ""                # 从哪个更轻的 mode 升级而来

    @property
    def total_thinking_tokens(self) -> int:
        return sum(s.thinking_tokens for s in self.steps)

    @property
    def total_tool_calls(self) -> int:
        return sum(s.tool_calls for s in self.steps)

    @property
    def total_latency_ms(self) -> int:
        return sum(s.latency_ms for s in self.steps)
```

Trace 一步一记 `hypothesis` / `evidence_refs` / `action` / `observation` / `decision`，形成可审计、可计量的决策流水。

### 7.3 推理健康度四指标

一个推理系统的健康度，归结为四个指标：

| 指标 | 含义 | 解读 |
|------|------|------|
| **验证通过率** | 最终结论通过业务规则、测试、执行结果或人工复核的比例 | 最直观的推理健康指标 |
| **首次路由命中率** | 第一次选的推理模式是否足够解决问题 | 大量直接请求最后升级到迭代 → 系统低估难度；反之 → 过度推理 |
| **单位验证成功成本** | 获得一个通过验证的正确结果平均需要多少成本和延迟 | 便宜模型反复失败、重试、升级，最终可能花费更高 |
| **推理漂移率** | 长任务推进中，目标、约束、已验证事实是否被逐渐遗忘或改写 | 锚点保护的关键 |

> 这四个指标都要按**任务类型**进行分桶。客服分类、代码修复、财务审批、研究调查，这些不同的任务需要建立自己的基线、SLO 和升级规则。

---

## 八、总结

感知解决"发生了什么"。

记忆解决"过去留下了什么"。

推理解决"基于这些信息，现在应该相信什么、选择什么、做什么"。

> **推理工程不是一味制造更长的思维链，而是调度一条最短、充分、可验证的决策路径。它的核心不是让 Agent 想得最多，而是让它用最少的充分计算，得到可验证的正确决定。**

### 8.1 四种模式的经济学选择

| 模式 | 用什么换什么 |
|------|------------|
| **CoT** | 用一定的 token，换取**问题分解和中间产物** |
| **复杂度路由** | 用轻量判断，换取**整体成本和延迟的下降** |
| **并行探索** | 用更多计算路径，换取**对单一路径偏见的抵抗** |
| **迭代假设验证** | 用更多轮次，换取**在未知环境中逐步逼近真相** |

**没有一种方式永远最好**。Agent 设计者真正要做的，是在准确率、成本、延迟、风险和工程复杂度之间，找到适合当前业务的点。**该快时快、该深时深，并且每个重要结论都能回到证据上。**

---

## 九、下一讲预告

下一讲，我们进入第一个具体推理模式：**思维链（Chain-of-Thought）**。

CoT 从 2022 年的提示技巧开始，后来延伸出 self-consistency、Tree of Thoughts 和各种推理时搜索方法。现在我们要回答的是：**一条链应该怎样拆，才不会把错误一路传下去。**

---

## 十、思考题

1. 你的 Agent 现在用什么模型做推理？是不管简单复杂都用同一个，还是按难度路由不同档？如果是固定一个模型，可以试着用三十天的账单估算一下，单用顶配模型和"便宜档加顶配档路由"能差多少钱？
2. 回想一次你的 Agent 明明拿到了足够信息，却还是绕了很远才给出答案的情况。它是真的需要推理，还是只是缺少一个明确的规则或流程模板？
3. 你的长任务 Agent，跑到第十步以后还记得住第一步定下的约束吗？如果不确定，今天就在 trace 里加上 `hypothesis` 和 `evidence_refs`，看看事实和规范是什么时候在后面被悄悄改写的。

---

## 精选留言摘录

> **陈健飞**：
> 这个课程太理论了，有能对应上的实践课程吗？
>
> **作者回复**：这个课程其实是结合案例讲解的理论课。希望后续能够设计出相关的全实操性质的课程。

> **Bug Killer**：
> 像 Claude Code 这种 agent，在运行的时候，怎么知道它使用的是哪种推理模式？
>
> **作者回复**：可以直接问它，它会告诉我们的。一个具体复杂的问题，它完成了之后，问它是怎么推理的，它内部有几种推理模式，都可以问。它的设计可能和我的设计模式有共性，也可能不一样。

> **有学识的兔子**：
> `hypothesis = "我现在主张什么"` `evidence_refs = "凭什么 —— 前面哪步、哪个来源支撑我这么主张"`
>
> 这个理解正确吗？根据 evidence 的引用，来看假定的事情是否与前面的内容不一致，来判定漂移程度？
>
> **作者回复**：理解对了。**漂移不是"假设和前面不一致"，因为假设本来就该随证据变；漂移是"锚点（目标/约束/已验证事实）被悄悄忘了或改了"**。evidence_refs 的作用，是把每个主张钉回出处，让"锚点在第几步被绕开"变得可审计。

> **有学识的兔子**（第二弹）：
> 1. 我现在使用时多个模型 deepseek / minimax / qwen；我之前定义过日常走 minimax 多模态，推理走 deepseek-pro；我没统计过，单从售价来说，同样的 token，便宜 vs 顶配，**费用相差要在 5 倍以上**；
> 2. 遇到过。现在看来，可能是路由没有做好，路由没有命中，进入了深度推理流程里去了。
>
> **作者回复**：需要在路由也就是分诊台那个节点好好做做文章。

> **陈小虎**：
> 这个课更像是教/总结解决问题的方法，太全面了~
>
> **作者回复**：是的兄弟，你说对了，**这是渔而不是🐟**。

> **元气🍣 🇨🇳**：
> 实际任务是很复杂多变的，这种契约如何建立？
>
> **作者回复**：后面咱们推理模块的各个模式，不就把这一套推理的契约详细的建立起来了么。我们往下学习。

> **antipas**：
> 复杂度路由（分诊台），判断后路由给 subagent，进而用不同垂直小模型继续，可否这样？
>
> **作者回复**：可以啊，这是**多 Agent 协作的一种实现形式**。

---

## 参考资料

- Newell, Shaw, Simon. *Report on a General Problem-Solving Program*. IFIP, 1959（GPS，程序成于 1957）.
- Fikes, Nilsson. *STRIPS: A New Approach to the Application of Theorem Proving to Problem Solving*. 1971.
- Shortliffe. *MYCIN: Computer-Based Medical Consultations*. 1976（专家系统 / 产生式规则）.
- Daniel Kahneman. *Thinking, Fast and Slow*. 2011（System 1 / System 2）.
- Wei et al. *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. arXiv:2201.11903, NeurIPS 2022.
- Wang et al. *Self-Consistency Improves Chain of Thought Reasoning*. arXiv:2203.11171, ICLR 2023.
- Yao et al. *Tree of Thoughts: Deliberate Problem Solving with Large Language Models*. arXiv:2305.10601, NeurIPS 2023.
- Yao et al. *ReAct: Synergizing Reasoning and Acting in Language Models*. arXiv:2210.03629, ICLR 2023.
- Ong et al. *RouteLLM: Learning to Route LLMs with Preference Data*. arXiv:2406.18665, ICLR 2025.
- DeepSeek-AI. *DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning*. Nature, 2025-09-17.
- OpenAI. *Introducing OpenAI o3 and o4-mini*. 2025-04-16. / *Introducing GPT-5*. 2025-08-07.
- Anthropic. *Adaptive thinking / extended thinking*（Claude API 文档，Opus 4.6–4.8）.
- Anthropic et al. *Inverse Scaling in Test-Time Compute*. arXiv:2507.14417, 2025-07.
- Shojaee, Mirzadeh et al. *The Illusion of Thinking*. arXiv:2506.06941, NeurIPS 2025.
- Anthropic. *Reasoning Models Don't Always Say What They Think*. arXiv:2505.05410, 2025.