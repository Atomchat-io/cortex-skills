---
name: cortex-info-collection
description: Capture customer data in a Cortex agent — the required gate on a node versus passive end-of-conversation collection, targetField mapping, and why info fields are the most over-used feature in the product. Use this whenever a Cortex needs to collect, store or read customer data, whenever someone says a Cortex will not move past a step, and whenever someone wants an optional field.
---

# Capturing data

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — run the `cortex-setup` skill, which walks through getting a URL and key and writing
> the config for their agent. Advising on an agent you cannot read is worse than saying you are not
> connected.

Start here: **most requests for a field do not need a field.**

People arriving from other tooling map everything to fields, because there a field was the only place
a value could live. In a Cortex it is not. Before adding one, work out which of these you actually
have:

| You want to… | Use | Skill |
|---|---|---|
| Remember something the customer said, later in this conversation | **Nothing.** The history already does it. | — |
| Pass a value into a tool | **A tool parameter** | `cortex-http-tools` |
| Require an answer before the conversation can advance | **Node info collection** | below |
| Store a value on the client record for the business | **Passive collection** | below |
| Read a value the client already has | **`/{keyword}`** | `cortex-prompts` |

Only three involve fields, and only two of them write.

The first row is where most mistakes live: capturing a field so the agent "remembers" it. It already
remembers. The field just adds a mandatory question and a blocking gate.

## The two moments data is written

### 1. Node info collection — required, and it gates

Fields configured on an agent node must be collected **before the conversation leaves that node**.
The engine attaches them to every transition out of the node and rejects any transition that omits
one.

It is strictly all-or-nothing. The engine injects, verbatim:

> All configured fields are mandatory. Never tell the user a field is optional or skippable.

**There is no optional mode.** If you want "these three always, these two only if relevant", node
info collection is the wrong mechanism — see the worked example below.

Use it for the small set of values the conversation genuinely cannot continue without. Every field
you add is another thing the customer must supply before the Cortex can move, and a customer who
will not supply one is stuck.

A useful test: *if the customer refuses this, should the conversation stop?* If no, it belongs in
passive collection.

**The gate only covers authored transitions.** The implicit back-to-parent and sibling transitions
are field-exempt. This is why drawing a sibling edge matters: it creates a second, ungated route to
the same node. See `cortex-agent-building`.

### 2. Passive collection — at the End node

When the conversation ends, **one LLM call reads the whole transcript** and decides, in a single
pass, what to record: the general save fields, the Stage, the Typification and the Tags.

Two consequences:

- **It is passive.** Nothing blocks. Mentioned values get captured; unmentioned ones do not.
- **It is never mentioned in a prompt.** It runs after the conversation is over. `Al final, guarda
  el presupuesto y etiqueta al cliente` instructs nothing.

This is the right home for anything the business wants *if available* — which is most things. Prefer
it over the gate unless the value is genuinely blocking.

#### Conditions are the authoring surface

Save fields are extracted by description, but **Stages, Typifications and Tags each carry a
`condition` you write.** That condition is what the evaluator judges the transcript against, so it
is the whole of your control over them.

Write conditions the way you write edge conditions — as an observable fact about what happened, not
an instruction:

```
✅  El cliente confirmó una cita con fecha y hora.
✅  El cliente pidió hablar con una persona en algún momento.
❌  Marcar como agendado.                    (an action, not a condition)
❌  Cliente interesado.                       (true of nearly every conversation)
```

Each behaves differently when several match, and the differences matter:

| | when several conditions match |
|---|---|
| **Stages** | All matched stages are recorded, in the order they were reached — you get the path, not just the destination |
| **Tags** | **All** matching tags are applied. Multi-valued by design. |
| **Typifications** | All are evaluated, but **only the last one chronologically is applied.** |

That typification rule is the one that surprises people: writing five typifications whose conditions
all match a long conversation does not tag it five ways — four are silently discarded. Write
typifications as mutually exclusive outcomes, the way you would edge conditions on siblings.

