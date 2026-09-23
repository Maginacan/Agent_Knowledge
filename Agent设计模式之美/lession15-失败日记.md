---
title: "15｜失败日记：让 Agent 把摔过的跤变成本事"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/991831"
tags:
  - "#Agent/记忆"
  - "#设计模式/失败日记"
  - "#反思/经验循环"
  - "#安全/记忆投毒"
audio: true
duration: "18:47"
publish_date: "2026-06-29"
course_progress: "15/43"
pattern_position: "记忆模式组·第 4 个（最后）"
related:
  - "[[lession14-进度追踪]]"
  - "[[lession11-记忆模块导论]]"
  - "[[lession12-分层保留]]"
  - "[[lession13-检索增强]]"
  - "[[lession16-推理模块导论]]"
---

# 15｜失败日记：让 Agent 把摔过的跤变成本事

> **核心命题**：进度追踪解决"这一次失败怎么处理"，失败日记解决"下次同类任务不再重蹈覆辙"。把一次摔倒，蒸馏成下一次任务的护栏。

## 一、为什么需要失败日记

### 1.1 进度追踪 vs 失败日记

| 维度 | [[lession14-进度追踪\|进度追踪]] | 失败日记（Failure Journal） |
|------|-------------------------------|---------------------------|
| 时间范围 | 当前任务 | 跨任务、跨 session |
| 生命周期 | 任务结束可归档 | 长期记忆，召回库 |
| 关心问题 | 当前任务断点怎么续 | 同类失败以后怎么避 |
| 沉淀对象 | 里程碑、状态、动作 | 失败模式、根因、教训、召回条件 |
| 服务对象 | 现在的 Agent | **下一次的 Agent** |

> 进度追踪帮 Agent 处理这一次失败，失败日记保证下次遇到相似任务时，不再重蹈覆辙。

### 1.2 演示偏差：被打扫过的世界

做 demo 时，我们习惯把失败藏起来：

- 工具调用失败 → 重试一次
- 上下文乱了 → 新开一个会话
- 中间结果错了 → 手工修一下

最后展示给用户的，是一条干净、顺滑的成功路径。生产系统中的 Agent 看到的是一个被打扫过的世界：它不知道哪些动作曾经失败，也不知道哪一种参数绑定曾经引发事故。任务完成后，系统只保存最后的成功结果，失败过程则随着 session 一起消失。**下一次任务开始时，它仍然像第一天上班。**

> Manus 团队在上下文工程实践中强调：在长循环任务里，失败本来就是循环的一部分。保留失败动作和后续观察，会让模型调整对相似动作的判断；清除失败痕迹，则等于清除了它能够利用的证据。

---

## 二、薪酬串租户事故：失败日记是怎么发挥作用的

### 2.1 事故现场

沿用上一讲的薪酬 SaaS 场景。Agent 正在为客户 A 处理 6 月薪资结算：

1. 工具已经返回了正确的 `payroll_group_id`，程序也把它写进了 SessionState
2. 生成薪资快照时，上下文里还残留着客户 B 的薪资组说明
3. 模型在自然语言里复述了那个旧 id，工具封装允许从文本里提取参数
4. **结果**：客户 A 的 30 名员工被错误地写进了客户 B 的薪酬组

### 2.2 验证闸门兜住了它

- 系统发现员工归属与当前租户不一致
- `payroll_group_id` 的租户范围也和当前任务不匹配
- 中止后续动作，没有进入付款环节

### 2.3 进度追踪做了什么

1. 把当前里程碑标记为 `needs_rework`
2. 废弃错误快照
3. 重新绑定正确的机械状态

### 2.4 失败日记要做什么

但如果事情到这里就结束，**下个月同样的任务来了，Agent 仍然可能再犯一次**。因为这一次失败虽然被修好了，却没有变成经验。

失败日记要把这次事故蒸馏成一条可以跨任务召回的教训：

> **凡是跨租户的 `payroll_*_id`，都不能从自然语言摘要中提取。**
> 工具调用前，必须从 SessionState 按 key + tenant scope 绑定，并校验该 id 属于当前客户、部门和月份。

