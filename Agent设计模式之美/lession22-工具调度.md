---
title: "22｜工具调度：从候选选择到执行准入"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/999645"
tags:
  - "#Agent/行动"
  - "#设计模式/工具调度"
  - "#行动/工具前沿"
  - "#行动/执行准入"
  - "#行动/业务幂等"
audio: true
duration: "30:49"
publish_date: "2026-07-23"
course_progress: "22/43"
pattern_position: "行动模式组·第 2 个"
related:
  - "[[lession21-行动模块导论]]"
  - "[[lession23-规划执行]]"
  - "[[lession24-提示链]]"
  - "[[lession25-护栏三明治]]"
---

# 22｜工具调度：从候选选择到执行准入

> **核心命题**：工具调度是**行动意图与工具 Handler 之间的执行准入组件**。它接收结构化的工具名、参数和调用上下文，在当前工具前沿中解析真实注册项，再检查调用配额、状态新鲜度和审批条件。**准入成功才会触发 Handler**。
>
> **最简工具集 + 工具调度的精华**：**先缩小能做什么，再判断这次能不能做，做完以后还要能证明发生了什么**。

---

## 一、上一讲留下的两类问题

回顾上一讲薪酬 Agent 在为 E0007 发薪过程中：

- 顺手修改了银行账号
- 清掉了异议记录
- 为"确保到账"重发了一次薪水

四条候选动作全被送进执行函数，导致**三个错误，分成两类**：

| 错误类型 | 具体表现 |
|---------|---------|
| **第一类：不该出现在当前任务里的工具** | 修改银行账号、清除异议备注都是真实可用的工具，但本轮目标只是发薪。**工具合法，任务范围不合法** |
| **第二类：必须保留、却被错误使用的工具** | `transfer_salary` 当然不能删，问题在于 Agent **跳过了核对步骤，还对同一员工执行了两次** |

### 1.1 两个问题

```text
怎样设计 Harness，让越界工具在本轮任务里不可被触达？
怎样让合法工具在满足状态和次数约束后才能执行？
```

- **第一道题** → 由**最简工具集**解决
- **第二道题** → 由**工具调度**解决

---

## 二、三个基本原则

贯穿本讲的三个"不等于"：

```text
1. 企业拥有的工具库  !=  当前任务应当看见的工具集
   （区分整体能力库和当前能力面）

2. 语义上选对了工具  !=  这一次调用已经获得执行准入
   （区分选择和授权）

3. 一次坏调用被拒绝  !=  原始业务目标已经完成
   （区分局部拒绝和全局任务恢复）
```

---

## 三、L0 / L1 / L2 三层实验结果

```bash
cd agent-design-patterns
uv sync --extra ui
uv run --extra ui python action/payroll-lab/web_app.py
# 浏览器打开 http://127.0.0.1:8765
# 选择第 22 讲，点击"运行三层对照"
```

| 实验 | 状态差异 | 出账 | 纪律 | 判定 |
|------|---------|------|------|------|
| **L0 裸循环** | 3 行 | 2 笔 | 无 | **受损** |
| **L1 最简工具集** | 1 行 | 2 笔 | 无 | **受损** |
| **L2 工具调度** | 1 行 | 1 笔 | 有 | **守住** |

> **L1-最简工具集和 L2-工具调度各自解决了一个问题**：
> - L0→L1：状态差异从 3 行降到 1 行，**银行账号和异议备注的问题被解决**
> - L1→L2：动作流水从 2 笔降为 1 笔，**付款前完成了核对**

> 只看数据库终态，L1 和 L2 执行完毕都是把一张工资单变成 `PAID`；但查看动作账目，**前者写了两遍，后者只执行了一次**。

---

## 四、第一层：让越界工具不可达

### 4.1 工具前沿（Tool Frontier）

L0 裸循环给执行链暴露了五项能力：

```python
FULL_TOOLS = {
    "query_payroll",
    "transfer_salary",
    "normalize_bank_account",
    "clear_payroll_note",
    "reverse_transfer",
}
```

对"给 E0007 发薪"这个目标，真正需要直接交给 Agent 的只有：

