论文链接：[https://arxiv.org/abs/2604.03088](https://arxiv.org/abs/2604.03088)

# SkVM 论文解读

日期：2026-04-18

论文：SkVM: Revisiting Language VM for Skills across Heterogenous LLMs and Harnesses

## 摘要

SkVM 的核心观点是：Agent skill 不应该只被当作一段可以塞进上下文的 Markdown 提示词，而应该被当作一种“自然语言程序”来编译、适配和运行。

现在的 skill 往往包含工具使用说明、操作流程、依赖环境、脚本模板和参考资料。它们看起来像文档，但在 agent 系统里承担的是可执行逻辑的角色。问题在于，同一份 skill 放到不同模型、不同 agent harness、不同本地环境中，实际效果会差很多。SkVM 试图解决的就是这个“skill 可移植性”问题。

它把整个系统类比成传统虚拟机：

```text
skill = 自然语言程序
LLM = 异构处理器
harness = 运行平台
SkVM = skill 的编译器和运行时
```

因此，SkVM 不希望一份原始 skill 到处裸跑，而是希望根据目标模型、目标 harness 和本地环境，先把 skill 编译成更适合当前目标的版本，再在运行时继续监控、修复和优化。

## 背景

Agent skill 可以理解为给大模型 agent 使用的能力包。一个 skill 通常包含：

- 什么时候应该触发这个 skill
- 任务应该按照什么流程做
- 应该调用哪些工具、库或 API
- 有哪些脚本、模板或参考资料可以复用
- 遇到失败时应该如何处理

论文分析了来自 clawhub.ai 和 skills.sh 的大量 skill。作者指出，skill 生态已经不小，其中相当多的 skill 并不是简单知识说明，而是在描述可执行流程。论文将常见 skill 大致分为三类：

| 类型 | 含义 |
|---|---|
| 工具/API/CLI 使用说明 | 告诉 agent 如何调用某个工具或接口 |
| 流程指导 | 规定多步任务应该如何执行 |
| 内容生成 | 指导 agent 生成某类文本、图像或结构化内容 |

这也解释了为什么 skill 的问题不能只当作 prompt engineering 问题。它更像是软件分发问题：同样一段逻辑，在不同运行目标上可能有不同依赖、不同工具接口、不同执行能力。

## 现有 Skill 的问题

论文重点指出三类 mismatch。

### 1. 模型不匹配

同一个 skill 对强模型可能很清楚，对弱模型可能就会误解。

例如一个 PPTX skill 推荐使用 PptxGenJS。强模型可能理解这是一个 JavaScript 库，应该写代码调用它；弱模型可能把它误当作命令行工具，尝试直接在 shell 里执行 `pptxgenjs`，结果失败。

也就是说，skill 原文隐含了对模型理解能力、代码能力、工具使用能力的假设。

### 2. Harness 不匹配

Harness 是 agent 的运行框架，负责上下文管理、工具注册、工具调用、文件访问、子任务调度等。

同一个模型在不同 harness 下能力表现可能不同，因为：

- 工具调用格式不同
- 系统提示不同
- 可用工具不同
- 是否支持批量工具调用不同
- 是否支持 sub-agent 不同
- 文件路径和工作目录处理方式不同

因此 SkVM 不是只对模型做画像，而是对 `(model, harness)` 这个组合做画像。

### 3. 环境不匹配

很多 skill 默认某些依赖已经存在，例如 Python 包、CLI 工具、系统服务或浏览器环境。但用户机器不一定具备这些依赖。

如果运行到一半才发现缺依赖，强模型也许能自我修复，但会多消耗 token 和时间；弱模型则可能直接失败。SkVM 试图把这些环境问题前移到编译和准备阶段。

## SkVM 做了什么

SkVM 分成两个大的阶段：

```text
AOT 编译阶段：任务执行前，根据目标模型、harness 和环境编译 skill
JIT 运行阶段：任务执行中，根据真实运行轨迹继续修复和优化 skill
```

### AOT：执行前编译

AOT 是 ahead-of-time compilation，可以理解为“安装或启用 skill 时先编译一次”。

SkVM 在 AOT 阶段主要做三件事：

1. 能力感知编译
2. 环境绑定
3. 并行性提取

### JIT：执行中优化

JIT 是 just-in-time optimization，可以理解为“任务跑起来之后继续观察和优化”。

SkVM 在 JIT 阶段主要做两件事：

1. 根据失败和恢复轨迹自适应重编译 skill
2. 将稳定重复的代码路径固化成可直接执行的函数

## 核心技术

### 1. Primitive Capabilities

SkVM 首先定义一组原子能力，论文称为 primitive capabilities。

这些能力用来描述一个 skill 需要模型和 harness 具备什么能力。论文中举到的能力包括：

| 能力 | 含义 |
|---|---|
| `gen.code.shell` | 生成 shell 命令 |
| `tool.exec` | 调用工具并处理参数、路径和多步执行 |
| `follow.procedure` | 按多步流程执行任务 |
| `reason.arithmetic` | 做算术和结构化推理 |
| `gen.code.python` | 生成 Python 代码 |

每个能力还可以分等级，例如 L1、L2、L3。以 `tool.exec` 为例：

| 等级 | 大致含义 |
|---|---|
| L1 | 执行简单命令 |
| L2 | 处理参数、相对路径和稍复杂的工具调用 |
| L3 | 多步链式工具执行 |

SkVM 一边分析 skill 需要哪些能力，一边测量目标 `(model, harness)` 具备哪些能力，然后对比两者之间的 gap。

### 2. 目标模型和 Harness 的能力画像

SkVM 不是靠模型名字猜能力，而是通过 microbenchmark 测出来。

流程大致如下：

```text
定义 primitive capabilities
-> 为每个能力等级设计 microbenchmark
-> 在具体 model + harness 组合上运行这些测试
-> 得到 target capability profile
-> 之后编译 skill 时复用这个 profile
```

例如当前目标是：

```text
model = Qwen3-30B
harness = BareAgent
```

SkVM 会让这个组合完成一组小任务：

- 是否能执行简单 shell 命令
- 是否能正确处理相对路径
- 是否能按多步流程读写文件
- 是否能生成可运行 Python 代码
- 是否能从工具错误中恢复

如果测试发现它能稳定执行简单命令，但经常在相对路径上出错，那么它的 `tool.exec` 可能只达到 L1，而不是 L2。

因此，所谓“Qwen3-30B 在 BareAgent 下对相对路径不稳”，不是主观判断，而是 profiling 得到的结果。并且这个结果属于具体组合：

```text
Qwen3-30B + BareAgent
```

而不是单独属于 Qwen3-30B。换一个 harness，能力画像可能不同。

### 3. Capability-Aware Transform

当 SkVM 发现 skill 需求和目标能力之间存在 gap，会选择两种策略。

第一种是 compensation，也就是补偿。

如果目标能力只是差一点，SkVM 会把 skill 改写得更明确，让它降低对模型隐式理解能力的要求。例如：

```text
原始 skill：读取 ./input 下的文件
编译后 skill：先解析为绝对路径 /Users/ricky/project/input，再读取该目录
```

这可以避免弱模型或特定 harness 在相对路径上踩坑。

第二种是 substitution，也就是替换。

如果目标模型明显不擅长某条技术路径，SkVM 可以把 skill 换成另一条等价路径。例如：

```text
原路径：让模型生成复杂 pandas 代码
替换路径：改用 SQL 或已有 CLI 工具完成同类数据处理
```

补偿适合小 gap，替换适合大 gap。

### 4. Environment Binding

SkVM 会从 skill 中抽取环境依赖，包括：

- Python 包
- Node 包
- CLI 工具
- 系统服务
- 浏览器或外部程序

然后它会生成幂等的环境检查和准备脚本。例如：

```bash
python -c "import pdfplumber"
which pandoc
```

如果依赖不存在，就提前安装或切换实现路径。

这个设计的关键是：不要让 agent 在任务执行到一半时才发现缺依赖，也不要让弱模型临时猜怎么修环境。

### 5. Concurrency Extraction

很多 skill 是顺序写的，但实际步骤并不一定都存在依赖关系。

例如：

```text
处理 100 个 PDF
-> 每个 PDF 抽取字段
-> 汇总成一个 CSV
```

每个 PDF 的抽取彼此独立，可以并行执行。SkVM 会分析步骤输入输出，构建 DAG，然后抽取并行机会。

论文里主要提到三类并行：

| 类型 | 含义 |
|---|---|
| DLP | 数据级并行，例如多个文件并行处理 |
| ILP | 指令级并行，例如多个无依赖工具调用批量派发 |
| TLP | 任务级并行，例如多个独立子任务交给 sub-agent |

这让 skill 不只是更稳，也可能跑得更快。

### 6. Adaptive Recompilation

JIT 阶段会记录真实执行中的失败、重试和恢复轨迹。

如果 SkVM 发现某类失败反复出现，例如：

```text
模型总是忘记创建 output 目录
模型总是把相对路径解析错
模型总是漏掉 CSV header
模型总是在某个 API 参数格式上失败
```

它会把这些运行日志反馈给编译器，重新生成更适合当前目标的 skill 版本。

这就是自适应重编译。它不是每次失败都盲目改 skill，而是尝试识别系统性失败模式。

### 7. Code Solidification

如果某段代码路径反复出现，而且结构稳定，SkVM 会把它固化成可直接调用的函数。

例如一个 PDF 发票抽取 skill，每次模型都生成类似代码：

```python
def extract_invoice_fields(pdf_paths, output_csv):
    ...
```

当 SkVM 观察到这个模式足够稳定，就可以把它提升为固化函数。之后遇到同类任务时，模型不需要重新生成整段代码，只需要提供参数：

```json
{
  "pdf_paths": ["/abs/a.pdf", "/abs/b.pdf"],
  "output_csv": "/abs/result.csv"
}
```

然后 SkVM 直接调用固化函数。

这带来两个好处：

- 减少 LLM 推理成本
- 提升运行速度和稳定性

如果固化函数匹配不上新任务，SkVM 仍然可以回退到普通 LLM 执行路径。

## 一个实际例子

假设有一个 skill：

```text
从一批 PDF 发票中抽取供应商、日期、金额，并输出 CSV。
```

原始 skill 可能只写：

```text
Use Python and pdfplumber to read all PDFs in ./input.
Extract vendor, date, and amount.
Save the result to ./output/result.csv.
```

### AOT 阶段会做什么

SkVM 会先分析这个 skill 需要：

- Python 代码生成能力
- 文件路径处理能力
- 多步流程执行能力
- PDF 解析库依赖
- CSV 输出能力

然后它对目标 `(model, harness)` 做 capability profiling。假设结果显示：

```text
目标能执行简单命令
但相对路径处理不稳定
复杂 Python 代码生成也不够稳
```

SkVM 可能把 skill 编译成：

```text
Always resolve input and output paths to absolute paths.
Input directory: /Users/ricky/project/input
Output file: /Users/ricky/project/output/result.csv
Use the provided Python script template instead of generating extraction code from scratch.
Before execution, ensure pdfplumber and pandas are installed.
```

同时，它会生成依赖检查：

```bash
python -c "import pdfplumber, pandas"
```

如果要处理 100 个 PDF，它还会识别出每个 PDF 的抽取可以并行。

所以 AOT 做的是：

```text
开跑前改写 skill、补足环境、降低模型误解概率、找出可并行结构
```

### JIT 阶段会做什么

真正执行后，SkVM 会观察运行轨迹。

如果它发现模型连续几次都忘记创建输出目录，就会把这个失败模式写回 skill：

```text
Before writing CSV, always create the parent directory of the output file.
```

如果它发现 PDF 抽取代码每次都长得差不多，就会把这段代码固化成函数。之后再处理同类发票时，直接调用函数，不再让模型重复生成代码。

所以 JIT 做的是：

```text
跑起来之后，根据真实失败和热点路径继续修复、缓存和加速
```

## 效果如何

论文实验覆盖多个模型和多个 harness，并在 SkillsBench、PinchBench 等任务上评估。主要结论包括：

- SkVM 平均提升任务完成率。
- 原始 skill 有时会让模型表现变差，SkVM 编译后的 skill 明显减少这种回归。
- AOT 能显著提升弱模型在复杂 skill 上的可用性。
- JIT 通过自适应重编译继续提升成功率。
- 环境绑定能减少依赖缺失导致的失败和额外 token 消耗。
- 并行提取可以带来端到端加速。
- 代码固化在稳定重复任务上可以显著减少延迟。

论文中比较醒目的结果是：

| 维度 | 结果 |
|---|---|
| 完成率 | SkVM 平均提升约 15.3% |
| 回归任务比例 | 原始 skill 约 15%，SkVM 后约 4.5% |
| 并行加速 | 最高约 3.2 倍 |
| 代码固化加速 | 某些 PDF 抽取任务可达约 19-50 倍 |

这些结果说明，SkVM 的收益不只来自“更好的提示词”，还来自环境准备、流程重写、并行调度和运行时缓存。

## SkVM 本身需要什么模型

SkVM 涉及两类模型角色。

第一类是 target/runtime model，也就是用户真正拿来执行任务的模型。例如论文实验里的 Claude、DeepSeek、Gemini、Qwen、Devstral 等。

第二类是 compiler model，也就是 SkVM 编译 skill 时用来分析和改写 skill 的模型。

两者可以是同一个，也可以不同。更实际的部署方式是：

```text
编译阶段：使用较强模型分析和改写 skill
执行阶段：使用用户实际要跑的模型完成任务
```

这样做的原因是，编译阶段通常是低频、可缓存的，而执行阶段是高频、成本敏感的。用强模型提前把 skill 编译好，再交给便宜或较弱的目标模型运行，正是 SkVM 想带来的工程收益之一。

JIT 阶段也可能继续用到模型，尤其是自适应重编译。但如果某段逻辑已经被 code solidification 固化，后续执行就可以直接调用函数，绕过 LLM。

## 我的理解

SkVM 最有价值的地方，是把 skill 从“上下文里的提示词片段”提升成了“可以编译和运行的软件组件”。

它并不是单纯说：我们写一个更长、更清楚的 prompt。它真正强调的是：

- skill 有目标平台适配问题
- skill 有依赖环境问题
- skill 有运行时失败反馈问题
- skill 有并行和缓存优化空间
- skill 可以被 profile、transform、recompile、solidify

这让 agent skill 更接近传统软件工程里的 package、IR、runtime 和 VM，而不是一次性的自然语言说明。

不过它也有成本和限制：

- 编译 skill 本身要消耗 LLM token
- primitive capability 的定义和 benchmark 需要维护
- 自然语言 skill 的解析仍然有不确定性
- 对一次性任务而言，编译成本未必值得
- 对开放式、强创意型任务，代码固化和流程 DAG 的收益可能有限

所以我会把 SkVM 理解为一种面向“可复用、可分发、重复执行”的 agent skill 基础设施。它最适合那些有稳定流程、明确工具、重复调用频率高、需要跨模型或跨 harness 运行的 skill。

一句话总结：

```text
SkVM 把 skill 从 prompt 变成可编译的自然语言程序，并用 AOT + JIT 的方式解决跨模型、跨 harness、跨环境的执行稳定性问题。
```

## 相关笔记方向

后续可以继续延伸几个问题：

- Skill 是否需要一种标准 IR
- Agent harness 是否会像操作系统一样形成生态分层
- Prompt package 和传统软件 package 的边界在哪里
- Skill marketplace 如何处理依赖、权限、安全和版本兼容
- 小模型能否通过更强的 skill 编译器获得接近大模型的任务执行能力
