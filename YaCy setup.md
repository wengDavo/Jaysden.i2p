# YaCy setup for Jaysden

This configures a single YaCy peer to crawl and serve only I2P. Keys are for the current YaCy release and replace the outdated ones in last year's guide (`network.unit.name`, `network.unit.dht` and `publicPort` are not real keys). Stop YaCy before editing `DATA/SETTINGS/yacy.conf`, then restart. Most of this is also settable under Administration in the web UI.

## 1. Network isolation (Robinson)

Run as a standalone peer with no index exchange with the public YaCy network:

```
cluster.mode=privatepeer
allowDistributeIndex=false
allowReceiveIndex=false
allowDistributeIndexWhileCrawling=false
```

`cluster.mode=privatepeer` is the UI's "Private Peer" under Administration > Use Case & Accounts (ConfigNetwork). YaCy has no single "DHT off" switch, so the isolation is `privatepeer` plus both index-distribution flags off. Confirm in the Network config page that the peer shows as isolated after restart.

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

4444 is I2P's default HTTP proxy port. Keep localhost and private ranges in `remoteProxyNoProxy` so YaCy's own API calls are not sent through I2P.

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

Do not set `publicPort`, it is not a key. `port` is the externally visible port; `bindPort` only matters if you bind internally to a different port behind forwarding, which you do not need here.

## 4. Robots.txt stays on

The old guide set `obeyRobotsTxt=false`. Do not. There is no such global key in current YaCy, and the crawler obeys robots.txt by default, which is what the project's Boundaries require. Leave per-crawl `crawlingObeyHtmlRobotsNoindex` and `crawlingObeyHtmlRobotsNofollow` at their defaults too. Politeness and blacklists are enforced per crawl, not by disabling them globally.

## Verify before relying on this

These keys are from YaCy's config documentation and the `yacy.init` defaults, which may differ slightly from the exact build you install. After first start, check the live values in Administration > System Administration > Advanced Properties, and confirm the peer reports isolated in the Network page.
