---
title: "12｜分层保留：给 Agent 的记忆建一套货架"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/988057"
tags:
  - "#pattern/memory"
  - "#axis/memory"
  - "#axis/hierarchy"
  - "#topic/layered-memory"
  - "#topic/working-set"
  - "#topic/policy"
  - "#topic/cover-rules"
  - "#concept/ttl"
  - "#concept/evidence"
  - "#concept/confidence"
audio: true
duration: "21:17"
publish_date: "2026-06-18"
course_progress: 12
pattern_position: "记忆 × 层级"
module: "记忆：沉淀之美（第 2 讲 / 共 5 讲）"
related:
  - "[[lession13-记忆模块导论]]"
  - "[[lession15-检索增强]]"
---

# 12｜分层保留：给 Agent 的记忆建一套货架

> **本讲定位**：记忆模式组第一个具体模式 → 解决"先给 Agent 的记忆搭一套合理货架"。

| 属性 | 内容 |
|------|------|
| 课程模块 | 记忆：沉淀之美（第 2 讲 / 共 5 讲）|
| 课程编号 | 12 |
| 双轴坐标 | **记忆 × 层级**（记忆：跨轮 / 跨会话 / 跨任务；层级：天然从属和覆盖关系）|
| 时长 | 21:17 |
| 发布日期 | 2026-06-18 |
| 前置讲 | [[lession13-记忆模块导论]] — 记忆是什么 |
| 后续讲 | [[lession15-检索增强]] — "取" |

## 一句话核心

> **分层保留（Hierarchical Retention）= 把 Agent 的记忆按作用域、生命周期和可信度切成多层，让每一层有独立的加载策略、写入规则、淘汰规则和 token 预算。**
>
> 真正的分层，**不是多建几张表，而是让不同类型的记忆拥有不同的存储位置、读取时机、覆盖规则和失效方式**。

---

## 一、为什么需要分层保留

### 1.1 最容易犯的错

> 做 Agent 记忆系统时，最容易犯的错是**把所有东西都叫"记忆"**：
>
> - 公司安全政策是记忆
> - 用户偏好是记忆
> - 项目规则是记忆
> - 当前任务进度是记忆
> - 刚刚工具返回的一段 JSON 也是记忆
>
> **名字都一样，但它们的作用范围、可信度和生命周期完全不同**。

### 1.2 不分层的三类污染

> 如果这些东西全都堆进同一个 prompt，短期看实现很快，长期看一定会乱：
>
> - **临时判断**会被当成长期事实复用
> - **长期规则**会被当前会话里的工具结果覆盖
> - **某个用户的偏好**会串到另一个用户身上
> - **某个项目的调试配置**会被误当成团队规范
>
> **系统越跑越久，记忆库越像一个没人整理的仓库**。

> **分层保留要做的，就是给 Agent 的记忆建一套货架**。
>
> - 什么东西应该常驻 context
> - 什么东西只在当前 session 有效
> - 什么东西先停在 scratchpad
> - 什么东西可以升级成长期记忆
> - 什么东西过期后应该降权或删除
>
> **都要先有位置**。

---

## 二、分层保留模式

### 2.1 在双轴图谱的位置

| 轴 | 类型 | 原因 |
|----|------|------|
| 认知功能 | **记忆** | 处理跨轮、跨会话、跨任务的信息怎么留下来 |
| 执行拓扑 | **层级** | 这些信息有**天然的从属关系和覆盖关系** |

### 2.2 核心金句

> 组织级规则 **高于** 项目偏好
> 项目规范 **高于** 当前会话里的临时猜测
> 当前任务目标 **高于** 局部工具输出

> **这里最关键的词不是"多层"，而是"各有命运"**。

### 2.3 五个坐标（比"分几层"更稳）

> 做分层时，**不要一上来就问"到底分三层还是五层"**。更稳妥的做法，是**先问五个坐标**。

| # | 坐标 | 问题 |
|---|------|------|
| 1 | **作用域** | 它属于**组织、项目、用户、任务、会话**，还是**当前一轮**？ |
| 2 | **生命周期** | 它应该活多久？是几分钟、一个 session、一个项目周期，还是长期有效？ |
| 3 | **权威来源** | 是**人写的、工具返回的、框架生成的**，还是**模型推断出来的**？ |
| 4 | **证据** | 是来自**测试结果、代码路径、用户确认**，还是**只是模型的一次猜测**？ |
| 5 | **token 预算** | 它应该**常驻 context**，还是**只在需要时被工具取回**？ |

> **这五个坐标，比简单区分热、温、冷层重要**。
>
> 冷热分层主要回答访问频率，但**回答不了**：
>
> - 公司合规规则能不能被覆盖
> - 用户偏好能不能串租户
> - scratchpad 里的临时判断能不能进入长期记忆
>
> 长程 Agent 出问题，常常不是因为**缺信息**，而是因为**不同可信度的信息被放在了同一个层级里**：
>
> - 一条公司安全红线
> - 一个用户口头偏好
> - 一次工具调用结果
> - 一个模型临时推断
>
> **被系统一视同仁**。
>
> **分层，就是先给这些记忆分责任**。

---

## 三、一套实用的五层货架

### 3.1 五层结构总览

