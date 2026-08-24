---
name: cortex-graph-schema
description: The structural contract of a Cortex agent graph — node types, edges and edge ordering, labels that become tool names, writing routing conditions, and reading a check_agent result. Use this whenever creating a Cortex, changing its structure, adding or removing nodes or transitions, or diagnosing a Cortex that routes to the wrong place.
---

# The Cortex graph

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — they need the server URL and their personal key from whoever administers the Atom
> project, then an MCP entry in their agent's config. `cortex-agent-building` has the per-agent
> commands. Advising on an agent you cannot read is worse than saying you are not connected.

**Field names come from `describe_agent_schema`, not from this document.** That tool derives its
answer from the schemas the write path validates against, so it cannot go stale. It returns the node
and edge contracts, the flow-level structures under `flowFields` (stages, typifications, tags, save
fields, timeout, inactivity recovery), and a minimal valid Cortex to start from. Call it before your
first write.

Whatever you write, **every id and keyword in it must come from `list_catalog`.** Invented values
are accepted and then silently inert — see `cortex-agent-building`.

This skill covers what a schema cannot express: the rules, and why they exist.

## Node types

Three:

| type | role |
|---|---|
| `start` | Where every conversation enters. Exactly one. Carries no logic. |
| `agent` | A step. Holds the Conversation Goal, tools, files, formats, required fields. |
| `end` | A terminal outcome. Its label is the Exit Port reported to Flowbuilder. |

Older Cortex documents may contain `selectAgent` or `tool` nodes. Those types have been removed —
they never had any runtime behaviour. If you read a Cortex that still has one, leave it in place and
tell the human; do not create new ones.

## Structural rules

`check_agent` reports these. Errors block a write; warnings do not.

**Entry** — error
- Exactly one `start`, with exactly one outgoing edge, targeting an **agent** node.
- Pointing Start at an End node yields a Cortex that cannot run, with no runtime error.
- A second edge out of Start is silently ignored — only the first is read.

**Targets** — error
- Every edge targets an `agent` or `end`. Anything else is inert: no tool, no error, no log.
- At least one `end` must exist.

**Labels** — warning, but treat as required
- Labels compile into the **tool names** the model picks between: `transfer_to_<label>`,
  `exit_<endLabel>`, `back_to_<label>`. "Ventas" and "ventas" become two near-identical tools
  separable only by description.
- Empty labels fall back to the literals `"default"` and `"other"`.
- Name the **destination**, distinctly.

**Conditions** — warning
- Every agent→agent edge needs a `conditionExpression`. Without one the model has only the target's
  name to route on.
- Edges into End nodes may omit it; the End node's description is used instead.

**Redundant edges** — warning
- No authored edge between nodes sharing a parent. They already reach each other, and the implicit
  path skips required field collection — so the authored duplicate creates an ungated route.

## Edge order is load-bearing

**A node's parent is the source of the first edge in the array that reaches it.** Reordering `edges`
can therefore reparent the tree — changing every implicit back- and sibling-transition — without any
node changing at all.

This is why `update_agent` takes **operations**, not an array:

```json
{
  "edgeOperations": [
    { "op": "add", "edge": {
        "id": "calificar-cierre", "source": "calificar", "target": "cierre",
        "data": { "label": "listo", "conditionExpression": "El cliente confirmó que quiere avanzar." } } },
    { "op": "update", "edgeId": "recepcion-info",
      "data": { "conditionExpression": "El cliente pregunta por precios u horarios." } },
    { "op": "remove", "edgeId": "info-agendar" }
  ]
}
```

Applied in order against the stored array. `add` appends, so existing parenthood survives by
construction. There is no way to submit a replacement array — deliberately.

Nodes **are** written as a full array; their order carries no meaning.

## Writing routing conditions

An edge condition is read by a model looking at the conversation. Write it as an **observable fact
about what has happened**, not an instruction:

| | |
|---|---|
| ✅ | `El cliente confirmó que quiere agendar una cita.` |
| ✅ | `El cliente preguntó por algo fuera de ventas: facturación, reclamos o soporte técnico.` |
| ❌ | `Ir al agente de agendamiento.` — an action, not a condition |
| ❌ | `Si es necesario.` — no observable content |
| ❌ | `El cliente está interesado.` — true of almost every conversation |

**Sibling conditions compete.** If two can be true at once the model picks arbitrarily and the Cortex
feels random. Make them mutually exclusive in fact, not just in intent. `check_agent` flags
semantically overlapping conditions — take those seriously; they are the most common cause of
unpredictable routing.

Descriptions are truncated at 500 characters.

## Building a tree

Start → one root agent → children. Draw parent→child edges and edges into End nodes, nothing else.
Back-to-parent and sibling transitions already exist. See `cortex-agent-building` for why.

A typical shape:

```
start
  └── Recepcion
        ├── Info      ──→ end: resuelto
        └── Agendar   ──→ end: cita_agendada
        └────────────────→ end: urgencia
```

`Info` and `Agendar` reach each other implicitly. Drawing that edge would be a warning and a bug.

## Positions

`position` is required by the schema but **ignored** — send `{ "x": 0, "y": 0 }`.

The builder owns canvas layout. It re-runs its own layout every time a Cortex is opened, anchored on
each parent and aware of real node sizes, and it freezes any group holding a node someone dragged.
Coordinates written from here would fight that, so the server discards them: an existing node keeps
exactly where it sits, and a new one is seeded below its parent for the builder to place properly.

Do not try to arrange the canvas. You cannot, and attempting it produces a worse layout than doing
nothing.

## Fields to leave alone

Preserve on read-modify-write; do not set:

| field | why |
|---|---|
| `isSuccess` | Canvas edge colour only |
| `overrideEndCondition` | Written by the UI, read by nothing |
| `composioScopeId` | Server-managed |
| `guardrails`, `adContext`, `timezone`, `voiceId` | Configured in the Atom UI |

## Reading `check_agent`

Returned automatically by `create_agent` and `update_agent`, and callable directly.

| section | meaning |
|---|---|
| `errors` | Structural problems. Fix before going further. |
| `warnings` | Authoring problems. Not blocking, usually real. |
| `flowbuilderImpact` | Which Flowbuilder flows embed this Cortex — who to warn about End node changes |
| `conditionConflicts` | Transition conditions too similar to each other |
| `whatsappFlowConflicts` | WhatsApp Flow prompts too similar to each other |

A non-empty `flowbuilderImpact` is not a problem to fix; it is a list of people to tell.

## When something looks wrong

- **"It never leaves the first node."** Required fields on that node. Look for `GateBlocked`.
- **"It jumps somewhere unexpected."** Overlapping conditions, or an authored sibling edge.
- **"It ended with `out_of_context`."** Loop protection after 5 consecutive transfers in one turn —
  usually two conditions that let it bounce between nodes.
- **"My edit did nothing."** The edge may target an ignored node type, or Start may have a second
  outgoing edge that is never read.
- **"The tree reshaped itself."** Edge order changed. Use operations.
