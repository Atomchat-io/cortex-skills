---
name: cortex-rag
description: Give a Cortex agent knowledge — attaching files and dynamic tables, writing descriptions that make retrieval work, and why naming a document in a prompt does nothing. Use this whenever a Cortex needs to answer from documentation, policies, prices, stock or catalogs, and whenever a Cortex fails to answer something that is definitely in a document.
---

# Knowledge: files and dynamic tables

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — they need a `CORTEX_MCP_URL` and a `CORTEX_MCP_KEY` from whoever administers the Atom
> project, then an MCP entry in their agent's config. `cortex-agent-building` has the per-agent
> commands. Advising on an agent you cannot read is worse than saying you are not connected.

Two ways a Cortex answers from data it was not told directly.

| | what it is | right for |
|---|---|---|
| **Files** | Documents in Atom's file system, vectorized at upload | Policies, FAQs, manuals, brochures — prose |
| **Dynamic Tables** | Tabular sources (API, spreadsheet) refreshed on a schedule | Prices, stock, catalogs, schedules — rows that change |

The rule of thumb: if the answer is a **passage**, it is a file. If it is a **row or a number that
changes**, it is a dynamic table. Putting a price list in a PDF is the classic wrong choice — it goes
stale and retrieves badly.

## How retrieval actually works

**Retrieval happens before the agent's turn, and the results are placed into its context
automatically.**

The agent never "decides to go and look something up" in any way you can instruct. By the time it is
writing a reply, the material is either already in front of it or it is not.

This is why these do nothing:

- ❌ `Busca la respuesta en la lista de precios.`
- ❌ `Si no lo sabes, consulta la base de conocimiento.`
- ❌ `Revisa el catálogo antes de responder.`

You have exactly two levers: **which sources are attached to the node**, and **how well they are
described**.

## Attaching

Get real ids — never invent one, since a made-up id silently retrieves nothing:

```
list_catalog(companyId, kind: "files")
list_catalog(companyId, kind: "dynamic_tables")
```

Then write them onto the agent node with `update_agent`: `knowledgeBases` for files (an array),
`dynamicTableId` for the table (**one per node**).

Attach only what is relevant **at that point in the conversation**. A node carrying every document
the company owns retrieves worse than one carrying the three that matter — more material to sift,
and a slower node. If different parts of the conversation need different knowledge, that is a
legitimate reason to split nodes. See `cortex-agent-building`.

Uploading files and creating dynamic tables happens in the Atom UI. These tools list and attach.

## Descriptions do the work

A source's **description** is what makes it findable. Write what it contains and what questions it
answers:

| | |
|---|---|
| ✅ | `Política de cancelación y reembolsos: plazos de aviso, penalizaciones y cómo se devuelve el dinero.` |
| ✅ | `Catálogo de productos con precio, stock disponible y tiempo de entrega por referencia.` |
| ❌ | `Documento de políticas` — matches almost nothing |
| ❌ | `Info` — matches everything, so it is retrieved for unrelated questions |

Both failure directions are real. Too vague and it never surfaces; too broad and it crowds out the
source that actually had the answer.

If a Cortex retrieves the wrong document, the fix is nearly always in the descriptions rather than
the files.

## What the agent reads directly

Separate from retrieval, and often confused with it:

- The agent **natively understands images and audio** a customer sends.
- It can **send images and files** when it has a URL — the platform presents them properly.
- It **cannot read PDFs or other document formats sent in a conversation.** A customer attaching a
  PDF is not something the agent can parse. That needs a tool that extracts the text — see
  `cortex-code-tools`.

Files in the knowledge base are a different mechanism entirely: they were vectorized at upload, not
read live.

## Testing retrieval

Retrieval is invisible in the reply, so test it through the trace:

1. `simulate_turn` with a question whose answer is in the source
2. Look for `KnowledgeBaseSearch` or `DynamicTableQuery` in `agentBuilderLogs`
3. Check `kbRetrieved` / `dtRetrieved` in the response

Three outcomes, three different fixes:

| what you see | meaning | fix |
|---|---|---|
| No log entry | Retrieval never ran for that message | Nothing attached, or the question did not read as one needing lookup |
| Entry, nothing useful retrieved | It searched and missed | Description too vague, or the content is not really there |
| Entry, wrong source retrieved | Descriptions overlap | Sharpen them, or attach fewer |

Ask the question several different ways. A source that only surfaces for one exact phrasing will
fail for real customers.

## When something looks wrong

- **"It doesn't know something that's in the document."** Check the trace first — no
  `KnowledgeBaseSearch` is a completely different problem from a search that found nothing.
- **"It gives stale prices."** Dynamic tables refresh on a schedule. Check when it last ran, and
  whether that data should be a table rather than a file.
- **"It ignores the file I named in the prompt."** Prompts cannot direct retrieval. Attach the file.
- **"It answers from the wrong document."** Too many sources attached, or two descriptions overlap.
- **"It can't read the PDF the customer sent."** Correct — that is not supported. Retrieval only
  covers files uploaded to the knowledge base.

### Mechanics — recognise, never reference

A classifier runs on each incoming message and decides two things: **whether** to retrieve at all,
and what to search for. It rewrites the customer's message into a self-contained query, which is why
a follow-up like "¿y el otro?" can still retrieve correctly.

The practical consequence is the one in the table above: *retrieval never ran* and *retrieval ran and
found nothing* look identical from outside, and only the log entries distinguish them. Never mention
the classifier in a prompt — it runs before the prompt is used.
