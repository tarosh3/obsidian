---
title: "BASE"
aliases: [BASE]
tags: [system-design, glossary]
---

# BASE

> [!info] Quick definition
> The distributed-systems counterpart to [[ACID]]: **B**asically **A**vailable (the system stays up even if part of it fails), **S**oft state (data may change over time even without new input, as replicas converge), **E**ventually consistent (reads may return stale data temporarily, but will converge given enough time).

The tradeoff most NoSQL databases (Cassandra, DynamoDB) make deliberately — favoring availability and horizontal scale over the immediate consistency ACID guarantees, on the bet that most use cases can tolerate brief staleness in exchange for never going down.

> [!tip] This is a glossary stub
> Full explanation with examples lives in: [[HLD/05 - Consistency, Availability & Consensus/Index|HLD 05 - Consistency, Availability & Consensus]] (once written).

**Used in:** [[HLD/03 - Databases & Storage/Index|HLD 03 - Databases & Storage]], [[HLD/05 - Consistency, Availability & Consensus/Index|HLD 05 - Consistency, Availability & Consensus]]
