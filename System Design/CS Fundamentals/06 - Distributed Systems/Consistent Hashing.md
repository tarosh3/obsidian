---
title: "Consistent Hashing"
aliases: [Consistent Hashing, Hash Ring, Virtual Nodes]
tags: [system-design, cs-fundamentals, distributed-systems, consistent-hashing]
status: reference-quality
---

# Consistent Hashing

> [!abstract] What you'll be able to do after this chapter
> Quantify precisely *why* modulo hashing catastrophically fails on scaling events, draw the ring and virtual-node mechanics from memory, and contrast this against Redis Cluster's genuinely different hash-slot approach with precision.

> [!info] Referenced everywhere in this handbook
> This chapter delivers the full depth already promised by the [[Glossary/Consistent Hashing|glossary stub]] and used throughout [[HLD/03 - Design a Distributed Cache (build Redis)/Design a Distributed Cache|Distributed Cache]], [[CS Fundamentals/03 - Databases/Cassandra Internals|Cassandra Internals]], and [[HLD/01 - Design TinyURL (URL Shortener)/Design TinyURL|TinyURL]].

---

## 1. Why it exists — the failure of naive modulo hashing

`hash(key) % N` (`N` = number of servers) is the obvious first approach to sharding. It fails catastrophically the moment `N` changes: adding or removing **one** server changes the modulus for **every** key, remapping almost the entire keyspace to a different server at once — a massive, unnecessary cache-miss storm or data-reshuffling event triggered by a routine scaling operation.

> [!bug] Quantify this precisely — a strong depth signal
> Going from `N` to `N+1` servers under modulo hashing remaps **`(N/(N+1))` of all keys** — for `N=10`, that's over 90% of the entire keyspace, just from adding one server. Consistent hashing bounds this to roughly **`1/(N+1)`** of keys on average — a night-and-day difference, and worth stating with the actual fraction rather than just "it's better."

## 2. The mechanism — a ring, not a modulus

Hash **both keys and server identifiers** onto the same circular space (e.g. `0` to `2^32-1`). A key belongs to the **first server encountered going clockwise** from the key's position on the ring.

```mermaid
graph TD
    subgraph "Hash Ring"
        S1(("Server A"))
        S2(("Server B"))
        S3(("Server C"))
    end
    K1["key: 'user:42'<br/>hashes here"] -.clockwise to.-> S2
    K2["key: 'user:99'<br/>hashes here"] -.clockwise to.-> S3
```

**Adding a server:** it claims a position on the ring; only keys between it and its counterclockwise neighbor (previously belonging to that neighbor) move — everything else is completely undisturbed. **Removing a server:** its keys move to its clockwise neighbor; again, nothing else is affected.

## 3. Virtual nodes — the refinement that makes this actually work well

With just **one** ring position per physical server, distribution can be quite **uneven** — especially with few servers, some can randomly end up owning much larger arcs of the ring than others purely by chance.

> [!tip] The fix, and why it also smooths rebalancing
> Give each physical server **many** points on the ring (commonly 100-200 **virtual nodes**), scattering its presence evenly. This dramatically smooths load distribution — and as a second benefit, it also smooths rebalancing: a newly-added server's virtual nodes are scattered around the ring, so it takes many **small** slices from many existing servers instead of one large slice from a single unlucky neighbor.

```mermaid
graph LR
    subgraph "Without virtual nodes"
    A1["Server A: 1 point<br/>owns one large, unpredictable arc"]
    end
    subgraph "With virtual nodes"
    A2["Server A: ~150 points<br/>scattered evenly around the ring"]
    end
```

## 4. Real system usage

The original **Amazon Dynamo** paper introduced this technique; **Cassandra** uses it directly with vnodes (see [[CS Fundamentals/03 - Databases/Cassandra Internals|Cassandra Internals]]); **DynamoDB** is built on the same lineage. Load balancers use consistent hashing for **session affinity** (sticky sessions) without needing a shared session store at all — the same client repeatedly hashes to the same backend.

> [!warning] Redis Cluster does NOT use this — say this precisely if compared
> As covered in [[CS Fundamentals/04 - Caching/Redis Internals|Redis Internals]], Redis Cluster instead uses **16,384 fixed hash slots**, reassigned in whole-slot chunks — a related but genuinely **different** mechanism (discrete, fixed-count slots vs. a continuous ring with virtual nodes). Conflating the two in an interview is a real, catchable imprecision.

## 5. When it's not needed

A **small, static cluster that never scales** doesn't need this — the entire benefit of consistent hashing is handling **dynamic membership changes** gracefully; if `N` never changes, plain modulo hashing's catastrophic-remap cost never actually triggers, and the simpler approach is fine.

---

## 🎯 Interview follow-up Q&A

> [!quote]- "Why does adding one server under naive modulo hashing remap almost everything?"
> Because the modulus itself changes — `hash(key) % N` becomes `hash(key) % (N+1)` for every key simultaneously, and there's no relationship between a key's old assigned server and its new one under a different modulus. Roughly `N/(N+1)` of all keys end up somewhere different.

> [!quote]- "What problem do virtual nodes solve that basic consistent hashing (one point per server) doesn't?"
> Uneven load distribution — with few ring points, some servers can randomly own disproportionately large arcs. Scattering each server across ~100-200 points averages this out, and as a side benefit, makes each rebalancing event spread thinly across many existing servers instead of concentrated on one unlucky neighbor.

> [!quote]- "How does Redis Cluster's approach differ from classic consistent hashing?"
> Redis Cluster uses a fixed 16,384-slot partitioning (`CRC16(key) mod 16384`), with resharding moving whole slots between nodes — a discrete, coarser-grained scheme, not a continuous ring with virtual nodes, even though both solve the same underlying "minimize remapping on topology change" problem.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[CS Fundamentals/04 - Caching/Redis Internals|Redis Internals]] · [[CS Fundamentals/03 - Databases/Cassandra Internals|Cassandra Internals]] · [[HLD/03 - Design a Distributed Cache (build Redis)/Design a Distributed Cache|Design a Distributed Cache]]*