```python
MINIMAL_TOOLS = {
    "query_payroll",
    "transfer_salary",
}
```

**工具前沿**指的是当前任务、当前身份、当前阶段**真正可以看见的那组工具**。系统完整的工具库可能有几百项，工具前沿就是本轮工具调用能够打开的那一扇窗口。

### 4.2 L1 执行链

```python
for proposal in proposals:
    if proposal["tool"] not in tools:
        print(f"REJECTED {proposal['tool']}: 候选前沿里没有")
        continue
    handlers[proposal["tool"]](**proposal["args"])
```

注入夹具仍提出 `normalize_bank_account` 和 `clear_payroll_note`。**最简工具集这道防线没有劝它改变主意，只是让这两项能力在当前阶段根本没有执行入口**。越界动作因此停在 Handler 之前。

### 4.3 最简工具集的定义

> **"最简"负责移走无关能力，"完备"保证原目标仍然能够完成**。

- 十个工具可能已经相互重叠
- 三十个边界清晰的工具也未必有问题
- **追求的不是某个固定数量**

**具体怎么确定最简工具集**，要看：

- 业务目标
- 身份
- 阶段
- 数据范围
- 副作用风险

### 4.4 工具前沿的生成顺序

```text
企业工具总目录
  → 来源、租户与身份硬过滤
  → 当前阶段与副作用等级过滤
  → 在剩余工具中做语义召回与重排
  → 常驻核心工具 + 少量相关扩展工具
  → 当前工具前沿
```

> **先做硬权限过滤，再进行语义召回**。因为无权使用的工具，从一开始就不应该先被向量检索推到模型眼前。语义召回负责相关性可以有误差，**硬过滤负责资格，必须准确**。

### 4.5 使用最简工具集不等于删除工具

- 银行账号维护能力**仍在企业工具库里**
- 将来进入主数据维护任务，它可以重新进入工具前沿
- 低频工具还可以通过**工具搜索（Tool Search）**按需发现
- 或者交给专门的**子 Agent（Sub-agent）**持有

---

## 五、第二层：合法工具也要先拿到准入

### 5.1 ToolMetadata（执行契约）

```python
ToolMetadata(
    name="transfer_salary",
    description="pay one employee",
    when_to_use="after fresh read",
    is_destructive=True,
    requires_fresh_state=True,
    quota_per_session=1,
    rollback_action="reverse_transfer",
    risk_level=RiskLevel.CRITICAL,
)
```

| 字段 | 含义 |
|------|------|
| `requires_fresh_state=True` | 写入前必须重新读取相关事实（**fresh read**） |
| `quota_per_session=1` | 同一会话、同一工具和同一对象最多行一次（**会话配额**） |
| `rollback_action` | 指向补偿动作（付款以后若要撤销，应该走冲正流程） |

### 5.2 行动意图（Action Intent）

```python
trace = dispatcher.dispatch(
    proposal["tool"],
    proposal["args"],
    session_id="payroll-run-2026-06",
)
```

这份"工具名加参数"的候选，叫**行动意图（Action Intent）**。它表达 Agent 想做什么，**但这并不等同于已经获得授权的命令**。

### 5.3 执行准入（Admission）

调度器先选中真实注册的工具，再根据执行契约给出 `allow` / `deny` / `await` 一类结果。这个过程叫**执行准入（Admission）**。

> **只有得到放行，负责真实 SQL 或 API 调用的函数（也就是工具的 Handler）才会运行**。

### 5.4 工具调度的完整链路

```text
行动意图
  → 当前工具前沿
  → 选择候选工具
  → 检查本次准入
  → Handler
  → 动作证据
```

> 它在 [[lession4-双轴框架（下）\|双轴图谱]] 里落在**"行动 × 路由"焦点**，核心动作是把一份行动意图导向一个受控工具入口。

### 5.5 工具调度的工程化定义

> **工具调度（Tool Dispatch）**是行动意图与工具 Handler 之间的执行准入组件。它接收结构化的工具名、参数和调用上下文，在当前工具前沿中解析真实注册项，再检查调用配额、状态新鲜度和审批条件。准入成功才会触发 Handler。每次尝试都返回 `success` / `rejected` / `failed` 轨迹，并保留拒绝原因或工具异常。
>
> **模型负责提议调用，调度器持有最终执行入口**。参数 Schema 与身份权限仍要由上游意图校验器和工具前沿生成器负责。

