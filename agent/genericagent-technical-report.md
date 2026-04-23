# GenericAgent 技术分析报告

日期：2026-04-22

更新：2026-04-23，补充第一次 / 第二次运行示例，以及短期记忆与长期记忆的区别。

## 摘要

GenericAgent 是一个极简、自我进化的本地自主 Agent 框架。它的核心目标不是再做一个聊天机器人，而是让大语言模型获得对本地计算机的真实执行能力：读写文件、运行代码、控制浏览器、调用系统工具、接入聊天平台、使用 OCR / Vision / ADB 等外部能力，并把成功执行经验沉淀为可复用的记忆、SOP 和工具脚本。

从架构上看，GenericAgent 的核心很薄：一个约百行的 Agent Loop、一组原子工具、一个多模型适配层、一个真实浏览器控制桥，以及一套 L1-L4 分层记忆系统。它的设计哲学是“不预装大量技能，而是在真实任务中长出技能”。因此，它更像是一个个人电脑上的 Agent 执行内核，而不是传统的插件市场或固定工作流系统。

## 1. 项目定位

GenericAgent 解决的是 LLM 从“知道怎么说”到“真的能做”的问题。

普通 LLM 的能力边界主要在文本生成。即使模型能够描述如何完成任务，也缺少稳定的物理执行通道。GenericAgent 通过少量高权限原子工具，把模型连接到真实操作系统、真实浏览器和本地文件系统，使它可以一步步探索、执行、观察结果并继续修正。

它适合的任务包括：

| 任务类型 | 示例 |
|---|---|
| 本地自动化 | 安装依赖、生成脚本、处理文件、配置环境 |
| 浏览器自动化 | 使用保留登录态的真实浏览器访问网页、执行 JS、提取页面内容 |
| 个人工作流 | 定时任务、消息平台接入、网页监控、资料整理 |
| 跨应用操作 | 通过 OCR、窗口控制、ADB 等方式操作桌面或移动端应用 |
| 能力沉淀 | 把成功经验写成 SOP、工具脚本或长期记忆，供下次复用 |

所以，GenericAgent 的本质是：

```text
LLM + 本地执行工具 + 浏览器桥接 + 分层记忆 = 可持续成长的个人 Agent
```

## 2. 总体架构

GenericAgent 可以分成六层：

```text
用户入口层
  -> 任务调度层
  -> LLM 适配层
  -> Agent Loop 层
  -> 工具执行层
  -> 记忆与自进化层
```

对应到代码结构：

| 层级 | 代表文件 | 作用 |
|---|---|---|
| 用户入口层 | `launch.pyw`, `frontends/*` | CLI、Streamlit、Qt、微信、QQ、飞书、企业微信、钉钉等入口 |
| 任务调度层 | `agentmain.py` | 维护任务队列、模型列表、会话历史、任务运行状态 |
| LLM 适配层 | `llmcore.py` | 适配 Claude、OpenAI 兼容接口、Native tool calling、多模型 fallback |
| Agent Loop 层 | `agent_loop.py` | 控制模型推理、工具调用、结果回传、循环终止 |
| 工具执行层 | `ga.py`, `TMWebDriver.py`, `simphtml.py` | 文件、代码、浏览器、记忆、用户确认等工具实现 |
| 记忆层 | `memory/*`, `assets/sys_prompt.txt` | L1-L4 记忆、SOP、辅助脚本、系统提示 |

一次任务的主链路如下：

```text
用户输入
-> 前端 put_task()
-> GeneraticAgent 从 task_queue 取任务
-> 组装 system prompt + global memory
-> agent_runner_loop 调用 LLM
-> LLM 输出 tool call
-> GenericAgentHandler 执行工具
-> 工具结果回传给 LLM
-> 循环直到任务完成或需要用户确认
-> 关键经验写入工作记忆或长期记忆
```

## 3. Agent Loop 的实现

`agent_loop.py` 是 GenericAgent 的核心循环。它没有复杂的图调度、Planner/Executor 多角色拆分，也没有庞大的状态机，而是用一个简单循环完成“思考-行动-观察”的闭环。

核心逻辑是：

