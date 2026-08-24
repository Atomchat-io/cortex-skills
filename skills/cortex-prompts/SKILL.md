---
name: cortex-prompts
description: Write System Instructions and Conversation Goals for a Cortex agent — what belongs in each, referencing tools with @[ToolName], and the /{keyword} interpolation rules. Use this whenever writing, reviewing or debugging any Cortex prompt text, whenever a Cortex repeats itself or ignores an instruction, and whenever someone asks why a name comes out as a placeholder.
---

# Writing Cortex prompts

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — they need the server URL and their personal key from whoever administers the Atom
> project, then an MCP entry in their agent's config. `cortex-agent-building` has the per-agent
> commands. Advising on an agent you cannot read is worse than saying you are not connected.

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

Per agent node. The objective of this step and the signal it is complete.

```
Averigua qué servicio necesita el cliente y si ya ha venido antes.
Pregunta una cosa a la vez.
Cuando tengas ambos datos, continúa.
```

Short is correct. A good Conversation Goal is often three or four lines.

**Never restate identity, tone or business context.** It is already inherited, and duplicating it
guarantees the two drift apart until they contradict each other — at which point the agent's
behaviour depends on which one it weighted, and you cannot reason about it.

Two smells:

- **"and then"** in a Conversation Goal usually means two nodes.
- **A Conversation Goal longer than the System Instructions** means something is in the wrong place.

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
| A tool called | The tool's own description |
| Moving to another node | The edge's condition — `cortex-graph-schema` |

Writing these into a prompt is not merely useless — it spends the agent's attention on instructions
it cannot follow, and makes the real prompt harder to follow.

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

## `/{keyword}` — read-only interpolation

Drop a value the client already has into prompt text:

```
Saluda al cliente por su nombre: /{name}
```

Get real keywords from `list_catalog` with `kind: "info_fields"` — it returns the exact `/{keyword}`
string to paste. Never guess one; an unknown keyword simply does not resolve.

Three rules, all regularly violated:

**1. It is read-only.** There is no write form:

```
❌ Guarda la respuesta en /{presupuesto} y luego usa /{presupuesto} para calcular.
```

Neither half works. The first does nothing; the second interpolates whatever was on the record when
the conversation began.

**2. Resolved once, at conversation start.** A value written mid-conversation will not appear. The
text the agent sees is fixed at turn one and never re-rendered.

**3. A missing value renders as the default** — not blank, not an error. There is no way to ask
whether the client has a value, so a prompt written assuming it is populated will cheerfully address
someone by a placeholder.

Two workarounds:

- **Tell the Conversation Goal what the default means:**
  ```
  El nombre del cliente aparece como /{name}. Si eso parece un marcador de posición
  en lugar de un nombre real, pídeselo antes de continuar.
  ```
- **Use a code tool** with `getFields(...)`, which reports what actually exists, when the branch
  genuinely depends on it. See `cortex-code-tools`.

## Recovery messages — writing for a silent customer

A Cortex can send **proactive messages when the customer stops replying**. Up to three attempts,
each after a delay you set, each with its own message.

If the customer still does not answer, one more timer runs — the **close timeout**, also configured
on the Cortex — and then the Cortex signals Flowbuilder, which routes the conversation through its
**inactivity-close exit**. That exit is a Flowbuilder branch, **not** one of your End nodes, so it
does not appear among your Exit Ports and nothing in the Cortex runs on that path. Whatever should
happen to an abandoned conversation belongs to that branch.

Any customer reply at any point cancels the whole chain, and the close is skipped if the
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