---

## 六、工具调度模式实现流程

### 6.1 dispatch() 五关

#### 第一关：查注册表

```python
meta = self.tools.get(tool_name)
if meta is None:
    trace.status = "rejected"
    trace.rejected_reason = "tool_hallucination"
    return self._finalize(trace, start)
```

> **模型编出一个不存在的工具名**，或者试图调用没有注册的函数，**会在 Handler 前被拒绝**。注册表既是工具目录，也是一张可执行白名单。

#### 第二关：检查调用次数

```python
key = self._quota_key(session_id, tool_name, args)
used = self.quota.get(key, 0)
if meta.quota_per_session != -1 and used >= meta.quota_per_session:
    trace.status = "rejected"
    trace.rejected_reason = (
        f"quota_exceeded:{used}/{meta.quota_per_session}"
    )
    return self._finalize(trace, start)
```

> **quota 是调用配额**，先用于拦住同一会话里的明显重复动作。

#### 第三关：检查最近是否读取过状态

```python
if meta.requires_fresh_state:
    last = self.last_state_refresh.get(session_id, 0.0)
    if time.time() - last > self.STATE_FRESHNESS_SECONDS:
        trace.status = "rejected"
        trace.rejected_reason = "stale_state_must_refresh"
        return self._finalize(trace, start)
```

#### 第四关：处理审批要求

```python
if meta.requires_approval:
    trace.status = "rejected"
    trace.rejected_reason = "awaiting_approval"
    return self._finalize(trace, start)
```

> 教学实现只返回"等待审批"，没有创建审批票据，也没有恢复执行的 API。

#### 第五关：Handler 执行

```python
try:
    trace.output = self.handlers[tool_name](**args)
    trace.status = "success"
except Exception as error:
    trace.status = "failed"
    trace.rejected_reason = f"{type(error).__name__}: {error}"
```

> **`rejected` 和 `failed` 一定要分开处理**：
> - `rejected` → 调用没有获得准入，**副作用尚未开始**
> - `failed` → Handler 已经运行并出现异常。**网络超时时，远端甚至可能已经成功**，因此异常也不代表执行并没有完成

---

## 七、被拒绝以后，谁把任务拉回正轨

### 7.1 上下半场分工

> **上半场**是拒绝偏航提案，**下半场**是继续完成原始目标。**工具调度只负责上半场**。它已经把危险的调用挡在门外，但 E0007 并没有领到工资，**因此原始任务还没完成**。

### 7.2 Harness 持有三类信息

```text
1. 北极星目标：给 E0007 正确发薪，不动员工主数据
2. 当前任务状态：例如 E0007 仍是 DRAFT
3. 每次工具尝试留下的 DispatchTrace，包括拒绝原因
```

Harness 因而能判断：

```text
四条偏航动作已经处理完，但北极星目标仍未完成
```

### 7.3 恢复策略

教学 Lab **没有再调用一个大模型进行"自我反省"**，而是把恢复策略写成确定性控制代码：

```python
d.dispatch("query_payroll", {"emp_id": "E0007"}, "s")   # 先读
t = d.dispatch("transfer_salary", {"emp_id": "E0007"}, "s")  # 再付
```

### 7.4 数据前后对比

| 证据 | 运行前 | 四条偏航动作处理后 | Harness 恢复后 |
|------|--------|----------------|---------------|
| **E0007 工资单状态** | `DRAFT` | `DRAFT` | `PAID` |
| **E0007 付款流水** | 0 笔 | 0 笔 | 1 笔，先读后写 |
| **E0007 银行账号** | 保留原值 | 保留原值 | 保留原值 |
| **E0012 异议备注** | 保留原值 | 保留原值 | 保留原值 |

`db.py --diff` 对员工、工资单和审批三张业务表做基线比较：

```text
[EDIT] payroll id=7: status: 'DRAFT' -> 'PAID'
1 rows differ from the baseline.
```

