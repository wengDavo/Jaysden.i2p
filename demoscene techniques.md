# Demoscene techniques for Jaysden

The demoscene gets large results out of small hardware by generating instead of storing, approximating instead of counting exactly, and packing data to the bit. Those are the same constraints Jaysden runs under, and more so now that the back-end is a distributed scrape rather than a single crawler. Many peers, each on hardware as small as a Raspberry Pi or a home router, pull from the sources in the README (the registry union, old address books, clearnet and Irc2P mining, the link crawl, the torrent DHT, submissions, floodfill observation), scrape the eepsites they discover, and share the results. Spreading the work across peers raises two problems a single crawler never had: deciding which peer scrapes which site without a coordinator, and merging each peer's coverage, dedup, and index into one view without shipping everything. The scene's minimal-state, approximate, bit-packed habits answer both. This file lists the techniques worth retrofitting, each with working code and the Jaysden feature it serves.

These run as Python modules alongside the YaCy fork, not inside it. A few (b32 packing, varint) could also live in YaCy's Java index core.

Hashing: the code uses `mmh3` (fast, non-cryptographic, the right choice here). If it is not installed, `hashlib.blake2b(data, digest_size=8)` is a drop-in 64-bit hash. The sketches, bit-packing, varint, SimHash, rendezvous hashing and IBLT below were tested; the image, compression and mmap helpers are standard library usage, so confirm them in your build.

A hard line before the media sections: perceptual hashes here (dHash, SimHash) are for dedup and mirror detection only. They are not the illegal-content gate. Nothing media is fetched or stored until the known-illegal hash check in REMOVAL.md clears it first. That ordering is not optional.

---

## 1. Probabilistic sketches (bounded memory)

The scene's "generate, don't store" applied to data: hold a fixed-size summary instead of the full set.

### Bloom filter — deduplication

Already in the design (the progressive cascade). Membership test in a fixed bitarray, no stored keys. Double hashing simulates k hashes from two.

```python
import math, mmh3
from bitarray import bitarray

class BloomFilter:
    def __init__(self, capacity, error_rate=0.001):
        self.size = max(1, int(-capacity * math.log(error_rate) / (math.log(2) ** 2)))
        self.hashes = max(1, round(self.size / capacity * math.log(2)))
        self.bits = bitarray(self.size); self.bits.setall(False)

    def _pos(self, item):
        d = item.encode() if isinstance(item, str) else item
        h1, h2 = mmh3.hash(d, 0), mmh3.hash(d, 1)
        return [(h1 + i * h2) % self.size for i in range(self.hashes)]

    def add(self, item):
        for p in self._pos(item): self.bits[p] = True

    def __contains__(self, item):
        return all(self.bits[p] for p in self._pos(item))
```

### HyperLogLog — coverage cardinality and source overlap

The strongest fit for the coverage report, and it earns its place twice over in a distributed scrape. It counts unique sites in ~16KB at under 1% error, and because the union of two HLLs is the register-wise max, that merge is both how you estimate overlap between discovery sources and how you aggregate coverage across peers. Each peer keeps its own HLL, and merging them gives the network-wide unique count without any peer holding every address. That is the "overlap estimates how many sites remain undiscovered" line in the README, now computed across the whole fleet.

```python
import mmh3
from math import log

class HyperLogLog:
    def __init__(self, p=14):                 # 2^14 registers, ~16KB, ~0.8% error
        self.p = p; self.m = 1 << p
        self.reg = bytearray(self.m)
        self.alpha = 0.7213 / (1 + 1.079 / self.m)

    def add(self, item):
        d = item.encode() if isinstance(item, str) else item
        x = mmh3.hash64(d, signed=False)[0]
        idx = x & (self.m - 1); w = x >> self.p
        rank = (64 - self.p) - w.bit_length() + 1 if w else (64 - self.p) + 1
        if rank > self.reg[idx]: self.reg[idx] = rank

    def count(self):
        Z = sum(2.0 ** -r for r in self.reg)
        E = self.alpha * self.m * self.m / Z
        if E <= 2.5 * self.m:                 # small-range correction
            V = self.reg.count(0)
            if V: E = self.m * log(self.m / V)
        return int(E)

    def merge(self, other):                   # union = register-wise max
        for i in range(self.m):
            if other.reg[i] > self.reg[i]: self.reg[i] = other.reg[i]

# overlap(A, B) = count(A) + count(B) - count(A ∪ B)
```

### MinHash — near-duplicate and set similarity

Jaccard similarity between two sets in fixed space. Use it three ways now: signatures per discovery source to estimate source overlap, shingle-set signatures per page to catch near-duplicates, and signatures per peer's address set to see how much two peers' scrapes overlap, which tells you whether the work-partitioning is spreading the load or two peers are scraping the same sites. Accuracy scales with `num_perm`; 128 is noisy, 256 to 512 tightens it.

