---
title: How This Handbook Works
aliases: [System Design Handbook, Book Map]
tags: [system-design, index, meta]
status: living-document
---

# 📖 How This Handbook Works

> [!abstract] What this is
> Not notes. Not a cheat sheet. A **handbook** — the kind of depth you'd get from an O'Reilly/Manning book, aimed at a 4+ YOE engineer prepping for senior SDE loops at Google/Uber/Rippling/Rubrik/Atlassian/Microsoft/Amazon/Walmart/Adobe-tier companies. Every chapter is written so that after reading it, you can explain the concept unprompted, draw it from memory, survive cross-questioning, and write real code for it — not just recognize the term.

> [!tip] Night-before-the-interview drilling
> [[00 - Start Here/100 System Design Interview Questions|100 System Design Interview Questions]] — the cross-cutting questions that recur in almost every interview regardless of which system is being designed (scaling, failure handling, race conditions, consistency, estimation), separate from each chapter's own case-study-specific Q&A.

---

## The three pillars

```
System Design/
├── CS Fundamentals/     ← the reusable building blocks. Each is its own deep chapter.
│                           Kafka internals, B+ Trees, CAP theorem, OS scheduling —
│                           taught ONCE, in full depth, then LINKED TO everywhere else.
├── HLD/                 ← interview questions as chapters ("Design Uber", "Design Kafka").
│                           Each chapter builds the system from scratch, introducing
│                           CS Fundamentals chapters exactly when the design needs them —
│                           never re-explaining a building block twice.
└── LLD/                 ← same idea for low-level design. Patterns are never listed —
                            they're taught by writing bad code first, feeling the pain,
                            then refactoring into the pattern. Every chapter ships
                            complete, compilable Go code.
```

Plus `Glossary/` — one atomic note per acronym/short-form term, linked from every chapter on first mention, so no chapter has to stop and define "TTL" inline.

## Why this structure (and not topic-by-topic notes)

The old version of this vault organized HLD as "Databases," "Caching," "Consistency" — topic-first, like a reference manual. That's how you'd organize a vault if the goal were quick lookup. **It is not how interviews work.** An interviewer asks "design Uber," not "explain replication." So this handbook is organized the way the interview is actually structured: a **problem**, and everything you pull in to solve it. The topic-deep-dives still exist — they live in `CS Fundamentals/`, written once, in full, and every HLD/LLD chapter that needs them links out instead of repeating a shallower version inline.

This also means: **read `CS Fundamentals/` chapters as they get referenced**, not necessarily front-to-back before touching a single HLD chapter. A chapter on "Design TinyURL" will tell you exactly which fundamentals chapter to read at the exact moment the design needs it — same as a good textbook footnoting a prerequisite instead of assuming you've read the whole book linearly.

---

## The standard every chapter is held to

> [!warning] Non-negotiable, every single chapter
> 1. **Never stop at a definition.** Every concept answers: why was it created, what problem does it solve, how does it work internally, why does it work that way, what are the alternatives, what are the trade-offs, when do you use it, when do you *not* use it, what mistakes do engineers make, what breaks in production, how do you debug it.
> 2. **HLD chapters follow 8 steps** (below) — interview question → requirements → estimation → incremental build-up → deep dive on every component used → full architecture with Mermaid diagrams → interviewer follow-ups, answered → production experience (monitoring, failure scenarios, debugging, cost).
> 3. **LLD chapters teach patterns by refactor, not by name.** Bad code first. Feel why it's bad. Refactor into the pattern. Every chapter ships **complete, compilable, production-quality Go code** — real interfaces, real error handling, real structure — never a snippet.
> 4. **Every acronym links to `Glossary/`** on first mention. Every diagram-shaped concept gets a Mermaid diagram. Every chapter ends with interview follow-up Q&A, each answer followed by at least one deeper follow-up.
> 5. **CS Fundamentals chapters go to internals.** Not "Kafka is a distributed log" — broker, topic, partition, segment, consumer group, ISR, controller, leader election, offset management, delivery guarantees, rebalancing, log compaction, on-disk storage format, why it's fast, when it's the wrong choice.