**Every keyword and id here must be real**, from `list_catalog` (`kind: "stages"`,
`"typifications"`, `"tags"`). A value you invented is written without complaint and then never
matches anything at runtime — the condition may be judged true and the stage still not recorded,
because the keyword resolves to nothing. There is no error to notice.

Which stage set applies depends on the Cortex's `pipelineType` — `venta` reads `stagesVenta`,
`servicio` reads `stagesServicio`, and a mismatch means no stage is ever recorded.

`describe_agent_schema` returns the exact shape of each of these under `flowFields`.

If nothing is configured — no stages, fields, typifications or tags — the evaluator call is skipped
entirely.

## `targetField` must be real

Both mechanisms write to a keyword from the company catalog:

```
list_catalog(companyId, kind: "info_fields")
```

Returns each field's `id`, `keyword`, `label`, `description`, and the literal `/{keyword}` string for
prompts. **Use the `keyword` as `targetField`.** An invented keyword writes nowhere, silently — no
error at write time, no error at runtime, just missing data.

## Reading what a client already has

Two ways, with different guarantees:

**`/{keyword}` in a prompt** — read-only, resolved once at conversation start, and **renders a
default when the client has no value**. It cannot tell you a value was missing. Fine for
personalisation, unsafe for anything conditional. See `cortex-prompts`.

**`getFields(...)` in a code tool** — reports what actually exists, returning `null` when the client
has none of the requested fields. The only reliable way to branch on whether a value is present. See
`cortex-code-tools`.

## Worked example: a tool that returns which data to ask for

A real case, and the clearest illustration of the whole skill.

An HTTP tool returned the data points needed for a given product — some flagged required, some not.
The builder tried to model it with node info collection, then reported a bug: *"some fields are
optional but info collection makes everything required."*

Info collection was never the mechanism. Nothing had told the agent what the tool's response
**meant**. The fix:

**1.** The tool returns its list, including `is_required` per item.

**2.** The Conversation Goal explains how to read it:

```
Llama a @[ObtenerDatosRequeridos] para saber qué necesita este producto.
Pide al cliente todos los datos donde is_required sea true.
Los demás, pídelos solo si el cliente los menciona.
Cuando tengas todos los obligatorios, continúa.
```

**3.** Anything worth keeping goes to passive collection at the End node.

The general lesson, which recurs across the product: **a tool's response is inert unless the
Conversation Goal says what to do with it.**

## Building it

```
list_catalog(companyId, kind: "info_fields")     → real keywords
update_agent(companyId, agentId, expectedUpdatedAt, nodes: [...])
```

Info collection lives on the agent node as `infoCollection`, each entry carrying an `id`, `label`,
`description` and `targetField`. The **description** is what the agent uses to recognise the value in
what the customer says, so `El correo electrónico donde recibir la confirmación` beats `email`.

Passive fields live at the Cortex level as `saveFields`.

Then test — `simulate_turn`, and specifically play a customer who **refuses** one of the required
fields. That is the case the gate exists for and the one nobody tries.

## When something looks wrong

- **"It won't move to the next node."** A required field is missing. `GateBlocked` in the trace names
  which. See `cortex-simulation`.
- **"It insists a field is required when I wanted it optional."** Node info collection has no optional
  mode. Move it to passive collection.
- **"It captured the value but nothing was saved."** The `targetField` keyword is not in the catalog.
- **"It asks for something the customer already gave."** The value arrived before the node holding
  the gate. The gate checks the transition's arguments, not conversation history.
- **"It skipped the required fields entirely."** The conversation left via an implicit sibling or
  parent transition, which are field-exempt — often because someone drew a sibling edge.
- **"Nothing was saved after a simulation."** Simulation writes nothing to real records, by design.
  The trace shows what would have been saved.
- **"Only one typification was applied when several fit."** By design — the last chronological match
  wins and the rest are discarded. Make the conditions mutually exclusive.
- **"No stage was recorded."** The Cortex's `pipelineType` selects which stage set is read, so
  stages configured on the other set are never evaluated.
- **"A tag/typification/stage never fires."** Its condition is not something observable in the
  transcript, or it is too similar to another that matched first.
