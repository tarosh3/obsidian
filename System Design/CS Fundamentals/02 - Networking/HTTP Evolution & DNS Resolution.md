---
title: "HTTP Evolution & DNS Resolution"
aliases: [HTTP/2, HTTP/3, QUIC, DNS Resolution Chain]
tags: [system-design, cs-fundamentals, networking, http, dns]
status: reference-quality
---

# HTTP Evolution & DNS Resolution

> [!abstract] What you'll be able to do after this chapter
> Explain exactly what problem each HTTP version fixed over the last, why HTTP/3 moved off TCP entirely, and trace a DNS lookup through every caching layer between browser and authoritative server.

> [!info] Builds on TCP Deep Dive
> [[CS Fundamentals/02 - Networking/TCP Deep Dive|The TCP chapter]] covers the transport layer these protocols run on (or, for HTTP/3, deliberately don't run on). This chapter doesn't re-derive TCP mechanics — it's about what HTTP and DNS build on top.

---

## HTTP/1.1 — the baseline problem

Each request needed its own connection unless `Keep-Alive` reused one — and even with a reused connection, requests on it were **head-of-line blocked**: request 2 couldn't start until request 1's *response* fully finished, single-file. Browsers worked around this by opening multiple parallel TCP connections per host (typically 6) — a real workaround, but each connection pays its own TCP handshake and (for HTTPS) TLS handshake cost.

## HTTP/2 — multiplexing over one connection

HTTP/2 fixes the head-of-line blocking at the **HTTP layer** by multiplexing multiple request/response streams over a **single** TCP connection — requests no longer need to wait for prior ones to finish. It also adds **HPACK header compression** (headers repeat heavily across requests to the same host; compressing them across the connection's lifetime, not per-request, cuts real overhead) and **server push** (the server can proactively send resources it knows the client will need, e.g. pushing a CSS file alongside the HTML that references it — though push saw limited real-world adoption since it's hard to predict correctly and can waste bandwidth on already-cached resources).

> [!bug] HTTP/2 solved head-of-line blocking at the HTTP layer, but not underneath it
> Multiplexing streams over **one TCP connection** means TCP itself doesn't know about the separate streams — if a single packet is lost, TCP's in-order delivery guarantee blocks **all** multiplexed streams behind that one lost packet, not just the stream it belonged to. HTTP/2 moved the head-of-line blocking problem down a layer; it didn't eliminate it.

## HTTP/3 & QUIC — moving off TCP entirely

HTTP/3 runs over **QUIC**, built on **UDP** instead of TCP, specifically to fix the problem HTTP/2 couldn't reach. QUIC implements its own reliability and multiplexing **above** UDP, but critically, it tracks loss and retransmission **per stream** — a lost packet only blocks the one stream it belonged to, not every multiplexed stream sharing the connection.

QUIC also folds the transport and TLS handshakes together, enabling **0-RTT** connection resumption for a client that has connected before — it can send actual request data in its very first packet to a known server, rather than paying separate round-trips for TCP handshake, then TLS handshake, then the request.

| | HTTP/1.1 | HTTP/2 | HTTP/3 (QUIC) |
|---|---|---|---|
| Transport | TCP | TCP | UDP (QUIC) |
| Head-of-line blocking | Yes (per connection) | Fixed at HTTP layer, still present at TCP layer | Fixed — per-stream loss handling |
| Handshake cost | TCP + TLS, separate | TCP + TLS, separate | Combined, 0-RTT on resumption |

## DNS resolution — the chain before any HTTP request even starts

A domain lookup passes through several layers, each with its own cache:

1. **Browser cache** — checks if it already resolved this hostname recently.
2. **OS resolver cache** — the operating system's own cache.
3. **Recursive resolver** (typically the ISP's or a public one like `8.8.8.8`) — if not cached, it does the actual multi-step lookup on the client's behalf.
4. **Root nameserver** — tells the resolver which TLD server handles `.com`/`.org`/etc.
5. **TLD nameserver** — tells the resolver which authoritative nameserver handles the specific domain.
6. **Authoritative nameserver** — returns the actual IP address for the domain.

```mermaid
sequenceDiagram
    participant Client
    participant Recursive as Recursive Resolver
    participant Root
    participant TLD
    participant Auth as Authoritative NS
    Client->>Recursive: resolve example.com
    Recursive->>Root: who handles .com?
    Root-->>Recursive: TLD server address
    Recursive->>TLD: who handles example.com?
    TLD-->>Recursive: authoritative NS address
    Recursive->>Auth: what's the IP for example.com?
    Auth-->>Recursive: IP address
    Recursive-->>Client: IP address (cached per TTL)
```

> [!tip] Every layer caches by TTL — this is the actual lever for "how fast does a DNS change propagate"
> Each response carries a **TTL** dictating how long it may be cached at every layer above it. Lowering a record's TTL *before* a planned change (e.g., migrating to a new IP) is the standard operational practice — it shrinks the window during which stale, cached answers keep routing to the old IP after the record is updated.

## Scaling: 1 request to global traffic

```mermaid
flowchart TD
    A["Few requests<br/>HTTP/1.1 is fine"] --> B["Many resources per page<br/>HTTP/2 multiplexing matters"]
    B --> C["Lossy / mobile networks,<br/>at scale<br/>HTTP/3's per-stream loss<br/>handling matters most"]
    C --> D["Global DNS scale<br/>anycast + geo-DNS essential<br/>for routing users to<br/>nearest infrastructure"]
```

At global scale, DNS itself becomes a routing mechanism, not just a lookup — returning different IPs to users in different regions (geo-DNS) or relying on anycast (the same mechanism [[CS Fundamentals/02 - Networking/CDN Internals|CDN Internals]] uses) so the "nearest" answer falls naturally out of network topology rather than needing centralized geographic logic. At this scale, HTTP/3's resilience to packet loss matters most precisely where it's most common — mobile and generally lossier last-mile networks — which is exactly the deployment context HTTP/3 was designed to help with most.

## Failure scenarios

> [!bug] What actually happens
> - **A DNS server becomes unreachable:** resolvers are configured with multiple nameservers and fall back automatically — redundancy is built into DNS resolution itself, not an afterthought.
> - **TTL misconfigured too high during an incident:** already covered above — a real, avoidable extension of downtime during exactly the moment fast propagation matters most.
> - **A CDN/anycast edge node failing:** traffic reroutes automatically to the next-nearest advertising node as BGP routing reconverges — no application-level failover logic needed, the same property already named in [[CS Fundamentals/02 - Networking/CDN Internals|CDN Internals]].

## Monitoring

> [!info] What to watch
> **DNS resolution time** — a rising trend signals resolver or authoritative-nameserver health issues upstream of the application entirely. **Cache hit rate at each resolution layer** — low hit rates mean more full resolution chains, directly adding latency. **TTL propagation lag during an active change** — the direct, measurable window during which some clients still route to the old answer.

## Common mistakes

> [!warning] Real, recurring errors
> 1. **Setting DNS TTL too high before a planned migration** — already covered above; the fix is lowering TTL proactively, well before the change, not during the incident it was meant to prevent.
> 2. **Assuming HTTP/2 solved head-of-line blocking completely** — it moved the problem to the TCP layer (the bug callout above), not eliminated it.
> 3. **Assuming HTTP/3 is a drop-in replacement everywhere** — some corporate/network middleboxes block or deprioritize UDP traffic, a real, practical deployment concern worth naming rather than assuming universal support.

---

## Interview Q&A

> [!info] Leveled by seniority
> **Beginner:** "What problem did HTTP/2 solve over HTTP/1.1?" — head-of-line blocking at the HTTP layer, via multiplexing over one connection. **Intermediate:** "Why did HTTP/3 move to UDP?" — Section on HTTP/3, per-stream loss handling QUIC needed that TCP's single in-order stream couldn't provide. **Senior:** "A migration to a new server IP is rolling out but some users are still hitting the old server hours later — diagnose it." — expects checking the DNS record's TTL first, the direct, common cause. **Staff:** "Design the DNS strategy for a service that needs to fail over to a backup region within seconds during an outage." — expects a very low TTL on the relevant record, set proactively (not reactively), plus likely a health-check-driven DNS failover mechanism, not a manual runbook alone. **Architect:** "How would you decide whether a new public API should be deployed with HTTP/3 support given the user base includes users on restrictive corporate networks?" — expects naming the UDP-blocking middlebox risk explicitly and a fallback-to-HTTP/2 strategy, not assuming HTTP/3 everywhere is a strict improvement with no deployment risk.

> [!question]- Why didn't HTTP/3 just fix TCP instead of switching to UDP?
> TCP's in-order, reliable-delivery guarantees are implemented in kernels and middleboxes across the entire internet — changing TCP's fundamental behavior isn't practically deployable at internet scale. Building fresh on UDP (which has no built-in ordering/reliability guarantees to work around) let QUIC implement exactly the semantics it needs — per-stream loss handling, integrated TLS — without fighting decades of entrenched TCP behavior.

> [!question]- What's the practical impact of DNS TTL being too high during an incident?
> If a server's IP needs to change urgently (failover, decommission) but the DNS TTL was set high, clients and resolvers keep routing to the old, now-wrong IP until their cached TTL expires — a real, avoidable extension of downtime. It's why critical records are often kept at low TTLs, or lowered proactively ahead of a planned migration.

> [!question]- Where does a CDN fit into this chain?
> A CDN typically returns *different* IPs to clients in different geographic locations for the *same* hostname (via DNS-based geo-routing or anycast) — see [[CS Fundamentals/02 - Networking/CDN Internals|CDN Internals]] for the full mechanism.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[CS Fundamentals/02 - Networking/TCP Deep Dive|TCP Deep Dive]] · [[CS Fundamentals/02 - Networking/CDN Internals|CDN Internals]]*
