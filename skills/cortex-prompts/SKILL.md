---
name: cortex-prompts
description: Write System Instructions and Conversation Goals for a Cortex agent — what belongs in each, referencing tools with @[ToolName], and the /{keyword} interpolation rules. Use this whenever writing, reviewing or debugging any Cortex prompt text, whenever a Cortex repeats itself or ignores an instruction, and whenever someone asks why a name comes out as a placeholder.
---

# Writing Cortex prompts

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — run the `cortex-setup` skill, which walks through getting a URL and key and writing
> the config for their agent. Advising on an agent you cannot read is worse than saying you are not
> connected.

Two places hold prompt text, doing different jobs. Putting content in the wrong one is the most
common authoring error in the product.

Worked templates by business type: [`references/templates.md`](references/templates.md) — read it
when starting a Cortex from scratch.

## System Instructions — who the agent is

Written **once** for the whole Cortex. Every node inherits it.

Cover:

- **Identity and business.** Who the agent represents and what the business does.
- **Tone and language.** Formal or warm, which language, how long replies run, whether to use the
  customer's name.
- **Global boundaries.** What it never does — invent prices, promise delivery dates, give medical or
  legal advice.
- **Escalation.** When a human should take over.

The test for whether something belongs here: **is it true in every moment of every conversation?**
If it is only true while booking an appointment, it belongs in that node.

## Conversation Goal — what to achieve right now

Per agent node. It has **sections**, in this order, and most nodes need more than one. A goal that
is only an objective is an unfinished goal.

### 1. What is already known

First, always: the fields this node can rely on, interpolated. The engine resolves them once when
the conversation opens, so by the time the node runs they are either a value or empty.

```
# Datos que ya tenés
nombre: /{full_name}
plan: /{plan_contratado}

Si `nombre` viene vacío, preguntalo. Si trae valor, usalo y no lo preguntes.
```

This is the difference between an agent that greets a returning customer by name and one that asks
a customer their name for the fourth time. **Writing "if they have not given their name, ask for
it" is not the same thing** — that reasons about the transcript, which starts empty, so it always
asks. The field is what knows.

Which fields exist comes from `list_catalog`; whether to read one rather than ask is a decision for
the human — see *Ask before using a field interpolation* below.

### 2. The objective, and the signal it is complete

```
Averiguá qué servicio necesita y si ya vino antes.
Preguntá una cosa a la vez.
Cuando tengas ambos datos, continuá.
```

### 3. When to reach for each tool

The tool's own description says what it does; the goal says **when, here, and what its answer
means** — which is the most common gap in a Cortex that has tools and never uses them:

```
Consultá @[VerificarStock] antes de prometer disponibilidad.
Si `disponible` es false, ofrecé las alternativas que devuelve.
```

### 4. When to use a response format

A format that is enabled is not a format that gets used. Say the moment:

```
Cuando ofrezcas los horarios disponibles, mostralos como lista, no como texto.
```

### 5. When to leave this node

Transferring and exiting are the same act — the conversation stops being this node's. Say the
condition in the customer's terms, and nothing about the machinery:

```
Si se enoja, insiste en hablar con alguien, o pregunta algo fuera del taller,
derivá a una persona.

Si ya agendaste la cita, terminá.
```

**Not the transition's name, and not `@[...]`.** The label and the condition on the edge are what
route this; naming them here is a second description of the same routing that drifts from the first.
`@[ToolName]` is for the node's own tools, never for a way out.

### On length

Only what this node needs. A node that reads one field, has no tools and one way out is four lines
and that is correct; a node that books an appointment against a calendar is longer, and shortening
it would only move the missing part somewhere it cannot be read.

**Never restate identity, tone or business context.** Those are inherited from the System
Instructions, and duplicating them guarantees the two drift apart until they contradict each other
— at which point the agent's behaviour depends on which one it weighted, and you cannot reason
about it.

The smell worth keeping: **"and then"** in a Conversation Goal usually means two nodes.

## The agent cannot see its own configuration

It has no view of its nodes, tools, files or field setup. These instruct nothing:

- ❌ `Guarda el email del cliente en el campo email.`
- ❌ `Busca la respuesta en el PDF de precios.`
- ❌ `Tienes una herramienta para agendar; úsala cuando corresponda.`
- ❌ `Transfiere al agente de Facturación.`

Each is configured elsewhere, and happens automatically:

| You want | Where it actually lives |
|---|---|
| A field captured before moving on | Node info collection — `cortex-info-collection` |
| Knowledge consulted | Retrieval, before the turn — `cortex-rag` |
| A tool to exist | The tool's own description |
| The condition for moving | The edge's condition — `cortex-graph-schema` |