1. 准备消息：`system` 放系统提示和记忆，`user` 放当前任务或下一轮提示。
2. 调用模型：`client.chat(messages, tools_schema)`。
3. 解析工具调用：如果模型返回 tool calls，就取出工具名和 JSON 参数。
4. 派发工具：调用 `handler.dispatch(tool_name, args, response)`。
5. 返回观察结果：把工具结果包装成 `tool_results`，送回下一轮。
6. 判断终止：没有工具调用、工具要求退出、达到最大轮数，都会结束循环。

这里最重要的抽象是 `StepOutcome`：

```text
StepOutcome = {
  data: 工具返回数据,
  next_prompt: 下一轮给模型的提示,
  should_exit: 是否退出任务
}
```

这让工具不仅能返回数据，还能影响下一轮模型上下文。例如文件读取工具会把文件内容放进 `data`，同时生成包含工作记忆的 `next_prompt`；`ask_user` 工具则会设置 `should_exit`，把控制权交还用户。

这种设计的好处是足够小、足够直接。模型每轮只需要回答两个问题：当前知道什么，下一步调用什么工具。

## 4. 多模型适配层

`llmcore.py` 负责把不同模型供应商统一成 GenericAgent 可用的接口。

它支持两类工具调用方式：

| 模式 | 代表类 | 特点 |
|---|---|---|
| Native tool calling | `NativeToolClient`, `NativeClaudeSession`, `NativeOAISession` | 使用模型 API 原生工具字段，适合 Claude / OpenAI 等已训练过工具调用的模型 |
| 文本协议工具调用 | `ToolClient`, `ClaudeSession`, `LLMSession` | 把工具 schema 写进 prompt，让模型输出 `<tool_use>{...}</tool_use>` |

`ToolClient` 会向模型注入一套交互协议：

```text
1. 在 <thinking> 中分析
2. 在 <summary> 中输出一句物理快照
3. 如需行动，输出 <tool_use> JSON 块
```

`NativeToolClient` 则直接把工具 schema 传给模型 API，并负责把 tool result 与 pending tool id 对齐。

此外，`MixinSession` 提供多模型故障转移能力。用户可以在 `mykey.py` 中配置多个后端，主模型失败时自动切到备用模型，并在一段时间后尝试回到主模型。这对于个人长期运行的 Agent 很重要，因为第三方中转、模型 API 和网络连接都可能不稳定。

## 5. 原子工具系统

GenericAgent 的工具设计很克制。它没有一开始就内置大量业务插件，而是提供少量通用、高杠杆的原子工具。

核心工具包括：

| 工具 | 作用 |
|---|---|
| `code_run` | 执行 Python、PowerShell 或 shell，用于安装依赖、运行脚本、探测环境 |
| `file_read` | 读取文件，支持行号、关键词定位、截断保护 |
| `file_write` | 创建、覆盖、追加文件，适合大块内容写入 |
| `file_patch` | 精确替换唯一文本块，降低误改风险 |
| `web_scan` | 获取浏览器标签页和简化后的网页 HTML / 文本 |
| `web_execute_js` | 在真实浏览器页面执行 JavaScript |
| `ask_user` | 在需要确认、授权、选择时中断并询问用户 |
| `update_working_checkpoint` | 更新短期工作记忆 |
| `start_long_term_update` | 启动长期记忆沉淀流程 |

工具实现集中在 `ga.py` 的 `GenericAgentHandler` 中。派发方式很简单：工具名 `file_read` 对应方法 `do_file_read`，工具名 `web_execute_js` 对应 `do_web_execute_js`。

这种设计有两个明显收益：

1. 工具少，模型不容易迷失在复杂 API 中。
2. 工具通用，Agent 可以通过 `code_run` 动态创建更高层能力。

例如第一次做 OCR 任务时，Agent 可以用 `code_run` 安装 `rapidocr`，写一个 `ocr_utils.py`，测试通过后把它沉淀到 `memory/`。下次再遇到 OCR 场景，就不需要重新推理和安装。

## 6. 真实浏览器控制机制

浏览器控制是 GenericAgent 的关键技术点之一。

很多 Agent 框架使用无头浏览器或沙箱浏览器，优点是隔离性强，缺点是缺少用户真实登录态。GenericAgent 选择控制真实浏览器：用户已经登录的网站、Cookie、已打开页面都能直接复用。

这套能力由两部分组成：

| 模块 | 作用 |
|---|---|
| `TMWebDriver.py` | 本地 WebSocket / HTTP 服务，管理浏览器 tab session，发送 JS 执行请求 |
| `assets/tmwd_cdp_bridge/*` | Chrome MV3 扩展，接收本地指令，调用 tabs、debugger、scripting、cookies 等能力 |

