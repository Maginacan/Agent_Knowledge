---
title: "32｜层级委派：把大目标拆成可验收的责任"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/1011427"
tags:
  - "#Agent/协作"
  - "#设计模式/层级委派"
  - "#协作/责任拆分"
  - "#协作/Assignment"
  - "#协作/工件闸"
  - "#协作/组合闸"
audio: true
duration: "16:08"
publish_date: "2026-09-01"
course_progress: "32/43"
pattern_position: "协作模式组·第 2 个"
related:
  - "[[lession31-协作模块导论]]"
  - "[[lession26-反思模块导论]]"
  - "[[lession30-自愈循环]]"
  - "[[lession21-行动模块导论]]"
  - "[[lession24-工具调度]]"
---

# 32｜层级委派：把大目标拆成可验收的责任

> **核心命题**：层级委派由一个**稳定责任人保留总目标**，根据当前输入拆出边界明确的子责任，把它们交给**相对隔离的执行者**，再通过**结构化工件与组合验收**，把局部结果收回总目标。
>
> **一句话**：父层守目标，子层担边界，**两道闸**（工件闸 + 组合闸）确认整体完成。
>
> 责任划分的判定方法——**拿掉一项子任务时，最终交付是否明确少掉一块**？如果拿掉谁都没区别，说明责任还没有分清。

---

## 一、释题与开篇问题

把大目标拆成可验收的责任。主管守住父目标，子 Agent 承担边界清楚的局部责任，最后由**工件闸**和**组合闸**确认整个任务真正完成。

上一讲把"多 Agent 框架选型简报"交给三位研究员执行，三人都勤快，但团队产出的是三份相似材料，状态管理与跨 Agent 交接仍然空缺。

Anthropic 复盘多 Agent 研究系统时也提到过类似问题：协调者把"研究半导体短缺"这种宽泛任务交给多个子 Agent，一位回头研究 2021 年汽车芯片危机，另外两位重复追踪当时的供应链。每个 Agent 都完成了一次像样的研究，但整个 Agent 团队却没有交付出一个完整的结果。

**问题出在派工那一刻**——协调者复制了目标，没有拆开责任。

> "我们派出去的是三个人，不是三份责任。"

子 Agent 需要明确的目标、输出格式、工具和来源指引，以及清楚的任务边界；描述过于宽泛时，**重复劳动和覆盖缺口会同时出现**。

层级委派要解决的正是这一类问题：一个大目标交给多位执行者后，怎样保证每人承担一块**可区分、可交付、可验收**的责任，最后再由一个稳定责任人确认整体目标已经完成。

---

## 二、先跑一遍 Demo

- 配套实验：`collaboration/light_labs/editorial_delegation_lab.py`
- 配套工作台：`collaboration/light_labs/web_app.py`

把"多 Agent 协作简报"问题复现一下。要求覆盖三个方向：

- **topology**：团队由谁协调，任务怎样分发
- **state**：Agent 的上下文和状态怎样隔离与保存
- **handoff**：结果怎样跨 Agent 或跨系统交接

第一轮不划分责任，先给三位研究员完全相同的任务：`Research multi-agent collaboration`。

```bash
python3 collaboration/light_labs/editorial_delegation_lab.py
```

第一段结果：

```text
== vague delegation ==
atlas  lane=topology source=CrewAI
birch  lane=topology source=CrewAI
comet  lane=topology source=CrewAI
coverage=1/3 duplicates=2 admitted=false
missing=state,handoff
```

三位研究员都返回了资料卡，每张卡也都带有来源。从单个 Worker 角度看，三次调用都成功结束。

但这里出现了层级系统中两种不同的"完成"：

- **Worker completion**：某位执行者已经返回结果。
- **Goal completion**：所有被要求的责任已经形成可接受的组合结果。

如果运行时只记录第一种完成，三位研究员的状态都会是 `SUCCEEDED`，从编审的角度看，整份简报却无法发布。**因此，不能把"所有子调用都结束"直接等同于"父目标已经完成"**。

用网页版工作台再跑一次，对比层级委派前后变化：

```bash
git clone https://github.com/huangjia2019/agent-design-patterns.git
cd agent-design-patterns
python3 collaboration/light_labs/web_app.py
```

终端出现 `Collaboration Mini Lab: http://127.0.0.1:8098` 后，在浏览器打开。左侧选择"32 层级委派"，点击"运行对照实验"。工作台左边跑模糊委派，右边跑责任委派，底部把程序实际做过的三步检查展开。

