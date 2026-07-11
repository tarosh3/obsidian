---
title: "Quorum (R + W > N)"
aliases: [Quorum, R+W>N]
tags: [system-design, glossary]
---

# Quorum (R + W > N)

> [!info] Quick definition
> In a system with `N` replicas, requiring `R` nodes to agree on a **read** and `W` nodes to agree on a **write**, chosen such that `R + W > N`, guarantees every read overlaps with at least one node that has the most recent write — tunable consistency without needing a single leader.

E.g. `N=3, W=2, R=2`: `2+2=4 > 3`, so any 2-node read set and any 2-node write set must share at least one common node. Tuning `R`/`W` lets you trade off read latency vs write latency vs consistency strength — this is exactly how Dynamo-style leaderless databases (Cassandra, DynamoDB) offer configurable consistency per query.

> [!tip] This is a glossary stub
> Full explanation lives in: [[HLD/05 - Consistency, Availability & Consensus/Index|HLD 05 - Consistency, Availability & Consensus]] (once written).

**Used in:** [[HLD/05 - Consistency, Availability & Consensus/Index|HLD 05 - Consistency, Availability & Consensus]]
