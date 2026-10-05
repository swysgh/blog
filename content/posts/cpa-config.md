---
title: CPA（CLIProxyAPI）配置详解：网关 config.yaml 与 Manager Plus 配置中心
subtitle:
date: 2026-10-05T13:00:00+08:00
slug: cpa-config
draft: false
description: CPA 网关 config.yaml 重要参数、CPA Manager Plus 配置中心两种编辑方式
keywords: CPA CLIProxyAPI AI 网关 配置
weight: 0
categories:
  - 教程
collections:
  - 搭建
tags:
  - CPA
  - CLIProxyAPI
  - AI
  - 网关
---

## 关系和定位

把两个项目先分清楚，后面的配置就不会搞混：

- **CPA（CLIProxyAPI）**：后端网关本体。一个 Go 写的常驻进程，对外提供与 OpenAI / Claude / Gemini / Codex 等兼容的 HTTP 接口，把请求路由到底层各提供商的 API Key 或 OAuth 凭据。配置写在 `config.yaml` 里，CPA 直接读这份文件启动。
- **CPA Manager Plus（CPAMP）**：网关的**可视化运维面板**。可以把它当成 CPA 的「远程控制器」：在网页上浏览/改写 `config.yaml`、看请求监控、做用量分析。它本身不处理模型请求，只是替你在浏览器里编辑 CPA 的配置。

所以本文分两块：

- **A 部分**：CPA 本体要看的 `config.yaml` 选项——你直接编辑文件、或者通过面板间接编辑，最终都会落到这些字段上。
- **B 部分**：CPA Manager Plus 的「配置中心」面板——什么时候该用可视化、什么时候该切到源文件视图、Manager Server 模式的几个连接字段是什么。

> 命名约定：帮助站文档正文（`config.example.yaml` 的 v8 布局）使用 v8 字段名（`server.*` / `management.*` / `access.api-keys` / `oauth.*` / `routing.*` / `requests.*` 等），旧字段名（`host` / `port` / `remote-management`、根级 `api-keys` / `auth-dir` 等）仍被接受并由 CPA 在加载/保存时自动映射到 v8。本文统一按 v8 字段名写。

## A. CPA config.yaml 配置详解

### 最小可用配置

不写任何提供商之前，先让 CPA 能起起来、能被远端管理、能被客户端调用，这是最小骨架（v8 布局，开头必须设 `config-version: 8`）：

```yaml
# v8 标识：存在时必须是 8；只设这一行不会自动迁移旧字段
config-version: 8

# 监听地址和端口；空字符串表示绑定所有 IPv4/IPv6 接口
server:
  host: ""
  port: 8317

# 管理 API（CPAMP 调它就是走这里）
management:
  allow-remote: true
  secret-key: "change-me-to-a-long-random-string"

# 客户端调用模型时需要带的 API Key（任意字符串，可多个）
# 注意：v8 把"客户端 key"和"上游提供商分组"分开了——根级 api-keys 只用于上游分组
access:
  api-keys:
    - "sk-client-1"
    - "sk-client-2"

# OAuth 文件和本地凭据的存放目录，支持 ~
oauth:
  auth-dir: "~/.cli-proxy-api"
```

跑起来后，CPAMP 用 `management.secret-key` 接管配置，客户端用 `access.api-keys` 里的某个值访问 `/v1/chat/completions` 之类的端点。两组 key 千万别混。

### 服务器监听

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `server.host` | string | `""` | 绑定地址。空字符串等于 `0.0.0.0`/`::`，监听所有接口；想只让本机访问填 `127.0.0.1`。 |
| `server.port` | integer | `8317` | HTTP 端口。改之前确认新端口没被占用，且防火墙已放行。 |
| `server.tls.enable` | boolean | `false` | 是否启用 HTTPS。 |
| `server.tls.cert` / `server.tls.key` | string | `""` | 证书和私钥路径。`enable: true` 时必须填绝对路径。 |

要不要本地绑定，取决于部署模型：放在公网 VPS 上一般保留 `""` 然后交给前置 Nginx 做 TLS；放在内网想直接暴露给同事就保持 `""`；只想本机调试就 `127.0.0.1`。CPA 自己并不强制 TLS，所以公网部署通常用 Nginx/Caddy 在前面套一层。

