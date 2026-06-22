> Originally published at: https://www.toxsec.com/p/ai-tar-pits-are-drowning-llm-scrapers

LLM crawlers are eating your bandwidth and you can't block your way out of it. Return a 403 and the scraper shrugs, rotates the IP, swaps the user-agent, and walks back in through a residential proxy an hour later. So a growing crowd of operators stopped blocking and started building mazes instead. Here's how to stand one up, and how to do it without nuking your own search ranking.

## Why blocking loses

A hard block is a signal. The scraper reads the 403, learns your defense, and adapts around it. That's the whole problem with deny rules: they teach the attacker exactly what tripped them.

The tar pit flips the move. It says yes to everything. The bot asks for a page, it gets a page, stuffed with links that loop right back into the maze. Every fake link looks like a fresh discovery, so the crawler chases it. The links lead deeper. There's no bottom. A human gets four pages into word salad and closes the tab. A scraper has no taste and no exit condition, so it just queues the next URL forever.

## Option 1: Nepenthes (the raw tool)

Nepenthes is the original. Named after the carnivorous pitcher plant, it sits behind your web server and serves any crawler an endless stream of randomly generated pages, each one packed with dozens of links that go nowhere but back in.

The clever part is determinism. The pages are random but generated deterministically, so the same URL always returns the same garbage. That matters. If a URL coughed up different junk on every visit, a smart crawler could flag it as dynamic and bail. Stable output fakes the one signal scrapers trust most: this looks like a flat static archive.

It also bakes in a deliberate stall. A small forced delay on every response wastes the bot's wall-clock time without bogging down your own box.
