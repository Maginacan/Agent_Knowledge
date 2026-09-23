---
title: "10｜多模态融合：日志、SQL 和 PDF 一起进 Agent"
column: "Agent 设计模式之美"
author: "黄佳"
source: "https://time.geekbang.org/column/article/986182"
tags:
  - "#pattern/perception"
  - "#axis/perception"
  - "#axis/parallel"
  - "#topic/multi-modal"
  - "#topic/vision-api"
  - "#topic/pdf-processing"
  - "#topic/lazy-loading"
  - "#concept/decision-card"
  - "#concept/atomic-tools"
  - "#concept/observability"
  - "#concept/data-shape"
audio: true
duration: "25:05"
publish_date: "2026-06-12"
course_progress: 10
pattern_position: "感知 × 并行"
module: "感知：世界之美（第 5 讲 / 共 5 讲）"
related:
  - "[[lession9-渐进发现]]"
  - "[[lession11-记忆模块导论]]"
---

# 10｜多模态融合：日志、SQL 和 PDF 一起进 Agent

> **本讲定位**：感知模块最后一个模式 → 解决"信息进来前应该先变成什么形态"。

| 属性 | 内容 |
|------|------|
| 课程模块 | 感知：世界之美（第 5 讲 / 共 5 讲）|
| 课程编号 | 10 |
| 双轴坐标 | **感知 × 并行**（认知功能：感知 / 执行拓扑：并行）|
| 时长 | 25:05 |
| 发布日期 | 2026-06-12 |
| 前置讲 | [[lession9-渐进发现]] — 探索未知空间 |
| 后续讲 | [[lession11-记忆模块导论]] — 跨会话经验沉淀 |

## 一句话核心

> **多模态融合（Multi-Modal Fusion）= 先判断每一种数据最适合以什么形态被模型消化，再把它们带着关联关系合并到推理层。让 Agent 看到恰当的少。**

---

## 一、从反例到动机：纯文本路线的崩塌

### 1.1 上一讲的尾巴与本讲的开头

> 前面三讲，我们一直在讲 Agent 怎么"**看对东西**"：
>
> - 第 7 讲 [[lession7-上下文分诊]] — 学习哪些信息先进来
> - 第 8 讲 [[lession8-语义压缩]] — 进来以后怎么压缩
> - 第 9 讲 [[lession9-渐进发现]] — 不知道在哪儿的信息怎么探索
>
> **今天这一讲再往前走一步：Agent 接收到的信息，很多时候一开始就不是同一种形态，我们需要想想如何整合。**

### 1.2 编辑的类比

> 一个老练的编辑产出一篇财经类稿件，他不会全做成纯文字。他知道什么该用图、什么该用表、什么该用文字。
>
> - **市场规模趋势用折线图**（空间关系是信号）
> - **财务数据用表格**（结构是信号）
> - **分析观点用文字**（逻辑是信号）

> **报纸编辑整理多模态信息的这种敏感度，Agent 工程师和 Agent 也需要好好学习**。

### 1.3 人的多模态世界模型

> 人的大脑有一个隐含的"**世界模型**"，会把不同感官来的信号拼在一起：
>
> - **文字**告诉我们逻辑
> - **图表**告诉我们关系
> - **表格**告诉我们结构
> - **声音**告诉我们情绪
> - **日志**告诉我们过程
>
> 哪条通道里的信号更关键，大脑会自动调权重。

### 1.4 开篇反例：金融研报分析 Agent

| 项目 | 内容 |
|------|------|
| 场景 | 一家券商需要做研报分析 Agent |
| 输入 | 给定一份 **80 页的 PDF 行业研报** |
| 输出 | 三类结果：核心论点摘要、所有数字结论的事实核查、给客户经理的销售要点提炼 |

**三种最容易想到的方式为什么不行**：

| 方式 | 问题 |
|------|------|
| **① 大一统（all in one prompt）** | 把 PDF 完整塞进 Claude。摘要部分可能还 OK，但图表理解和数字核查很可能出问题。例：把"市场规模 5800 亿"误报成"**5800 万**" |
| **② 全转文本** | OCR + PDF 文本提取成 markdown 喂进去，**所有图表的空间信息丢光了**。Agent 只看到"图 4.2 市场规模趋势图"这行字，看不到图本身 |
| **③ 暴力塞整份 PDF** | 80 页研报如果暴力塞，可能接近 90K token。**成本 + 稳定性双重压力** |

> 上面场景和提示词够不够好、模型强不强关系不大，**根因是数据形态不对**。

### 1.5 解决思路

| 数据形态 | 处理方式 |
|----------|----------|
| **文本部分** | 用 PDF 解析器（如 Unstructed）抽取文本骨架（TOC、章节摘要、关键句子）|
| **表格** | 用 Tabula 转成 markdown 格式 |
| **关键图表** | 保留为图片，通过 API 传给多模态模型 |
| **装饰图** | 公司 logo、模板图标，直接丢弃 |

> 在这个过程中，**图、文、表这些信息还需要通过元数据来进行链接**，让 Agent 知道它们之前的关联。

---

## 二、多模态融合模式定义

> 多模态模式处理的是这样一类问题：
>
> **Agent 接到的输入不再只是纯文本**，而是同时包含图片、文字、表格、日志、PDF、截图，甚至音频和视频。
>
> 工程师要做的是**先判断每一种数据最适合以什么形态被模型消化，再把它们带着关联关系合并到推理层**。

### 2.1 在双轴图谱的位置

| 轴 | 类型 | 原因 |
|----|------|------|
| 认知功能 | **感知** | 它决定 Agent 最终看到什么 |
| 执行拓扑 | **并行** | 多种异构数据源会同时进入系统，每一种都应该走最适合自己的处理路径 |

