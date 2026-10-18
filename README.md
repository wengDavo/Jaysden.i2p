# Jaysden

A search engine for I2P, built on YaCy, named in memory of a mutual friend (of one of the contributers) that passed away.

I2P's search engines are going down, whilst it should go without saying that any network with no search looks empty to anyone arriving for the first time, which means fewer visitors, fewer eepsites, and a network that looks emptier still. Jaysden.i2p breaks that loop by not only scraping and subsequently indexing more of I2P than any index before it, but also keeping what it finds reachable.

Status: Concept stage i.e nothing runs yet.

## How it works

Jaysden is a loosely YaCy-based crawler and index with a back-end that runs unattended. YaCy is open source, so the code is not the new part. The coverage, the freshness, and the work that sustains both are.

**Finding sites.** Every address enters one seed set, tagged with the source that found it:

- Registries and directories, merged and diffed (reg.i2p, identiguy.i2p, notbob.i2p and the rest)
- Old address books from past releases, retried on a schedule
- Clearnet-posted .i2p and .b32.i2p addresses from GitHub lists, forums, blogs and wikis
- Outbound links from every crawled page
- The i2psnark DHT and open trackers, for destinations running torrent clients
- Irc2P channel pastes
- A submission form and ping endpoint for site owners
- Floodfill LeaseSet observation, one source among the rest, within the limits below

Source tags make each method's yield measurable, so any method that finds nothing new gets switched off without touching the others.

**Liveness.** A destination is up if its LeaseSet resolves. Every candidate is resolved before indexing; failures go to the retry schedule, not the index. This is the only "still up" test, and it is content-neutral, so every category is checked the same way.

**Keeping sites findable.** Eepsites flicker, so Jaysden stores a compressed deduplicated snapshot of each page, with an age limit. Results work while a eepsite is down and survive after it dies, yet even in spite of that, only all media on eepsites that pass the hard-rule filter, hash-matched first, is archived by default.

**Filtering.** Hard rules run first: blacklists, hash matches, opt-outs, robots.txt, and no-index or no-archive markers. Matches are never stored or cached. A small text classifier scores the rest. A quantized model handles only the cases the cheap stages cannot settle, as a second layer and never the sole gate (see [REMOVAL.md]).

## Why demoscene?

Demoscene work gets the most from limited hardware, and I2P routers run on that kind of hardware. Jaysden applies the same discipline to the compute-bound parts, the classifier, index and archive: cheap filters before expensive ones, a text-first index, and an archive small enough to mirror. Webscraping speed is bound by tunnel builds, latency and uptime, not by code, and Jaysden claims no speedup there. The snapshot archive follows the scene's preservation tradition, whilst also giving us oppertunities for workaround for acheiving being able to snapshot what we find beyond text-based snapshots via applying the same sort of hardware restriction/limitation loopholing that developers in the demoscene apply to developing programs under the demoscene.

## Boundaries

- Opt-outs are honored fast, so no-index and no-archive markers are respected.
- Queries are never logged.
- Crawling is polite, with per-site rate limits stricter than clearnet, since many eepsites run on home connections.
- Encrypted and blinded LeaseSets are never targeted, just Floodfill-sourced addresses are checked against robots.txt and no-index markers before any fetch.
- The index is built to be shared between YaCy peers and mirrored, so Jaysden is not the single point of failure it replaces.
- The illegal-content policy is written before the first crawl, not after.

## Measuring coverage

Overlap between independent sources estimates how many sites remain undiscovered, so each crawl publishes that estimate with its results.

## Where to reach us

- https://discord.com/channels/1241171637978791987/1353883408786260029
