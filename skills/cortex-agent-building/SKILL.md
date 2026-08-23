---
name: cortex-agent-building
description: The mental model for Atom Cortex conversational agents, plus the end-to-end workflow for building one with the Cortex MCP tools. Use this whenever someone wants to create, edit, review or understand a Cortex agent, whenever a Cortex behaves in a way that seems inexplicable, and before using any other cortex-* skill — it explains the model everything else assumes.
---

# Building a Cortex

Cortex is Atom's conversational agent product. This skill is the mental model plus the working
order; the other `cortex-*` skills cover individual surfaces.

## First: are the Cortex tools available?

Everything below depends on the Cortex MCP server. Before anything else, check whether tools named
`list_agents`, `get_agent` and `describe_agent_schema` are available to you.

**If they are not, stop and tell the user how to connect it.** Do not work around it — there is no
other route to a Cortex, and reading these skills without the tools would let you give confident,
unverifiable advice about an agent you cannot see.

Tell them roughly this, adapted to whichever agent they are using:

> I have the Cortex skills but not the Cortex tools, so I can't read or change any agent yet.
>
> You need two values from whoever administers the Atom project:
>
> - **`CORTEX_MCP_URL`** — the `cortexMcp` function URL. This is the only thing separating QA from
>   production, so check which one you have.
> - **`CORTEX_MCP_KEY`** — your personal key, starting `cxk_`. It identifies you: every Cortex you
>   create or edit records your name, so don't share it.
>
> Export both in your shell profile, then connect the server:
>
> **Claude Code**
> ```bash
> claude mcp add --transport http cortex "$CORTEX_MCP_URL" \
>   --header "x-cortex-key: $CORTEX_MCP_KEY" --scope user
> ```
>
> **Cursor** — in `~/.cursor/mcp.json`:
> ```json
> { "mcpServers": { "cortex": {
>     "url": "${env:CORTEX_MCP_URL}",
>     "headers": { "x-cortex-key": "${env:CORTEX_MCP_KEY}" } } } }
> ```
>
> **Codex** — in `~/.codex/config.toml`:
> ```toml
> [mcp_servers.cortex]
> url = "<your CORTEX_MCP_URL>"
> env_http_headers = { "x-cortex-key" = "CORTEX_MCP_KEY" }
> ```
>
> **Other agents** use the same URL and `x-cortex-key` header in their own MCP config —
> see the `INSTALL.md` that ships alongside these skills.
>
> Restart the agent afterwards — the variables must be set **before** it starts.

If the tools *are* present but a call returns an authentication error, the key is missing, wrong or
revoked. Same fix, and worth confirming `CORTEX_MCP_URL` points at the environment they think it
does.

## The one thing to get right

**A Cortex is one agent, not a team of agents.**

A Cortex contains several *agent nodes*, and it is natural to read them as separate agents handing
work to one another. They are not. There is one identity, one memory, one conversation. Moving to
another node swaps the **Conversation Goal** — what the agent is currently trying to achieve.
Nothing else changes. The customer is talking to the same entity the whole time.

Nearly every misuse traces back to the opposite belief. The tell is thinking of the Cortex as a
program you are configuring rather than a person you are briefing.

Two consequences that decide most of your choices:

**Identity is stated once**, in the System Instructions. Who the business is, tone, language, what
is out of scope. Repeating any of it per node makes prompts longer and eventually contradictory.

**The agent cannot see its own configuration.** It has no view of its nodes, tools, files or fields.
`Save the email to the email field`, `use the pricing document`, `transfer to the billing agent` all
instruct something with no such lever. Each is configured elsewhere — see `cortex-prompts`.

## When to add a node

Add one when there is a real reason:

- **The customer's path forks.** New patient versus returning patient; sales versus support.
- **A distinct stage** with a different objective. Qualifying is not booking.
- **Performance.** A node carrying many tools and files is slower and less accurate; splitting by
  what is actually needed at that point helps.

Do not add nodes to tidy the canvas, mirror a script's paragraphs, or "separate concerns". Each node
boundary is a decision the model can get wrong, and each transition is a tool competing for its
attention. **If two nodes would have nearly the same Conversation Goal, they are one node.**

## Shape: build a tree

Start → one root agent node → children below.

Two kinds of connection **already exist without being drawn**:

- **Back to the parent** — any node can fall back when it cannot proceed
- **Between siblings** — nodes sharing a parent reach each other directly

So draw parent→child edges and edges into End nodes. Nothing else.

Drawing a sibling edge does not add a path; it adds a **second** path to the same place — and the
implicit one **skips the node's required field collection**. Trying to be thorough silently puts a
hole in your data capture. `check_agent` flags it.

Back-edges and cross-tree edges are allowed where genuinely needed. They are exceptions, not a
pattern.

## Where a Cortex lives, and where it stops

A Cortex does not run alone. It is embedded as a node inside a **Flowbuilder** flow, and that flow is
what actually receives the WhatsApp conversation.

**The Flowbuilder node has one branch per End node in your Cortex.** When the conversation finishes,
the Cortex reports which End node it exited through, and Flowbuilder follows the matching branch.
That is the entire interface between them: a Cortex ends, and names how it ended.

