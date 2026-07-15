---
title: "Cassandra Internals"
aliases: [Cassandra, Cassandra Internals, Wide-Column Store]
tags: [system-design, cs-fundamentals, databases, cassandra, lsm-tree]
status: reference-quality
---

# Cassandra Internals

> [!abstract] What you'll be able to do after this chapter
> Explain Cassandra's write path precisely (why it's so write-optimized), deliver the LSM-Tree depth [[CS Fundamentals/03 - Databases/Indexes & B+ Trees|the B+ Tree chapter]] promised as the "when B+ Trees are wrong" answer, and name the tombstone problem as the concrete reason "Cassandra as a queue" is a well-known anti-pattern.

---

## 1. Why Cassandra exists

Built at Facebook (for inbox search) by combining two ideas: **Amazon Dynamo's** distributed architecture (peer-to-peer, no single leader, tunable consistency) with **Google Bigtable's** data model (wide-column, sparse, sorted). The goal: a database that stays available and keeps accepting writes across multiple data centers, with no single point of failure, at a scale where node failures are a routine, expected event rather than an exception.

## 2. The data model — wide-column, not relational

A Cassandra "row" is identified by a **partition key**; within a partition, data is organized into columns that can genuinely vary row-to-row (sparse), sorted by **clustering columns**. The partition key determines **which node(s)** own the data — via [[Glossary/Consistent Hashing|consistent hashing]] of the key onto the cluster ring (with virtual nodes/"vnodes" for even load distribution). Clustering columns determine the **physical sort order within that partition on disk** — meaning a well-chosen clustering key makes range queries within a partition genuinely fast, a design decision made at schema time, not query time.

## 3. The write path — LSM-Trees, and the depth the B+ Tree chapter promised

> [!tip] This delivers on [[CS Fundamentals/03 - Databases/Indexes & B+ Trees|Indexes & B+ Trees]]'s "when B+ Trees are the wrong choice" pointer
> Cassandra is built on an **LSM-Tree** (Log-Structured Merge-Tree), the write-optimized counterpart to the B+ Tree's read/range-optimized design.

The mechanics: a write goes to an in-memory **memtable**, plus a **commit log** on disk (a pure sequential append, for durability — if the process crashes, the commit log replays to rebuild the memtable). **No read-before-write, no random disk I/O on the write path at all** — this is the entire reason Cassandra is so write-optimized. Periodically, a full memtable is flushed to disk as an immutable **SSTable** (Sorted String Table). A background **compaction** process merges multiple SSTables over time, discarding data that's been overwritten or deleted (see tombstones, below).

```mermaid
graph LR
    W["Write"] --> M["Memtable<br/>(in-memory)"]
    W --> CL["Commit Log<br/>(sequential append, durability)"]
    M -->|"periodic flush"| S1["SSTable 1<br/>(immutable, on disk)"]
    M -->|"periodic flush"| S2["SSTable 2<br/>(immutable, on disk)"]
    S1 -.->|"background compaction"| S3["Merged SSTable<br/>(old versions discarded)"]
    S2 -.-> S3
```

**The read-side cost of this tradeoff:** a read might need to check the memtable *and* multiple SSTables to find the latest value for a key. Mitigated by a **[[Glossary/Bloom Filter|Bloom filter]] per SSTable** — a cheap, no-false-negative check to quickly rule out SSTables that definitely don't contain the key, avoiding unnecessary disk reads.

## 4. Tunable consistency — delivering on [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC|the CAP/PACELC chapter's]] promise

Cassandra's signature feature: **per-query consistency level** (`ONE`, `QUORUM`, `ALL`, `LOCAL_QUORUM` for multi-datacenter setups). This is [[Glossary/Quorum (R + W over N)|the quorum mechanism]], made directly configurable — a single application can read with `ONE` (fast, weaker) for a low-stakes lookup and `QUORUM` (stronger, slower) for a critical read, on the same cluster, chosen per query rather than fixed system-wide.