### 管理 API（`management.*`）

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `management.allow-remote` | boolean | `false` | 是否允许非 localhost 的管理访问。即使为 `false`，localhost 仍需要 key。 |
| `management.secret-key` | string | `""` | 管理密钥。明文写在文件里没事，CPA 启动时会内存里哈希；**留空则所有 `/v0/management` 路由返回 404**——意味着彻底关掉管理 API。 |
| `management.disable-control-panel` | boolean | `false` | 是否禁用内置管理面板资源和路由（CPAMP 是单独部署的，这个开关主要影响 CPA 内嵌的那个静态页面）。 |
| `management.disable-auto-update-panel` | boolean | `false` | 是否禁止面板周期性后台更新。保留默认即可，除非你被面板升级炸过。 |
| `management.panel-github-repository` | string | 默认指向 `Cli-Proxy-API-Management-Center` 仓库 | 面板包来源，可换成自己的 fork 或私有 releases。 |

易错点：

- `secret-key` 留空 = 管理 API 完全下线（404），不是「免密」。CPAMP 会连不上。
- 想让 CPAMP 在另一台机器上连过来，必须 `allow-remote: true` 并且 `server.host` 不要绑 `127.0.0.1`。
- 明文 key 写在 `config.yaml` 里没关系（CPA 启动后内存里会哈希），但**这个文件本身要管好权限**，因为里面还会出现明文的 `access.api-keys` 和各种 provider key。

### 凭据目录与 API Key

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `oauth.auth-dir` | string | `~/.cli-proxy-api` | OAuth 登录和文件型凭据的存放目录，支持 `~`。多用户环境改成绝对路径更清楚。 |
| `access.api-keys` | string[] | `[]` | 客户端调用 CPA 时要带的 API Key。空数组意味着**任意 key 都能进来**——别这么留。 |