```python
import mmh3

class MinHash:
    def __init__(self, num_perm=256):
        self.n = num_perm; self.sig = [2**64 - 1] * num_perm

    def add(self, item):
        d = item.encode() if isinstance(item, str) else item
        for i in range(self.n):
            v = mmh3.hash64(d, seed=i, signed=False)[0]
            if v < self.sig[i]: self.sig[i] = v

    def jaccard(self, other):
        return sum(1 for a, b in zip(self.sig, other.sig) if a == b) / self.n
```

### Count-Min Sketch — frequency in bounded memory

How many sources reference an address, or term frequencies for ranking, without a per-key table. Over-estimates only, never under.

```python
import mmh3

class CountMinSketch:
    def __init__(self, width=1 << 16, depth=5):
        self.width = width; self.depth = depth
        self.table = [[0] * width for _ in range(depth)]

    def add(self, item, count=1):
        d = item.encode() if isinstance(item, str) else item
        for i in range(self.depth):
            self.table[i][mmh3.hash(d, i) % self.width] += count

    def estimate(self, item):
        d = item.encode() if isinstance(item, str) else item
        return min(self.table[i][mmh3.hash(d, i) % self.width] for i in range(self.depth))
```

### Reservoir sampling — uniform sample from an unbounded stream

The honest version of "probabilistic sampling past max depth" in the bloom cascade, and the way to sample pages for coverage estimation. O(k) memory, one pass, every item equally likely.

```python
import random

class Reservoir:
    def __init__(self, k):
        self.k = k; self.n = 0; self.items = []

    def offer(self, item):
        self.n += 1
        if len(self.items) < self.k:
            self.items.append(item)
        else:
            j = random.randint(0, self.n - 1)
            if j < self.k: self.items[j] = item
```

---

## 2. Fingerprinting (dedup, mirror detection, origin)

Compress a document to a small number that stays close for similar inputs. This is what the stats page calls content fingerprinting, and it feeds origin tagging.

### SimHash — text near-duplicate and mirror detection

A 64-bit fingerprint where similar text has small Hamming distance. Two pages a few bits apart are near-identical, which is a direct mirror signal for origin classification.

```python
import mmh3

def simhash(tokens, bits=64):
    v = [0] * bits
    for t in tokens:
        x = mmh3.hash64(t.encode(), signed=False)[0]
        for i in range(bits):
            v[i] += 1 if (x >> i) & 1 else -1
    out = 0
    for i in range(bits):
        if v[i] > 0: out |= (1 << i)
    return out

def hamming(a, b):
    return bin(a ^ b).count("1")

# hamming <= 3 across pages on different addresses ≈ mirror
```

### dHash — image fingerprint (dedup and mirror only)

Difference hash: a 64-bit perceptual fingerprint of an image. Catches the same image reused across mirrored sites, and lets you store a fingerprint instead of a file. This is dedup, not the safety gate.

```python
from PIL import Image

def dhash(path, hash_size=8):
    img = Image.open(path).convert("L").resize((hash_size + 1, hash_size), Image.LANCZOS)
    bits = 0; idx = 0
    for y in range(hash_size):
        for x in range(hash_size):
            if img.getpixel((x, y)) > img.getpixel((x + 1, y)):
                bits |= (1 << idx)
            idx += 1
    return bits
```

---

## 3. Size coding (storage)

Pack data to the bit so the archive and index stay small enough to mirror.

### Base32 address packing

The b32 alphabet is 32 symbols, so each character is exactly 5 bits. A 52-character address body packs from 52 bytes to 33 (260 bits). Tested round-trip.

```python
_ALPH = "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567"
_DEC = {c: i for i, c in enumerate(_ALPH)}

def pack_b32(s):
    acc = nbits = 0; out = bytearray()
    for c in s:
        acc = (acc << 5) | _DEC[c]; nbits += 5
        while nbits >= 8:
            nbits -= 8; out.append((acc >> nbits) & 0xFF)
    if nbits: out.append((acc << (8 - nbits)) & 0xFF)
    return bytes(out)

def unpack_b32(data, length):
    acc = nbits = 0; out = []
    for byte in data:
        acc = (acc << 8) | byte; nbits += 8
        while nbits >= 5 and len(out) < length:
            nbits -= 5; out.append(_ALPH[(acc >> nbits) & 0x1F])
    return "".join(out)
```

### Varint + delta encoding — posting lists

Classic inverted-index compression and pure size coding. Sort the IDs, store gaps, write each gap as a variable-length integer. In the test, 2000 sorted IDs went from 8000 bytes to 3552.

