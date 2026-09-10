# ⚙️ 配置详解

<p align="center">
  <img src="https://img.shields.io/badge/version-1.5.0-blue" alt="Version">
</p>

---

## 📌 配置方式

> 配置优先级：**环境变量 > config.json > 默认值**

---

## 🔧 环境变量

### 核心配置

| 变量 | 默认值 | 说明 |
|:-----|:-------|:-----|
| `PORT` / `OPENCODE_PROXY_PORT` | `10000` | 代理服务端口 |
| `OPENCODE_SERVER_PORT` | `10001` | OpenCode 后端服务端口 |
| `API_KEY` | - | Bearer Token 认证密钥 |
| `BIND_HOST` | `0.0.0.0` | 绑定地址 |
| `OPENCODE_SERVER_URL` | `http://127.0.0.1:10001` | OpenCode 后端地址 |
| `OPENCODE_SERVER_PASSWORD` | - | OpenCode 后端密码 |

### 功能配置

| 变量 | 默认值 | 说明 |
|:-----|:-------|:-----|
| `DISABLE_TOOLS` | `true` | 禁用 OpenCode 工具调用 |
| `OPENCODE_EXTERNAL_TOOLS_MODE` | `proxy-bridge` | 外部工具桥接模式；当前仅支持 `proxy-bridge` |
| `OPENCODE_EXTERNAL_TOOLS_CONFLICT_POLICY` | `namespace` | 外部工具冲突隔离策略；当前仅支持 `namespace` |
| `OPENCODE_INTERNAL_WEB_FETCH_ENABLED` | `false` | 兼容旧开关；未显式配置 allowlist 时，启用后默认放行 `web_fetch` |
| `OPENCODE_INTERNAL_ALLOWED_TOOLS` | `(none)` | 当请求未传入 `tools` 时允许使用的 OpenCode 内置工具列表，逗号分隔 |
| `OPENCODE_INTERNAL_TOOL_METRICS_ENABLED` | `true` | 输出 internal allowlist 模式的调试/指标日志 |
| `OPENCODE_TOOL_DISCOVERY_FIXTURE` | `(none)` | 集成测试/本地调试用的固定后端工具 ID 列表，逗号分隔 |
| `OPENCODE_HEALTH_DETAILS_ENABLED` | `true` | 控制 `/health/details` 是否暴露 |
| `OPENCODE_HEALTH_DETAILS_REQUIRE_AUTH` | `true` | 控制 `/health/details` 是否要求 Bearer 认证 |
| `OPENCODE_METRICS_ENABLED` | `false` | 控制 Prometheus `/metrics` 是否暴露 |
| `OPENCODE_METRICS_REQUIRE_AUTH` | `true` | 控制 `/metrics` 是否要求 Bearer 认证 |
| `OPENCODE_ISOLATION` | `keep-auth` | 后端沙箱隔离级别：`keep-auth` / `full` / `none`（见下文「后端隔离」） |
| `OPENCODE_JAIL_INLINE_KEYS` | `false` | 为 `true` 时在 jail 配置中保留 provider 内联 `apiKey`（仅建议单用户主机开启） |
| `USE_ISOLATED_HOME` | `(废弃)` | 旧布尔开关；`true`→`full`，`false`→`none`。已被 `OPENCODE_ISOLATION` 取代 |
| `PROMPT_MODE` | `standard` | 提示词处理模式 |
| `OMIT_SYSTEM_PROMPT` | `false` | 忽略传入的 system prompt |
| `AUTO_CLEANUP_CONVERSATIONS` | `false` | 自动清理会话存储 |
| `CLEANUP_INTERVAL_MS` | `43200000` | 清理间隔 (毫秒) |
| `CLEANUP_MAX_AGE_MS` | `86400000` | 最大存储时间 (毫秒) |
| `REQUEST_TIMEOUT_MS` | `180000` | 请求超时时间 (毫秒) |

### 调试配置