| # | 层级 | 英文 | 解决什么 |
|---|------|------|----------|
| 1 | **策略 / 管理层** | Policy / Managed | 组织级规则、合规红线、安全边界 |
| 2 | **项目 / 领域层** | Project / Domain | 项目规范、领域模型、代码架构、业务流程 |
| 3 | **用户 / 租户层** | User / Tenant | 用户偏好、租户配置、长期个性化上下文 |
| 4 | **任务 / 会话层** | Task / Session | 当前目标、里程碑、状态、checkpoint、进度追踪 |
| 5 | **草稿纸 / 轮次层** | Scratchpad / Turn | 当前一轮工具结果、中间判断、候选方案、临时变量 |

### 3.2 每一层详解

**策略 / 管理层（最高层）**：

> 放组织级规则、合规红线和安全边界。比如：
>
> - 生产数据库不能直接写
> - 客户数据不能发送到未授权服务
> - 涉及权限和账单的变更必须先生成计划再等待人工确认
>
> **这一层不是偏好，也不是建议，而是边界**。
>
> 它**不能被用户一句话覆盖，也不能被当前 session 里的临时需求覆盖**。

**项目 / 领域层**：

> 放项目规范、领域模型、代码架构和业务流程：
>
> - **开发者 Agent**：可能是 `CLAUDE.md`、`AGENTS.md`、`README`、测试命令、目录说明
> - **课程 Agent**：可能是课程大纲、章节目标、作业结构
>
> **这一层最好贴近项目本身**，因为项目规则不只是 Agent 要看，人也要看。
>
> **需要能进仓库、能 review、能 diff、能回滚**。

**用户 / 租户层**：

> 放用户偏好、租户配置和长期个性化上下文：
>
> - 用户喜欢中文还是英文
> - 喜欢先讲原理还是直接给结论
> - 学生已经掌握哪些知识点
> - 企业租户有哪些特殊配置
>
> **用户层最怕自由文本堆积**。
>
> "小李大概会一点装饰器"这种句子人能懂，但系统很难维护。
>
> 半年以后，同一个概念可能出现多种写法，**检索、统计、迁移都会变得混乱**。
>
> **长期用户记忆应该像数据库资产，而不是聊天日记**。

**任务 / 会话层**：

> 放当前目标、里程碑、状态、checkpoint 和进度追踪（progress tracking）。
>
> 它回答的是：**这一轮任务现在推进到哪里，已经做了什么，下一步是什么，哪些方案被排除**。
>
> 它和 User 层的区别很重要：
>
> - 学生今天学装饰器时卡住了 → **当前 session 的状态**
> - 不等于这个学生**长期学得慢**
>
> **短期状态可以影响当前策略，但不能随便污染长期画像**。

**草稿纸 / 轮次层**：

> 放当前一轮工具结果、中间判断、候选方案和临时变量。
>
> **它是工作台，不是档案馆**。
>
> - 工具刚返回
> - 规则还没核完
> - 几个方案还没排除
> - 下一步动作还没决定
>
> 这些东西都可以先放在草稿纸。
>
> 等任务收束，系统再决定：
>
> - 哪些要扔掉
> - 哪些要写进任务层
> - 哪些要进入用户层
> - 哪些要沉淀成失败日记或长期经验

### 3.3 层数不是越多越好

| Agent 类型 | 推荐层数 |
|------------|----------|
| **个人助手** | 至少需要 用户 / 会话 / 轮次 3 层 |
| **开发者 Agent** | 通常至少需要 项目 / 用户 / 会话 / 轮次 4 层 |
| **企业系统** | 往往需要 组织 / 团队 / 项目 / 用户 / 会话 这样的更细边界 |

> **层数不是越多越好，关键是层数要和产品里的作用域一致**。
>
> 记住：**分层不是为了显得架构复杂，而是为了让 Agent 分得清**：
>
> - 什么是边界
> - 什么是偏好
> - 什么是状态
> - 什么是草稿

---

## 四、覆盖关系比层数更重要

### 4.1 覆盖矩阵

| 层 | 谁能覆盖谁 |
|----|------------|
| **策略层** | **不能被覆盖，只能被执行** |
| **项目层** | 可以覆盖通用默认值，**但不能覆盖前者** |
| **用户层** | 可以影响表达方式和个性化偏好，**但不能覆盖项目的硬性规则** |
| **任务层** | 可以决定当前优先级，**但不能改写用户或项目里的长期事实** |
| **草稿纸** | **只能提出候选判断，不能直接污染长期记忆** |

### 4.2 反例工程

> 这些常识人类可能下意识就做好判断了，但 Agent 却不行：

| 反例 | 错误记忆 |
|------|----------|
| **销售 Agent** | 把用户这一次"我今天想看便宜点的方案"写成长期偏好，后面三个月都只推荐低价版本 |
| **代码 Agent** | 把一次临时 debug 的 mock server 地址写进项目规则，下一次生产发布还在读 mock |
| **客服 Agent** | 把某个租户的特殊折扣记成全局价格规则，另一个租户也被影响 |

> 这些问题**不能只靠数据库修好**。**它们的根因，是分层边界没守住**。

### 4.3 升层机制

> 比较好的做法是：
>
> **内层可以临时影响外层的使用，但不能自动改写外层的事实**。
>
> 当前会话里，用户说"我已经会装饰器了"，Agent 可以**暂时**把讲解难度调高一点。
>
> **但要把用户层里的 mastery score 从 0.3 改到 0.6，需要更多证据**，比如：
>
> - 完成练习
> - 连续解释正确
> - 通过测试
> - 用户确认

