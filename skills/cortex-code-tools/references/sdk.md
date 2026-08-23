# Atom code-tool SDK reference

Everything available inside the sandbox. No imports needed. Read the sections you need.

- [Utilities](#utilities)
- [Maths helpers](#maths-helpers)
- [Platform functions](#platform-functions)
- [Toolkits (integrations)](#toolkits-integrations)
- [Available built-ins](#available-built-ins)
- [Blocked globals](#blocked-globals)
- [Images and files](#images-and-files)
- [Errors and timeouts](#errors-and-timeouts)
- [Worked examples](#worked-examples)

## Utilities

### `http`

An axios instance. **The response body is in `.data`** — a mistake worth stating twice, because
`await http.get(url)` returns the response envelope, not the payload.

```js
const response = await http.get("https://api.example.com/product/123");
const producto = response.data;   // the JSON is HERE
```

`http.get` · `http.post` · `http.put` · `http.delete`

- User-Agent `AtomChat-Agent/1.0`
- 15 second timeout
- Maximum 5 redirects
- Maximum 1 MiB response — a larger payload fails rather than truncating

Never hardcode a sensitive token here: code is visible to anyone who can edit the Cortex.

### `log(message)`

Writes to the platform logs and the `test_code_tool` result. **Never visible to the customer.** The
main debugging instrument — use it liberally, especially to inspect an unfamiliar response shape:

```js
log(JSON.stringify(resultado));
```

### `wait(ms)`

Pauses execution. `setTimeout` and `setInterval` are blocked, so this is the only delay available.
Counts against `timeoutSeconds`.

## Maths helpers

Atom-specific shortcuts. Full `Math` (`Math.pow`, `Math.sqrt`, `Math.max`…) is also available.

| function | returns | notes |
|---|---|---|
| `mean(...values)` | number | Ignores non-numeric values. No valid values → `0` |
| `median(...values)` | number | No valid values → `0` |
| `percentage(part, total)` | number | `total` of `0` → `0`, not `NaN` |
| `tax(amount, rate?)` | number | `rate` is a percentage; defaults to 21 |

```js
mean(10, 20, 30);        // 20
percentage(50, 200);     // 25
tax(1000);               // 210
tax(1000, 19);           // 190
```

The zero-guards matter: these return `0` rather than `NaN`, so a bad input produces a wrong number
rather than a crash. Validate inputs you don't trust.

## Platform functions

Real business logic against Atom's services. **None of them ever throws.** With no client or
conversation in scope they silently do nothing — which is what makes a code tool safe to run in a
simulation, and also why a tool can appear to "do nothing" when it is in fact working as designed.

| function | returns | effect |
|---|---|---|
| `tipify(keyword)` | `Promise<void>` | Typifies the conversation. No-op without a conversation. |
| `setStage(keyword)` | `Promise<void>` | Assigns a funnel stage by keyword. **Unknown keyword → silent warning.** |
| `setField(key, value)` | `Promise<void>` | Stores a client field. Non-strings are `JSON.stringify`d. No-op without a client. |
| `setTag(keyword)` | `Promise<void>` | Tags the client. No-op without a client. |
| `getFields(...names)` | `Promise<object \| null>` | Reads client fields. No arguments → all of them. |

Keywords must exist in the company catalog — check with `list_catalog` (`kind: "stages"`,
`"typifications"`, `"tags"`, `"info_fields"`).

### `getFields` returns a dictionary

The most misread function in the SDK. It does **not** return values positionally:

```js
const datos = await getFields("nombre", "email");

// datos is NOT ["Juan Pérez", "juan@mail.com"]
// datos IS  { nombre: "Juan Pérez", email: "juan@mail.com" }

log(datos.nombre);
```

It returns `null` when there is no recognised client, or when none of the requested fields exist.
Check before dereferencing:

```js
const datos = await getFields("nombre", "email");
if (!datos) return { ok: false, motivo: "sin cliente" };
```

In `test_code_tool` it **always** returns `null` — there is no client in a test.

## Toolkits (integrations)

Present as the global `toolkits` when the Cortex has connected integrations.

```js
toolkits.<toolkitSlug>.<ActionInPascalCase>(args?)
```

The slug is lowercase (`googlecalendar`, `gmail`, `slack`); the PascalCase action maps to Composio's
`TOOLKIT_ACTION_NAME`, so `toolkits.googlecalendar.CreateEvent(...)` calls
`GOOGLECALENDAR_CREATE_EVENT`. Omitted `args` sends `{}`.

Returns whatever that Composio action returns — the shape differs per integration, so log it once
before relying on field names.

```js
const evento = await toolkits.googlecalendar.CreateEvent({
  summary: "Llamada de seguimiento",
  start: { dateTime: params.fechaISO },
});

const mensajes = await toolkits.gmail.ListMessages();

try {
  await toolkits.slack.PostMessage({ channel: "#ventas", text: "Nuevo lead" });
} catch (err) {
  log("Slack falló: " + err.message);
}
```

### Escape hatch

For an action whose slug doesn't follow the mapping:

```js
await toolkits.execute("GMAIL_SEND_EMAIL", { to: "...", subject: "...", body: "..." });
```

### Two rules

**Guard when it might be absent.** `toolkits` is `undefined` outside a Composio scope — including in
a `test_code_tool` run without `companyId` and `agentId`:

```js
if (typeof toolkits === "undefined") {
  return { ok: false, error: "No hay integraciones conectadas" };
}
```

**Never index dynamically.** Publishing statically scans the code for `toolkits.<slug>` and blocks
if a referenced toolkit isn't connected. `toolkits[algunaVariable].Foo()` defeats that scan, so a
missing connection reaches production undetected.

## Available built-ins

`Math` `JSON` `Object` `Array` `String` `Number` `Boolean` `Date` `Promise` `parseInt` `parseFloat`
`isNaN` `isFinite` `Error` `RegExp` `Map` `Set` `NaN` `Infinity` `undefined`

## Blocked globals

All return `undefined`:

| blocked | instead |
|---|---|
| `require` | No filesystem or module access |
| `process` | No access to the Node process |
| `console` | `log()` |
| `setTimeout` `setInterval` `setImmediate` | `wait(ms)` |
| `eval` `Function` | No dynamic execution |
| `global` `globalThis` | No host scope |

## Images and files

Not a code-tool feature — a general preprocessing step applied to every tool's arguments.

Incoming images and files are stored in the conversation as `[image-url:URL]` / `[file-url:URL]`, and
the model sees a short aliased URL (`img.atomchat.io/<token>`) so it cannot corrupt a long signed
URL by copying it.

Before your code runs, any argument that is one of those aliases is **resolved back to the real,
fetchable signed URL**. `main(params)` receives the real URL; you never resolve an alias yourself.

A code tool **cannot produce a file as output.** It returns data. If a result contains a file URL,
it is the agent — not the tool — that presents it to the customer.

## Errors and timeouts

| situation | result |
|---|---|
| No `main` (empty code) | `No main function defined in your code` |
| Syntax error | `Code tool "<name>" compilation failed: <message>` |
| Exception inside `main` | `Code tool "<name>" failed: <message>` |
| Exceeds `timeoutSeconds` | `Code execution timed out after <ms>ms` |
| `http` 4xx/5xx or timeout | `Request failed with status code <code>: <up to 500 chars of body>` |
| Platform function with no client | Silent no-op — never fails the tool |
| `toolkits.*` provider error | Rejects normally — catch it |

**What the agent sees.** The error arrives as a tool message reading `Error: <message>`, and the
engine appends an instruction to tell the user the tool could not complete and continue the
conversation normally. This is deliberate — a tool failure should not be read as a routing signal or
end the conversation. The practical consequence: **a broken tool looks like a vague reply, not a
crash.** Check the trace.

Beyond the per-tool timeout there is a **120 second cap across every tool call in a single turn**.
Exceed it and all tools in that round return `{ error: "tool_timeout", tool: "<name>" }` without
running.

## Worked examples

### Calculation

```js
const { monto, meses } = params;
const iva = tax(monto);
const total = monto + iva;
const cuota = total / meses;

log("Cuota calculada: " + cuota);

return { monto, iva, total, meses, cuota, cuotaFormateada: "$" + cuota.toFixed(2) };
```

### API call with a guard

```js
const { productoId } = params;

const response = await http.get("https://api.mitienda.com/productos/" + productoId);
const producto = response.data;

if (!producto.stock || producto.stock < 1) {
  return { disponible: false, mensaje: "Producto sin stock" };
}

const iva = tax(producto.precio, producto.tasa_iva || 21);

return {
  disponible: true,
  nombre: producto.nombre,
  precio: producto.precio,
  precioFinal: producto.precio + iva,
  stock: producto.stock,
};
```

### Parsing an array parameter

```js
// `productos` is declared type `array`, so it arrives as a JSON string.
const productos = JSON.parse(params.productos);

const precios = productos.map((p) => Number(p.precio ?? p));
const promedio = mean(...precios);

return {
  promedio: "$" + promedio.toFixed(2),
  mediana: "$" + median(...precios).toFixed(2),
  rango: { min: Math.min(...precios), max: Math.max(...precios) },
  sobrePromedio: percentage(precios.filter((p) => p > promedio).length, precios.length).toFixed(1) + "%",
};
```

### A whole process in one tool

Fetch, decide, record, return — the shape a code tool exists for.

```js
const { tipoConsulta } = params;

const cliente = await getFields("nombre", "email", "ultima_compra");

if (tipoConsulta === "reclamo") {
  await tipify("reclamo");
  await setTag("urgencia-alta");
  await setStage("gestion_reclamo");
  return { accion: "reclamo", prioridad: "alta", cliente };
}

if (tipoConsulta === "consulta") {
  await tipify("consulta_producto");
  await setStage("info_producto");
  return { accion: "consulta", cliente };
}

await tipify("otro");
return { accion: "derivar", cliente };
```

### Integration with a guard and error handling

```js
const { titulo, fechaISO } = params;

if (typeof toolkits === "undefined") {
  return { ok: false, error: "No hay integraciones conectadas en este flujo" };
}

try {
  const evento = await toolkits.googlecalendar.CreateEvent({
    summary: titulo,
    start: { dateTime: fechaISO },
  });
  await setTag("reunion-agendada");
  return { ok: true, eventId: evento.id };
} catch (err) {
  log("Fallo al crear evento: " + err.message);
  return { ok: false, error: err.message };
}
```