---

## The HLD chapter template (all 8 steps, every time)

1. **The interview question** — stated exactly as an interviewer would ask it.
2. **Requirements** — functional, non-functional, constraints, assumptions, edge cases, *and why each one matters*.
3. **Back-of-envelope estimation** — users, traffic, storage, read/write ratio, QPS, bandwidth, memory, scaling numbers, full math shown.
4. **Incremental build-up** — start from nothing, add one capability at a time, and the moment a new component is needed (storage, cache, queue, search), stop and either link an existing `CS Fundamentals` chapter or note that one needs writing.
5. **Deep dive on every component actually used** — not "we use Kafka," but everything the CS Fundamentals chapter on Kafka covers, applied to *this specific design's* numbers and constraints.
6. **Full architecture** — Mermaid diagrams: request flow, data flow, failure flow, retry flow, scaling strategy, fault tolerance.
7. **Interviewer follow-ups, answered** — "why not RabbitMQ," "what if this component dies," "how do you 100x this," each with a real answer.
8. **Production experience** — monitoring, metrics, alerting, logging, real failure scenarios, debugging approach, cost tradeoffs.

## The LLD chapter template

Interview question → requirement clarification → **bad first draft** (the naive solution, written out in full) → **why it breaks** (concretely — add one new requirement and watch it crack) → **refactor into the pattern(s)** step by step → UML class diagram + sequence diagram (Mermaid) → SOLID principles applied, named → alternatives considered and why rejected → **complete compilable Go code** (full file layout, interfaces, enums, error handling, unit-test-friendly) → interviewer follow-ups, answered.

---

## 🗺️ Roadmap — what's planned, chapter by chapter

Chapters are written **one at a time, on request** — say *"write [chapter name]"* and it's produced to the full standard above. The three below are already written as the reference standard for everything that follows.

### ✅ Written (reference-quality, read these to calibrate the bar) — 76 chapters so far

