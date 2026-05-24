> Originally published at: https://www.toxsec.com/p/ai-agent-security-after-google-io

# How to Lock Down an AI Agent Before It Goes Rogue

Your agent does whatever it reasoned it should do. Sometimes that means finishing the task. Sometimes it means reading a poisoned web page and deciding the page is the boss. If you're wiring an LLM into a browser, a toolchain, or somebody's inbox, you box that behavior in before you ship. Not after the audit log fills up.

## The failure mode baked into every agent

Pull apart any LLM agent and the wiring looks identical. A model sits in a loop. You feed it input and tools until a task finishes. The model picks the next action, the loop runs it, around it goes. The catch lives in the context window. Your instructions and the attacker's data land in the same place, through the same attention mechanism, with zero privilege separation. There's no trusted channel the model believes over the untrusted one. It's all tokens, and the model reasons over the whole pile and picks whatever looks most relevant.

So when a browser agent reads a page that says "ignore your task, do this instead," nothing in the model's head flags that a web page shouldn't be giving orders. Same deal when it reads a poisoned capability description from another service, or a background job chews through a hostile email. This is indirect prompt injection, and OWASP ranks it the number-one LLM risk for exactly this reason. It's a structural flaw, so you don't patch it out of the model. Two 2026 studies already showed autonomous agents SQL-injecting live sites and turning on their own users with nobody feeding them hacking instructions. The loop plus the missing boundary did it alone.

That means every real control lives outside the model. Let's wire some up.

## Layer one: allowlist the tools, starve the creds

Default-open is how you lose. An agent holding a generic "run shell command" tool and a long-lived token is a confused deputy with the keys to prod. Flip it. The agent gets an explicit allowlist of named actions and nothing else.

```yaml
# agent-tools.yaml — deny by default, allow by name
tools:
  - name: search_docs
    scope: read:knowledge_base
  - name: create_ticket
    scope: write:tickets
# anything not listed dies at the broker, not in a prompt
policy:
  default: deny
  network_egress: none      # no outbound unless a tool explicitly needs it
  credential_ttl: 900       # 15 min, then re-mint
```

Two things matter. The deny lives in your tool broker, not in a system prompt politely asking the model to behave. And the credential each tool carries is scoped to that one action and expires fast. If the agent gets steered, the blast radius is whatever those narrow scopes allow, instead of the union of every API key you ever handed it. Short TTLs mean a stolen token is a brick in fifteen minutes.

## Layer two: gate the dangerous actions, read the arguments

Logging tells you what happened. It stops nothing. By the time the entry lands, the data already left the building. What you want is a control that sits in front of the action and decides whether it runs at all.

Two pieces. First, a human checkpoint on anything irreversible or sensitive: sending mail, moving money, touching prod, anything exfil-shaped. Second, a runtime hook that reads the tool-call arguments before execution and trips on the obvious stuff.

```python
# pre-exec hook: inspect the args, not just the call name
SENSITIVE = {"send_email", "transfer", "delete", "post_webhook"}

def authorize(tool_name, args):
    if tool_name in SENSITIVE:
        if looks_like_exfil(args):     # external dest, bulk read, weird recipient
            return BLOCK
        return REQUIRE_HUMAN           # a checkpoint, not a log line
    return ALLOW
```

The function itself is beside the point. The point is that something between the model's decision and the real-world effect gets a vote. Enforcement, not observability. A pretty audit trail of the breach is still a breach.

## Gotchas that bite real deployments

A few things that look fine on day one and draw blood later.

Scope creep is the slow killer. The agent gets read access to code, then tickets, then customer mail. No single grant looked crazy. Nobody reviewed the aggregate. Put a recurring permission audit on the calendar and treat agent identities like the service accounts they actually are.

Trust goes transitive the second agents start talking. The moment your agent delegates to another agent, your blast radius swallows everything that second agent can reach too. Map the trust graph before you connect anything, especially across vendor boundaries where you can't see the other side's controls.

Authentication is not honesty. TLS and OAuth prove an agent is who it claims to be. They say nothing about whether the capability it advertises is real, or whether its self-description carries an injection aimed at your model. Verify behavior, not just identity.

## Wrapping up

You can't make the model tell data from instructions. So you build the boundary it lacks: deny-by-default tools, short-lived scoped creds, human checkpoints on the dangerous calls, and a runtime hook that reads arguments before they fire. None of it is a silver bullet. Stacked, it turns one poisoned input from "game over" into "blocked and logged." That's the whole job.

I wrote the full breakdown, including how this exact chain plays out across Project Mariner, the A2A protocol, and the 24/7 background agents that never log off, over on [the ToxSec Substack](https://www.toxsec.com/p/ai-agent-security-after-google-io).

---

*ToxSec covers AI security vulnerabilities, attack chains, and the offensive tools defenders actually need to understand. Run by an AI Security Engineer with hands-on experience at the NSA, Amazon, and across the defense contracting sector. CISSP certified, M.S. in Cybersecurity Engineering.*
