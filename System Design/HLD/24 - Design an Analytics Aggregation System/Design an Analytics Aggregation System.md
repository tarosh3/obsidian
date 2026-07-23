---
title: "Design an Analytics Aggregation System (Ad-Click / Unique-Visitor Counting)"
aliases: [Ad Click Aggregation, Unique Visitor Counting, HyperLogLog System Design]
tags: [system-design, hld, case-study, probabilistic-data-structures, kafka]
status: reference-quality
---

# Design an Analytics Aggregation System (Ad-Click / Unique-Visitor Counting)

> [!abstract] How to read this chapter
> Built phase by phase around one deliberate tradeoff — approximate counting with a *bounded, known error* — and the structure that delivers it, HyperLogLog, including exactly why it can't "forget old data." Each phase adds one idea, exposes the next bottleneck, and fixes it.

> [!question] The interview question
> "Design a system that counts ad clicks/impressions and unique visitors, in near real-time, across billions of events per day — approximate results with a bounded, known error rate are acceptable."

---

## Requirements

**Functional**
- Ingest click/impression events.
- Count **total clicks** per campaign.
- Count **unique visitors** (distinct count) per campaign over a rolling window.
- Near-real-time dashboards.

**Non-functional**

| Requirement | Why it matters here specifically |
|---|---|
| **Billions of events/day** | Ingestion is a firehose; the counting structure must be cheap per event. |
| **Approximate is a deliberate choice** | Exact distinct-counting needs unbounded memory — bounded, known error is the entire premise, not a compromise nobody chose. |

---

## Phase 00 — Capacity math you can defend

| Quantity | Derivation | Result |
|---|---|---|
| Events | 10B/day | ~115,000/s average, higher at launches |
| **Exact distinct cost** | 100M uniques × 8-byte IDs, one campaign | ~800 MB — for **one** campaign |

> [!example] In plain words
> ~800 MB for one campaign's exact unique set, times thousands of active campaigns, is untenable in memory — and querying it live is worse. That number is *why* the whole design turns to approximate counting.

---

## Phase 01 — The naive version: exact distinct counting

*Start with `COUNT(DISTINCT)` / a HashSet so its memory names the fix.*

`SELECT COUNT(DISTINCT visitor_id)` per campaign, or a `HashSet` of every seen ID. Breaks exactly as the capacity math shows — unbounded memory growth per campaign, and live queries over it are worse.

| 🔴 Bottleneck | 🟢 Next fix |
|---|---|
| Exact distinct sets grow unboundedly with true cardinality — untenable across thousands of campaigns. | Approximate distinct counting with HyperLogLog (Phase 2). |

---

## Phase 02 — HyperLogLog for approximate distinct counting

*Fixed tiny memory, ~2% error, regardless of whether the count is a thousand or a billion.*

> [!tip] The core idea, precisely
> Hash every element to a uniform-random bit string. The **position of the leftmost 1-bit** is a probabilistic signal: seeing a hash with its leftmost 1-bit at position `k` is evidence that roughly `2^k` distinct elements have been hashed (rare patterns imply more elements were needed to produce them by chance). HLL splits elements across many small **registers** (chosen by a few bits of the hash), each tracking the *maximum* leftmost-1-bit-position seen, then combines them via a harmonic mean — averaging across registers suppresses the high variance any single register would have alone.

The remarkable property: **~2% error using a fixed, tiny footprint (a few KB)** — regardless of true cardinality. Memory doesn't grow with count at all.

**Mergeable across shards.** HLL registers from different servers combine via a simple **per-register max** to get a correct estimate for the *union* — without the raw data. Each consuming shard keeps its own local sketch; a global count is a cheap merge, not a recomputation.

> [!info] Count-Min Sketch — a different structure for a different question
> HLL answers "how many *distinct* things." **Count-Min Sketch** answers "how many *times* did this specific thing happen" (approximate frequency) — the right structure for "top-K campaigns by click volume." Don't conflate the two.

| 🔴 Bottleneck | 🟢 Next fix |
|---|---|
| A single ever-growing HLL can only answer "unique visitors *ever*" — it structurally can't do "last 24 hours." | Time-bucketed sketches + a precise error bound (Phase 3). |

---

## Phase 03 — Deep dive: windowing and error bounds

