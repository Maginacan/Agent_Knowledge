---
title: "14｜进度追踪：长任务中别让 Agent 走丢"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/990515"
tags:
  - "#pattern/memory"
  - "#axis/memory"
  - "#axis/orchestration"
  - "#topic/progress-tracking"
  - "#topic/goal-drift"
  - "#topic/long-horizon"
  - "#topic/saga"
  - "#topic/event-sourcing"
  - "#concept/goal-contract"
  - "#concept/anchor-ledger-set"
  - "#concept/drift-watchdog"
audio: true
duration: "26:49"
publish_date: "2026-06-26"
course_progress: 14
pattern_position: "记忆 × 编排"
module: "记忆：沉淀之美（第 4 讲 / 共 5 讲）"
related:
  -"[[lession15-检索增强]]"
  -"[[lession17-失败日记]]"
---

# 14｜进度追踪：长任务中别让 Agent 走丢

> **本讲定位**：记忆模式组第三个具体模式 → 解决"长任务跑到一半，Agent 怎么记住自己已经做了什么、为什么这么做、下一步该怎么接"。

| 属性 | 内容 |
|------|------|
| 课程模块 | 记忆：沉淀之美（第 4 讲 / 共 5 讲）|
| 课程编号 | 14 |
| 双轴坐标 | **记忆 × 编排**（记忆：保存执行轨迹；编排：协调者横切维护任务台账）|
| 时长 | 26:49 |
| 发布日期 | 2026-06-26 |
| 前置讲 | [[lession15-检索增强]] — 业务证据取回 |
| 后续讲 | [[lession17-失败日记]] — 把失败做成可召回经验 |

## 一句话核心

> **进度追踪（Progress Tracking）= 长任务里的防迷失机制**。
>
> 它**不是 todo list 这么简单**，而是要补一整套 Agent 天然缺失的工程支架：
>
> - 一份**像会议纪要一样的目标契约**，反复提醒"这次到底要交付什么，不要交付什么"
> - 一份**像项目计划一样的里程碑状态**
> - 一份**像工作日志一样的进度账本**（不是流水账，是会计账本）
> - 一套**像测试清单和审批节点一样的验证闸门**
> - 还要有**自动化的漂移哨兵**

---

## 一、人类团队的长周期项目管控

### 1.1 现实场景

> 周一早上，工程经理把团队叫进会议室，说这个月最重要的任务，是把**薪酬系统里的异常核验流程补齐**，让 6 月批次能顺利上线。
>
> 会上大家分了工：有人核对规则，有人接口联调，有人补测试，有人准备人审清单。
>
> 散开后：
>
> - 写接口的人在 debug 权限报错 → 越修发现问题越深 → **不知不觉花了半天**
> - 补测试的人发现历史数据字段命名不统一 → **开始顺手清洗数据**
> - 负责规则的人**被拉去开另一个会** → 回来不翻会议纪要想不起来讨论到哪一步

### 1.2 防跑偏装置

> 如果这个团队没有任何外部项目管理工具，**这个项目很快就会散掉**。大家每个人都在勤奋工作，但**勤奋不等于还朝同一个目标前进**。
>
> **人之所以没有那么容易彻底迷失，是因为我们的工作环境里已经有了一整套"防跑偏装置"**：

| 防跑偏装置 | 防止什么 |
|------------|----------|
| **会议纪要** | 把原始目标重新固定，提醒"我们这周要交付的不是所有问题都解决" |
| **项目计划** | 长目标拆阶段，让人知道现在做的是"异常核验"，不是"完整规则治理" |
| **任务清单** | 告诉你还有哪些事没做 |
| **测试清单** | 告诉你事情做完没有 |
| **审批节点** | 拦住那些不能直接进行的高风险动作 |
| **同事提醒 + code review + 晨会同步** | 把局部工作重新拉回到全局目标 |
| **业务系统 + 数据库** | 随时查阅数据的来源和状态 |

> 人做长项目，靠的是**会议纪要、项目计划、任务清单、测试清单、审批节点和同事提醒**。
>
> **Agent 做长项目，也要有对应的外部支架，而且要更显式、更结构化、更可恢复**。

---

## 二、为什么要有进度追踪

### 2.1 草稿纸的进化路径

> 人并不是只靠脑子记住一切。人是**靠一整套外部化支架**，才把一个长项目稳稳推进下去的。
>
> 进度追踪做的，就是把**草稿纸上那些值钱的判断，蒸馏成一份下一轮还能接着用的进度账单**。

### 2.2 长程 Agent 的脆弱性

> **长程任务的 Agent 尤其需要进度追踪**。因为真实工作流里，**任务动辄几十轮、上百轮**：
>
> - Agent 一开始记得目标
> - 走着走着**被细节牵走**
> - 一个局部错误没纠正
> - 后面十步都在这个错误上搭楼
>
> 它解决了一个小问题，**却忘了原来要交付什么**。
>
> 最后日志很长、工具调用很多，**结果离目标越来越远**。

### 2.3 进度追踪 ≠ todo list

> **长程 Agent 里的进度追踪，不能只理解成一张 todo list**。
>
> 它真正要补的，是**人类工作里那些默认存在、但 Agent 天然缺失的工程支架**。

### 2.4 五种迷失方式

> 长任务里的迷失，通常是**一点点积累**出来的，很少一步崩掉。常见问题有五种。

