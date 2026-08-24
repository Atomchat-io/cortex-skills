---
name: cortex-setup
description: Connect this agent to the Cortex MCP server, or work out why it is not connected. Use this whenever someone runs /cortex-setup, asks how to set up or install Cortex, says the Cortex tools are missing or not authenticated, or when any cortex-* skill finds the tools unavailable. Walks through getting a URL and key and writing the right config for whichever coding agent they use.
---

# Connecting to Cortex

The Cortex skills teach how the product works. The **Cortex MCP server** is what lets you actually
read, create, edit and simulate agents. They install separately, and this walks through the second
one.

## First: check whether it is already connected

Look for tools named `list_agents`, `get_agent` and `describe_agent_schema`.

**If they are there, you are done.** Confirm it by asking the user for a `companyId` and running:

```
list_agents(companyId)
```

If that returns agents, everything works. Do not change any configuration.

## What the user needs

Two values, from whoever administers their Atom deployment. Ask for both before writing any config
— guessing either wastes a restart.

| | |
|---|---|
| **Server URL** | The Cortex MCP endpoint |
| **Key** | Their personal key, starting `cxk_` |

Three things to tell them while they fetch these:

- **The key identifies them personally.** Every Cortex created or edited records their name. It is
  not a shared team credential and should not be pasted into a group chat.
- **The URL selects the environment.** There is no environment switch in the tools, so the URL is
  the only thing separating a test deployment from a live one. They should know which they have.
- **Keys are revocable.** If one leaks, an administrator disables it.

## Writing the config

Ask which agent they are using, then give them exactly one of these. Put the real values in — no
environment variables, no placeholders they have to substitute later.

### Claude Code

```bash
claude mcp add --transport http cortex "<url>" \
  --header "x-cortex-key: <key>" --scope user
```

`--scope user` makes it available in every project. Drop it for the current project only.

### Cursor

`~/.cursor/mcp.json` for all projects, or `.cursor/mcp.json` for one:

```json
{
  "mcpServers": {
    "cortex": {
      "url": "<url>",
      "headers": { "x-cortex-key": "<key>" }
    }
  }
}
```

No `type` field — Cursor infers the transport from the URL.

### Codex CLI

`~/.codex/config.toml`, or `.codex/config.toml` per project:

```toml
[mcp_servers.cortex]
url = "<url>"
http_headers = { "x-cortex-key" = "<key>" }
```

### Windsurf, Zed, Cline, Antigravity, and other JSON-config agents

Same shape as Cursor:

```json
{
  "mcpServers": {
    "cortex": {
      "type": "http",
      "url": "<url>",
      "headers": { "x-cortex-key": "<key>" }
    }
  }
}
```

Two details vary between them: some require `type: "http"` and some infer it, and a few name the
top-level key `servers` rather than `mcpServers`. Check that agent's own MCP documentation for
those two. The URL and the `x-cortex-key` header are the same everywhere.

### Agents that only speak stdio

Proxy the remote server:

```json
{
  "mcpServers": {
    "cortex": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "<url>", "--header", "x-cortex-key: <key>"]
    }
  }
}
```

A last resort — it adds an npx step to every startup. Prefer a native HTTP config if the agent has
one.

### Claude.ai and Claude Desktop connectors

**Not supported.** Custom connectors there accept OAuth only, with no field for a static header, so
a key has nowhere to go. The skills can still be installed on their own; they will report that the
tools are unavailable.

## Then restart

MCP servers connect when the agent starts. Tell the user to restart, then verify:

```
list_agents(<their companyId>)
```

## If it still does not work

Work down this list — the causes are distinguishable, so do not guess.

**The tools are still missing entirely.** The config was not picked up. Check the file is where that
agent expects it, that the JSON or TOML parses, and that the agent was restarted after saving.

**An error mentioning the key, as JSON.** The request reached the server and the key was rejected —
missing, mistyped, or revoked. Have them re-copy it, or ask an administrator whether it is still
active.

**An HTML `403 Forbidden`.** The request never reached the server; something in front of it refused.
This is an infrastructure problem, not a credential one — the key is irrelevant. Report it to
whoever administers the deployment rather than re-issuing keys.

The distinction between the last two is the **content type**: JSON is the Cortex server answering,
HTML is something in front of it. Worth checking directly:

```bash
curl -s -o /dev/null -w '%{http_code} %{content_type}\n' -X POST "<url>" \
  -H 'Content-Type: application/json' -d '{}'
```

`401 application/json` means the server is reachable and the key is the issue. `403 text/html` means
it is not reachable at all.

**A literal `${SOMETHING}` appears where the key should be.** They used an environment-variable form
in the config and the variable was not set before the agent started. Replace it with the real value,
which is what these instructions do by default.

## Once connected

Point them at `cortex-agent-building` — the mental model and the working order — before they start
editing anything. And tell them the two things that surprise people:

- **A save is live.** A published Cortex is serving real conversations and a save takes effect
  immediately; there is no deploy step.
- **These tools cannot publish or delete.** Those stay in the Atom UI, deliberately.
