---
name: cortex-http-tools
description: Build and test HTTP tools for a Cortex agent — handlebars parameters versus stored fields, auth, saving response values, and the test_http_tool loop. Use this whenever a Cortex needs to call an external API or REST endpoint, whenever someone wants a Cortex to look something up in another system, and whenever an HTTP tool is being called with the wrong values or its response is being ignored.
---

# HTTP tools

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — run the `cortex-setup` skill, which walks through getting a URL and key and writing
> the config for their agent. Advising on an agent you cannot read is worse than saying you are not
> connected.

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

`{{handlebars}}` work in the **URL, headers and body template**.

**Default to parameters for everything.** Reach for a stored field only when you can say why the
value cannot come from the conversation — see the next section for what counts.

## Client fields: the exception, not the default

You *can* inject a value the client already has on record. Treat it as the exception.

A `{{parameter}}` costs nothing: the model supplies it from the conversation, it needs nothing to
exist beforehand, and the call always goes out. A field reference is a **dependency** — the value
must already be on that client's record, and if it is not, the request is never sent at all.

So the bar is high. A field reference is justified when the value is one **nobody should ever be
asked for in the conversation** and the request cannot be made without it:

- an identifier the record already carries and the customer has no reason to recite — `documento`,
  `numero_de_poliza`, a CRM id written there by an earlier automation
- an assignment the business made, not the customer — `asesor_asignado`, `sucursal`
- anything set by the system before the session opened

It is **not** justified for a value the customer will mention anyway, or one the agent could simply
ask for. "The product code" is a parameter. "The customer's own name, because we happen to store it"
is a parameter too, unless the request genuinely cannot proceed without the stored one.

**Ask the human before adding one.** "Should this read `documento` from the client record, or should
the agent ask for it?" — the same question as in `cortex-prompts`, and it has the same consequence:
reading from the record means the tool is dead for any client the record never covered.

### The syntax is not the same everywhere

When you are justified, get the syntax right — this is the most common thing to get wrong:

| where | syntax | example |
|---|---|---|
| **URL** | `:field`, immediately after a `/` | `https://api.example.com/clientes/:documento` |
| **Headers and body** | `/{field}` | `{"asesor": "/{asesor_asignado}"}` |

Using `/{field}` in a URL does nothing — it is left in the request literally. Using `:field` in a
header or body does nothing either. They are matched by different patterns.

Two details on the URL form:

- The `:field` must **follow a slash**. `?doc=:documento` is not substituted; `/doc/:documento` is.
- Its value is **URL-encoded** automatically. `{{handlebars}}` in a URL is **not** — so a parameter
  containing a space, `&` or `?` can break the request. If a parameter has to sit in a path segment,
  make sure it is a clean one.

### A missing field aborts the call

This is the important difference from prompts, and the reason for the bar above. In a prompt, an
absent `/{keyword}` is simply left in the text and the conversation carries on. In an HTTP tool,
**the request is not sent at all**:

```
This action was not executed: the required field(s) documento have no value.
```

The agent receives that message and continues the conversation. So a tool that quietly never runs is
usually a field reference for a value the client does not have — check the `ToolCall` entry in the
trace before assuming the endpoint is at fault.

Every `:field` in the URL and every `/{field}` in the headers or body is checked this way, so a tool
depending on five fields will not run until all five exist. Five field references is not a
configuration, it is a tool that will fail for most of the customer base.

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
| `headers` | `{{params}}` and `/{field}` both interpolate |
| `bodyTemplate` | Same interpolation. **Only sent for non-GET methods** — a body on a GET is ignored. |

The body is parsed as JSON after interpolation. If the result is valid JSON it is sent as JSON and
`Content-Type: application/json` is added for you; if interpolation leaves it malformed, it is sent
as a **raw string instead of failing**, which usually reaches the API as a confusing 400. When a
tool returns an unexpected 400, check the interpolated body before the endpoint.

Two guards you cannot switch off: only `http`/`https` URLs resolving to public hosts are allowed
(private and loopback addresses are rejected), and a header value containing CR/LF is rejected as
header injection. Both raise before the request is made.

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
- **"The tool silently never runs."** A `:field` or `/{field}` refers to a value this client does
  not have, so the request was never sent. The trace carries the message naming the fields. Ask
  whether that field reference was justified at all — a `{{parameter}}` would have gone out.
- **"The field placeholder came through literally."** Wrong syntax for that position — `:field` in
  the URL, `/{field}` in headers and body.
- **"The URL is malformed at runtime."** A `{{parameter}}` in the URL is not URL-encoded. Make sure
  it is a clean path segment; a value with a space, `&` or `?` needs escaping upstream.
- **"Nothing was saved."** The `targetField` keyword doesn't exist in the catalog.
- **"The conversation carried on as if nothing failed."** By design — a tool error tells the agent
  to inform the user and continue, so a broken endpoint reads as a vague reply rather than an error.
  Check the trace. See `cortex-simulation`.
