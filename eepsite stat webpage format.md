# Jaysden.i2p statistics page

The stats page reports what the crawler has found, grouped so that genuinely-I2P content is never visually merged with content that only passes through.

## Layout

A symmetrical 50/50 split, two equal columns, mirror image of each other:

- **Left = Native.** - Subcategories: native, multi-homed, unknown.
- **Right = Derivatives.** - Subcategories: mirror, instance, proxy/gateway.

Each column shows its group total at the top and a count per subcategory below. The left/right assignment is cosmetic, since the halves are mirrored, but native leads because it is the number the whole project exists to grow.

Subcategory meanings:

- **Native:** built for and living on I2P, with no clearnet, Tor, or other origin.
- **Multi-homed:** run on I2P by its operator, who also runs it elsewhere and says so.
- **Unknown:** signals conflict or are too thin to classify.
- **Mirror:** a copy of a specific clearnet or onion site's content.
- **Instance:** a fresh deployment of clearnet open-source software (SearXNG, MediaWiki, Lemmy), not a copy of one site.
- **Proxy/gateway:** relays clearnet rather than hosting anything, such as the metasearch front ends and inproxies.

## Counting rule

The headline native figure counts the native subcategory only. Multi-homed and unknown render as distinct subcounts inside the Native column and are not summed into the headline, so a misclassification fails toward undercounting native rather than inflating it. The Derivatives column is never folded into the headline. Overlap between independent sources gives a per-group estimate of how many sites remain undiscovered.

## Detecting origin

Cheap signals first, staged like the content filter so the expensive step runs rarely.

Stage 1, from the page already fetched:

- `rel="canonical"` or alternate-link tags pointing at a clearnet or .onion URL
- Generator meta tag and server header, which fingerprint known software (SearXNG, MediaWiki, Lemmy)
- Outbound link ratio: a native site links mostly to other .i2p and .b32 addresses, a mirror carries clearnet http/https links
- Clearnet asset references that fail to load inside I2P. A lazy mirror hardcodes clearnet image and script URLs, and that breakage is itself a fingerprint
- Explicit "mirror of X" banners or footer branding naming a clearnet site
- Favicon hash matched against a small set of known clearnet favicons

Stage 2, only when Stage 1 is ambiguous:

- Content fingerprint (SimHash or shingles) of title, description and body, compared against a reference set. A full clearnet corpus is too heavy for the hardware target, so keep the reference set to the specific sites expected to be mirrored, not the whole web.

Tor has a convention worth checking for, the Onion-Location header that advertises an onion equivalent. I2P has no equally standard header, so for I2P the link tags and content signals above do the work. Verify this against current I2P practice before building on it.

## Privacy boundary

Linking an .i2p address to a clearnet or onion identity is a deanonymization vector, the same one the removal policy treats as doxxing. The rules:

- Record a cross-network link only when the site itself advertises it, through a canonical tag, a self-link, or a published bridge list.
- Never infer a clearnet owner from correlation and then publish it.
- A site that is quietly multi-homed and did not say so is tagged unknown, not outed.

Without this rule the feature becomes a correlation engine that undoes the thing I2P is for.

## Storage

Each index record carries an origin block rather than a flat tag, so the UI can badge results and filter to native only:

```json
"origin": {
  "type": "instance",
  "confidence": 0.82,
  "signals": ["generator:searxng", "clearnet_link_ratio:0.0", "no_canonical"],
  "disclosed_links": []
}
```

`disclosed_links` holds only operator-advertised equivalents and stays empty otherwise, which enforces the privacy rule in the schema itself.