## 5. No leader — gossip and consensus for cluster coordination

Cassandra is **peer-to-peer / leaderless** — no single node coordinates the cluster. Cluster membership and failure detection run via the [[Glossary/Gossip Protocol|gossip protocol]] — nodes periodically exchange state with random peers, propagating liveness information epidemic-style without any central coordinator.

## 6. Tombstones — the deletion problem, and why "Cassandra as a queue" is a known anti-pattern

> [!bug] The genuinely famous Cassandra gotcha
> A delete doesn't remove data immediately — it writes a **tombstone** marker, which must persist for a configured grace period (long enough to propagate to all replicas) before compaction can permanently remove it. A workload with **frequent deletes** — a queue-like access pattern being the classic example — accumulates large numbers of tombstones, and reads must scan **past** them to find live data, causing real, measurable read latency degradation. This specific mechanism is *why* "don't use Cassandra as a queue" is such a widely-repeated piece of advice — it's not folklore, it's a direct consequence of how deletion actually works internally.

## 7. When NOT to use Cassandra

Needing traditional **multi-row ACID transactions or joins** — Cassandra has no native support for either in the relational sense (lightweight transactions via Paxos exist, but only for narrow compare-and-swap use cases, not general multi-statement transactions). Needing **strong consistency by default** rather than opted into per-query. Workloads with **frequent deletes** (the tombstone problem above). Small scale, where the operational complexity of running and tuning a Cassandra cluster isn't justified by the workload.

## 8. Scaling: 1 user to 1 billion

```mermaid
flowchart TD
    A["Small cluster<br/>a few nodes,<br/>replication factor 3 typical"] --> B["Growing<br/>add nodes — vnodes auto-rebalance<br/>via consistent hashing, NO manual<br/>resharding needed unlike traditional SQL"]
    B --> C["Massive scale<br/>multi-datacenter deployments,<br/>NetworkTopologyStrategy,<br/>LOCAL_QUORUM avoids cross-DC latency"]
    C --> D["Global scale<br/>per-operation consistency-level<br/>choice becomes essential — not every<br/>read/write can afford cross-region coordination"]
```

A genuinely distinguishing property vs. traditional sharded SQL: adding a Cassandra node doesn't require a separate, deliberate resharding *process* — the consistent-hashing ring (with vnodes) automatically absorbs the new node, redistributing a bounded fraction of the keyspace to it, the exact mechanism already covered generally in [[CS Fundamentals/06 - Distributed Systems/Consistent Hashing|Consistent Hashing]]. At multi-datacenter scale, `NetworkTopologyStrategy` lets replication be placement-aware (e.g., 3 replicas per datacenter), and `LOCAL_QUORUM` lets a query require agreement only from replicas in the *local* datacenter, avoiding the latency cost of coordinating across a continent for every operation. At true global scale, the per-query consistency-level choice (Section 4) stops being a nice feature and becomes an essential, deliberate decision for every operation — not every read or write can afford to pay cross-region coordination latency.

## 9. Failure scenarios

> [!bug] What actually happens, precisely — including a Cassandra-specific mechanism
> - **A node goes down:** gossip detects it; other replicas continue serving per the configured consistency level — no single point of failure, the direct payoff of the leaderless design from Section 5.
> - **A whole datacenter goes down** (multi-DC deployment): other datacenters continue serving if `NetworkTopologyStrategy` was configured with cross-DC replication — a genuinely different failure domain than a single node, protected against by a genuinely different mechanism (replication topology, not just node-level redundancy).
> - **Hinted handoff — a real, Cassandra-specific recovery mechanism:** when a replica is temporarily unreachable during a write, the coordinator node stores a "hint" (the pending write) and replays it to the replica once it recovers — bounded recovery from a *brief* outage without requiring a full repair, worth naming explicitly as distinct from the longer-running anti-entropy repair process for larger gaps.

## 10. Monitoring