### 2.2 与前三讲的关键区别

| 模式 | 处理什么 |
|------|----------|
| [[lession7-上下文分诊]] | 哪些 token 进来 |
| [[lession8-语义压缩]] | 进来后怎么压 |
| [[lession9-渐进发现]] | 怎么找 |
| **多模态融合** | **数据进入 Agent 之前，应该先变成什么形态** |

### 2.3 何时该用 / 何时不用

| 触发条件 | 适用场景 |
|----------|----------|
| ✅ 输入不是干净纯文本 | 金融研报分析、运维日志诊断、PDF 合同审阅、客户工单截图分析、代码评审架构图理解、医疗影像+病历联合分析 |
| ❌ 输入本身就是干净文本 | 简单客服问答、文本翻译、短代码补全 |

> **真实生产系统里，纯文本输入正在变少，混合形态输入才是常态**。
>
> ⚠️ 常见误判：很多看起来是文本任务的场景，实际运行时用户会上传截图、贴日志、转发邮件、附 PDF。**只要这些异构材料出现，多模态融合就已经进入系统了**。

### 2.4 黄金原则

> **空间关系本身是信号，就保留为图；否则，尽量转成紧凑、可检索、易压缩的文本或者清晰的结构（如 JSON Schema）**。

| 输入类型 | 推荐处理 |
|----------|----------|
| 架构图、流程图、UI 设计稿 | 保留为图（价值在空间关系）|
| 表格、字段截图、错误日志、合同条款 | 转 markdown / JSON / 结构化文本 |

---

## 三、工程现场切片

### 切片对比总览

| 切片 | 来源 | 核心机制 |
|------|------|----------|
| 切片一 | Claude Vision API | **token 数学**：`tokens = width × height / 750` |
| 切片二 | Hermes Agent | **lazy load** + 多模态融合工程（含音频 STT）|

### 切片一：Claude Vision API 的 token 数学

> 很多工程师对"保留为图"还是"转成文本"的成本判断并不准确。

**Anthropic Vision API 的 token 计算方式**：

```
tokens = width × height / 750
```

| 图片规格 | 估算 token |
|----------|------------|
| 1024 × 1024 | 约 1400 token |
| 一段 1500 字中文 markdown | 约 800 token |

> **图片并没有想象中那么贵**。一张 1024×1024 的图 ≈ 1.5 倍同长度文本。

**架构图示例对比**：

| 表示方式 | token 数 |
|----------|----------|
| 5 组件 + 8 连线的图片 | ~1400 |
| 文字描述（800-1200 字）| 400-600 |

> **保留为图片反而更划算，也更不容易丢空间关系**。

**PDF 的情况就不同**：

| 文档 | token 消耗 |
|------|------------|
| 30 页文本密集型 PDF | 5-6 万 token |
| 80 页研报整份暴力喂 | 容易超过 **15 万 token** |

**关键变量：prompt caching**

> 重复使用的 PDF、图片或长上下文，如果能走 cache，**后续调用成本会明显下降**。
>
> 对于合同审阅、研报分析这类场景，同一份文件往往会被多次问答、摘要、核查。
>
> **把可复用内容放进 cache，可能直接改变这个 Agent 的经济模型**。

**工程启发**：

> **统计你的 Agent 每天处理多少重复文档**。如果同一份 PDF、合同、研报被反复调用，cache 通常值得优先接入。

### 切片二：Hermes Agent 多模态融合的最完整工程

> Hermes Agent 的价值不只在于能处理图文，还在于**它把音频输入也纳入了 Agent 流水线**：
>
> - 支持**麦克风实时录音**
> - 支持 **mp3 / wav 文件输入**
> - 适配 macOS、Linux、Windows、WSL 等环境

**最值得注意的设计：lazy load**

```python
# Lazy audio imports -- never imported at module level to avoid crashing
# in headless environments (SSH, Docker, WSL, no PortAudio).
```

**为什么必须 lazy load**？

> Agent 会运行在各种环境里：本机、CI、Docker、远程 SSH、WSL。
>
> **很多环境没有音频设备，也没有 PortAudio**。如果启动时就硬 import 音频库，Agent 可能在还没处理任何音频任务之前就直接崩掉。

**Hermes 的做法**：

- 把这些非纯文本能力**延迟加载**
- 只有用户真的使用音频功能时，才加载相关库
- 如果当前环境不支持，**就优雅降级**，而不是影响整个 Agent 启动

**适用原则（不仅限于音频）**：

| 模块类型 | 处理 |
|----------|------|
| **80% 用户不会用到** + 对运行环境有依赖的库 | 应考虑放到函数内部 lazy import，或用 try-except 做优雅降级 |
| 例子 | `cv2`、`pydub`、`pdfplumber`、`transformers` |

> **这一步看起来很小，但对 Agent 的部署兼容性非常关键**。

---

## 四、8 框架横切：每家怎么做多模态

> 把 8 个 Agent Harness 放在一起看，**多模态融合这一栏的差异非常明显**。

| 投入程度 | 框架 | 场景特征 |
|----------|------|----------|
| **重投入** | Claude Code / Hermes / Gemini CLI | 通用助理、客服、研究分析、金融研报 |
| **几乎不投入** | Codex CLI / Aider | 编程 Agent（主战场是 repo search、edit loop、test loop）|

**核心判断**：

> **多模态投入和 Agent 场景强相关**。
>
> - 编程 Agent 的主战场是代码、文件、终端、测试和 git diff → 不一定需要重点关注多模态
> - 通用助理、客服、研究分析、金融研报 → **多模态融合不是锦上添花，而是基础能力**