> **上面各个 Trace 一个都不能忽略，少看其中任何一本账，都有可能把"双付后所显示的 PAID 状态"误判为成功**。

### 7.5 关键纪律

> **Harness 还需要确保两次被拒绝的调用提案没有消耗配额，真正成功的那次付款才消耗配额**。否则，攻击者只要先制造一次失败调用，就可能把合法付款的名额占掉。

### 7.6 职责边界

> 工具调度负责判断一次调用能否进入 Handler，**Harness 负责依据原目标决定重读、等待、改计划或停止**。
>
> **目前的实验中这个恢复路径是固定逻辑，只是为了单独观察工具调度的作用**。生产系统可以把这段逻辑交给状态机、计划执行器或人工审批流程。而调度器本身不应偷偷承担整项任务的目标恢复。

---

## 八、把"选工具"拆成四个动作

> 在项目设计中，我们常把四件事都称为"选工具"，如果这样，出问题时便只能用一句"模型选错了"来搪塞。

| 动作 | 放到薪酬例子里 | 它回答的问题 |
|------|--------------|----------|
| **发现（Discovery）** | 企业工具库中存在发薪能力 | **系统里有没有** |
| **建立工具前沿（Frontier Building）** | 本轮只开放查询与发薪 | **当前该看见哪些** |
| **选择（Selection）** | 行动意图匹配 `transfer_salary` | **语义上用哪个** |
| **准入（Admission）** | 检查 fresh state、次数和审批 | **此刻能不能执行** |

### 8.1 五种问题诊断

| 现象 | 诊断 |
|------|------|
| 正确工具根本没搜到 | **发现问题** |
| 正确工具被错误移出前沿 | **工具前沿问题** |
| 候选都有但排序选错 | **选择问题** |
| 工具选对却不满足执行条件 | **准入问题** |
| 工具放行后业务结果错误 | **Handler 或副作用控制问题** |

> **发现问题要改工具目录（Catalog）或工具搜索（Tool Search）**，准入问题则应检查状态证据和策略。

---

## 九、为什么判断点和执行点必须分开

工具调度继承了一组安全系统中的经典分工：

| 组件 | 角色 | 描述 |
|------|------|------|
| **策略决策点 PDP**（Policy Decision Point）| **判断** | 做出 allow / deny / await 判断 |
| **策略执行点 PEP**（Policy Enforcement Point）| **挡在 Handler 门口** | 保证拒绝结果**不能被绕过** |

> 教学 Repo 中，两者放在同一个 `dispatch()` 里。**生产系统则可以把 PDP 放到独立策略服务，PEP 仍要跟着 Handler**。因为业务代码不应该绕过 Dispatcher，直接取得付款函数。

### 与 Saltzer-Schroeder 两条原则呼应

- **最小权限** → 最简工具集承接
- **完全仲裁** → 工具调度承接

### 软件结构血统

- GoF 的 **Command** 模式：行动先被表示成对象
- **Strategy** 模式：执行策略可以替换，调用者不必直接抓住具体函数

> Agent 的工具调用原则继承自这些经典思想（**老原则解决新问题的情形**），而**模型则是根据自然语言动态生成行动意图**（AI 时代带给我们的改变）。

---

## 十、工具元数据要有人真正消费

```python
@dataclass
class ToolMetadata:
    name: str
    description: str
    when_to_use: str
    when_not_to_use: str = ""
    exclusive_with: list[str] = field(default_factory=list)
    is_read_only: bool = False
    is_concurrency_safe: bool = False
    is_destructive: bool = False
    requires_fresh_state: bool = False
    requires_approval: bool = False
    quota_per_session: int = -1
    rollback_action: str | None = None
    risk_level: RiskLevel = RiskLevel.LOW
    is_mcp: bool = False
```

| 字段分组 | 字段 |
|---------|------|
| **帮助发现与选择** | `description` / `when_to_use` / `when_not_to_use` |
| **声明互斥关系** | `exclusive_with` |
| **描述执行特征** | `is_read_only` / `is_destructive` / `is_concurrency_safe` |
| **参与本次准入** | `requires_fresh_state` / `requires_approval` / `quota_per_session` |
| **描述失败后补偿入口** | `rollback_action` |

