---
title: "Split-Brain"
aliases: [Split-Brain, Split Brain]
tags: [system-design, glossary]
---

# Split-Brain

> [!info] Definition
> A failure mode where a cluster gets partitioned into two (or more) groups, and **each group independently believes it's the legitimate leader/primary** — both accept writes, and the data diverges with no single source of truth.

This is exactly the failure [[Raft (Consensus)|consensus algorithms]] like Raft are designed to prevent — by requiring a **majority (quorum)** of nodes to agree before electing a leader or committing a write, a minority partition can never independently believe it's in charge, because it can't reach a majority. This is also why quorum systems favor an *odd* number of nodes (a 3-node cluster tolerates 1 failure with a clean majority; a 4-node cluster doesn't meaningfully improve on that).

**Used in:** [[HLD/05 - Consistency, Availability & Consensus/Index|HLD 05 - Consistency, Availability & Consensus]]
