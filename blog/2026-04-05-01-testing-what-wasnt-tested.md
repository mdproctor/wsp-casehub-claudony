# RemoteCC — Testing What Wasn't Being Tested

**Date:** 2026-04-05
**Type:** phase-update

---

## What I was trying to achieve: proving the end-to-end chain actually works

The system existed and functioned — 64 tests passing, native binary building in 70ms, MCP server exposing 8 tools. But the whole point of RemoteCC is a controller Claude managing sessions through conversation. That flow had never been exercised with a real Claude instance. Not once.

I wanted it rock solid before calling it done. Unit tests, integration tests, and something that actually ran `claude` CLI processes against the MCP server.

## What we believed going in: 64 tests meant the gaps were small

I brought Claude in to work through the gaps systematically. We expected the interesting part to be the end-to-end validation — getting real Claude connected to the MCP server — and the other gaps to be minor fill-in work.

We were half right.

## A bug that had been hiding since day one, and 25 tests that hadn't existed

Before any new tests, we found something unexpected: `TmuxService.sendKeys()` was broken.

The method the REST input endpoint uses to forward text to tmux didn't pass the `-l` (literal) flag. Without it, tmux interprets certain words — `Escape`, `Enter`, `Tab` — as key names rather than literal text. Sending `"Escape"` via the REST API would fire the actual Escape key. The fix was one line, but the behaviour had been there since the start and all 64 tests had missed it because none of them sent text that matched a tmux key name exactly.

The subagent writing the failing test first tried `"echo Escape marker"` as the input and got confused when the test didn't expose the bug. Claude came back and explained: tmux's key lookup is per-argument, not per-word within an argument. `"echo Escape marker"` is one argument and doesn't match any key name. `"Escape"` alone does. The test was adjusted.

We then added 25 tests across the full stack: rename and resize endpoints with zero prior coverage, McpServer unit tests for all 8 tools and error paths, integration tests covering the full MCP handshake sequence as Claude actually sends it (`initialize` → `notifications/initialized` → `tools/list`), WebSocket history replay and concurrent connection tests, and a bootstrap test proving `ServerStartup.bootstrapRegistry()` picks up tmux sessions created after startup.

### The MCP config format that wasn't what I expected

For the E2E tests, the approach was: write a temp config file pointing `claude` at the MCP server, run `claude -p "..."`, assert on tmux state rather than Claude's words. If the session appeared in tmux, the tool was called. The LLM's exact phrasing is irrelevant.

The first run failed:

```
Error: Invalid MCP configuration:
mcpServers: Does not adhere to MCP server configuration schema
```

I'd read plugin examples using top-level server names without a wrapper. The actual `--mcp-config` format requires the same `mcpServers` wrapper as `settings.json`. One misdirecting example, a few minutes of confusion, then fixed.

Both E2E tests passed. Claude connected, listed all 8 tools, and created a real tmux session. `tmux has-session -t remotecc-e2e-test` exited 0.

## 106 tests later

The chain — Claude CLI → MCP → REST → TmuxService → tmux — has been exercised with real processes at every layer. E2E tests sit behind `-Pe2e` so CI doesn't touch them, but they're there for pre-release validation.

The sendKeys fix felt small. One flag, one line. But it meant REST input and WebSocket input had been behaving differently since the start, and nothing caught it because none of the tests sent a word that tmux treats as a key name.