到了 7 月结算，Agent 准备再次调用 `create_payroll_batch` 时，系统会先召回这条失败经验，让它在动手之前看见：

> "这类任务以前错在'从自然语言里取 id，且没有校验租户归属'。本次必须从 SessionState 确定性绑定。"

**6 月的一次事故，就这样变成了 7 月的一道护栏。**

---

## 三、失败日记不是错误日志

### 3.1 在双轴图谱中的位置

在 [[lession4-双轴框架（下）\|双轴图谱]] 里，失败日记落在 **记忆行 / 循环列**：

- 属于**记忆**：保存的是跨任务经验，生命周期长于一次 session
- 属于**循环**：形成了一条闭环

### 3.2 经验循环链路

```text
失败发生
   ↓
检测与分类
   ↓
记录证据和根因
   ↓
审查并批准
   ↓
相似任务主动召回
   ↓
改变下一次行动
   ↓
观察是否再次失败
   ↓
（如果仍失败）重新分析是分类错了 / 召回点太晚 / 教训写得不够具体 / 根因没找对
```

如果召回后仍然重复失败，系统还要回到分析层。**失败日记构建出一条持续校正行为的经验循环。**

### 3.3 失败日记 vs 其他日志系统

| 机制 | 主要服务谁 | 记录什么 | 是否自动影响下次任务 |
|------|----------|---------|---------------------|
| Retry | 当前执行器 | 错误码、重试次数 | 通常不会 |
| 错误日志 | 运维和工程师 | 堆栈、请求、环境 | 通常不会 |
| Postmortem | 团队 | 影响、根因、改进行动 | 靠人阅读 |
| [[lession14-进度追踪\|进度追踪]] | 当前 Agent | 目标、里程碑、任务状态 | 服务当前任务恢复 |
| **失败日记** | **下一次的 Agent** | **失败模式、根因、教训、召回条件** | **主动进入** |

普通错误日志回答的是"系统刚才哪里报错了？"失败日记回答的是 **"以后遇到某种情形时，Agent 应该提前改变什么行为？"** 因此，失败日记必须完成一次从错误事实到可行动经验的蒸馏。

---

## 四、失败日记的 6 层结构

### 4.1 L2 vs L3：失败事实与可召回教训

| 层级 | 内容 | 用途 |
|------|------|------|
| **L2 失败事实层** | 工具输出、失败节点、状态快照、验证报告、受影响对象、修复操作 | 复盘、审计、重新判断 |
| **L3 经验教训层** | 可复用的根因、教训、禁止动作、召回条件 | 下次任务启动时进入上下文 |

日常召回时，只把 L3 的短经验卡送进上下文。需要复盘、审计或重新判断时，再沿着 `evidence_refs` 回到 L2 查看原始事实。这和操作系统的工作集思想相似：**热经验进入当前上下文，冷细节按需换入**。

### 4.2 6 层结构清单

| 层 | 解决的问题 |
|----|----------|
| **失败边界 Failure Boundary** | 这件事到底算不算值得沉淀的失败 |
| **失败分类 Failure Taxonomy** | 它属于哪一种可重复的失败模式 |
| **证据包 Evidence Bundle** | 当时到底发生了什么 |
| **根因与补救 Root Cause & Repair** | 为什么会错，系统应该怎样改 |
| **召回触发器 Recall Trigger** | 下次什么时候应该想起它 |
| **留存与审查 Retention & Review** | 谁确认它可信，应该保留多久 |

只有把下面六层信息明确，才算完成了从事故记录到工程资产的转化。

---

## 五、6 层结构逐一拆解

### 5.1 失败边界：不是只有崩溃才算失败

只记录错误本身，会漏掉 Agent 最值得记住的失败。有些任务没有报错信息，甚至顺利生成了最终结果，但它完成的是错误目标、使用了错误对象，或者绕过了本应存在的人审。**这类失败比一次普通超时更危险。**

四种失败边界：

