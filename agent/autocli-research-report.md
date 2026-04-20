# AutoCLI 调研报告

日期：2026-04-16

## 摘要

AutoCLI 更适合被理解为一个“网站能力适配器系统”，而不是传统意义上的通用爬虫。它把网站 API、浏览器会话、Web 应用操作以及部分本地工具封装成结构化命令，让用户或 AI Agent 可以通过统一的命令行接口获取数据或执行动作。

它的核心抽象是 adapter。一个 adapter 是一份 YAML 配置文件，用来描述一个命令的名称、参数、执行步骤、认证策略和输出字段。AutoCLI 根据这些 adapter 动态生成命令，并通过 Pipeline 执行器完成请求、过滤、字段映射、浏览器自动化、网络拦截或 UI 操作。

在桌面环境中，AutoCLI 可以利用普通 CLI 进程、本地 daemon、Chrome 扩展、浏览器登录态和外部命令工具获得很强的执行能力。但如果要在鸿蒙原生应用层面复刻 AutoCLI，就不能简单照搬桌面架构。普通鸿蒙 App 受到移动系统沙箱限制，不能随意读取其他 App 的登录态、拦截其他 App 的网络请求、控制任意第三方 App 或执行任意系统命令。因此，鸿蒙原生版更适合做成“原生应用能力适配器平台”，重点支持公开 API、用户授权 API、Ability 调用、合作 SDK 和受限 UI 自动化。

## 1. AutoCLI 的功能定位

AutoCLI 的基本目标是把不同网站或工具的能力统一包装成命令：

```text
网站 / Web App / 外部工具 -> adapter -> autocli 命令
```

常见使用方式如下：

```bash
autocli hackernews top --limit 10
autocli bilibili hot --limit 20
autocli twitter search "AI agent"
autocli generate "https://example.com" --goal search --ai
```

它主要提供这些能力：

| 能力 | 说明 |
|---|---|
| 网站数据获取 | 获取热榜、搜索结果、用户信息、评论、动态、收藏等 |
| 登录态复用 | 通过浏览器模式复用已登录状态 |
| 浏览器自动化 | 打开页面、滚动、点击、输入、执行 JS、截图 |
| 网络请求拦截 | 捕获页面真实触发的 API 请求 |
| 媒体和文章下载 | 下载视频、图片、文章，并可本地化资源 |
| 多格式输出 | 支持 table、JSON、YAML、CSV、Markdown |
| AI 生成 adapter | 根据页面/API/HTML 上下文自动生成 YAML 配置 |
| 外部 CLI 透传 | 集成 `gh`、`docker`、`kubectl` 等本地命令工具 |

AutoCLI 和传统爬虫的区别在于：传统爬虫通常从一个种子 URL 出发，递归抓取页面和链接；AutoCLI 则执行一个明确命令，例如 `hot`、`search`、`feed`、`comments` 或 `download`。每个命令背后都有一个 adapter，adapter 描述数据从哪里来、如何处理、如何输出。

因此，AutoCLI 可以用于抓取和采集数据，但它的产品形态更像是“网站能力命令化平台”。

## 2. YAML Pipeline 和 Adapter

Adapter 是 AutoCLI 的核心扩展机制。它是一份 YAML 命令定义，回答以下问题：

- 这个命令属于哪个站点？
- 命令叫什么？
- 接收哪些参数？
- 是否需要浏览器？
- 使用什么访问策略？
- 执行哪些步骤？
- 最终输出哪些字段？

一个简化 adapter 示例：

```yaml
site: hackernews
name: top
description: Hacker News top stories
strategy: public
browser: false

args:
  limit:
    type: int
    default: 20

pipeline:
  - fetch:
      url: https://hacker-news.firebaseio.com/v0/topstories.json
  - limit: "${{ args.limit }}"
  - map:
      rank: "${{ index + 1 }}"
      title: "${{ item.title }}"

columns: [rank, title]
```

可以这样理解：

```text
adapter = 命令说明书
pipeline = 命令内部的执行步骤
```