```json
{
  "candidate_update": {
    "layer_from": "session",
    "layer_to": "user",
    "field": "mastery.decorator",
    "old_value": 0.3,
    "new_value": 0.6,
    "evidence": [
      "用户正确完成 decorator_exercise_2",
      "用户能解释 wrapper 返回函数"
    ],
    "confidence": 0.78,
    "requires_review": false
  }
}
```

> 这就是**升层机制**。
>
> - 草稿纸里的内容**先是候选**
> - 任务层里的状态是**当前事实**
> - 用户和项目层里的内容则需要**更强证据**
>
> **越长期、越高权威的层，越不能靠模型一句"我觉得"直接写入**。

---

## 五、写入路由：先放工作台，再决定升层

### 5.1 完整升层路径

```
Agent 推理中间判断、工具分析、候选方案
              ↓
        scratchpad 草稿纸
              ↓
   任务收束 / 验证通过
              ↓
     Memory Router 决定升层
              ↓
┌─────────┬─────────┬─────────┬─────────┐
   工具调用        排除方案       用户明确      项目约定
   结果验证        + 测试证据      偏好确认       + 仓库证据
   ↓              ↓             ↓             ↓
任务/会话层    进度追踪/失败日志  用户层        项目层
```

| 升层路径 | 触发条件 |
|----------|----------|
| **scratchpad → 任务/会话层** | 工具调用结果**通过验证** |
| **scratchpad → 进度追踪 / 失败日志** | 被排除的方案有**测试证据** |
| **scratchpad → 用户层** | 用户明确偏好**多次出现或被确认** |
| **scratchpad → 项目层** | 项目约定有**人类 review 或仓库证据** |
| **scratchpad → 失败日志 / 经验记录** | 风险教训**完成复盘** |

### 5.2 避开两个极端

| 极端 | 表现 | 后果 |
|------|------|------|
| **全都不记** | Agent 每一轮都认真思考，任务结束后只留一句摘要 | 下一次又从头来 |
| **全都记** | scratchpad 里的猜测、临时变量、过期工具结果和半成品判断**全都**进入长期记忆 | 跑久了以后，系统就像一个**没有清理过的仓库** |

> **分层保留要建立的，是中间路线**：
>
> - 先允许 Agent 把**当前工作台用好**
> - 再用框架**约束什么能沉淀、沉淀到哪一层、带什么证据、什么时候过期**

---

## 六、分层保留的最小代码骨架

### 6.1 完整代码（5 层 + 5 来源 + 7 字段）

```python
from dataclasses import dataclass, field
from datetime import datetime, timedelta
from enum import Enum
from typing import Any

class MemoryLayer(Enum):
    POLICY = "policy"
    PROJECT = "project"
    USER = "user"
    TASK = "task"
    SCRATCHPAD = "scratchpad"

class MemorySource(Enum):
    HUMAN = "human"
    TOOL = "tool"
    AGENT_INFERENCE = "agent_inference"
    VERIFIED_TRACE = "verified_trace"
    FAILURE_REVIEW = "failure_review"

@dataclass
class MemoryEntry:
    key: str
    value: Any
    layer: MemoryLayer
    source: MemorySource
    evidence_refs: list[str] = field(default_factory=list)
    confidence: float = 1.0
    token_estimate: int = 0
    valid_from: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    valid_until: str | None = None
    created_at: str = field(default_factory=lambda: datetime.utcnow().isoformat())
    last_accessed_at: str | None = None

    def is_expired(self) -> bool:
        if self.valid_until is None:
            return False
        return datetime.utcnow() > datetime.fromisoformat(self.valid_until)

    def is_verified(self) -> bool:
        return bool(self.evidence_refs) or self.source in {
            MemorySource.HUMAN,
            MemorySource.TOOL,
            MemorySource.VERIFIED_TRACE,
            MemorySource.FAILURE_REVIEW,
        }

@dataclass
class LayerPolicy:
    layer: MemoryLayer
    token_budget: int
    ttl: timedelta | None
    allow_agent_write: bool
    require_evidence: bool
    backend: str

class HierarchicalMemory:
    def __init__(self) -> None:
        self.entries: dict[str, MemoryEntry] = {}
        self.policies = {
            MemoryLayer.POLICY: LayerPolicy(
                layer=MemoryLayer.POLICY,
                token_budget=1200,
                ttl=None,
                allow_agent_write=False,
                require_evidence=True,
                backend="managed_file",
            ),
            MemoryLayer.PROJECT: LayerPolicy(
                layer=MemoryLayer.PROJECT,
                token_budget=3000,
                ttl=None,
                allow_agent_write=False,
                require_evidence=True,
                backend="git_file",
            ),
            MemoryLayer.USER: LayerPolicy(
                layer=MemoryLayer.USER,
                token_budget=1500,
                ttl=None,
                allow_agent_write=True,
                require_evidence=True,
                backend="postgres",
            ),
            MemoryLayer.TASK: LayerPolicy(
                layer=MemoryLayer.TASK,
                token_budget=5000,
                ttl=timedelta(days=7),
                allow_agent_write=True,
                require_evidence=True,
                backend="checkpointer",
            ),
            MemoryLayer.SCRATCHPAD: LayerPolicy(
                layer=MemoryLayer.SCRATCHPAD,
                token_budget=2500,
                ttl=timedelta(hours=2),
                allow_agent_write=True,
                require_evidence=False,
                backend="runtime_state",
            ),
        }

    def write(self, entry: MemoryEntry) -> None:
        policy = self.policies[entry.layer]
        if not policy.allow_agent_write and entry.source == MemorySource.AGENT_INFERENCE:
            raise ValueError(f"Agent cannot write directly to {entry.layer.value}")
        if policy.require_evidence and not entry.is_verified():
            raise ValueError(f"{entry.layer.value} memory requires evidence")
        self.entries[entry.key] = entry

    def propose_from_scratchpad(
        self,
        entry: MemoryEntry,
        target_layer: MemoryLayer,
    ) -> MemoryEntry:
        if entry.layer != MemoryLayer.SCRATCHPAD:
            raise ValueError("Only scratchpad entries can be promoted")
        return MemoryEntry(
            key=entry.key,
            value=entry.value,
            layer=target_layer,
            source=MemorySource.VERIFIED_TRACE,
            evidence_refs=entry.evidence_refs,
            confidence=entry.confidence,
            token_estimate=entry.token_estimate,
        )

    def assemble_context(self) -> list[MemoryEntry]:
        selected: list[MemoryEntry] = []
        for layer in [
            MemoryLayer.POLICY,
            MemoryLayer.PROJECT,
            MemoryLayer.USER,
            MemoryLayer.TASK,
            MemoryLayer.SCRATCHPAD,
        ]:
            budget = self.policies[layer].token_budget
            used = 0
            layer_entries = [
                entry
                for entry in self.entries.values()
                if entry.layer == layer and not entry.is_expired()
            ]
            layer_entries.sort(
                key=lambda entry: (
                    entry.confidence,
                    entry.last_accessed_at or entry.created_at,
                ),
                reverse=True,
            )
            for entry in layer_entries:
                if used + entry.token_estimate > budget:
                    continue
                selected.append(entry)
                used += entry.token_estimate
                entry.last_accessed_at = datetime.utcnow().isoformat()
        return selected

    def health_report(self) -> dict[str, Any]:
        return {
            "layers": {
                layer.value: {
                    "backend": policy.backend,
                    "token_budget": policy.token_budget,
                    "ttl_seconds": None if policy.ttl is None else policy.ttl.total_seconds(),
                    "allow_agent_write": policy.allow_agent_write,
                    "require_evidence": policy.require_evidence,
                    "entry_count": sum(
                        1 for entry in self.entries.values() if entry.layer == layer
                    ),
                }
                for layer, policy in self.policies.items()
            }
        }
```