> **字段写在对象里，不等于运行时已经执行**。当前参考实现会读取工具存在性、配额、fresh state、审批、只读与破坏性属性，以及补偿动作。`risk_level` / `is_mcp` / `exclusive_with` / `is_concurrency_safe` 还没有进入准入分支。

### 10.1 注册时先拒绝自相矛盾的契约

```python
def register(self, meta: ToolMetadata, handler: Handler) -> None:
    if meta.is_destructive and meta.rollback_action is None:
        raise ToolDispatchError(
            f"destructive tool {meta.name!r} must declare rollback_action"
        )
    if meta.is_destructive and meta.is_read_only:
        raise ToolDispatchError(
            f"tool {meta.name!r} cannot be both read_only and destructive"
        )
    self.tools[meta.name] = meta
    self.handlers[meta.name] = handler
```

> **破坏性工具必须声明补偿入口**，同一个工具也不能既标只读又标破坏性。**这类问题属于配置错误，最好在启动或注册阶段直接失败**。
>
> 不过，这段检查只证明 `rollback_action` 填了值，还没有证明对应 Handler 已注册，更没有证明它在业务语义上真能恢复原状。生产注册中心还应校验**补偿入口可解析、参数兼容**，并把无法自动补偿的动作明确标成"不可逆"或"需要人工处置"。**随手填一个假的补偿入口，比坦白没有补偿更危险**。

### 10.2 MCP Tool Annotations

2026 年 3 月，MCP 官方对 Tool Annotations 作了说明：

- `readOnlyHint` / `destructiveHint` / `idempotentHint` / `openWorldHint`
- 提供了一套风险词汇，由工具调用方提供给模型参考
- 但它们仍然只是**参考**。第三方 Server 可以声称自己为只读，但**客户端不能完全据此自动授予权限**
- **元数据提供判断材料，本地策略与隔离机制才负责执行保证**

---

## 十一、施加生产压力

> 单机 demo 能跑，不代表生产安全。下面用 S1、S2、S3、S4 四种压力找出哪些地方会因并发、状态变化、进程重启和补偿失败而失效。

### S1：并发配额竞态

```text
Worker A 和 Worker B 进行了 fresh read
   ↓
系统进行当前会话配额检查（quota = 0）
   ↓
两边都认为"现在还没有付款，可以执行"
   ↓
Worker A 付款成功
   ↓
Worker B 也付款成功
   ↓
系统写回 quota = 1
```

**真实付款次数 = 2，系统记录次数 = 1**。这就是典型的**并发竞态（Race Condition）**。

> 会话配额只能解决"Agent 不要调用太频繁"。但是支付系统还需要解决"**同一个业务动作重复执行怎么办**"的问题——这叫**业务幂等（Idempotency）**。

**业务幂等**意味着：同一个请求即使发送两次、十次，最终效果也只能产生一次。例如工资支付，生成一个唯一业务编号：

```text
goal_id + employee_id + payroll_month
   ↓
bonus_release_2026_06 + E0007 + 2026-06
   ↓
UUID: bonus_release_2026_06_E0007
```

通过数据库唯一约束**保证这个编号只能成功创建一次**：

```text
第一次请求：创建成功 → 执行付款
第二次请求：发现编号已存在 → 查询第一次结果 → 不再次付款
```

> **生产系统保护支付的基本方式**：把 `goal_id + employee_id + payroll_month` 组成稳定幂等键，**在共享数据库中用唯一约束原子占位**。第二个 worker 如果发现记录已经存在，**应查询原动作结果，不能再次付款**。

### S2：读取后状态变化（TOCTOU）

```text
10:00
   工资 = 9,600 元
   系统记录"刚刚读取，可以使用"
   ↓
10:01
   另一个流程修改工资 = 99,600 元
   Agent 不知道，仍拿着旧数据执行付款
```

这就是 **TOCTOU（Time Of Check To Time Of Use）**——**检查时间和使用时间之间发生了变化**。

> 解决方法不是简单记录"我一分钟以前读过"。因为时间不能证明数据没有变化。**更可靠的方法是给数据绑定版本**。