Pipeline 是一个顺序执行的数据管道。每一步接收上一阶段的数据，处理后传给下一步。

从代码注册情况看，当前 Pipeline 定义了 19 个 step：

| 类别 | 步骤 | 作用 | 是否需要浏览器 |
|---|---|---|---:|
| 数据处理 | `select` | 从 JSON 中选择嵌套路径，如 `data.items[0]` | 否 |
| 数据处理 | `map` | 将数组或对象映射成新的输出结构 | 否 |
| 数据处理 | `filter` | 按表达式过滤数组元素 | 否 |
| 数据处理 | `sort` | 按字段排序，支持升序和降序 | 否 |
| 数据处理 | `limit` | 截断结果数量 | 否 |
| HTTP 请求 | `fetch` | 发送 HTTP 请求，支持 `url`、`method`、`headers`、`body`、`params` | 否 |
| 浏览器操作 | `navigate` | 在浏览器中打开页面，并等待页面稳定 | 是 |
| 浏览器操作 | `click` | 点击页面 CSS selector | 是 |
| 浏览器操作 | `type` | 向指定输入框输入文本 | 是 |
| 浏览器操作 | `wait` | 等待时间、selector 或文本出现 | 是 |
| 浏览器操作 | `press` | 触发键盘事件，如 `Enter` | 是 |
| 浏览器操作 | `evaluate` | 在页面上下文中执行 JavaScript | 是 |
| 浏览器操作 | `snapshot` | 获取页面结构快照，可用于提取 DOM/可见内容 | 是 |
| 浏览器操作 | `screenshot` | 截图，返回 base64 或保存到指定路径 | 是 |
| 浏览器操作 | `scroll` | 自动滚动页面，用于触发懒加载或分页加载 | 是 |
| 网络拦截 | `intercept` | 安装请求拦截器，捕获匹配 URL 的网络请求 | 是 |
| 网络拦截 | `collect` | 收集已拦截请求，并用 JS `parse` 函数处理 | 是 |
| 前端状态 | `tap` | 调用 Pinia/Vuex store action，并捕获对应网络响应 | 是 |
| 下载 | `download` | 下载媒体、文章，或调用 `yt-dlp` 下载视频 | 通常是 |

这些动作可以按执行环境理解：

```text
纯数据处理：select / map / filter / sort / limit
网络请求：fetch
浏览器页面：navigate / click / type / wait / press / evaluate / snapshot / screenshot / scroll
浏览器网络：intercept / collect
前端状态桥接：tap
下载：download
```

其中 `fetch` 支持三种常见模式：

```yaml
- fetch: https://api.example.com/items

- fetch:
    url: https://api.example.com/search
    method: GET
    params:
      q: "${{ args.keyword }}"
    headers:
      accept: application/json

- fetch:
    url: https://api.example.com/items/${{ item.id }}
```

如果上一步数据是数组，并且 `fetch` URL 引用了 `item`，AutoCLI 会进入逐项并发请求模式。

浏览器侧的动作通常依赖 Chrome 扩展、本地 daemon 和页面上下文。例如：

```yaml
pipeline:
  - navigate: https://example.com
  - wait:
      selector: ".item"
  - scroll:
      count: 3
      delay: 300
  - evaluate: |
      Array.from(document.querySelectorAll(".item")).map(el => ({
        title: el.innerText
      }))
```

网络拦截可以拆成“安装拦截”和“收集处理”两段：

```yaml
pipeline:
  - navigate: https://example.com
  - intercept:
      pattern: "/api/feed"
      collect: false
  - click: ".load-more"
  - collect:
      parse: |
        (requests) => requests.flatMap(r => r.response?.data?.items || [])
```

Pipeline 表达式使用类似模板语法：

```yaml
${{ args.limit }}
${{ item.title }}
${{ index + 1 }}
```

主要变量含义：

| 变量 | 含义 |
|---|---|
| `args` | 命令行参数 |
| `data` | 当前 Pipeline 数据 |
| `item` | 当前遍历到的数据项 |
| `index` | 当前数据项下标 |