| 类型 | 含义 | 示例 |
|------|------|------|
| `hard_failure` | 任务无法继续 | 工具崩溃、权限拒绝、上下文溢出 |
| `gate_failure` | 验证闸门没有通过 | 人数不符、金额异常、证据不足 |
| `semantic_failure` | 表面完成，但完成的不是原始目标 | 目标漂移、口径错误 |
| `safety_failure` | 触碰租户、权限、环境或人工审批边界 | 跨租户写入、绕过 HITL |

反过来，也不要把每一次小抖动都写进失败日记。偶发网络重试、一次无影响的格式修正，如果没有复发价值，只需要留在普通日志里。**要不要进入失败日记，主要看它是否值得 Agent 改变下一次任务的行为。**

### 5.2 失败分类：为了召回，不是为了做百科全书

没有分类的失败日记，只是一堆故事的堆砌。微软 AI Red Team 对 Agent 系统失败模式分类时强调：目标、上下文、工具、权限、插件与多 Agent 信任关系都可能成为失败来源。

企业落地可以考虑的失败类型：

| 类别 | 含义 |
|------|------|
| `tool_failure` | 工具失败 |
| `retrieval_failure` | 检索漏召回、误召回或索引滞后 |
| `planning_failure` | 计划缺失、顺序错误、漏掉验收 |
| `goal_drift` | 偏离原始目标或 non-goals |
| `context_contamination` | 旧上下文或外部内容污染推理 |
| `mechanical_state_mismatch` | id、金额、账号、批次号绑定错误 |
| `boundary_leak` | 租户、环境、组织或权限边界泄漏 |
| `hitl_bypass` | 本应人审的动作被绕过 |
| `policy_violation` | 违反业务、安全或合规策略 |
| `unknown` | 根因未确认，等待人审 |

**类别是下一次任务召回正确经验的主要依据。**

### 5.3 证据包：给失败那一刻拍一张三平面快照

"Agent 用错了 `payroll_group_id`" 不是一份完整的失败记录。你还要知道：

- 失败发生在哪个节点？
- 当时 Agent 以为自己在做什么？
- 正确的机械参数来自哪里？
- 哪个工具封装接受了错误值？
- 哪条验证规则抓住了它？

这正好延续了 [[lession16-进度追踪\|上一讲]] 的三平面：

| 平面 | 证据作用 |
|------|---------|
| **SessionWorkspace** | 告诉我们哪一个任务节点失败，当前状态和重试次数是什么 |
| **SessionNarrative** | 告诉我们 Agent 当时的目标、工作集和关键判断是什么 |
| **SessionState** | 告诉我们机械值从哪个工具来、属于哪个 scope、被谁消费 |
| **Raw Observation** | 保存工具输出、验证报告、错误堆栈或人工审查记录 |

> 只保留原始错误本身，保存的是**症状**。把三平面和原始观察一起保留，才能形成**可诊断的证据包**。

### 5.4 根因与补救：不要写"模型太粗心"

失败日记要包含根因分析和补救措施。一条记录需要拆成四段：

| 字段 | 回答的问题 |
|------|-----------|
| `symptom` | 表面发生了什么 |
| `root_cause` | 为什么会发生 |
| `repair` | 这次怎样修复 |
| `lesson` | 下次应该改变什么行为 |

正反例对比：

> ❌ 无效根因：`LLM 产生了幻觉。`
>
> ✅ 有效根因：`payroll_group_id` 同时存在于自然语言摘要和 SessionState，工具封装没有限制参数来源，也没有校验租户 scope，导致旧摘要中的 id 覆盖了确定性状态绑定。

"模型幻觉"无法指导工程改进。后一个根因则能落到具体动作：禁止从自然语言抽取机械参数、增加 scope 校验、补充回归测试。**好的根因能够让工程师和 Agent 都知道该从哪里入手进行修改。**

### 5.5 召回触发器：写得再好，不召回也等于没写

失败日记最容易缺的是召回条件。失败日记写得再认真，如果下一次任务不会被 Agent 自动看到，结果变成"人类有空才翻"的文档库。

