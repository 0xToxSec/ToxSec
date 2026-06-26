> Originally published at: https://www.toxsec.com/p/what-did-your-agent-actually-do-last

Your agent deleted something it shouldn't have at 2am. The alert fired. Now answer three questions: what did it do, why did it do it, and what did it touch. If you're grepping JSON for the next 30 minutes, you don't have tracing. You have logs with a worse UI.

This is the part nobody instruments until after the first incident. So let's instrument it before.

## Heartbeat logging vs decision tracing

Most agent logging captures the heartbeat. Agent ran. Tool called. Response returned. Everything's HTTP 200 and everything's useless.

```text
[02:14:07] agent.run        status=200
[02:14:09] tool.call  db_query     status=200
[02:14:09] tool.call  db_delete    status=200
[02:14:10] tool.call  backup_purge status=200
[02:14:11] agent.complete   status=200
```

Five green lines. Zero answers. The one thing you need is why `db_delete` fired, and that lives in the reasoning step that produced the call plus the context that fed it. Heartbeat logging throws both away before the pager goes off.

Here's the test, and it's brutal. Start from an alert. Try to jump straight to the branch where the agent picked the wrong tool, passed malformed args, or ran out of context before a critical step. If that jump takes more than a couple minutes of manual searching, you failed. You have logs, not traces.

The fix is to treat every model call, tool execution, and retrieval as its own span, with the reasoning attached as a queryable attribute. Then an investigator replays the plan instead of guessing at it.

## Why you build against OpenTelemetry GenAI, not a vendor schema

Quick shorthand check before we go further. A span is one timed unit of work with a start, an end, and attached metadata. A trace is the tree of spans for one logical operation. Instrument right and the agent run becomes a tree you can walk, not a log you scroll.

The OpenTelemetry GenAI semantic conventions give you a vendor-neutral vocabulary for exactly this. The spec is in Development status as of mid-2026, attributes still flagged experimental, but Datadog, Honeycomb, New Relic, and the big frameworks already map to it. Build against it now and your traces stay portable when you swap backends or get told to consolidate. Lock into a proprietary format and you'll be re-instrumenting under fire during your first real incident.

The conventions define the operations you care about. `invoke_agent` for an agent invocation. `execute_tool` for a tool call. Standard `gen_ai.*` attributes like `gen_ai.request.model` and the token counts. The naming is the whole point: any OTLP backend understands it without custom parse rules.

```bash
pip install opentelemetry-sdk opentelemetry-exporter-otlp
pip install opentelemetry-instrumentation-anthropic
```

Auto-instrumentation gets you LLM client spans with model and token metadata immediately, before you write a line of manual span code. That's the floor, not the ceiling.

## Capture the decision, not just the call

Auto-instrumentation gives you the heartbeat with better structure. It does not give you the why. For that you wrap the agent loop yourself and attach the reasoning and its source as attributes on the tool span.

```python
from opentelemetry import trace

tracer = trace.get_tracer("toxsec.agent")

def run_tool(tool_name, args, reasoning, context_source, risk):
    with tracer.start_as_current_span(f"execute_tool {tool_name}") as span:
        # gen_ai conventions: identify the operation
        span.set_attribute("gen_ai.operation.name", "execute_tool")
        span.set_attribute("gen_ai.tool.name", tool_name)

        # the part standard logging drops on the floor
        span.set_attribute("agent.decision.reasoning", reasoning)
        span.set_attribute("agent.decision.context_source", context_source)
        span.set_attribute("agent.decision.risk", risk)

        result = dispatch(tool_name, args)
        span.set_attribute("agent.tool.result_status", result.status)
        return result
```

`agent.decision.context_source` is the load-bearing one. When the agent does something insane, the first question is where the trigger came from. Operator instruction? Tool output? A retrieved document that quietly rewrote the objective? Poisoned context hides in that field, and if you never recorded it, your investigation is over before it starts.

One caveat the GenAI spec is loud about: do not jam full prompt bodies into span attributes. Attributes are always indexed, always exported, size-capped, and a great way to leak PII into your trace backend. Store large content as span events instead, where the Collector can filter or drop it before it leaves your perimeter. Content capture is off by default for a reason. You opt in deliberately:

