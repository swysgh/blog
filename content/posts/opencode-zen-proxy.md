---
title: 反代 OpenCode Zen：两种把免费模型变成 OpenAI 兼容 API 的实现路线
subtitle:
date: 2026-10-08T00:47:00+08:00
slug: opencode-zen-proxy
draft: false
description: 对比 9router 与 OpenCode2API 两种反代 OpenCode Zen 的思路：伪造 HTTP 客户端指纹 vs 跑一个真的 opencode serve 当上游。
keywords: OpenCode Zen 反代 9router OpenCode2API OpenAI 兼容 网关 免费模型
weight: 0
categories:
  - 教程
collections:
  - 搭建
tags:
  - opencode
  - 反代
  - ai
---

OpenCode Zen 上有一批不用付费就能用的模型。麻烦在于，它不是"给你一个 key 就随便调"的普通 API：免费档的请求必须**看起来像是 OpenCode 官方客户端发出来的**，否则直接 403。

于是就有了"反代 OpenCode"这件事。这里整理两个真实项目的实现方式：

- [decolua/9router](https://github.com/decolua/9router)：一个通用 AI 路由器，把 OpenCode Zen 当成 40+ 个上游之一，靠**伪造 HTTP 指纹**过闸；
- [TiaraBasori/OpenCode2API](https://github.com/TiaraBasori/OpenCode2API)：在你本机跑一个**真的 `opencode serve`** 当上游，代理只做协议翻译，指纹天然合法。

两条路线解决的是同一个问题，代价和收益完全不同。下面先把上游讲清楚，再分别拆实现。

## 一、上游长什么样：OpenCode Zen 的模型分档与端点

OpenCode Zen 是 OpenCode 官方维护的模型网关（curated list），本质是一个"按模型选端点"的聚合 API。它不是单一 OpenAI 兼容端点，而是按模型对应的 AI SDK 包分流：

| 端点 | 格式 | 典型模型 |
|:-----|:-----|:---------|
| `https://opencode.ai/zen/v1/chat/completions` | OpenAI Chat Completions | GLM、Kimi、MiniMax、DeepSeek、Big Pickle、`*-free` |
| `https://opencode.ai/zen/v1/messages` | Anthropic Messages | Claude 系列、Qwen Plus |
| `https://opencode.ai/zen/v1/responses` | OpenAI Responses | GPT 系列、Grok、Muse Spark |
| `https://opencode.ai/zen/v1/systemone` | 自有协议 | Jev（结构化决策模型，不产文本） |

另外两个辅助端点：

- `GET /zen/v1/models`：模型清单与元数据；
- `GET /zen/v1/usage`：带 key 时的配额用量（rolling / weekly / monthly 百分比）。

在 OpenCode 自己的配置里，模型 id 写作 `opencode/<model-id>`，例如 `opencode/gpt-5.5`。

除了免费档，Zen 还有两条"带 key"的车道，路径前缀不同：

| 车道 | 前缀 | 说明 |
|:-----|:-----|:-----|
| OpenCode Free | `/zen/v1/*` | 无需认证，`Authorization: Bearer public` |
| OpenCode Zen（PAYG） | `/zen/v1/*` | 按量付费，用 key 换同一批模型（含付费模型） |
| OpenCode Go | `/zen/go/v1/*` | $5/月的订阅档，Kimi / GLM / Qwen / MiMo / MiniMax 等 |

### 免费档到底卡什么

这是整篇文章的关键。从两个项目的实测注释里能拼出免费档的校验项：

1. **工具列表**必须是官方客户端的那一份，至少包含 `bash` / `glob` / `grep` / `read` 四件套。少一个、多一个变体、或者按请求把工具关掉，都会 403 `FreeTierError`，报错文本是 `free tier can only be used from within OpenCode`。
2. **必须流式**。免费模型上 `stream: false` 会被 403 拒绝。
3. **要带 OpenCode 自己的会话头**。`x-opencode-session` 是上游用来算配额和做上下文归类的标识，格式形如 `ses_<12位十六进制><14位base62>`；请求级还有一个 `x-opencode-request`（`msg_...`）。9router 的代码注释提到，OpenCode Go 从 2026-09-06 起会拒绝缺少该头的请求。
4. **配额按 session 计**。每个请求都新造一个 session id，会以 `429 FreeUsageLimitError` 的形式把额度烧穿，并且 reset 延迟越来越长——真客户端是"一段会话复用同一个 session"，反代必须模仿这一点。
5. **UA 版本**要够新：`opencode/<major>.<minor>`，判定条件是 major > 1 或 (major == 1 且 minor >= 17)。

所以"反代 OpenCode"的技术核心不是协议转换（那部分反而是最简单的），而是**让上游认为对面坐着一个真的 OpenCode 客户端**。两个项目在这一步上选了相反的路。

## 二、两条路线的整体形状

```
路线 A：纯 HTTP 反代（9router）
  Claude Code / Cline / opencode ...
        │  http://127.0.0.1:20128/v1   （OpenAI 兼容）
        ▼
  [ 9router 路由器 ]  指纹伪造 + 格式翻译 + 配额统计 + 多账号轮询
        │  https://opencode.ai/zen/v1/{chat/completions|messages|responses}
        ▼
     OpenCode Zen

路线 B：运行时封装（OpenCode2API）
  OpenAI 兼容客户端
        │  http://127.0.0.1:10000/v1   （OpenAI 兼容）
        ▼
  [ opencode2api 代理 ]  协议翻译 + 会话/工具编排
        │  @opencode-ai/sdk → http://127.0.0.1:10001
        ▼
  [ 真的 opencode serve 进程 ]  ← 官方客户端行为，指纹合法
        │
        ▼
     OpenCode Zen
```

一句话概括区别：**A 把 Zen 当 HTTP API 骗过去，B 把 Zen 当"真客户端的服务端"用过去。**

## 三、路线 A：9router 的指纹伪造

9router 本身不装 opencode、不起任何本地进程。它把 `https://opencode.ai` 登记成一个普通 provider，然后在出站请求上做三件事：改头、改 ID、改工具列表。

### 3.1 provider 注册表

它用一份声明式的注册表描述上游（`open-sse/providers/registry/opencode.js`）：

```js
export default {
  id: "opencode",
  alias: "oc",
  category: "free",
  noAuth: true,                       // 免费档：不需要任何凭证
  transport: {
    baseUrl: "https://opencode.ai",
    headers: { "x-opencode-client": "desktop" },
    forceStream: true,                // 免费档必须流式
    quirks: {
      forceAutoToolChoiceModels: ["muse-spark-1.3-contributor-free"],
    },
  },
  models: [
    { id: "muse-spark-1.2-contributor-free", targetFormat: "openai-responses" },
    { id: "union-alpha", targetFormat: "claude" },
  ],
  modelsFetcher: { url: "https://opencode.ai/zen/v1/models", type: "opencode-free" },
  passthroughModels: true,            // 清单外模型也放行，不写死
};
```

`targetFormat` 决定这个模型的请求要翻译成哪种格式、发到哪个端点；`passthroughModels` 让它不必跟着上游模型清单改动而发版——模型列表直接抓 `/zen/v1/models`。

### 3.2 出站请求头

免费档的 executor 里，最终发出去的头是这一组（`open-sse/executors/opencode.js` 的 `buildHeaders`）：

| 头 | 值 | 为什么 |
|:---|:---|:-------|
| `Authorization` | `Bearer public` | 免费档的"占位"凭证，不是真 key |
| `User-Agent` | `opencode/1.18.31` | 版本要满足 ≥ 1.17；若下游 UA 本身就是合法的 opencode UA 则原样透传 |
| `x-opencode-client` | `desktop` | 官方客户端的客户端类型 |
| `x-opencode-session` | `ses_...` | 会话标识，配额按它算 |
| `x-opencode-request` | `msg_...` | 请求标识，重试时要保持一致 |
| `x-opencode-project` | `global` | 缺省项目 |
| `Accept` | `text/event-stream` | 配合强制流式 |
| `anthropic-version` | Anthropic 版本号 | 仅当目标是 `/messages` 端点时补 |

注意 `Authorization: Bearer public` 这一手：它说明上游对免费档**根本没有做认证**，只是把"客户端指纹"当门禁——这也解释了为什么这条路能走通。

### 3.3 会话 ID 与请求 ID：既要像，还要稳

这是 9router 里最讲究的一块。上游要的 `ses_...` 有两种来源：

- 下游如果本来就是 opencode（UA 合法且带了 `x-opencode-session`），原样透传；
- 否则，下游（Claude Code、Cline、Cursor…）给的会话标识格式五花八门，必须**确定性地映射**成 opencode 的格式：

```js
// ses_ + 12 位十六进制 + 14 位 base62
export const OPENCODE_SESSION_RE = /^ses_[0-9a-f]{12}[0-9A-Za-z]{14}$/;

export function translateSessionId(sessionId, clientTool = "") {
  const digest = crypto.createHash("sha256")
    .update(`opencode\0${clientTool || "generic"}\0${sessionId || ""}`)
    .digest();
  // 前 6 字节 → 12 位 hex，其后 14 字节 → 14 位 base62
  return `ses_${timeHex}${randomPart}`;
}
```

两个性质很关键：

- **幂等**：同一个下游会话每次都算出同一个上游 id，于是上游看到的是一条连续的会话，而不是每请求一个新 session（否则就是前面说的 429）。
- **隔离**：命名空间里塞了 `clientTool`，所以不同 agent 用同一个原始 id 也不会撞车。

下游连会话标识都不给时，退化成 `stableSessionId()`：按 `connectionId` 或 `Authorization` 头的 sha256 做键，维护一张最多 1000 条的 LRU 表，给每个下游身份固定一个 session。请求 ID 同理——真客户端重试时会复用同一个 `msg_...`，所以它用 `sha256(会话 + 最后一条用户消息)` 推导，保证重试幂等。

### 3.4 工具指纹四件套

免费档要求工具列表里有 `bash`/`glob`/`grep`/`read`。但下游客户端声明的可能是 `Bash`、`Read` 这种大小写不同的名字，而**同名变体同时出现会被上游当重复拒绝**。所以 `opencodeFingerprint.js` 做三件事：

1. 把四件套的大小写归一化成小写，并去重（只保留一份声明）；
2. 补齐缺失的成员（补一条"当前不可用、禁止调用"的空声明，纯粹为了满足指纹）；
3. 用 `WeakMap` 记住"发出去的名字 → 调用方原来的名字"，在响应侧再把 `tool_calls` / `function_call` 里的名字还原回去。

第 3 步是细节里的细节：如果不还原，下游客户端会收到一个它不认识的工具名，工具调用链直接断掉。

顺带一个语义修正：聊天请求若本来没有工具，注入的诱饵声明会把 `tool_choice` 设成 `none`，避免模型去调那四个假工具。

### 3.5 端点分派与格式翻译

同一个 provider 下，不同模型发往不同端点，翻译规则也不一样：

| 情况 | 处理 |
|:-----|:-----|
| 目标是 `/responses` | `max_tokens` / `max_completion_tokens` 改名为 `max_output_tokens`；`reasoning_effort` 收成 `reasoning: {effort, summary: "auto"}`；`store: false`；工具声明从 `{function:{...}}` 拍平成 `{name, parameters}` |
| Responses 的历史项 | 丢掉上一轮的 `reasoning` 项并删掉 `encrypted_content`——上游是账号池，`encrypted_content` 只能被签发它的账号解密，跨账号转发会 400 |
| 目标是 `/messages` | 补 `anthropic-version` 头，按 Anthropic 格式翻译 |
| 其他 | OpenAI Chat Completions 原样 |
| 非流式客户端 | 上游强制 `stream: true`，由代理自己聚合后再返回 |

`reasoning_effort` 还有一层降级映射：请求 `max`/`ultra` 而模型只支持到 `xhigh` 时，往下降一级，而不是把非法值直接扔给上游。

### 3.6 三档 provider 与配额

同一份代码库用三个注册表项覆盖 Zen 的三条车道：`oc`（免费，无认证）、`ocz`（PAYG，带 key）、`ocg`（Go 订阅，`/zen/go/v1/*`）。配额查询走 `GET /zen/v1/usage`，把返回的 rolling / weekly / monthly 百分比转成统一的配额结构，喂给面板和自动降级（订阅 → 便宜 → 免费）。

有意思的是它的第三个 executor 是**为了兼容一个上游变更而单独加的**：`opencode-go` 原本用通用 executor，但上游从 2026-09-06 起要求 `x-opencode-session`，于是专门写了 `OpenCodeGoExecutor`，只为了在头里多塞一个字段。这基本就是路线 A 的日常——上游改一次指纹，反代就要跟一次。

### 3.7 顺手把 opencode 配成自己的客户端

9router 还有一个反向的便利功能：直接写 `~/.config/opencode/opencode.json`，把 9router 注册成 opencode 的一个 OpenAI 兼容 provider：

```json
{
  "provider": {
    "9router": {
      "npm": "@ai-sdk/openai-compatible",
      "options": { "baseURL": "http://127.0.0.1:20128/v1", "apiKey": "<从面板复制的 key>" },
      "models": {
        "cc/claude-opus-4-6": {
          "name": "cc/claude-opus-4-6",
          "modalities": { "input": ["text", "image"], "output": ["text"] }
        }
      }
    }
  }
}
```

于是 opencode 自己也能用上路由器后面的 40+ 上游——反代和被反代互为客户端。

## 四、路线 B：OpenCode2API 的运行时封装

OpenCode2API 不伪造任何东西。它假设"合法客户端"这件事无法长期伪造，于是**直接跑一个真的 opencode**，把自己插在客户端和这个进程之间做协议翻译。

### 4.1 后端编排：把 opencode serve 当成上游

`ensureBackend()` 干的事：

```js
// 1) 先探活：/global/health（不是 /health，那不是 API）
// 2) 探不到且 MANAGE_BACKEND=true 就自己拉起来
spawn(opencodeBin, ['serve', '--port', '10001', '--hostname', '127.0.0.1'], {
  cwd: emptyWorkspace,          // 空工作区，避免把用户项目当上下文
  env: {
    OPENCODE_CONFIG_CONTENT,    // 注入工具锁插件
    OPENCODE_SERVER_PASSWORD,   // 保护后端 HTTP API
    OPENCODE_API_KEY,           // 可选：付费模型用
  },
});
// 3) 最多轮询 60 次 /global/health 等它就绪
```

几个设计点值得抄：

- **后端只监听 127.0.0.1 并设密码**：后端 API 是"完全体 opencode"，暴露出去等于把整台机器的 shell 交出去。
- **`OPENCODE_CONFIG_CONTENT` 注入插件**：代理拉起后端时，会把 `plugin/opencode2api-tool-lock.js` 的绝对路径合并进后端配置的 `plugin` 数组（保留用户原有的 plugin）。这是后面工具策略的落点。
- **空工作区 + 可选隔离 HOME**（`USE_ISOLATED_HOME`）：默认复用本机登录态（能直接用已登录的 Zen 免费档），需要干净环境时切到 jail 目录。反过来，容器里出现"`/v1/models` 正常但请求卡住"，官方排查建议就是把 `USE_ISOLATED_HOME` 设成 `false`。

### 4.2 用官方 SDK 当上游客户端

代理依赖 `@opencode-ai/sdk`，把 opencode 的 server API 当普通 HTTP 服务用：

| SDK 调用 | 用途 |
|:---------|:-----|
| `client.config.providers()` | 拉模型清单，转成 `/v1/models`（id 形如 `opencode/big-pickle`） |
| `client.config.update({activeModel})` | 设置本次请求用哪个模型 |
| `client.session.create()` | 建会话 |
| `client.session.prompt({path, body})` | 发一轮对话（核心） |
| `client.session.messages()` | 轮询拉消息（回程兜底） |
| `client.event.subscribe()` | 订阅全局 SSE 事件流（回程主路） |
| `client.session.delete()` | 清理会话 |
| `client.config.get()` | 检查工具锁插件是否已加载 |

### 4.3 请求翻译：OpenAI messages → opencode parts

`buildPromptParts()` 把 OpenAI 的消息数组拍平成 opencode 的 `parts`：

- `system` 单独抽出来，走 `body.system`；
- 普通文本变成 `USER: ...` / `ASSISTANT: ...` 这样的角色前缀文本 part（opencode 的 part 本身不带角色）；
- `image_url` 抓取后转成 data URI 的 `file` part（限流、失败就跳过，不拖垮整个请求）；
- 历史的 `tool_calls` 序列化成 `<function_calls>[...]</function_calls>` 文本，`tool` 角色消息序列化成 `TOOL_RESULT: {...}`。

然后一次 `session.prompt` 发出去：

```js
const promptParams = {
  path: { id: sessionId },
  body: {
    model: { providerID: pID, modelID: mID },
    system: systemWithGuard,
    parts,                    // 最后再追加一条"契约提醒"
    temperature, top_p, max_tokens,
  },
};
```

### 4.4 回程：SSE 事件流为主，轮询兜底

这是路线 B 的工程重心。opencode 的回复有两条取回路径：

**主路 `collectFromEvents()`**：订阅 `client.event.subscribe()` 的全局事件流，只挑属于本会话的事件（`message.part.updated` / `message.part.delta` / 消息完成事件），边收边翻成 OpenAI 的 SSE chunk。它有三个超时/状态处理：

| 参数 | 默认 | 作用 |
|:-----|:-----|:-----|
| `EVENT_FIRST_DELTA_TIMEOUT_MS` | 30000 | 首个 delta 迟迟不来就放弃事件流 |
| `EVENT_IDLE_TIMEOUT_MS` | 8000 | 空闲太久就收尾；**但只要有内置工具正在执行，就不判空闲**，继续等 |
| `REQUEST_TIMEOUT_MS` | 180000 | 整体上限 |

"工具执行中不判空闲"这条是从真实故障里抠出来的：早期实现里，模型调了内置工具、后端在跑 `bash`，事件流上恰好没有文本 delta，空闲超时就把响应截断了。

**兜底 `pollForAssistantResponse()`**：每 500ms 拉一次 `session.messages()`，直到消息完成（`finish` 存在且不是 `tool`，或 `time.completed` 存在）。触发条件包括：事件流空闲超时、事件流为空、只流出了 reasoning 而没有正文。这里还有个坑被写进注释：推理模型先出 `reasoning` part、后出文本 part，**看到第一个非空快照就返回会把答案截成只剩思考过程**，所以中途快照只作为超时兜底，正常要等消息真正结束。

### 4.5 会话模型：和路线 A 完全不同

| 端点 | 会话策略 |
|:-----|:---------|
| `/v1/chat/completions` | **每个请求新建一个 session**，历史由客户端每次重发；出错时删掉该 session |
| `/v1/responses` | 支持 `previous_response_id`：把返回的 `resp_*` 映射到上游 session，**30 分钟 TTL**，过期后清扫并顺手删掉不再被引用的上游会话 |

也就是说，路线 B 在聊天端点上是"无状态"的——它不试图猜下游的会话边界，而是把 opencode 的 session 当成一次性的执行上下文。这跟路线 A 的思路正好相反：A 拼命让同一个下游会话复用同一个上游 session，因为免费档配额按 session 计；B 之所以敢每请求一个 session，是因为**流量是走真 opencode 发出去的，配额消耗方式与官方客户端一致**。

另外还有一个可选的存储清理（`AUTO_CLEANUP_CONVERSATIONS`）：直接删 `~/.local/share/opencode/storage/{message,session}` 下超过 `CLEANUP_MAX_AGE_MS` 的目录，默认关闭。

### 4.6 工具：三态策略 + 工具锁插件 + 文本契约

这是路线 B 最有意思的部分，因为它撞上了和路线 A 同一堵墙——**免费档不允许你改工具列表**。B 的解法不是伪造，而是"列表原样不动，策略换个地方带进去"。

**（1）三态策略**，由请求是否带 `tools` 决定：

| 模式 | 触发 | 行为 |
|:-----|:-----|:-----|
| `external-bridge` | 请求带 `tools` | 客户端的工具被代理虚拟化，opencode 内置工具全程禁用 |
| `internal-allowlist` | 请求不带 `tools` | 只放行 `OPENCODE_INTERNAL_ALLOWED_TOOLS` 里的内置工具（如 `webfetch`） |
| `disabled` | 默认（`DISABLE_TOOLS=true`） | 所有工具关闭 |

请求体里可以传 `opencode.internal_allowed_tools` 做请求级覆盖，但**只能求交集缩小，不能扩大**——这是刻意的权限收窄设计。

**（2）工具锁插件**：既然不能按请求改工具列表，就把策略写进**会话标题**：

```text
opencode2api [tools:none]
opencode2api [tools:*]
opencode2api [tools:webfetch,read]
```

代理建会话时设置标题，后端插件在 `tool.execute.before` 钩子里解析标题（沿 `parentID` 向上找，支持子会话继承），不在白名单的工具直接抛错拒绝：

```js
"tool.execute.before": async (input, output) => {
  const policy = await policyOf(input.sessionID);   // 从会话标题取策略
  if (policy === "*") return;
  const allowed = policy && policy !== "none" ? policy.split(",") : [];
  if (allowed.some((name) => matches(input.tool, name))) return;
  throw new Error(`Tool "${input.tool}" is disabled by opencode2api.`);
}
```

如果后端不是由代理拉起的（`MANAGE_BACKEND=false`），插件就没加载，代理会退回"按请求覆盖 `tools` map"的老办法——而这条路在免费模型上会被拒，日志里会明确警告。这个降级链本身就是免费档限制的证据。

**（3）外部工具的文本契约**：客户端的 `tools` 不注册成 opencode 内置工具，而是变成系统提示里的一段契约——需要调工具时，模型的**整条回复只能是一个 `<function_calls>` 块**：

```text
<function_calls>{"name":"external__web_fetch","arguments":{"url":"https://example.com"}}</function_calls>
```

工具名加了 `external__` 命名空间前缀，避免和内置工具同名冲突。响应侧再把这段标记解析回标准的 `tool_calls` / `function_call`，客户端完全看不出中间发生了什么。

**（4）方言解析**：免费模型经常无视契约，回落到自己训练时用的格式。解析器里列了一份"外来方言"清单，全部归一化成 `<function_calls>`：

| 方言 | 形态 |
|:-----|:-----|
| DSML（DeepSeek 原生） | `<｜｜DSML｜｜tool_calls><｜｜DSML｜｜invoke name="...">...`（注意是全角竖线 U+FF5C） |
| JSON 包装 | `<tool_call>{"name":...,"arguments":{...}}</tool_call>` |
| 带属性标签 | `<external__bash arguments='{"command":"ls"}'/>` |
| 带正文标签 | `<external__bash>{"command":"ls"}</external__bash>` |
| 裸 JSON | 整条消息就是一个 `{"name":...,"arguments":{...}}` |

歧义策略很克制：前两种自带分隔符，无条件识别；后三种**只有名字命中本次请求的注册表**才认，且裸 JSON 必须占满整条消息，免得把散文里的 JSON 片段当成工具调用。

**（5）契约提醒的位置**：契约只放在系统提示里是不够的——注释里给了实测数据：只放系统提示时，一个明显的单工具请求 8 次里只有 4 次产出可解析的标记；把契约提醒**作为最后一个 part 追加**后变成 8/8。原因是 agent 客户端的系统提示动辄 16KB+，契约会被淹掉。所以代理会把一段简短的提醒拼在 `parts` 末尾。

官方文档还建议 agent 客户端把"调工具"和"答问题"拆成两跳（先要求只产出工具调用，回灌结果后再问），比混在一条消息里稳定。

### 4.7 可观测性

路线 B 的运维面做得更细：`/health/details` 输出结构化诊断（工具模式计数、工具发现失败次数、工具 id 缓存状态），`/metrics` 输出 Prometheus 文本指标，两者都能单独开关并单独要求鉴权。工具模式的选择、工具发现、降级原因都会打日志，但**不记录工具返回内容**。

## 五、两条路线怎么选

| 维度 | 路线 A：纯 HTTP 反代（9router） | 路线 B：运行时封装（OpenCode2API） |
|:-----|:--------------------------------|:-----------------------------------|
| 依赖 | 一个 Node 进程，不装 opencode | Node 进程 + 本机安装的 opencode |
| 上游形态 | `opencode.ai` 的 HTTP API | 本地 `opencode serve` 进程（SDK + SSE） |
| 免费档过闸方式 | 伪造 UA / 头 / 会话 id / 工具指纹 | 真客户端，无需伪造 |
| 上游变更的脆弱性 | 高：指纹、会话头、工具列表任一改动都要跟 | 低：跟随 opencode 自身的兼容性 |
| 会话语义 | 下游会话 → 稳定映射到上游 session（配额友好） | chat 每请求一个 session；responses 用 `previous_response_id` |
| 工具调用 | 原生 `tool_calls`，直接透传 | 文本契约虚拟化 + 方言解析，再翻回 `tool_calls` |
| 多账号 / 多档 | 内建轮询、配额面板、自动降级 | 单后端单身份 |
| 可观测性 | 面板 + 用量统计 | `/health/details` + Prometheus 指标 |
| 适合谁 | 已有网关/多上游，想把 Zen 当一个免费上游挂进去 | 只想把免费模型稳定地喂给某个 agent 客户端 |

实操上的粗略结论：

- 你的目标是"多一个免费上游、能进现有路由/网关体系"——走 A 的思路，把它当 OpenAI 兼容 provider 登记即可，代价是接受它对上游指纹的强耦合。
- 你的目标是"给 agent 客户端提供可预测的工具调用行为"——B 的桥接更可控（工具策略可收窄、调用格式归一化、两段式调用），代价是要多跑一个 opencode 进程，且工具语义由代理自己定义。
- 两条路都要接受同一个风险：**免费档的准入规则是上游随时可改的实现细节**，不是承诺的 API 契约。9router 因为一次会话头变更就要加一个专用 executor，就是最直接的例子。真要长期用，至少得有一组针对"免费档还能不能用"的回归测试。

## 参考

- [decolua/9router](https://github.com/decolua/9router) —— 重点看 `open-sse/providers/registry/opencode*.js`、`open-sse/executors/opencode*.js`、`open-sse/utils/opencodeFingerprint.js`
- [TiaraBasori/OpenCode2API](https://github.com/TiaraBasori/OpenCode2API) —— 重点看 `src/proxy.js`、`src/tool-runtime/`、`plugin/opencode2api-tool-lock.js`
- [OpenCode Zen 文档](https://opencode.ai/docs/zen/) —— 模型与端点对照表
- [OpenCode Server / SDK](https://opencode.ai/docs/) —— `opencode serve` 与 `@opencode-ai/sdk`