这种设计让 AutoCLI 的扩展成本很低。新增站点能力时，大多数情况下只需要新增一个 YAML adapter，而不需要修改 Rust 代码。

## 3. 如何生成 Adapter

AutoCLI 有三种主要 adapter 生成方式：规则生成、AI 生成和手写。

### 3.1 规则生成

规则生成使用本地启发式逻辑：

```bash
autocli generate "https://example.com" --goal hot
```

大致流程如下：

```text
打开页面
-> 发现 API endpoint
-> 分析响应字段和认证特征
-> 根据 goal 和 endpoint URL 推断能力
-> 合成 YAML Pipeline
-> 保存 adapter 到本地
```

这里的 `goal` 不是自然语言需求，而是一个能力标签。常见规范 goal 包括：

| 规范 goal | 可识别别名 |
|---|---|
| `search` | `search`、`搜索`、`查找`、`query`、`keyword` |
| `hot` | `hot`、`热门`、`热榜`、`热搜`、`popular`、`top`、`ranking` |
| `trending` | `trending`、`趋势`、`流行`、`discover` |
| `feed` | `feed`、`动态`、`关注`、`时间线`、`timeline`、`following` |
| `me` | `profile`、`me`、`个人信息`、`myinfo`、`账号` |
| `detail` | `detail`、`详情`、`video`、`article`、`view` |
| `comments` | `comments`、`评论`、`回复`、`reply` |
| `history` | `history`、`历史`、`记录` |
| `favorite` | `favorite`、`收藏`、`bookmark`、`collect` |

例如：

```text
热门 -> hot
搜索 -> search
评论 -> comments
收藏 -> favorite
```

`goal` 的作用主要有三点：

| 作用 | 说明 |
|---|---|
| 能力命名 | 如果用户传了 goal，生成出的 adapter 名称可能直接使用这个 goal |
| 候选匹配 | 在多个候选 endpoint 中优先选择与 goal 更接近的能力 |
| 参数推断 | 对少数内置 goal 有特殊逻辑，例如 `search` 会优先选择带搜索参数的 endpoint，并推荐 `keyword` 参数 |

需要注意的是，`goal` 不是现有命令的全集，也不是完整自然语言理解系统。AutoCLI 现有 adapter 里可以有 `download`、`post`、`reply-dm`、`creator-stats`、`quote`、`add-to-cart` 等命令名，但规则生成阶段内置的规范 goal 主要是上表 9 个。

规则版的 goal 匹配机制可以简化理解为：

```text
用户输入 goal
-> trim + lowercase
-> 按别名表顺序做 contains 匹配
-> 命中则返回规范 goal
-> 未命中则不做规范化
```

如果用户没有传 goal，AutoCLI 会尝试从 endpoint URL 推断能力名：

| URL 特征 | 推断能力 |
|---|---|
| 包含 `hot`、`popular`、`ranking`、`trending` | `hot` |
| 包含 `search` | `search` |
| 包含 `feed`、`timeline`、`dynamic` | `feed` |
| 包含 `comment`、`reply` | `comments` |
| 包含 `history` | `history` |
| 包含 `profile`、`userinfo`、`/me` | `me` |
| 包含 `favorite`、`collect`、`bookmark` | `favorite` |
| 其他情况 | 使用 URL 最后一个有意义路径段，或退回到 `data` |

如果用户填写 `latest`、`products`、`jobs`、`add-to-cart` 这类未内置 goal，当前规则版通常不会理解其完整语义，但原始 goal 不会被丢弃。它可能被用作 adapter 名称或候选偏好。

以 `add-to-cart` 为例：

```bash
autocli generate "https://example.com/product/1" --goal add-to-cart
```

当前规则版的行为大致是：

```text
add-to-cart 不在规范 goal 别名表中
-> normalize_goal 返回空
-> 原始 goal 继续传给合成逻辑
-> 生成出的 adapter 名称可能叫 add-to-cart
-> 但规则版未必能自动识别“加入购物车”的接口语义
```