> [!success] Second gap-audit pass: ACID, SOLID, Design Patterns
> Added [[CS Fundamentals/Databases/ACID & Isolation Levels|ACID & Isolation Levels]] (previously only a broken glossary stub), [[CS Fundamentals/Design Principles/SOLID Principles|SOLID Principles]], and [[CS Fundamentals/Design Principles/Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] (all 23 GoF patterns — 8 with full LLD chapter treatment, 15 as cheat-sheet-level entries) — consolidating principles that were previously taught correctly but only scattered inline across individual LLD chapters.

> [!success] Core checklist complete, plus a real gap-audit pass
> All 20 HLD + all 20 LLD case studies are ✅ (25 HLD folders + 22 LLD folders on disk — bonus chapters push some folder numbers past 20; the checklist tables below are the authoritative status source, not folder numbers). Plus 21 CS Fundamentals chapters (8 added in a gap-audit pass: Authentication & Authorization, Consensus/Raft & Paxos, Sharding & Partitioning, Resilience Patterns, API Gateway, Load Balancing, HTTP Evolution & DNS Resolution, CDN Internals) and 7 bonus-only case studies (Google Calendar HLD+LLD, Google Meet HLD+LLD, Food Delivery HLD, Analytics Aggregation/HyperLogLog HLD, Search Engine HLD — Food Delivery's LLD bonus content was folded into the existing core LLD #13 rather than a new folder). 68 chapters total.

> [!tip] Full status lives in the checklist tables below
> This table shows the earliest chapters as calibration references. The HLD/LLD/CS-Fundamentals checklist tables further down are the authoritative, currently-maintained status tracker — check ✅/🔜 there for anything not listed here.

| Pillar | Chapter | Teaches |
|---|---|---|
| CS Fundamentals — Messaging | [[CS Fundamentals/Messaging & Streaming/Kafka Internals\|Kafka Internals]] | Broker/topic/partition/segment/ISR/controller/leader-election/offsets/delivery-guarantees/rebalancing/log-compaction/why-fast |
| CS Fundamentals — Messaging | [[CS Fundamentals/Messaging & Streaming/RabbitMQ Internals\|RabbitMQ Internals]] | Exchange-based routing (direct/fanout/topic/headers), push vs pull, ack/nack, DLX, honest RabbitMQ-vs-Kafka framework |
| CS Fundamentals — Operating Systems | [[CS Fundamentals/Operating Systems/Processes, Threads & Context Switching\|Processes, Threads & Context Switching]] | Process vs thread internals, why a thread switch is cheaper (TLB/cache locality), ties directly to why goroutines are cheaper still |
| CS Fundamentals — Networking | [[CS Fundamentals/Networking/TCP Deep Dive\|TCP Deep Dive]] | 3-way handshake, flow vs congestion control, TIME_WAIT + port exhaustion, Nagle's algorithm production gotcha |
| CS Fundamentals — Databases | [[CS Fundamentals/Databases/Indexes & B+ Trees\|Indexes & B+ Trees]] | Why B+ over B-Tree/binary tree, clustered vs non-clustered, leftmost-prefix rule |
| CS Fundamentals — Databases | [[CS Fundamentals/Databases/SQL Query Execution Deep Dive\|SQL Query Execution Deep Dive]] | Parse→plan→optimize→execute, why indexes are actually fast, `EXPLAIN ANALYZE`, MVCC, InnoDB vs Postgres |
| CS Fundamentals — Databases | [[CS Fundamentals/Databases/Cassandra Internals\|Cassandra Internals]] | LSM-Tree write path, wide-column model, tunable consistency, the tombstone/"Cassandra as a queue" gotcha |
| CS Fundamentals — Databases | [[CS Fundamentals/Databases/MongoDB Internals\|MongoDB Internals]] | Document model & denormalization-by-design, WiredTiger vs MMAPv1, the hot-shard-key mistake |
| CS Fundamentals — Caching | [[CS Fundamentals/Caching/Caching Strategies\|Caching Strategies]] | Cache-aside/write-through/write-back/refresh-ahead, LRU/LFU internals, stampede vs penetration (with Go's `singleflight`) |
| CS Fundamentals — Caching | [[CS Fundamentals/Caching/Redis Internals\|Redis Internals]] | Single-threaded execution model, data structures (esp. sorted sets), RDB vs AOF, hash-slot sharding |
| CS Fundamentals — Caching | [[CS Fundamentals/Caching/Memcached Internals\|Memcached Internals]] | Slab allocator, multi-threaded model vs Redis, slab calcification |
| CS Fundamentals — Distributed Systems | [[CS Fundamentals/Distributed Systems/CAP Theorem & PACELC\|CAP Theorem & PACELC]] | Precise CAP definitions (linearizability, not the folk version), PACELC, real-system classification |
| HLD 1 | [[HLD/01 - Design TinyURL (URL Shortener)/Design TinyURL\|Design TinyURL]] | Base62 encoding, SQL vs NoSQL choice, caching, the full 8-step template, applied end to end |
| HLD 2 | [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter\|Design a Rate Limiter]] | 4-algorithm comparison, atomic Redis Lua script, fail-open vs fail-closed |
| HLD 3 | [[HLD/03 - Design a Distributed Cache (build Redis)/Design a Distributed Cache\|Design a Distributed Cache]] | Consistent hashing in practice, client-side vs proxy routing, sync/async replication as live PACELC, hot-key problem |
| HLD 4 | [[HLD/04 - Design a Notification Service/Design a Notification Service\|Design a Notification Service]] | Fan-out at scale, outbound rate limiting, priority-queue isolation (HOL blocking), idempotent delivery |
| HLD 5 | [[HLD/05 - Design a Distributed Message Queue (build Kafka)/Design a Distributed Message Queue\|Design a Distributed Message Queue]] | Metadata/discovery problem, controller election, partition-count capacity planning |
| HLD 6 | [[HLD/06 - Design Twitter - News Feed/Design Twitter - News Feed\|Design Twitter / News Feed]] | Fan-out-on-write, the celebrity problem, hybrid push/pull |
| HLD 7 | [[HLD/07 - Design WhatsApp - Chat System/Design WhatsApp - Chat System\|Design WhatsApp / Chat System]] | Connection-routing layer, persistent-connection scaling, reconnect storms |
| LLD 1 | [[LLD/01 - Design a Parking Lot/Design a Parking Lot\|Design a Parking Lot]] | Strategy + Factory taught via bad-code-first refactor, complete compilable Go |
| LLD 2 | [[LLD/02 - Design a Vending Machine/Design a Vending Machine\|Design a Vending Machine]] | State pattern, and when NOT to force a noun into a State |
| LLD 3 | [[LLD/03 - Design an Elevator System/Design an Elevator System\|Design an Elevator System]] | State (movement) + Strategy (dispatch) combined; direction as data, not a state |
| LLD 4 | [[LLD/04 - Design a Rate Limiter/Design a Rate Limiter\|Design a Rate Limiter (LLD)]] | Strategy pattern, both token bucket AND sliding-window-counter fully implemented |
| LLD 5 | [[LLD/05 - Design Splitwise/Design Splitwise\|Design Splitwise]] | Strategy pattern + real heap-based greedy debt-simplification algorithm, integer-cents money correctness |
| LLD 6 | [[LLD/06 - Design BookMyShow - Seat Booking/Design BookMyShow - Seat Booking\|Design BookMyShow / Seat Booking]] | The canonical check-then-act race condition, fixed; seat-hold-with-expiry; when NOT to use the State pattern |
| LLD 7 | [[LLD/07 - Design a Logger Framework/Design a Logger Framework\|Design a Logger Framework]] | Chain of Responsibility — including the deliberate deviation from the textbook "first handler wins" version |

### 🎯 The full HLD checklist — 20 interview questions

| # | Question | Status |
|---|---|---|
| 1 | Design TinyURL (URL Shortener) | ✅ |
| 2 | Design a Rate Limiter | ✅ |
| 3 | Design a Distributed Cache (build Redis) | ✅ |
| 4 | Design a Notification Service | ✅ |
| 5 | Design a Distributed Message Queue (build Kafka) | ✅ |
| 6 | Design Twitter / News Feed | ✅ |
| 7 | Design WhatsApp / Chat System | ✅ |
| 8 | Design Google Drive / Dropbox | ✅ |
| 9 | Design YouTube / Netflix | ✅ |
| 10 | Design Uber | ✅ |
| 11 | Design Search Autocomplete / Typeahead | ✅ |
| 12 | Design a Web Crawler | ✅ |
| 13 | Design Distributed File Storage (S3/GFS-style) | ✅ |
| 14 | [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter\|Design a Multi-Region Rate Limiter]] | ✅ |
| 15 | [[HLD/15 - Design a Ticket Booking System/Design a Ticket Booking System\|Design a Ticket Booking System (BookMyShow, HLD-level)]] | ✅ |
| 16 | [[HLD/23 - Design an E-commerce System/Design an E-commerce System\|Design an E-commerce System (Amazon-style)]] | ✅ |
| 17 | Design a Payment System (Stripe-style, idempotency-heavy) | ✅ |
| 18 | Design a Distributed Lock Service (build ZooKeeper/etcd-lite) | ✅ |
| 19 | Design a Real-Time Leaderboard | ✅ |
| 20 | [[HLD/20 - Design a Log Aggregation and Monitoring System/Design a Log Aggregation and Monitoring System\|Design a Log Aggregation / Monitoring System]] | ✅ |

### 🎁 Bonus chapters (beyond the core 20+20)

| Pillar | Chapter | Notes |
|---|---|---|
| HLD | [[HLD/16 - Design a Food Delivery System/Design a Food Delivery System\|Design a Food Delivery System]] | ✅ Geospatial reuse from Uber, two-leg matching, continuous surge pricing |
| LLD | [[LLD/13 - Design a Food Delivery System/Design a Food Delivery System\|Design a Food Delivery System]] | ✅ Expanded with PricingStrategy (promo/discount) + Observer wired into State transitions |
| HLD | [[HLD/21 - Design Google Calendar/Design Google Calendar\|Design Google Calendar]] | ✅ RRULE-style recurrence, exceptions, DST-safe storage |
| LLD | [[LLD/21 - Design Google Calendar/Design Google Calendar\|Design Google Calendar]] | ✅ RecurrenceRule Strategy, the time.Time-as-map-key gotcha |
| HLD | [[HLD/22 - Design Google Meet/Design Google Meet\|Design Google Meet]] | ✅ SFU vs MCU, simulcast, full-mesh O(N²) proof |
| LLD | [[LLD/22 - Design Google Meet/Design Google Meet\|Design Google Meet]] | ✅ Room-scoped Observer, host-only permission checks |
| LLD | [[LLD/23 - Design Snake and Ladder/Design Snake and Ladder\|Design Snake and Ladder]] | ✅ Strategy (swappable dice), snake/ladder unified as one "jump" concept, intentionally no locking |
| HLD | [[HLD/24 - Design an Analytics Aggregation System/Design an Analytics Aggregation System\|Design an Analytics Aggregation System]] | ✅ HyperLogLog, Count-Min Sketch, mergeable sketches, windowed counting |
| HLD | [[HLD/25 - Design a Search Engine/Design a Search Engine\|Design a Search Engine (Google Search)]] | ✅ Inverted index, TF-IDF/BM25, posting-list compression, sharded search |
| LLD | [[LLD/24 - Design Twitter/Design Twitter\|Design Twitter]] | ✅ Heap-based K-way timeline merge (Merge K Sorted Lists), idempotent like/retweet |
| LLD | [[LLD/25 - Design Instagram/Design Instagram\|Design Instagram]] | ✅ Interface Segregation (Likeable/Commentable), lazy story expiry reused from BookMyShow |
| LLD | [[LLD/26 - Design YouTube/Design YouTube\|Design YouTube]] | ✅ Lost-update race fixed with sync/atomic (not a mutex), per-user view dedup, Observer reused for subscriptions |
| LLD | [[LLD/27 - Design WhatsApp/Design WhatsApp\|Design WhatsApp]] | ✅ Per-recipient read receipts (group-chat fix), guarded-enum-not-State judgment call reused a third time |

### 🎯 The full LLD checklist — 20 interview questions (all with complete compilable Go)

| # | Question | Pattern(s) taught | Status |
|---|---|---|---|
| 1 | Design a Parking Lot | Strategy + Factory | ✅ |
| 2 | Design a Vending Machine | State | ✅ |
| 3 | Design an Elevator System | State + Strategy | ✅ |
| 4 | Design a Rate Limiter | Strategy | ✅ |
| 5 | Design Splitwise | Graph algorithms + clean modeling | ✅ |
| 6 | Design BookMyShow / Seat Booking | Concurrency, thread-safe design | ✅ |
| 7 | Design a Logger Framework | Chain of Responsibility | ✅ |
| 8 | Design a Notification System | Observer + Chain of Responsibility | ✅ |
| 9 | Design Chess | Composite/polymorphism (replaces switch), shared slide-move logic | ✅ |
| 10 | Design Tic-Tac-Toe (with AI) | Strategy (swappable difficulty), real minimax | ✅ |
| 11 | Design an ATM | State + Command | ✅ |
| 12 | Design a Library Management System | Factory + Observer | ✅ |
| 13 | Design a Food Delivery System | Strategy + State | ✅ |
| 14 | Design an LRU Cache (as a system, pluggable eviction) | Strategy | ✅ |
| 15 | [[LLD/15 - Design a Text Editor/Design a Text Editor\|Design a Text Editor (Undo/Redo)]] | Command (Memento considered, rejected — see chapter) | ✅ |
| 16 | [[LLD/16 - Design a File System/Design a File System\|Design a File System]] | Composite | ✅ |
| 17 | Design a Task Scheduler / Cron | Strategy + Observer | ✅ |
| 18 | [[LLD/18 - Design a Chat Application/Design a Chat Application\|Design a Chat Application (LLD-level)]] | Mediator (contrasted with Observer) | ✅ |
| 19 | [[LLD/19 - Design a Stock Exchange/Design a Stock Exchange\|Design a Stock Exchange / Order Matching Engine]] | Strategy, concurrency-heavy | ✅ |
| 20 | Design a Distributed ID Generator (Snowflake-style) | Real Snowflake bit-layout implementation | ✅ |

### 🎯 CS Fundamentals — full technology checklist

**Operating Systems:** Processes & Threads ✅ · CPU Scheduling · Virtual Memory & Paging · Segmentation · Synchronization Primitives (Mutex/Semaphore) · Deadlocks · Interrupts & System Calls · NUMA · Cache Coherency.
**Networking:** TCP Deep Dive ✅ · HTTP/1.1 → HTTP/2 → HTTP/3 & QUIC ✅ · DNS Resolution ✅ · Load Balancing (L4/L7) ✅ · API Gateway ✅ · CDN Internals ✅ · UDP · TLS Handshake · NAT · Packet Routing · **GraphQL** (query language internals, N+1 problem, resolvers).
**Security:** Authentication & Authorization (sessions, JWT, OAuth2, refresh-token rotation) ✅ · Encryption at Rest/Transit · Zero Trust · Secrets Management.
**Databases:** Indexes & B+ Trees ✅ · SQL Query Execution Deep Dive ✅ · Cassandra Internals ✅ · MongoDB Internals ✅ · Partitioning/Sharding ✅ (see Distributed Systems) · ACID & Isolation Levels (dirty/non-repeatable/phantom reads) ✅ · Write-Ahead Log (WAL) · Replication · SQL vs NoSQL.
**Caching:** Caching Strategies ✅ · Redis Internals ✅ · Memcached Internals ✅.
**Messaging & Streaming:** Kafka Internals ✅ · RabbitMQ Internals ✅ · Pulsar · NATS · Redis Streams.
**Distributed Systems:** CAP & PACELC ✅ · Consistent Hashing ✅ · Raft & Paxos / Consensus ✅ · Sharding & Partitioning ✅ · Resilience Patterns (Circuit Breaker, Retry, Bulkhead) ✅ · Gossip Protocol · Leader Election · Vector Clocks & Lamport Clocks · Distributed Locks · Idempotency · ZooKeeper.
**Design Principles:** SOLID Principles (all 5, with the code proving each) ✅ · Design Patterns Cheat Sheet (all 23 GoF patterns) ✅.

---

## 🔁 Workflow

> [!tip] How to keep growing this
> This file **is** the table of contents — update the roadmap table as each chapter gets written (move it from 🔜 to ✅). Request the next chapter by name; it gets written to the full standard, in the right folder, cross-linked to whatever `CS Fundamentals` chapters it depends on (writing those first if they don't exist yet).

---
*Related: [[CS Fundamentals/Messaging & Streaming/Kafka Internals|Kafka Internals]] · [[HLD/01 - Design TinyURL (URL Shortener)/Design TinyURL|Design TinyURL]] · [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]]*