| # | 迷失方式 | 表现 |
|---|----------|------|
| 1 | **目标漂移** | 原来要"完成可上线的 auth.py 重构"，跑着跑着变成"把当前这个 import error 修干净"。局部目标越来越具体、越来越占注意力，最后这个小目标做完了，**Agent 却没有回到全局** |
| 2 | **状态漂移** | Agent 以为环境是 A，真实环境已经是 B；以为文件改完了，其实保存失败；以为测试跑过了，其实只跑了一部分。**目标看着没错，但它对世界的记账错了** |
| 3 | **错误放大** | 长任务是链式依赖。第一步误判了一个 schema 字段，后面的查询、验证、文档都会顺着这个误判长出来。**最后是一整套看似完整、方向却错的产物** |
| 4 | **细节过载** | 碰上一个不完美的设计、一条 Warning Message，Agent 容易围着它越钻越深。**把预算耗在一个小坑里，忘了这个坑只是当前里程碑的一小部分** |
| 5 | **完成幻觉** | Agent 把"**计划上的 todo 都打勾了**"当成"**目标达成**"。更新定价文档、通知销售、改 FAQ 都做了，**却忘了验证真实报价系统有没有同步** |

---

## 三、进度追踪的来龙去脉

### 3.1 在双轴框架的位置

| 轴 | 类型 | 原因 |
|----|------|------|
| 认知功能 | **记忆** | 它保存的是一次任务的**执行轨迹**：原始目标、当前里程碑、已完成步骤、关键决策、阻塞点、验证结果、下一步动作 |
| 执行拓扑 | **编排**（不是链式）| 进度状态不是"某一步算完之后把结果传给下一步"那么简单，**它更像一个横切在所有步骤之上的协调者，始终拥有一份任务台账** |

> **每一个编排模式都是该模式组中最复杂、最不好掌握的模式**，还望大家认真学习、耐心去理解。

### 3.2 经典分布式系统血脉

| 年份 | 来源 | 核心思想 |
|------|------|----------|
| **1987** | **Garcia-Molina & Salem**：Saga | 把长事务拆成可交错执行的子事务，**要么全部完成，要么用补偿动作回滚**已做的部分 → **验证闸门 + 回滚的祖先** |
| **1992** | **ARIES**（C. Mohan et al）| **write-ahead logging（WAL）**：先把每一步写进日志，崩了再按日志恢复 → **断点恢复的祖先** |
| **2005** | **Martin Fowler Event Sourcing** | 所有状态变更都以一串**只追加的事件**存下来："**过往条目永不擦改，纠错靠追加补偿条目**" → 会计账本的比喻 |
| **2023** | **CoALA**（Sumers et al）| 把长期记忆分成**情景、语义、程序**三类，进度追踪监控的正是**情景记忆**（这次任务到底发生了什么）|
| **2024-11** | **Microsoft Magentic-One** | 编排器维护两本账：**Task Ledger**（事实/猜测/计划）+ **Progress Ledger**（当前进度/分工/完成判断）|
| **2025-06** | **Anthropic SubAgents** | 主 Agent 先把**计划写进持久记忆**，再派生子 Agent，**子 Agent 只回传一两千 token 的蒸馏摘要** |

### 3.3 为什么不能直接照搬

> 传统系统的状态是**确定的**。一个 dashboard 请求要写哪张表、哪个字段，设计时就定死了。
>
> Agent 不一样，它的状态一半是**基于当前语义叙事状态的**（不是明确的字段值或 Yes / No），**会被压缩、被改写、被重新解释**，而且它每一步真正需要哪段记忆是**不确定的**。
>
> 所以**我们不能只搬结果，还要专门防范目标和判断在时间里漂移**。

### 3.4 工业界方案

**Magentic-One 两本账**：

```
外层 Task Ledger     记事实、猜测、计划
内层 Progress Ledger  记当前进度、各 Agent 分工、"完成没有"的判断
专家 Agent 只负责执行单步
```

**Anthropic SubAgents 模式**：

> 主 Agent 先把计划写进持久记忆，**再派生几个子 Agent**，每个子 Agent 在自己独立的上下文窗口里干活，**只回传一两千 token 的蒸馏摘要**，主 Agent 负责综合，**而不去接管子 Agent 的工作轨迹**。

> **如果让子 Agent 把自己的本地消息、重试、中间失败一股脑灌回共享上下文，主 Agent 很快就会被污染**。
>
> **所以进度状态要由一个协调者集中维护，让每一步的局部噪音被隔离在子上下文里，目标、状态和机械参数不随链式传递漂移**。

---

## 四、三平面分治：把"迷失"拆成三件事

> 执行型 Agent 跑偏的代价比内容生成型严重得多。
>
> - 内容型跑偏，结果是报告不准、摘要偏题
> - 执行型跑偏，做的是**薪酬、报销、入职、代发、报税**这些异构 API 编排，**要传业务实体 id、金额、账号、税号、批次号**，一个参数串错就是业务交付失败，甚至误操作

### 4.1 三个平面

| 平面 | 名称 | 职责 |
|------|------|------|
| **SessionWorkspace** | 调度态 | **任务 DAG**：ready / blocked / completed |
| **SessionNarrative** | 叙事态 | **锚、账、集** |
| **SessionState** | 机械态 | **API 入参的可审计真值**（带 Provenance）|

> 很多长任务跑偏，就是**把这三件事混在了一起**：
>
> - 自然语言摘要里夹着业务 id
> - 工具结果里夹着目标解释
> - 又要求 LLM 从一堆文本里**自己找出下一步该传哪个参数**
>
> **分平面，就是先把责任分清楚**。

### 4.2 一句话总览

> **叙事态管意义，机械态管真值，调度态管顺序，编排器负责把三者对齐**。

---

## 五、叙事状态平面：锚、账、集