So **adding or removing an End node changes the branches on the Flowbuilder side**, which then needs
rewiring by whoever owns that flow — outside the Cortex, and outside these tools. `check_agent`
reports which Flowbuilder flows depend on the Cortex you are editing; say so when it is non-empty.

### A Cortex cannot act after it ends

This is the boundary people most often expect to be somewhere else. In particular:

**A Cortex cannot assign a conversation to a human.** There is no tool, no node type and no setting
for it. What it can do is *end* — and the Flowbuilder branch for that exit does the assignment.

The same applies to anything else that should happen afterwards: notifying a team, starting another
flow, waiting, sending a follow-up hours later. Once a Cortex exits, Flowbuilder can do whatever it
likes. Inside the Cortex, the only lever is **which End node you exit through**.

So name End nodes after **outcomes**, not actions:

| ✅ outcome | ❌ action |
|---|---|
| `cita_agendada` | `enviar_confirmacion` |
| `sin_stock` | `avisar_a_ventas` |
| `fuera_de_alcance` | `asignar_a_humano` |

The outcome is what the Cortex actually knows. What to *do* about it is Flowbuilder's decision, and
it can change without touching the Cortex at all.

**Several exits can lead to the same thing.** Routing `fuera_de_alcance`, `cliente_molesto` and
`pide_humano` all to one assignment block is normal and common — you do not need a single dedicated
escalation exit, and collapsing three genuinely different outcomes into one just to simplify the
Flowbuilder side loses information the business may want later. Split by what happened; let
Flowbuilder converge them.

Practically, when someone asks for "escalate to an agent":

1. Work out which outcome is really being described, and whether an existing End node already covers it
2. Write the edge condition for when to take it
3. Have the Cortex tell the customer a person will take over — that is a *message*, not an action
4. Tell the human that the Flowbuilder branch for that exit needs to perform the assignment

Step 4 is the one that gets forgotten, and the symptom is a conversation that politely promises a
human and then goes nowhere.

## Saving is live

There is no staging and no deploy step. A published Cortex is serving real conversations and **a save
takes effect immediately**. The tools say so when it applies — relay it *before* writing, not after.

Publishing and deleting are deliberately not available here; they stay in the Atom UI.

## The workflow

**1. Get the `companyId` from the human.** Never guess it. There is no lookup by name, on purpose.

**2. Survey.**
```
list_agents(companyId)                    → what exists
get_agent(companyId, agentId)             → the full document
```
Keep the `updatedAt` from `get_agent` — you need it to write back.

**3. Learn the contract**, before your first write:
```
describe_agent_schema()   → live field names, structural rules, a valid starting template
```

**4. Resolve real ids.** Never invent one:
```
list_catalog(companyId, kind: "info_fields" | "stages" | "typifications" | "tags"
                            | "whatsapp_flows" | "files" | "dynamic_tables" | "models")
```

**5. Write.**
```
create_agent(companyId, name, baseSystemPrompt, nodes, edges)
update_agent(companyId, agentId, expectedUpdatedAt, set?, nodes?, edgeOperations?)
```
`expectedUpdatedAt` guards against overwriting someone editing in the browser; if it is rejected,
re-read and re-apply rather than forcing. Positions are assigned server-side — never send them.
Edges are edited through **operations**, not by replacing the array, because edge order determines
the node tree. See `cortex-graph-schema`.

**6. Check.**
```
check_agent(companyId, agentId)   → errors, warnings, Flowbuilder impact, conflicting conditions
```
Both `create_agent` and `update_agent` return this automatically. Read it; do not just look at
whether the write succeeded.

**7. Test.**
```
simulate_turn(companyId, agentId, message)             → new conversation, returns a sessionId
simulate_turn(companyId, agentId, message, sessionId)  → continue it
```
Read `agentBuilderLogs`, not just the reply. See `cortex-simulation`.

**8. Iterate, then tell the human to publish.**

## Which skill for what

| task | skill |
|---|---|
| Node types, edges, structural rules | `cortex-graph-schema` |
| System Instructions, Conversation Goals, `/{keyword}` | `cortex-prompts` |
| Capturing customer data | `cortex-info-collection` |
| Calling an API | `cortex-http-tools` |
| Custom logic, multi-step processes | `cortex-code-tools` |
| Composio toolkits and MCP servers | `cortex-integrations` |
| Documents and catalogs | `cortex-rag` |
| Buttons, lists, carousels, audio, forms | `cortex-response-formats` |
| Testing and diagnosis | `cortex-simulation` |

## When something looks wrong

Mechanics to recognise, never to write into a prompt:

- **Transitions are tools.** Each edge compiles to one the model may call, carrying a reason and,
  where the gate applies, the node's required fields.
- **Loop protection** forces an exit through `out_of_context` after 5 consecutive transfers. A
  conversation ending there was bouncing between nodes — usually overlapping conditions.
- **Summarization** only triggers at 600 messages or ~700k tokens. It is not why your Cortex forgot
  something.