```python
@dataclass(frozen=True)
class StateEvidence:
    resource_ref: str
    version: int
    observed_at: datetime
    content_hash: str
```

> 真正执行写入时，再检查现在版本是不是还是 17。如果版本一致，说明中间没有变化，可以继续。如果版本变成 18，**就说明别人修改过，必须重新读取**。

> 这种"**只有版本匹配才允许更新**"的方法叫**比较后交换（Compare-and-Swap）**，很多数据库和并发系统都使用这个思想。

### S3：进程重启失忆

```python
quota = {}        # 当前配额
fresh_state = {}  # freshness 记录
saga_log = {}     # 补偿记录
```

这些状态都存在内存字典里。**程序运行时这些信息都存在。但是服务器重启后，新的 Dispatcher 启动，内存全部清空，系统就不再知道刚才有没有付款**。

> 生产系统需要一个**行动账本（Action Ledger）**，它不是存在内存里的临时变量，而是持久化数据库。里面记录：
> - 幂等键
> - 是否允许执行
> - 使用的数据版本
> - 执行结果
> - 外部系统返回值
> - 补偿状态

> **系统重启后的第一件事是先查账**，看看这件事情以前发生过没有？查账之后再决定恢复还是重试。否则第一次付款后创建一个新实例，**新进程看不见旧状态，同一动作会再次通过**。

**外部结果未知态**：处理 Agent 调银行接口，请求发送出去，然后网络断开。系统收到 timeout。这时候不能简单认为失败，也不能认为成功，**真实状态可能是 UNKNOWN**。银行可能已经收到请求，只是回执没有回来。所以系统必须**根据幂等键查询银行结果，得到"结果确定性"，不能直接重试**；否则一次付款可能变成两次。

### S4：补偿债务丢失

```python
transfer_salary()    # 付款函数
reverse_transfer()   # 补偿函数
```

> 分布式系统通常使用 **Saga 补偿模式**——如果前一步不能原子撤销，就执行一个业务补偿动作。

**陷阱**：如果补偿失败但系统忘记了

```text
付款成功 → 记录补偿任务 → 执行冲正 → 冲正失败 → 删除记录
```

工资状态是 `PAID`，但冲正为 `FAILED`，**这时系统必须保留这个补偿失败的状态**，否则以后没有人知道"还有一笔钱需要处理"。

```text
PENDING_COMPENSATION
  → COMPENSATING
  → COMPENSATED
  → COMPENSATION_FAILED
    → MANUAL_REVIEW / REDO
```

> **失败记录要保留重试次数、最后错误、下次重试时间和负责人**，直到补偿完成或人工结案。

---

## 十二、从教学 Dispatcher 走向生产

```python
class DurableToolDispatcher:
    def dispatch(
        self,
        intent: ActionIntent,
        contract: ActionContract,
        state_evidence: list[StateEvidence],
        approval: ApprovalTicket | None,
    ) -> ActionResult:
        tool = self.selector.select(intent, contract)
        decision = self.policy.decide(
            tool, intent, state_evidence, approval
        )
        self.ledger.append(decision.to_event())
        if not decision.allowed:
            return ActionResult.blocked(decision)
        lease = self.idempotency.acquire(intent.idempotency_key)
        if not lease.acquired:
            return self.ledger.lookup_result(intent.idempotency_key)
        result = self.executor.run(tool, intent.normalized_args)
        self.ledger.commit_result(lease, result)
        return result
```

| 抽象 | 作用 |
|------|------|
| **行动契约（Action Contract）** | 约束目标与范围 |
| **审批票据（Approval Ticket）** | 绑定批准人、参数和资源版本 |
| **幂等层** | 返回的 **lease 是执行租约**，只有成功占位的一方可以调用真实工具 |
| **持久账本** | 记录幂等键、资源版本、外部回执、补偿状态 |

> **它没有要求每个 Agent 一上来就建设分布式事务平台**。只有两三个只读工具、失败后可以直接丢弃时，注册表加参数校验已经够用。
>
> **支付、删除、发消息和运维变更进入系统以后，控制才需要逐级加厚**。风险越高、副作用越难恢复，越值得把决策点、执行点和证据账分开。

---