`access.api-keys` 是访问控制层面唯一一道闸：客户端请求里 `Authorization: Bearer *** 必须落在这份列表里。所以它要足够随机、足够长，并且定期轮换。OAuth 文件和 provider API key 都不在这层校验里，那是后面路由和配额的事。注意：v8 把「客户端 key」和「上游提供商分组」分到了两个不同的键上——客户端 key 在 `access.api-keys`，上游分组在 `api-keys.<provider>`，详见文末对照表。

### 日志与调试

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `observability.logs.debug` | boolean | `false` | 打开调试日志，输出更细的内部信息。 |
| `observability.logs.request-log` | boolean | `false` | 记录每个请求和响应的完整内容。**生产环境打开会非常占磁盘**，除非真在排查问题。 |
| `observability.logs.logging-to-file` | boolean | `false` | 日志写到滚动文件而不是 stdout。systemd / Docker 部署推荐打开，方便事后取证。 |
| `observability.logs.logs-max-total-size-mb` | integer | `0` | 日志目录总大小上限（MB），`0` 不限制。建议设一个值防止撑爆磁盘。 |
| `observability.logs.error-logs-max-files` | integer | `10` | 请求日志关闭时保留的错误日志文件数；`0` 不清理。 |
| `observability.usage.usage-statistics-enabled` | boolean | `false` | 内存用量统计聚合。CPAMP 的请求监控要用到这里。 |

一般部署建议：`observability.logs.logging-to-file: true` + 一个合理的 `observability.logs.logs-max-total-size-mb`（比如 500），其它都关。`observability.logs.debug` 和 `observability.logs.request-log` 只在排查时临时打开。

### 性能与诊断

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `server.commercial-mode` | boolean | `false` | 关闭高开销的请求日志和中间件以省内存。**给正式商用的多租户场景准备的**，单机自用一般不用开。 |
| `observability.pprof.enable` | boolean | `false` | 开启 pprof 调试端点。 |
| `observability.pprof.addr` | string | `127.0.0.1:8316` | pprof 绑定地址。**应该只绑本机**——这个端点能 dump 内存和 CPU profile，暴露到公网很危险。 |

### 网络与代理

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `requests.proxy-url` | string | `""` | 全局出站代理，支持 `socks5`/`http`/`https`。 |
| `routing.force-model-prefix` | boolean | `false` | `true` 时，没有前缀的模型请求只能匹配没前缀的凭据（除非前缀和模型名相同）。 |
| `requests.passthrough-headers` | boolean | `false` | 把经过筛选的上游响应头透传给客户端（比如某些流式响应里要带的 trace id）。 |

`requests.proxy-url` 是网关本身访问上游（Google / Anthropic / OpenAI）时走的代理；客户端→CPA 这一段不走它。本地某条凭据想绕开这个代理，把它的 `proxy-url`（key 级字段）设成 `direct` 或 `none` 即可。

`routing.force-model-prefix` 是个路由规则：当你用 `prefix/model` 调用带前缀的凭据时，强制要求请求必须带前缀才匹配该凭据。混用前缀和无前缀时避免误路由，但代价是「默认」入口被收窄，建议和你的客户端调用约定一起规划。

### 重试与冷却

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `routing.retry.request-retry` | integer | `3` | 对 403/408/500/502/503/504 的重试次数。 |
| `routing.retry.max-retry-credentials` | integer | `0` | 单次失败请求最多尝试多少条凭据。`0` 等于旧行为：尝试所有。 |
| `routing.retry.max-retry-interval` | integer | `30` | 重试前等冷却中凭据「醒来」的最长秒数。 |
| `routing.cooldown.disable-cooling` | boolean | `false` | 全局关掉凭据/模型冷却调度。 |
| `routing.cooldown.save-cooldown-status` | boolean | `false` | 把凭据冷却状态持久化为 `.cds` 文件，CPA 重启后接着用。 |
| `routing.cooldown.transient-error-cooldown-seconds` | integer | `0` | 临时错误（408/500/502/503/504）的冷却秒数；`0` 等于旧的 60 秒，`-1` 禁用。 |
| `oauth.providers.claude.disable-claude-cloak-mode` | boolean | `false` | 全局关闭 Claude 请求伪装；单条凭据仍可单独覆盖。 |

冷却机制的存在是因为同一组 OAuth 凭据被并发打爆会触发上游限流，所以 CPA 会把触发限流的凭据临时标记为「冷却中」，用其它凭据顶上。`routing.retry.request-retry` 是请求层重试，`routing.cooldown.transient-error-cooldown-seconds` 是凭据层冷却，两者协同工作。

调小 `routing.retry.max-retry-credentials` 可以缩短最坏情况下的延迟（不至于把所有凭据挨个试一遍），代价是失败的请求会更快报错。

### 流式响应与 WebSocket

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `requests.streaming.keepalive-seconds` | integer | `0` | SSE 保活间隔（秒）。`≤0` 禁用。 |
| `requests.streaming.bootstrap-retries` | integer | `0` | 首字节前的安全流式重试次数。 |
| `requests.nonstream-keepalive-interval` | integer | `0` | 非流式响应每 N 秒输出一行空行；`0` 禁用。 |
| `oauth.providers.aistudio.ws-auth` | boolean | `true` | `/v1/ws` 是否要求认证。 |

`requests.streaming.keepalive-seconds` 用来防止长 SSE 流被中间代理超时切断；如果你前面有 Nginx，默认 60 秒的 `proxy_read_timeout` 会把流切断，把 keepalive 设成 30 之类能撑过中间超时。

`requests.nonstream-keepalive-interval` 是个少见但有用的开关：客户端和上游都等待非流式响应时，每 N 秒塞一个空行过去，防止某些客户端的超时机制过早掐断。

### 路由与配额

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `routing.strategy` | string | `round-robin` | 凭据选择策略：`round-robin` / `fill-first`。新版 CPA 还支持 `weighted-round-robin`（见 B 部分）。 |
| `routing.session-affinity` | boolean | `false` | 把会话绑定到某条凭据。session id 来自 `metadata.user_id` / `X-Session-ID` / `conversation_id` 等。 |
| `routing.session-affinity-ttl` | string | `1h` | 会话到凭据绑定的 TTL。 |
| `quota-exceeded.switch-project` | boolean | `true` | 配额耗尽时自动切换项目。**v8 模板不再包含此字段**——只在旧文件和 `/v0/management` 里能读/写。 |
| `quota-exceeded.switch-preview-model` | boolean | `true` | 配额耗尽时自动切到预览模型。**v8 模板不再包含此字段**——只在旧文件和 `/v0/management` 里能读/写。 |
| `oauth.providers.antigravity.antigravity-credits` | boolean | `true` | Claude 的最后兜底：所有 free-tier 凭据耗尽（429/503）时，回退到带 Google One AI credits 的凭据。 |
| `oauth.providers.codex.identity-confuse` | boolean | `false` | 用 `fill-first` 或会话粘性时，按选中的凭据重映射 Codex 缓存和安装标识。 |

`round-robin` 让所有凭据均匀分摊请求，适合多账号等权重。`fill-first` 把请求塞满第一个凭据直到触发冷却，适合想集中用某个账号的额度。`weighted-round-robin`（B 部分会再讲）让你按权重配比，介于两者之间。

`routing.session-affinity` 解决的是「同一个对话上下文必须命中同一个凭据」的诉求——多轮对话里某些功能（缓存、文件 ID）跟上游账号绑定，强制换账号会丢上下文。开启后同一会话会持续落在首次选中的凭据上，但故障转移仍然有效。

### 提供商凭据：通用字段

每种 provider 的字段名虽然不同（`api-keys.gemini`、`api-keys.claude`、`api-keys.openai-compatibility` 等），但内部条目共享一套语义。先看这套通用字段，再看每个 provider 的差异。

| 字段 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `keys[].api-key` | string | `""` | 凭据值（API Key 文本或 OAuth 凭据）。 |
| `priority` | integer | `0` | 越大越优先。调度时先选最高 priority 里可用的凭据。 |
| `prefix` | string | `""` | 可选前缀。设置后，客户端必须用 `prefix/model` 才会命中这条凭据。 |
| `base-url` | string | 视 provider 而定 | 上游端点，覆盖默认值。留空就用默认 base URL。 |
| `headers` | object | `{}` | 附加请求头，比如 SaaS 网关要求 `X-Custom-Auth`。 |
| `proxy-url` | string | `""` | 本条凭据的代理覆盖。`direct`/`none` 绕过全局 `requests.proxy-url` 和环境代理。 |
| `disable-cooling` | boolean | `false` | 关掉本条凭据的冷却调度（一般别开）。 |
| `excluded-models` | string[] | `[]` | 要排除的模型，支持通配符。 |

模型映射放在每条凭据的 `models` 子字段里（v8 中 `api-keys.<provider>` 是分组列表，每组共享 `priority` / `prefix` / `models` 等字段，具体的 key 文本放在 `keys[]` 里）：

```yaml
api-keys:
  gemini:
    - keys:
        - api-key: ***
      priority: 10
      prefix: "g4"
      models:
        gemini-2.5-pro:
          alias: "pro"
          display-name: "Gemini Pro"
          force-mapping: true