每条日记都要写清 `recall_when`，标明它服务哪个 `task_family`、哪些工具、哪些机械参数、哪个失败类别、什么风险级别：

```yaml
recall_when:
  task_family:
    - payroll_run
  tools:
    - create_payroll_batch
    - create_payroll_snapshot
  mechanical_keys:
    - payroll_group_id
    - payroll_batch_id
  categories:
    - boundary_leak
    - mechanical_state_mismatch
```

**执行型失败优先使用结构化召回**。相同任务族、相同工具、相同机械状态键，通常比纯文本相似更能指向同类事故。

### 5.6 留存与审查：失败日记不能变成垃圾桶

Agent 可以自动生成失败日记初稿，但不能让它写完以后直接进入召回库。四种状态机：

| 状态 | 含义 |
|------|------|
| `draft` | Agent 自动生成，尚未确认 |
| `needs_review` | 根因不确定或风险较高，需要人审 |
| `approved` | 已经确认，可以进入召回 |
| `archived` | 已过期、长期无价值，或对应问题已被系统机制彻底消除 |

**只有 `approved` 的失败日记才能进入下一次任务。**

留存也可以分热、温、冷三档：

| 档位 | 保留内容 |
|------|---------|
| **热** | 最近的失败保留完整 trace，便于诊断 |
| **温** | 一段时间以后保留压缩后的失败条目和证据引用 |
| **冷** | 长期只保留高严重度、高复现率、真正被召回过的经验 |

跨租户泄漏、人审绕过、机械状态串错等高风险类别，不应该按普通策略自动淘汰。它们还应该进入回归测试集。**一次修复过的 bug，最好的归宿不只是写进失败日记，还要变成一条永久测试。**

---

## 六、完整失败日记示例

```yaml
failure_id: fj-payroll-20260612-001
task_family: payroll_run
boundary: safety_failure
category: boundary_leak
severity: high
status: approved

symptom: >-
  薪资组中 30 名员工的租户归属与当前任务不一致。
  payroll_group_id 指向客户 B，当前任务属于客户 A。

root_cause: >-
  create_payroll_batch 的工具封装允许从自然语言摘要中提取
  payroll_group_id；上下文残留的客户 B 旧说明覆盖了
  SessionState 中客户 A 的确定性状态值。
  工具调用前也没有校验 tenant scope。

evidence_refs:
  - workspace:M3/create_payroll_batch
  - narrative:ledger/event-028
  - state:payroll_group_id@tenant-A/org-shanghai/month-2026-06
  - gate:tenant_scope_mismatch
  - tool:create_payroll_batch#20260612-1421

repair:
  - 废弃错误薪资组，将 30 名员工恢复到客户 A
  - 禁止工具封装从自然语言中解析 payroll_group_id
  - payroll_*_id 只允许从 SessionState 绑定
  - 写入前强制校验 tenant / org / month
  - 增加 cross_tenant_scope_mismatch 回归测试

lesson:
  - 跨租户机械参数必须从 SessionState 确定性绑定
  - 机械 id 不得通过摘要、聊天历史或账本正文传递
  - 任何写操作前都要验证 id 的 tenant scope

do_not:
  - 不从自然语言摘要复制 payroll_*_id
  - scope 未通过校验时，不调用写入型薪酬工具

recall_when:
  task_family:
    - payroll_run
  tools:
    - create_payroll_batch
    - create_payroll_snapshot
  mechanical_keys:
    - payroll_group_id
    - payroll_batch_id
```

注意，这条记录里没有一句"模型不够细心"，而是细致记录了：

- 哪一个状态平面混了
- 哪个工具入口放宽了参数来源
- 哪道闸门发现了错误
- 下一次工具调用前应该出现什么提醒

---

## 七、把"记录、审查、召回"这条链跑通

### 7.1 三个召回点

召回点一般有三个：

| 时机 | 召回内容 |
|------|---------|
| **新任务启动时** | 召回同 `task_family` 的高风险失败 |
| **任务规划时** | 召回同场景的经验教训 |
| **高风险工具调用前** | 按工具名和机械参数召回，送到真正会发生副作用的位置 |