> 叙事态可以收束为三个关键词：**锚、账、集**。
>
> - **锚**：锁住原始目标和边界
> - **账**：记录里程碑级的进展和关键决策
> - **集**：每一步从账里**投影出来的当前最该看的工作集**
>
> - 锚用于**预防目标漂移**
> - 账用于**预防历史断片**
> - 集用于**预防上下文过载**

### 5.1 锚：目标契约

> 长任务不能只靠一句自然语言目标。
>
> 用户说"帮我把这个月薪资跑一下"其实太过宽泛，Agent 会把它理解成建薪资组、改规则、同步考勤、核对社保、生成代发表、准备报税一大堆方向。
>
> **所以要把目标冻结成一份可引用的契约（Spec）**。

```yaml
goal_id: payroll-run-shanghai-sales-202606
user_goal: 为上海市场部准备 2026 年 6 月薪资批次，按上月规则生成快照，核验异常后提交人审
success_criteria:
  - 员工范围为上海市场部 6 月在职员工
  - 复用 5 月规则，并记录本月差异
  - 创建薪资组和批次，相关 id 由 SessionState 托管
  - 生成快照，完成异常核验
  - 代发、报税、正式提交前进入人工 review
non_goals:
  - 不修改员工主数据
  - 不改规则模板
  - 不直接发起银行付款或税局申报
constraints:
  - 金额、账号、员工 id 只从工具返回和 SessionState 读取
  - 业务 API 入参需要记录 Provenance
  - 发现规则缺口或金额异常时暂停并请求人审
```

> **这份契约比 todo 稳定得多**：
>
> - todo 可以随便增删，目标契约不能
> - 要改也得写一条 `goal_changed` 事件说明**谁改的、为什么**
> - 它还同时写了 **`success_criteria` 和 `non_goals`**
>
> **很多 Agent 跑偏，就是因为只知道要做什么，不知道现在哪些事不要做**。
>
> `non_goals` 是**防止细节过载的一道护栏**：
>
> - Agent 看到薪资项名称不统一，可能花十几轮去做完整薪资科目治理
> - 而当前任务只是生成可审核快照

### 5.2 账：进度账本（不是日志）

> **账不等同于日志**。
>
> - **log** 像是流水账
> - **ledger** 像是**会计账本**
>
> 账本要**能对账**，这正和前面提到过的事件溯源概念对应。
>
> **一条进度账至少记四样**：

```yaml
event: 创建 2026 年 6 月上海市场部薪资组
decision: 复用 5 月薪资规则模板
reason: 用户要求"按上月规则"，本月规则变更未确认
evidence_refs:
  - tool:create_payroll_group#20260612-1005
  - policy/payroll-rule-2026-05.md
state_delta:
  write: [STATE.payroll_group_id]
  scope: company:acme / org:shanghai-sales / month:2026-06
next_action: 生成薪资批次，绑定 STATE.payroll_group_id
```

> 这样写，Agent 恢复时**不用读完整对话，只读目标契约、当前里程碑以及最近几笔账就够了**。
>
> 在人进行审核的时候，也能一眼看出**究竟是哪个决策把任务带偏了**。
>
> 而错误也不会悄悄放大，因为**每个关键判断都有 `evidence_refs` 进行证据的溯源，证据错了可以沿链路回滚**。

### 5.3 集：从账里蒸馏的当前工作集

> 账会越来越长，**不能每一步都塞进上下文**。
>
> 集就是从账里**再蒸馏、裁剪出当前这一步最相关的一小包材料**，和锚一起注入模型。
>
> 它同时解决两件事：
>
> - 每次注入都**带着锚**，Agent 不会忘原始目标
> - 每次**只给当前子任务要的材料**，Agent 也不会被全部历史淹没
>
> 这就是长任务里的"**接手包**"，**新一轮 Agent 先读锚和集，再按需回账本查证**。

### 5.4 三者关系

> 这个锚、账、集的设计，比普通 todo 更接近长程任务的真实需要：
>
> - Agent 每一步不需要看全部历史，**只需要看当前步骤相关的那一小包"集"**
> - 但这个"集"**必须带着锚**，否则它会越来越像最近几轮聊天的摘要，**慢慢脱离原目标**

> **进度追踪不是把上一步结果简单传给下一步，而是由编排器持续维护整份任务台账**。

---

## 六、机械状态平面：别让 LLM 拼接真值

### 6.1 两个平面的区别

| 平面 | 服务 | 保存 | 组成 |
|------|------|------|------|
| **叙事态** | 模型推理 | **任务的意义和脉络** | 锚、账、集 |
| **机械态** | **确定性执行** | **API 真正需要的精确参数** | **由程序维护**，不让模型在自然语言中复制和拼接 |

> **两者通过状态引用衔接**。
>
> 叙事态不会把一个薪资批次的 ID `pg_84721` 直接塞进进度摘要，而是会写：
>
> "下一步创建薪资批次，需要 `STATE.payroll_group_id`。"
>
> 执行前，编排器再从 SessionState 中解析这个引用，**把真实值绑定到工具参数**。

### 6.2 一次完整动作的闭环

```
叙事态提出动作意图
       ↓
调度态确认动作 ready
       ↓
编排器解析状态引用
       ↓
机械态提供精确参数
       ↓
工具执行
       ↓
机械态写入新真值
       ↓
叙事账本追加事件
       ↓
验证闸门对账
       ↓
调度态推进到下一步
```

### 6.3 机械态字段示例

```python
key: payroll_batch_id
scope: company:acme / org:shanghai-sales / month:2026-06
provider: create_payroll_batch
runtime_layer: plan_exec.M3.step1
value_ref: STATE.payroll_batch_id
trust: tool_output
```

