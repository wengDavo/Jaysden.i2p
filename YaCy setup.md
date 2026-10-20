# YaCy setup for Jaysden

This configures a single YaCy peer that's only supposed to crawl and serve I2P, which is why you should stop YaCy before editing `DATA/SETTINGS/yacy.conf`, then restart. Most of this is also settable under Administration in the web UI, but the finalized version of Jaysden.i2p shouldn't have any option to access "Administration in the web UI", but administration shouldn't be viewable in settings and only accessible in the back-end, so remove all webpages for settings before publishing Jaysden.i2p on I2P.

## 1. Network isolation (Robinson)

Run as a standalone peer with no index exchange with the public YaCy network:

```
cluster.mode=privatepeer
allowDistributeIndex=false
allowReceiveIndex=false
allowDistributeIndexWhileCrawling=false
```

`cluster.mode=privatepeer` is the UI's "Private Peer" under Administration > Use Case & Accounts (ConfigNetwork). YaCy has no single "DHT off" switch, so the isolation is `privatepeer` plus both index-distribution flags off. Confirm in the Network config page that the peer shows as isolated after restart, then remove the Network config page before publishing Jaysden.i2p on I2P.

For isolation at the network-definition level as well, point `network.unit.definition` at a custom or the intranet unit file instead of `defaults/yacy.network.freeworld.unit`. With `privatepeer` already set this is belt-and-suspenders, so only do it if you want the peer fully detached from the freeworld definition.

## 2. Route outbound crawling through the I2P HTTP proxy

So every fetch leaves over I2P and nothing touches the clearnet:

```
remoteProxyUse=true
remoteProxyUse4SSL=true
remoteProxyHost=127.0.0.1
remoteProxyPort=4444
remoteProxyNoProxy=127.*,10.*,192.168.*,localhost
```

4444 is I2P's default HTTP proxy port, keep localhost and private ranges in `remoteProxyNoProxy` so YaCy's own API calls are not sent through I2P.

## 3. Serve Jaysden.i2p

In the I2P router console (http://127.0.0.1:7657/i2ptunnelmgr/), create an HTTP Server tunnel:

```
Tunnel type:  HTTP Server
Target host:  127.0.0.1
Target port:  <YaCy local HTTP port, commonly 8090; confirm in your build>
Website name: jaysden.i2p
```

Point YaCy at that local port and let it advertise the b32:

```
port=<same local port as the tunnel target>
staticIP=<your jaysden b32 address>
```

Do not set `publicPort`, it is not a key. `port` is the externally visible port. `bindPort` only matters if you bind internally to a different port behind forwarding, which you do not need here.

## 4. Robots.txt stays on

The old guide set `obeyRobotsTxt=false`. Do not. There is no such global key in current YaCy, and the crawler obeys robots.txt by default, which is what the project's Boundaries require. Leave per-crawl `crawlingObeyHtmlRobotsNoindex` and `crawlingObeyHtmlRobotsNofollow` at their defaults too. Politeness and blacklists are enforced per crawl, not by disabling them globally.

## 5. Keep the heap modest

Last year's guide set a 16GB Java heap, which does not fit the hardware Jaysden targets. Size the JVM heap to the device and raise it only when the index actually needs it. YaCy wants more heap as the index grows, so this is a dial to watch rather than set-and-forget, but a Pi or a home router is nowhere near 16GB and should never be configured as if it were.

## Verify before relying on this

These keys are from YaCy's config documentation and the `yacy.init` defaults, which may differ slightly from the exact build you install. After first start, check the live values in Administration > System Administration > Advanced Properties, and confirm the peer reports isolated in the Network page.
