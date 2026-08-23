# Installing

Two pieces, and you want both:

| | what it is | how it installs |
|---|---|---|
| **The skills** | How Cortex works, so your agent doesn't guess | `npx skills add` |
| **The MCP server** | The tools — read, create, edit and simulate agents | Per-agent config, below |

The skills work alone as documentation. The tools work alone but your agent will make the mistakes
the skills exist to prevent. Together is the point.

---

## 1. Skills

One command for every agent — the CLI supports 77+ of them and writes to the right place:

```bash
npx skills add Atomchat-io/cortex-skills
```

Useful flags:

```bash
npx skills add Atomchat-io/cortex-skills --list     # see what's there, install nothing
npx skills add Atomchat-io/cortex-skills --global   # user-level rather than just this project
npx skills add Atomchat-io/cortex-skills --all      # every skill, every detected agent
npx skills update                     # pull the latest later
```

You can also install a single skill with `--skill cortex-code-tools`.

For a Claude Code plugin instead — the same skills plus the MCP server declaration in one step:

```bash
claude plugin marketplace add Atomchat-io/cortex-skills
claude plugin install cortex@atom
```

---

## 2. Credentials for the MCP server

The Cortex MCP server runs inside Atom's infrastructure. You need two values from whoever
administers your Atom deployment:

| | |
|---|---|
| `CORTEX_MCP_URL` | The server's URL |
| `CORTEX_MCP_KEY` | Your personal key |

Three things to know:

- **The key identifies you.** Every agent you create or edit records your name. Don't share it.
- **The URL selects the environment.** There is no environment switch in the tools, so the URL is
  the only thing separating a test deployment from a live one. Check which you have.
- **Keys are revocable** by your administrator.

Keep them in your environment, not in a config file:

```bash
# ~/.zshrc or ~/.bashrc
export CORTEX_MCP_URL="https://…/cortexMcp"
export CORTEX_MCP_KEY="cxk_…"
```

Every config below reads from there, so nothing secret lands in a file you might commit.

---

## 3. Connect the server

### Claude Code

```bash
claude mcp add --transport http cortex "$CORTEX_MCP_URL" \
  --header "x-cortex-key: $CORTEX_MCP_KEY" --scope user
```

Drop `--scope user` to limit it to the current project. Verify with `claude mcp list`, or `/mcp` in
a session.

### Cursor

`~/.cursor/mcp.json` globally, or `.cursor/mcp.json` per project:

```json
{
  "mcpServers": {
    "cortex": {
      "url": "${env:CORTEX_MCP_URL}",
      "headers": {
        "x-cortex-key": "${env:CORTEX_MCP_KEY}"
      }
    }
  }
}
```

Cursor infers the transport from the URL, so there is no `type` field.

### Codex CLI

`~/.codex/config.toml` globally, or `.codex/config.toml` per project:

```toml
[mcp_servers.cortex]
url = "https://…/cortexMcp"
env_http_headers = { "x-cortex-key" = "CORTEX_MCP_KEY" }
```

`env_http_headers` maps a header to the **name of an environment variable**, which is what you want.
`http_headers` takes a literal value and would put your key in the file. Codex does not expand
variables in `url`, so that one is written out.

### Windsurf, Zed, Cline, Antigravity and other JSON-config agents

Most follow Cursor's shape:

```json
{
  "mcpServers": {
    "cortex": {
      "type": "http",
      "url": "<your CORTEX_MCP_URL>",
      "headers": { "x-cortex-key": "<your CORTEX_MCP_KEY>" }
    }
  }
}
```

Two details vary: some require `type: "http"` and some infer it; some name the top-level key
`servers` rather than `mcpServers`. Check your agent's MCP docs for those. The URL and header are
the same everywhere.

### Agents that only speak stdio

Proxy the remote server:

```json
{
  "mcpServers": {
    "cortex": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "<your CORTEX_MCP_URL>",
               "--header", "x-cortex-key: <your CORTEX_MCP_KEY>"]
    }
  }
}
```

A last resort: it writes your key into a plaintext file and adds an npx step to every startup.

### Claude.ai and Claude Desktop connectors

**Not supported.** Custom connectors there accept OAuth only, with no field for a static header, so
an API key has nowhere to go. Use one of the agents above. The skills can still be installed on
claude.ai on their own; they will report that the tools are unavailable.

---

## 4. Check it worked

Ask your agent:

```
Using the cortex tools, list the agents for company <a companyId you have>
```

A `companyId` is always required and is never looked up by name, so have one ready.

If your agent says it has the Cortex skills but not the tools, the server is not connected. Almost
always this is the environment variables not being set **before** the agent started, in the same
shell.

---

## What the tools can and cannot do

Can: list and read Cortex agents, create and edit them, read the company catalogs (fields, stages,
files, tables, flows), search integrations, test HTTP and code tools, and simulate conversations.

Cannot, by design: **publish**, **delete**, connect an integration, upload a file, or write to any
real customer record. Those stay in the Atom UI, where a human does them deliberately.

Two things to keep in mind:

- **A save is live.** A published Cortex is serving real conversations, and a save takes effect
  immediately — there is no separate deploy step. The tools tell you when this applies.
- **Simulation is safe.** It cannot touch a customer record, stage, tag or typification.

## Troubleshooting

| symptom | cause |
|---|---|
| Agent has skills but no tools | Server not connected — check your agent's MCP list |
| `401` or missing-key error | Key absent, wrong, or revoked. Check the variable is set in the shell that launched the agent |
| Connects, every call fails | `CORTEX_MCP_URL` points somewhere stale or wrong |
| Config shows a literal `${CORTEX_MCP_KEY}` | The variable was not set when the agent started |
| Tools work, agent still guesses | Skills not installed — run `npx skills add` |