| 变量 | 默认值 | 说明 |
|:-----|:-------|:-----|
| `DEBUG` / `OPENCODE_PROXY_DEBUG` | `false` | 开启调试日志 |
| `OPENCODE_PATH` | `opencode` | OpenCode 可执行文件路径 |
| `OPENCODE_ZEN_API_KEY` | - | Zen API Key 透传 |

---

## 📄 config.json 示例

```json
{
    "PORT": 10000,
    "API_KEY": "your-secret-api-key",
    "BIND_HOST": "0.0.0.0",
    "DISABLE_TOOLS": true,
    "EXTERNAL_TOOLS_MODE": "proxy-bridge",
    "EXTERNAL_TOOLS_CONFLICT_POLICY": "namespace",
    "INTERNAL_WEB_FETCH_ENABLED": false,
    "INTERNAL_ALLOWED_TOOLS": ["web_fetch"],
    "INTERNAL_TOOL_METRICS_ENABLED": true,
    "INTERNAL_TOOL_DISCOVERY_FIXTURE": [],
    "ISOLATION": "keep-auth",
    "JAIL_INLINE_KEYS": false,
    "USE_ISOLATED_HOME": false,
    "PROMPT_MODE": "standard",
    "OMIT_SYSTEM_PROMPT": false,
    "AUTO_CLEANUP_CONVERSATIONS": false,
    "CLEANUP_INTERVAL_MS": 43200000,
    "CLEANUP_MAX_AGE_MS": 86400000,
    "DEBUG": false,
    "OPENCODE_SERVER_URL": "http://127.0.0.1:10001",
    "OPENCODE_PATH": "opencode",
    "REQUEST_TIMEOUT_MS": 180000
}
```

---

## 🛡️ 后端隔离（"客户端即 Agent"）

OpenCode2API 的使用模型是：**后端 = 纯净模型路由器，客户端 = Agent**。工具执行在客户端侧完成（OpenAI 标准 tool calling 循环），因此后端不应携带任何本地 opencode 环境的"人格"（提示词、skills、agents、modes、commands、plugins、MCP servers、AGENTS.md）。

默认（`OPENCODE_ISOLATION=keep-auth`）下，代理每次自动启动后端时都会：

1. **重定向全部配置路径**进每次启动新建的 jail 目录（`HOME` / `USERPROFILE`、`XDG_CONFIG_HOME`、`XDG_DATA_HOME`、`XDG_CACHE_HOME`），并让 `opencode serve` 在空工作目录中运行。操作者本地 `~/.config/opencode/` 下的 skills/agents/提示词/plugins 一个都不会被加载（已用 `opencode debug config` 实测确认）。
2. **写入锁定配置** `opencode.json`：`instructions: []`、`autoupdate: false`、`snapshot: false`，并从真实全局配置中**只提取模型访问相关的白名单**（`provider` / `model` / `small_model` / `disabled_providers` / `enabled_providers`）——其余键（指令、agent、MCP、插件、主题……）全部丢弃。
3. **剥离内联凭据**：jail 配置文件中的 provider `apiKey`/`token`/`secret` 等字段会被移除（防止临时目录中的配置泄露密钥）。凭据默认通过复制本机 `auth.json`（`keep-auth`）和环境变量透传获得。
4. **纵深防御**：注入 `OPENCODE_CONFIG_DIR`（指向 jail 内空目录）与 `OPENCODE_CONFIG_CONTENT={"instructions":[]}`，确保即使有其他配置源被合并进来，指令也是空的。
5. **`--pure` 参数**：`opencode serve --pure` 禁用外部插件。

| 级别 | 说明 |
|:-----|:-----|
| `keep-auth` | 完全沙箱 + 复制真实 `~/.local/share/opencode/auth.json` 进 jail（默认。用于转发本机 `/connect` 已登录的模型）。 |
| `full` | 完全沙箱。仅继承 provider/model 配置，凭据只靠环境变量。 |
| `none` | 使用真实用户主目录（仅调试/排障用）。 |