```python
def encode_varint(n):
    out = bytearray()
    while True:
        b = n & 0x7F; n >>= 7
        if n: out.append(b | 0x80)
        else: out.append(b); return bytes(out)

def encode_postings(sorted_ids):
    out = bytearray(); prev = 0
    for i in sorted_ids:
        out += encode_varint(i - prev); prev = i
    return bytes(out)

def decode_postings(data):
    ids = []; cur = shift = acc = 0
    for b in data:
        acc |= (b & 0x7F) << shift
        if b & 0x80: shift += 7
        else: cur += acc; ids.append(cur); acc = shift = 0
    return ids
```

### Roaring bitmaps — compressed sets

For large "seen document" sets and fast union/intersection, which also gives exact overlap when you want it instead of the HLL estimate. In the distributed scrape it is the compact form a peer ships when another peer needs its exact seen-set, and the intersection gives the redundant overlap directly. Do not hand-roll it, use `pyroaring`.

```python
from pyroaring import BitMap

seen = BitMap(); seen.add(doc_id)
overlap = len(source_a & source_b)      # exact, compressed
```

### Shared-dictionary compression — the small-page archive

Eepsite pages are small and share boilerplate, which is the worst case for per-document compression. Train one zstd dictionary on a sample of pages, then compress each page against it for far better ratios than LZ4 alone. Keep LZ4 for hot paths, use zstd-plus-dictionary for the archive.

```python
import zstandard as zstd

dict_data = zstd.train_dictionary(112640, sample_pages)   # sample_pages: list[bytes]
cctx = zstd.ZstdCompressor(dict_data=dict_data, level=19)
dctx = zstd.ZstdDecompressor(dict_data=dict_data)

blob = cctx.compress(page_bytes)
page = dctx.decompress(blob)
```

---

## 4. Systems techniques (low hardware)

### Memory-mapped index — serve a big index with little RAM

The sane version of the old codebase's ring buffer. mmap a packed index file and let the kernel page it in on demand, instead of loading it into a large heap. Read-mostly data never needs to be fully resident.

```python
import mmap, os, struct

class PackedIndex:
    def __init__(self, path):
        self.fd = os.open(path, os.O_RDONLY)
        self.mm = mmap.mmap(self.fd, 0, prot=mmap.PROT_READ)

    def read_u32(self, offset):
        return struct.unpack_from("<I", self.mm, offset)[0]

    def close(self):
        self.mm.close(); os.close(self.fd)
```

### Fixed-point decay and deterministic jitter

Integer-only priority so recrawl math runs on routers without an FPU, plus deterministic per-host jitter so crawls spread across a window instead of synchronizing, with no stored schedule table. This is the corrected scheduler: the tier comes from observed uptime (misses), the hash only decorrelates timing inside a tier. It never sets the tier, which was the old code's mistake.

```python
import mmh3

def recrawl_priority(misses, base=1000):
    return base >> min(misses, 16)          # halve per consecutive miss, integer only

def crawl_jitter(host, window_seconds):
    return mmh3.hash(host.encode(), 42) % window_seconds   # spread, not schedule
```

---

## 5. The media ambition: store a proxy, not the file

The realistic "demoscene loophole" for snapshotting media on tiny storage is to keep a perceptual proxy, a downscaled low-quality thumbnail plus its fingerprint, rather than the original. A 160px WebP thumbnail is a few KB against megabytes, and the fingerprint handles dedup and mirror matching.

This runs only after the illegal-content hash gate has cleared the source media. The thumbnail and dHash below are storage and dedup, never the safety check.

```python
from PIL import Image
import io

def media_proxy(path, max_dim=160, quality=40):
    img = Image.open(path).convert("RGB")
    img.thumbnail((max_dim, max_dim), Image.LANCZOS)
    buf = io.BytesIO()
    img.save(buf, format="WEBP", quality=quality, method=6)
    return buf.getvalue()        # a few KB, plus store dhash(path) alongside
```

---

## 6. Distributed scrape coordination

The back-end is many peers now, not one crawler. These are the scene's minimal-state and compact-sync habits applied to spreading the scrape and merging it back.

### Rendezvous (HRW) hashing — assign sites to peers with no table

Each peer decides which sites it owns by scoring every (peer, site) pair and taking the highest. No coordinator, no stored assignment table, and when a peer joins or leaves only the sites it owned move. This is "generate, don't store" applied to work partitioning: ownership is a function you compute, not a map you keep. Tested: even spread across peers, and removing a peer reassigns only its own sites.