也就是说，它可能生成一个名为 `add-to-cart` 的 adapter，但不保证选中的 endpoint 就是真正的加购物车接口。除非页面探索时刚好捕获到很明显的接口，例如 `/api/cart/add`、`/cart/add`、`/basket/items`，并且字段结构足够清晰。

如果要让规则版稳定支持 `add-to-cart` 这类动作型 goal，需要新增专门规则：

- 将 `add-to-cart`、`add_to_cart`、`cart_add`、`加入购物车`、`加购` 加入 goal 别名。
- endpoint 选择优先匹配 `cart/add`、`add-to-cart`、`basket`、`checkout/cart`。
- 方法优先识别 `POST`。
- 参数优先识别 `productId`、`skuId`、`itemId`、`quantity`。
- 策略优先考虑 `cookie` 或 `header`，因为加购物车通常需要登录态和 CSRF token。
- 动作类能力应默认标记为需要用户确认或至少需要显式授权。

因此，规则生成适合结构简单、接口明显、字段清晰的网站；复杂动作型 goal 更适合 AI 生成或人工手写后再沉淀为规则。

### 3.2 AI 生成

AI 生成命令示例：

```bash
autocli generate "https://example.com" --goal search --ai
```

AI 生成会先采集页面上下文，包括：

- 页面 URL、标题、描述、关键词
- 浏览器 Performance 中出现的 API URL
- 部分 JSON API 响应
- 页面主要 HTML
- 框架信息，如 Vue、Pinia、React、Next.js、Nuxt
- 全局状态，如 `__INITIAL_STATE__`、`__NEXT_DATA__`、`__NUXT__`

这些上下文会发送到 AutoCLI.ai 服务端，由服务端负责构造 prompt 并调用大模型生成 YAML adapter。客户端拿到 YAML 后，会做一些清理和修正，例如去掉 Markdown 代码块、修复 `evaluate` 结构、去重 `columns`，并强制写入检测到的 `site` 和用户指定的 `name`。

AI 生成不要求用户本地准备专用模型。理论上，只要模型足够擅长代码生成、JSON/HTML 理解和 YAML 输出，就可以完成 adapter 生成任务。专用微调模型不是必须，但强模型和好的 prompt 会提升成功率。

AI 生成更适合：

- API 结构复杂
- 字段路径很深
- 页面中有 SSR 或全局状态
- 规则版选错 endpoint
- 需要更智能地理解页面语义

### 3.3 手写 Adapter

手写 adapter 适合这些情况：

- 已经知道目标 API
- 生成结果接近可用但需要调整
- 网站交互比较特殊
- 对稳定性要求高
- 需要精细控制输出字段或错误处理

手写 adapter 也可以沉淀为样例，反过来帮助后续规则生成或 AI 生成。

## 4. public、cookie、header、intercept、ui 策略区别

AutoCLI 的 `strategy` 描述命令访问数据时需要多接近真实浏览器或真实用户行为。

整体上可以按从轻到重理解：

```text
public -> cookie -> header -> intercept -> ui
```

| 策略 | 是否需要浏览器 | 核心方式 | 适合场景 |
|---|---:|---|---|
| `public` | 否 | 直接 HTTP 请求公开接口 | 公开 API、无需登录的数据 |
| `cookie` | 是 | 复用浏览器 Cookie 和登录态 | 登录后才能访问的 feed、收藏、通知 |
| `header` | 是 | 获取或复用动态 header/token | CSRF、Bearer token、签名 header |
| `intercept` | 是 | 让页面自己发请求，AutoCLI 拦截 | 复杂签名、懒加载、难复现请求 |
| `ui` | 是 | 直接操作页面 UI 或 DOM | 发帖、评论、Web App 控制 |

### 4.1 public

`public` 是最轻量的策略。它不需要浏览器，也不需要登录态，Pipeline 可以直接请求公开 API。

适合：

- Hacker News
- Wikipedia
- Arxiv
- 公开榜单
- 无需登录的 JSON 接口

优点是快、稳定、容易测试。缺点是拿不到登录后数据，也处理不了动态签名和私有接口。

### 4.2 cookie

`cookie` 需要浏览器。它通常在已登录页面中执行 JavaScript：