Writing these into a prompt is not merely useless — it spends the agent's attention on instructions
it cannot follow, and makes the real prompt harder to follow.

### What it *can* see, and can therefore be told about

The line is what has a name the agent is given:

| ✅ | ❌ | why |
|---|---|---|
| `consultá @[VerificarStock] antes de prometer` | `tenés una herramienta de stock` | `@[ToolName]` names a tool this node holds. A vague mention names nothing. |
| ``si `nombre` viene vacío, preguntalo`` | `guardá el nombre en el campo nombre` | An interpolated value is in the text it reads. Capture happens outside the prompt. |
| `si se enoja, derivá a una persona` | `transferí al agente de Facturación` / ``finalizá por `solicita_humano` `` | Say the situation. The edge's label and condition do the routing, and repeating them here gives the same decision two descriptions that drift. |

So a Conversation Goal describing *when* to use a tool, *what is already known*, and *the situations
that end this node's part* is talking about things the agent can act on. The same goal naming a
node, a transition, a file or a field to write is describing machinery it cannot reach.

## Referencing a tool: `@[ToolName]`

The one legitimate way to mention a tool. Use it when a specific moment needs guidance the tool's own
description should not carry:

```
Al agendar, llama siempre a @[CreateEvent] con la zona horaria America/Bogota.
```

This clarifies **how** to use a tool here. It is not how you tell the agent a tool exists — the
tool's description does that, and improving the description beats any prompt text. It is also where
you explain **what a tool's response means**, which is one of the most common gaps:

```
Consulta @[VerificarStock] antes de prometer disponibilidad.
Si disponible es false, ofrece las opciones de alternativas.
```

**Nothing substitutes it.** The builder UI offers autocomplete and highlights the mention, but the
engine passes the text through untouched — the model simply reads `@[VerificarStock]` and connects
it to the tool of that name. Two consequences:

- The name has to **match the tool exactly**. A typo is not an error; the model just sees a
  reference to a tool that does not exist and works around it.
- Nothing checks that the referenced tool is attached to this node. Renaming or removing a tool
  leaves stale mentions in prompts, silently.

## Values: prefer variables, map fields once at the top

Two ways a value reaches the agent, and they are not equal.

**A tool parameter** — a `{{handlebars}}` variable on an HTTP tool, or a declared parameter on a
code tool. The model supplies it at call time from the conversation. **Prefer this.** It is what
tools actually understand, it needs nothing to exist on the client record beforehand, and it never
silently fails.

**A field interpolation** — `/{keyword}` in prompt text, pulling a value the client already has.
Useful, but constrained: read-only, resolved once when the conversation starts, and it only works if
the field already exists on that client *before the session begins*.

### Ask before using a field interpolation

This is a decision for the human, not for you. Before writing any `/{keyword}`, ask which they want:

**Use an existing field** — the value is already on the client record, and you interpolate it. Fast,
no questions asked of the customer, but it is only there if the record already had it.

**Ask the customer during the conversation** — no interpolation at all. The agent asks, and the
value is written back by passive collection at the end of the session. Slower, always works, and it
is how the record gets populated in the first place.

Do not guess between these. "Do you want me to read `documento` from the client record, or have the
agent ask for it?" takes one line and avoids building a prompt that reads a field nobody populates.

The bar for an interpolation is a value **nobody should ever be asked for** and that the agent
nonetheless needs: an identifier already on the record, an assignment the business made
(`asesor_asignado`, `sucursal`), something the system set before the session opened. A value the
customer would happily state is not a reason to add a field — let the agent ask, and let passive
collection store it.

### The context block

Map the fields once, under a heading, then refer to those names in natural language everywhere else.

**Where it goes depends on who needs it.** A field every node relies on — the customer's name, their
plan — belongs at the top of the System Instructions. A field only one node reads belongs in that
node's Conversation Goal, as its first section, where whoever reads the node can see what it knows
without opening the Cortex-level prompt.

```
# Contexto
nombre: /{name}
plan_actual: /{plan_contratado}
asesor: /{asesor_asignado}
```

Then, in a Conversation Goal:

```
Saluda al cliente por su nombre.
Si su plan_actual es Premium, ofrécele la revisión sin coste.
Al agendar, llama a @[CreateEvent] con el asesor del contexto.
```

Two reasons this beats scattering `/{...}` through the text. The mapping is in one place, so what
the Cortex depends on is visible at a glance instead of buried in five paragraphs. And the rest of
the prompt reads as instructions to a person rather than a template — which is also how a tool call
gets described: *"con el asesor del contexto"*, not `/{asesor_asignado}` pasted into an argument.

### What an unresolved value looks like

If the client has no value, **the placeholder is left in the text exactly as written** — the agent
literally sees `/{plan_contratado}`, braces and all. Not blank, not a default.