```

- `name` 是上游真实的模型 ID；`alias` 是客户端看到的名字。客户端写 `alias` 就自动映射到 `name`。
- `force-mapping: true` 让上游响应里的 `model` 字段也写成 `alias`，客户端日志里看到的就是别名而不是上游 ID——多 provider 混用时日志更可读。
- `display-name` 是 CPAMP 模型目录里显示给人类看的标签，不影响调用。

### Gemini 和原生 Interactions

`api-keys.gemini[]` 和 `api-keys.interactions[]` 的字段结构完全相同，后者专用于直连 `/v1beta/interactions` 端点。`base-url` 默认 `https://generativelanguage.googleapis.com`，自建代理或第三方转发时改它。

### Codex 和 xAI

| 特有字段 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `base-url` | string | 必填 | 自定义端点；留空这条凭据会被丢弃。Codex 通常指 `https://api.openai.com` 或中转，xAI 指 `https://api.x.ai`。 |
| `websockets` | boolean | `false` | 用上游 Responses API 的 WebSocket 传输，某些场景比 HTTP 更稳。 |

### Claude（含 cloak 伪装）

Claude 这一组最复杂，因为存在「cloak 伪装」机制——把客户端请求伪装成 Claude Code 官方客户端发出的格式，绕过部分上游检查。

| 字段 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `rebuild-mid-system-message` | boolean | `false` | 把角色为 `system` 的消息挪到 Claude 的顶层 system 字段。 |
| `cloak.mode` | string | `auto` | `auto` = 非 Claude Code 客户端时伪装；`always` / `never` 强制开/关。 |
| `cloak.strict-mode` | boolean | `false` | 删除用户 system 消息，只保留 Claude Code 的提示——更严格的伪装。 |
| `cloak.sensitive-words` | string[] | `[]` | 用零宽字符混淆的词，绕开关键词审查。 |
| `cloak.cache-user-id` | boolean | `false` | 复用缓存的 `user_id`，让多次调用看起来像同一客户端。 |
| `experimental-cch-signing` | boolean | `false` | 用当前 Claude Code 的 CCH 算法给最终请求体签名。 |