调用链路如下：

```text
LLM 调用 web_execute_js
-> GenericAgentHandler.do_web_execute_js
-> TMWebDriver.execute_js
-> WebSocket / HTTP 发给 Chrome 扩展
-> 扩展在目标 tab 执行 JS 或 CDP 命令
-> 返回执行结果
-> 结果进入下一轮 LLM 上下文
```

`web_scan` 则依赖 `simphtml.py` 对页面 DOM 做压缩和清洗。它会过滤脚本、样式、不可见元素、被覆盖区域、无意义容器等，把真实网页变成更适合 LLM 阅读的简化 HTML 或纯文本。

这解决了两个问题：

- LLM 不需要看完整网页源码，节省 token。
- Agent 可以基于真实页面状态做下一步动作，而不是猜测。

## 7. 分层记忆系统

GenericAgent 的自我进化建立在分层记忆上。它把不同粒度的信息放在不同层级中：

| 层级 | 文件 / 目录 | 职责 |
|---|---|---|
| L0 | 系统提示与记忆管理 SOP | 基础行为规则和记忆写入原则 |
| L1 | `memory/global_mem_insight.txt` | 极简索引层，只保存关键词和入口 |
| L2 | `memory/global_mem.txt` | 稳定环境事实，如路径、配置、长期参数 |
| L3 | `memory/*.md`, `memory/*.py` | 任务 SOP、可复用脚本、具体避坑经验 |
| L4 | `memory/L4_raw_sessions/` | 历史会话归档，用于长周期回溯 |

最重要的原则是：

```text
No Execution, No Memory.
```

也就是说，只有被工具调用验证成功的信息才能进入长期记忆。模型的猜测、计划、通用常识、临时状态都不应该写入。

L1 的作用是“让 Agent 知道应该去哪里找”。它不能写大段教程，只保留关键词和指针。L2 存储稳定事实。L3 存储具体 SOP 和脚本。L4 则由会话日志压缩器定期整理历史任务记录。

这种分层设计让 GenericAgent 不需要每次都把所有知识塞进上下文。它先加载很小的索引，命中关键词后再读取具体 SOP。

### 7.1 短期记忆与长期记忆

GenericAgent 的记忆可以先按用途理解成两类：

```text
短期记忆：服务当前任务，回答“现在做到哪了”
长期记忆：服务未来任务，回答“以后再做类似任务该怎么做”
```

短期记忆主要存在于 `GenericAgentHandler.working` 中，由 `update_working_checkpoint` 更新，并通过 `_get_anchor_prompt()` 在后续轮次继续注入模型上下文。它保存的是当前任务的临时重点，例如用户需求、关键约束、当前进度、相关 SOP、已经验证的中间状态和下一步计划。

长期记忆则写入 `memory/` 下的 L1-L4 文件体系。它保存跨会话仍然有效的信息，例如稳定环境事实、可复用 SOP、工具脚本、踩坑经验和历史会话归档。长期记忆必须遵循 “No Execution, No Memory”，也就是只保存行动验证成功的信息。

两者的区别如下：

| 维度 | 短期记忆 | 长期记忆 |
|---|---|---|
| 生命周期 | 当前任务或当前会话 | 跨会话长期保留 |
| 主要目的 | 防止长任务中丢上下文 | 让后续同类任务更快、更稳 |
| 典型内容 | 当前需求、进度、约束、下一步 | 稳定事实、SOP、脚本、避坑经验 |
| 存储位置 | `GenericAgentHandler.working` | `memory/global_mem_insight.txt`, `memory/global_mem.txt`, `memory/*.md`, `memory/*.py`, L4 归档 |
| 更新工具 | `update_working_checkpoint` | `start_long_term_update` 后按 SOP 写入文件 |
| 是否适合保存临时状态 | 适合 | 不适合 |

举例来说，如果用户要求“抓取今天 GitHub Trending 的 TypeScript 项目并保存成 Markdown”，短期记忆可以保存“当前语言是 TypeScript、输出 Markdown、已经打开 Trending 页面、下一步执行提取 JS”。长期记忆则应该保存“GitHub Trending 的 URL 模板、项目列表 selector、字段提取方式、写入后验证流程”。

