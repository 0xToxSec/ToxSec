> Originally published at: https://www.toxsec.com/p/ai-tar-pits-are-drowning-llm-scrapers

LLM crawlers are eating your bandwidth and you can't block your way out of it. Return a 403 and the scraper shrugs, rotates the IP, swaps the user-agent, and walks back in through a residential proxy an hour later. So a growing crowd of operators stopped blocking and started building mazes instead. Here's how to stand one up, and how to do it without nuking your own search ranking.

## Why blocking loses

A hard block is a signal. The scraper reads the 403, learns your defense, and adapts around it. That's the whole problem with deny rules: they teach the attacker exactly what tripped them.

The tar pit flips the move. It says yes to everything. The bot asks for a page, it gets a page, stuffed with links that loop right back into the maze. Every fake link looks like a fresh discovery, so the crawler chases it. The links lead deeper. There's no bottom. A human gets four pages into word salad and closes the tab. A scraper has no taste and no exit condition, so it just queues the next URL forever.

## Option 1: Nepenthes (the raw tool)

Nepenthes is the original. Named after the carnivorous pitcher plant, it sits behind your web server and serves any crawler an endless stream of randomly generated pages, each one packed with dozens of links that go nowhere but back in.

The clever part is determinism. The pages are random but generated deterministically, so the same URL always returns the same garbage. That matters. If a URL coughed up different junk on every visit, a smart crawler could flag it as dynamic and bail. Stable output fakes the one signal scrapers trust most: this looks like a flat static archive.

It also bakes in a deliberate stall. A small forced delay on every response wastes the bot's wall-clock time without bogging down your own box.

```yaml
GET /maze/a8f3/index.html      200   1.4s   38 links
GET /maze/a8f3/c19b.html       200   1.5s   41 links
GET /maze/a8f3/c19b/77de.html  200   1.4s   39 links
[depth: 4]  [unique pages so far: 6,212]  [exit: none]
```
The work queue never empties because the link count never hits zero. The crawler thinks it's making progress six thousand pages deep.

## Option 2: Cloudflare AI Labyrinth (the easy button)

If you don't want to babysit your own maze, Cloudflare shipped the same idea as a product. AI Labyrinth is an opt-in toggle in the dashboard, available even on the free plan. When it spots improper bot activity, it auto-deploys a network of linked AI-generated pages. No custom rules.

Cloudflare bolted on a piece the indie tools skipped: detection. The decoy pages hide behind nofollow links a real browser never renders, so the only thing that walks in is something crawling the raw graph. Walk the maze, get fingerprinted, get added to a shared bad-actor list every other Cloudflare customer pulls from. The trap doubles as a sensor.

```yaml
# the shape of the trap, not the trap
labyrinth:
  trigger: suspected_ai_crawler
  inject: nofollow_decoy_links     # human browsers never render these
  serve: generated_pages
  on_traversal:
    confidence: high_bot
    action: fingerprint_and_share   # feeds the global block list
```

## The poison layer

Most tar pits ship an optional Markov-chain generator, and this is the part AI companies actually fear. Markov output reads almost right: real words, real sentence shapes, zero meaning. It sails through a naive "is this English" filter and fails every "is this true" check nobody runs at scale. Feed enough of it into a training corpus and you accelerate model collapse, the documented failure mode where models trained on recursive synthetic slop lose the tails of their distribution and rot. Iocaine, the follow-on tool, leans all the way into this angle.

One detail the operators are honest about: no corpus ships with the tool. You bring your own text, so every install looks different and harder to fingerprint.

## Gotchas that bite

A few ways this blows up in your face:

- **Search engines are crawlers too.** A raw Nepenthes setup makes no distinction between an LLM scraper and Googlebot. Deploy it carelessly and you can get dropped from search results. Scope it hard.
- **It draws traffic on purpose.** The trap feeds bots exactly what they hunt for, so it pulls constant crawler traffic that spikes server CPU.
- **The author calls it malicious software.** Nepenthes' own creator labels it deliberately malicious and warns against running it unless you understand the fallout. Believe him. Cloudflare's version is the safer route because it scopes the maze to suspected bots and keeps it off pages real users see.

## Wrapping up

If you run content and the AI crawlers are eating you alive, you've got two real moves today: stand up Nepenthes and accept the operational risk, or flip on AI Labyrinth and let Cloudflare run the maze for you. Both burn the scraper's compute. Both can poison its training set on the way out. Neither wins forever, but neither has to. They just have to make your site the expensive meal so the bot goes chews on somebody else.

I wrote the full breakdown, including the data-poisoning math and where Cloudflare says the labyrinth is headed next, over on [the ToxSec Substack](https://www.toxsec.com/p/ai-tar-pits-are-drowning-llm-scrapers).

---

*ToxSec covers AI security vulnerabilities, attack chains, and the offensive tools defenders actually need to understand. Run by a USMC veteran and AI Security Engineer with hands-on experience at the NSA and Amazon. CISSP certified, M.S. in Cybersecurity Engineering.*