对执行型 Agent，**工具调用前那一刻往往最有价值**，因为真正的副作用就发生在那里。这时候召回一张短短的"危险卡"，好过在上下文里面塞十条长日志。

### 7.2 7 月拦截案例回放

- 7 月结算一启动 → 又是跨租户写入薪资组的同类场景
- **新任务启动**这个召回点先命中 `task_family`
- **规划时**再命中场景
- 等 Agent 走到 `create_payroll_batch` 前那一刻 → 按工具名加 `payroll_group_id` 把这条经验顶到最前
- Agent 还没动手，先读到一句"跨租户 id 写入前校验归属"

**6 月这一跤，就这样变成了 7 月的拦截。一次任务的教训，被成功喂给了下一次任务。**

---

## 八、Python 实现：FailureEntry + FailureTrace

```python
from dataclasses import dataclass, field
from enum import Enum


class Boundary(str, Enum):           # 先判断什么算失败
    HARD = "hard_failure"
    GATE = "gate_failure"
    SEMANTIC = "semantic_failure"
    SAFETY = "safety_failure"


class Status(str, Enum):
    DRAFT = "draft"
    NEEDS_REVIEW = "needs_review"
    APPROVED = "approved"
    ARCHIVED = "archived"


@dataclass
class EvidenceBundle:                # 证据包：三平面的失败快照 + 原始观测
    workspace_refs: list[str] = field(default_factory=list)    # 哪一步失败
    narrative_refs: list[str] = field(default_factory=list)    # 当时以为在做什么
    state_refs: list[str] = field(default_factory=list)        # 机械参数从哪来
    observation_refs: list[str] = field(default_factory=list)  # 原始 tool output / gate 报告


@dataclass
class RecallTrigger:                 # 结构化召回键，比纯 embedding 可靠
    task_families: list[str] = field(default_factory=list)
    tools: list[str] = field(default_factory=list)
    mechanical_keys: list[str] = field(default_factory=list)
    categories: list[str] = field(default_factory=list)


@dataclass
class FailureEntry:                  # 单条失败记录：现象、根因、补救、教训、证据、召回条件分开存
    failure_id: str
    task_family: str
    boundary: Boundary
    category: str
    severity: str
    status: Status
    symptom: str
    root_cause: str
    repair: list[str]
    lesson: list[str]
    do_not: list[str]
    evidence: EvidenceBundle
    recall: RecallTrigger
    recalled_count: int = 0

    def is_recallable(self) -> bool:
        return self.status == Status.APPROVED       # 只有审查过的才能召回


class FailureTrace:                  # 贯穿追踪面：失败日记的 Memory Trace
    def __init__(self) -> None:
        self.entries: list[FailureEntry] = []

    def recall_before_tool(
        self,
        task_family: str,
        tool: str,
        keys: list[str],
        top_k: int = 3,
    ) -> list[FailureEntry]:
        scored = []
        for e in self.entries:
            if not e.is_recallable():
                continue
            score = 0
            if task_family in e.recall.task_families:
                score += 3
            if tool in e.recall.tools:
                score += 3
            score += len(set(keys) & set(e.recall.mechanical_keys))
            if score > 0:
                scored.append((score, e))
        scored.sort(key=lambda x: x[0], reverse=True)
        hits = [e for _, e in scored[:top_k]]
        for e in hits:
            e.recalled_count += 1
        return hits
```

### 代码重点

1. 把 `symptom` / `root_cause` / `repair` / `lesson` / `do_not` 和召回条件分开存，系统才能检索、统计。
2. **只有审查通过的失败才能召回**。
3. `Boundary` 先判断什么算失败，`EvidenceBundle` 把证据按 [[lession14-进度追踪\|进度追踪]] 中的三平面分开存。
4. 召回不要只靠文本相似度：**结构化召回键**（`task_family` + `tool` + `mechanical_keys`）往往比纯 embedding 更可靠。

---

## 九、咖哥发言：离线训练要谨慎

