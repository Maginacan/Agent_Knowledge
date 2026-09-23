---
title: "20｜迭代假设验证：用科学方法猜至证据的收敛"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/997381"
tags:
  - "#Agent/推理"
  - "#设计模式/迭代假设验证"
  - "#推理/科学方法"
  - "#推理/证伪"
  - "#推理/LockedFacts"
audio: true
duration: "29:30"
publish_date: "2026-07-16"
course_progress: "20/43"
pattern_position: "推理模式组·第 5 个（收官）"
related:
  - "[[lession16-推理模块导论]]"
  - "[[lession17-思维链]]"
  - "[[lession18-复杂度路由]]"
  - "[[lession19-并行探索]]"
  - "[[lession21-行动模块导论]]"
---

# 20｜迭代假设验证：用科学方法猜至证据的收敛

> **核心命题**：迭代假设验证，是让 Agent 把复杂问题拆成一串**可证伪的假设**，并通过多轮外部证据验证，把一次性猜测变成**可停机的自收敛逼近过程**。
>
> 这句话里的关键词不是"多轮"，而是**可证伪、外部证据、可停机**。

---

## 一、迭代假设在推理契约中的位置

在 [[lession4-双轴框架（下）\|双轴图谱]] 里，迭代假设验证落在**推理 × 循环**的交点：

- **属于推理**：因为它要解决的是基于当前证据，哪个解释更可信
- **属于循环**：因为它不是一次性生成答案，而是反复走一条闭环

```text
提出假设 → 设计验证 → 观察证据 → 修正判断 → 进入下一轮
```

### 1.1 与普通多轮对话的区别

> **普通多轮对话可能只是模型换个说法继续解释**。第一轮说"可能是奖金"，第二轮说"进一步看也可能是奖金"，第三轮又补充一段更完整的奖金分析。表面上它在迭代，实际上**没有新证据进入，也没有任何假设被推翻**。这样的循环，只是在旧想法上继续加厚。

迭代假设验证不一样。它要求每一轮都带着一个**可以被推翻的命题**进入真实环境：

- 系统不光问"有什么证据支持我"
- 还要问**"什么证据能证明我错了"**

| 情形 | 处理 |
|------|------|
| 证据推翻了当前假设 | **换方向** |
| 证据只解释了一部分 | 把已解释的部分**锁住**，再追查剩余缺口 |
| 连续几轮都没有新增解释 | **停止自动迭代**，把半成品交给人 |

> **迭代假设验证强调的是让 Agent 一轮轮被证据纠偏，直到收敛或交人审。**

---

## 二、这其实就是科学方法的演进

| 时代 | 关键贡献 | 核心思想 |
|------|---------|---------|
| 1620 | **培根《Novum Organum》** | 归纳法：先观察和证据，再推演道理 |
| 1739-1740 | **休谟《人类理解研究》** | 归纳困境：再多正面例子也不能证明假设必然成立 |
| 18世纪 | **贝叶斯** | 证据更新信念：让新证据持续修正原有判断 |
| 1934 | **波普尔《研究的逻辑》** | 证伪原则：主动寻找能推翻假设的反例 |
| 1948 | **维纳《控制论》** | 反馈闭环：不断测量偏差，调整下一步行动 |
| 2022 | **ReAct** | 推理（Thought）与行动（Action）交错，环境返回 Observation 喂回下一步 |
| 2023 | **Self-Refine** | 模型先给一版答案，再自我批评修订 |
| 2023 | **Reflexion** | 用"言语强化学习"把每次失败的教训写成文字，带进下一轮 |

> **主线**：人类和大模型都学会了遇到复杂情况别猜，而是**提假设、设计实验、用证据证伪、再修正**，这就是科学方法。
>
> **迭代假设验证，是把这套方法搬进了 Agent 系统设计。**

---

## 三、真实的工业故障诊断场景

一家化工厂有一套故障诊断 Agent，接在 PLC / SCADA 系统后面。当一线工程师收到报警时，Agent 不直接给结论，而是生成一棵**假设树（hypothesis tree）**：可能根因是什么，每个假设应该查什么证据，证据回来以后这个假设是被证伪、被确认、还是需要继续下钻。

### 3.1 凌晨 3:47 报警

聚乙烯生产线报警。反应釜温度异常上升：

- **正常工况**：80℃ 左右
- **现场读数**：92℃
- **风险**：温度继续上行 → 轻则产品批次报废，重则触发联锁停产

### 3.2 第一轮假设

```text
H1：冷却水循环泵故障
H2：温度传感器漂移
H3：工艺配方异常
H4：催化剂活性突变
H5：PID 参数被异常修改
```