> 旧开关 `USE_ISOLATED_HOME=true/false` 仍然识别，分别等价于 `full` / `none`。新配置 `OPENCODE_ISOLATION` 优先。
>
> 单用户主机如果必须保留 provider 内联 `apiKey`，可设 `OPENCODE_JAIL_INLINE_KEYS=true`。

## 🛠️ 外部工具桥接

OpenCode2API 现在支持把外部客户端传入的 OpenAI-compatible `tools` 桥接到代理层，而不是把这些工具直接暴露为 OpenCode 内置工具。

### 当前支持的模式

| 配置项 | 支持值 | 说明 |
|:------|:------|:-----|
| `OPENCODE_EXTERNAL_TOOLS_MODE` / `EXTERNAL_TOOLS_MODE` | `proxy-bridge` | 由代理虚拟化外部工具，并返回 OpenAI-compatible tool calling 结果 |
| `OPENCODE_EXTERNAL_TOOLS_CONFLICT_POLICY` / `EXTERNAL_TOOLS_CONFLICT_POLICY` | `namespace` | 使用代理内部命名空间隔离同名冲突 |

### 工具冲突策略

- 外部客户端工具优先以“代理桥接”的方式参与对话。
- OpenCode 内置工具仍按现有 `DISABLE_TOOLS` 机制管理，不会因为客户端传入同名工具而被误触发。
- 代理内部会使用类似 `external__web_fetch` 的命名空间名避免冲突。
- 这些内部命名空间名称不会作为公开 API 的一部分暴露给客户端。

### 内置工具 allowlist

- 当请求 **未传入** `tools` 时，代理会进入 internal allowlist 模式，只允许 `OPENCODE_INTERNAL_ALLOWED_TOOLS` 中声明的 OpenCode 内置工具。
- `OPENCODE_INTERNAL_WEB_FETCH_ENABLED=true` 仅用于兼容旧配置：如果未显式配置 allowlist，则默认把 allowlist 视为 `web_fetch`。
- 代理会读取后端工具列表，并通过精确匹配或 `.<tool>` / `/<tool>` 后缀匹配解析最终可用工具。
- 如果配置的 allowlist 在后端工具列表中一个也没有匹配到，代理会自动回退到“全部内置工具禁用”的安全模式。
- `OPENCODE_INTERNAL_TOOL_METRICS_ENABLED=true` 时，会输出 internal allowlist 模式的调试/指标日志，记录模式选择、后端工具发现、allowlist 命中情况和降级原因，但不会记录工具输出内容。
- `OPENCODE_TOOL_DISCOVERY_FIXTURE` 可在集成测试或本地调试时绕过真实 `client.tool.ids()`，直接提供固定工具 ID 列表。
- 一旦客户端传入 `tools`，请求立即切回外部工具桥接模式，所有 OpenCode 内置工具继续保持禁用。

### 请求级 allowlist 覆盖 (Request-Level Override)

在请求未传入 `tools` 的前提下，客户端可以在请求体中传入自定义字段 `opencode.internal_allowed_tools` 来覆盖服务端的默认内置工具列表。
出于安全隔离原则，请求级覆盖**只能缩小（求交集），不能扩大**服务端的 allowlist 权限：
- 如果请求了服务端未开启的内置工具，该工具会被自动忽略。
- `effective_allowlist = intersection(server_allowlist, request_allowlist)`

**示例：**
```json
{
  "model": "opencode/kimi-k2.5",
  "messages": [{"role": "user", "content": "Fetch this URL"}],
  "opencode": {
    "internal_allowed_tools": ["web_fetch"]
  }
}
```

### 结构化健康诊断接口