> 这些字段位于 `oauth.providers.claude.cloak.*` 等位置（v8 把 Claude 段整段搬到了 `oauth.providers.claude` 下，原 `claude-api-key` / `claude` 顶层段都映射到这里），`cloak.*` 的具体子路径在 v8 文档正文中没有逐一列出，但映射表覆盖了整段。

Cloak 是上游风控和开发者体验之间的妥协：上游越来越认客户端指纹，所以 CPA 提供这套机制让请求看起来「像」官方客户端。开得过严（如 `oauth.providers.claude.cloak.strict-mode: true`）可能丢上下文；完全不开，部分账号会被限制。

### OpenAI 兼容提供商

`api-keys.openai-compatibility` 是最常用的兜底段——任何 OpenAI 格式的中转/自建服务都能挂进来（v8 把旧的顶层 `openai-compatibility` 段整段搬到了 `api-keys.openai-compatibility`，旧的 `api-key-entries` 数组改名 `keys`）：

```yaml
api-keys:
  openai-compatibility:
    - name: "my-relay"
      priority: 5
      base-url: "https://relay.example.com/v1"
      keys:
        - api-key: ***
      models:
        gpt-4o:
          alias: "gpt4o-relay"
          display-name: "GPT-4o (中转)"
          image: true
          input-modalities: ["text", "image"]
          output-modalities: ["text"]
          thinking:
            levels: ["low", "medium", "high"]
```

特有字段：

- `name`：面板里显示的提供商名。
- `disabled` / `disable-cooling`：临时关停或绕过冷却。
- `keys`：v8 里取代了旧的 `api-key-entries`，里面每条 `keys[].api-key` 是**调用上游中转**用的 key——和 v8 里另一个 `access.api-keys`（客户端访问 CPA 的 key）完全是两套东西，千万别混。
- `models.*.image`：允许该模型响应 `/v1/images/generations` 和 `/v1/images/edits`。
- `models.*.input-modalities` / `output-modalities`：声明能力，给 CPAMP 模型目录用。
- `models.*.thinking.levels`：声明该模型支持哪些推理强度档位（`low` / `medium` / `high`）。客户端用 `reasoning_effort` 之类的字段控制，CPAMP 会按你声明的档位暴露成下拉选项。

### Vertex 兼容 API Key

`api-keys.vertex[]` 字段集和 Gemini 大体一致，区别是 `base-url` 默认指向 Vertex 端点，常用来挂 Vertex 兼容的中转服务。

### OAuth 模型别名与排除

```yaml
oauth:
  model-alias:
    claude:
      "claude-sonnet-4.5":
        name: "claude-sonnet-4-5-20250929"
        alias: "cs45"
        fork: false
        force-mapping: true
        display-name: "Claude Sonnet 4.5"

  excluded-models:
    claude:
      - "*-preview-*"
```

- `name` / `alias`：和提供商段里的模型映射语义一致，但作用于 OAuth 渠道。
- `fork: true`：保留上游模型同时再额外暴露别名——客户端既能调原名也能调别名。
- `oauth.excluded-models.<channel>`：按渠道排除模型，支持通配符。不想暴露某个不稳定的预览模型时用它。

### 默认 Header

```yaml
oauth:
  providers:
    claude:
      header-defaults:
        user-agent: "claude-cli/1.0.0"
        os: "linux"
        arch: "x86_64"
        stabilize-device-profile: true

    codex:
      header-defaults:
        user-agent: "codex-cli/0.46.0"
        beta-features: "responses_websockets"
```

这些是 Claude/Codex OAuth 凭据调用时缺省的请求头。上游会校验客户端指纹是否稳定，所以**单台机器上建议固定**——`stabilize-device-profile: true` 帮每个凭据把 OS/arch 锁定成配置里的值，避免运行时动态推导导致指纹漂移。

### Payload 规则（高级）

CPA 支持四类规则改写请求/响应体（v8 把它们整体搬到了 `requests.payload` 下）：