> 如果这是普通问答系统，它可能会直接说："最大可能是冷却水循环泵故障，请优先检查冷却水系统。"这句话**看起来没错，但它还不是诊断**。
>
> **真正的诊断不是给出一个最像答案的答案，而是让每个假设进入验证流程。**

### 3.3 第一轮验证

| 假设 | 验证动作 | 证据结果 | 裁决 |
|------|---------|---------|------|
| **H1** 冷却水循环泵故障 | 查冷却水流量、泵压、回水温度 | 都正常 | **证伪** |
| **H2** 温度传感器漂移 | 对比冗余温度传感器读数 | 两只读数一致 | **证伪** |
| **H3** 工艺配方异常 | 查进料流量、配方切换记录、最近 1 小时 log | 都正常 | **证伪** |
| **H4** 催化剂活性突变 | 等实验室分析（≥30 分钟） | 等待中 | **needs_more_evidence** |
| **H5** PID 参数被异常修改 | 查控制参数变更日志、远程登录记录 | 待查 | **未开始** |

> **注意，这不是系统失败，而是迭代假设验证的正常工作方式**。假设本来就是拿来被推翻的。**初始概率只能决定验证顺序，不能决定最终真相**。

### 3.4 关键转折：HITL 触发

问题卡在 H4。H4 是催化剂活性突变，这类判断不能只靠 SCADA 实时曲线，要等实验室分析。取样、送检、分析，至少 30 分钟。生产线已停了 1 小时，每继续等待一轮，都是实打实的损失。

**一线工程师补充了一个新事实**：

> 02:33 有一次远程登录修改控制参数。

这条新事实进入系统以后，诊断发生关键转折。Agent 不应该只在原来的 H1 到 H5 上微调权重，**因为这条证据改变了问题的因果结构**。

### 3.5 Context Reset：重建假设树

```text
H5'：PID 参数被异常修改
    └─ 查 02:33 远程登录的具体改动
        └─ 发现 P 参数从 0.8 改到 2.5
            └─ 反推该改动会导致控温过激
                └─ 与当前温度震荡上行吻合
```

**根因**：另一个班组做参数优化测试时，把 PID 的 P 参数从 0.8 改到了 2.5，测试后忘记回滚。**恢复参数后，反应釜温度回归正常**。

### 3.6 闭环证据链

```text
02:33 远程登录
   ↓
PID P 参数 0.8 → 2.5
   ↓
控制行为异常
   ↓
温度震荡上行
   ↓
恢复参数后温度回归正常
```

> **这条链满足工业诊断里最重要的条件：不仅解释现象，还能被干预验证。恢复参数以后温度回归，就是最强的收敛证据。**

---

## 四、这次事故的迭代流程六步

### 第一步：Hypothesis Generator

根据当前症状生成假设清单：

```text
输入：
  - 反应釜温度异常上升
  - 正常 80℃，当前 92℃
  - 生产线：聚乙烯
  - 报警来源：PLC / SCADA

输出：
  H1 冷却水循环泵故障
  H2 温度传感器漂移
  H3 工艺配方异常
  H4 催化剂活性突变
  H5 PID 参数被异常修改
```

> 这里的输出**不是答案**，而是一个**待验证集合**。H1 到 H5 的概率排序，只是告诉工程师"先查哪个最划算"。

### 第二步：Verification Executor

把每个假设翻译成**可执行检查**：

| 假设 | 验证动作 |
|------|---------|
| H1 冷却水循环泵 | 查冷却水流量、泵压、回水温度 |
| H2 温度传感器漂移 | 查冗余传感器读数是否一致 |
| H3 工艺配方异常 | 查进料流量、配方切换记录、近 1 小时 log |
| H4 催化剂活性突变 | 触发实验室分析 |
| H5 PID 参数被异常修改 | 查控制参数变更日志、远程登录记录、参数 diff |

> **这一步最能区分工业 Agent 和聊天机器人**。聊天机器人会说"可能是冷却系统问题"。**诊断 Agent 必须继续问**："要查哪一个 metric？哪张 log？哪个传感器？哪个时间窗口？查到什么算证伪？"

### 第三步：Evidence Evaluator

根据证据裁决每个假设：

| 假设 | 裁决 |
|------|------|
| H1 | **falsified**（冷却水流量正常，不是概率下降一点，而是被当前证据证伪） |
| H2 | **falsified**（冗余传感器一致） |
| H3 | **falsified**（进料 log 正常） |
| H4 | **needs_more_evidence**（需要实验室分析，验证成本高、等待时间长） |
| H5 | **未开始** |