```python
import mmh3

def owner(site, peers):
    return max(peers, key=lambda p: mmh3.hash64((p + ":" + site).encode(), signed=False)[0])

def owners(site, peers, r):          # r-way assignment for redundancy
    return sorted(peers, key=lambda p: mmh3.hash64((p + ":" + site).encode(), signed=False)[0],
                  reverse=True)[:r]
```

A peer scrapes `site` when it is in `owners(site, peers, r)`, so r peers cover each site and a lost peer's sites are already assigned to the next in line. Feed `peers` from the same address sources as everything else.

### IBLT — reconcile two peers' sets in space proportional to the difference

When two peers compare which addresses or which indexed documents they hold, shipping the whole set is wasteful once the sets are mostly equal. An Invertible Bloom Lookup Table encodes a set into a fixed number of cells; subtract one peer's table from another's and decode to get exactly the symmetric difference, in space that scales with the difference, not the sets. Tested: it recovers the difference in 80 cells even with a thousand shared keys. Keys are 64-bit ids (the first 8 bytes of a hash of the address); map ids back to addresses locally or request the few that differ.

```python
import mmh3

class IBLT:
    def __init__(self, cells=80, k=4):
        self.m = cells; self.k = k
        self.count = [0] * cells; self.keysum = [0] * cells; self.hashsum = [0] * cells

    def _pos(self, key):
        kb = key.to_bytes(8, "big")
        return set(mmh3.hash(kb, i) % self.m for i in range(self.k))

    def _hk(self, key):
        return mmh3.hash(key.to_bytes(8, "big"), 99) & 0xFFFFFFFF

    def _op(self, key, sign):
        hk = self._hk(key)
        for p in self._pos(key):
            self.count[p] += sign; self.keysum[p] ^= key; self.hashsum[p] ^= hk

    def insert(self, key):
        self._op(key, 1)

    def subtract(self, other):
        for i in range(self.m):
            self.count[i] -= other.count[i]
            self.keysum[i] ^= other.keysum[i]
            self.hashsum[i] ^= other.hashsum[i]

    def decode(self):
        a_only, b_only = set(), set(); changed = True
        while changed:
            changed = False
            for i in range(self.m):
                if self.count[i] in (1, -1) and self.hashsum[i] == self._hk(self.keysum[i]):
                    key = self.keysum[i]
                    (a_only if self.count[i] == 1 else b_only).add(key)
                    self._op(key, -self.count[i]); changed = True
        return a_only, b_only       # a_only: in A not B; b_only: in B not A
```

Size cells to roughly 1.5x the expected difference. If a decode stalls with cells left un-peeled, the difference outgrew the table, so retry larger.

### Gossip — spread new addresses without a broadcast

A peer that finds a new address does not tell everyone. It tells a few random peers, who tell a few more, and the address reaches the fleet in about log(N) rounds with bounded per-peer traffic. Pair it with the two above: gossip carries the IBLT or HLL summaries, and a peer pulls only the differing entries. The merge is a set union, so order and duplication do not matter.

```python
import random

def gossip_round(known: set, peer_samples, fanout=3):
    # peer_samples: list of callables, each returning a peer's address set
    for fetch in random.sample(peer_samples, min(fanout, len(peer_samples))):
        known |= fetch()
    return known
```

Cross-peer coverage and overlap ride on techniques already above: each peer keeps an HLL and the register-wise merge gives network-wide coverage, and per-peer MinHash signatures show whether two peers are scraping the same sites.

---

## Mapping to the design

| Technique | Jaysden use | Status |
|---|---|---|
| Bloom filter | Crawl-frontier dedup | In design (cascade) |
| HyperLogLog | Coverage cardinality, source and cross-peer overlap | New, strong fit |
| MinHash | Source, page, and cross-peer overlap | New |
| Count-Min Sketch | Address popularity, term frequency | New |
| Reservoir sampling | Sampling past the depth cap | In design (as "probabilistic sampling") |
| SimHash | Text mirror detection for origin tagging | In design (Stage 2) |
| dHash | Image dedup and mirror detection | New |
| Base32 packing | URL store size | In design (address packing) |
| Varint + delta | Posting-list compression | New |
| Roaring bitmaps | Compressed seen-sets, exact cross-peer overlap | New |
| zstd shared dictionary | Small-page archive compression | Extends LZ4 storage |
| Memory-mapped index | Serving on low RAM | Replaces the ring buffer idea |
| Fixed-point + jitter | No-FPU routers, load spreading | Corrects the old scheduler |
| Media proxy + perceptual hash | Media snapshots on tiny storage | Supports the gated media phase |
| Rendezvous (HRW) hashing | Assign sites to peers, no coordinator | New (distributed scrape) |
| IBLT | Reconcile peer sets by the difference | New (distributed scrape) |
| Gossip | Spread new addresses across peers | New (distributed scrape) |