```js
fetch(url, { credentials: 'include' })
```

这样请求会自动携带浏览器会话中的 Cookie。它适合登录后才能访问的数据，例如个人动态、收藏、通知、后台数据等。

### 4.3 header

`header` 适用于 Cookie 不够的场景。有些接口还要求：

- `Authorization`
- CSRF token
- 设备 ID
- 动态签名
- nonce 或 request id

这些值可能来自页面状态、localStorage、meta 标签、全局变量或之前的网络请求。`header` 策略表示 adapter 需要从浏览器上下文中获取或复用这些 header。

### 4.4 intercept

`intercept` 适合请求难以复刻的场景。它不强行手写完整请求，而是让真实页面按正常流程触发请求，然后 AutoCLI 捕获匹配的网络数据。

适合：

- GraphQL payload 很复杂
- 请求体有动态签名
- 点击 tab 后才加载数据
- 滚动后才分页加载
- 接口参数难以推断

它比 `public` 和 `cookie` 更强，但也更脆弱，因为依赖页面交互和选择器。

### 4.5 ui

`ui` 是最重的策略，更接近 RPA。它直接操作页面 UI 或 DOM：

- 点击按钮
- 输入文本
- 读取 DOM
- 提交表单
- 控制 Web 或 Electron 应用

适合没有稳定 API 或命令本身是“动作”的场景，例如发帖、评论、发布内容、控制 ChatGPT/Codex/Cursor 等。

策略选择原则：

```text
能 public 就 public
需要登录就 cookie
Cookie 不够就 header
请求难复现就 intercept
没有稳定 API 或要操作页面就 ui
```

## 5. 为什么需要 Browser 模式

纯网络请求是最优选择，但现代网站经常让纯请求不够用。

Browser 模式存在的原因包括：

- 登录态可能不只是 Cookie，还包括 localStorage、sessionStorage、IndexedDB、CSRF token
- 接口参数可能由前端动态签名
- API 可能需要浏览器生成的 header
- 数据可能只有滚动、点击、搜索或切换 tab 后才加载
- 有些数据从 DOM 提取比从私有 API 提取更稳定
- 动作类命令往往需要真实 UI 行为
- 反自动化系统可能拒绝普通 HTTP 客户端

因此 AutoCLI 的取舍是：

```text
能直接请求就直接请求
直接请求不可行或成本太高时使用 Browser 模式
```

## 6. 鸿蒙原生应用做 AutoCLI 的可行性和限制

如果目标是不依赖 WebView，而是在鸿蒙原生 App 层面做 AutoCLI，需要重新定义执行层。

桌面 AutoCLI 依赖：

- 普通 CLI 进程
- 本地 daemon
- Chrome 扩展
- Chrome debugger/CDP
- 用户级文件和进程权限
- 外部命令执行能力

普通鸿蒙 App 则受到移动系统沙箱限制，不能随意：

- 读取其他 App 私有文件或 token
- 复用其他 App 的登录态
- 拦截其他 App 的网络请求
- 向其他 App 注入代码
- 执行任意 shell 命令
- 无授权控制任意第三方 App UI
- 像桌面 daemon 一样长期后台运行

这意味着鸿蒙原生版不应直接照搬 AutoCLI 的 Web 策略，而应设计原生策略。

## 7. 鸿蒙原生 AutoCLI 的策略建议

更适合鸿蒙原生场景的策略包括：

| 原生策略 | 含义 |
|---|---|
| `public_api` | 调用公开后端 API |
| `authorized_api` | 用户在 AutoCLI App 内授权后调用 API |
| `ability` | 调用目标 App 暴露的 Ability 或 Service |
| `sdk` | 通过合作 App 或平台提供的 SDK 调用能力 |
| `accessibility` | 经用户授权后使用辅助功能读取和操作 UI |
| `uitest` | 在开发、测试或企业环境中使用 UI 测试自动化 |
| `system` | 使用系统签名或企业设备管理能力 |

这意味着系统定位要从：

```text
面向网站的浏览器自动化
```

转为：

```text
面向原生 App 和系统服务的能力编排
```