> 所以问题不是"框架有没有多模态"，而是**你的 Agent 面对的世界，是不是本来就是多模态的**。

### Gemini CLI 的另一条路线

> Gemini CLI 的大上下文和视频能力代表了另一条路线：
>
> **模型窗口足够大时**，可以减少一部分预处理，让模型直接接收更多原始材料。
>
> 这在视频、复杂图文混合材料里有价值，因为手工抽帧、切片、转写、对齐本身成本很高。

**但这不代表所有输入都该直接塞进大窗口**：

> 长日志、SQL 大结果、海量表格、批量 PDF **仍然需要流水线处理**。
>
> **窗口变大，只是提高了上限，不会自动解决成本、噪声、延迟和可控性问题**。

**成熟做法**：

> 先判断业务输入的形态，再决定多模态的投资深度。
>
> - **不要因为模型支持多模态，就把所有东西原样塞进去**
> - **也不要因为系统当前是文本 Agent，就假设用户永远不会上传截图、PDF 或日志**

---

## 五、工业级实现：可观测多模态融合的最小骨架

> 多模态融合不能只是一个 `parse_file()` 函数。
>
> **生产里的融合器（fuser）至少要做三件事**：
>
> 1. **识别输入形态**
> 2. **按形态分发到不同处理路径**
> 3. **记录每次处理的 trace**，方便后续排查成本、延迟和质量问题

### 5.1 七种输入形态枚举

```python
from dataclasses import dataclass, field
from enum import Enum
from typing import Any, Callable, Optional
from datetime import datetime

class ModalityType(Enum):
    TEXT = "text"              # 直接进入上下文
    IMAGE = "image"            # 空间关系是信号，保留为图
    TABLE = "table"            # 转 markdown
    LOG = "log"                # bash 预过滤 + sub-agent
    PDF = "pdf"                # TOC + 关键页 + 抽取文本
    AUDIO = "audio"            # STT 转文本
    SQL_RESULT = "sql_result"  # 抽样 / compact table
```

### 5.2 业务提示对象

```python
@dataclass
class ModalityInput:
    type: ModalityType
    payload: Any
    hint: str = ""           # 业务含义，如"市场规模图" / "auth 服务日志" / "用户上传截图"
    keep_as_image: bool = False
```

> **`hint` 很重要**，它告诉系统这份材料的业务含义。

### 5.3 融合事件（FusionEvent）

> **多模态融合一定要可观测**，否则你只知道 token 涨了，却不知道是图片涨了、PDF 涨了，还是日志预过滤失效了。

```python
@dataclass
class FusionEvent:
    modality: ModalityType
    tokens_out: int
    processing_ms: int
    method: str              # direct / vision / table_to_md / pdf_extract / bash_filter+subagent / stt / fallback
    timestamp: str = field(
        default_factory=lambda: datetime.utcnow().isoformat()
    )
```

### 5.4 核心类 MultiModalFuser（原子工具注入）

```python
class MultiModalFuser:
    def __init__(
        self,
        ocr_tool: Optional[Callable] = None,
        stt_tool: Optional[Callable] = None,
        pdf_extract: Optional[Callable] = None,
        log_subagent: Optional[Callable] = None,
        bash_filter: Optional[Callable] = None,
    ):
        self.ocr = ocr_tool
        self.stt = stt_tool
        self.pdf_extract = pdf_extract
        self.log_subagent = log_subagent
        self.bash_filter = bash_filter
        self.events: list[FusionEvent] = []
```

> 用的是**原子工具注入**，而不是把 OCR、STT、PDF 解析、日志分析全部写死在类里。
>
> 这样同一套融合器可以跑在不同环境：**本地 OCR、云端 OCR、企业内部 PDF 解析器**，都可以替换。

### 5.5 核心分发逻辑

```python
def fuse(self, inputs: list[ModalityInput]) -> dict:
    content_blocks = []
    for inp in inputs:
        t0 = datetime.utcnow()
        if inp.type == ModalityType.TEXT:
            output = {"type": "text", "text": inp.payload}
            method = "direct"
        elif inp.type == ModalityType.IMAGE or inp.keep_as_image:
            output = self._as_image_block(inp.payload)
            method = "vision"
        elif inp.type == ModalityType.TABLE:
            md = self._table_to_markdown(inp.payload)
            output = {"type": "text", "text": md}
            method = "table_to_md"
        elif inp.type == ModalityType.PDF:
            extracted = self.pdf_extract(inp.payload)
            text = self._build_pdf_summary(extracted, inp.hint)
            output = {"type": "text", "text": text}
            method = "pdf_extract"
        elif inp.type == ModalityType.LOG:
            filtered = self.bash_filter(inp.payload, inp.hint)
            structured = self.log_subagent(filtered)
            text = self._format_log_structured(structured)
            output = {"type": "text", "text": text}
            method = "bash_filter+subagent"
        elif inp.type == ModalityType.AUDIO:
            transcript = self.stt(inp.payload)
            output = {"type": "text", "text": transcript}
            method = "stt"
        else:
            text = str(inp.payload)[:5000]
            output = {"type": "text", "text": text}
            method = "fallback"
        content_blocks.append(output)
        tokens = len(str(output)) // 4
        self.events.append(FusionEvent(
            modality=inp.type,
            tokens_out=tokens,
            method=method,
            processing_ms=int(
                (datetime.utcnow() - t0).total_seconds() * 1000
            ),
        ))
    return {
        "content": content_blocks,
        "total_tokens_estimate": sum(e.tokens_out for e in self.events),
        "fusion_trace": self.events,
    }
```

### 5.6 关键工程决策