```bash
export OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=false
```

Reasoning summaries, risk class, context source: those are short, low-cardinality, safe as attributes. Full message content: events, redacted, or not at all.

## Correlation IDs across agent handoffs

Single agent is the easy case. The timeline shatters the second one agent delegates to another. Without IDs that survive the handoff and explicit parent-child span links, root cause becomes stitched-together log forensics across three systems at once.

OTel propagates trace context for you when you wire it through. The parent agent's span context rides along into the child so the whole delegation chain lands in one trace tree.

```python
from opentelemetry import context, propagate

def delegate(subagent, task, parent_reasoning):
    carrier = {}
    propagate.inject(carrier)  # serialize current trace context

    with tracer.start_as_current_span(f"invoke_agent {subagent}") as span:
        span.set_attribute("gen_ai.operation.name", "invoke_agent")
        span.set_attribute("gen_ai.agent.name", subagent)
        span.set_attribute("agent.delegation.reason", parent_reasoning)
        return subagent.run(task, trace_carrier=carrier)
```

On the receiving side you extract that carrier and start the child span inside the propagated context. Now the sub-agent's tool calls hang off the parent in the trace tree instead of floating in a separate void. Wire this in before you ship the second agent. Retrofitting correlation IDs after a multi-agent cascade is how weekends disappear.

## The decision-logging contract

Instrumentation captures what the runtime sees. You can also force the model to declare intent before it acts, which gives your trace store something concrete and gives a human gate something to halt on. Drop this in as a standing system-prompt block on any agent holding write or delete tools.

```text
DECISION LOGGING CONTRACT (applies every turn)

Before calling any tool that writes, deletes, modifies state,
sends data externally, or changes access, first emit a decision
record as a single JSON object on its own line:

{
  "intent": "<one sentence: what you are about to do>",
  "why": "<the trigger: what in context made this the next step>",
  "context_source": "<user msg | tool output | retrieved doc | file>",
  "risk": "read | write | destructive | external | access_change",
  "reversible": true | false
}

Rules:
- destructive or access_change: emit the record, then STOP and
  wait for explicit human approval. Do not proceed on your own.
- Never collapse multiple state changes into one unlogged step.
- If context_source is anything other than the operator's direct
  instruction, say so plainly.
```

Pipe that JSON straight into the matching span as attributes so it's queryable, not buried in stdout. Now your `agent.decision.context_source` field populates itself from the model's own declaration, and your gate has a clean `risk == destructive` condition to block on.

## Gotchas that bite people

**Span kinds aren't decoration.** Tool execution is INTERNAL, it's code your app owns. Inference is CLIENT, or INTERNAL when the model runs in-process. Retrieval against a vector store is CLIENT because it crosses a process boundary. Get these wrong and your service map draws arrows backwards, which makes the 2am trace read like fiction.

**Retention kills slow-burn cases.** Privacy-default short retention means the spans explaining a Tuesday incident are gone by Thursday. Agent attacks run low and slow, poison memory Monday, cash out Friday. Treat decision traces as security telemetry with a real retention policy, not debug noise you rotate out nightly.

**The recovery plane can't share the agent's identity.** This isn't tracing, but it's the one that turns an incident into an extinction event. If your backups sit behind the same credentials the agent holds, they're a second copy waiting for the same token. Air-gap the recovery vault out of the agent's blast radius.

## Wrapping up

Instrument every model call, tool execution, and retrieval as its own span. Attach the reasoning and the context source as attributes. Build on the GenAI conventions so it stays portable, propagate context across handoffs, and enforce a decision-logging contract on anything holding destructive tools. Do that and "what did it do, why, and what did it touch" becomes a query, not a weekend of archaeology.

I wrote the full breakdown, including the nine-second production-database wipe that makes this concrete and the operator checklist, over on [the ToxSec Substack](https://www.toxsec.com/p/what-did-your-agent-actually-do-last).

---

*ToxSec covers AI security vulnerabilities, attack chains, and the offensive tools defenders actually need to understand. Run by an AI Security Engineer with hands-on experience at the NSA, Amazon, and across the defense contracting sector. CISSP certified, M.S. in Cybersecurity Engineering.*