### 第四步：HITL 触发

人类补充关键事实：02:33 有远程登录修改。**工业现场很多关键信息不一定已经结构化进入系统**。它可能来自工程师刚看到的一条控制日志，也可能来自班组交接，也可能来自一句"昨天有人做过测试"。

> **这个事实一进入系统，就应该成为高权重证据事件（evidence event），而不是普通聊天上下文。**

### 第五步：Context Reset（重建假设树）

> 系统不应该把新事实塞进原来的 H1-H5 列表里继续微调。因为这条证据改变了问题的因果结构。所以更稳的做法是 **reset**：

**保留**：

- 当前症状：温度从 80℃ 上升到 92℃
- 已证伪假设：H1/H2/H3
- 未验证但高成本假设：H4
- 新关键证据：02:33 远程登录修改
- 时间压力：停线每小时 45 万
- 目标：尽快定位可验证根因，避免继续无效等待

**清空**：

- 旧假设树的排序惯性
- 已被证伪假设的上下文噪声
- 模型对 H1 的初始强先验

让新的 Hypothesis Generator 基于 **handoff artifact** 重新建树。结果就是 H5' 被拉到 95%，并开始下钻参数 diff。

### 第六步：验证并收敛

形成闭环证据链：

```text
02:33 远程登录
   ↓
PID P 参数 0.8 → 2.5
   ↓
控制行为异常
   ↓
温度震荡上行
   ↓
恢复参数后温度回归正常
```

> **这次事故对应的流程，就是一个带 reset 的假设验证闭环**。这张图要强调的是系统允许初始先验被证据推翻，并允许关键新事实触发重建假设树。**这才是迭代假设验证。**

### 四个要点

1. **初始概率可以作为验证顺序**。H1 初始最像答案，但证据回来以后，它必须被剪掉，因为**初始概率并不等于最终结果**。
2. **验证动作必须可执行**。不能只说"可能是冷却水问题"，而要明确查哪个 metric、哪段 log、哪个 sensor、哪个时间窗口，查到什么算证伪。
3. **新事实可以触发上下文的重置（context reset）**。当新证据改变问题的因果结构时，系统不应该背着旧上下文继续前进，而应该生成一份**可以交接的工件（handoff artifact）**，用干净上下文重建假设树。
4. **人在回路是新信息的获取渠道**。一线工程师补充的"02:33 远程登录修改"，可能是系统暂时没有结构化接入的关键事实，**有些证据最先存在于人的观察里**，此时需要先进行人的输入或判断。

---

## 五、三个角色：生成假设 / 执行验证 / 裁决证据

架构设计上，这个故障诊断系统**不应该由一个模型从头猜到尾**，可以拆成三个角色：

| 角色 | 职责 | 关键纪律 |
|------|------|---------|
| **假设生成器 Hypothesis Generator** | 根据报警、工艺拓扑、历史 case 和当前上下文，提出一组可验证假设，并给出优先级 | 责任是**打开搜索空间**，不是证明哪个原因是真的 |
| **验证执行器 Verification Executor** | 把每个假设翻译成具体动作：查 metric、读 log、对比 sensor、调用实验室流程、拉参数 diff | 责任是**拿证据回来**，不是偏袒某个假设 |
| **证据评估器 Evidence Evaluator** | 根据验证结果判断每个假设是 `verified` / `falsified` / `partial` / `needs_more_evidence` / `human_required` | 责任是**剪枝、回溯或收敛**，独立判断 |

### 5.1 三角色拆分降低确认偏误

> **如果同一个模型既生成假设、又执行验证、又评价自己，它很容易围着初始高概率假设继续找补**。H1 初始概率最高，它后面就倾向于在冷却水系统里不断补充支持这个假设的解释。

独立 Evaluator 的任务**不是帮 Agent 把概率最高的初始 H1 圆回来**，而是冷静地问：

```text
冷却水流量正常吗？泵压正常吗？回水温度正常吗？
如果都正常，H1 就证伪。初始概率再高也没有用。
```

> **工业诊断 Agent 的核心不是"生成一个聪明的假设清单"，而是把三个 Agent 负责的角色分开**：生成假设的人负责打开搜索空间，执行验证的人负责拿证据，评价证据的人负责剪枝。

### 5.2 与 Anthropic 三 Agent Harness 同构

| 工业故障诊断 | Anthropic 三 Agent Harness（软件开发） |
|------------|--------------------------------------|
| 假设生成器 | 规划器（Planner）|
| 验证执行器 | 生成器（Generator）|
| 证据评估器 | 评估器（Evaluator）|