### 6.2 五个关键设计点

| # | 设计点 | 含义 |
|---|--------|------|
| 1 | **SCRATCHPAD 是正式层** | 不能只是"随便写几句"，而要有 **TTL、有预算、有升层规则** |
| 2 | **POLICY 和 PROJECT 默认不允许 Agent 直接写** | Agent 可以提出修改建议，但要经过**人类 review、仓库证据或外部工具证据** |
| 3 | **require_evidence 把"能不能长期保留"跟证据绑定** | 长期记忆**不能只靠模型一句"我觉得"** |
| 4 | **assemble_context() 按层组装** | 不是简单按时间排序，组织规则、项目约定、用户偏好、任务状态、scratchpad **不会互相挤** |
| 5 | **MemoryEntry 数据结构** | 分层保留不是只有 layer 名字，还要把**来源、证据、过期时间、token 预算和写入权限**一起纳入结构化的 schema |

### 6.3 各层参数一览

| 层级 | token 预算 | TTL | Agent 写入 | 要求证据 | 后端 |
|------|-----------|-----|------------|----------|------|
| **POLICY** | 1200 | 永久 | ❌ | ✅ | managed_file |
| **PROJECT** | 3000 | 永久 | ❌ | ✅ | git_file |
| **USER** | 1500 | 永久 | ✅ | ✅ | postgres |
| **TASK** | 5000 | 7 天 | ✅ | ✅ | checkpointer |
| **SCRATCHPAD** | 2500 | 2 小时 | ✅ | ❌ | runtime_state |

---

## 七、用户场景：编程教练 Agent 的四层落地

### 7.1 三版迭代对比

| 版本 | 表现 | 问题 |
|------|------|------|
| **V1 无状态** | 学生第三次来，Agent 还问"你学过 list 吗？" | 每次会话像换了一个老师 |
| **V2 全量历史** | 新会话开始时**全量**塞进 prompt | 不再问 list 了，但一直围着 list/dict/loop 打转，忘了学生已学到装饰器 —— **不是没记住，而是记得太多、分不清轻重** |
| **V3 分层** | 用户/项目/会话/草稿纸四层独立加载 | 像真正的教练 |

### 7.2 完整四层对象示例

