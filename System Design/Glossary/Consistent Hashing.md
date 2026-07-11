---
title: "Consistent Hashing"
aliases: [Consistent Hashing, Hash Ring]
tags: [system-design, glossary]
---

# Consistent Hashing

> [!info] Quick definition
> A hashing scheme mapping both data keys and servers onto a conceptual ring — so adding or removing a server only remaps a **small fraction** of keys, instead of the near-total remap a naive `hash(key) % N` causes when `N` changes.

Each server (often as multiple **virtual nodes** for better load balance) sits at a position on the ring; a key is assigned to the next server clockwise from its hash position. Add a server → it only takes over keys between it and its counterclockwise neighbor. This is *the* algorithm behind sharding, distributed caching, and consistent load balancing at scale.

> [!tip] This is a glossary stub
> Full explanation with ring diagrams, virtual-node mechanics, and the Redis Cluster contrast: [[CS Fundamentals/Distributed Systems/Consistent Hashing|Consistent Hashing]].

**Used in:** [[CS Fundamentals/Distributed Systems/Consistent Hashing|Consistent Hashing]], [[HLD/03 - Design a Distributed Cache (build Redis)/Design a Distributed Cache|Design a Distributed Cache]], [[CS Fundamentals/Databases/Cassandra Internals|Cassandra Internals]]
