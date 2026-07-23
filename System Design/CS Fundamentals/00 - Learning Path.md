---
title: "CS Fundamentals Learning Path"
aliases: [CS Fundamentals Order, Foundation Handbook, Pre-System-Design Curriculum]
tags: [system-design, cs-fundamentals, index, meta]
status: living-document
---

# 📚 CS Fundamentals Learning Path

> [!abstract] What this is
> Not a folder listing — a **dependency-ordered curriculum**. Each tier below assumes you've internalized the tiers above it. This is the fix for "CS Fundamentals feels unordered": read top to bottom, not alphabetically by folder name. By the end, you should be able to understand almost any HLD or LLD chapter in this handbook without a knowledge gap forcing you to backtrack.

> [!success] All 12 tiers complete
> Tiers 0-11 are fully written — Computer Architecture through Software Design Principles, plus Search Engines, including Storage Engines, Service Discovery, the full Architecture & Deployment Patterns tier (Monolith vs. Microservices, Event-Driven Architecture, Docker, Kubernetes, Cloud Fundamentals), broader Security, and the full Operational Excellence tier (Observability, Reliability Engineering, Performance Engineering). The folder numbering (00-11, with no remaining gaps) now matches this table exactly.

> [!tip] How to use this
> If you're already confident in a tier (e.g., you've built production systems using TCP/HTTP for years), skim it for the specific framing this book uses and move on — every chapter is written so its "why this matters later" callouts are the fast way to check whether you already have the mental model.

---

## Tier 0 — Computer Architecture (the true foundation)

Everything else in this handbook silently assumes you understand *why* some operations are fast and others are slow. This tier makes that explicit.

| # | Chapter | Status |
|---|---|---|
| 0.1 | [[CS Fundamentals/00 - Computer Architecture/CPU, Memory & Cache Hierarchy\|CPU, Memory & Cache Hierarchy]] | ✅ |

## Tier 1 — Operating Systems

How a single machine shares its CPU and memory across many concurrent programs — the layer every database, cache, and server process runs on top of.

| # | Chapter | Status |
|---|---|---|
| 1.1 | [[CS Fundamentals/01 - Operating Systems/Processes, Threads & Context Switching\|Processes, Threads & Context Switching]] | ✅ |
| 1.2 | [[CS Fundamentals/01 - Operating Systems/Memory Management & Virtual Memory\|Memory Management & Virtual Memory]] | ✅ |
| 1.3 | [[CS Fundamentals/01 - Operating Systems/I-O Models - Blocking, Non-Blocking, and Async\|I/O Models: Blocking, Non-Blocking & Async]] | ✅ |

## Tier 2 — Networking

How machines talk to each other — the layer every distributed system, HLD chapter, and API depends on. Ordered low-level to high-level: transport first, then application protocols, then the infrastructure built on top of them.

| # | Chapter | Status |
|---|---|---|
| 2.1 | [[CS Fundamentals/02 - Networking/TCP Deep Dive\|TCP Deep Dive]] | ✅ |
| 2.2 | [[CS Fundamentals/02 - Networking/HTTP Evolution & DNS Resolution\|HTTP Evolution & DNS Resolution]] | ✅ |
| 2.3 | [[CS Fundamentals/02 - Networking/Load Balancing\|Load Balancing]] | ✅ |
| 2.4 | [[CS Fundamentals/02 - Networking/API Gateway\|API Gateway]] | ✅ |
| 2.5 | [[CS Fundamentals/02 - Networking/CDN Internals\|CDN Internals]] | ✅ |
| 2.6 | [[CS Fundamentals/02 - Networking/Reverse Proxy\|Reverse Proxy]] | ✅ |
| 2.7 | [[CS Fundamentals/02 - Networking/REST, gRPC and GraphQL\|REST, gRPC & GraphQL]] | ✅ |

## Tier 3 — Databases & Storage Engines

How data is durably stored, indexed, and queried — the layer every HLD chapter's "which database" decision depends on.

| # | Chapter | Status |
|---|---|---|
| 3.1 | [[CS Fundamentals/03 - Databases/Storage Engines - B-Tree vs LSM-Tree\|Storage Engines: B-Tree vs. LSM-Tree]] | ✅ |
| 3.2 | [[CS Fundamentals/03 - Databases/Indexes & B+ Trees\|Indexes & B+ Trees]] | ✅ |
| 3.3 | [[CS Fundamentals/03 - Databases/ACID & Isolation Levels\|ACID & Isolation Levels]] | ✅ |
| 3.4 | [[CS Fundamentals/03 - Databases/SQL Query Execution Deep Dive\|SQL Query Execution Deep Dive]] | ✅ |
| 3.5 | [[CS Fundamentals/03 - Databases/Cassandra Internals\|Cassandra Internals]] | ✅ |
| 3.6 | [[CS Fundamentals/03 - Databases/MongoDB Internals\|MongoDB Internals]] | ✅ |