可以通过 `GET /health/details` 接口获取代理内部的运行状态与指标。这不仅有助于问题排查，也是用于编写集成行为测试的重要依据。
- `OPENCODE_HEALTH_DETAILS_ENABLED=false` 时，接口返回 `404`。
- `OPENCODE_HEALTH_DETAILS_REQUIRE_AUTH=true` 时，接口要求 Bearer 认证；否则返回 `401`。
返回值格式如下：
```json
{
  "status": "ok",
  "proxy": true,
  "internal_tools": {
    "config": {
      "allowed_tools": ["web_fetch", "filesystem"],
      "metrics_enabled": true,
      "discovery_fixture": ["web_fetch", "filesystem", "bash"]
    },
    "metrics": {
      "externalBridgeRequests": 12,
      "internalAllowlistRequests": 8,
      "disabledRequests": 21,
      "discoveryFailures": 1,
      "fallbackToDisabled": 2
    },
    "cache": {
      "tool_ids_cached": true,
      "tool_id_count": 3,
      "age_ms": 12000
    },
    "audit": {
      "available": true,
      "fields": [
        "requestedAllowlist",
        "allowedToolNames",
        "deniedRequestedTools",
        "resolutionPath",
        "resultingMode"
      ]
    }
  }
}
```

### Prometheus 指标接口

可以通过 `GET /metrics` 获取 Prometheus 文本格式指标。
- `OPENCODE_METRICS_ENABLED=false` 时，接口返回 `404`。
- `OPENCODE_METRICS_REQUIRE_AUTH=true` 时，接口要求 Bearer 认证；否则返回 `401`。
当前暴露的核心指标包括：
- `opencode_internal_tool_mode_requests_total{mode="external_bridge"}`
- `opencode_internal_tool_mode_requests_total{mode="internal_allowlist"}`
- `opencode_internal_tool_mode_requests_total{mode="disabled"}`
- `opencode_internal_tool_discovery_failures_total`
- `opencode_internal_tool_fallback_disabled_total`
- `opencode_internal_tool_cache_ids`

### 推荐生产配置

```bash
DISABLE_TOOLS=true
OPENCODE_EXTERNAL_TOOLS_MODE=proxy-bridge
OPENCODE_EXTERNAL_TOOLS_CONFLICT_POLICY=namespace
OPENCODE_INTERNAL_ALLOWED_TOOLS=web_fetch
OPENCODE_INTERNAL_TOOL_METRICS_ENABLED=true
OPENCODE_TOOL_DISCOVERY_FIXTURE=
OPENCODE_HEALTH_DETAILS_ENABLED=true
OPENCODE_HEALTH_DETAILS_REQUIRE_AUTH=true
OPENCODE_METRICS_ENABLED=false
OPENCODE_METRICS_REQUIRE_AUTH=true
OPENCODE_PROXY_PROMPT_MODE=plugin-inject
OPENCODE_PROXY_OMIT_SYSTEM_PROMPT=true
OPENCODE_PROXY_AUTO_CLEANUP_CONVERSATIONS=true
```


---

## 🎯 Prompt Mode 说明

| 模式 | 说明 |
|:-----|:-----|
| **standard** (默认) | 标准模式，完整处理提示词 |
| **plugin-inject** | 插件注入模式，减小模型侧提示词大小，通常与 `OMIT_SYSTEM_PROMPT=true` 配合使用 |

---

## ⭐ 推荐配置

### 🐳 Docker 生产环境

```bash
DISABLE_TOOLS=true
OPENCODE_EXTERNAL_TOOLS_MODE=proxy-bridge
OPENCODE_EXTERNAL_TOOLS_CONFLICT_POLICY=namespace
OPENCODE_INTERNAL_ALLOWED_TOOLS=web_fetch
OPENCODE_INTERNAL_TOOL_METRICS_ENABLED=true
OPENCODE_TOOL_DISCOVERY_FIXTURE=
OPENCODE_HEALTH_DETAILS_ENABLED=true
OPENCODE_HEALTH_DETAILS_REQUIRE_AUTH=true
OPENCODE_METRICS_ENABLED=false
OPENCODE_METRICS_REQUIRE_AUTH=true
OPENCODE_PROXY_PROMPT_MODE=plugin-inject
OPENCODE_PROXY_OMIT_SYSTEM_PROMPT=true
OPENCODE_PROXY_AUTO_CLEANUP_CONVERSATIONS=true
```


### 💻 本地开发

```bash
DISABLE_TOOLS=false
OPENCODE_PROXY_DEBUG=true
```
