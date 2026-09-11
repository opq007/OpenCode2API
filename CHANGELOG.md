# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **Agent Loop Continuity Fixes** (root-cause fixes for "the agent replies once and stops" when driving coding agents through the proxy):
  - **Tool-intent repair (auto mode)**: when the model mentions a tool in prose but emits no parseable markup, the proxy re-prompts once on the same session so the intent becomes a real `tool_calls` response instead of degrading to `finish_reason: 'stop'` (which made the calling agent end the task). Enabled by default; disable with `OPENCODE_PROXY_TOOL_INTENT_REPAIR=false`.
  - **Concurrency**: replaced the global FIFO mutex (concurrency = 1) that serialized every `/v1/chat/completions` request with a configurable semaphore (`OPENCODE_PROXY_MAX_CONCURRENCY`, default `4`). Parallel requests from coding agents no longer queue into "Request timeout" rejections.
  - **Per-request model**: removed the global `config.update(activeModel)` writes; the model now travels on each `session.prompt` body, so concurrent requests cannot run on the wrong model.
- **Backend Isolation ("client is the agent")**: The spawned opencode backend is now sandboxed by default (`OPENCODE_ISOLATION=keep-auth`). Operator-local prompts, skills, agents, plugins, MCP servers and `AGENTS.md` no longer leak into client requests. Provider/model definitions are carried over and the real `auth.json` is copied so locally logged-in models keep working. Inline credentials in jail config are stripped (`full` skips `auth.json`; `none` uses the real home). Windows is no longer exempt. Defense-in-depth: `HOME`/`USERPROFILE`/`XDG_*` redirects, `OPENCODE_CONFIG_DIR`/`OPENCODE_CONFIG_CONTENT` pinning, and `opencode serve --pure`. Legacy `USE_ISOLATED_HOME` still maps `true`→`full` / `false`→`none`. Hidden OpenCode `title`/`summary` agents are disabled in the jail so one client request does not trigger a second upstream LLM call for session titles.

### Fixed

- **Tool calls silently dropped by policy**: `require_confirmation` policy decisions (inferred for every `write`/`update`/`create`/`set`/`post`/`send`-named tool, e.g. a coding agent's `Write`) passed through to the client instead of being blocked. The calling agent is the confirmation surface; blocking here turned a valid tool call into `finish_reason: 'stop'` and ended the agent's loop mid-task. Explicit denylist blocking is unchanged.
- **Invalid tool calls dropped silently**: schema-invalid tool calls now pass through as best-effort `tool_calls` so the client's tool runner reports the error back into the conversation and the model self-corrects next turn, instead of the reply degrading to `stop`.
- **Backend Health Check**: The opencode backend health probe now uses the correct `/global/health` endpoint (the old `/health` path fell through to the web UI catch-all and could report 200 HTML as healthy). The probe now validates the JSON `healthy` field.
- **Backend Auto-Start**: `OPENCODE_PROXY_MANAGE_BACKEND` now defaults to `true`, so `npm start` auto-spawns `opencode serve` when the backend is unreachable. The spawn no longer passes the non-existent `--password` CLI flag; the server password is injected via the `OPENCODE_SERVER_PASSWORD` env var and the upstream provider key via `OPENCODE_API_KEY`.
- **Model Name Splitting**: Model references are now split on the first `/` only, so model names containing multiple `/` (e.g. `new-api/deepseek/deepseek-v4-flash`) resolve without truncation.

## [1.5.0] - 2026-04-18

### Added

- **External Tool Bridge**: Added proxy-level bridging for external OpenAI-compatible `tools` across `/v1/chat/completions` and `/v1/responses`.
- **Streaming Tool Call Parity**: Added streaming support for external tool calls in both Chat Completions and Responses APIs.
- **Explicit External Tool Config**: Added explicit `EXTERNAL_TOOLS_MODE=proxy-bridge` and `EXTERNAL_TOOLS_CONFLICT_POLICY=namespace` configuration surface and documentation.

### Changed

- **Project Version**: Bumped the repository version to `1.5.0` across package metadata and documentation badges.

### Fixed

- **Jest Test Shutdown**: Removed a lingering queue rescheduling timer from the proxy request lock flow and updated the default test command to use the verified clean Jest invocation, eliminating the previous generic open-handle warning during `npm test`.

## [1.0.0] - 2025-04-11

### Added

- **OpenAI-compatible API**: `/v1/models`, `/v1/chat/completions`, `/v1/responses` endpoints
- **Streaming Support**: Full SSE streaming for Chat Completions and Responses API
- **Model Aliases**: GPT-style model aliasing (e.g., `gpt5-nano` → `gpt-5-nano`)
- **Docker Deployment**: Complete Docker setup with healthcheck and volume management
- **Configuration**: Environment variables and config.json support
- **Auto Cleanup**: Configurable automatic conversation/session storage cleanup

### Changed

- **Default Security**: `DISABLE_TOOLS` defaults to `true` for safer out-of-box behavior