- `requests.payload.default[]`：只写缺失字段，不覆盖已有。
- `requests.payload.override[]`：强制写入。
- `requests.payload.default-raw[]` / `requests.payload.override-raw[]`：同上，但 value 是原始 JSON。
- `requests.payload.filter[]`：删除指定 JSON 路径。

匹配条件挂在 `models[]` 上：

```yaml
requests:
  payload:
    override:
      - models:
          - name: "gpt-4o*"
            protocol: "openai"
            headers: {"user-agent": "*Codex*"}
        params:
          "temperature": 0.2
          "max_tokens": 4096
```

适用场景：上游非要某个固定 `temperature`、客户端传来被拒的字段、想批量注入 `safety_settings`。规则写错容易静默失效或污染请求，调试时先只放一条 `default` 验证命中，再叠加。

### 思考量（reasoning effort）

在 `api-keys.openai-compatibility[*].models[*].thinking.levels` 里声明模型支持的档位（默认 `["low", "medium", "high"]`）。然后客户端用 OpenAI/Anthropic 兼容的 reasoning 字段控制，CPA 把值转发到上游并尽量按上游支持的语义处理。

如果上游不返回 reasoning 字段（很多中转把 reasoning 内容裁掉了），即使客户端传了档位也不会有 thinking 内容出现——这时把 `levels` 留空或缩短可以减少无意义的协议协商。

### 插件与热重载

`plugins.*` 段是受信任的进程内动态插件：插件发现目录、官方/私有商店 URL、商店认证等。一般用户用不上；如果你 fork 了上游想加自定义中间件，从这里入手。

**热重载**：CPA 在多数字段上是热加载的——改 `config.yaml` 保存后，正在跑的进程会重新加载。但有几类变更需要重启：

- `server.host` / `server.port`（监听地址变了）
- `server.tls.*`（TLS 配置整体）
- 部分 `management.*` 字段

CPAMP 在保存后会提示是否需要重启运行时；判断标准就是看 CPA 是否完全采用了新值。

### 旧布局 → v8 对照表

上面各小节里所有 v8 字段名都是从帮助站文档正文的 v8 布局照搬过来的。帮助站「配置选项」页同时维护了一张「旧字段 → v8」的对照表（页面底部的「旧布局」节），下面是它的完整版：

| 旧字段 | v8 字段 |
|---|---|
| `host`、`port`、`trusted-proxies`、`tls`、`commercial-mode`、`discovery` | `server.*` |
| `remote-management` | `management` |
| 根级 `api-keys`（字符串列表，**客户端 key**） | `access.api-keys` |
| `credential-concurrency`、`credential-in-flight` | `credentials.concurrency`、`credentials.in-flight` |
| `force-model-prefix` | `routing.force-model-prefix` |
| `request-retry`、`max-retry-credentials`、`max-retry-interval` | `routing.retry.*` |
| `disable-cooling`、`save-cooldown-status`、`transient-error-cooldown-seconds` | `routing.cooldown.*` |
| `proxy-url`、`passthrough-headers`、`nonstream-keepalive-interval`、`streaming`、`payload` | `requests.*` |
| `auth-dir`、`auth-auto-refresh-workers` | `oauth.auth-dir`、`oauth.auth-auto-refresh-workers` |
| `oauth-model-alias`、`oauth-excluded-models`、`oauth-request-scoped-errors` | `oauth.model-alias`、`oauth.excluded-models`、`oauth.request-scoped-errors` |
| `ws-auth` | `oauth.providers.aistudio.ws-auth` |
| `codex`、`codex-header-defaults` | `oauth.providers.codex`，含 header-defaults |
| `claude`、`claude-code`、`disable-claude-cloak-mode`、`claude-header-defaults` | `oauth.providers.claude` |
| `antigravity`、`antigravity-signature-cache-enabled`、`antigravity-signature-bypass-strict` | `oauth.providers.antigravity` |
| `quota-exceeded.antigravity-credits` | `oauth.providers.antigravity.antigravity-credits` |
| `xai`、`devin` | `oauth.providers.xai`、`oauth.providers.devin` |
| `disable-image-generation`、`gpt-image-2-base-model`、`video-result-auth-cache-ttl` | `multimedia.*` |
| `debug`、`logging-to-file`、`logs-max-total-size-mb`、`error-logs-max-files`、`request-log` | `observability.logs.*` |
| `usage-statistics-enabled`、`redis-usage-queue-retention-seconds` | `observability.usage.*` |
| `pprof` | `observability.pprof` |
| `gemini-api-key`、`interactions-api-key`、`vertex-api-key`、`codex-api-key`、`claude-api-key`、`xai-api-key`、`meta-api-key`（带 keys 列表的） | `api-keys.<provider>` 分组 |
| `openai-compatibility`、`api-key-entries` | `api-keys.openai-compatibility`、`keys` |

