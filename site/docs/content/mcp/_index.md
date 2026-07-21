+++
title = "MCP"
weight = 6
[extra]
group = "Reference"
+++

# MCP (Model Context Protocol)

Maki connects to external tool servers over MCP. Both **stdio** and **HTTP** transports are supported.

## Configuration

Add servers under `[mcp.*]` in your MCP config:

- **Global**: `~/.config/maki/mcp.toml`
- **Project**: `.maki/mcp.toml` (project config wins when both set a value)

### Stdio

```toml
[mcp.filesystem]
command = ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/tmp"]

[mcp.github]
command = ["gh", "mcp-server"]
environment = { GITHUB_TOKEN = "ghp_xxxx" }
timeout = 10000
enabled = false
```

### HTTP

```toml
[mcp.analytics]
url = "https://mcp.example.com/mcp"
headers = { Authorization = "Bearer tok123" }
```

### All options

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `command` | array | | Stdio: program + args |
| `url` | string | | HTTP: server URL |
| `environment` | map | | Stdio only |
| `headers` | map | | HTTP only |
| `timeout` | u64 | 30000 | Milliseconds (1-300000) |
| `enabled` | bool | true | |
| `always_load` | bool | false | Skip tool search, load all tools upfront |

Set `command` for stdio, `url` for HTTP. Pick one.

One option lives at the top level of `mcp.toml`, outside any server:

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `defer_tools` | usize | 10 | Defer tools only when more than this many exist |

## Tool search

Every tool definition a server exposes costs context window space, on every request. Big servers like GitHub's ship dozens of tools when a task often needs three.

Maki solves this the same way Claude Code does: MCP tool definitions are deferred by default. The model only sees a small `tool_search` tool that carries the names of all deferred tools. When a task needs one, the model searches by keywords, matches are ranked against tool names, descriptions, and parameter names, and the best 5 load with their full definitions on the next request. An exact tool name always ranks first, and extra matches are listed by name so the model can load them with one more search. Loaded tools stay available for the rest of the session. Each session keeps its own loads: a subagent searching for tools never adds them to the main conversation, and the other way around.

You don't configure anything for this. Add servers, and only the tools the model actually uses take up context.

Loads are sticky. A searched tool stays loaded for the rest of the session, calling a deferred tool loads it too, and when you resume a session the tools it was already using come back loaded.

Deferral only kicks in when it pays off. With 10 or fewer tools across all your servers, everything loads upfront and there is no search step: a search round-trip, plus the prompt cache miss it causes, would cost more than the definitions themselves. The top-level `defer_tools` key moves that line:

```toml
defer_tools = 30

[mcp.github]
url = "https://api.githubcopilot.com/mcp/"
```

Set it to 0 to always defer, or to a number larger than your tool count to turn deferral off entirely. Tools from `always_load` servers never count toward the threshold.

If one server's tools should always be visible without a search step, opt it out:

```toml
[mcp.linear]
command = ["linear-mcp-server"]
always_load = true
```

Use this for small servers you rely on every turn. Each upfront definition consumes context on every request, and loading new tools mid-session invalidates the provider's prompt cache once, so `always_load` on a big server costs more than a search round-trip.

## Naming and namespacing

Server names are ASCII alphanumeric, hyphens ok. Tools get prefixed with their server name: a `read` tool on the `filesystem` server becomes `filesystem__read`. Because of this, `__` is reserved and names can't collide with built-in tools.

## Runtime toggling

Turn servers on/off from the MCP picker in the UI. Changes save back to your config.

## Status

| Status | Meaning |
|--------|---------|
| Connecting | Waiting for the server to come up |
| Running | Tools available |
| Disabled | Off in config or toggled off in UI |
| Failed | Error shown in UI |
| NeedsAuth | Waiting for OAuth (see below) |

If one server fails, the rest still work.

## OAuth

Some HTTP servers need auth. When that happens, Maki opens your browser to log in. Other servers keep working while you authenticate. Tokens refresh on their own. If you change the server URL, you log in again.

```bash
maki mcp auth <server-name>     # manually trigger auth
maki mcp logout <server-name>   # remove stored tokens
```

### Headless machines

On a machine without a browser (say, a dev server over SSH), run `maki mcp auth <server-name>`. Maki prints the login URL. Open it on your laptop and log in. The browser lands on a `http://127.0.0.1:19876/...` page that fails to load. Copy that full URL from the address bar and paste it into the terminal to finish the login.

## Prompts

MCP servers can expose prompts (reusable message templates). Maki shows them as slash commands in the command palette: `/server:prompt-name`. Type `/` to filter.

```
/github:create-pr           # no arguments
/analytics:report monthly   # one argument
/review:code src tests      # multiple, positional
```

Skip a required argument and Maki shows a usage hint. Prompts are fetched at startup and on reconnect, so new ones need a restart. Only text content is supported.