> [!info] What to watch
> **Read/write latency, broken down per consistency level** — since different operations may use different levels, an aggregate latency number hides real per-level differences. **Compaction backlog** — falling behind on compaction means more SSTables to check per read, directly degrading read latency over time. **Tombstone count/ratio** — the direct, measurable early-warning signal for Section 6's read-degradation problem, before it becomes customer-visible. **Hinted-handoff backlog** — a growing backlog signals a node has been down long enough that hint storage itself is becoming a real resource concern.

## 11. Common mistakes

> [!warning] Real, recurring errors
> 1. **Using Cassandra as a queue** — Section 6 covers this precisely; frequent deletes accumulate tombstones faster than compaction can clear them.
> 2. **Choosing `ALL` consistency by default "to be safe"** — defeats much of the availability benefit the whole architecture is built around; `QUORUM` is usually the right default, with `ALL` reserved for genuinely critical operations.
> 3. **Poor partition key choice causing unbounded-wide partitions** — a partition that grows without bound (e.g., keyed only by a low-cardinality value with unlimited clustering rows) is a real, distinct anti-pattern from MongoDB's hot-shard problem, causing its own performance degradation as a single partition grows too large to handle efficiently.
> 4. **Not planning clustering-column sort order at schema design time** — this is a physical, on-disk decision made once at table creation, not something easily changed later without a data migration.

---

## 🎯 Interview follow-up Q&A

> [!info] Leveled by seniority
> **Beginner:** "What kind of database is Cassandra?" — a wide-column, leaderless, LSM-Tree-based store built for write-heavy, always-available workloads. **Intermediate:** "Why is Cassandra fast at writes?" — Section 3, no read-before-write, purely sequential I/O on the write path. **Senior:** "A Cassandra table's read latency has been climbing steadily — diagnose it." — expects checking tombstone ratio and compaction backlog first, the two direct, named causes of gradual read-latency degradation in this chapter. **Staff:** "Design the replication and consistency strategy for a Cassandra deployment spanning 3 datacenters on different continents." — expects `NetworkTopologyStrategy` and `LOCAL_QUORUM` named explicitly, with a clear articulation of which operations can tolerate local-only consistency vs. which need cross-DC guarantees. **Architect:** "How would you decide between Cassandra and a sharded relational database for a new write-heavy system?" — expects a real tradeoff: Cassandra trades relational query flexibility (joins, ACID transactions) for write throughput and operational simplicity of scaling (no manual resharding), the right call specifically when the access patterns are known upfront and don't need relational flexibility.

> [!quote]- "Why is Cassandra so much faster at writes than a typical relational database?"
> Writes never touch the read path at all — they go to an in-memory memtable plus a purely sequential commit-log append. No read-before-write, no random disk I/O, no index maintenance blocking the write.
>
> **Follow-up: "What's the cost of that design on the read side?"**
> A read may need to check the memtable and multiple on-disk SSTables to assemble the latest value for a key — mitigated, but not eliminated, by per-SSTable Bloom filters that cheaply skip SSTables known not to contain the key.

> [!quote]- "How does Cassandra's tunable consistency actually work?"
> Each query specifies a consistency level (`ONE`, `QUORUM`, `ALL`, etc.) that determines how many replicas must respond before the operation is considered successful — directly implementing the `R + W > N` quorum math, chosen per-query rather than fixed for the whole system.

> [!quote]- "Why is Cassandra known to be a bad fit for queue-like workloads?"
> Deletes create tombstones that must persist for a grace period before being compacted away — a workload with frequent deletes (exactly what a queue does constantly) accumulates tombstones faster than compaction can clean them up, and reads have to scan past them, causing real, measurable latency degradation over time.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[CS Fundamentals/03 - Databases/Indexes & B+ Trees|Indexes & B+ Trees]] · [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC|CAP Theorem & PACELC]] · [[Glossary/Bloom Filter|Bloom Filter]] · [[Glossary/Gossip Protocol|Gossip Protocol]] · [[Glossary/Quorum (R + W over N)|Quorum]]*
