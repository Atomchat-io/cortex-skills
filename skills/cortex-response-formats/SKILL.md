---
name: cortex-response-formats
description: Rich replies from a Cortex agent — buttons, lists, carousels, audio and WhatsApp Flows, when to enable each, and the hard WhatsApp limits that silently reshape a message. Use this whenever a Cortex should reply with anything other than plain text, whenever someone wants to offer choices or show products, and whenever a carousel or buttons arrive looking wrong.
---

# Response formats

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — they need a `CORTEX_MCP_URL` and a `CORTEX_MCP_KEY` from whoever administers the Atom
> project, then an MCP entry in their agent's config. `cortex-agent-building` has the per-agent
> commands. Advising on an agent you cannot read is worse than saying you are not connected.

By default a Cortex replies with text. A node can also be allowed to reply with:

| format | what it is | right for |
|---|---|---|
| `buttons` | Up to **3** tappable replies | A small closed choice |
| `list` | A scrollable menu, up to **10** rows | More options than buttons allow |
| `carousel` | Swipeable cards: image, text, buttons | Products, services, appointment slots |
| `audio` | A spoken reply | Audiences who prefer voice |
| `whatsappFlow` | A form inside WhatsApp | Several structured values at once |

Enable formats **per node**. The right way to reply while qualifying is not the right way while
showing products, and a Cortex with carousels enabled everywhere will use them everywhere.

## Say when to use each

Enabling a format only makes it *available*. Which to use is a judgment the model makes every turn,
so it needs criteria — stated in **both** the format's own `condition` and the node's Conversation
Goal:

```
Usa botones cuando ofrezcas elegir entre dos o tres servicios.
Usa el carrusel para mostrar horarios disponibles.
En cualquier otro caso, responde con texto.
```

Leaving this out produces one of two failures, both common: every reply becomes a carousel, or the
format is never used at all. The explicit "otherwise, plain text" line is what prevents the first.

## The limits are real, and silent

WhatsApp rejects an over-limit message outright, so the platform reshapes it before sending. **You
get a different message than you designed, with no error.**

| | limit |
|---|---|
| Buttons per message | **3** |
| Button label | 20 characters |
| List rows (total, across all sections) | **10** |
| List sections | 10 |
| List row title | 24 characters |
| List row description | 72 characters |
| Carousel cards | **minimum 2** |
| Text body | 4096 characters (longer is split into several messages) |
| Interactive body (buttons) | 1024 characters |
| Header / footer | 60 characters |

What this means in practice:

- **"Offer these five plans as buttons" is a bug.** Five do not fit. Use a list.
- **A one-item carousel will not send.** Carousels need at least two cards — so a "show matching
  products" carousel breaks precisely when there is exactly one match. Say what to do then:
  `Si solo hay una opción, descríbela en texto en lugar de usar el carrusel.`
- **Long button labels get cut.** `Agendar una cita para la próxima semana` is not 20 characters.

When the options come from a tool, say how many to show:
`Muestra como máximo 3 opciones en botones; si hay más, usa una lista.`

## WhatsApp Flows

A WhatsApp Flow is a **form** — good for collecting several structured values in one interaction
rather than a long back-and-forth.

```
list_catalog(companyId, kind: "whatsapp_flows")
```

Returns published flows, with calendar flows excluded (those are triggered by their own mechanism,
never attached to a node).

Attach by `flowId`, and give each one a **prompt** explaining when to send it. Validation rejects a
flow without a prompt, and rejects the format being enabled with no flow attached — both are errors
from `check_agent`, not warnings.

Flows are built elsewhere in Atom; these tools attach existing ones.

Consider a Flow instead of node info collection when you need four or five values at once — one form
beats five turns of interrogation. Note the data still needs mapping to fields; see
`cortex-info-collection`.

## Media

- The agent **understands images and audio** a customer sends.
- It can **send an image or file** given a URL — presented properly, not as a bare link.
- It **cannot read PDFs or other document formats.** That needs a tool. See `cortex-rag`.

## Testing

Formats fail in ways the reply text alone will not reveal. After enabling one:

1. `simulate_turn` and drive the conversation to where the format should appear
2. Check `segments` in the response — the `format` field says what was actually produced
3. Check `agentBuilderLogs` for any `Response*` entry

If you asked for buttons and got `format: "text"`, something was reshaped or repaired. The log says
which.

## When something looks wrong

- **"My carousel arrived as plain text."** It failed validation and fell back. Look for
  `ResponseValidationFailed`, `ResponseAutoRepaired` or `ResponseFallbackApplied`.
- **"Some buttons vanished."** More than 3, or labels over 20 characters. Look for
  `ResponseAdaptedByLimit`.
- **"The carousel didn't send."** Fewer than 2 cards.
- **"It never uses the format."** No condition, or one the conversation never satisfies.
- **"It uses the carousel for everything."** The condition is too broad — say explicitly what to do
  otherwise.
- **"The Flow wasn't sent."** Check for `WhatsappFlowSent`, and confirm the flow is still published.

### Mechanics — recognise, never reference

Rich replies are produced by the model emitting `[FORMAT:...]` blocks, which the platform parses into
real WhatsApp components. Those blocks come from the engine's own system prompt, with validation,
auto-repair and fallback wrapped around them.

**Never mention `[FORMAT:...]` in System Instructions or a Conversation Goal.** Describing the syntax
to the model interferes with machinery that already works, and is a reliable way to produce malformed
output. Its only use to you is diagnostic: the four `Response*` log types tell you a block was
malformed and what the platform did about it.
