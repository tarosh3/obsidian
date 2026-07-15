---
title: "Reverse Proxy"
aliases: [Forward Proxy vs Reverse Proxy, Nginx Reverse Proxy]
tags: [system-design, cs-fundamentals, networking]
status: reference-quality
---

# Reverse Proxy

> [!abstract] What you'll be able to do after this chapter
> State the forward-proxy vs. reverse-proxy distinction precisely (a classic interview trap), and explain exactly how Reverse Proxy, Load Balancer, and API Gateway relate — three names for overlapping capability, not three competing architectures.

> [!info] Why this chapter exists despite Load Balancing and API Gateway already being written
> [[CS Fundamentals/02 - Networking/Load Balancing|Load Balancing]] and [[CS Fundamentals/02 - Networking/API Gateway|API Gateway]] are both, technically, **specific applications of the reverse-proxy concept** — this chapter is the general idea underneath both, and exists specifically to draw the boundaries between all three precisely, since conflating them is one of the most common imprecisions in system design interviews.

---

## What is it, and why does it exist?

A reverse proxy is a server that sits in front of one or more backend servers, receiving client requests on the backends' behalf and forwarding them, then returning the response back to the client as if the proxy itself had generated it. The client never talks to the backend directly, and often never even knows the backend exists.

**The problem this solves:** exposing backend servers' real addresses directly to the internet is a security liability (attackers can target them directly), makes it impossible to add uniform behavior (TLS termination, compression, caching) without modifying every single backend, and tightly couples clients to specific server addresses — you can't swap, scale, or relocate a backend without breaking every client that hardcoded its address.

> [!example] Layman analogy
> A hotel receptionist. Guests never call a room's direct extension — they call the front desk, and the receptionist routes the call to whichever room the guest actually needs. The hotel can move a guest to a different room entirely, and callers never notice, because they were never talking to the room directly in the first place.

## Forward proxy vs. reverse proxy — the precise distinction

> [!warning] A classic, easy-to-get-backward interview trap
> A **forward proxy** sits in front of **clients** — it hides the client's identity from the servers they're talking to (a corporate proxy, a VPN, a privacy tool). The server doesn't know which specific client it's really talking to. A **reverse proxy** sits in front of **servers** — it hides the backend's identity/topology from clients. The client doesn't know which specific server actually handled its request. Same underlying mechanism (something in the middle, forwarding traffic), opposite side of the conversation it's protecting.

```mermaid
graph LR
    subgraph "Forward Proxy — hides the CLIENT"
    C1["Client"] --> FP["Forward Proxy"] --> S1["Server<br/>(doesn't know which client)"]
    end
    subgraph "Reverse Proxy — hides the SERVER"
    C2["Client<br/>(doesn't know which server)"] --> RP["Reverse Proxy"] --> S2["Backend Server"]
    end
```

## Internal working

A request arrives at the reverse proxy's public IP. The proxy inspects it (path, host header, headers generally), decides where it should go, forwards it to the appropriate backend, and relays the response back — optionally terminating TLS along the way, caching the response, compressing it, or rewriting headers, all without the backend needing to implement any of that itself.

```mermaid
sequenceDiagram
    participant C as Client
    participant RP as Reverse Proxy
    participant B as Backend Server

    C->>RP: HTTPS request
    RP->>RP: terminate TLS
    RP->>B: plain HTTP (internal network)
    B-->>RP: response
    RP->>RP: optionally cache/compress
    RP-->>C: HTTPS response
```

## Reverse Proxy vs. Load Balancer vs. API Gateway, precisely

> [!tip] The actual, high-value content of this chapter
> All three are reverse proxies. The difference is what problem each is **specialized** for solving — and a single piece of software (Nginx, Envoy, HAProxy) can play all three roles simultaneously depending on configuration.

| | Reverse Proxy | Load Balancer | API Gateway |
|---|---|---|---|
| **General concept** | Anything forwarding traffic on behalf of servers | A reverse proxy specialized for distributing load across **many identical replicas** of one service | A reverse proxy specialized for **API-specific concerns**: auth, rate limiting, routing to **different services** by path |
| **Typical decision** | "Where does this request go?" | "Which of these N identical instances should handle this?" | "Which *service* should handle this, and is the caller even allowed to?" |
| **Covered in depth** | This chapter | [[CS Fundamentals/02 - Networking/Load Balancing\|Load Balancing]] | [[CS Fundamentals/02 - Networking/API Gateway\|API Gateway]] |

## Tradeoffs

Adding a reverse proxy adds a network hop (real, if usually small, latency cost) and introduces a new component that must itself be made redundant — a single reverse-proxy instance is exactly the kind of single point of failure [[00 - Start Here/100 System Design Interview Questions|the 100-questions file]] warns about naming explicitly. In exchange: backend isolation, uniform TLS/caching/compression handling, and the ability to change backend topology without touching any client.

## Production usage

Nginx, HAProxy, and Envoy are the standard real-world reverse-proxy implementations — often serving static assets directly from the proxy layer (never even reaching a backend for a cached, unchanging file), a real, common performance optimization worth naming.

## Scaling: 1 backend to global deployment