| # | 决策 | 含义 |
|---|------|------|
| 1 | **`keep_as_image=True`** 强制保留图片 | 默认规则能覆盖大多数情况，但**双 Y 轴图、架构图、UI 设计稿** —— 空间关系本身就是信号，不能简单转成文字 |
| 2 | **日志必须走流水线** | `bash_filter → log_subagent → structured summary` —— 长日志不能直接进主上下文 |
| 3 | **PDF 不应该默认整份塞进去** | `TOC + key_pages + business_hint` —— 先提取目录、关键页、章节摘要，再按任务选择关键图表和表格 |

### 5.7 健康检查

```python
def health_check(self) -> dict[str, str]:
    if not self.events:
        return {"status": "no fusion events"}
    report = {}
    total = sum(e.tokens_out for e in self.events) or 1
    for modality in ModalityType:
        tokens = sum(
            e.tokens_out for e in self.events
            if e.modality == modality
        )
        ratio = tokens / total
        if modality == ModalityType.IMAGE and ratio > 0.5:
            report["image_token_overshoot"] = (
                f"image tokens = {ratio:.1%}, "
                "check whether some charts should be tables or markdown"
            )
        if modality == ModalityType.LOG and ratio > 0.4:
            report["log_token_overshoot"] = (
                f"log tokens = {ratio:.1%}, "
                "bash filtering may not be working"
            )
    return report
```

**健康检查的价值**：

| 异常 | 可能原因 |
|------|----------|
| `image_token_overshoot`（image 占比 > 50%）| 有大量表格或装饰图被当成图片保留 |
| `log_token_overshoot`（log 占比 > 40%）| bash 预过滤没有生效 |
| PDF token 持续过高 | 关键页抽取规则太松 |

> **没有 FusionEvent，多模态融合就是黑盒**。
>
> 出了问题以后，你只知道 Agent 变贵、变慢、变不准，却不知道是哪一种输入形态在拖垮系统。

---

## 六、业务级实现：金融研报分析 Agent

> **前面的骨架版代码说明了 Multi-Modal Fusion 的基本结构**。现在把它放进一个真实业务场景。

### 6.1 业务场景

| 项目 | 数值 |
|------|------|
| 输入 | 80 页 PDF 研报 |
| 文件大小 | 14 MB |
| 三类输出 | 核心论点摘要 / 数字结论核查 / 销售要点 |

> **最容易犯的错误**：把整份 PDF 原样交给模型。
>
> 这样做看起来简单，但成本高、噪声大，而且图表和数字核查容易出错。
>
> 正确做法是**先做多模态拆解**，把不同形态的数据转成最适合推理的表示。

### 6.2 步骤一：融合层分发

**Agent 接到 PDF 后，先由 MultiModalFuser 拆分输入**：

| 部件 | 处理方式 | 估算 token |
|------|----------|------------|
| **PDF 主体** | 抽 TOC、章节摘要、关键页 | TOC ~120 + 关键页 3×2K ≈ **6K** |
| **关键图表** | 识别市场规模、市占率、增长趋势、估值、渗透率等 → 保留为图 | 5 张 × 1.4K ≈ **7K** |
| **表格** | 全部转 markdown | 12 张 × 200 ≈ **2.4K** |
| **装饰图** | logo、模板图标、章节封面、页脚装饰 → **直接丢弃** | 0 |
| **合计** | Fusion 总产出 | **~16K** |
| 对照 | 暴力全喂 | ~90K |

**业务对象 `ResearchReport`**：

```python
@dataclass
class ResearchReport:
    pdf_path: str
    report_type: str      # industry / company / macro
    industry: str
    target_audience: str  # institutional / retail / broker
```

**关键图表识别规则 `CriticalChartSpec`**（行业知识）：

```python
@dataclass
class CriticalChartSpec:
    pattern_keywords: list[str] = field(default_factory=lambda: [
        "市场规模", "市占率", "增长趋势",
        "营收", "利润率", "毛利率", "ROE",
        "渗透率", "用户数", "ARPU",
        "估值", "PE", "PB", "PS",
    ])
```

> 这组关键词**不是普通配置，而是行业知识**。金融研报里，这些图表往往承载最关键的数字结论，所以要优先保留为图。

**输入拆解 `_build_inputs`**：

```python
def _build_inputs(self, report: ResearchReport) -> list[ModalityInput]:
    inputs = []
    # 1. PDF 主体：抽 TOC、关键页、章节摘要
    inputs.append(ModalityInput(
        type=ModalityType.PDF,
        payload=report.pdf_path,
        hint=f"{report.report_type} 研报，行业：{report.industry}",
    ))
    # 2. 关键图表：保留为图
    for chart in self._extract_critical_charts(report.pdf_path):
        inputs.append(ModalityInput(
            type=ModalityType.IMAGE,
            payload=chart["image_bytes"],
            hint=f"图 {chart['fig_no']}: {chart['caption']}",
            keep_as_image=True,
        ))
    # 3. 表格：转 markdown
    for table in self._extract_tables(report.pdf_path):
        inputs.append(ModalityInput(
            type=ModalityType.TABLE,
            payload=table["dataframe"],
            hint=f"表 {table['tab_no']}: {table['caption']}",
        ))
    return inputs
```

### 6.3 步骤二：三任务并行

**Fusion 产出的 content blocks 会被复用到三个任务里**：

| 任务 | 输出 |
|------|------|
| **摘要任务** | 800 字核心论点摘要 |
| **数字核查** | 抽取所有数字结论，要求给出 page 或 chart 引用 |
| **销售要点** | 根据目标受众，生成 5-7 条销售话术 |

**主流程**：