> **领域不同，结构一样**：先把问题拆成可验证假设，再让执行器拿证据，最后由独立评价器剪枝、回溯或收敛。

---

## 六、三种迭代循环范式

| 范式 | 检验信号来源 | 适用任务 | 代表论文 |
|------|-----------|---------|---------|
| **ReAct**（边想边做）| **外部环境** | 工业故障诊断、财务归因、日志排查、合规核查（探查类）| arXiv:2210.03629 (ICLR 2023) |
| **Self-Refine**（先做再改）| **模型自己** | 写作、代码、解释优化（模型能自己看出问题的任务）| arXiv:2303.17651 (NeurIPS 2023) |
| **Plan-then-Execute**（先规划再执行）| **执行中的偏差** | 流程相对清楚、步骤可以预先排出来的长任务 | — |

### 6.1 关键区别

- **ReAct**：走一步看一步，计划随时变
- **Plan-then-Execute**：先排出大致路径，再边执行边微调
- **Self-Refine**：模型自己看不出来的盲区，自己批评自己也照样漏

> 一个判断该用哪种，**先看它的检验信号能不能从外部拿到**：能拿到真实证据的，**优先 ReAct**，因为外部证据比模型自评可靠很多。

---

## 七、关键纪律：一定要设停机条件

> 迭代循环有一个致命风险：**它不一定收敛**。有时候，**想得更多，反而越改越糟**。

### 7.1 为什么容易失控

模型一旦进入循环，很容易产生一种"**推理动量**"：

1. 第一轮提出一个假设
2. 第二轮不是冷静地证伪，而是不自觉地替这个假设**找支持**
3. 第三轮又在找补的基础上继续推，越走越远，越走越自信
4. 最后生成一大段**看起来很认真、其实没有证据增量**的解释

> 这正好和波普尔的精神相反。**健康的循环，每一轮都在试图推翻自己；而失控的循环，每一轮都在加固自己**。

### 7.2 三道停机条件

```text
提假设 → 设计验证 → 看证据 → 判断收敛 → 没收敛就修正再来一轮

三道停机条件守在循环外：
   1. 预算上限 → 到顶就停
   2. 收敛判据 → 够了就停
   3. 反思哨兵 → 原地打转就跳出
```

| 刹车 | 职能 | 防什么 |
|------|------|-------|
| **预算上限** | 最后的硬墙，最多迭代几轮 / 多少 token / 多少次工具调用 | **失控**（无界烧算力）|
| **收敛判据** | 循环的大脑：证据到底够不够支撑结论 | **过度推理**（该停不停）|
| **反思哨兵** | 中途急刹车：连续两轮证据增量几乎为零、质量没提升 | **空转**（原地打转）|

> 三道都没触发又收敛不了，把**部分结论和未解释的缺口**一起交人审。

### 7.3 交给人的半成品账

停不下来的时候，**不是再迭代一轮，而是把部分结论和未解释缺口一起交给人审**，并交出一份半成品账：

```text
- 已经解释了多少
- 每一块对应什么证据
- 还剩多少没解释
- 试过哪些假设
- 哪些假设被证伪
- 卡在哪一步
- 建议人类优先看什么
```

> **人接手的已经是一个被缩小过范围的问题**。即使 Agent 没有把问题完全破解，**人也不是从头开始判断**。

---

## 八、推理漂移率 + Locked Facts

### 8.1 推理漂移率

> 导论里，我们给推理系统设定了几个生产指标：验证通过率、首次路由命中率、单位验证成功成本、推理漂移率。**迭代假设验证的关键指标，是推理漂移率**。

什么叫推理漂移？如果前面已经确认"奖金政策以审批日期为准"，十步以后 Agent 又按发放日期计算了。**这就是推理漂移**。

它衡量的是**长任务推进过程中，已经验证过的目标、约束和事实，会不会被 Agent 悄悄遗忘或改写**。

### 8.2 为什么迭代假设验证最容易漂移

> 因为它的**循环最长**：
>
> - Direct 一轮就完
> - CoT 是一条链
> - 并行探索虽然有多条路径，但各条路径通常有明确边界
> - **只有迭代假设验证是一轮接一轮，每一轮都把上一轮结论带进来继续推**
>
> 循环越长，上下文越厚，早期已经锁定的事实就越容易被埋掉。

例如在总账缺口案例里：

- 第二轮已经确认"社保基数调整按 6 月生效记录归集"
- 到了第五轮，如果模型又按发放日期重算社保，**整张拆解表就会悄悄错掉**
- 表面上每一轮都在推进，trace 也很完整，但**某个口径中途被改写了**

### 8.3 Locked Facts 防漂移