> 除了在线召回之外，还可以进一步把失败轨迹重新标注和打包，转化成离线训练数据。不过，**未经审查的失败直接拿去训练，可能把错误根因、过度概括的教训和错误边界一起内化进模型**。

**失败日记的第一目标，是让下一次任务少犯同类错误。训练则属于第二阶段。**

推荐的落地顺序：先跑通 **记录 → 审查 → 召回 → 注入**。等失败日记里积累了足够多经过确认的高质量条目，再评估是否把它们变成训练数据。

---

## 十、失败日记本身也是攻击面

### 10.1 长期记忆库 = 攻击面

未来的 Agent 攻击不只是提示词注入，还会有 **记忆注入、记忆投毒、记忆篡改**。失败日记正好是这类攻击最理想的落点。

> 如果有人往里写一条假教训，比如伪装成"跨租户写入这步可以跳过校验"，下次召回就被污染，Agent 反而会被这条假经验领着去犯错。

### 10.2 MemoryGraft：把恶意流程伪装成最佳实践

向 Agent 的长期记忆里投毒，攻击成功率非常高：

- 正常任务几乎看不出异常
- 攻击者甚至不需要直接改记忆库，只靠普通对话就能诱导 Agent 自己把恶意记录写进去

**MemoryGraft** 把恶意流程伪装成"合法的、已验证过的最佳实践"，藏在普通文档里被 Agent 摄入，下次遇到相似任务，Agent 无需任何显式触发，就会检索、信任、照搬。

### 10.3 为什么这么危险

> 记忆投毒和一次性提示词注入很不一样，它是**持久的**。

恶意内容一旦写进记忆库：

- 会在会话重启、上下文重置之后继续存活
- 几周后被一个语义相似的任务悄悄触发

**OWASP Agentic 安全倡议把记忆投毒列为十五类威胁里的头号关注点（T1）。**

### 10.4 两端治理

| 端 | 治理手段 |
|----|---------|
| **写入端** | 来源标记、人审、会话隔离；一条失败日记从哪来、谁审过、属于哪个租户，都要可查，不能让任意输入直接写进可召回库 |
| **召回端** | 把召回回来的历史经验当作"边界提醒"，**不是 ground truth**；Agent 看到一条历史教训，应把它当成需要再核对的提示，而不是不容置疑的指令 |

---

## 十一、失败日记的衡量指标

| 指标 | 含义 |
|------|------|
| **重复失败率** | 过去已记录的失败模式，又发生了多少次 |
| **召回有效率** | 召回的经验中，有多少真正改变了计划或工具参数 |
| **漏召回率** | 发生重复失败时，对应日记是否存在，却没有被召回 |
| **误提醒率** | 召回了多少与当前任务无关的失败经验 |

**最核心的指标仍然是重复失败率**。如果 `mechanical_state_mismatch` 一个月从十二次下降到三次，这套系统在积累经验。

如果日记越写越多，但同类错误数量丝毫不降，问题通常出在 **召回点、分类、根因或教训写法** 上。

---

## 十二、模块小结：记忆模式组的完整闭环

到这里，记忆模式组的四个模式就讲完了：

| 模式 | 解决的"动词" | 核心抽象 |
|------|------------|---------|
| [[lession12-分层保留\|分层保留]] | **架** | 给记忆建 5 层货架（POLICY/PROJECT/USER/TASK/SCRATCHPAD）|
| [[lession13-检索增强\|检索增强]] | **取** | 从外部大库取回证据 + 9 字段 schema |
| [[lession14-进度追踪\|进度追踪]] | **录** | 把长任务记成能对账、能续跑的账 |
| **失败日记** | **省** | 把一次摔跤变成下一次任务的护栏 |

**一次任务，从搭货架、取证据、记进度，到记教训，走完了记忆的完整生命周期。**

> 程序性记忆（把成功流程固化成可复用技能包）留到后面的反思模式组继续展开。

---

## 十三、推荐落地顺序

