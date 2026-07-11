---
title: "CDN Internals"
aliases: [Content Delivery Network, CDN Cache Invalidation, Anycast]
tags: [system-design, cs-fundamentals, networking, cdn]
status: reference-quality
---

# CDN Internals

> [!abstract] What you'll be able to do after this chapter
> Explain why CDN cache invalidation is genuinely hard and what the real practical workaround is, and choose correctly between push and pull CDN models for a given content type.

> [!info] Where this already showed up
> [[HLD/09 - Design YouTube - Netflix/Design YouTube - Netflix|The YouTube/Netflix chapter]] and [[HLD/22 - Design Google Meet/Design Google Meet|Google Meet's HLD chapter]] both reach for "put it on a CDN" without unpacking the mechanism. This is that mechanism, in full.

---

## Why this exists

Serving every request from one origin server means every user, everywhere, pays the network latency to reach that one location — a user in Mumbai fetching from a server in Virginia pays real, physical speed-of-light latency no amount of server-side optimization can remove. A CDN solves this by **replicating content to edge servers geographically close to users**, so requests are served from nearby infrastructure instead of crossing the globe.

## Push vs. pull CDN

- **Push:** the origin proactively uploads content to edge servers ahead of time. Suits content known in advance and rarely changing (a video's transcoded renditions, static site assets) — the origin controls exactly what's distributed and when.
- **Pull:** edge servers fetch content from the origin **on first request**, then cache it for subsequent requests (a **cache miss** triggers an origin fetch; **cache hits** serve directly from the edge). Suits large or unpredictable catalogs where pre-pushing everything would waste edge storage on content that's never actually requested from that region.

## Anycast routing — the "which edge" mechanism

The same IP address is announced from multiple physical locations; the network layer's routing naturally sends each client's traffic to the **topologically nearest** location advertising that address — no application-level logic decides this, it falls out of how internet routing already works. This is the same idea [[CS Fundamentals/Networking/Load Balancing|the Load Balancing chapter]] mentioned for LB redundancy, applied here to route users to the nearest edge location rather than to a fleet.

## The hard problem: cache invalidation

> [!bug] Purging a CDN is slow, unreliable, and often not worth attempting
> Content lives on potentially thousands of edge servers worldwide. Explicitly purging a specific object means propagating that purge to every edge that might have cached it — slow, and some edges may be temporarily unreachable when the purge fires, leaving stale content served from that location until its TTL naturally expires anyway.
>
> **The real, standard practical fix: versioned URLs (cache-busting), not purging.** Instead of updating `styles.css` in place and trying to invalidate every cached copy, deploy `styles-a3f9c2.css` (a new URL, typically hash-of-content or a version number) and update references to point at it. The old URL's stale cached copies become irrelevant — nothing points at them anymore — and they simply age out of cache naturally. The new URL is a guaranteed cache miss everywhere, fetched fresh from origin exactly once per edge, then cached correctly from that point on. This sidesteps invalidation entirely rather than solving it.

For content that genuinely can't be versioned (a live API response, personalized content), CDNs use **short TTLs** instead — accepting that changes take up to the TTL to propagate, a deliberate consistency-for-latency tradeoff, the same shape of tradeoff seen throughout this book (BookMyShow's seat-map browsing cache, the E-commerce search index).

## Edge compute — CDNs serving more than static files

Modern CDNs (Cloudflare Workers, Lambda@Edge) run **actual application code** at edge locations, not just cached bytes — enabling things like A/B test routing, auth checks, or request rewriting to happen at the edge, closer to the user, before the request ever reaches origin infrastructure. Worth naming as the direction CDNs have moved beyond pure static-asset caching, without needing deep implementation detail for an interview.

---

## Interview Q&A

> [!question]- Why not just purge the CDN cache immediately whenever content changes?
> Purges must propagate to every edge location that might hold a copy, which is slow and not instantaneously reliable across a globally distributed fleet — some edges can be temporarily unreachable or slow to receive the purge. Versioned URLs sidestep the problem entirely: there's nothing to purge, since the old URL is simply never referenced again and ages out naturally.

> [!question]- When would you choose push over pull?
> When the full content set is known ahead of time and small enough to pre-distribute — a video's finished transcoded renditions ([[HLD/09 - Design YouTube - Netflix/Design YouTube - Netflix|from the YouTube chapter]]) are a good fit, since they're finalized before any viewer requests them and benefit from being ready at the edge before first request rather than paying a cold cache-miss penalty for the first viewer in each region.

> [!question]- How does a CDN help with something interactive, like Google Meet, rather than just static files?
> [[HLD/22 - Design Google Meet/Design Google Meet|Google Meet's HLD chapter]] uses CDN infrastructure specifically for **large webinars** — broadcasting one stream to a huge passive audience is structurally similar to video-on-demand distribution (one piece of content, many viewers, cacheable), unlike the interactive many-to-many SFU path used for regular meeting participants, which is genuinely real-time and not cacheable at all.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/09 - Design YouTube - Netflix/Design YouTube - Netflix|Design YouTube / Netflix]] · [[CS Fundamentals/Networking/Load Balancing|Load Balancing]] · [[CS Fundamentals/Networking/HTTP Evolution & DNS Resolution|HTTP Evolution & DNS Resolution]]*
