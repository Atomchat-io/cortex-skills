---
name: cortex-http-tools
description: Build and test HTTP tools for a Cortex agent — handlebars parameters versus stored fields, auth, saving response values, and the test_http_tool loop. Use this whenever a Cortex needs to call an external API or REST endpoint, whenever someone wants a Cortex to look something up in another system, and whenever an HTTP tool is being called with the wrong values or its response is being ignored.
---

# HTTP tools

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — they need a `CORTEX_MCP_URL` and a `CORTEX_MCP_KEY` from whoever administers the Atom
> project, then an MCP entry in their agent's config. `cortex-agent-building` has the per-agent
> commands. Advising on an agent you cannot read is worse than saying you are not connected.

An HTTP tool is **one outbound request**. That is its whole scope, and its strength: no code to
maintain, configured entirely in the tool definition.

The moment you need two calls, a condition, or any computation between steps, you want a code tool
instead — see `cortex-code-tools`. Do not build a chain of HTTP tools and hope the model sequences
them correctly; that is a decision you can make deterministically in code.

## Parameters are not fields

The single most important idea here, and the one people get wrong most often.

Dynamic values come from **`{{handlebars}}` placeholders**, which are **tool parameters**: the model
fills them at call time from the conversation.

```
GET https://api.example.com/productos/{{productoId}}/stock
```

`productoId` is not stored. It does not exist in the field catalog. It does not need to be captured
from the customer with a required question. The model reads it from what was said and passes it in.

The anti-pattern — and it is extremely common among people arriving from other tooling — is:

> capture a field → require it before the node can advance → read the field back → feed the request

That adds a mandatory question, a catalog entry and a blocking gate, all to move a value the model
already had in front of it.

**If the value should also be stored**, that is a separate decision: add it to passive collection at
the End node. Passing a value and storing a value are unrelated. See `cortex-info-collection`.

`{{handlebars}}` work in the URL, headers and body template.

## `/{keyword}` is different

You can also interpolate a value the client **already has** on record:

```
GET https://api.example.com/clientes/{{documento}}?asesor=/{asesor_asignado}
```

`/{keyword}` is read-only, resolved once when the conversation starts, and renders a **default** when
the client has no value — it cannot signal absence. Use it for stable context, never for something
gathered during this conversation. See `cortex-prompts`.

## Writing the description

The description is the **only** thing the model reads when deciding whether to call the tool. It
outweighs anything you write in a prompt.

Say what it does *and when to reach for it*:

- Good: `Checks live stock and price for a product code. Use when the customer asks whether
  something is available or what it costs.`
- Bad: `Product API`

Same for every parameter. `productoId` deserves `The product code the customer mentioned, e.g.
"A-1024"` — a bare `id` invites the model to pass whatever is nearest.

## Tell the agent what the response means

The tool returns data; nothing interprets it. If the response should change behaviour, say so in the
Conversation Goal, referencing the tool:

```
Consulta @[VerificarStock] antes de prometer disponibilidad.
Si disponible es false, ofrece las opciones de alternativas en lugar de disculparte.
```

This is the fix for the most-reported "bug" in the product: a tool returns structured data — a list
of items, some flagged required — and the agent ignores the flags, because nobody told it the flags
existed. See the worked example in `cortex-info-collection`.

## Saving values from the response

`saveResponseFields` maps a JSON path in the response to a field, writing it without the agent being
involved:

| field | meaning |
|---|---|
| `jsonPath` | Path into the response body, e.g. `data.orderId` |
| `targetField` | A **real** keyword from `list_catalog` (`kind: "info_fields"`) |
| `label` | Human-readable name |

Use it for values the business needs kept — a booking reference, an order number. An invented
`targetField` writes nowhere, silently.

## Auth and configuration

| setting | notes |
|---|---|
| `auth.type` | `none`, `bearer`, `basic`, or `api-key` |
| `successCodes` | Which statuses count as success; anything else surfaces as an error |
| `timeoutSeconds` | 1–120. Lower it for anything that should fail fast. |
| `headers` | Static headers; `{{params}}` and `/{keyword}` both interpolate |
| `bodyTemplate` | Request body, with the same interpolation |

**Credentials are visible to anyone who can edit the Cortex.** Use a scoped service credential, never
a personal token, and never one with more access than this single endpoint needs.

## Testing with `test_http_tool`

Test before wiring the tool into a node. Debugging a wrong URL through simulated conversations is
far slower than calling it directly.

```
test_http_tool(
  config: {
    id: "verificar-stock",
    name: "VerificarStock",
    description: "Consulta stock y precio de un producto.",
    method: "GET",
    url: "https://api.mitienda.com/productos/{{productoId}}/stock",
    auth: { type: "bearer", token: "..." },
    successCodes: "200"
  },
  testValues: { productoId: "A-1024" }
)
```

**It sends a real request.** A `POST` that creates a record will create one — use a sandbox endpoint
or a safe payload for anything non-idempotent.

A working loop:

1. `test_http_tool` until the URL, auth and response shape are confirmed
2. Write the tool onto the node with `update_agent`
3. Say in the Conversation Goal when to call it and what its response means
4. `simulate_turn`, then check the `ToolCall` entry in the trace — was it called at all, and with
   what arguments?
5. Check the agent actually used the result

Steps 4 and 5 fail for different reasons and have different fixes: not called means the tool
description is weak; called-and-ignored means the Conversation Goal is.

## When something looks wrong

- **"The tool never gets called."** The description doesn't say *when* to use it. This is nearly
  always the cause. No `ToolCall` in the trace confirms it.
- **"It calls it with the wrong values."** Parameter descriptions are too thin.
- **"It got the data and ignored it."** Nothing told the agent what the response means.
- **"Works in the test, not in a conversation."** The value you hardcoded as `testValues` isn't
  arriving as a parameter. Check the arguments on the `ToolCall` entry.
- **"Nothing was saved."** The `targetField` keyword doesn't exist in the catalog.
- **"The conversation carried on as if nothing failed."** By design — a tool error tells the agent
  to inform the user and continue, so a broken endpoint reads as a vague reply rather than an error.
  Check the trace. See `cortex-simulation`.