> **防漂移不能靠模型记性，要靠 Trace**，也就是推理模块仪表盘上，Reasoning Trace 的 `ITERATIVE` mode。每一轮都要写进 trace：

```text
这一轮提出了什么假设
为什么提出这个假设
设计了什么验证
调用了什么工具
查到了什么证据
这个假设解释了多少缺口
哪些事实被锁定
为什么继续或停止
```

> **已验证的事实一旦进入 trace，就应该变成 `locked fact`**。后续每一轮提出新假设时，都要和 `locked facts` 对一遍。如果冲突，就**报警、回滚或要求人审**。

---

## 九、回到薪酬系统：37 万总账缺口怎么查

### 9.1 不会迭代 vs 会迭代

| 不会迭代的 Agent | 会迭代的 Agent |
|---------------|--------------|
| "可能是奖金普涨，可能是社保基数调整，可能是人员增加" | **三块证据**：奖金普涨 + 社保基数调整 + 新入职 |
| 笼统分析，没有证据增量 | **能对账的拆解**，链接到具体证据 |

### 9.2 迭代过程

**Round 1**：

- **假设**：缺口主要来自奖金普涨
- **验证**：调 6 月奖金审批单总额，和 5 月对比
- **证据**：奖金确实多发，能解释掉一部分缺口，但**不够解释全部 37 万**
- **结论**：partial，解释了一块，剩余缺口仍然存在

**Round 2**：

- **修正假设**：剩下的缺口可能来自社保基数年度调整
- **验证**：查社保基数生效记录，看 6 月有没有一批人的缴费基数集体上调
- **证据**：确实有一次年度基数调整在 6 月生效，**又解释掉一块**
- **结论**：partial，但还差一小截

**Round 3**：

- **再修正**：剩下那一小截可能来自新入职
- **验证**：查入离职台账，看 6 月是否新增员工
- **证据**：几个新员工的工资正好补上最后缺口
- **结论**：verified

### 9.3 37 万缺口拆解

```text
奖金普涨      → 解释一块
社保基数调整  → 解释一块
新入职        → 解释一块
P 那类个人调薪 → 解释一小块
```

链接上具体的证据（奖金审批单、社保生效记录、入离职台账、调薪审批），下游回看的时候，**才能精确对账到来源**。

> 这就是迭代假设验证在执行型 Agent 里的优势所在：**不会迭代的 Agent 给咱们"三个可能"；会迭代的 Agent 给咱们"三块证据"**。

---

## 十、最小生产级代码骨架

### 10.1 四个核心组件

```text
Hypothesis         表示当前假设
VerificationResult 表示验证结果
IterationTrace     记录每一轮
StopCondition      决定什么时候停止迭代循环
```

### 10.2 数据结构

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Protocol


class HypothesisStatus(str, Enum):
    DRAFT = "draft"
    VERIFIED = "verified"
    FALSIFIED = "falsified"
    PARTIAL = "partial"
    NEEDS_MORE_EVIDENCE = "needs_more_evidence"
    HUMAN_REQUIRED = "human_required"


@dataclass(frozen=True)
class EvidenceRef:
    source_id: str
    source_type: str
    version: str | None = None


@dataclass
class Hypothesis:
    hypothesis_id: str
    text: str
    status: HypothesisStatus = HypothesisStatus.DRAFT
    # 本轮新解释掉的缺口比例，0 到 1
    explained_delta: float = 0.0
    # 支撑或证伪它的证据
    evidence_refs: list[EvidenceRef] = field(default_factory=list)
    # 如果这个假设成立，下一步要查什么
    next_check: str | None = None


@dataclass
class VerificationResult:
    status: HypothesisStatus
    explained_delta: float
    evidence_refs: list[EvidenceRef]
    observation: str


@dataclass
class IterationRecord:
    round_id: int
    hypothesis: Hypothesis
    verification: VerificationResult
    cumulative_explained: float
    remaining_gap: float


@dataclass
class IterationTrace:
    task_id: str
    target_explained: float = 0.9
    max_rounds: int = 5
    min_progress_delta: float = 0.02
    records: list[IterationRecord] = field(default_factory=list)
    locked_facts: list[str] = field(default_factory=list)
    final_status: str | None = None
    stop_reason: str | None = None

    @property
    def cumulative_explained(self) -> float:
        return sum(r.verification.explained_delta for r in self.records)

    @property
    def remaining_gap(self) -> float:
        return max(0.0, 1.0 - self.cumulative_explained)

    def should_converge(self) -> bool:
        return self.cumulative_explained >= self.target_explained

    def is_stalling(self) -> bool:
        if len(self.records) < 2:
            return False
        last_two = self.records[-2:]
        return all(
            r.verification.explained_delta < self.min_progress_delta
            for r in last_two
        )
