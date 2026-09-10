# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Fixed

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