## Tier 4 — Caching

How to avoid paying Tier 3's storage cost on every read — direct application of Tier 0's memory-hierarchy reasoning.

| # | Chapter | Status |
|---|---|---|
| 4.1 | [[CS Fundamentals/04 - Caching/Caching Strategies\|Caching Strategies]] (general theory first) | ✅ |
| 4.2 | [[CS Fundamentals/04 - Caching/Redis Internals\|Redis Internals]] | ✅ |
| 4.3 | [[CS Fundamentals/04 - Caching/Memcached Internals\|Memcached Internals]] | ✅ |

## Tier 5 — Messaging & Streaming

How systems communicate asynchronously instead of via direct request/response — required before Distributed Systems, since consensus/replication concepts are easier to grasp once you've seen an append-only log.

| # | Chapter | Status |
|---|---|---|
| 5.1 | [[CS Fundamentals/05 - Messaging & Streaming/Kafka Internals\|Kafka Internals]] | ✅ |
| 5.2 | [[CS Fundamentals/05 - Messaging & Streaming/Kafka Ecosystem and Production Patterns\|Kafka Ecosystem & Production Patterns]] (Streams, Connect, Schema Registry, Avro/Protobuf, Outbox, DLQ, capacity planning) | ✅ |
| 5.3 | [[CS Fundamentals/05 - Messaging & Streaming/Kafka Alternatives - Pulsar, NATS and Redis Streams\|Kafka Alternatives: Pulsar, NATS & Redis Streams]] | ✅ |
| 5.4 | [[CS Fundamentals/05 - Messaging & Streaming/RabbitMQ Internals\|RabbitMQ Internals]] | ✅ |

> [!success] Kafka now gets the standalone-book depth originally planned
> Across [[CS Fundamentals/05 - Messaging & Streaming/Kafka Internals|Kafka Internals]] and [[CS Fundamentals/05 - Messaging & Streaming/Kafka Ecosystem and Production Patterns|Kafka Ecosystem & Production Patterns]]: segments, offset/timestamp indexes, ISR mechanics, exactly-once semantics/transactions, Kafka Streams, Kafka Connect, Schema Registry, Avro/Protobuf, the Outbox Pattern, Kafka-style DLQ, and capacity planning are all covered.

## Tier 6 — Distributed Systems

How multiple machines agree, coordinate, and stay available when parts of the system fail — the theory underneath almost every HLD chapter's hardest problems.

| # | Chapter | Status |
|---|---|---|
| 6.1 | [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC\|CAP Theorem & PACELC]] (the theory framing everything below) | ✅ |
| 6.2 | [[CS Fundamentals/06 - Distributed Systems/Consistent Hashing\|Consistent Hashing]] | ✅ |
| 6.3 | [[CS Fundamentals/06 - Distributed Systems/Sharding & Partitioning\|Sharding & Partitioning]] | ✅ |
| 6.4 | [[CS Fundamentals/06 - Distributed Systems/Consensus (Raft & Paxos)\|Consensus (Raft & Paxos)]] | ✅ |
| 6.5 | [[CS Fundamentals/06 - Distributed Systems/Resilience Patterns\|Resilience Patterns]] | ✅ |
| 6.6 | [[CS Fundamentals/06 - Distributed Systems/Service Discovery\|Service Discovery]] | ✅ |
| 6.7 | [[CS Fundamentals/06 - Distributed Systems/Scalability Patterns\|Scalability Patterns]] (consolidated reference — horizontal/vertical, replicas, sharding, caching, async absorption, geography) | ✅ |
| 6.8 | [[CS Fundamentals/06 - Distributed Systems/Distributed Transactions - Saga Pattern and Two-Phase Commit\|Distributed Transactions: Saga Pattern & Two-Phase Commit]] | ✅ |
| 6.9 | [[CS Fundamentals/06 - Distributed Systems/Idempotency\|Idempotency]] | ✅ |
| 6.10 | [[CS Fundamentals/06 - Distributed Systems/Bloom Filter and Probabilistic Membership\|Bloom Filter & Probabilistic Membership]] | ✅ |
| 6.11 | [[CS Fundamentals/06 - Distributed Systems/Geospatial Indexing\|Geospatial Indexing]] (geohash, quadtree, S2/H3) | ✅ |

## Tier 7 — Architecture & Deployment Patterns

How individual services are packaged, deployed, and composed into a larger system — the layer between "I understand one database" and "I can design a whole platform."