> LLM 可以说"**下一步提交 6 月的薪资批次**"，但真正传给 API 的 `payroll_batch_id`，由程序按坐标从 SessionState 读取。
>
> 这样进度追踪才能回答执行型系统里**最关键的问题**：
>
> - 这个参数**从哪来**
> - **在哪一步产生**
> - 后来被**哪些工具用了**

> 如果目标没偏、但参数串了，任务也一样会失败。
>
> **叙事态靠锚账集，机械态靠状态平面和数据来源（Provenance）**。

---

## 七、三个调度收敛器：复诵、哨兵、闸门

> 在调度平面中，还要有**三个动作持续把任务往回收**。

### 7.1 复诵（Recitation）

> **Manus** 在文章指出：一个典型复杂任务平均要调**大约五十次工具**，这么长的循环，Agent 很容易偏题。
>
> Manus 的做法是**不断重写 todo.md**，**把全局计划反复推回上下文的尾部**。

### 7.2 漂移哨兵（Drift Watchdog）

> 光让 Agent 自己写进度还不够，**要有一个哨兵定期问**：
>
> "**你现在做的事，还跟原目标有关吗？**"

**哨兵评估的四个分数**：

| 分数 | 含义 | 异常处理 |
|------|------|----------|
| `goal_relevance` | 当前动作和原目标的相关度 | 下降 → **触发复诵** |
| `milestone_progress` | 当前里程碑有没有推进 | 停滞 → **要求缩小范围** |
| `evidence_health` | 关键结论有没有证据 | 变差 → **禁止汇报完成** |
| `error_pressure` | 最近错误是不是在累积 | 上升 → **暂停进入诊断** |

> 信号异常时**不一定要终止任务，可以分级处理**。

### 7.3 验证闸门（Verification Gate）

> 长任务最怕"**看起来很努力，结果没验收**"。
>
> 闸门要把每个里程碑的验收条件落成**具体检查**：

| 闸门检查项 | 含义 |
|------------|------|
| 快照人数 | **等于员工范围吗** |
| 金额波动 | **在可解释范围内吗** |
| 异常项 | **都标出了吗** |
| 关键 id | **都有 Provenance 吗** |
| 高风险动作 | **还 blocked 吗** |
| 待人审清单 | **生成了吗** |

> 不通过就置成 `needs_rework`，写一条 `failed_gate` 事件，**回到对应里程碑，不许进入下一阶段**。

> **很多企业长程 Agent 失败，就是因为没有闸门**。
>
> 从一个阶段自然滑到下一个，**上一阶段的错误也自然跟着传递过去**。
>
> 这就是 Agent 时代的 **Saga 补偿思路的具体实现**。

### 7.4 三收敛器协同

> **锚账集加机械态稳住状态**，再用**复诵、漂移哨兵、验证闸门三个收敛器持续把任务拉回原目标**，**断点从锚账集恢复**。

---

## 八、一份可执行的长程任务状态

### 8.1 完整 Schema 代码

```python
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime, timezone
from enum import Enum
from typing import Any

class TaskStatus(str, Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    BLOCKED = "blocked"
    NEEDS_REVIEW = "needs_review"
    NEEDS_REWORK = "needs_rework"
    COMPLETED = "completed"

class DriftLevel(str, Enum):
    OK = "ok"
    WATCH = "watch"
    RECENTER = "recenter"
    PAUSE = "pause"

@dataclass
class GoalContract:
    """目标契约：把目标、边界和验收标准冻结下来。"""
    goal_id: str
    user_goal: str
    success_criteria: list[str]
    non_goals: list[str]
    constraints: list[str]
    version: int = 1

@dataclass
class Milestone:
    """里程碑：把远目标拆成可验收的近目标。"""
    milestone_id: str
    title: str
    acceptance: list[str]
    status: TaskStatus = TaskStatus.PENDING
    active_subgoal: str | None = None

@dataclass
class ProgressEvent:
    """
    进度账本中的一条事件。
    账本只追加，不覆盖。每一条记录都应说明：
    发生了什么、为什么这样决策、依据是什么、状态发生了什么变化。
    """
    event: str
    decision: str | None = None
    reason: str | None = None
    evidence_refs: list[str] = field(default_factory=list)
    state_delta: dict[str, list[str]] = field(default_factory=dict)
    next_action: str | None = None
    created_at: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )

@dataclass
class MechanicalValue:
    """机械态：每个可复用的真值都带来源、层级和信任信息。"""
    key: str
    scope: str
    value_ref: str
    provider: str
    runtime_layer: str
    trust: str

@dataclass
class DriftSignal:
    """漂移哨兵：判断当前执行是否偏离目标、证据或里程碑。"""
    level: DriftLevel
    goal_relevance: float
    milestone_progress: float
    evidence_health: float
    error_pressure: float
    reason: str

@dataclass
class LongHorizonTaskState:
    """长程任务状态：保存恢复任务所需的最小可靠上下文。"""
    task_id: str
    goal: GoalContract
    milestones: list[Milestone]
    current_milestone_id: str
    status: TaskStatus
    ledger: list[ProgressEvent] = field(default_factory=list)
    working_collection: dict[str, Any] = field(default_factory=dict)
    mechanical_state: dict[str, MechanicalValue] = field(default_factory=dict)
    open_blockers: list[str] = field(default_factory=list)
    next_action: str | None = None
    last_drift_signal: DriftSignal | None = None

    def current_milestone(self) -> Milestone:
        """返回当前正在推进的里程碑。"""
        for milestone in self.milestones:
            if milestone.milestone_id == self.current_milestone_id:
                return milestone
        raise ValueError(
            f"Current milestone not found: {self.current_milestone_id}"
        )

    def resume_packet(self) -> dict[str, Any]:
        """
        生成恢复包。
        任务中断后，新一轮 Agent 不需要翻完整聊天记录，
        只读取这个包，就能知道目标、当前近目标、最近账本、开放阻塞和机械态索引。
        """
        return {
            "goal": self.goal,
            "current_milestone": self.current_milestone(),
            "working_collection": self.working_collection,
            "recent_ledger": self.ledger[-5:],
            "open_blockers": self.open_blockers,
            "mechanical_state_keys": sorted(self.mechanical_state.keys()),
            "last_drift_signal": self.last_drift_signal,
            "next_action": self.next_action,
        }

    def recitation_prompt(self) -> str:
        """
        生成复诵提示。
        每次继续执行前，把目标锚、当前里程碑和下一步动作推回上下文尾部，
        降低长程任务中的目标漂移。
        """
        milestone = self.current_milestone()
        return "\n".join(
            [
                "Re-center before continuing.",
                f"Original goal: {self.goal.user_goal}",
                f"Success criteria: {self.goal.success_criteria}",
                f"Current milestone: {milestone.title}",
                f"Active subgoal: {milestone.active_subgoal or 'decide it'}",
                f"Non-goals: {self.goal.non_goals}",
                f"Constraints: {self.goal.constraints}",
                f"Open blockers: {self.open_blockers or 'none'}",
                f"Next action: {self.next_action or 'decide it'}",
                "Do not report completion until the current verification gate passes.",
            ]
        )
```

