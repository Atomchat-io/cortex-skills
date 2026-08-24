---
name: cortex-simulation
description: Test a Cortex agent with simulate_turn and diagnose it by reading the execution trace — what every agentBuilderLogs entry means and which symptom maps to which cause. Use this after every change to a Cortex, whenever a Cortex behaves unexpectedly, and whenever someone asks why an agent said or did something.
---

# Simulating and diagnosing

> **Cortex tools required.** This skill assumes the Cortex MCP server is connected. If tools like
> `list_agents` and `describe_agent_schema` are not available to you, stop and tell the user how to
> connect it — they need the server URL and their personal key from whoever administers the Atom
> project, then an MCP entry in their agent's config. `cortex-agent-building` has the per-agent
> commands. Advising on an agent you cannot read is worse than saying you are not connected.

`simulate_turn` plays one customer message against a Cortex and returns its reply **plus the trace of
what actually happened**.

```
simulate_turn(companyId, agentId, message)             → new conversation, returns a sessionId
simulate_turn(companyId, agentId, message, sessionId)  → continue that conversation
```

Omit `sessionId` to start fresh. There is no reset tool — a new conversation is a call without one.

## Read the trace, not the reply

This is the habit that separates useful testing from theatre.

A plausible reply can come from a Cortex doing entirely the wrong thing: retrieving nothing, skipping
a tool, sitting in a node you thought it had left. The customer-facing text looks fine and the Cortex
is broken. The reply tells you how it sounded; `agentBuilderLogs` tells you what it did.

| log | means |
|---|---|
| `NodeTransfer` | Moved to another node |
| `GateBlocked` | A transition was refused — required fields missing. **Names them.** |
| `ToolCall` | A tool ran |
| `KnowledgeBaseSearch` | Files were searched |
| `DynamicTableQuery` | A dynamic table was queried |
| `InfoFieldSaved` | A field was captured |
| `PipelineStageEntered` | A funnel stage was recorded |
| `TypificationApplied` / `TagApplied` | End-of-conversation analysis |
| `FlowEnd` | Finished. Names the Exit Port; `isError` marks an abnormal end. |
| `ResponseValidationFailed` | A rich format block was malformed |
| `ResponseAutoRepaired` | Malformed, and fixed automatically |
| `ResponseFallbackApplied` | Could not be fixed — fell back to text |
| `ResponseAdaptedByLimit` | Exceeded a WhatsApp limit and was reshaped |
| `WhatsappFlowSent` | A form was sent |
| `GuardrailTriggered` | A guardrail fired |

Also returned: `currentNodeId`, `collectedInfo`, `exitPort`, `finished`, `segments` (each with the
`format` actually produced), and `kbRetrieved` / `dtRetrieved`.

**An absent log is evidence.** No `KnowledgeBaseSearch` means retrieval never ran — a different
problem, with a different fix, from retrieval running and finding nothing.

## Simulation touches nothing real

No client record, no stage history, no tags, no typification. The tool has no parameter for a client
or a conversation, so it cannot.

Two consequences worth internalising:

- **Passive collection is not exercised.** You can see what *would* have been captured in
  `collectedInfo` and the trace, but nothing is written. Confirming data truly lands is a separate
  exercise with a real conversation.
- **Code tool platform calls are captured, not executed** — see `cortex-code-tools`.

Simulation runs against the Cortex **as currently saved**. There is no way to test unsaved changes:
save, then simulate. If the Cortex is published, that save is already live.

## How to test well

The happy path is the one path that was designed. It is the least informative thing you can try.

**1. The happy path**, once, to confirm it works at all.

**2. Every branch.** Each fork exists because someone expected a different kind of customer. Play
each one. A branch you cannot reach in simulation, real customers cannot reach either.

**3. The uncooperative customer.** This is where the real bugs are:
   - refuses a required field → does the gate hold, and is the refusal handled gracefully?
   - changes their mind mid-flow → can it move back?
   - asks something off-topic → does it recover, or derail?
   - answers two questions at once → does it double-ask?

**4. The vague opener.** Just `hola`. Many Cortexes fall apart when the first message carries no
intent, because every transition condition was written assuming one.

**5. Every End node.** Confirm each is reachable and exits with the port you expect — Flowbuilder
routes on that.

Between runs, change **one** thing. Two changes and a different outcome tells you nothing about
either.

## A worked diagnosis

> "The agent asks for the email twice."

1. `simulate_turn` through to the repeat.
2. Look at the trace around the second ask. Is there a `GateBlocked`? If so, read which fields it
   names.
3. `GateBlocked` naming `email` after the customer already gave it means the value did not reach the
   transition's arguments — the customer supplied it *before* the node holding the gate, and the gate
   checks arguments, not history.
4. Fix: move the field to the node where it is actually asked, or to passive collection if it should
   not block at all.

The general shape: **symptom → trace → the specific log entry → the configuration that produced it.**
Guessing from the reply text skips the two steps that carry the information.

## Symptom table

| symptom | look for | usual cause |
|---|---|---|
| Won't move on | `GateBlocked` | Required fields; all are mandatory |
| Asks twice | `GateBlocked` | Value arrived before the gated node |
| Wrong node | `NodeTransfer` | Overlapping conditions, or an authored sibling edge |
| Ignores a document | no `KnowledgeBaseSearch` | Source not attached, or weak description |
| Searched, found nothing | `KnowledgeBaseSearch` present | Description too vague, or content absent |
| Never calls a tool | no `ToolCall` | Tool description doesn't say *when* to use it |
| Calls it, ignores result | `ToolCall` present | Conversation Goal doesn't say what the response means |
| Ends unexpectedly | `FlowEnd` with `isError` | Loop protection, or a guardrail |
| Ends at `out_of_context` | `FlowEnd` | 5 consecutive transfers — conditions let it bounce |
| Buttons missing | `ResponseAdaptedByLimit` | Over 3 buttons, or labels over 20 chars |
| Format came as text | `Response*Failed` / `Fallback` | Malformed block |
| Repeats its introduction | — | Identity duplicated in Conversation Goals |
| Inconsistent between runs | — | System Instructions and a Conversation Goal conflict |

## Before handing it back

- [ ] Every End node reached at least once, exiting with the expected port
- [ ] Every branch walked
- [ ] The uncooperative customer tried
- [ ] `hola` and nothing else tried
- [ ] No `GateBlocked` on a field that should not have been blocking
- [ ] `check_agent` shows no errors, and you have read the warnings
- [ ] If `flowbuilderImpact` is non-empty, the human knows which flows are affected

Then tell them to publish. That step is theirs — you cannot do it, deliberately.

### Mechanics — recognise, never reference

`isSimulation` is derived from the absence of a conversation and only controls analytics logging. The
absence of a client is what prevents record writes. Neither is configurable — they follow from the
tool not exposing those parameters at all.
