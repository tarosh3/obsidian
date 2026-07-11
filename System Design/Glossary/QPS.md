---
title: "QPS (Queries Per Second)"
aliases: [QPS, RPS, Requests Per Second]
tags: [system-design, glossary]
---

# QPS — Queries Per Second

> [!info] Definition
> The number of requests a system handles per second. The base unit almost every back-of-envelope capacity estimate is built from.

Also written **RPS** (Requests Per Second) — same thing, interchangeable in practice. Everything downstream (server count, DB load, bandwidth) gets derived from QPS: `total_daily_requests / 86400 ≈ average QPS`, then multiplied by a peak factor (often 2–3×) for **peak QPS**, which is the number you actually design for — average QPS undersizes a system for its busiest moments.

**Used in:** [[HLD/01 - Foundations & Estimation/Index|HLD 01 - Foundations & Estimation]]
