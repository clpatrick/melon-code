# Melon Code

**A self-hosted, Claude-Code-class coding agent that runs on a local model, built to learn exactly how frontier coding agents work and to make that loop survive on hardware you own.**

Status: in daily use as a local coding agent since January 2026. Source is private; this repository describes the system, the measurements, and what was learned. Successor to MelonStudio after the pivot from ONNX Runtime to llama.cpp.

## Why it exists

Cloud coding agents are excellent, expensive, and send your code off-box. I wanted the same agent loop, with tools, permissions, subagents, task lists, compaction, and rewind, running against a model on my own workstation, and I wanted to study how Claude Code actually behaves by pointing it at a server I control. Local models also surface problems cloud users never see: a real context window far smaller than the configured one, missing structured tool calls when the chat template is off, reasoning tags leaking into history, and long sessions that overflow or truncate mid-JSON. Melon Code is the harness built to absorb all of that.

## What it does

- **Providers:** local llama.cpp server (OpenAI-compatible), NVIDIA NIM (build.nvidia.com) for hosted open models, Anthropic, and OpenAI, with OAuth device flows for Claude, ChatGPT, and GitHub Copilot studied and implemented.
- **Anthropic Messages API endpoint** (`/v1/messages`, `count_tokens`) in proxy or translate mode, following Claude Code's published gateway spec. Point `ANTHROPIC_BASE_URL` at Melon Code and Claude Code runs on a local model. Traffic capture (with secrets redacted and bodies off by default) turned the closed loop into something I could read.
- **Tools without an MCP server:** bash over a persistent PTY, read, write, edit, text editor, grep, glob, web fetch with SSRF guards, web search. External MCP tools attach through the official SDK.
- **Agent features:** a permission engine with four modes and per-tool approval, diff preview before any write, subagents with their own context, a tool-loop guard, background bash, todo tracking, slash commands, message queueing.
- **Context engineering:** a token-aware context manager, compaction that reserves output room so replies never truncate mid-sentence, a per-session repo map (tree-sitter, under 5k tokens) snapshotted so the KV-cache prefix stays stable, checkpoints and rewind that restore warm from the server's prompt cache.
- **Observability:** per-turn model, token counts, time, and tokens per second in the UI; prefill/decode timings and an audit log on disk.

## Architecture

```
React UI  ->  REST / WebSocket  ->  AgentBridge  ->  Session  ->  Agent  ->  Provider  ->  stream
                                                        down
                                        tools (local) - MCP clients - permission engine - safeguard
```

TypeScript on Node 22 (Express 5, ws, node-pty, web-tree-sitter, `@modelcontextprotocol/sdk`); React 19, Vite 7, Zustand 5, Tailwind 4. Inference local by default: in July 2026 the daily model was DeepSeek V4 Flash (284B mixture-of-experts, 3-bit quantized) on one RTX 4090 with 127 GB of RAM; earlier GPT-OSS-120B and Qwen3.5-35B. Cloud providers reachable only with keys.

## Security posture

- Secrets live in an ignored env file and a user-profile credential store, never in settings.
- A safeguard module blocks protected paths and extensions, enforces a command allowlist, checks path boundaries, and validates shell metacharacters per segment.
- Permission engine with four modes; every file write previews a diff first.
- Web fetch and search carry SSRF guards; captured Claude Code traffic always redacts API keys and authorization headers.
- Audit trail of tool calls and turns on disk; session access control and forking.

## Measurements

- **KV-cache prefix stability:** a warm turn dropped from 5,621 prompt tokens and 36 seconds to 18 tokens and 1.3 seconds once the repo map was snapshotted per session and the prompt cache used.
- **Warm rewind:** 1,612 tokens re-processed became 4, served from the server's RAM prompt cache, so disk KV slot saves were made opt-in rather than default. A decision recorded with its evidence.
- **Local reality that shaped the design:** about 9 tokens per second decode and 44-90 prefill on the 284B model, so the goal became minimize generated tokens and maximize prefix reuse.
- **Test suite:** sixteen test files across registry, auth, permissions, session forking, agent abort, compaction, loop guard, checkpoints, provider retry, subagents, and tools; 100 tests green on the July 2026 parity branch (121 files, +12,480 / -4,066 lines).
- Produced a patched llama.cpp server fixing a slot-restore checkpoint gap, later judged unnecessary once the RAM cache path was measured.

## How it was built

Product definition was mine: plan, act, and ask modes; the telemetry footer; the repo-map budget; artifact behavior. The July 2026 parity overhaul ran parallel worker agents with disjoint file ownership, an adversarial review after each security-relevant wave, and a full gate between waves; merge waited on my go-ahead. Code review ran both ways: I asked the agents to verify my fixes, and I returned severity-ordered findings on theirs. Thirteen plan documents and an archive of thirteen more record the decisions, including a codebase audit that tabulated failures with root causes.

## Lineage

Started as an MCP tool server and supervisor/sub-agent client for a local GPT-OSS-120B (January 2026): sandboxed file I/O with drive allowlists and size caps, rate-limited web search with provider fallback, and a supervisor that delegates to a sub-agent with its own context and must synthesize its report rather than paste it. That prototype was renamed Melon Code on January 17, 2026.