```python
def analyze(self, report: ResearchReport) -> dict:
    inputs = self._build_inputs(report)
    fused = self.fuser.fuse(inputs)
    summary = self._run_summary(fused["content"], report)
    fact_check = self._run_fact_check(fused["content"], report)
    sales_points = self._run_sales_points(fused["content"], report)
    return {
        "summary": summary,
        "fact_check": fact_check,
        "sales_points": sales_points,
        "fusion_trace": fused["fusion_trace"],
    }
```

**关键：数字核查必须强制引用**

```python
system = (
    "你是事实核查员。从研报中抽取所有数字结论，"
    "输出 JSON 列表，每项包含 claim / number / "
    "page_or_chart_ref / confidence。"
    "找不到 page 或 chart 引用的，confidence 标为 low。"
)
```

> 这条规则能明显降低"**数字说得很自信但没有出处**"的风险。
>
> **每个数字都必须挂到页码或图表**，后续人工核查才有据可依。

### 6.4 步骤三：结果合并

最后，三个产物合并成一份报告，交给客户经理：

| # | 内容 |
|---|------|
| 1 | 800 字摘要 |
| 2 | 数字核查清单 |
| 3 | 5-7 条销售要点 |
| 4 | **Fusion trace**（保留了多少张关键图 / 多少张表转成 markdown / 多少装饰图被丢弃 / 总 token 消耗）|

> **fusion_trace** 很重要。它记录了：
>
> - 保留了多少张关键图
> - 多少张表转成 markdown
> - 多少装饰图被丢弃
> - 总共消耗多少 token
>
> 这让质量审查有依据。后面如果某个数字核查错了，可以回头看：**是关键图没保留，还是表格没抽出来，还是引用规则没执行**。

### 6.5 三大业务工程决策

| # | 决策 | 含义 |
|---|------|------|
| 1 | **关键图表识别是核心** | CriticalChartSpec 写得好 → 关键图保留；写得差 → 重要图表被当装饰图丢掉。**这一步往往比 prompt 调优更重要** |
| 2 | **装饰图要丢掉** | 80 页研报可能有几十张图片，但真正有信息量的只有少数几张。采用**白名单 + 黑名单**：业务关键词命中保留，明显装饰性丢弃 |
| 3 | **同一份报告要复用** | 摘要、数字核查、销售要点基于同一份研报内容 → 配合**提示词缓存或批处理 API**，避免每个任务都重新喂一遍 PDF |

> 所以，金融研报 Agent 的关键**不在于"模型能不能读 PDF"，而在于工程师有没有把 PDF 拆成正确的业务形态**。
>
> 概括一下，就是：
>
> - **文本给逻辑**
> - **表格给结构**
> - **图表给空间关系**
> - **trace 给质量审查**

---

## 七、工业 trace：融合层的可观测性实战

> **我推荐三个可观测指标**。

### 三个核心指标

| 指标 | 含义 | 健康区间 | 异常信号 |
|------|------|----------|----------|
| **`token_distribution_by_modality`** | 按形态的 token 占比 | text 40-60% / image 10-30% / structured (table/json) 10-20% / log/Sub-Agent 摘要 5-15% | image 突然涨到 60% → 有该转的图没转换形态 |
| **`fusion_processing_p99_ms`** | 单次 fusion 总处理时间的 p99 | **< 5 秒** | 飙到 30 秒+ → 多半是 PDF 抽取或 OCR 卡住 |
| **`bash_filter_compression_ratio`** | bash 预过滤压缩比（filtered/original）| **0.01 - 0.05**（500MB 日志 → 5-25MB）| > 0.1 → 过滤规则太松；< 0.005 → 过滤太狠（可能丢信号）|

**`bash_filter_compression_ratio` 是日志类 Agent 健康度的命门**。

> 落地时建议把这三个指标做成实时 dashboard（可以借助 Datadog / Grafana 搭建）。
>
> **没有 trace 的 fusion 是黑盒**，某天 Agent 突然变贵、变慢、出错了，你都不知道是哪一层流水线在出问题。

---

## 八、决策卡：图保留还是转文本

> 多模态融合里，最常见的坑是：**一看到图片就丢给视觉模型，一看到 PDF 就整份塞进去**。
>
> 真正动手做工程时，应该先分析**这份材料的价值，藏在布局里，还是藏在文字和结构里**？
>
> - **如果价值在布局、箭头、位置关系里，就保留图**
> - **如果价值在文字、数字、表格结构里，就转成 Markdown、Mermaid、CSV 或 JSON**

### 8.1 决策卡 1：架构图 / 流程图

| 维度 | 详情 |
|------|------|
| **推荐** | 能转 Mermaid 就转 Mermaid |
| **为什么** | 8 个节点架构图：Mermaid 几十 token；draw.io XML 上千 token；PNG 不便检索、修改、diff |
| **例外** | UI 布局、视觉标注、复杂空间位置关系很关键时，保留为图 |

> **服务调用、组件依赖、审批流程、数据流向**，用 Mermaid 表达，模型通常读得清楚。

### 8.2 决策卡 2：表格 / 电子表格

| 维度 | 详情 |
|------|------|
| **推荐** | 默认转 Markdown |
| **为什么** | 核心价值在行、列、字段和数字，不在截图本身 |
| **判断标准** | 如果 Markdown 能还原 95% 信息，就转 Markdown |
| **保留为图的场景** | 跨页表、多层表头、合并单元格、斜线表头、嵌套分组 |

### 8.3 决策卡 3：图表 / 热力图

| 维度 | 详情 |
|------|------|
| **推荐** | 保留图，但**数字要落到数据** |
| **为什么** | 柱状图、折线图、散点图、热力图 —— 很容易让人高估视觉模型的能力 |
| **风险** | 坐标轴密、图例多、双 Y 轴、颜色映射复杂时，**答案可能看起来很顺，数字却错了** |
| **做法** | 分两步：① 图像保留用来定位和理解趋势；② 数字抽出转 CSV/JSON 再计算 |