```json
{
  "user_layer": {
    "student_profile": {
      "age_group": "high_school",
      "english_level": "B1",
      "python_experience_months": 6,
      "teaching_preference": "example_first",
      "learning_goal": "3 个月内做出一个个人项目"
    },
    "mastery": [
      {
        "topic_id": "list",
        "score": 0.9,
        "last_practiced": "2026-04-10",
        "common_mistakes": []
      },
      {
        "topic_id": "decorator",
        "score": 0.4,
        "last_practiced": "2026-04-15",
        "common_mistakes": ["忘记写 @", "分不清 wrapper 和 inner"]
      }
    ]
  },
  "project_layer": {
    "course": "Python 进阶 · 装饰器与上下文管理器",
    "current_chapter": 7,
    "total_chapters": 12,
    "chapter_goal": "理解 decorator、closure、context manager 的关系",
    "course_policy": "每个新概念先给最小代码例子，再解释术语"
  },
  "session_layer": {
    "session_goal": "理解 contextmanager 装饰器",
    "covered_topics": [
      "@contextmanager 基本用法",
      "__enter__ / __exit__ 协议",
      "嵌套 with"
    ],
    "student_questions": [
      "为什么 yield 后面的代码会在退出 with 时执行？"
    ],
    "emotion_state": "confused",
    "next_step": "用文件打开/关闭的例子重新解释 contextmanager"
  },
  "scratchpad_layer": {
    "current_user_code": "with open('a.txt') as f:\n    data = f.read()",
    "tool_result": "syntax_check_passed",
    "temporary_hypothesis": "学生理解 with 语法，但不理解 contextmanager 的 yield 分界",
    "response_plan": "从 with 的进入和退出时机讲起"
  }
}
```

### 7.3 关键细节

| 升层决策 | 处理 |
|----------|------|
| 学生"今天卡住了" | **先放会话层，不能马上写进用户层**。今天卡住 ≠ 长期学得慢 |
| 学生连续三次在装饰器上犯同一个错 | 从会话层**升到用户层**，写成 `common_mistake`。需要三次作业、三次会话 trace，或一次明确练习结果 |
| 本轮代码片段 | 只放**草稿纸 / 轮次层**。对当前讲解很重要，但不该长期保存，除非暴露可复用的学习弱点 |
| 课程大纲 | 放**项目层**，由人或课程系统维护。Agent 不应因一次对话就改写课程结构 |

> **分层做对之后，Agent 不是"记性变好"这么简单，而是开始分得清**：
>
> - 什么是学生长期特点
> - 什么是这节课的状态
> - 什么是这一轮的草稿

---

## 八、工程落地时看的三件事

### 8.1 存哪里（按层选 backend）

| 层 | 推荐 backend | 原因 |
|----|--------------|------|
| **策略层** | 配置中心、只读文件、管理后台 | 需要 review、不可改写 |
| **项目层** | **文件系统 + Git 仓库** | 需要 review / diff / blame |
| **用户层** | **profile store / 关系数据库** | 长期资产，需要 schema、版本、迁移 |
| **任务 / 会话层** | **checkpoint store** | 要恢复状态 |
| **草稿纸 / 轮次层** | **runtime state** | 高频变化、短期有效，用完清理 |

### 8.2 何时读

| 层 | 读取策略 |
|----|----------|
| **策略层** | 启动时加载关键规则，但**要短** |
| **项目层** | 启动时加载核心约定，**长文档只放 handle，需要时再读** |
| **用户层** | 启动时加载**摘要**，必要时按字段查详情 |
| **任务层** | **恢复最近状态**，不加载完整历史 |
| **草稿纸 / 轮次层** | 只在当前模型调用前**拼接**，**用完就丢** |

> **分层存储如果最后又全量读取，只是把仓库分了区，出门时还是把整个仓库背在身上**。

### 8.3 何时写

| 风险级别 | 写入策略 |
|----------|----------|
| **普通 Agent** | 定时写回 + session 结束兜底 |
| **高风险场景**（金融、医疗、权限审批）| **关键状态实时写回 + 一般状态批量写回** |

> **不要只等 session 结束再写**。
>
> 长程 Agent 跑半小时，中途崩一次就丢一大段状态，这种体验会让人很快失去信任。

### 8.4 长期层 schema 纪律

**不好的写法**（裸字段）：

```json
{ "learning_pace": "slow" }
```

**正确的写法**（带 metadata）：

```json
{
  "learning_pace": {
    "value": "slow",
    "source": "agent_inference",
    "confidence": 0.68,
    "evidence": "连续两次在 decorator 练习中请求基础解释",
    "set_at": "2026-06-15",
    "scope": "current_course",
    "schema_version": "2026-06-01"
  }
}
```

> **多几个 metadata 字段，写的时候麻烦一点，debug 时会救命**。因为你会知道：
>
> - 这是用户自己说的，还是 Agent 推断的？
> - 置信度是多少？
> - 什么时候写入？
> - 适用范围是当前课程，还是所有学习场景？

---

## 九、分层记忆 = 操作系统的工作集

### 9.1 传统 vs LLM Agent

> 你可能会觉得，**分层保留不就是数据库 schema 设计吗**？
>
> User 表、Project 表、Session 表、Turn 表，加上 TTL、索引和权限，传统 Web 应用也这么干。

> **这个判断只对了一半**。

### 9.2 另一半：LLM Agent 独有

| 维度 | 传统 Web 应用 | LLM Agent |
|------|---------------|-----------|
| **要查什么** | 一个 dashboard 请求要查哪些表、哪些字段，**通常是确定的** | 这一轮它可能需要用户偏好，下一轮需要项目测试命令，再下一轮需要上周某次失败经验 |
| **prompt context** | 没有限制 | **有限** |
| **推理所需的记忆** | 设计时确定 | **不确定** |

> **这就像操作系统里的工作集（working set）**。
>
> - 一个进程在某段时间内真正活跃使用的内存页，应该**留在 RAM 里**
> - 不活跃的可以**换出到磁盘**，需要时再换回来
>
> **估得准系统运行就流畅；估不准就会频繁 swap，性能崩掉**。