## 8. 自我进化机制

GenericAgent 的自我进化不是训练模型参数，而是“执行经验的本地结晶”。

完整流程如下：

```text
遇到新任务
-> 读取 L1/L2/L3，寻找已有经验
-> 如果没有，使用 code_run / web / file 等工具探索
-> 安装依赖、编写脚本、调试、验证
-> 成功后提炼关键步骤和坑点
-> 写入 L2 事实或 L3 SOP / 脚本
-> 同步 L1 索引
-> 下次类似任务直接召回复用
```

例如：

| 用户需求 | 第一次执行 | 后续复用 |
|---|---|---|
| 配置网页自动化 | 安装扩展、验证 tab 连接、测试 `web_scan` / `web_execute_js` | 直接按 `web_setup_sop.md` 执行 |
| 使用 OCR | 安装 OCR 依赖、写截图识别脚本、验证 bbox | 直接调用 `ocr_utils.py` |
| 控制 Android | 配置 ADB、dump UI、解析节点、封装点击 | 直接调用 `adb_ui.py` |
| 设置定时任务 | 探索 scheduler JSON 格式、验证触发和报告写入 | 直接按 `scheduled_task_sop.md` 新增任务 |

这里有两个关键工具：

- `update_working_checkpoint`：任务进行中保存短期重点，避免长任务丢上下文。
- `start_long_term_update`：任务完成后启动长期记忆沉淀，按 SOP 最小化修改记忆文件。

短期记忆每轮都会通过 `_get_anchor_prompt()` 注入模型，包括最近摘要、当前轮次、关键约束和相关 SOP。长期记忆则只保存事实验证成功、跨会话仍有价值的信息。

### 8.1 第一次运行与第二次运行示例

以“抓取 GitHub Trending 今日热门项目并保存成 Markdown”为例，可以更直观看到 GenericAgent 的自我进化过程。

第一次用户提出：

```text
帮我打开 GitHub Trending，抓取今天 Python 热门项目，整理成 Markdown 文件。
```

如果本地没有相关经验，GenericAgent 会先从探索开始：

1. 读取 L1/L2/L3，确认没有 GitHub Trending 抓取 SOP。
2. 判断任务需要浏览器访问、页面内容提取和文件写入。
3. 调用 `web_scan` 检查浏览器连接；如果 Web 工具未配置，则读取 `web_setup_sop.md` 并完成扩展安装或连接验证。
4. 使用 `web_execute_js` 打开 `https://github.com/trending/python?since=daily`。
5. 通过 `web_scan` 或页面 JS 分析 DOM，找到项目列表结构，例如 `article.Box-row`。
6. 执行提取脚本，获取仓库名、描述、语言、star 数等字段。
7. 调用 `file_write` 生成 Markdown，再用 `file_read` 验证文件内容。
8. 如果确认流程稳定，就调用 `start_long_term_update`，把成功路径写成 L3 SOP，并在 L1 中加入索引。

第一次运行的重点是“学会怎么做”。它会有更多探测、试错和验证，工具调用也更多。最终产物不只是 Markdown 文件，还包括一条未来可复用的经验。

后续用户再次提出：

```text
帮我抓取今天 GitHub Trending 里的 TypeScript 热门项目，整理成 Markdown。
```

第二次运行会变短：

1. 任务关键词命中 L1 中的 GitHub Trending 索引。
2. Agent 读取对应 L3 SOP。
3. 用 `update_working_checkpoint` 保存当前变量：语言是 TypeScript、输出 Markdown、沿用 GitHub Trending 抓取 SOP。
4. 直接打开 `https://github.com/trending/typescript?since=daily`。
5. 复用已验证的 selector 和提取 JS。
6. 写入 Markdown 并读取验证。
7. 如果页面结构没有变化，通常不再更新长期记忆。

第一次和第二次的差异可以概括为：

| 阶段 | 第一次运行 | 第二次运行 |
|---|---|---|
| 经验来源 | 没有现成 SOP，需要探索 | L1 命中，读取 L3 SOP |
| 工具调用 | 多，包含探测、调试、验证 | 少，主要是执行已验证流程 |
| 失败概率 | 相对高 | 相对低 |
| Token 消耗 | 较高 | 较低 |
| 结果 | 完成任务并沉淀经验 | 复用经验快速完成 |