> **让视觉模型帮你看图，结构化数据负责算数**。
>
> 例：市场规模趋势图保留为图，让 Agent 知道这张图讲什么；但如果要回答"2024 到 2026 CAGR 是多少"，就应该让 Agent **基于结构化数据算**。

### 8.4 决策卡 4：密集文字截图

| 维度 | 详情 |
|------|------|
| **推荐** | 有时**直接当图更划算** |
| **为什么** | 一整页密密麻麻的代码 / 报告截图 / 日志截图 / 表格截图 → OCR 成文本可能很长，还会带来格式噪声 |
| **做法** | 把图片压到合适尺寸，直接交给 vision |
| **适用场景** | 长代码截图、整页报告截图、密集 UI 截图、带大量文字的监控页面 |
| **例外** | 后续要搜索、diff、计算、引用字段 → 还是要转成文本或结构化数据 |

---

## 九、收束：感知模块到此结束

### 9.1 多模态融合的本质

> **多模态融合的核心是数据形态设计**。
>
> **多模态融合是数据形态工程**，也就是让每种数据找到最适合 Agent 消化的形态。

### 9.2 模型供应商 vs 工程师的职责分工

> - **模型供应商**负责让模型能看图、读 PDF、听音频
> - **工程师**负责判断这张图、这页 PDF、这段日志、这段音频该用什么形态进入 Agent

**工程师决策清单**：

| 输入 | 处理 |
|------|------|
| 架构图 | 转 Mermaid |
| 普通表格 | 转 Markdown |
| 图表 | 先抽成 CSV/JSON 再算数 |
| 长日志 | 走预过滤 + Sub-Agent |
| 音频 | 先 STT |
| PDF | 拆成 TOC、关键页、表格、关键图 |

### 9.3 三类事故：完整塞进去的代价

> **好的 fusion 让 Agent 看到恰当的少**。
>
> 完整塞进去，看起来省事，实际会带来三类事故。

| 事故 | 含义 | 应对 |
|------|------|------|
| **① 看错图** | 研报里 5800 亿读成 5800 万 | 图表帮助定位趋势，**精确数字要落到结构化数据**；图像抽取的数字也要和正文、表格的同名数字做交叉校验 |
| **② 图片账单爆炸** | Agent loop 每跑一步可能重新打包同一张图。Demo 看不出问题，生产跑几十步图片成本按轮次放大 | 使用**缩略图（thumbnail）+ 提示缓存机制** |
| **③ Sub-Agent 死循环和关键发现丢失** | 多个 Agent 互相调用时，每次都传整段上下文又没预算上限 → 很容易烧钱 | 关键结论放进**状态存储**，上下文里只传一个指针。每个 Sub-Agent 启动前必须声明 **token 预算 / 时间预算 / 递归预算（recursion budget）** |

### 9.4 四个模式的顺序

> 感知模块四讲，其实是在切**同一个问题**：当下这个 session 里，Agent 怎么看清楚世界。

| 顺序 | 模式 | 职责 |
|------|------|------|
| 1 | **多模态融合**（本讲）| 先决定数据形态 |
| 2 | **[[lession7-上下文分诊]]** | 决定哪些信息靠近模型 |
| 3 | **[[lession8-语义压缩]]** | 长会话里的工作记忆 |
| 4 | **[[lession9-渐进发现]]** | 未知空间里的探索 |

> **如果形态一开始就错了，后面如何分诊、压缩、探索，都是在错误材料上继续消耗**。

### 9.5 收尾

> 到这里，感知模块可以收住了。
>
> 它管的是**当前会话内 Agent 看见什么**。会话结束后，这些上下文都会被回收。
>
> **生产 Agent 不能每次从零开始，所以下一模块进入记忆：怎么把这次会话学到的东西，跨会话留下来**。

---

## 十、思考题

### 题 1：Token 分布审计

> 算一下你 Agent 当前处理输入的 token 分布，**按形态分别做个统计**（text / image / table / log / pdf / audio）。
>
> 有没有什么内容**占了你 60% 以上的 token 但只贡献 20% 信息量**？

### 题 2：图片 vs 转文本原则

> 设计一些你系统中**图片 vs 转文本的原则**？
>
> 是"看起来复杂的就保留"，还是有明确规则？**写出 5 条具体规则**（比如"普通条形图转表格"）。

### 题 3：图表问答实验

> 拿一张你的 Agent 真实处理过的图表（柱状图、折线图、热力图、市场规模图），**设计 10 个问题**：
>
> | 类型 | 数量 |
> |------|------|
> | 具体数值类 | 3 道 |
> | 趋势方向类 | 2 道 |
> | 类别比较类 | 2 道 |
> | 坐标范围类 | 2 道 |
> | 图例理解类 | 1 道 |
>
> 让模型直接看图回答，再把答案和原始数据对比。记录：
>
> 1. **错了几道？**
> 2. 错在**读数、坐标轴、图例**，还是**单位**？
> 3. 哪些问题必须转成 CSV/JSON 后再算？
> 4. 哪些问题可以直接让 vision 模型回答？

### 题 4：业务 Fusion 决策卡

> 给你的 PDF / 研报 Agent **设计一张融合（Fusion）决策卡**。
>
> 找一份你业务里的典型 PDF（合同、研报、手册、审计报告、病历、招标文件），按下面格式拆解：

