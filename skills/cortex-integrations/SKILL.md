---
name: cortex-integrations
description: Connect third-party systems to a Cortex agent — Composio toolkits and external MCP servers, how per-Cortex connection scoping works, finding the right action, and attaching it to a node. Use this whenever a Cortex needs to act on Google Calendar, Gmail, HubSpot, Slack or any external system, and whenever an integration action is missing or publishing is blocked on an unconnected toolkit.
---

# Integrations

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — run the `cortex-setup` skill, which walks through getting a URL and key and writing
> the config for their agent. Advising on an agent you cannot read is worse than saying you are not
> connected.

Two ways to give a Cortex abilities in an external system:

- **Composio toolkits** — the built-in catalog: Google Calendar, Gmail, HubSpot, Slack and many more
- **MCP servers** — any external MCP endpoint

Configured differently, used the same way: pick the specific actions a node needs.

## Connecting happens in the UI

Connecting a Composio toolkit requires a browser OAuth flow. **That cannot happen through these
tools, and there is deliberately no tool for it.**

Ask the human to connect it in the Atom builder UI, then verify:

```
list_connected_integrations(companyId, agentId)
```

**Connections are scoped per Cortex, not per company.** A toolkit connected for one Cortex is not
available to another. This surprises people constantly — "we already connected Google Calendar" is
usually true and usually about a different Cortex.

## Finding the right action

```
search_composio_tools(query)
    → Composio's own semantic search across EVERY toolkit. Each result reports the toolkit
      it belongs to. Start here when you know the task but not the product.

search_composio_tools(query, toolkitSlug)
    → the same search, narrowed to one toolkit.

search_composio_tools(toolkitSlug)          # no query
    → lists that toolkit's actions in full.

list_composio_toolkits()
    → the global catalog of integrations. Its isConnected is NOT per-Cortex — ignore that field.

list_connected_integrations(companyId, agentId)
    → what THIS Cortex can actually use. The one that matters before attaching anything.
```

### The query is keywords, not a description

The index matches **words, not meaning**, and gets this wrong in two ways that both look like the
action not existing:

| query | result |
|---|---|
| `create event` | ✅ `GOOGLECALENDAR_CREATE_EVENT` |
| `create a calendar event` | ✅ same |
| `create a calendar event for an appointment` | ❌ nothing — five or more words matches nothing |
| `schedule appointment` | ❌ nothing — the vendor calls it *create event* |

So: **two to four English words, naming the operation the vendor would name** — create, update,
list, delete, send, search — plus its object. Not the job you are doing, and not Spanish.

An empty result means the query was wrong far more often than it means the integration cannot do
it. Try fewer words, and the vendor's own verb, before telling anyone it is not supported.

A working sequence:

1. `search_composio_tools` with two to four keywords — no toolkit
2. Note the `toolkitSlug` on the result you want
3. `list_connected_integrations` — is that toolkit connected for *this* Cortex?
4. If not, ask the human to connect it in the UI, then re-check
5. Read the action's `inputParameters` before attaching it

Steps 1 and 3 are separate on purpose: search tells you what exists across all of Composio, and only
`list_connected_integrations` tells you what this Cortex may actually call.

## Attaching actions to a node

Attach the **specific actions needed at that point in the conversation** — never a whole toolkit.

Each attached action is another tool competing for the model's attention. A node carrying thirty
Gmail actions picks the wrong one; a node carrying `SendEmail` does not. This is also the most
common reason a node gets slow.

If a node genuinely needs many actions, that usually means it is doing too much — see
`cortex-agent-building`.

Attach via `update_agent`, writing the `tools` array on the agent node: each entry carries the
`toolkitSlug`, the `toolSlug`, a `name` and a `description`.

### Copy the name and description from Composio verbatim

**Never write your own.** Paste the `name` and `description` exactly as `search_composio_tools`
returned them.