> [!bug] HyperLogLog cannot forget data — a real, named limitation
> HLL only supports **merge/union** — it can't subtract or expire old data, since a register only ever tracks a maximum and there's no way to know if that maximum came from data that should now be excluded. One ever-growing sketch can never answer "unique visitors in just the last 24 hours" — only "unique visitors ever."
>
> **The fix: time-bucketed sketches.** Maintain a separate HLL per fixed time bucket (e.g. one per hour). "Unique visitors in the last 24 hours" merges the last 24 hourly sketches — merging is cheap and correct — while old buckets outside the window are simply **dropped, not subtracted.** The standard real answer to windowed unique counting with HLL.

**Error bound, precisely.** The standard error is approximately **`1.04/√m`**, where `m` is the number of registers. More registers → better accuracy → (still small) more memory. A real, quantifiable, tunable tradeoff — precision over hand-waving "it's approximately right."

| 🔴 Bottleneck | 🟢 Next fix |
|---|---|
| Individual pieces handled — assemble the pipeline. | Final architecture (Phase 4). |

---

## Phase 04 — The final combined architecture

```mermaid
graph TD
    Events["Click/impression events"] --> Kafka[("Kafka, sharded by campaign")]
    Kafka --> W1["Worker: hourly HLL sketch<br/>(shard 1)"]
    Kafka --> W2["Worker: hourly HLL sketch<br/>(shard 2)"]
    W1 --> Store[("Sketch store<br/>(one HLL per campaign per hour)")]
    W2 --> Store
    Store --> Merge["Query-time merge<br/>(per-register max across hour buckets)"]
    Merge --> Dashboard["Dashboard"]
```

**Five principles to close with:**
1. Approximate counting with a bounded, known error is a deliberate choice — exact distinct counting is untenable in memory.
2. HLL gives ~2% error in a few KB regardless of cardinality — memory doesn't grow with the count.
3. HLL merges across shards by per-register max — perfect for a sharded ingestion pipeline.
4. HLL can't forget — use time-bucketed sketches and merge only the window; drop old buckets, never subtract.
5. Error is `1.04/√m` (tunable by register count); use exact counting for anything billing-critical, HLL for dashboards.

---

## Interviewer follow-ups, answered

> [!quote]- "Why not exact counting?"
> Memory and cost — an exact hash set per campaign grows unboundedly with true cardinality.

> [!quote]- "How accurate is this, precisely?"
> `~1.04/√m` standard error — a real, quotable formula, not "pretty close."

> [!quote]- "Windowed (e.g. daily) unique count if HLL can't delete data?"
> Time-bucketed sketches, merged only over the requested window; old buckets are dropped, not subtracted.

> [!quote]- "Counts that are billing-critical — still use this?"
> No — a deliberate exception. Anything where the count directly determines money (actual ad spend charged to an advertiser) needs **exact** counting despite the cost, because a 2% error translates directly into over/under-charging. Approximate structures are for dashboards and trend visibility, not billing — knowing where the approximation stops being acceptable is the real judgment, not applying HLL everywhere.

---

## Production experience

> [!info] What to monitor
> HLL estimate drift — periodically spot-check the approximate count against an **exact** count on a small sampled subset of campaigns, to catch implementation bugs (bad hash function, register-count misconfiguration) that would silently skew every estimate. Ingestion lag (Kafka consumer lag, same practice as [[HLD/20 - Design a Log Aggregation and Monitoring System/Design a Log Aggregation and Monitoring System|the Log Aggregation chapter]]). Sketch-merge query latency as the number of time buckets requested grows.

---

## Cheat sheet — if you remember nothing else

1. Approximate counting with a bounded, known error is the premise — exact distinct sets need unbounded memory (~800 MB for one campaign).
2. HyperLogLog: ~2% error in a few KB, independent of cardinality — leftmost-1-bit position across many registers, harmonic mean.
3. HLL merges across shards by per-register max — a global count is a cheap sketch merge, not a raw recompute.
4. HLL can't forget — time-bucketed sketches, merge the window, drop old buckets (never subtract).
5. Error is `1.04/√m`; use exact counting for billing, HLL for dashboards — knowing where to stop is the judgment.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[CS Fundamentals/05 - Messaging & Streaming/Kafka Internals|Kafka Internals]] · [[Glossary/Bloom Filter|Bloom Filter]] (a related but distinct probabilistic structure — membership testing, not counting)*