| 拆解项 | 问题 |
|--------|------|
| 文本主体 | 怎么抽？ |
| 目录 / 章节 | 是否保留？ |
| 关键页 | 怎么识别？ |
| 表格 | 转 markdown 还是 CSV？ |
| 图表 | 哪些保留为图？ |
| 装饰图 | 哪些直接丢？ |
| 原文 | 是否保留 P3 句柄？ |

**输出一张业务决策卡**：

| 输入类型 | 推荐处理方式 | 保留理由 | token 估算 |
|----------|--------------|----------|------------|
| 合同条款 | 转 markdown | 需要引用 | ... |
| 财务表格 | 转 CSV | 需要计算 | ... |
| 架构图 | 转 Mermaid / 留图 | 看空间关系 | ... |
| 封面 / logo | 丢弃 | 无业务信号 | ... |

---

## 十一、精选留言

### 留言 1：PatrickL — 多模态推荐处理方式

> **多模态数据推荐处理方式**：
>
> - 架构图 / 流程图 → Mermaid
> - 表格 → Markdown
> - 图表 → 图片 + CSV/JSON
> - 密集文字截图 → 保留图片
> - 长日志 → 预过滤 + SubAgent 提炼结构化摘要
> - 音频 → STT
> - PDF → 拆成 TOC、关键页、表格、关键图表

> **作者回复**：点赞

### 留言 2：Geek_0dc445 — 通用 PDF 解析的三步法

> 黄老师我想咨询一下，**一个通用智能体在进行 PDF 识别的时候是没法知道下面的信息的**，所以如果通用智能体做通用 PDF 解析的时候：
>
> - 下面的分类是交给 LLM 判断？
> - 还是先识别文档类型再交给模型判断？
> - 还是说把要抽取的 PDF 主题作为一个提示词发给 LLM？
>
> 具体到文中示例："PDF 主体：抽取 TOC、章节摘要和关键页" / "关键图表：识别市场规模、市占率、增长趋势、估值、渗透率等图表"。

> **作者回复**：生产里通常分三步：
>
> 1. **结构识别** —— 用 PDF 解析器或版面模型找出标题、目录、正文块、表格、图片和图注。这一步主要回答"它是什么形态"
> 2. **文档类型路由** —— 研报、合同、产品手册分别加载不同的抽取规则。但只知道"这是研报"还不够
> 3. **结合任务做语义判断** —— 用户要分析市场规模，模型就结合图注、邻近正文和章节标题，找出市场规模、市占率、增长趋势等候选图表
>
> 例：解析器先发现第 23 页有一张折线图，图注是"中国云计算市场规模及增速"。当前任务又要求核查市场规模，这张图才会被标为关键图。另一张反复出现在页眉相同位置的公司 Logo，可以通过位置、重复次数等规则直接判为装饰图，**无须调用 LLM**。
>
> 推荐的流程：**版面解析 → 文档类型路由 → 结合任务目标做语义判断 → 按图、文、表分别处理**。
>
> 其中：
>
> - **结构明确的部分尽量交给程序**
> - 语义性的"这张图对当前问题是否重要"再交给模型
> - 模型判断不确定时，**先保留页码、缩略图和原文句柄，别急着删除**
> - 表格也不必一律转 Markdown：用于阅读可以转 Markdown，需要计算时更适合 CSV 或 JSON，复杂的跨页表还应保留原图核对
>
> **工程上自己做其实挺复杂的，ChatGPT/Codex 都不一定能全部最对，需要核对**。

### 留言 3：Geek_75ba88 — 是不是过度复杂？

> 我怎么觉得这个处理太复杂了，像目前项目里面我思考的处理方式就是**直接 PDF 整个丢给 MinerU 模型，转成 Markdown 格式 + 相关图片 URL**，然后就是给出本次分析目标和输出结果，以及分析思路，丢给 Agent 自己利用一些原子工具不停迭代直到输出结果。
>
> 按照老师说的前期这么多复杂的分类和处理流程是必须的吗？我觉得 Agent 迭代过程中自己就能理解，最近正打算做一个合同要素提取的能力，看了老师的文章后对比自己的思路，感觉老师的是不是稍微有点复杂化了？

> **作者回复**：是的。**我这个套路绝对不是必须的**。
>
> 具体情况具体分析，你的流程其实清晰而准确地解决了你要完成的目标，这就是**奥卡姆剃刀原理**。我认为是正确的流程选择。
>
> 我这整个专栏都是尽可能地往复杂了写，是有原因的。因为目前整个世界范围内还没有出现这么成体系化的 Agent 设计思路，因此我希望多给出一些设计的思维和方向。
>
> 你提出了一个很重要的要点，我应该强调：
>
> 1. **不是所有 Agent 设计都需要所有模式**
> 2. **不是所有模式都需要遵循我给出的设计**
>
> 我给出的很多设计重在启发，告诉大家有这个东西可以选用，**不代表大家一定必须用到哈**。

### 留言 4：Geek4329 — PDF 图片识别

> 黄老师，工程实践上，**一个 PDF 里的图片一般怎么识别出来呢**，以及识别出来怎么判定是不是关键图片呢？

> **作者回复**：现在最好的做法当然是**调用多模态模型来识别图片**了。然后转换成文字之后，通过当前场景、目标来确定这张图片是否是关键图片。

### 留言 5：Kin — PDF 关键页判断

