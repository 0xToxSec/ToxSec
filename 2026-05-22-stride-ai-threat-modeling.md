> Originally published at: https://www.toxsec.com/p/how-to-threat-model-ai-applications

STRIDE-GPT takes your architecture description and spits out a full STRIDE threat model in one shot. But the tool only works if you know which assets to point it at. AI applications carry assets traditional threat modeling never covered: system prompts, RAG documents, tool descriptions, embedding stores, agent reasoning chains. Point STRIDE-GPT at the wrong diagram and you get a traditional app threat model with an LLM bolted on. Here's how to run it right.

## What Changes When You Add an LLM

Traditional STRIDE assumes deterministic execution. Same input, same output. Clear trust boundaries between user, app, and data store. An LLM context window breaks all of that simultaneously. Developer instructions and attacker payloads both arrive as tokens through the same attention pipeline. There's no ring separation, no kernel mode, no privilege boundary the model actually enforces.

Your threat model needs to treat these as first-class assets:

- System prompt (it will leak, design like it already has)
- RAG retrieval corpus and every document inside it
- Tool descriptions in any connected MCP server
- Vector embeddings (treat them as plaintext, they can be inverted)
- Agent reasoning chains and the full tool call sequence

Every place untrusted text can reach the context window is a trust boundary. Mark all of them before you run STRIDE.

## Setting Up STRIDE-GPT

STRIDE-GPT is open-source and generates a STRIDE pass against your written architecture description with explicit OWASP LLM Top 10 support.

```bash
pip install stride-gpt
```

Write your architecture description before you open the tool. Include:

- Every component: user, API gateway, orchestrator, model provider, tool set, data stores
- Every data flow: where user input enters, how it reaches the model, what the model can write to
- Every trust boundary: anywhere you'd draw a line between trusted and not trusted
- Every tool the agent can invoke, including MCP servers and their descriptions

"An AI chatbot with RAG" gets you generic output. "A FastAPI app with a Pinecone RAG corpus, three MCP tools including a file write endpoint, and a GPT-4o backend behind an API gateway" gets you a threat model you can actually act on.

## Covering Repudiation: Log the Full Context

Most agent frameworks log the final answer. That's not enough for any post-incident reconstruction worth running. For every agent decision you need a structured trace with five fields minimum:

```python
span = tracer.start_span("agent_decision")
span.set_attribute("system_prompt_hash", hash(system_prompt))
span.set_attribute("retrieved_context_ids", json.dumps(chunk_ids))
span.set_attribute("tool_calls", json.dumps(tool_calls))
span.set_attribute("model_output", response)
span.set_attribute("session_id", session_id)
span.set_attribute("user_id", user_id)
```

Langfuse and Phoenix both wrap OpenTelemetry for LLM-native tracing. Sign or hash entries that touch privileged operations. Without the full context window logged, an attacker who poisons your agent's memory leaves no trace of when the state changed. The tampered state just sits there across sessions looking normal.

## Covering Denial of Wallet: Three Layers

Request-based rate limits don't protect against token drain attacks. One multi-step agentic query can cost 500x more than a cached response and still register as one request against your rate limiter. The limiter never fires.

Layer 1: AWS Budgets with BudgetActions. When the daily ceiling hits, the API automatically revokes Bedrock invoke permissions. Hard kill, not an alert.

```json
{
  "BudgetName": "bedrock-daily-cap",
  "BudgetLimit": { "Amount": "50", "Unit": "USD" },
  "BudgetType": "COST",
  "BudgetActions": [{
    "ActionType": "APPLY_IAM_POLICY",
    "ActionThreshold": {
      "ActionThresholdValue": 100,
      "ActionThresholdType": "PERCENTAGE"
    }
  }]
}
```

Layer 2: AI gateway enforcing per-key token-based rate limits in front of the model provider. Cloudflare AI Gateway, Portkey, and Helicone all support token counting. Count tokens, not requests.

Layer 3: Vendor-side caps at the model provider. OpenAI usage tiers, Anthropic spend limits, Google Cloud quotas. All three layers independently. Any single layer alone is a single point of failure.

## Covering Elevation of Privilege: Scope With OPA

The model holds your tools' permissions. Prompt injection inherits them all. The only real fix is scope enforcement outside the model entirely.

Open Policy Agent at tool dispatch checks every invocation against an allowlist tied to the current session's user identity:

```rego
package tool_dispatch

default allow = false

allow {
  input.tool_name == permitted_tools[_]
  input.session.user_role == "standard"
}

permitted_tools := ["search", "read_file", "summarize"]
```

Destructive operations, deletes, writes, payments, external sends, get a `requires_human_approval` flag enforced at the dispatch layer before the call fires. The model never sees the approval token, so prompt injection can't bypass the gate by telling the model to approve itself.

## Three Gotchas That Bite People

System prompt exposure. Anything you'd panic about on Pastebin doesn't belong in the prompt template. Pull credentials, internal URLs, and business logic from a real authorization layer at runtime. The prompt will be extracted eventually.

Embedding inversion. Vector databases store documents as numerical embeddings. Research has shown embeddings can be inverted back into the original text. If your vector store is reachable from any process holding an API key, you have an information disclosure problem regardless of how the documents are stored.

Threat model drift. Every MCP server you bolt on grants capabilities the original model never covered. Re-run STRIDE every time a new tool, RAG corpus, or data source gets connected. Twenty minutes of walkthrough beats a postmortem.

## What You Can Ship Today

Run STRIDE-GPT against a written architecture description with all five AI-specific assets called out explicitly. Set one hard spending cap that kills the key. Add the six-field structured trace to your agent's decision loop. Those three changes close the highest-exposure gaps across Repudiation, Denial of Service, and Elevation of Privilege before anything else gets shipped.

I wrote the full STRIDE-AI breakdown including the seven production red flags, the copy-paste threat model prompt, and the complete three-layer denial-of-wallet circuit breaker over on [the ToxSec Substack](https://www.toxsec.com/p/how-to-threat-model-ai-applications).

---

*ToxSec covers AI security vulnerabilities, attack chains, and the offensive tools defenders actually need to understand. Run by an AI Security Engineer with hands-on experience at the NSA, Amazon, and across the defense contracting sector. CISSP certified, M.S. in Cybersecurity Engineering.*