### 9.3 对应关系

| 类比 | 含义 |
|------|------|
| **LLM Agent 上下文窗口** | **RAM** |
| **外部文件、数据库、向量库、图数据库** | **磁盘** |
| **当前这次推理真正需要的记忆集合** | **working set** |

> **分层保留的本质，是给 Agent 一个工作集的先验**。
>
> - **策略层**：通常常驻，因为它定义边界
> - **Project 层**：常驻或半常驻，因为它定义项目规则
> - **用户层**：用摘要常驻，因为 Agent 总要知道用户是谁
> - **任务 / 会话层**：**动态进入**，因为只需要最近任务状态
> - **草稿纸 / 轮次层**：只在当前轮进入，**用完就丢**
> - **大体量语义记忆**：通常不常驻，而是交给下一讲的 **RAG 按需取回**

> **分层保留不是为了让 schema 看起来漂亮，而是为了提高 Agent 推理时的工作集命中率**。
>
> **好的分层，不看你存了多少，而看这一次该进入上下文的东西有没有进入，不该进入的东西有没有挡在外面**。

---

## 十一、分层保留的四个常见问题

> 最常见的是**把所有历史对话塞进 prompt**：
>
> 看起来最省事，实际上很快会把 Agent 拖进噪声里。
>
> **长程任务后半段的很多迷失，不是因为 Agent 没有上下文，而是因为上下文里旧信息太多、权重又没有分层**。

| # | 问题 | 后果 |
|---|------|------|
| 1 | **全塞 prompt** | 上下文里旧信息太多、权重又没有分层 |
| 2 | **自由文本长期记忆** | 半年后同一概念十种写法，**检索不稳定，更新不可靠** |
| 3 | **scratchpad 直通长期记忆** | 半成品里的猜测、临时解释、未验证观察**原样长期保存** |
| 4 | **没有降权机制** | 用户偏好、业务规则、代码结构、政策都会变，旧记忆变新判断的**毒药** |

> **长期层至少要把高频字段 schema 化**。
>
> scratchpad 应该被**验证、蒸馏、加证据后再升层**，而不是原样长期保存。

> **分层的目标，不是把记忆放整齐，而是让 Agent 在长程任务里分得清**：
>
> - 什么该信
> - 什么该用
> - 什么该忘

---

## 十二、6 框架对比

> 黄佳老师在本讲**没有给出 8 框架横切**，因为"上面的讲解示例清晰，一气呵成"。
>
> **对各种 Harness 的分层保留分析就留给大家完成**，欢迎在评论区分享你熟悉的 Harness 如何做分层保留。

### 读者贡献：Claude Code 五层映射（PatrickL）

> | 层级 | 路径 |
> |------|------|
> | **企业级** | `/etc/claude-code/CLAUDE.md` |
> | **用户级** | `~/.claude/CLAUDE.md` |
> | **项目级** | `./CLAUDE.md` |
> | **规则级** | `.claude/rules/*.md` |
> | **本地级** | `./CLAUDE.local.md` |

> **作者回复**：总结的好！

### 读者贡献：Hermes 三层架构（AIKO Nexus）

> 1. **内置策展记忆**：稳定事实 / 用户画像，**默认加载**（存储：`MEMORY.md` / `USER.md`）
> 2. **完整历史对话**：**按需检索**（存储：`SQLite+FTS5`）
> 3. **外部 Provider**：honcho、mem0、openviking...，按需检索查找（存储：外部服务 / 本地库）

### 读者贡献：Openclaw 多层记忆（AIKO Nexus）

