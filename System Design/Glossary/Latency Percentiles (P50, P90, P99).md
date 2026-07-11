---
title: "Latency Percentiles (P50, P90, P99)"
aliases: [P50, P90, P99, P999, Tail Latency, Percentile Latency]
tags: [system-design, glossary]
---

# Latency Percentiles — P50, P90, P99

> [!info] Definition
> The latency value below which a given percentage of requests fall. **P50** (median) = half of requests are faster than this. **P99** = 99% of requests are faster than this — only the slowest 1% exceed it.

**Why not just use the average?** A handful of very slow outliers barely move the average but can dominate real user experience — if 1% of requests take 5 seconds, a P99-blind average might still look fine while a meaningful chunk of real users are having a bad time. This is why SRE/production dashboards report **P50/P90/P99 (and sometimes P99.9)** instead of a single average — it's also why [[SLA vs SLO vs SLI|SLOs]] are almost always written in percentile terms ("P99 latency < 300ms"), not average terms.

> [!example] Layman's terms
> If 100 people ran a race, P50 is the 50th finisher's time (the median). P99 is the 1st-from-last finisher's time — it tells you how bad the *worst normal* experience is, which matters more for user trust than the average runner's time.

**Used in:** [[HLD/01 - Foundations & Estimation/Index|HLD 01 - Foundations & Estimation]], [[HLD/08 - Security & Observability/Index|HLD 08 - Security & Observability]]