这组对照是一项**消融实验**：保留 Worker、资料卡、来源和组合闸，只撤掉责任划分。责任划分消失后，重复与缺口同时出现；重新加入责任面后，覆盖恢复，组合闸才允许结果继续流转。

- 左侧：覆盖 1/3，重复 2，组合闸不通过。
- 右侧：覆盖 3/3，重复 0，验收器允许通过。

两次运行过程中模型和资料没有变化，**改变发生在协调接口**。

---

## 三、层级委派起作用的核心机制

如果只是在图上画一个 manager，再从它下面拉几条线连到 worker，这些线本身并不代表它拆出了三份不同责任；而三个 worker 都返回成功，也不代表总目标获得了完整覆盖；主 Agent 最后把三段文本拼在一起，更不代表它完成了组合验收。

层级委派位于双轴框架的**协作行 × 层级列**。

### 工程化定义

> **层级委派**由一个稳定责任人保留总目标，根据当前输入拆出边界明确的子责任，把它们交给相对隔离的执行者，再通过结构化工件与组合验收，把局部结果收回总目标。

这条定义包含四项内容：

1. **主管保留总目标**。他可以把 state 交给 birch，自己仍然要记得最终交付包含 topology、state 和 handoff 三部分。别觉得分工出去了，任务就结束了。
2. **子任务严格根据当前目标生成**。研究 Harness 时按拓扑、状态和交接拆分；处理上百份合同时按合同分片；调查线上事故时按时间线、代码变更、基础设施和用户影响拆分。子 Agent 一般**不应该在运行时动态拆解目标**。
3. **Worker 只承担自己的局部责任**。每个子 Agent 只需要得到足够完成当前任务的上下文、工具和预算，并不需要继承主管的全部会话历史。
4. **主管验收组合结果**。每位 Worker 的完成是局部完成，整体父目标是否完成，要看所有局部工件放在一起后是否覆盖总目标、是否存在冲突、是否违反全局约束。

### 责任的层级

层级委派中的"层级"主要是一种**责任层级**：

- **父层**：保留目标、拆分责任、处理缺口、验收整体。
- **子层**：执行局部责任、形成工件、报告失败与证据。

主管可以由模型担任，也可以由确定性代码、工作流引擎或人机混合系统承担，关键在于**总目标始终有一个明确的持有者**。

---

## 四、层级委派的必要性

### 三个收益

1. **注意力覆盖**。一位 Agent 要同时记住总目标、搜索资料、核对来源和整理结构，很容易被最显眼的线索吸住。把责任拆开，每位 Worker 可以在较小的上下文里把一个方向做深，主管继续守住全局缺口。
2. **失败隔离**。状态研究超时，不必把拓扑和交接两份已经完成的资料一起丢掉。主管知道缺的是哪一块，可以只重派那项责任。
3. **替换执行者的自由**。责任契约稳定以后，某一块可以交给进程内子 Agent，也可以换成 Codex、Claude Code 或外部服务。系统可以任意切换工作方式或 Agent 的具体实现，但**总目标与验收口径（业务规则）已经固化下来**。

要获得这些收益的前提是：任务一定要有**真实可拆的责任面**，否则多 Agent 带来的协调成本往往高于收益。

---

## 五、层级委派的实现细节

### 1. 责任的划分

实验中的最小任务对象叫 `Assignment`：

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Assignment:
    """A bounded responsibility handed from the editor to one researcher."""
    worker_id: str
    lane: Lane | None
    question: str
    deliverable: str = "one claim card with a source"
```

任务对象记录四件事：

- 谁负责（`worker_id`）
- 责任面是什么（`lane`）
- 本次要回答哪个问题（`question`）
- 交回什么（`deliverable`）

在生产系统中还需要输入引用、工具范围、权限、预算、超时和目标版本等信息。

把任务改成责任分明：

```python
Assignment("atlas", "topology", "Who coordinates the team and dispatches work?")
Assignment("birch", "state",    "How are agent context and state isolated?")
Assignment("comet", "handoff",  "How do results move across agent boundaries?")
```

第二段输出随之变成：

```text
== scoped delegation ==
atlas  lane=topology source=CrewAI
birch  lane=state    source=Claude Managed Agents
comet  lane=handoff  source=A2A v1.0
coverage=3/3 duplicates=0 admitted=true
```

> **责任面是否形成的判定**：拿掉某个子任务时，最终交付是否明确少掉一块？如果答案明确，说明责任边界通常已经成形。如果拿掉谁都没区别，说明还没有分清责任。

### 2. 把结果转成工件

子 Agent 中间可以搜索十次、推翻两次假设、读几十页文档。主管如果把这些过程全部收回来，他自己要消化的上下文就太多了。层级委派要求 Worker **压缩过程，交回一份能验收的小工件**。

实验里的 `ResearchCard` 设计：

```python
@dataclass(frozen=True)
class ResearchCard:
    """The small artifact returned by a researcher."""
    worker_id: str
    lane: Lane
    claim: str
    source_id: str
    source_url: str
