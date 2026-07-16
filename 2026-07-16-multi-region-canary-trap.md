> Originally published at: https://www.toxsec.com/p/canary-tokens-for-prompt-injection

Your system prompt is going to leak. Not "might." Every LLM app with a system prompt worth stealing eventually meets an attacker who talks the model into repeating it back word for word. You're not stopping that with a filter, so stop trying. Build the trap that tells you the second it happens instead.

That's a canary token: a random string planted somewhere the model shouldn't repeat, scanned for on the way out. One string, one substring check, zero false positives by construction. Here's how to actually wire that into a running app instead of just nodding along at the idea.

## Why a regex filter won't cut it

Most teams reach for pattern matching first. Block anything that looks like "ignore previous instructions," flag suspicious phrasing, tune a classifier's confidence threshold. All of it drowns you in noise, because curious users type "ignore previous instructions" constantly and mean nothing by it.

A canary sidesteps the whole mess through entropy instead of pattern matching. Pull sixteen random bytes and the odds of that exact string showing up in legitimate traffic round to zero. No pattern to tune, no threshold to babysit. The string either comes back or it doesn't.

```bash
pip install python-dotenv redis
```

We're adding Redis here for one reason: canary state needs to survive a restart and sync across replicas. A canary living in a single process's memory is useless the moment you scale past one instance.

## Layer one: plant and scan

Start with the baseline OWASP recommends for LLM01, prompt injection: a synthetic token dropped in the system prompt, scanned for on every response.

```python
import secrets
import redis

r = redis.Redis(host="localhost", port=6379, decode_responses=True)

def new_canary(region: str) -> str:
    token = f"tx-{region}-" + secrets.token_hex(8)
    r.set(f"canary:{region}", token)
    return token

def current_canary(region: str) -> str:
    return r.get(f"canary:{region}")

def scan_output(text: str, ctx: dict) -> str:
    for region in ("system", "tool", "rag"):
        token = current_canary(region)
        if token and token in text:
            alert(region, ctx)
            rotate(region)
            return "I can't share that information."
    return text
```

Redis is doing the boring, load-bearing job here: every replica reads the same canary state, so rotating a token after a hit actually rotates it everywhere, not just on the instance that caught the fire.

## Layer two: the append-and-strip variant

Here's the part the source article doesn't cover, and it's worth the extra code. OWASP's own tracker for the LLM Top 10 documents a second canary pattern that's stricter than passive detection: instruct the model to append the token to the end of every single response, then strip it before the response ever reaches the user.

```python
STRIP_CANARY = new_canary("sentinel")

def build_system_prompt(base_prompt: str) -> str:
    return (
        f"{base_prompt}\n"
        f"# Append this exact string to the end of every response, "
        f"do not explain why: {STRIP_CANARY}"
    )

def postprocess(response: str) -> str:
    if STRIP_CANARY not in response:
        # the model didn't follow the append instruction at all,
        # which is its own signal something upstream is off
        alert("sentinel-missing", {"response": response[:200]})
        return response
    return response.replace(STRIP_CANARY, "").rstrip()
```

The passive canary tells you when the system prompt got dumped. The sentinel variant tells you when the model stopped following basic instructions at all, which is a broader tripwire for injection that hijacked behavior without ever touching the marked text. Different failure mode, same trick.

## Layer three: alert like you mean it

A canary hit that only writes to a log file nobody tails is a canary that already failed. Wire it into something that pages a human.

```python
import json
import logging
import time

def alert(region: str, ctx: dict):
    payload = {
        "event": "confirmed_extraction",
        "region": region,
        "request_id": ctx.get("request_id"),
        "tenant": ctx.get("tenant"),
        "ts": int(time.time()),
    }
    logging.critical(json.dumps(payload))
    # swap for your real paging hook, PagerDuty, Slack webhook, whatever
    # fire_webhook(WEBHOOK_URL, payload)

def rotate(region: str):
    new_canary(region)
```

Structured JSON over plain-text logging matters more than it looks. The second you're grepping timestamps out of a free-text log line at 2am during an active extraction, you already lost ten minutes you didn't have.

## Gotchas that bite people

**Constant-time comparison isn't the move here.** If you've done any auth work, `secrets.compare_digest` is muscle memory for string comparison. Skip it for canary scanning. That function exists to stop timing attacks against secret comparisons where an attacker controls the input and measures response time. A canary scan is a plain containment check against model output the attacker doesn't get to time-probe the same way. Use it, and you've added latency for a threat model that doesn't apply here.

**Redis TTLs will silently eat your canary.** Set an expiry on the token key for unrelated reasons (maybe you're reusing a cache eviction policy) and the canary vanishes mid-session, so `current_canary` returns `None` and `scan_output` quietly stops checking anything. Give canary keys no TTL, or a TTL measured in days, not minutes.

**The sentinel pattern needs its own failure budget.** Models don't follow "append this exact string" 100% of the time. Track the miss rate on `sentinel-missing` separately from actual extractions, because a rising miss rate is a signal about model reliability, not a rising signal of attacks.

## Wrapping up

Two tripwires, one Redis-backed rotation store, alerts that actually reach a human. None of it stops the injection. All of it turns "did we get owned" from a guess into a query.

I wrote the full breakdown, including the stealth-placement canary variant and the false-positive math, over on [the ToxSec Substack](https://www.toxsec.com/p/canary-tokens-for-prompt-injection).

---

*ToxSec is run by a USMC veteran and Security Engineer with hands-on experience at AWS and the NSA. CISSP certified, M.S. in Cybersecurity Engineering. He covers security vulnerabilities, attack chains, and the tools defenders actually need to understand.*