## 8. 鸿蒙原生版的实际产品方向

### 8.1 最优方向：自有 App 和合作 App

最现实的方向是让自有 App 或合作 App 主动暴露能力，AutoCLI 负责统一编排和调用。这里的关键不是让 AutoCLI “破解”其他 App，而是让目标 App 提供稳定、授权、可描述的能力入口。

目标 App 可以通过以下形式暴露能力：

| 暴露形式 | 适合能力 | AutoCLI 调用方式 |
|---|---|---|
| `ServiceExtensionAbility` + RPC | 动作类、查询类、需要业务逻辑封装的能力 | 绑定目标服务，调用约定好的方法 |
| `DataShareExtensionAbility` | 列表、记录、收藏、历史、配置等结构化数据 | 通过 URI 查询、插入、更新或删除数据 |
| `Want` / Deep Link / `UIAbility` | 打开页面、跳转详情、触发轻量动作 | 构造 Want，拉起目标 Ability |
| SDK / HAR 包 | 深度合作、强类型接口、复杂业务能力 | AutoCLI 集成 SDK adapter |
| 云端 OpenAPI | 跨设备、跨端一致的数据服务 | 使用 `public_api` 或 `authorized_api` 调用后端接口 |

推荐优先级是：

```text
云端 OpenAPI / authorized_api
-> ServiceExtensionAbility + RPC
-> DataShareExtensionAbility
-> SDK / HAR
-> Want / Deep Link
-> UI 自动化
```

原因是 API 和 Service 能提供稳定输入输出，DataShare 适合结构化数据，Want 更适合导航和拉起页面，而 UI 自动化只能作为兜底。

合作 App 最好同时暴露一份能力描述文件，类似原生版 adapter manifest：

```yaml
app: com.example.news
displayName: Example News
version: 1

capabilities:
  - name: hot
    description: 获取新闻热榜
    strategy: ability
    transport: service
    bundleName: com.example.news
    abilityName: NewsServiceAbility
    method: getHot
    args:
      limit:
        type: int
        default: 20
    returns:
      type: array
      item:
        title: string
        source: string
        time: string

  - name: favorite
    description: 获取用户收藏
    strategy: datashare
    uri: datashareproxy://com.example.news/favorites
    permission: ohos.permission.READ_NEWS_FAVORITES
```

AutoCLI 可以通过这份 manifest 生成 adapter，或在运行时发现可用能力。这样用户输入 `news hot --limit 20` 时，AutoCLI 实际执行的是一次受控的 Ability、DataShare、SDK 或 API 调用，而不是读取目标 App 沙箱。

示例：

```yaml
app: com.example.news
name: hot
strategy: ability

args:
  limit:
    type: int
    default: 20

pipeline:
  - callAbility:
      bundleName: com.example.news
      abilityName: NewsDataAbility
      action: getHot
      params:
        limit: "${{ args.limit }}"
  - map:
      title: "${{ item.title }}"
      source: "${{ item.source }}"
      time: "${{ item.time }}"

columns: [title, source, time]
```

如果使用 Service/RPC 方式，调用链可以设计成：

```text
用户命令
-> AutoCLI 解析 adapter
-> 校验参数和权限
-> 绑定目标 App 暴露的 ServiceExtensionAbility
-> 调用 method，例如 getHot(limit)
-> 接收结构化结果
-> Pipeline 做 map、filter、limit、输出格式化
```

如果使用 DataShare 方式，调用链可以设计成：

```text
用户命令
-> AutoCLI 解析 adapter
-> 构造 DataShare URI 和查询条件
-> 系统权限校验
-> 目标 App 返回结构化数据
-> Pipeline 做字段映射和输出
```

如果使用 Want / Deep Link 方式，AutoCLI 更适合执行“打开某页面”或“把用户带到指定上下文”的命令，例如：

```yaml
app: com.example.news
name: open_article
strategy: ability

args:
  id:
    type: string
    required: true

pipeline:
  - startAbility:
      bundleName: com.example.news
      abilityName: NewsMainAbility
      action: viewArticle
      params:
        id: "${{ args.id }}"
```

