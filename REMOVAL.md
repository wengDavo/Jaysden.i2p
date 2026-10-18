# Removal and opt-out policy

Jaysden indexes public eepsites and keeps snapshots in the form of various media formats of their pages. This policy covers how a site operator keeps their site out, and how anyone gets content removed after the fact. Thereby, requests are honored regardless of reason.

## Before scraping, we stay out entirely

Jaysden obeys these on fetch, so a site that sets any of them is never indexed or archived:

- `robots.txt` disallow rules, read before crawling a host
- A `noindex` meta tag or `X-Robots-Tag: noindex` header, which drops the page from the index
- A `noarchive` marker, which allows indexing but keeps no stored snapshot
- An entry on the published blacklist

A site that wants nothing to do with Jaysden can also mail the contact address and be added to a permanent do-not-crawl list keyed by destination, with no need to run anything.

## For removal requests after webscraping

Anyone can request removal of an indexed site or an archived snapshot. Requests go to [contact], and the following are removed within [turnaround, e.g. 72 hours]:

- The index entry, so the site stops appearing in results
- Every stored snapshot of the affected pages
- The address from future crawls, if the requester asks for that too

**Verifying operators.** To remove a whole destination rather than a single page, the requester proves control of it: a signed message from the destination's key, a tag file served at a named path on the site, or a reply from the site's published contact. This stops one person from erasing another's site. Single-page takedowns on clear grounds (see below) need no proof.

**Third-party requests.** Someone who is not the operator can still request removal of a specific page on these grounds, and does not need to prove control:

- Illegal content under the operating jurisdiction
- Personal information published without consent (doxxing)
- Content that matches the blacklist or hash sets

## Independent of any request, Jaysden doesn't store:

- Anything matching the explicitly illegal-content filter on fetch
- Media files, by default
- Search queries

## Scope and limits

Removal from Jaysden removes a site from this index and this archive only. It has no effect on the eepsite itself, on other indexes, or on copies held elsewhere. Jaysden.i2p can't remove anything from I2P, because it does not host the sites it lists.

## Record

Removals are logged as a count and a date, with no requester identity and no description of the removed content, so the log cannot be used to find what was taken down.