| # | Chapter | Status |
|---|---|---|
| 7.1 | [[CS Fundamentals/07 - Architecture and Deployment Patterns/Monolith vs Microservices\|Monolith vs. Microservices]] | ✅ |
| 7.2 | [[CS Fundamentals/07 - Architecture and Deployment Patterns/Event-Driven Architecture\|Event-Driven Architecture]] | ✅ |
| 7.3 | [[CS Fundamentals/07 - Architecture and Deployment Patterns/Docker Fundamentals\|Docker Fundamentals]] | ✅ |
| 7.4 | [[CS Fundamentals/07 - Architecture and Deployment Patterns/Kubernetes Fundamentals\|Kubernetes Fundamentals]] | ✅ |
| 7.5 | [[CS Fundamentals/07 - Architecture and Deployment Patterns/Cloud Fundamentals\|Cloud Fundamentals]] | ✅ |

## Tier 8 — Security

| # | Chapter | Status |
|---|---|---|
| 8.1 | [[CS Fundamentals/08 - Security/Authentication & Authorization\|Authentication & Authorization]] | ✅ |
| 8.2 | [[CS Fundamentals/08 - Security/Encryption, Zero Trust and Secrets Management\|Encryption, Zero Trust & Secrets Management]] | ✅ |

## Tier 9 — Operational Excellence

How you know a system is healthy, how you keep it healthy, and how you make it fast — the layer that separates "it works in the demo" from "it works at 3am under load."

| # | Chapter | Status |
|---|---|---|
| 9.1 | [[CS Fundamentals/09 - Operational Excellence/Observability - Metrics, Logs and Traces\|Observability: Metrics, Logs & Traces]] | ✅ |
| 9.2 | [[CS Fundamentals/09 - Operational Excellence/Reliability Engineering\|Reliability Engineering]] | ✅ |
| 9.3 | [[CS Fundamentals/09 - Operational Excellence/Performance Engineering\|Performance Engineering]] | ✅ |

## Tier 10 — Software Design Principles

> [!info] A different KIND of foundation
> Everything in Tiers 0-9 is about *systems* — hardware, networks, distributed coordination. This tier is about *code structure* — how to write maintainable object-oriented Go, which is what every LLD chapter in this handbook actually exercises. Listed last deliberately: it's foundational to LLD specifically, not to system-level reasoning generally.

| # | Chapter | Status |
|---|---|---|
| 10.1 | [[CS Fundamentals/10 - Design Principles/01 - SOLID Principles\|SOLID Principles]] | ✅ |
| 10.2 | [[CS Fundamentals/10 - Design Principles/02 - Design Patterns Cheat Sheet\|Design Patterns Cheat Sheet]] (all 23 GoF patterns, recognition-level) | ✅ |
| 10.3 | [[CS Fundamentals/10 - Design Principles/03 - Creational Design Patterns - Full Code Deep Dive\|Creational Design Patterns]] (full before/after Go code) | ✅ |
| 10.4 | [[CS Fundamentals/10 - Design Principles/04 - Structural Design Patterns - Full Code Deep Dive\|Structural Design Patterns]] (full before/after Go code) | ✅ |
| 10.5 | [[CS Fundamentals/10 - Design Principles/05 - Behavioral Design Patterns - Full Code Deep Dive\|Behavioral Design Patterns]] (full before/after Go code) | ✅ |
| 10.6 | [[CS Fundamentals/10 - Design Principles/06 - Concurrency Patterns in Go\|Concurrency Patterns in Go]] (not GoF — Worker Pool, Pipeline, Fan-out/Fan-in, Context Cancellation, Pub-Sub) | ✅ |

## Tier 11 — Search Engines

How full-text search actually works — the specialized secondary store behind "search this," autocomplete, and log analytics. Builds directly on Tier 3 (indexes, LSM-style immutable segments) and Tier 6 (sharding, consensus); sits after them because a search engine is best understood as a database projection, not a primary store.

| # | Chapter | Status |
|---|---|---|
| 11.1 | [[CS Fundamentals/11 - Search Engines/Full-Text Search - Elasticsearch, OpenSearch and Solr\|Full-Text Search: Elasticsearch, OpenSearch & Solr]] (inverted index, Lucene, BM25, shards/replicas/segments, NRT, the OpenSearch fork, Solr tradeoffs) | ✅ |

---

## 🗺️ How to actually use this for interview prep

1. **Read Tiers 0-2 once, fully**, even if you think you know networking — this handbook's specific framing (the latency-ratio table, the TCP/HTTP layering) gets referenced by name throughout later HLD chapters.
2. **Read Tiers 3-6 as you encounter HLD chapters that need them** — every HLD chapter links to the specific CS Fundamentals chapter it depends on, exactly when it's needed, rather than requiring you to have pre-read everything.
3. **Tiers 7-9** are the newest additions to this curriculum — genuinely important for senior-level system design conversations (deployment, observability, reliability) but less likely to be the *center* of a single interview question the way Tier 3-6 topics are.
4. **Tier 10** — read alongside your first LLD chapter, not before it; the principles land better with a concrete bad-draft-to-refactor example in front of you.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]]*
