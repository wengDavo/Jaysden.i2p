# Backend architecture notes

These fold the reusable engineering from the prior ("Armwork") codebase into the current design. The pieces that survived are the ones that fit the modest-hardware target. The discovery-by-address-mutation, hash-pattern scheduling, CUDA path, lock-free ring buffer and 16GB heap from that codebase were dropped as broken or contrary to that target.

Context: the earlier I2P search engines, including the now-defunct seeker.i2p, find4me.i2p and i2pyacy.bandura.i2p, are no longer usable. The design below favors low resource use, a mirrorable index and no single operator, so Jaysden does not fail the same way.

## Duplicate detection: progressive bloom cascade

A cascade of bloom filters whose precision degrades with crawl depth, each backed by a bitarray so memory stays bounded. Shallow depths get high precision (low false-positive rate), deep crawls get coarse filters, and past the deepest tier the crawler falls back to probabilistic sampling. This keeps dedup memory flat on a small box instead of growing with the frontier.

## Storage: LZ4 and address packing

Snapshot pages and index payloads are compressed with LZ4 in fast mode, which suits the text-only archive and keeps it small enough for other peers to mirror. b32 addresses are a fixed 52-character charset, so they bit-pack to roughly 32 bytes, which is worth doing for the URL store given how many addresses the index holds. Both serve the "small enough to mirror" goal rather than raw speed.

## Recrawl scheduling: uptime-driven tiers

Keep the tier idea from the old scheduler (critical, active, normal, archive, from roughly hourly to roughly weekly), but assign a site's tier from its observed uptime history, not from bits of its address hash. The liveness test already produces that history: a site that resolves on most checks earns a faster tier, one that is usually down drops toward archive, one that stops resolving for long enough leaves the index. Scheduling by address hash, as the old code did, set frequency at random and is not carried over.