> **state/**：`SQLite openclaw.sqlite` —— 公司级：网关 registry/插件KV/cron
>
> | 文件 | 作用 |
> |------|------|
> | `AGENTS.md` / `SOUL.md` / `USER.md` | 项目指令 / 身份设定 / 用户画像 |
> | `MEMORY.md` | 长期精炼记忆 |
> | `memory/YYYY-MM-DD-*.md` | 任务/会话级（单日对话元数据 + memoryFlush 总结出的 daily notes）|
> | `DREAMS.md` | 记忆升层机制，六个维度加权达标后记录升层到 MEMORY.md |
>
> **agent/openclaw-agent.sqlite**：per-agent 索引 + embedding + cache_entries（scratchpad）
>
> **sessions/{sessions.json, *.jsonl}**：每轮会话的原始数据和索引

---

## 十三、精选留言

### 留言 1：街角·陌路△ — 记忆的关键是遗忘

> 其实分层这个就很像在模仿人脑的记忆模式，我上讲提到那个**富内斯**的例子，其实很好地诠释了分层的价值。
>
> 富内斯其实就是一个没办法给记忆分层的典型例子，它就有点像把所有的内容都放到 prompt，这其实也回答了上次我给佳哥提的那个问题，就是即使上下文窗口再大，也需要记忆。
>
> **首先没有分层就意味着所有内容都是平等的**。那么就会出现富内斯那个情况，就是回忆什么东西都是按着顺序来回忆，因为他没办法知道哪些重要，哪些是不重要的，只能按着顺序来做。那对应大模型的模式呢，就是类似**大海捞针测试**的模式，虽然模型在一直提升这个能力，但是从工程化角度来说，这样的成本太高了。
>
> **没有分层意味着就没办法区分哪些是重点，也就是没办法进行内容的抽象**。这也是佳哥提到很重要的点，就是要给不同的记忆进行分级，甚至还有记忆的升级。而在富内斯的情况下，因为他没办法区分重点，所以他把不同斑点的狗都认为是不同的物种，这就好比记忆没办法进行合并，最后就是一堆冗余信息。
>
> **其实我反倒认为记这个事很容易，因为无非就是把内容放到某个位置里，只不过放的位置不同。但是如何过滤噪声？如何进行遗忘，我认为才是一个更关键的事情**。遗忘就意味着就要给信息进行分级。不是说用户所有对话都要记下来，也不是说所有的对话内容都是平等重要的。正如我们不会记住上个月每天都吃了什么，但是我们会记住上个月的十五号，因为我和一个重要的人吃饭，所以我能回忆起当时吃了什么。
>
> **我个人认为如何更好地遗忘，减少噪声才是上下文管理最大的价值**。

> **作者回复**：**如何更好地遗忘，减少噪声才是上下文管理最大的价值**。—— 完全同意。
>
> 当我能够学会**忽视某些我不需要关注的声音**，我的世界更聚焦了。

### 留言 2：Jason Ding — 置信度如何评估？

> 想了解一下**置信度**如何评估？

> **作者回复**：置信度**不能只让模型自报"我有 80% 把握"**，它应该来自这条记忆背后的证据。
>
> 比如 Agent 想记录"用户已经掌握 Python 装饰器"：
>
> | 阶段 | 证据 | 置信度 |
> |------|------|--------|
> | 用户只说过一句"我用"过 | 低 | 低置信候选 |
> | 完成一道相关练习 | 中 | 中等置信 |
> | 在不同任务中多次正确使用，并通过测试验证 | 高 | **高置信长期记忆** |
> | 后续任务暴露出明显错误 | — | **降权或重新确认** |
>
> 工程上通常看：
>
> 1. **来源是否可靠**
> 2. **证据是否充分**
> 3. **多个证据是否一致**
> 4. **信息是否仍然新鲜且适用于当前范围**
>
> 低置信内容留在当前会话或等待确认，**高置信内容才允许进入长期层**。
>
> 刚开始不必追求精确的数字，用"低、中、高"配合明确规则反而更可靠。
>
> 真要使用数值，**还要拿历史样本校准**：标为 0.8 的记忆，后来是否大约有八成被证实。

### 留言 3：AI2026 — K8s SaaS 环境下本地 MD 方案是否合理？

> 想请教两个企业级 Agent 架构生产落地问题，重点评估本地文件方案在 K8s SaaS 环境的适用性与行业最佳实践：
>
> **一、Agent Memory 存储设计**
>
> 现有方案：workspace 下 memory 目录，Agent 按日期 + 时间戳生成 Markdown 记忆文件，长期会累积大量文件。三个问题：
>
> 1. 多 Pod 调度场景：同一会话多轮对话可能落到不同 Pod。Pod 内 MD 文件是临时存储，跨 Pod 无法读取历史记忆；多用户请求分散到不同 Pod 也会记忆隔离。**请问这种运行时本地文件存储，是否不适合集群生产环境？**
>
> 2. 若挂载 K8s 持久卷共享文件：长期累积大量 MD 文件会持续膨胀存储。**海量文件会不会拖慢 Pod 启动、降低资源利用率？把记忆文件放进镜像或持久卷，是否属于不合理的生产设计？**
>
> 3. **最佳实践**：Markdown 文件记忆方案是不是仅用于教学和本地调试，不适合生产？企业生产环境，Agent 记忆是否应该存数据库、OSS/S3、向量库或专用 Memory Store？
>
> **二、Agent Skill 存储与扩展**
>
> 当前：Skill 能力用 Markdown 放在项目目录，随代码打包部署。三个问题：
>
> 1. 大量 Skill 文件打包进镜像，镜像体积持续膨胀，**增加镜像拉取、Pod 启动耗时**
>
> 2. Skill 变更比代码频繁，但**每次修改都要重新构建、发布镜像**
>
> 3. **设想 Skill 放到数据库 / 知识库 / 配置中心**，和 Agent 代码解耦；Agent 动态按需加载 Skill，按用户、场景路由选择能力，支持独立维护、热更新

> **作者回复**：问题清晰，我直接回答了哈，供讨论。
>
> **一、Agent Memory 存储设计**
>
> 1. **记忆正式保存在持久存储中，不能靠 Pod**。加入机制：任务开始时，把需要的内容放进当前工作目录，供 Agent 阅读和处理。需要长期保留的变化，再可靠地写回持久存储。**这里，本地文件是工作副本**。需要处理任务重试和写回冲突。多用户之间的记忆隔离通常是常见需求，与记忆如何存储无关
>
> 2. **持久卷则可以用于正式生产**。文件多不一定一定启动慢，要看启动时是否扫描全部文件、重建索引，或者递归修改文件权限。可考虑多节点读写，注意两个 Agent 同时改一份记忆，谁的修改生效这种场景。存储膨胀则无论换成文件还是数据库都会发生，**和 Agent 无关，必须设计保留期限、合并、归档和删除规则**
>
> 3. 具体情况具体分析：
>
> | 数据类型 | 推荐存储 |
> |----------|----------|
> | 对话、任务进度、恢复检查点 | **数据库** |
> | 用户偏好、已确认的长期记忆 | **数据库**，正文可以仍是 Markdown |
> | 大文档、附件、原始材料 | **OSS/S3** |
> | 按含义检索相关记忆 | **向量索引**（需要时再加）|
> | 本次任务的临时文件 | **Pod 本地工作区** |
>
> **二、Agent Skill 存储与扩展**
>
> 太累了，这个问题简答哈，过几天再详细回答。**Skill 随代码发布和独立发布，都可以是合理方案**。**稳定 Skill 随应用发布，频繁变化的 Skill 独立管理、按版本加载**。

### 留言 4：老实人Honey — 学习助手场景

> 如果分层记忆做得好，那么千问豆包等应该成为一个很好的学习助手，带领我每天学习复习新的编程语言，安排我的每天学习进度，出题目并根据我的解题情况记录错题集等。那么**系统级、助手级、对话级（每天新的）、错误日志**的都包括了。

> **作者回复**：**这个场景非常适合分层记忆**。
>
> - 课程计划和规则属于**长期约束**
> - 每日进度属于**任务状态**
> - 错题是**带证据的失败经验**
> - 个人偏好和掌握度则需要**缓慢更新**
>
> **关键是每层都有写入条件、过期规则和可纠正入口**。

### 留言 5：Geek_7f0e61 — 会话属于上下文？

> 会话和轮次，是不是应该属于上下文？

> **作者回复**：**也许属于上下文，更应该属于 Log 和 Trace**。

---

## 十四、思考题

> 你可能注意到了分层保留模式的讲解过程中，**我并没有给出各种 Harness 的工程现场切片**，因为我觉得上面的讲解示例清晰，一气呵成。
>
> **对各种 Harness 的分层保留分析就留给大家完成**，请在评论区里面谈一谈你熟悉的 Harness 系统如何做分层保留（可以用 AI 辅助分析）。

### 题 1：画出你的记忆货架

> 画出你当前 Agent 的记忆货架：
>
> - 它有哪些层？
> - 公司级、项目级、用户级、任务级、scratchpad 级分别在哪里？
> - 有没有几类信息现在被塞在同一个 `history` 字段里？

### 题 2：审计长期记忆的 schema 完整性

> 找一条你系统里最近写入的长期记忆。
>
> - 它有没有来源、证据、版本和失效条件？
> - 如果没有，**下一次它被错误召回时，你准备怎么判断它已经不该用了**？

### 题 3：scratchpad 升层规则

> 检查你的草稿纸。
>
> - 它是**只服务当前任务的工作台**，还是已经变成**长期记忆的垃圾入口**？
> - 给它设计一条最小升层规则：
>   - 什么内容能进入**任务层**
>   - 什么内容能进入**用户层**
>   - 什么内容必须**任务结束后丢掉**

---

## 十五、下一讲预告：检索增强（RAG）

> 下一讲我们正式进入**第二个具体记忆模式——检索增强（RAG）**。
>
> - 分层保留解决"**货架怎么搭**"
> - 检索增强解决"**货架太大以后，当前这一轮到底该取哪几件东西**"
>
> 检索增强也是：
>
> - **2023 年大火**
> - **2024 年泛滥**
> - **2025 年开始反思**
> - **2026 年正在重新被定位**的一个模式
>
> 到了 Agent 时代，**它不能再被当成万能入口**。它要回到记忆系统里，成为**语义记忆那一层的取回方式**。

→ [[lession15-检索增强]]

---

## 附：本讲核心要点速查

| 关键词 | 含义 |
|--------|------|
| **Hierarchical Retention** | 按作用域、生命周期、可信度切成多层，每层独立 backend/TTL/budget |
| |
| **五个坐标** | 作用域 / 生命周期 / 权威来源 / 证据 / token 预算 |
| **五层货架** | POLICY / PROJECT / USER / TASK / SCRATCHPAD |
| **覆盖关系** | POLICY 不可覆盖 → PROJECT 不能覆盖前者 → USER 不能覆盖项目硬性规则 → TASK 不能改写长期事实 → SCRATCHPAD 只能提候选 |
| **升层机制** | scratchpad 内容需验证、蒸馏、加证据后才能进长期层；需 evidence_refs + VERIFIED_TRACE source |
| **完整 schema 7 字段** | key / value / layer / source / evidence_refs / confidence / token_estimate + valid_from + valid_until + last_accessed_at |
| |
| **backend 选型** | managed_file / git_file / postgres / checkpointer / runtime_state |
| **token 预算** | 1200 / 3000 / 1500 / 5000 / 2500 |
| **TTL** | None / None / None / 7d / 2h |
| |
| **覆盖反例** | 销售推低价 3 月 / 代码 mock 地址进生产 / 租户折扣变全局规则 |
| **working set 类比** | 上下文窗口 = RAM / 外部存储 = 磁盘 / working set = 当前推理所需记忆集合 |
| |
| **schema 纪律** | 长期层带版本号、来源、置信度、证据、有效期、schema_version |
| **四个常见问题** | 全塞 prompt / 自由文本长期 / scratchpad 直通 / 没有降权机制 |
| |
| **Claude Code 五层映射** | `/etc` / `~/.claude` / `./CLAUDE.md` / `.claude/rules/` / `./CLAUDE.local.md` |
| **Hermes 三层** | 策展记忆（默认加载）/ 完整历史（按需）/ 外部 Provider（按需）|
| **Openclaw 升层机制** | DREAMS.md：六个维度加权达标后升层到 MEMORY.md |
| **金句** | 「如何更好地遗忘，减少噪声才是上下文管理最大的价值」 |