```

> **`explained_delta` 代表着本轮证据解释掉了多少缺口**：
> - 对总账归因 → 它可以是金额比例
> - 对故障诊断 → 它可以是诊断不确定性的下降
> - 对日志排查 → 它可以是已解释错误样本比例

### 10.3 Proposer / Verifier 协议

```python
class Proposer(Protocol):
    def __call__(self, question: str, trace: IterationTrace) -> Hypothesis:
        ...


class Verifier(Protocol):
    def __call__(self, hypothesis: Hypothesis) -> VerificationResult:
        ...


def iterative_hypothesis_test(
    question: str,
    trace: IterationTrace,
    propose: Proposer,
    verify: Verifier,
) -> IterationTrace:
    for round_id in range(1, trace.max_rounds + 1):
        hypothesis = propose(question, trace)
        result = verify(hypothesis)
        hypothesis.status = result.status
        hypothesis.explained_delta = result.explained_delta
        hypothesis.evidence_refs = result.evidence_refs
        record = IterationRecord(
            round_id=round_id,
            hypothesis=hypothesis,
            verification=result,
            cumulative_explained=trace.cumulative_explained + result.explained_delta,
            remaining_gap=max(
                0.0,
                1.0 - trace.cumulative_explained - result.explained_delta,
            ),
        )
        trace.records.append(record)
        if result.status is HypothesisStatus.VERIFIED:
            trace.locked_facts.append(hypothesis.text)
        if trace.should_converge():
            trace.final_status = "converged"
            trace.stop_reason = "target_explained_reached"
            return trace
        if trace.is_stalling():
            trace.final_status = "needs_human"
            trace.stop_reason = "reflection_sentinel_stalling"
            return trace
        if result.status is HypothesisStatus.HUMAN_REQUIRED:
            trace.final_status = "needs_human"
            trace.stop_reason = "human_required_by_verifier"
            return trace
    trace.final_status = "needs_human"
    trace.stop_reason = "max_rounds_reached"
    return trace
```

### 10.4 三个循环结束的出口

| 出口 | 触发条件 | 含义 |
|------|---------|------|
| **收敛** | `target_explained_reached` | 证据足够了，停止 |
| **反思哨兵** | `reflection_sentinel_stalling` | 连续两轮没有进展，停止 |
| **预算上限** | `max_rounds_reached` | 轮数到顶，停止 |

> **没有任何一个出口允许系统"继续想，直到模型满意"。这才是安全合理的迭代**。如果最后实在无法收敛，trace 也不是废的，它也会带着已经解释的部分、被证伪的假设、剩余缺口和证据引用交给人审。

### 10.5 37 万总账缺口的 Trace 形态

```text
Round 1
  Hypothesis: 缺口主要来自奖金普涨
  Evidence:   6 月奖金审批单总额
  Result:     partial
  Explained:  0.42

Round 2
  Hypothesis: 剩余缺口来自社保基数年度调整
  Evidence:   社保基数 6 月生效记录
  Result:     partial
  Explained:  0.35

Round 3
  Hypothesis: 剩余缺口来自新入职员工
  Evidence:   6 月入离职台账
  Result:     verified
  Explained:  0.18

Stop
  Reason: target_explained_reached
