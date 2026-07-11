---
title: "Redis Internals"
aliases: [Redis, Redis Internals]
tags: [system-design, cs-fundamentals, caching, redis]
status: reference-quality
---

# Redis Internals

> [!abstract] What you'll be able to do after this chapter
> Explain why single-threaded Redis outperforms most multi-threaded stores for its workload, pick the right data structure for a given problem (not just "use a string"), justify RDB vs AOF, and explain hash-slot sharding precisely enough to contrast it with generic [[Glossary/Consistent Hashing|consistent hashing]].

---

## 1. Why Redis exists

A generic cache-in-front-of-a-database gets you fast key-value lookups, but a lot of real application logic — "top 10 scores," "is this user in the set of banned IDs," "push this job onto a queue" — still requires pulling data *out* of the cache and processing it in the application. Redis's actual pitch is different: it's an **in-memory data structure server** — push that logic **down into the data layer** by giving you rich, purpose-built types (not just strings) with atomic operations on them.

## 2. Single-threaded execution — and why that's a feature, not a limitation

> [!warning] Precision matters here
> Redis's **command execution** is single-threaded. Since Redis 6+, **I/O** (reading/writing client sockets) can be multi-threaded — these are two different things, and conflating them is a common imprecision.

Why single-threaded execution works well:
1. In-memory operations are fast enough that CPU is rarely the bottleneck for typical workloads — the win from parallelizing command execution would be small.
2. It **eliminates lock contention entirely** for command execution — no mutex around shared data structures, no context-switch overhead, no risk of the exact class of [[Glossary/Race Condition|race conditions]] a multi-threaded design would need to guard against.
3. Built on an **event loop** (epoll/kqueue-based I/O multiplexing) — handles many concurrent client connections without one-OS-thread-per-connection.

## 3. Data structures — pick the one that matches the problem

| Type | Backing structure | Real use case |
|---|---|---|
| **String** | Binary-safe byte string | Simple caching, atomic counters (`INCR`) |
| **List** | Linked list | Queues (`LPUSH`/`RPOP`) |
| **Set** | Hash table | Unique membership, `O(1)` membership checks |
| **Hash** | Hash table of field→value | Representing an object's fields under one key, without needing N separate keys |
| **Sorted Set (ZSET)** | **Skip list + hash table combo** | `O(log n)` insertion *and* `O(log n)` range-by-score queries — the exact data structure behind real-time leaderboards |

> [!tip] The Sorted Set is the interview-favorite detail
> A skip list is a probabilistic, multi-level linked list giving `O(log n)` search without the rebalancing complexity of a tree. Combined with a hash table (for `O(1)` "what's this member's current score" lookups), ZSET gives Redis both fast score-based range queries *and* fast individual-member lookups from one structure — worth naming both halves of the combo, not just "it uses a skip list."

## 4. Persistence — RDB vs AOF, and why you'd use both

- **RDB (snapshot):** a periodic, point-in-time binary dump of the entire dataset. Fast to load on restart, compact on disk. **Data since the last snapshot is lost on crash.**
- **AOF (append-only file):** every write operation is logged; replayed on restart to rebuild state. Configurable fsync policy (`always`/`everysec`/`never`) trades durability directly against write throughput. Larger on disk, slower to replay than RDB (mitigated by periodic AOF rewrite/compaction).

> [!success] Real production configuration
> Running **both together** is common: AOF as the durable source of truth (minimal data loss), RDB snapshots for fast cold-start recovery — the combination trades a bit of disk space for both fast restarts *and* strong durability, rather than picking one weakness to live with.

## 5. Replication & sharding

**Replication:** leader-follower (master-replica), **asynchronous by default** — meaning a follower can lag behind the leader, a real consideration if you're reading from replicas expecting strong consistency. **Redis Sentinel** monitors leader health and automates failover (electing a new leader among replicas) for a single logical dataset.

**Redis Cluster (sharding):** uses **hash slots** — 16,384 fixed slots, each key mapped to a slot via `CRC16(key) mod 16384`, each slot owned by a specific node.

> [!bug] Different mechanism from generic consistent hashing — say this explicitly
> [[Glossary/Consistent Hashing|Consistent hashing]] (as used in Dynamo-style systems) maps keys and nodes onto a continuous ring, remapping only a fraction of keys when a node joins/leaves. Redis Cluster instead uses a **fixed number of discrete slots**, reassigned in whole-slot chunks between nodes during resharding — conceptually related (both minimize full-dataset remapping on topology change) but a genuinely different mechanism, worth distinguishing precisely if asked to compare.

## 6. Why Redis is fast — the actual mechanics

Everything lives in RAM (no disk seek on the read path at all). Single-threaded execution avoids lock overhead entirely. Purpose-built data structures (skip lists, hash tables) keep every operation at its structure's optimal complexity. A simple text protocol (RESP) keeps parsing overhead minimal per command.

## 7. When NOT to use Redis

Dataset far exceeding available RAM — Redis isn't designed as a primary store for huge, mostly-cold datasets; a disk-based store fits better. Complex relational queries/joins — Redis has no query planner or join capability; that's not its job. Needing the strongest possible synchronous durability on *every* write (`AOF fsync=always`) has a real, measurable throughput cost — for that specific critical-path data, a proper WAL-backed relational DB with real replication guarantees may be the better fit.

---

## 🎯 Interview follow-up Q&A

> [!quote]- "Why is Redis fast despite being single-threaded?"
> In-memory access removes disk latency entirely; single-threaded command execution removes lock/context-switch overhead; purpose-built data structures keep every operation near-optimal complexity for its use case.
>
> **Follow-up: "Doesn't single-threaded execution mean one slow command blocks every other client?"**
> Yes — this is a genuine, real production gotcha. An `O(n)` command like `KEYS *` on a large dataset, or an unbounded range query on a huge sorted set, blocks *every* other client for its duration. The fix: avoid `O(n)` blocking commands in production (use `SCAN`, which is cursor-based and incremental, instead of `KEYS`), and be deliberate about which commands can run against large collections.

> [!quote]- "RDB or AOF — which would you use?"
> Depends on the durability/performance tradeoff needed — but in practice, most production Redis deployments use **both together**: AOF for durability, RDB for fast cold-start recovery.
>
> **Follow-up: "Why would you use both instead of picking the stronger one (AOF) alone?"**
> AOF replay on restart is slower than loading an RDB snapshot — combining both means restarting from the (fast) RDB snapshot and then replaying only the (smaller) AOF segment written since that snapshot, getting both fast recovery and strong durability rather than trading one for the other.

> [!quote]- "How does Redis Cluster shard data across nodes?"
> Via 16,384 fixed hash slots, each key assigned to a slot by `CRC16(key) mod 16384`, and each slot owned by a specific node.
>
> **Follow-up: "How is this different from consistent hashing?"**
> Consistent hashing maps keys and nodes onto a continuous ring and remaps only a thin slice of keys near a topology change. Redis Cluster instead reassigns whole discrete slots between nodes during resharding — a fixed, coarser-grained partitioning scheme rather than a continuous ring.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[CS Fundamentals/Caching/Caching Strategies|Caching Strategies]] · [[Glossary/Consistent Hashing|Consistent Hashing]]*