> 咖哥好，看完这篇文章收益颇多，但有个问题想问问，**GitHub 的代码中没有包含 `pdf_extract` 的实现**，我问了 ClaudeCode 按预期返回格式做实现方案，大概内容如下：
>
> 如果你想自己实现一个满足格式的 `pdf_extract`，大致的思路：
>
> - **目录（TOC）** —— 用 PyMuPDF（fitz）或 pypdf 提取页面/章节结构，或者用 marker-based 解析（标题字号突变识别章节）
> - **关键页** —— 需要一个策略判断哪些页是"关键"的：目录页、摘要页、图表页、结论页。简单做法是"前 3 页 + 最后 2 页 + 含图页面"
> - **返回格式** —— 必须返回 `{"toc": str, "key_pages": [{"page": int, "text": str}, ...]}`
>
> **想问问这个关键页判断是怎么得到的呢？是否需要请求 LLM 呢？如果请求 LLM，需要给什么信息**？
>
> 我能想到的是根据每页内容的简单摘要（当前页章节信息，当前页摘要，当前页包含哪些图表、图片）去做判断，但是解析阶段这些信息的获取是需要时间的，而且可能也会浪费相当多的 token，**相当于过了一遍 PDF**。想问问咖哥有什么好的方案吗？

> **作者回复**：确实要"过一遍 PDF"，但**过的是 CPU，并不是 LLM**，理解只发生在筛出来的那几页。具体来说：
>
> 1. **第一档：结构信号，零 LLM**。`fitz` 可以给出确定性结构
> 2. **第二档：要语义排序，别给每页做 LLM 摘要**，给每页拼个廉价指纹（页码 + 标题 + 首行 + 有无图表 + 前 100 字，约 30-50 token/页），200 页整份索引也不大
> 3. **第三档：关键页依赖 query，就上检索** —— 指纹索引 + query 做 embedding 检索只对选中页取全文 —— **这是渐进发现**

### 留言 6：jl886 — 数据工程视角

> 这部分其实就是在做数据收集和清理，工作量也是在不断升高，有点像搞机器学习、深度学习，**做数据清理的工作量占比很高**。

> **作者回复**：对，数据工程工作

### 留言 7：街角·陌路△ — 多场景视角

> 看完整个文章，给我的感觉就是，**与其说是多模态，不如说是多场景**。
>
> 模态只是一个载体，一段文字可以是直接用文字发出，也可以是截图，也可以是文档，甚至是一个录屏。**但重点是这段文字到底具体是什么业务或者是表达什么内容**。
>
> 如果这样看的话，是不是在多模态之前应该有一段路由来识别不同的内容到底属于什么？当然可能成本有点高。但我觉得如果用谷歌的 Gemma 4 这种多模态的模型加上一定的工程化手段，我觉得应该可以做到。**因为这一个阶段的识别不需要具体内容是什么，只需要给这个结构化的返回结果就好**。应该不需要什么特别强大的模型，本地模型应该就可以做到。
>
> **个人判断，希望老师给一些参考**。
>
> 还有个核心就是不管什么输入，**一定要可溯源、可监控**。这也体现了"垃圾输入垃圾输出"这句话的价值。

> **作者回复**：是的，关键是整理出结构，这个阶段模型不必强大。**但是工作流程，结构要清晰**。人都搞不懂，Agent 怎么懂。别一股脑地丢给后续阶段。

---

## 十二、下一讲预告：记忆模块导论

> 下一讲咱们进入 **03 模块 — 记忆：沉淀之美**。
>
> 前面四讲解决的全是**当下会话内的感知**，会话一结束就清零啦。
>
> 生产级别 Agent 不能每次从零开始：
>
> - 用户三天前问过的问题
> - Agent 上次踩过的坑
> - 本租户的偏好设置
>
> 这些**跨会话的状态应该如何保留并积累下来**，这些都属于记忆的范畴。

→ [[lession11-记忆模块导论]]

---

## 附：本讲核心要点速查

| 关键词 | 含义 |
|--------|------|
| **Multi-Modal Fusion** | 先判断每种数据最适合的形态，再带着关联合并到推理层 |
| **双轴定位** | 感知 × 并行（多种异构数据同时进入）|
| **黄金原则** | 空间关系是信号 → 保留为图；否则转 markdown / JSON / 结构化文本 |
| **Vision API token 公式** | `tokens = width × height / 750`（1024×1024 ≈ 1400 token）|
| **PDF token 估算** | 80 页研报 ≈ 90K（暴力）/ 16K（融合后）|
| **Hermes Lazy Audio Imports** | 永远不在 module 顶层 import 音频库 —— SSH/Docker/WSL 没 PortAudio |
| **七种 ModalityType** | TEXT / IMAGE / TABLE / LOG / PDF / AUDIO / SQL_RESULT |
| **三事件对象** | `ModalityInput(hint, keep_as_image)` / `FusionEvent(method)` / `MultiModalFuser` |
| **`bash_filter → log_subagent → structured`** | 长日志三步流水线 |
| **`TOC + key_pages + business_hint`** | PDF 不默认整份塞 |
| **三大业务对象** | `ResearchReport` / `CriticalChartSpec` / `_build_inputs` |
| **数字核查 prompt 铁律** | 每个数字必须挂 page 或 chart 引用，找不到的 confidence 标 low |
| **可观测三指标** | `token_distribution_by_modality` / `fusion_processing_p99_ms (<5s)` / `bash_filter_compression_ratio (0.01-0.05)` |
| **决策卡 1：架构图** | 转 Mermaid（除非 UI 空间关系关键）|
| **决策卡 2：表格** | 默认转 Markdown（Markdown 能还原 95% 就转）|
| **决策卡 3：图表** | 图像保留 + 数字抽 CSV/JSON |
| **决策卡 4：密集文字截图** | 直接当图更划算 |
| **三类事故** | 看错图 / 图片账单爆炸 / Sub-Agent 死循环 |
| **三 Sub-Agent 预算** | token / time / recursion budget |
| **感知四模式顺序** | 多模态融合 → 上下文分诊 → 语义压缩 → 渐进发现 |
| **数据形态工程** | 让每种数据找到最适合 Agent 消化的形态 |