```mermaid
flowchart TD
    A["1 backend<br/>no reverse proxy needed"] --> B["A few backends<br/>single reverse-proxy<br/>instance is fine"]
    B --> C["Many backends, high traffic<br/>the proxy itself needs<br/>redundancy — the SPOF<br/>warning above, made concrete"]
    C --> D["Global scale<br/>reverse proxies deployed<br/>PER REGION, combined with<br/>geographic routing (Load Balancing/CDN)"]
```

## Failure scenarios

> [!bug] What actually happens
> - **The reverse proxy instance itself crashes:** every backend behind it becomes unreachable, **even though those backends are perfectly healthy** — the exact SPOF risk already named, made concrete: the failure is entirely at the proxy layer, invisible to any backend-level health check.
> - **A misconfigured routing rule:** traffic gets silently sent to the wrong backend, or dropped entirely — a real, often slow-to-detect class of incident, since the individual services involved may show no errors of their own at all.
> - **TLS certificate expiry at the proxy:** since TLS termination is centralized (the exact benefit named in "why this exists"), a single expired certificate can take down access to *every* backend behind that proxy simultaneously — a real, deliberate consequence of the same centralization that reduces day-to-day cert-management effort.

## Monitoring

> [!info] What to watch
> **Proxy-added latency** — every request pays this overhead, making it a direct, always-relevant metric. **Backend health status as seen by the proxy** — the proxy's own view of backend health, which can diverge from what the backend believes about itself during a routing misconfiguration. **TLS certificate expiry date** — a simple, often-overlooked monitoring gap given the outsized blast radius of missing a renewal at this centralized layer.

## Common mistakes

> [!warning] Real, recurring errors
> 1. **Running a single reverse-proxy instance with no redundancy** — the SPOF risk named throughout this chapter.
> 2. **Not monitoring certificate expiry centrally** — the same centralization that simplifies cert management also means a missed renewal has a larger blast radius than it would per-backend.
> 3. **Overloading the reverse proxy with business logic** — the same "gateway becomes a monolith" anti-pattern named in the [[CS Fundamentals/02 - Networking/API Gateway|API Gateway chapter]] applies here too, since a reverse proxy is the more general case that gateway specializes.

---

## Interview Q&A

> [!info] Leveled by seniority
> **Beginner:** "What does a reverse proxy do?" — forwards client requests to backend servers on their behalf, hiding backend topology from clients. **Intermediate:** "What's the difference between a forward proxy and a reverse proxy?" — Section 2's precise distinction: which side of the conversation is being hidden. **Senior:** "Every backend behind a reverse proxy is reporting healthy, but users report the whole site is down — diagnose it." — expects checking the proxy layer itself first (Failure Scenarios), since backend-level health checks can't detect a proxy-layer failure. **Staff:** "Design certificate management for a fleet of reverse proxies across multiple regions." — expects centralized, automated renewal (not per-instance manual management) precisely because of the outsized blast radius named above, plus monitoring expiry dates as a first-class metric. **Architect:** "How would you decide whether a system needs a dedicated reverse proxy layer versus relying purely on a service mesh for internal routing?" — expects the north-south vs. east-west distinction from the API Gateway chapter, applied generally: a reverse proxy for external-facing traffic, a mesh for internal service-to-service traffic, often both present simultaneously rather than one replacing the other.

> [!question]- Is a CDN edge node a reverse proxy?
> Functionally, yes — a CDN edge server receives client requests and either serves a cached response directly or forwards to origin on the client's behalf, exactly the reverse-proxy shape, specialized further for geographic distribution and caching. Worth naming this overlap explicitly if asked — see [[CS Fundamentals/02 - Networking/CDN Internals|CDN Internals]].

> [!question]- If Nginx can act as a load balancer AND a reverse proxy AND (with more config) an API gateway, why does this handbook have three separate chapters?
> Because the *configuration and reasoning* differs meaningfully even when the software doesn't — deciding "route by path to a different service with an auth check" (gateway) is a genuinely different design problem than "spread traffic evenly across N replicas of the same service" (load balancer), even if the same binary implements both. Precision about which problem you're solving matters more than which tool you'd reach for.

> [!question]- Why terminate TLS at the reverse proxy instead of at each backend server?
> Centralizes certificate management to one place instead of N backends each needing their own certs and renewal process, and offloads the CPU cost of the TLS handshake from application servers — the same reasoning already covered in the API Gateway chapter, since TLS termination is one of the capabilities that chapter's "API Gateway" role includes.

## Summary / Cheat Sheet

- **Forward proxy** hides the **client** from the server. **Reverse proxy** hides the **server** from the client. Opposite sides.
- **Load Balancer** = reverse proxy specialized for spreading load across identical replicas.
- **API Gateway** = reverse proxy specialized for API concerns (auth, rate limiting, routing to different services).
- One piece of software can be all three at once — the distinction is about **problem being solved**, not separate technologies.

---
*Related: [[CS Fundamentals/00 - Learning Path|CS Fundamentals Learning Path]] · [[CS Fundamentals/02 - Networking/Load Balancing|Load Balancing]] · [[CS Fundamentals/02 - Networking/API Gateway|API Gateway]] · [[CS Fundamentals/02 - Networking/CDN Internals|CDN Internals]]*
