---
name: cortex-code-tools
description: Write and test code tools for a Cortex agent — the JavaScript sandbox, defining parameters, the Atom SDK, calling integrations from code, and the test_code_tool loop. Use this whenever a Cortex needs custom logic, a multi-step process, calculations, conditional behaviour, or anything combining an API call with decisions — and whenever someone asks why a code tool works in testing but fails in a real conversation.
---

# Code tools

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — run the `cortex-setup` skill, which walks through getting a URL and key and writing
> the config for their agent. Advising on an agent you cannot read is worse than saying you are not
> connected.

A code tool is a JavaScript function a Cortex can call. It runs in Atom's sandbox with HTTP access,
the connected integrations, and platform functions for writing to the client record.

**This is the powerful option.** When a task involves logic, several steps, conditions, or
coordination between systems, the code tool is what you reach for. Do not push that complexity out
to an external API just to keep the tool small — writing it here is the point.

## Choosing between a code tool and an HTTP tool

| The task | Use |
|---|---|
| One request to an existing endpoint, no logic | **HTTP tool** — see `cortex-http-tools` |
| Several calls, or a decision between them | **Code tool** |
| Fetch, compute, then store | **Code tool** |
| Anything conditional | **Code tool** |
| Combining an integration with an API and a calculation | **Code tool** |

The one case where an HTTP tool wins is genuinely a single call. The moment you write an `if`, you
want a code tool.

**Split into separate code tools only when the choice between them is a decision the model makes
from the conversation** — `book_appointment` versus `cancel_appointment`. Steps within one process
belong in one tool: your code sequences them deterministically, where the model would have to guess
the order.

## Defining parameters

Parameters are how the model passes data into your code. Each has four parts:

| field | notes |
|---|---|
| **name** | `[a-zA-Z0-9_]`, up to 64 characters. This is the key in `params`. |
| **type** | `string`, `number`, `boolean`, `array`, or `object` |
| **description** | Up to 500 characters. **This is what the model reads to decide what to pass.** |
| **required** | Whether the model must supply it |

Up to 20 per tool.

The description does the real work. `productId` alone tells the model nothing; `The product code the
customer mentioned, e.g. "A-1024"` tells it exactly what to look for in the conversation. Thin
descriptions are the most common reason a tool is called with wrong values.

**Parameters are not fields.** A parameter is supplied at call time from the conversation and is not
stored anywhere. Reach for a parameter first; only add a field when the business needs the value
kept, and then add it to passive collection separately. See `cortex-info-collection`.

## How parameters arrive

Your code receives a frozen `params` object:

```js
const { monto, meses } = params;
```

Three behaviours that cause real bugs:

**1. An absent optional parameter is missing, not `undefined`.** If `required: false` and the model
didn't supply it, the key is not on the object:

```js
const descuento = "descuento" in params ? params.descuento : 0;
const notas = params.notas ?? "";
```

**2. `array` and `object` parameters arrive as JSON strings.** Only `number` and `boolean` come
through with their real type. Everything else behaves like a string:

```js
const items = JSON.parse(params.items); // params.items is '["a","b"]', not ["a","b"]
```

**3. The test panel does not reproduce behaviour 2.** It hands you a native `[]` or `{}`. A tool
that passes its test can still fail in production for exactly this reason — so when testing an
array or object parameter, **pass it as a JSON string**, which is what a real conversation delivers.

`params` is frozen, so you cannot mutate it. Copy first if you need to normalise values.

## Writing the code

The editor owns the function signature; you write the body of `main(params)`:

```js
const { monto, meses } = params;
const total = monto + tax(monto);

log("cuota calculada para " + meses + " meses");

return { cuota: total / meses, total, meses };
```

**Return objects, not bare strings.** Named fields let the model use individual values; a formatted
sentence forces it to re-parse your prose.

The tool is exposed to the model as `code_<name>`, and its **description** is the only thing the
model reads when deciding to call it. Say what it does and when to use it — `Calculates the monthly
instalment including tax, given an amount and a number of months`, not `Does calculations`.

## The SDK, in brief

Available without imports. Full reference with signatures and edge cases:
[`references/sdk.md`](references/sdk.md) — read it when you need exact behaviour.

```js
const res = await http.get(url);   // axios. THE BODY IS IN res.data
log("message");                     // debug output, never seen by the customer
await wait(500);                    // setTimeout is blocked
tax(1000);                          // 210 — also mean, median, percentage, and full Math
```

Platform functions write to the client record. They never throw; without a client or conversation in
scope they silently do nothing:

```js
await setField("presupuesto", 5000);
await setStage("cotizacion_enviada");   // silent warning if the keyword doesn't exist
await tipify("consulta_precio");
await setTag("lead-caliente");

const datos = await getFields("nombre", "email");
// A DICTIONARY, not a list: { nombre: "...", email: "..." }
// null when there is no client or none of those fields exist — check before reading.
```

`getFields` is also the only reliable way to discover whether a client **has** a value. `/{keyword}`
interpolation renders a default when the value is absent and cannot tell you it was missing, so when
behaviour must depend on what exists, check it here. See `cortex-prompts`.

## Calling integrations

```js
const evento = await toolkits.googlecalendar.CreateEvent({
  summary: "Llamada de seguimiento",
  start: { dateTime: params.fechaISO },
});
```

Lowercase slug, PascalCase action. `toolkits.execute("GMAIL_SEND_EMAIL", {...})` when you know the
exact Composio slug.

- **`toolkits` is `undefined`** when the Cortex has no connected integrations. Guard with
  `typeof toolkits !== "undefined"` if the tool can run without them.
- Provider errors reject — wrap in `try/catch`.
- Response shapes differ per action. `log(JSON.stringify(result))` the first time rather than
  guessing at field names.
- Publishing **statically scans** for `toolkits.<slug>` and blocks on anything unconnected. The scan
  cannot follow `toolkits[variable]`, so avoid that — it defeats a check that would otherwise catch
  a broken Cortex before it goes live.

See `cortex-integrations` for connecting toolkits.

## Testing with `test_code_tool`

Do not write a code tool into a node and then debug it through simulated conversations. Test it
directly first — the loop is seconds instead of minutes.

```
test_code_tool(
  code:        "const { monto, meses } = params; ...",
  parameters:  [{ name: "monto", type: "number", description: "...", required: true }],
  paramValues: { monto: 5000, meses: 12 },
  companyId:   "<companyId>",
  agentId:     "<cortexId>"
)
```

`companyId` and `agentId` matter: they supply the Composio scope. Without them `toolkits` is
`undefined`, so a tool that calls an integration will fail in a way that has nothing to do with your
code.

You get back the return value, `log()` output, duration, and `sdkCalls`.

What is and isn't real:

| | in a test |
|---|---|
| `setField` `setStage` `tipify` `setTag` | **Captured, not executed.** Shown in `sdkCalls`. |
| `getFields` | Always returns `null` — there is no client |
| `http` | **Real request.** Careful with anything non-idempotent. |
| `toolkits.*` | **Real call**, when scope is supplied |

A working loop:

1. `test_code_tool` with representative values until the return value is right
2. Test an array or object parameter **as a JSON string** — the one production behaviour the test
   does not mirror
3. Write the tool into the node with `update_agent`
4. `simulate_turn` and confirm the `ToolCall` entry shows it being called with sensible arguments
5. Confirm the agent does something useful with the result — if not, the Conversation Goal needs to
   say what the response means

## Tell the agent what the response means

A tool returns data. Nothing tells the agent what to do with it unless you say so, in the
Conversation Goal, referencing the tool:

```
Call @[VerificarStock] before promising availability.
If disponible is false, offer the items in alternativas instead of apologising.
```

Without this the agent receives an object and improvises. This is one of the most common causes of
"the tool ran but the agent ignored it".

## Limits

50 code tools per node · 20 parameters · 50,000 characters of code · `timeoutSeconds` 1–120
(default 120) · a 120s cap across **all** tool calls in a turn · `http` 15s timeout, 1 MiB response.

## When something looks wrong

- **"Works in test, fails live."** Almost always the array/object JSON-string difference. Re-test
  passing the value as a string.
- **"`toolkits` is undefined."** No connected integrations, or you tested without `companyId` and
  `agentId`.
- **"`setStage` did nothing."** The keyword doesn't exist — it warns silently. Check `list_catalog`
  with `kind: "stages"`.
- **"`getFields` returned null."** No client in scope (always true in tests), or none of those
  fields exist for that client.
- **"The model never calls it."** The tool description doesn't say *when* to use it.
- **"It calls it with nonsense arguments."** Parameter descriptions are too thin.
- **"The tool failed but the conversation continued normally."** By design: errors reach the agent as
  `Error: <message>` with an instruction to inform the user and carry on. A broken tool looks like a
  vague reply, not a crash — check the `ToolCall` entry in the trace. See `cortex-simulation`.