旧字段之外，v8 还强制要求一个新字段：

| 新字段 | 说明 |
|---|---|
| `config-version: 8` | v8 文件的根级标识，存在时必须为 8。**只设这一行不会自动迁移旧字段**——加载和保存是两件事。 |

和「客户端 key」相关的两个键在 v8 里被拆成了不同含义，复制旧配置时务必看清楚：

| 旧用法 | v8 含义 |
|---|---|
| 根级 `api-keys: ["sk-client-..."]`（字符串列表） | **客户端 key** → `access.api-keys` |
| `api-keys: <provider>:` 这样的分组结构 | **上游提供商 key** → `api-keys.<provider>` 分组 |

帮助站文档原文里强调的三个关键行为，迁移和升级时一定先看清楚：

1. **旧配置仍可用**——旧字段名的文件和 `/v0/management` 写入请求继续被接受，不会立刻报错。混用文件中如果想重新使用旧字段，先把对应的 v8 字段删除。
2. **同一设置同时存在 v8 和旧写法时，v8 胜出**——包括 `false`、`0` 和空集合（这是文档原话，意思是 v8 设的"显式空"也会盖掉旧字段的"非空值"）。加载或保存时，v8 字段会把对应的旧字段一并删掉。
3. **v8 管理写入会拒绝旧字段名和未知的根配置节**——也就是说，CPAMP 通过 `/v8/management` 写入配置时会校验字段路径，旧字段名直接报错。这一点和第 1 条的「旧配置仍可用」是分两层：读和写旧字段都没问题，但**主动往 v8 schema 里写旧字段不行**。

文档原文还提到两条相关规则，理解它们能避免一些踩坑：

- **只设置 `config-version: 8`，或只读取配置，都不会自动迁移文件**——CPA 不会在你加上一行 `config-version: 8` 之后帮你把旧字段重写成 v8。需要迁移时，必须通过 `/v8/management` 写入一次完整的新 schema，旧字段才会被规范化掉。
- **仅存在于旧布局中的字段会保留到一次成功的 `/v8/management` 写入为止**——在那之前，文件里的旧字段（包括 `quota-exceeded.switch-project` / `quota-exceeded.switch-preview-model` 这些 v8 已经不带、但旧文件里仍能读写的字段）会被原样保留；写入一次 v8 schema 之后，旧字段要么被规范化、要么（如果 v8 不再带该字段）就被丢弃。

## B. CPA Manager Plus 配置中心

CPA Manager Plus（CPAMP）本身不处理模型请求，它只是 CPA 的远程配置和监控 UI。本节只讲它的「配置中心」页面，其他页面（AI 提供商、OAuth 登录、凭证管理、请求监控、用量分析）属于另外的入口，不在本文范围。

### 三种编辑方式

配置中心通常并列三类入口，按使用频率排序：

1. **可视化配置**：把 `config.yaml` 按功能分组，做成表单。字段少、有提示，适合日常调整。
2. **源文件配置**：直接编辑 `config.yaml` 全文。改动精确，但容易写错格式（缩进、引号、数组语法）。
3. **Manager Server 配置**：只在 **完整模式** 下出现，用于让 CPAMP 自身连上 CPA、开启请求监控、存 CPAMP 自己的运行设置。轻量面板下没有这一段。

如果某个 Tab 不可见，先确认安装模式——Docker 或原生完整包才有 Manager Server 能力；轻量面板是「只读 + 少量编辑」。

### Manager Server 配置（完整模式）

绝大多数情况下只需要确认三个字段：