```

→ 生成一份**能对账的推理轨迹**。

---

## 十一、推理模块小结

到这里，**推理模块四个模式就配齐了**。它们不是四个孤立技巧，而是**一座分诊台加三间诊室**：

| 模式 | 角色 | 收什么任务 | 怎么干 |
|------|------|----------|-------|
| **[[lession18-复杂度路由\|复杂度路由]]** | 分诊台 | 任何任务 | 判断值不值得深想、该进哪间诊室 |
| **[[lession19-思维链\|CoT]]** | 链式诊室 | 证据齐、规则明、主路径清楚 | 拆成一串可验证命题，对账 |
| **[[lession19-并行探索\|并行探索]]** | 广度诊室 | 多个口径都说得通 | 几条路径同时跑，分歧本身就是信号 |
| **迭代假设验证** | 深度诊室 | 根因藏得深、一次想不到底 | 一轮轮查证据、剪假设、锁事实、追剩余缺口 |

### 控制链

> **三间诊室出来的结论，都不能直接放行。它们还要过验证器**：通过才输出，不通过就换路、升级模型、补证据或交人审。
>
> **这就是推理模块的完整控制链**：
>
> ```text
> 先路由 → 再推理 → 推理之后要验证 → 验证不过就换路 → 达到停止条件才输出
> ```

### 工程公式

```text
Hypothesis → Verification → Evidence → Evaluation → Update / Stop
```

---

## 十二、思考题

1. CoT、ReAct、Self-Refine、Reflexion、Self-Consistency、Tree of Thoughts 以及其它一系列大模型推理相关的论文经典，它们都属于哪一类设计模式？大家给这些论文落个盘（**不一定有标准答案**）。
2. 你的 Agent 在处理归因、诊断、排查问题时，是否会一轮轮提出假设、查证据、修正方向？有没有**证据链的设计**？
3. 你系统里有没有跑着跑着停不下来的 Agent 任务？连续两轮没有证据增量时，它会**提前停**，还是继续空转？应该如何设计？
4. 你的 trace 里有没有 `locked facts`？如果第二轮已经确认"社保按生效记录归集"，第五轮模型又想换成发放日期口径，系统能不能发现这是**推理漂移**？当停不下来的时候，你的 Agent 交给人的是一句"未能完成"，还是一份 **Trace**：已经解释了多少、还差多少、试过哪些假设、哪些被证伪、下一步建议查什么？

---

## 十三、下一讲预告

到这里，**推理模式组四个模式就讲完了**。

- CoT 让 Agent 想得显式、能对账
- 复杂度路由让它想得分档、按需
- 并行探索让它想得广、择优
- 迭代假设验证让它想得深、收敛

四个加起来，是 Agent 把感知到的信息、记住的经验，组织成一个**可靠判断**的四种形式。**一座分诊台，三间诊室，这就是推理模块交给你的东西。**

但判断做出来，只是想清楚了，**还没动手**。

下一组我们讲**行动（Action）**：一个判断要落到真实世界，要改状态、要调外部系统，怎么让 Agent 安全、可控地把它做出来。**想清楚之后怎么稳稳地动手，是下一个模式组的事。**

---

## 精选留言摘录

> **Geek_75ba88**：
> 想问下老师想这样的设计模式实际落地生产的表现形式是怎么样的，看老师的伪代码感觉好像实现起来是要自己编排复杂的流程比如 langgraph 去实现这一套流程，还是说通过 agent 加上抽象出一个 skill 定义好 sop，结合一些自定义的 tool 去实际落地？基于当下流行的 harness 模式我偏向是后者，不过感觉这么复杂的流程好像又不是一个 skill 能搞定的，太复杂的逻辑很难保证模型一步一步按照我们的要求走，应该又要和前面课程关于记忆等等相关内容又有结合起来看，总之要实际落地还是有点懵~
>
> **作者回复**：可以用 Skill 落地简单的模式实现；但最主要的还是通过 **Agentic Workflow** 参照理论和伪代码来具体落地。从行动模块开始，我要给出完善的落地 Demo 和落地实操过程了，应大家要求，换个讲述方法。
>
> 如果我们举例把这一讲的落地流程说的更详细一些：
> - **编排搭骨架**——while 循环、预算上限、收敛判据、反思哨兵、交人审出口
> - **agent 填每轮推理**——propose/verify 里的判断
> - **skill 装领域 SOP**——"怎么算证据充分、假设从哪些维度提"（薪资诊断：先奖金→再社保→再入离职）
> - **tool 接外部**——查审批单/社保/入离职（verify 靠它拿真实证据）
> - **memory 记跨轮**——试过哪些假设、排除了什么、还差多少（progress ledger + failure journal），**这就是"要跟记忆结合"**

> **开心小毛**：
> 想请教老师，Verifier 类该如何呢，他的 Verify 方法也是谓词对应的工具调用么，是要和 CoT 的 Validator 一样通过 Registry 注入到假设生成器的系统提示里么？
>
> **作者回复**：**建议把 Verifier 做成运行时能力注册表，而不是把工具实现整包塞进假设生成器提示词**。生成器只知道可验证的证据类型和成本，编排器按谓词与风险选择验证器；验证器再调用规则、查询或评测模型，返回结构化结果。

> **街角·陌路△**：
> 为了保证业务稳定，我们通常还是会尽量用 DAG，把关键流程、工具权限、验证节点和停止条件固定下来，让 Agent 尽可能按照我们预期的流程去执行。但这样好像也会带来一个问题：Agent 能看到哪些证据、可以从哪些方向进行验证，其实已经被设计者提前限定了。最后得到的结论，可能只是在我们人为划定的证据范围里成立。
>
> 但如果反过来，把证据收集和验证路径更多地交给 Agent 自己决定，又容易出现另外两个问题：一个是它可能不断调用工具、重复搜索，造成 Token 空转；另一个是它可能会带着自己原来的假设，专门去寻找能够证明自己是对的证据，而不是真的尝试去推翻自己。
>
> 所以我现在的理解是，工程上可能不是在 DAG 和自主 Agent 之间二选一，而是要判断：**哪些东西应该提前固定下来，哪些地方又应该给 Agent 留出一定的探索空间**。
>
> 想请教一下老师，在真实项目里，您一般是怎么划分这条边界的？
>
> **作者回复**：**真实项目里通常固定"安全骨架"，开放"证据探索窗口"**。权限、预算、必查事实、停止线和写动作顺序放进 DAG；搜索关键词、候选假设和只读工具顺序可以让 Agent 探索。**越接近不可逆动作，自由度越小**。

> **有学识的兔子**：
> 1. 都属于 reasoning 的范畴；cot：思维链，一步步的推导；self-consistency：并行探索，多路采样的投票机制；tot：像是老鼠走迷宫这个场景，并行探索+路由选择；self-refine：生成，评估，改进；reflexion：迭代+反思
> 2. 会有，但缺少证据链的设计。比如提供事实的原子信息，在验证假设，提供证据信息，可以提升验证流程的效率和可信度
> 3. 在使用 copilot 会经常出现。可以降低迭代的层数，限制资源的上限；同时缺少依据时及时人工协助，降低无效的多轮的推理
>
> **作者回复**：这组归类很扎实，尤其是你补出的证据链和停止线。**再加一条边界：多路径若共享同一份错误上下文，投票只会放大共识；迭代若没有新的外部证据，也可能只是反复润色原假设**。

> **PatrickL**：
> 推理×循环中"迭代假设"的范式之一"Plan-then-Execute(先规划后执行)"和行动×编排中的"规划执行"有一定的重叠，说明双轴模式也不是完全正交的，会有一定的耦合，就像 GoF 的 23 种设计模式也不是完全解耦的。
>
> 编程就好比是写作，设计模式就好比是字词典。字词典在解释词汇时并不是低耦合的，而是互相引用，但优秀的文章必然是结构清晰（金字塔结构）和逻辑缜密（遵循 MECE 原则）的。
>
> **作者回复**：**这个观察很准确**。双轴提供主坐标，不承诺每个模式彼此绝缘。**推理侧的 Plan-then-Execute 关心怎样形成和修正假设，行动侧的规划执行关心计划怎样约束真实动作**；两者会接线，也会共享对象，但守的是不同失败面。

> **PatrickL**（第二弹）：
> 科学方法：
> 提出假设→设计验证→观察证据→修正判断→继续/收敛
> Hypothesis→Verification→Evidence→Evaluation→Update / Stop
>
> 科学方法的演进：
> 培根的归纳法→休谟的归纳法困境→贝叶斯的证据更新信念→波普尔的证伪原则→维纳的控制论→大模型的迭代假设验证
>
> 迭代假设验证是科学方法的工程化。
>
> 迭代假设验证的三种范式：
> - ReAct（边想边做）
> - Self-Refine（先做再改）
> - Plan-then-Execute（先规划后执行）
>
> **作者回复**：👻

> **哈喽**：
> 老师，这块儿能举一些真实 AI Agent 的代码示例吗，更容易理解消化。
>
> **作者回复**：**下一讲开始，全是实操+代码案例了**。理论还是有，而且还是很多，融合在实操步骤里面讲解。更多的代码案例可以去我的 Github 和 ADPS 社区看（希望未来有人向社区贡献更多真实 AI Agent 的代码示例）。

---

## 参考资料

- Francis Bacon. *Novum Organum*. 1620.
- David Hume. *A Treatise of Human Nature*. 1739–1740.
- Thomas Bayes. *An Essay towards Solving a Problem in the Doctrine of Chances*.
- Karl Popper. *Logik der Forschung*. 1934 / *The Logic of Scientific Discovery*.
- Norbert Wiener. *Cybernetics: or Control and Communication in the Animal and the Machine*. 1948.
- Yao et al. *ReAct: Synergizing Reasoning and Acting in Language Models*. arXiv:2210.03629, ICLR 2023.
- Madaan et al. *Self-Refine: Iterative Refinement with Self-Feedback*. arXiv:2303.17651, NeurIPS 2023.
- Shinn et al. *Reflexion: Language Agents with Verbal Reinforcement Learning*. arXiv:2303.11366, NeurIPS 2023.
- Gema et al. (Anthropic). *Inverse Scaling in Test-Time Compute*. arXiv:2507.14417, 2025.