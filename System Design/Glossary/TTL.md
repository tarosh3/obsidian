---
title: "TTL (Time To Live)"
aliases: [TTL, Time To Live]
tags: [system-design, glossary]
---

# TTL — Time To Live

> [!info] Definition
> How long a piece of data stays valid before it must be refreshed or discarded. Appears everywhere data gets cached or cloned rather than fetched fresh every time.

Two common contexts: a **DNS record's TTL** tells resolvers how long they may cache an IP before re-querying. A **cache entry's TTL** tells the cache when to expire/evict a key automatically. Short TTL = fresher data, more origin load. Long TTL = less load, more risk of serving stale data. Picking TTL is always this one tradeoff, restated per use case.

**Used in:** [[CS Fundamentals/02 - Networking/HTTP Evolution & DNS Resolution|HTTP Evolution & DNS Resolution]] (DNS TTL) · [[CS Fundamentals/04 - Caching/Caching Strategies|Caching Strategies]] (cache TTL) · [[CS Fundamentals/06 - Distributed Systems/Idempotency|Idempotency]] (idempotency-key TTL)