### 8.2 七个核心对象的设计意图

| 对象 | 角色 | 防什么 |
|------|------|--------|
| **GoalContract** | 锚：冻结原始目标、成功标准、非目标和约束 | 防止长程执行过程中**目标被悄悄改写** |
| **Milestone** | 把远目标拆成近目标，让 Agent 每一步都知道推进到哪个阶段 | **进度不可见** |
| **ProgressEvent** | 只追加的进度账本，记录为什么决策 / 依据 / 下一步 / state_delta | **决策链断裂** |
| **`working_collection`** | 每一步**投影**给模型的"集"，控制模型此刻真正需要看的内容 | **上下文过载** |
| **MechanicalValue** | 记录机械参数的**Provenance**（来源/层/信任）| **参数串错** |
| **DriftSignal** | 漂移哨兵的判断：是否偏离原始目标 / 当前里程碑 / 可靠证据 | **目标漂移** |
| **recitation_prompt()** | 把目标锚 + 当前里程碑 + 下一步动作**重新推回上下文尾部** | **长循环偏题** |
| **resume_packet()** | 任务中断后，新一轮 Agent 不需翻聊天记录就能接上 | **断点丢失** |

### 8.3 todo list vs LongHorizonTaskState

> **LongHorizonTaskState 才是整个长程任务的状态容器**。它把：
>
> - 目标锚
> - 里程碑列表
> - 当前里程碑
> - 任务状态
> - 进度账本
> - 工作集
> - 机械态
> - 阻塞项
> - 下一步动作
> - 漂移信号
>
> **统一收拢在一起**。换句话说，它定义了"**这个任务现在到底处在什么位置**"。

> **没有这个总状态，GoalContract、Milestone、ProgressEvent 和 MechanicalValue 都只是分散的数据结构；**
>
> **有了 LongHorizonTaskState，它们就被组织成一个可以持续推进、可以暂停、可以恢复、可以审计的任务状态机**。

> **todo 列表只能告诉 Agent"还有哪些事没做"，但它无法说明**：
>
> - 目标为什么这样定义
> - 当前阶段验收到哪里
> - 哪些状态已经被可靠写入
> - 哪些证据支撑了上一步判断
> - **中断后应该怎样安全恢复**

> **长程任务真正需要的是一套能维持目标、状态、证据和恢复路径的执行状态结构**。

---

## 九、断了怎么接上：进度追踪的恢复

> 进度追踪不只服务当前推理，**还要服务故障恢复**。
>
> 长任务跑半小时，中途崩一次就丢一大段状态。

### 9.1 工业界的可恢复方案

| 来源 | 机制 |
|------|------|
| **LangGraph checkpointer** | 按线程给每一步存状态快照，传入同一个 `thread_id` 就能从最后一个检查点重建、从中断处续跑，还能 **time travel 回任意一步** |
| **OpenAI Agents SDK + Temporal**（2026-03 GA）| 把每一步 **journaling**，崩溃后可以精确续跑 → **预写日志 WAL 在 Agent 工作流里的现代版** |

### 9.2 SaaS 系统的对齐设计

> 锚和长期目标 → 放 store 或项目文件
> 任务 DAG 和里程碑 → 放 checkpointer
> **账本 → 走 append-only 日志**
> **机械状态 → 走 SessionState**
>
> **中断以后从 resume_packet 一次性恢复，而不是回聊天记录里去猜**。

### 9.3 现实事故演示

> 对于我们的 SaaS 系统，如果没有这一套进度追踪体系：
>
> - Agent 识别出"薪资组"任务，创建了上海市场部 6 月薪资组
> - 工具返回 `payroll_group_id`
> - LLM 在后面创建快照时则可能**误用了上月薪资组 id 或测试环境 id**
> - **异常核验看起来通过了，其实核验的是另一个批次**
> - 最终汇报写得很完整、**业务对象却串了**

> **每一步都有日志，Agent 看起来在好好干活，似乎如常在推进，最后才暴露出对象错了、月份错了、账号错了**。

### 9.4 三平面 + 三收敛器协同

> **三平面分治**：把"迷失"拆成三件事，分别由 SessionWorkspace / SessionNarrative / SessionState 三个平面负责。