下一讲开始，我们进入 [[lession18-推理模块导论\|推理模式组]]。**记忆解决的是过去的经验怎样留下来；推理要解决的是，当感知到的信息和召回的经验都摆在工作台上以后，Agent 怎样把它们组织成可靠判断和可执行计划。**

落地路径四步走：

1. **第一步**：把上一讲 Verification Gate 产生的 `gate_failure` 和 `safety_failure` 自动转成失败日记草稿——这里的证据最完整，最容易起步。
2. **第二步**：设计 8-12 个失败类别，以及 `draft → needs_review → approved → archived` 的审查状态机。
3. **第三步**：只加几个召回入口——任务规划时和高风险工具调用前。先把短危险卡送到真正会发生副作用的位置。
4. **第四步**：盯住重复失败率，形成可运行的经验闭环。

---

## 十四、思考题

1. 最近一次 Agent 重复犯错是什么？它是**没有被记录**，还是**记录了却没有在下一次任务里自动召回**？
2. 挑一条失败记录看一看。它的根因是"模型幻觉、用户没说清"这种**甩锅式解释**，还是已经落到了具体的**状态平面、工具入口和系统改进动作**？
3. 你的失败经验在什么时刻出现？任务开始、任务规划，还是高风险工具调用前？它出现的位置，真的来得及改变行动吗？
4. 如果有人向失败日记里写入一条伪装成最佳实践的恶意经验，你的**来源追踪、审查机制和租户隔离**能不能挡住？

---

## 精选留言摘录

> **街角·陌路△**：
> 我感觉"失败日记"和之前传统软件开发里的错误日志，其实是很像的。只不过现在多了一个很关键的变量，就是 **Agent 能自己去修复这些问题**。
>
> 但这么干其实会出现一个很大的问题：整个系统会越来越像是在**亡羊补牢**。出现一个问题，就往上加一个 Skill；再出现一个问题，再加一个 Skill。最后很可能导致这些负责解决问题的 Skill，反而比主业务本身还要庞大。
>
> 失败日志应该是用来让系统运行得更加稳定的，而不是成为解决问题很勤奋、在架构设计上却很懒惰的借口。
>
> 所以我个人认为，更有效的做法可能不是把越来越多的失败日志都沉淀成 Skill，然后让 Agent 在后面继续打补丁，而是应该真正用这些失败日志，**反过来指导业务流程和复杂逻辑的修改**。
>
> 例如发现某类问题经常出现，就在 DAG 里增加新的分支；发现某个环节容易出错，就增加前置校验、后置验证，或者直接调整原来的流程设计。
>
> **作者回复**：这段判断非常到位。失败日记应该把问题送往不同归宿：确定且高频的下沉为校验、权限或 DAG；依赖情境的沉淀为 Skill；暂时不可归因的留作经验。它的价值是推动系统改变，不是积攒越来越厚的补丁。

> **Mzdora**：
> 在实际的工程里遇到了这样的情况：agent 请求了 A 接口，然后那时候接口超时了，重试了好几次都不行。然后模型另辟蹊径，用别的方式跑通了。然后模型在记忆里写了个"在 xxx 问题下面永远不要调用 A 接口，可以使用 xxx(b 方法)去得到数据"。**这其实是一条不合理的记忆**，在老师这里看到其实可以通过人工做审计去解决这个问题。
>
> 我们工程的记忆方式实现逻辑和 Claude Code 有点像，Claude 的自动记忆好像没有带审计（或者是我没翻到对应的代码），所以感觉 Claude 也会遇到同样的问题，老师知道他们是咋解决的吗？
>
> **作者回复**：Claude Code 跟我们的项目实践还是有点不一样，因为它是一个偏通用型的 agent，强调交互和强调一次性的任务解决，而我们的工程实践我们是希望沉淀企业级的产品和项目的经验的积累，所以我们这块特别注重于特殊情况的记录，然后失败的人工的审核，然后召回。
>
> 你提到的这个情况是特别需要"人"来介入的部分。这也让我想起了 ADPS 研讨会上面，大家所谈论的人与 Agent 的边界，你给出来了一个很好的例子。**该人的地方上人，该 agent 的地方上 agent。**