That is workable, but only if you say so:

```
# Contexto
plan_actual: /{plan_contratado}

Si plan_actual aparece literalmente como /{plan_contratado}, es que no tenemos el dato:
no lo menciones y pregúntalo si hace falta.
```

Without that line the agent will happily tell a customer their plan is `/{plan_contratado}`.

When a branch genuinely depends on whether a value exists, do not infer it from the placeholder —
use a code tool with `getFields(...)`, which reports what is actually there. See `cortex-code-tools`.

### It is read-only, and frozen at turn one

There is no write form:

```
❌ Guarda la respuesta en /{presupuesto} y luego usa /{presupuesto} para calcular.
```

Neither half works. And a value written mid-conversation will not appear — the text is resolved once
and never re-rendered.

Get real keywords from `list_catalog` with `kind: "info_fields"`, which returns the exact
`/{keyword}` string to paste. Never invent one.

## Recovery messages — writing for a silent customer

A Cortex can send **proactive messages when the customer stops replying**. Up to three attempts,
each with its own delay and its own message, configured in `inactivityRecovery`. It is the only
inactivity mechanism there is.

### The timers stack

Each delay is measured from the previous step, not from when the customer went quiet, and the
**close timeout starts after the last attempt** rather than bounding the whole thing:

```
inactivityRecovery: {
  enabled: true,
  recoveryAttempts: [{ value: 5, unit: "minutes" }],
  recoveryPrompts:  ["…"],
  closeTimeout:     { value: 30, unit: "minutes" },
}
```

reads as *"5 minutes of silence → send the message → 30 more minutes → give up"*, so the
conversation stays open for **35 minutes**, not 30. With three attempts of 5, 30 and 60 minutes and
a 30-minute close, it is **just over two hours**.

This catches people out: a `closeTimeout` of 30 looks like a half-hour cap and never is. Add the
attempts up before telling anyone how long a conversation stays open.

### What happens at the end

When the attempts run out, the Cortex signals Flowbuilder, which routes the conversation through its
**inactivity-close exit**. That exit is a Flowbuilder branch, **not** one of your End nodes — it does
not appear among your Exit Ports and nothing inside the Cortex runs on that path. Whatever should
happen to an abandoned conversation belongs to that branch.

Any customer reply cancels the whole chain at any point, and the close is skipped if the
conversation has already been closed.

These are the only messages the agent sends unprompted, and they are written badly more often than
any other text in the product — because they get written as if continuing a conversation that is
still happening.

Remember the situation you are actually writing for: the customer went quiet, possibly hours ago,
possibly mid-question, and has since been doing something else entirely. They may not remember
writing to you.

```
❌ ¿Entonces qué prefieres?
❌ Sigo esperando tu respuesta.
❌ ¿Hola? ¿Sigues ahí?
```

The first assumes they remember the question. The second is passive-aggressive. The third is what a
bot sounds like.

```
✅ Hola de nuevo. Te escribía por la cita que estábamos agendando —
   ¿te sigue interesando? Si prefieres, lo retomamos otro día.
```

What makes it work: it re-establishes context in one line, asks something answerable cold, and gives
an exit. A recovery message that only makes sense if you scroll up has failed.

Escalate the tone across attempts rather than repeating it. First attempt: a light nudge. Second:
re-state the value and make it easy to say no. Third, if you use one: say plainly that you will stop
here and how to come back.

Never send three variations of the same sentence. If you cannot think of a genuinely different
second message, one attempt is better than three.

## Fields are not memory

The conversation history already remembers what was said. A field exists because the **business**
needs the value stored — not so the agent can recall it three turns later. Capturing fields for
recall is the most over-used pattern in the product and it makes conversations rigid for no benefit.
See `cortex-info-collection`.

## Reviewing an existing Cortex

Read the System Instructions and every Conversation Goal together, and look for:

1. **Identity repeated** in any Conversation Goal → move it up, delete the copies
2. **Instructions the agent cannot act on** → re-home them per the table above
3. **`/{keyword}` used as a variable** → rewrite
4. **A Conversation Goal with several objectives** → probably several nodes
5. **A tool whose response is never explained** → add an `@[ToolName]` line

## When something looks wrong

- **"It greets people by a placeholder."** The client has no value for that field. Rule 3.
- **"It ignores my instruction to save something."** Prompts cannot save. Configure collection.
- **"It repeats its introduction at every step."** Identity duplicated in Conversation Goals.
- **"It won't use the document I named."** Retrieval is automatic and prompt-invisible — improve the
  file's description instead.
- **"It behaves inconsistently between similar conversations."** Usually System Instructions and a
  Conversation Goal pulling in different directions.
