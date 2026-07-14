---
title: "MongoDB Internals"
aliases: [MongoDB, MongoDB Internals, WiredTiger, Document Database]
tags: [system-design, cs-fundamentals, databases, mongodb]
status: reference-quality
---

# MongoDB Internals

> [!abstract] What you'll be able to do after this chapter
> Explain why the document model reduces joins by design (not by accident), describe WiredTiger's concurrency improvement over the old MMAPv1 engine precisely, and name the exact shard-key mistake that creates a hot shard.

---

## 1. Why MongoDB exists

Relational schemas are rigid — every row in a table must match the same column structure, and evolving that structure at scale means real migrations. MongoDB's pitch: store data as flexible, nested **documents** (BSON — binary JSON) that don't require a uniform schema across a collection, letting applications iterate quickly without constant schema migrations. Related data can be **nested inside one document** (arrays, sub-documents) instead of requiring a join across separate tables — the document model is, in effect, denormalization-by-default, trading storage duplication and update complexity for read-path simplicity, the same fundamental tradeoff normalization-vs-denormalization always makes, just baked into the storage model itself.

## 2. Data model — collections of documents, not tables of rows

A **collection** groups documents loosely (analogous to a table), but individual documents within it can have different fields — modern MongoDB supports **optional schema validation** for teams that want structure back, but it's opt-in, not the default. Documents can nest arrays and sub-documents arbitrarily deep, letting a single read fetch a fully-formed object without joining across collections at all.

## 3. Storage engine — WiredTiger, and the concurrency leap it represented

**WiredTiger** is the modern default storage engine (replacing the original **MMAPv1**). It's B-Tree based internally (with an optional LSM-tree configuration available), and adds **document-level concurrency control** with compression (snappy/zlib by default, reducing storage footprint).

> [!bug] A real historical detail worth knowing
> MMAPv1 used much **coarser locking** — historically collection-level, and even earlier, database-level — meaning concurrent writes to *different* documents in the same collection could still block each other. WiredTiger's move to document-level locking was a genuine, significant concurrency improvement, not a marginal tuning change — worth naming specifically if asked about MongoDB's concurrency history, since it shows awareness of how the system actually evolved rather than describing only its current state.

## 4. Replication — replica sets and automatic failover

MongoDB uses **replica sets**: one primary, multiple secondaries — conceptually the same leader-follower replication pattern covered generally elsewhere, with MongoDB-specific mechanics: automatic **failover** via an election protocol among secondaries when the primary becomes unreachable, conceptually similar to [[Glossary/Raft (Consensus)|Raft-style consensus]] (a distinct implementation, but the same underlying idea — a majority of nodes must agree on the new primary). Reads can be routed to secondaries (trading consistency for read scalability — a real, direct [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC|PACELC]] tradeoff) or required to be majority-committed for stronger consistency guarantees, at added latency cost.

## 5. Sharding — and the shard-key mistake that creates a hot shard

Data is partitioned across multiple replica sets (**shards**) using a **shard key** — the MongoDB-specific detail worth internalizing is that **choosing a good shard key is the single highest-leverage decision** in a sharded MongoDB deployment.

> [!bug] The concrete mistake, named precisely
> Choosing a **monotonically increasing** shard key (a timestamp, an auto-incrementing ID) means **every new write lands on the same shard** — the one currently responsible for the highest key range — creating a hot shard while every other shard sits idle for writes. This is the exact same hot-key/hot-partition failure mode already covered for caches and consistent hashing, just showing up in the context of a shard-key choice instead.

MongoDB offers two sharding strategies with a genuine tradeoff: **hashed sharding** (hashes the shard key, distributing writes evenly — but destroys the shard key's natural ordering, making efficient range queries across the sharded field impossible) vs **ranged sharding** (preserves natural ordering for efficient range queries — but is exactly what creates the hot-shard risk described above if the key is monotonically increasing).

## 6. When NOT to use MongoDB

Needing **multi-document ACID transactions across many collections at high throughput** — supported since MongoDB 4.0, but with a real, non-trivial performance cost; worth stating precisely rather than the outdated "MongoDB doesn't support transactions" claim, which is simply wrong for modern versions. **Highly relational data with frequent many-to-many joins** — MongoDB's `$lookup` aggregation stage exists, but isn't as efficient as native relational joins at scale. Needing **strict schema enforcement** as a first-class, always-on guarantee rather than an opt-in validation layer.

---

## 🎯 Interview follow-up Q&A

> [!quote]- "Why does MongoDB's document model reduce the need for joins?"
> Related data can be nested directly inside one document (arrays, sub-documents) instead of living in separate tables that need to be joined at query time — a single document read returns a fully-formed object. This is denormalization built into the storage model itself, trading some data duplication and update complexity for simpler, faster reads.

> [!quote]- "What changed with the move from MMAPv1 to WiredTiger?"
> Locking granularity — MMAPv1's coarser (historically collection-level, even earlier database-level) locking meant concurrent writes to different documents in the same collection could still contend; WiredTiger's document-level locking removed that unnecessary contention, a genuine concurrency improvement rather than an incremental tuning change.

> [!quote]- "What makes a bad MongoDB shard key, concretely?"
> A monotonically increasing key (timestamp, auto-increment ID) — every new write targets the shard currently owning the highest key range, concentrating all write load onto one shard while the rest of the cluster sits idle for writes.
>
> **Follow-up: "How would you fix that while still supporting range queries on that field?"**
> A common approach is a **compound shard key** combining a well-distributed prefix (e.g. a hashed or bucketed field) with the naturally-ordered field as a secondary component — spreading writes across shards while still allowing efficient range scans within each shard's slice of the ordered field.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC|CAP Theorem & PACELC]] · [[Glossary/Raft (Consensus)|Raft]] · [[HLD/03 - Design a Distributed Cache (build Redis)/Design a Distributed Cache|Design a Distributed Cache]] (hot-key parallel)*