> **三件事**就能稳得多：
>
> 1. **目标先冻结成契约**，里面写清 `success_criteria` 和 `non_goals`，付款和报税进 `non_goals`
> 2. **关键业务参数全部存为会话状态**（SessionState + Provenance），**不要让 LLM 在自然语言里处理 id 和金额**
> 3. **每个里程碑都配上一道验证闸门**，断点通过靠 `resume_packet` 恢复

> 如果 Agent 围着"薪资项名称规范"修了好几轮（**目标偏移**），复诵插进来一句"当前里程碑是生成快照，不做完整薪资科目治理，命名问题记入待人审清单后继续"，**就能把 Agent 从细节里拉回来**。
>
> 哨兵发现最近八个动作都在改命名、没推进快照，**给出 `recenter` 的命令**。
>
> 每个里程碑结束前都会验证闸门，快照人数、金额边界、异常清单、数据溯源、付款 blocked 状态全部检查通过，**才进入下一阶段**。

---

## 十、总结一下

### 10.1 进度追踪 vs 长任务防迷失

| 维度 | 普通进度追踪 | 长任务防迷失 |
|------|--------------|--------------|
| 回答什么 | **做到哪了** | **我还在服务原目标吗，当前里程碑推进了吗，错误放大了吗，进下一阶段前验收过吗** |
| 执行型 Agent 额外 | — | **关键业务参数还在正确的状态平面里吗** |

### 10.2 好账本 vs 流水账

> **一本好账本要能回答**：

| 维度 | 含义 |
|------|------|
| **决策（decision）** | 当时为什么这么做 |
| **证据（evidence）** | 根据什么判断 |
| **状态（state）** | 现在到哪一步了 |
| **边界（non_goals）** | 下一步不能做什么 |
| **trace** | 出了错能不能追回到是哪一笔带偏的 |

> **流水账只记"做了什么"，账本能对账**。
>
> **长任务防迷失，差的往往就是这个"能对账"**。

### 10.3 记忆模块主线收口

> 回到记忆模块的主线：
>
> - **RAG 让 Agent 在下结论前先拿到证据**
> - **进度追踪让 Agent 在长任务里始终知道自己在哪、要去哪、哪些坑不能再踩**
>
> 两个加起来，**Agent 才不会"每一步看着都合理，合在一起却跑偏"**。

### 10.4 企业落地三件事

> 真要在企业里落地，可以先挑一条**高重复、高损失**的任务族，比如**薪资快照、报销审批、合同审阅**。
>
> 然后只需要先做三件事：

| # | 动作 |
|---|------|
| 1 | 把那些会反复出问题的目标**先冻成 Goal Contract**，写清 `success_criteria` 和 `non_goals` |
| 2 | **关键业务参数全部存为会话状态**（SessionState + Provenance），**不要让 LLM 在自然语言里处理 id 和金额** |
| 3 | 每个里程碑都**配上一道验证闸门**，**断点通过靠 `resume_packet` 恢复** |

> 剩下的就是观测同类失败的复发率有没有下降，**如果持续下降，说明 Agent 在积累经验，而进度追踪有效**。

### 10.5 终极压缩

> 长程执行型 Agent 不迷失，靠的是**三平面分治**：
>
> - **锚账集稳住叙事**
> - **任务状态机稳住调度**
> - **机械状态平面稳住参数**
>
> 再用**复诵、漂移哨兵和验证闸门**持续把它收敛回原目标。

---

## 十一、精选留言

### 留言 1：街角·陌路△ — Layer 1/2/3 三层架构

> 如果从工程实现角度看，进度追踪本身可能会变得很重，甚至在一些场景里，追踪、账本、恢复和验证相关的代码量会超过业务代码本身。
>
> 所以我在想，多数业务是不是未必需要像 Claude Code 这样高度通用的智能体，而是可以采用"**简单专一的 Agent + DAG 编排 + 状态机**"的方式，把任务边界和执行路径控制得更清楚。
>
> 但这里我还有一个疑问：对于长任务来说，进度追踪更多是在执行过程中防止 Agent 跑偏。那如果任务已经跑完，并且系统后来才发现中间某一步判断错了、参数错了，或者某个里程碑其实没有真正通过验收，这时**应该如何做结果纠偏**？
>
> 这种纠偏应该依赖进度账本去追溯错误来源，然后触发补偿动作、局部回滚或从某个 checkpoint 重跑吗？还是说在 Agent 工作流里，必须把"**后验验收—定位错误—补偿执行—重新验证**"也设计成状态机的一部分？
>
> 换句话说，进度追踪是不是只能解决"别走丢"的问题，而长任务真正落地时，还需要一套类似 **Saga / 补偿事务 / 可重入执行**的结果纠偏机制？

> **作者回复**：是的。我很同意。
>
> 多数确定性工作流用 DAG + 状态机就够，Progress Tracking 的适用场景是 **Agent 有自主决策自由度、任务边界要动态发现的情况**。
>
> **完整的 Agent 长任务架构是三层结构**：
>
> | Layer | 职责 |
> |-------|------|
> | **Layer 1 前向执行** | Agent step-by-step 推进 |
> | **Layer 2 Progress Ledger** | 可追溯 —— event / decision / evidence / state_delta，同时给每步挂上 `compensate_op` 和 `idempotency_key` |
> | **Layer 3 Compensation Layer** | 可纠偏 —— Saga（正向+反向）+ Checkpoint 快照 + 幂等重放 + 后验验收 |
>
> **Progress Ledger 是错在哪，Compensation Layer 是错了怎么办**。两件事应该分开落地。
>
> **"后验验收—定位错误—补偿执行—重新验证"必须显式设计成状态机的一部分，就是 Layer 3**。
>
> 至于 Layer 3 该多重，看你业务里**不可逆动作的比例**，**不可逆动作越多（支付、外部 API）就必须越完整**。

