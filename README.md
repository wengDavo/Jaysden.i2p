# Jaysden

A search engine for I2P, built on YaCy, named in memory of a mutual friend (of one of the contributers) that passed away.

I2P's search engines are going down, whilst it should go without saying that any network with no search looks empty to anyone arriving for the first time, which means fewer visitors, fewer eepsites, and a network that looks emptier still. Jaysden.i2p breaks that loop by not only scraping and subsequently indexing more of I2P than any index before it, but also keeping what it finds reachable.

[**Status:** Concept stage i.e nothing runs yet.]

## How it works

Jaysden is a loosely YaCy-based crawler and index with a back-end that runs unattended. YaCy is open source, so the code is not the new part. The coverage, the freshness, and the work that sustains both are.

## Finding sites

**Every address enters one seed set, tagged with the source that found it:**

- Registries and directories, merged and diffed (reg.i2p, identiguy.i2p, notbob.i2p and the rest).
- Old address books from past releases, retried on a schedule.
- Clearnet-posted .i2p and .b32.i2p addresses from GitHub lists, forums, blogs and wikis.
- Outbound links from every scraped page.
- The i2psnark DHT and open trackers, for destinations running torrent clients.
- Irc2P channel pastes.
- A submission form and ping endpoint for site owners.
- Floodfill LeaseSet observation, one source among the rest, within the limits below.
- Source tags make each method's yield measurable.
- Any method that finds nothing new gets switched off without touching the others.

## The '4 Horsemen' of Jaysden's auto-filtering approach

**Hard rules run first:** This includes blacklists, hash matches, opt-outs, robots.txt, and no-index or no-archive markers. Matches are never stored or cached, so a small text classifier scores the rest. A quantized model handles only the cases the cheap stages cannot settle, as a second layer and never the sole gate (see [REMOVAL.md](<REMOVAL.md>)).

**Liveness:** A destination is up if its LeaseSet resolves, as every candidate is resolved before indexing. This means that failures go to the retry schedule, rather than to the index. This is the only "still up" test, and it is content-neutral, so every category is checked the same way.

**Keeping sites findable:** Eepsites flicker, so Jaysden stores a compressed, deduplicated snapshot of each page, with an age limit. Results work while an eepsite is down and survive after it dies. Snapshots are text-first. Media is archived only once a known-illegal-content hash gate is live, and then only media that clears the hard-rule filter and is checked against the hash sets first, so anything that matches is never fetched into the archive. Until that hash source is in place, the archive stays text-only.

**Origin tagging:** Each indexed site is classified by origin, native versus derivative (mirror, instance, proxy/gateway), with multi-homed and unknown as their own cases. Detection runs on signals already gathered during the crawl, cheapest first. A cross-network link to a clearnet or onion address is recorded only when the site advertises it; A site that appears multi-homed without saying so is tagged unknown, never outed. Full detection, schema and the stats-page layout are in [the stats page format](<eepsite stat webpage format.md>).

## Why demoscene?

Demoscene work gets the most from limited hardware, and I2P routers run on that kind of hardware. Jaysden applies the same discipline to the compute-bound parts, the classifier, index and archive: cheap filters before expensive ones, a text-first index, and an archive compressed small enough to mirror. Webscraping speed is bound by tunnel builds, latency and uptime, not by code, and Jaysden claims no speedup there. The snapshot archive follows the scene's preservation tradition. That same size-coding discipline is what later makes media archiving feasible on this hardware, once the hash gate above is live, since fitting media snapshots into a mirrorable footprint is a compression problem of exactly the kind the demoscene exists to solve.

## Under the hood

- **Deduplication:** a cascade of bloom filters whose precision degrades with crawl depth, each backed by a bitarray so memory stays bounded. Shallow depths get high precision, deep crawls get coarse filters, and past the deepest tier the crawler falls back to probabilistic sampling. Dedup memory stays flat on a small box instead of growing with the frontier.

- **Storage:** snapshots and index payloads compress with LZ4 in fast mode, which keeps the text-first archive small enough to mirror. b32 addresses are a fixed 52-character charset, so they bit-pack to roughly 32 bytes, worth doing for the URL store given how many the index holds. Media follows the hash-gate rule in Keeping sites findable and [REMOVAL.md](<REMOVAL.md>)).

- **Recrawl scheduling:** the tier idea from the old scheduler (critical, active, normal, archive, roughly hourly to roughly weekly), but a site's tier comes from its observed uptime history, not from bits of its address hash. The liveness test already produces that history, so a site that resolves on most checks earns a faster tier and one that stops resolving long enough leaves the index. Scheduling by address hash set frequency at random and is not carried over.

## Boundaries

- Opt-outs are honored fast, so no-index and no-archive markers are respected.
- Queries are never logged.
- Crawling is polite, with per-site rate limits stricter than clearnet, since many eepsites run on home connections.
- Encrypted and blinded LeaseSets are never targeted, just Floodfill-sourced addresses are checked against robots.txt and no-index markers before any fetch.
- The index is built to be shared between YaCy peers and mirrored, so Jaysden is not the single point of failure it replaces.
- The illegal-content policy is written before the first crawl, not after.
- A clearnet or onion identity is linked to an .i2p address only when the site advertises it, so Jaysden never infers and publishes an undisclosed cross-network identity, due to that being a deanonymization vector.

## Coverage reporting

Keep in mind, that the headline coverage number only counts sites native to I2P. Hence, Multi-homed and unknown show as their own subcounts rather than being directly folded into it, so a misclassification undercounts native rather than inflating it. Mirrors, instances and proxies are reported separately and never counted toward the headline. Overlap between independent sources estimates how many sites remain undiscovered, per origin type. This is what keeps "more than any index so far" honest, since a total padded with mirrors and software instances is the exact inflation that made earlier indexes look bigger than the network was.