```

- `claim` 是结论
- `source_id` 和 `source_url` 把结论连回来源
- `lane` 说明它履行了哪一块责任

主管接收这张卡，不需要接管研究员的整段思考过程。

**工件压缩并不意味着过程必须全部丢弃**。完整轨迹仍然可以留在子会话、Trace 或工件存储中，主管默认只接收压缩结果，需要调查时再沿引用展开。Anthropic 在多 Agent 研究系统的复盘中也提到，子 Agent 可以把完整结果直接写入外部工件系统，再把轻量引用交给协调者，减少多轮转述造成的信息损失，也避免大段结果在主会话中反复复制。

不过，`lane=state` 仍然只是 Worker 的自报。系统不能因为模型填对了字段，就认定它已经完成状态管理研究。**生产验收还要把工件与原始 Assignment、来源内容、目标版本和外部规则对照**。模型可以提交履约声明，但不能单凭自己的声明完成验收。

### 3. 两道闸门

最后一个值得记住的工程细节，是通过两道闸门把**局部验收**与**组合验收**分开。

**第一道：工件闸**。它看一张资料卡有没有约定字段、来源、正确的任务版本与生产者。以发薪场景为例，它还会检查这批工件是否覆盖契约指定的员工，金额和输入指纹是否一致。

**第二道：组合闸**。它把所有资料卡摊开，检查三个责任面是否各出现一次。在薪酬系统里，它还要检查五个部门加起来是否覆盖 800 人、是否存在重复员工、月度总额是否超过资金线。

`review_cards()` 是组合闸的简单实现：

```python
counts = Counter(card.lane for card in cards)
missing = tuple(lane for lane in REQUIRED_LANES if counts[lane] == 0)
duplicate_count = sum(max(0, count - 1) for count in counts.values())
sources_present = all(card.source_id and card.source_url for card in cards)
accepted = not missing and duplicate_count == 0 and sources_present
```

**为什么不能只设第一道闸？** 开场的三张卡都能通过单卡检查，整份简报仍缺两块。

**为什么不能只设第二道闸？** 三个方向虽然齐全，但其中一张卡可能没有来源，或者属于旧版本目标。

两道闸分别回答：

- **这一个责任单元能不能接收？**
- **所有责任单元合起来，总目标能不能接收？**

这个问题并不只属于 Agent：

- 微服务场景：每个微服务都满足自己的延迟预算，整条调用链仍可能超过端到端 SLA。
- 薪酬场景：每个部门的付款批次都通过局部检查，所有批次合起来仍可能出现重复员工。
- 测试场景：每位测试工程师都完成了自己的用例，整次发布仍可能漏掉关键业务路径。

> **局部正确不会自动推出整体正确。**

---

## 六、层级委派 vs 子 Agent 隔离

层级委派解决**责任怎样拆分、交付与验收**。子 Agent 隔离解决**局部执行怎样形成真正的运行边界**。两者经常一起出现，却不是同一个模式。

| 隔离面 | 在研究简报中的含义 | 生产做法 |
| --- | --- | --- |
| 上下文 | 每位研究员只获得自己的问题和必要背景 | 新会话、最小输入引用、压缩工件 |
| 工具 | 研究员可以搜索，不能直接发布官网 | 工具白名单、沙箱、只读数据源 |
| 预算 | 一项研究不能吃掉整组 token 和时间 | 单任务超时、调用上限、父子预算 |
| 失败 | 一位研究员失败，不拖垮其他已完成结果 | failure artifact、局部重派、保留已接收工件 |
| 递归 | Worker 不能无限继续创建 Worker | 最大委派深度、并发上限、子任务总数 |

- **上下文隔离 ≠ 权限隔离**。子 Agent 开启了新会话，却仍继承付款工具和生产凭证，风险并没有被真正缩小。
- **工具约束的可靠限制要在运行时执行**：工具不出现在子 Agent 的可见集合中，确保它绝不会被没有授权的子 Agent 调用。
- **设置最大委派深度**是为了防止责任链无限展开。父 Agent 创建三个 Worker，每个 Worker 再创建十个下属，系统很快会同时失去预算、覆盖和追踪能力。

> **层级委派定义责任树，子 Agent 隔离定义每条边能够传递什么。**

---

## 七、什么时候适合使用层级委派模式

### 适合

存在清楚责任面的任务：

- **开放研究**：按地区、公司、主题或资料源拆分，主管负责统一口径。
- **合同 / 简历 / 工单**：按输入主键分片，每片独立交付。
- **安全 / 财务 / 法务**：给不同 Worker 配置专门工具与数据权限。
- **长任务**：把大段探索过程留在子上下文，只把压缩工件送回主管。

### 谨慎

1. **任务很短**，一个 Agent 的上下文和工具已经足够——增加 manager、Worker 和验收层只会增加通信开销。
2. **步骤之间存在密集的隐含依赖**——每位 Worker 都必须知道上一位执行者刚刚做出的临场决定，交接成本可能高于上下文隔离的收益。
3. **多位 Agent 同时修改同一份状态**，却没有独立工作区、合并规则和全局验证——责任虽然已经拆开，但写入时仍然会互相覆盖，容易造成混乱。
4. **团队只能描述角色，不能定义交付**——诸如"你是高级研究员""你负责深入思考"这样的提示词并没有说明交回什么，也无法形成验收。
5. **父 Agent 既负责拆解，又只凭自己的感觉宣布拆解完整**——在开放性问题中，主管可能遗漏自己没有想到的责任面。**能编码的覆盖要求应进入确定性规则**；无法预先枚举的部分需要独立评审、外部清单或人工验收。

---

## 八、小结

层级委派先由稳定责任人保留父目标，再把目标拆成带有边界的 `Assignment`。Worker 在隔离环境中完成局部责任，用结构化工件交回结论与证据。**工件闸**判断单项结果能不能接收，**组合闸**判断所有结果合起来能不能完成父目标。

本讲实验中，模糊委派产生三张合法资料卡，却只覆盖一个责任面。把任务拆成 topology、state 和 handoff 后，覆盖率从 1/3 变成 3/3，重复数从 2 变成 0，组合结果才获得接收资格。

DeepSeek Harness、CrewAI、OpenAI Agents SDK 和 Claude Managed Agents 这些主流 Agent 框架都可以提供不同的子 Agent 运行方式。它们负责启动、会话、工具、控制权和生命周期，**责任是否拆全、工件是否合格、父目标是否完成，仍然要由模式契约和业务验收——也就是通过我们的 Harness 系统设计来回答**。

---

## 九、思考题

1. 你现在的子 Agent 清单里，**删掉哪一个不会让最终交付少掉任何明确内容**？
2. 你的主管只检查每位 Worker 是否完成，还是会检查所有结果合起来是否覆盖总目标？
3. 如果把进程内子 Agent 换成外部 Codex 或 Claude Code，**哪些责任契约应保持不变，哪些运行边界必须重新验证**？

---

## 十、下一讲预告

下一讲进入**扇出聚合**。层级委派先解决"每个人负责哪一块"。当多路结果一起回来后，新的问题会出现：有的结果缺席，有的重复，有的互相矛盾，还有的使用了不同版本的事实。

系统需要决定怎样等待、去重、保留溯源、暴露冲突，并在多数、高分和外部事实之间选择裁决依据。

---

## 十一、参考资料

1. Anthropic, *How we built our multi-agent research system*, 2025-06-13，包含模糊委派导致重复研究的实际复盘。
2. ADPS, *C1 Hierarchical Delegation*，动态拆分、隔离执行与中心化综合。
3. ADPS, *C5 Sub-Agent Isolation*，上下文、工具与失败边界。
4. CrewAI Documentation，Crew、Flow 与层级流程。
5. OpenAI Agents SDK: Multi-agent orchestration，agents-as-tools 与 handoff。
6. Claude Managed Agents: Multiagent orchestration，隔离 session 与版本化团队名单。
7. DeepSeek Harness: Subagent subsystem，统一子代理接缝与 provider 能力。
8. DeepSeek Harness README，Developer Preview、插件架构与源码运行方式。