### 留言 2：♞ — 读完后感觉还是很散

> 佳哥读完之后感觉还是很散，串不起来，和之前的文章还有很多相似，不知道怎么用；**建议加些最佳实践**。

> **作者回复**：明白的，兄弟。这是理论 + 案例，仍然还没有走到实操那一步。而且 **Gap 不小**。需要将来佳哥通过训练营的方式串起来，是一个大活计。

### 留言 3：Geek_75440d — 与语义压缩的区别

> 这个模块实现的是和语义压缩模块类似的目标么，**语义压缩也是在长上下文窗口的积累中来提取出当前主要任务之类的关键信息**？

> **作者回复**：**核心思路有相当大部分相似之处**。—— 都是力图保留最关键的信息。
>
> 目标上：
>
> - **语义压缩**是因为上下文窗口有限，**压缩时保留重要信息**
> - **进度追踪**是为了时长程任务不偏离预设的轨道，**保留关键 Milestone**

### 留言 4：Helios — Spec 怎么拆解

> 老师，用户说"**帮我把这个月薪资跑一下**"这类宽泛的话术，模型是怎么拆解为 Spec 的呢？

> **作者回复**：LLM **不是一次性**把"跑薪资"转成 spec 的，生产里这是一个有 **6-7 个阶段的多步 pipeline**，每一步都有它的工程职责：
>
> | Step | 职责 |
> | |------|
> | **1 Intent classification** | 分类 |
> | **2 Schema selection** | 选择 spec 框架 |
> | **3 Slot filling** | 槽位填充 |
> | **4 Gap detection** | 缺口检测 |
> | ... | ... |
>
> 为何如此？原因如下：
>
> 1. **不确定的问题没法靠模型变强解决**。用户没说的信息，再强的模型也猜不出 —— 解决的方法是**只能先问用户**。多阶段 pipeline 的存在就是为了在 LLM 没法回答的时候**正确地停下来问**
> 2. **多阶段允许在不同步骤用不同模型**。一次性 prompt 会迫使你用同一个最贵的模型干所有事
> 3. **可观测、可纠错**。每一步是一个独立可检查的中间态

### 留言 5：lyon — 三平面是模型自主还是人为控制？

> 里面的三平面分治是通过指令让模型自主实现还是人为控制的，这样的实现方式还能作为一个通过 agent 么，是否只适合完成特定任务？

> **作者回复**：三平面分治，我这里值得肯定是**工程控制**。**不是人，是人来设计 Agentic 的 workflow**。
>
> **我认为这是工程师拿回 Agent 控制权的唯一方法**。否则，什么东西都一股脑交给 Claude Code 的最强模型自己搞定，我们工程师设计什么？
>
> 我认为工程还是有边界的，**我们需要通过三平面分治来确定执行型 Agent 的特定任务边界**。
>
> 当然，让内容生产型 Agent 做个 PPT，搞个网站，那么不需要这样复杂的设计。

### 留言 6：森少 — 与分层记忆任务层的区别

> 佳哥，这里的任务状态记录和分层记忆的任务层是不是职能是一样的，实际使用的场景可以和分层记忆那一章结合使用吗？

> **作者回复**：是的是的，非常相似，**可以结合起来使用**。

### 留言 7：Helios — 五种验证机制（账本质量）

> "发生了什么（event）、为什么这么做（decision）、根据什么判断（evidence）、状态发生了什么变化（state_delta）"
>
> 老师，这些问题是让LLM自己回答出来么，那怎么 check 那表达的正确和错误呢？

> **作者回复**：不完全是。
>
> **架构原则**：**让 LLM 做决策，但记录"世界上实际发生了什么"必须由确定性 capture，不能让 LLM 叙述**。
>
> | 字段 | 谁负责 |
> |------|--------|
> | **event（发生了什么）** | **不该 LLM 负责，这个是事实** |
> | **decision（做了什么决策）** | **LLM 从 schema 约束的 enum 里选一个，不能自由发明新动作类别** |
> | **evidence（依据是什么）** | **LLM 提出 + wrapper 验证** |
> | **state_delta（状态变化）** | **不该 LLM 负责，这个是程序工作流** |
>
> 按可靠性从高到低，**五种验证机制**：
>
> 1. **System-of-record cross-reference**（最可靠）—— event 字段必须能和底层工具的真实响应对得上。工具 wrapper 在调用之外独立 log 一份原始 response，**账本写入前比对**
> 2. **引用锚定验证**（适用于 evidence）—— LLM 给的引用形如 `{doc_id, paragraph_id, claimed_text}`。Wrapper 去对应源读出实际文本，跟 claimed_text 做匹配
> 3. **Diff-based state_delta**（避免 LLM 撒谎的最彻底方法）—— 动作前 snapshot 一次系统状态、动作后再 snapshot 一次、**用代码计算两次 snapshot 的差集**
> 4. **Schema 约束的 decision enum** —— decision 字段不开放写，是一个有限枚举，**LLM 必须从这个集合里选**
> 5. **Trip-wire 测试**（持续验证机制）—— 往生产里插入**已知场景**，**观察实际账本和理想账本的偏差**

### 留言 8：元气🍣 🇨🇳 — 像人一样不忘初心

> 像人一样，时刻不要忘记初心。这个通过工程化落实。

> **作者回复**：是的。类人体需要像人一样设计。

---

## 十二、思考题

> 回到你自己的系统，找一个**最容易跑偏的长任务**，做一次小审计。