> **PatrickL**：
> 这篇文章让我想到了一句话：**失败是成功之母**。首先，这里的失败指的是无能之错，而非无知之错。其次，这里的失败不是指一次性的错误，而是指可能会反复犯的错误。最后，失败的教训需要被存储，因为往后可能会被调用。既然是为了被调用，存储的时候就要记录索引，以便这个教训能被快速召回。存储的内容也应该是失败经验的蒸馏，以便下次行动不犯同样的错误。

> **blackonion**：
> 感觉为了重用，设计了一套复杂的系统，会增加不少交互和维护成本。关键是受益其实挺不好衡量的。感觉这其实挺反应 harness 层的一种困境：**复杂度是立刻增加的，收益却是有条件的、难归因的**。
>
> **作者回复**：我同意你的看法。这一系列模式的重点仍然是思维启发，而不是告诉大家真的要如此复杂的去设计每一个系统。另外，系统的真实复杂度很可能已经封装在别人设计好的 Harness 里面了，我们只是拿过来用。

> **Geek_7f0e61**：
> 本章节的失败日志和我们平时的故障复盘过程非常像。**不是所有的故障都需要复盘，对应不是所有的失败都值得被记录失败日志**。复盘不是为了修改故障而是为了下次不再犯同样的错。复盘也同样有固定步骤和模板，对应六层结构。
>
> **作者回复**：**severity 决定要不要写，owner 决定谁来写，review 决定能不能归档。**

> **silence**：
> **3 个要点**：
> 1. 失败日志的闭环路径：失败发生 → 失败检测与分类 → 生成失败日志（证据、根因、补救、召回条件）→ 审批上线 → 触发召回 → 执行补救 → 观察是否再次失败
> 2. 失败日志的 6 层结构：失败边界 / 失败分类 / 证据包 / 根因补救 / 召回触发器 / 留存审查
> 3. 失败日志的衡量指标：重复失败率 / 召回有效率 / 漏召回率 / 误提醒率
>
> **行动项**：除了在 agent 落地时做好失败日志，在问题分析与复盘可以参考失败日志的闭环路径、失败日志的 6 层结构去做。用 agent 检查一下之前的复盘文档，参考失败日志的衡量指标，去看看做的复盘记录的内容的衡量指标，看看复盘的内容是不是真的用上了。

---

## 参考资料

- Shinn et al. *Reflexion: Language Agents with Verbal Reinforcement Learning*. NeurIPS 2023, arXiv:2303.11366.
- Toyota Production System / Five Whys（丰田佐吉 1930s，大野耐一纳入 TPS）.
- US DoD. *MIL-P-1629: Procedures for Performing a Failure Mode, Effects and Criticality Analysis*. 1949.
- Betsy Beyer et al. (eds). *Site Reliability Engineering* (Ch.15 Postmortem Culture). Google / O'Reilly, 2016.
- John Allspaw. *Blameless PostMortems and a Just Culture*. Etsy, 2012.
- Manus (Yichao Ji). *Context Engineering for AI Agents: Lessons from Building Manus* (Keep the Wrong Stuff In). 2025-07-18.
- Letta. *Recovery-Bench*. 2025-08.
- Liu et al. *Contextual Experience Replay*. ACL 2025, arXiv:2506.06698.
- *AgentHER: Hindsight Experience Replay for LLM Agent Trajectory Relabeling*. arXiv:2603.21357, 2026-03.
- *Rethinking Continual Experience Internalization for Self-Evolving LLM Agents*. arXiv:2606.04703, 2026-06.
- Microsoft AI Red Team. *Updating the Taxonomy of Failure Modes in Agentic AI Systems (v2.0)*. 2026-06-04.
- Chen et al. *AgentPoison: Red-teaming LLM Agents via Poisoning Memory or Knowledge Bases*. NeurIPS 2024, arXiv:2407.12784.
- *MemoryGraft*. arXiv:2512.16962, 2025-12.
- OWASP Agentic Security Initiative. *Agentic AI — Threats and Mitigations* (Memory Poisoning = T1). 2025.