# Cortex skills

Skills that teach a coding agent how **Cortex** — Atom's conversational agent product — actually
works, so it can build and debug Cortex agents without guessing.

Pair them with the Cortex MCP server and your agent can read, create, edit and simulate Cortex
agents directly. Without it the skills still explain the product, and will tell you they cannot see
any agent rather than inventing one.

## Start here

```bash
npx skills add Atomchat-io/cortex-skills
```

Then, in your agent:

```
/cortex-setup
```

**Run that first, every time.** It checks whether you are already connected and tells you so — or
walks you through connecting, asking for the two things you need from your Atom administrator: a
server **URL** and your personal **key**. If something is already wrong, it also tells you which
kind of wrong and who to go to.

No environment variables, no shared secrets committed anywhere, nothing to export before your agent
starts.

[INSTALL.md](./INSTALL.md) is the same thing written out, if you would rather read than be walked
through it.

```bash
npx skills add Atomchat-io/cortex-skills
```

## The skills

| skill | covers |
|---|---|
| `cortex-setup` | Connecting this agent to the Cortex MCP server. Run `/cortex-setup`. |
| `cortex-agent-building` | The mental model and the end-to-end workflow. Start here. |
| `cortex-graph-schema` | Node types, edges, edge ordering, structural rules |
| `cortex-prompts` | System Instructions vs Conversation Goal, `/{keyword}` interpolation |
| `cortex-info-collection` | Required gates, passive collection, why fields get over-used |
| `cortex-http-tools` | Calling an API, parameters vs stored fields |
| `cortex-code-tools` | The JavaScript sandbox and the Atom SDK |
| `cortex-integrations` | Composio toolkits and MCP servers |
| `cortex-rag` | Files and dynamic tables |
| `cortex-response-formats` | Buttons, lists, carousels, audio, WhatsApp Flows |
| `cortex-simulation` | Testing, and reading the execution trace |

Each ends with a **"when something looks wrong"** section: the internal mechanics worth recognising
while diagnosing, clearly separated from what you should actually write. That distinction matters —
a model told about a mechanism will try to invoke it, and several of these are best left alone.

## What they are opinionated about

The skills are organised around the belief that produces most Cortex mistakes: **that you are
programming the agent.** You are not — you are briefing one.

Concretely, they will push back on:

- Treating a Cortex's several agent nodes as several agents. It is one agent whose objective changes.
- Repeating identity and tone in every node instead of stating it once.
- Writing "save this field" or "use this document" into a prompt, when the agent cannot see its own
  configuration.
- Using `/{keyword}` as a variable. It is read-only, resolved once, and renders a default when empty.
- Capturing fields so the agent can "remember" something. The conversation already remembers.
- Building a code tool as a wrapper around an HTTP call it could have made directly.
- Calling a tool and never telling the agent what its response means.

## Contributing

These are generated from a private repository where they live beside the engine they describe, so a
schema change and its documentation land in the same review. **Pull requests here will not survive
the next sync** — open an issue instead, or contact Atom.

## Licence

MIT.