因此，GenericAgent 的“进化”不是模型参数变化，而是本地经验库发生变化：第一次运行把成功路径结晶成可召回资产，第二次运行直接调用这份资产。

## 9. 前端与运行形态

GenericAgent 支持多种交互入口：

| 入口 | 说明 |
|---|---|
| CLI | `python agentmain.py`，最基础的命令行模式 |
| 桌面启动器 | `python launch.pyw`，启动 Streamlit + pywebview 窗口 |
| Streamlit / Qt | 图形化聊天界面 |
| Telegram / QQ / 飞书 / 企业微信 / 钉钉 | 远程消息入口 |
| 个人微信 Bot | 扫码登录后通过微信消息交互 |
| Reflect 模式 | 定时或条件触发任务，适合自动巡检和后台任务 |

这些前端最终都会把用户消息转成任务，塞进 `GeneraticAgent.task_queue`。因此核心 Agent 逻辑不依赖具体 UI。

Reflect 模式值得单独注意。`agentmain.py --reflect xxx.py` 会定期执行脚本中的 `check()`，如果返回任务文本，就自动投递给 Agent。例如 `reflect/scheduler.py` 会扫描计划任务配置，并定期触发 L4 会话归档。

## 10. 技术收益

GenericAgent 的收益主要来自“极简内核 + 高权限工具 + 记忆沉淀”的组合。

| 收益 | 说明 |
|---|---|
| 部署轻 | 核心依赖少，最小启动只需要 Python、少量包和 API Key |
| 执行面广 | 可以操作文件、终端、浏览器、桌面、移动设备和消息平台 |
| 成本低 | 分层记忆避免每次加载大量上下文 |
| 个性化强 | 长期使用后会形成贴合个人电脑环境的技能树 |
| 可扩展 | 通过 `code_run` 动态安装依赖、写脚本、封装新能力 |
| 真实环境友好 | 控制真实浏览器，保留登录态，适合日常个人工作流 |

这套架构的优势不在于“框架多复杂”，而在于它把 LLM 放进了一个可行动、可观察、可积累经验的闭环里。

## 11. 风险与局限

GenericAgent 的能力边界也很清楚。

第一，它拥有很强的本地执行权限。`code_run` 可以运行任意代码，浏览器扩展也具有较高权限。因此它更适合个人可信环境，不适合无隔离地开放给陌生用户。

第二，记忆质量决定长期表现。如果把未验证信息、临时状态或过长教程写入记忆，后续任务会被污染。项目通过记忆 SOP、最小 patch、L1 索引限制来降低风险，但仍需要使用者保持纪律。

第三，真实浏览器控制虽然强大，但也更依赖本机环境。浏览器版本、扩展安装状态、页面结构变化、登录态失效，都会影响稳定性。

第四，它不是传统企业级编排平台。它没有强隔离沙箱、权限审批系统、审计工作流或多租户控制。它更适合个人电脑上的自主执行和能力积累。

## 12. 与常见 Agent 框架的差异

GenericAgent 和许多 Agent 框架的区别可以概括为：

| 维度 | GenericAgent | 常见插件式 Agent |
|---|---|---|
| 核心思路 | 少量原子工具 + 自我沉淀 | 预置大量工具或插件 |
| 能力来源 | 任务中探索并固化 | 开发者提前实现 |
| 运行环境 | 用户真实电脑和真实浏览器 | 沙箱、云端或无头浏览器 |
| 记忆机制 | L1-L4 分层记忆和 SOP | 通常依赖会话上下文或向量库 |
| 适用场景 | 个人工作流、本地自动化、长期使用 | 标准化任务、服务端工具调用 |

它的重点不是“给模型更多工具”，而是“让模型学会把工具使用经验保存下来”。

## 13. 总结

GenericAgent 是一个小而有野心的自主 Agent 框架。它用很薄的 Agent Loop，把 LLM、多模型适配、真实浏览器控制、本地代码执行、文件系统和分层记忆连接起来。

它最有价值的部分是自我进化机制：每一次成功执行都可能变成未来可复用的 SOP、脚本或记忆索引。随着使用时间增加，GenericAgent 不只是“调用一个模型”，而是在用户本机上逐渐长出一套个人化能力树。

如果把传统 Agent 看成“带工具的聊天模型”，GenericAgent 更接近“可执行、可学习、可积累经验的个人自动化内核”。