The reason is that they are not what the model reads. At runtime the engine fetches the action from
Composio by `toolSlug` and binds **Composio's own live description and its `inputParameters`
schema** — the stored `name` and `description` reach no model at all; they are what a human sees in
the builder. So an improved description does not improve tool selection by one word. All it does is
make the builder disagree with what the agent is actually working from, and send whoever debugs it
next looking in the wrong place.

`toolSlug` is the only part of the entry with runtime meaning. Get it exactly right, and let the
other two be Composio's.

If a description is genuinely too vague for the model to pick the action reliably, the fix is in the
**Conversation Goal** — say when to reach for that action — not in a rewritten `description` that
nothing reads.

## Calling integrations from code

A code tool can call any connected toolkit directly:

```js
await toolkits.googlecalendar.CreateEvent({ summary: "Llamada", start: { dateTime: params.fecha } });
```

**Choose between attaching and coding by asking who decides.**

| The decision to call it is… | Do this |
|---|---|
| Made by the model, from the conversation | Attach the action to the node |
| Already determined by your logic | Call it from a code tool |

"Book the appointment the customer just agreed to, then record it in the CRM and confirm" is one
deterministic process — a code tool. "Book, cancel, or reschedule depending on what they want" is a
choice — attached actions.

Publishing statically scans code tools for unconnected toolkits and blocks. The scan cannot follow
`toolkits[variable]`, so avoid dynamic indexing — it defeats a check that would otherwise catch the
problem before production. See `cortex-code-tools`.

## MCP servers

An agent node can consume tools from an external MCP server. **Connect, list, attach:**

```
list_mcp_tools(serverUrl, transport?, headers?)
```

This probes the live server and returns its tools. `transport` defaults to streamable HTTP; pass
`headers` for auth.

Then write an `mcpTools` entry on the node with the URL, transport, headers, and — importantly —
the **selected tools**. The engine keeps that snapshot so that if the server is unreachable at
runtime, the agent can report the outage rather than silently losing abilities mid-conversation.

Treat headers as credentials: they are visible to anyone who can edit the Cortex.

## Choosing the mechanism

| Situation | Use |
|---|---|
| Standard SaaS action, model decides when | Composio action on the node |
| One call to a plain REST API | HTTP tool — `cortex-http-tools` |
| Several steps, or logic between them | Code tool — `cortex-code-tools` |
| A system that already speaks MCP | MCP server on the node |
| Booking an appointment | A calendar action on the node — see below |

### Appointments

Attaching a calendar action from Composio is how a Cortex books appointments today. Nothing special
about it: pick the action, attach it to the node, and let the agent collect the date and time in
conversation like any other data.

Worth telling the user, though: **Atom also has appointment scheduling built in**, where the
customer receives the WhatsApp scheduling form and submitting it creates the booking. It is
experimental, still in development, and **cannot be configured through these tools** — it is set up
by hand in the Atom UI, on the agent node under Tools → Herramientas de Atom. Mention it so they can
choose; the Composio route is what you can actually wire up from here.

## When something looks wrong

- **"The action isn't available."** Connected for a different Cortex, or not at all. Check
  `list_connected_integrations` with *this* `agentId`.
- **"Publishing is blocked on an unconnected toolkit."** A code tool references a toolkit this
  Cortex has not connected. Connect it in the UI.
- **"`toolkits` is undefined in my code tool."** No connected integrations in scope — including a
  `test_code_tool` run without `companyId` and `agentId`.
- **"The MCP tools disappeared."** The server was unreachable; the snapshot lets the agent say so
  rather than pretend the tools never existed.
- **"It calls the wrong action."** Too many attached, with overlapping descriptions. Attach fewer.
  Note the descriptions in play are Composio's, not the ones stored on the node — rewriting those
  changes nothing. Steer it from the Conversation Goal instead.
- **"Search returns nothing."** Try describing the task differently, or drop the toolkit filter and
  search globally — the match is semantic, so wording matters more than keywords.