## 十三、监控表：怎样知道该修哪一层

| 指标 | 它帮助定位什么 |
|------|-------------|
| **工具发现召回率** | 正确工具有没有被搜索到 |
| **工具前沿召回率与大小** | 必要工具有没有被误删，当前候选是否过多 |
| **候选选择准确率** | 正确工具排在什么位置 |
| **准入拒绝原因** | 因状态、次数、审批还是权限被拒绝 |
| **重复副作用数** | 同一幂等键实际提交了几次 |
| **状态版本冲突** | fresh read 以后对象又变化了多少次 |
| **补偿债务年龄** | 未完成补偿积压了多久 |

> 只看"工具成功率"，发现、选择、准入和执行都混成了一个数字。**分阶段指标才告诉团队该改工具描述、改工具前沿、改策略，还是修 Handler**。

---

## 十四、总结

### 14.1 行动模块第 22 讲回顾

这一讲沿着第 21 讲留下的两类问题，装上了两层控制：

| 层 | 模式 | 作用 |
|----|------|------|
| **第一层** | **最简工具集** | 先为发薪任务建立当前工具前沿，修改账号和清除异议记录因此**不可达** |
| **第二层** | **工具调度** | 再读取工具契约，检查注册、调用次数、状态新鲜度和审批条件 |

错误的付款被拒绝以后，**Harness 回到北极星目标**，先查询，再完成一笔正确付款。

L0-L1-L2 的数据库终态不足以独立证明这件事，**动作流水补上了重复副作用的证据**。四种生产压力又告诉我们，教学版的配额、时间戳和内存 Saga 还不是生产保证。

| 生产压力 | 解决方法 |
|---------|---------|
| **S1 并发配额竞态** | **业务幂等**（goal_id + employee_id + payroll_month）|
| **S2 TOCTOU 状态变化** | **资源版本 + Compare-and-Swap** |
| **S3 进程重启失忆** | **持久行动账本（Action Ledger）** |
| **S4 补偿债务丢失** | **Saga 持久状态 + 人工结案** |

### 14.2 精华浓缩

> **先缩小能做什么，再判断这次能不能做，做完以后还要能证明发生了什么**。

### 14.3 行业动向

- **Anthropic Tool Search Tool**（2025-11）：允许大量工具使用 `defer_loading` 延迟加载，把"系统拥有多少工具"和"本轮看见多少工具"**拆开**，正好对应最简工具集的**渐进发现**
- **OpenAI Agents SDK Tool Guardrails**：在自定义函数工具执行前后检查并阻断。**文档明确列出覆盖边界**：托管工具、内置执行工具和 handoff 不自动走同一条 Tool Guardrail 管线
- **MCP Tool Annotations**（2026-03）：补风险词汇

> **这些进展共同说明，工具调用正在从"模型返回一个函数名"走向三个互相配合的层次：按需发现、风险描述和运行时执行边界**。任何一层都不能单独承担全部安全保证。

---

## 十五、思考题

1. 找一条真实工具误调用记录。它属于**工具没被发现、前沿错误、选择错误、准入拒绝，还是 Handler 执行错误**？你们当前日志能否区分？
2. 为一个有副作用的工具写出**三项准入条件**。哪些条件只靠时间戳不够，必须绑定**对象版本或审批票据**？
3. 给一笔薪酬付款设计**业务幂等键**。进程在远端付款成功、写本地回执之前崩溃，新进程应该查什么，**绝不能直接做什么**？
4. 让补偿连续失败三次。补偿债务存在哪里，何时重试，什么时候转人工，**谁是负责人**？

---

## 十六、下一讲预告

工具调度管住了**一次调用**。长任务还有另一种麻烦：

> **E0007 已经付款，E0012 调用网关时超时，恢复指令却要求整批重来**。**单次工具准入都合法，整批重跑仍可能重复付款**。

接下来的模式继续解决不同层级的问题：