这种模式最稳定，也最符合平台安全模型，因为目标 App 是主动参与的。它同时保留了 AutoCLI 的核心思想：用 adapter 描述能力，用 Pipeline 统一编排，用最轻量可行的策略执行。

需要注意的是，原生 App 主动暴露能力时必须把安全边界前置：

- 能力必须显式声明，不应默认暴露全部内部接口。
- 敏感能力需要用户授权、调用方白名单、签名校验或权限校验。
- adapter 只能调用 manifest 中声明过的能力。
- 返回数据应最小化，避免泄露内部 token、设备标识和隐私字段。
- 动作类能力需要幂等性、确认机制或审计日志。

### 8.2 公开 API 和用户授权 API

这也是可行方向。AutoCLI App 可以直接请求公开 API，也可以让用户在 AutoCLI App 内完成授权登录，然后调用对应接口。

限制是：它不能从另一个 App 的沙箱中拿 token。

### 8.3 原生 UI 自动化

辅助功能或 UI 测试能力可以支持类似 RPA 的命令：

```text
启动 App
查找文本
点击
输入
滑动
读取可见文本
提取列表
```

这适合测试、企业流程或个人自动化，但它比 API/SDK 更脆弱，对权限和合规要求也更高。

### 8.4 不推荐方向：普通 App 控制任意第三方 App

如果目标是做一个普通上架 App，自动读取和控制任意第三方鸿蒙原生 App，这不是好的主方向。它会遇到：

- 平台权限限制
- 应用商店审核风险
- UI 易变导致稳定性差
- 数据可靠性差
- 用户隐私和合规风险

## 9. 桌面 AutoCLI 与鸿蒙原生 AutoCLI 对比

| 维度 | 桌面 AutoCLI | 鸿蒙原生 AutoCLI-like 系统 |
|---|---|---|
| 主要目标 | 网站和 Web App | 原生 App、服务和 API |
| 执行层 | CLI、daemon、Chrome 扩展 | App 沙箱、Ability、SDK、授权 API |
| 登录态复用 | 浏览器 Profile 和扩展 | 只能使用自身或用户授权的凭证 |
| 外部命令 | 支持较强 | 基本不可用 |
| 浏览器自动化 | 通过 Chrome 扩展/CDP 实现 | 如果不使用 WebView，则不适用 |
| 第三方 App 控制 | 桌面可通过自动化权限实现一部分 | 普通 App 严格受限 |
| 最适合场景 | AI Agent 获取 Web 数据 | 原生应用能力编排 |

## 10. 建议

对 AutoCLI 本身：

- 优先使用 `public`，因为最快、最稳定。
- 只有遇到登录态、动态 token、复杂请求或 UI 操作时才使用浏览器策略。
- 将规则生成视为 adapter 启动器，而不是绝对正确的最终结果。
- 对复杂页面使用 AI 生成，但仍需要验证和人工微调。

对鸿蒙原生 AutoCLI：

- 第一阶段支持 `public_api`、`authorized_api`、`ability`。
- 设计严格、可验证的 adapter schema。
- 不要把“控制任意第三方 App”作为默认承诺。
- 如果有生态合作需求，尽早支持 SDK adapter。
- UI 自动化适合作为高级能力或企业能力，而不是基础能力。
- AI 生成 adapter 的输入应从网页上下文转为 API schema、Ability 元数据、SDK 文档、UI 录制轨迹或测试 trace。

## 结论

AutoCLI 最值得复用的思想不是“爬网页”，而是：

```text
用声明式配置描述一个能力
用最轻量可行的策略执行它
把结果统一成结构化输出
```

在桌面环境中，可用策略包括 HTTP 请求、浏览器会话、请求拦截、页面 JavaScript 和外部 CLI。

在鸿蒙原生环境中，可用策略应转换为公开 API、用户授权 API、Ability 调用、SDK 集成、辅助功能、测试自动化和系统级能力。

Adapter 和 Pipeline 的抽象可以迁移；桌面浏览器和 CLI 的执行假设不能直接迁移。