### 题 1：Goal Contract 与 non_goals

> 这个任务的 **Goal Contract** 写得出来吗？`non_goals` 有没有写？
>
> Agent 最近一次跑偏，是不是做了一件"**看起来有用、当前却不该做**"的事？

### 题 2：账本 vs 流水账

> 你的进度记录是**流水账还是能对账的账本**？
>
> 最近一次错误放大，能不能从账本追回到是**哪一个早期决策带偏**的？
>
> 哪些关键参数还在让 LLM 从上下文里复制和传递？

### 题 3：断点恢复包

> 如果今天断掉，明天新 session 能不能只靠一个 **`resume_packet`** 就接上，而不是去翻聊天记录？

---

## 十三、下一讲预告：失败日记

> 下一讲我们讲**记忆模块的第四个模式——失败日记（Failure Journals）**。
>
> - 进度追踪记录的是**任务怎么往前走**
> - 失败日记记录的是**任务在哪里摔倒过**
>
> 长任务里，**失败本身就是很重要的信号**。
>
> 如果错误 trace 被清掉、失败原因没分类、补救动作没写下来，**下次类似任务来了，Agent 还会从头摔一遍**。
>
> 下一讲，我们就看**怎么把失败也做成可召回的经验**。

→ [[lession17-失败日记]]

---

## 十四、参考资料

> | # | 来源 |
> |---|------|
> | 1 | Hector Garcia-Molina, Kenneth Salem. **Sagas**. ACM SIGMOD 1987. |
> | 2 | C. Mohan et al. **ARIES: A Transaction Recovery Method Using Write-Ahead Logging**. ACM TODS, 1992. |
> | 3 | Martin Fowler. **Event Sourcing**. 2005-12-12. |
> | 4 | Sumers et al. **Cognitive Architectures for Language Agents (CoALA)**. arXiv:2309.02427, 2023. |
> | 5 | METR. **Measuring AI Ability to Complete Long Software Tasks**. arXiv:2503.14499, 2025-03. |
> | 6 | Arike et al. **Technical Report: Evaluating Goal Drift in Language Model Agents**. arXiv:2505.02709, 2025-05. |
> | 7 | Liu et al. **Lost in the Middle: How Language Models Use Long Contexts**. TACL 2024, arXiv:2307.03172. |
> | 8 | Yichao Ji (Manus). **Context Engineering for AI Agents: Lessons from Building Manus**. 2025-07-18. |
> | 9 | **The Long-Horizon Task Mirage? Diagnosing Where and Why Agentic Systems Break**. arXiv:2604.11978, 2026-04. |
> | 10 | Microsoft Research. **Magentic-One: A Generalist Multi-Agent System for Solving Complex Tasks**. 2024-11. |
> | 11 | Anthropic. **How we built our multi-agent research system**. 2025-06-13. |
> | 12 | LangChain. **LangGraph Persistence**. |
> | 13 | Temporal. **Announcing OpenAI Agents SDK Integration**（GA 2026-03-23）. |
> | 14 | Anthropic. **Todo Lists / Task tools (Claude Agent SDK)**. |

---

## 附：本讲核心要点速查

| 关键词 | 含义 |
|--------|------|
| **Progress Tracking** | 长任务里的防迷失机制，由协调者横切维护任务台账 |
| **双轴定位** | 记忆 × 编排（不是链式）|
| |
| **五种迷失** | 目标漂移 / 状态漂移 / 错误放大 / 细节过载 / 完成幻觉 |
| **三平面分治** | SessionWorkspace（调度 DAG）/ SessionNarrative（叙事 锚账集）/ SessionState（机械真值）|
| **三个收敛器** | 复诵 Recitation / 漂移哨兵 Drift Watchdog / 验证闸门 Verification Gate |
| |
| **锚 GoalContract** | goal_id / user_goal / success_criteria / non_goals / constraints |
| **账 ProgressEvent** | event / decision / reason / evidence_refs / state_delta / next_action（append-only）|
| **集 working_collection** | 每一步投影给模型的当前最相关材料 |
| |
| **七个对象** | GoalContract / Milestone / ProgressEvent / MechanicalValue / DriftSignal / LongHorizonTaskState / resume_packet |
| **`recitation_prompt()`** | 把目标锚+里程碑+下一步推回上下文尾部 |
| **`resume_packet()`** | 中断后新一代 Agent 不用翻聊天记录就能接上 |
| |
| **机械态字段** | key / scope / value_ref / provider / runtime_layer / trust |
| **`mechanical_state_refs`** | 依赖哪些机械状态，但不交给 RAG 生成 |
| |
| **三种状态机** | pending / in_progress / blocked / needs_review / needs_rework / completed |
| **四种漂移等级** | ok / watch / recenter / pause |
| **四个哨兵分数** | goal_relevance / milestone_progress / evidence_health / error_pressure |
| |
| **演进血脉** | Saga(1987) → ARIES(1992) → Fowler Event Sourcing(2005) → CoALA(2023) → Magentic-One(2024) → Anthropic SubAgents(2025) |
| **断点恢复三方案** | LangGraph checkpointer / Temporal durable execution / resume_packet |
| |
| **三层架构（留言 1）** | Layer 1 前向执行 / Layer 2 Progress Ledger（带 compensate_op + idempotency_key）/ Layer 3 Compensation Layer |
| **账本质量五种验证** | System-of-record cross-reference / 引用锚定验证 / Diff-based state_delta / Schema 约束 enum / Trip-wire 测试 |
| **金句** | 「进度追踪不是把上一步结果简单传给下一步，而是由编排器持续维护整份任务台账」 |