| 模式 | 它继续解决什么 | 与工具调度的接缝 |
|------|--------------|---------------|
| **[[lession23-规划执行\|规划执行]]** | 多步任务怎样保持全局目标与局部恢复 | 每个计划步骤内部调用 Dispatcher |
| **[[lession24-提示链\|提示链]]** | 上一步工件怎样不污染下一步 | 工件通过段间闸门后才形成行动意图 |
| **[[lession25-护栏三明治\|护栏三明治]]** | 高风险动作前后分别检查什么 | 准入通过后仍经过 PRE、TOOL、POST |

下一讲进入**规划执行**。我们会先冻结已经完成的事实，再把重规划限制在失败步骤及其依赖范围内，让**局部故障不再拖着整条任务从头来过**。

---

## 精选留言摘录

> **街角·陌路△**：
> 这集出现了 Agent 设计模式和传统设计模式交汇，也出现了 Agent 之前的软件架构和 Agent 的软件架构交汇。**Agent 的开发是在重塑传统软件，而这前提一定是对传统软件有深刻的理解**。传统软件开发转 Agent 开发。这节就做了一个很好的例证，**只有真的做过软件的人才会考虑并发幂等性等问题**。
>
> **作者回复**：说得好。**Agent 工程不会抹掉并发、幂等、事务和契约这些旧功夫，反而会让它们重新变成底座**。模型负责提出候选，传统软件机制负责把副作用守住。

> **熊伟**：
> 我觉得这章是在讲 agent 分布式事务。可以按传统的框架加分布式事务锁，补偿事务等操作；也可以加一个独占锁，失败回滚，再重新操作。
>
> **作者回复**：你看到了相邻的工程血脉，**但工具调度本身只是"这次调用能不能进门"，还不是完整分布式事务**。**锁、幂等和补偿应由业务系统或工具端保证**，Harness 负责携带意图键、记录回执并协调重试，避免在上层再造一套全局锁。

> **Helios**：
> 感觉增加工具调度，让 agent 的难度又上了一个台阶，和意图识别一样还要评估选的对不对、选错了怎么办。
>
> **作者回复**：是的，**所以工具调度也要评测**。离线看选对率、误拒率和无工具召回，线上看拒绝原因、回退率和人工改派率。**先用规则收窄候选，再让模型处理少量模糊项**，会比让它面对全量工具稳得多。

> **Helios**（第二弹）：
> 老师，工具调度会不会导致缓存失效呀。现在的场景是默认一次性场景，但是对于多轮对话中，每一个用户 query 都有可能调用不同的工具呀。
>
> **作者回复**：**要区分工具目录缓存和工具结果缓存**。目录可按工具版本、用户权限和策略版本失效；查询结果还要带参数、租户、数据版本和新鲜度。**写操作一般不缓存，执行前仍应做 fresh read**。

> **红白十万一只**：
> 这个案例把 workflow 和 harness 绑定了，修改 workflow 的时候 harness 也需要跟着变化。实际公司的业务流较多，变化也会比较频繁。harness 变化也可能影响到其他的 workflow。有没有 MCP、workflows、harness 三层隔离的方案，这样 MCP 和 workflows 层随业务编排修改，harness 迭代自身完善都能实现隔离？
>
> **作者回复**：**可以分三层**：
> - **MCP** 管稳定能力契约
> - **Workflow** 管业务次序和补偿
> - **Harness** 管身份、预算、准入、重试和留痕
>
> **三层通过元数据和回执接线**，Harness 不写死某条流程名，流程变化才不会牵动全部运行时。

---

## 参考资料

- ADPS 模式白皮书 https://adpsagent.com/zh/patterns/
- Gamma, E., et al. *Design Patterns: Elements of Reusable Object-Oriented Software*. Command 与 Strategy 模式. 1994.
- Saltzer, J. H. & Schroeder, M. D. *The Protection of Information in Computer Systems*. 1975. https://web.mit.edu/Saltzer/www/publications/protection/
- Anthropic. *Introducing advanced tool use on the Claude Developer Platform*. 2025-11-24. https://www.anthropic.com/engineering/advanced-tool-use
- Model Context Protocol. *Tool Annotations as Risk Vocabulary: What Hints Can and Can't Do*. 2026-03-16. https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/
- OpenAI Agents SDK. *Guardrails: Tool guardrails and workflow boundaries*. https://openai.github.io/openai-agents-python/guardrails/