| 字段 | 含义 |
|---|---|
| **CPA 地址** | CPAMP 访问 CPA 的地址。容器部署时常是 Docker 网络里的服务名（如 `http://cpa:8317`），不一定是浏览器访问 CPA 的地址。 |
| **CPA Management Key** | CPAMP 调 CPA 管理 API 的密钥，对应 `management.secret-key`（旧字段名 `remote-management.secret-key` 仍可用）。**不是**客户端调模型用的 API Key（即不是 `access.api-keys`）。 |
| **请求监控** | 开启后会启用 CPA 用量统计，并启动 CPAMP 自带的采集器。 |

保存后回到仪表盘确认 CPA 已连接，再发一条真实请求看请求监控里有没有事件——三步缺一不可。

高级采集设置一般保持默认即可：

- **采集模式**：自动 / RESP / HTTP。RESP 模式要求直连 CPA API 端口，不能走普通 HTTP 反代。
- **轮询间隔 / 批量大小 / 查询上限**：影响延迟和单次读取量。**轮询间隔不要超过 CPA 的队列保留时间**（对应 `observability.usage.redis-usage-queue-retention-seconds`，默认 60 秒）。
- **配置来源**：环境变量 / SQLite / 未保存。环境变量优先级最高——面板保存改不了它，要改部署环境并重启。

### 可视化 vs 源文件：什么时候用哪个

| 场景 | 推荐 |
|---|---|
| 改一两个常用字段（端口、API Key） | 可视化 |
| 一次性调整多段（如批量加多个 provider 入口） | 源文件 |
| 复制现有 `config.yaml` 到另一台机器 | 源文件 |
| 检查可视化没暴露的字段（如 `requests.payload.*`、`plugins.*`） | 源文件 |
| 跟生产配置做 diff 比对 | 源文件 |

源文件视图看起来像直接编辑 YAML，但它的提交逻辑是「先在面板里暂存，再发到 CPA 的 `/v0/management` 写入」。所以：

- 保存前一定要确认缩进、数组、字符串引号。
- 保存后如果页面没刷新，看 CPA 是否支持热重载，或需要重启运行时。

### 加权轮询路由

需要「同优先级里多份凭据按比例承接请求」时，在可视化配置里把路由策略设成「加权轮询」，对应源文件：

```yaml
routing:
  strategy: weighted-round-robin
```

CPA 会先筛出当前最高且可用的 `priority` 层级，再在那一层里按凭据 `weight` 做加权轮询。规则：

- 未设 `weight` 默认是 `1`。
- 权重必须整数，范围 `1` 到 `1,000,000`。
- 非正数（≤ 0）会让该凭据在加权策略下不参与分配——这是个有用的「暂时下线」开关。
- 在 CPAMP 里把权重输入框清空，会移除显式 `weight`，恢复默认 `1`。

注意：加权策略需要 CPA 本身支持这个能力。**较老版本的 CPA 会忽略或拒绝**这个字段；先确认你的 CPA 版本，再去「AI 提供商」或「凭证管理」给每条凭据设权重。验证比例时用多个独立请求——启用了「会话亲和性」后，同一会话可能持续绑定既有凭据，统计上看起来权重失灵。

### 保存后怎么验证

1. 回仪表盘确认连接和采集器状态。
2. 进「AI 提供商」确认提供商配置还在（有时保存会触发 schema 校验，过期的 provider 配置可能被清掉）。
3. 发一条低成本请求（短 prompt）。
4. 到「请求监控」看是否出现该请求的事件。
5. 改动涉及用量统计的话，去「用量分析」看统计是否更新。

### 常见误区

- **四类 key 容易混**：CPAMP 管理员密钥、CPA Management Key、客户端 API Key、Provider API Key，是四套不同的字符串，分属四个层面。
- **关 CPAMP 采集器 ≠ 清空 CPA 用量队列**：CPA 侧有自己的保留时间，关采集只会让 CPAMP 不再展示新数据。
- **模型价格只影响成本估算**：CPAMP 里的模型价格字段只用来显示预估成本，不会改变提供商实际计费。
- **环境变量优先**：用环境变量给的配置项，面板保存改不了。要改就去部署环境（Docker Compose / systemd unit）并重启。

## 参考

- CLIProxyAPI 配置选项（中文）：<https://help.router-for.me/cn/configuration/options.html>
- CPA Manager Plus 配置中心（中文）：<https://seakee.github.io/CPA-Manager-Plus/docs/manual/configuration.html>
