---
title: "Two-Phase Commit (2PC)"
aliases: [2PC, Two-Phase Commit]
tags: [system-design, glossary]
---

# Two-Phase Commit (2PC)

> [!info] Quick definition
> A protocol for committing a transaction across multiple databases/services atomically: a coordinator asks every participant to **"prepare"** (can you commit?), and only if *all* say yes does it send **"commit"** to everyone — if any says no, everyone rolls back.

Strongly consistent, but **blocking** — if the coordinator crashes between prepare and commit, participants are stuck holding locks indefinitely. Doesn't scale well across many services or high latency links, which is why [[Saga Pattern|Saga]] is the more common modern answer for distributed transactions at scale.

> [!tip] This is a glossary stub
> Full explanation lives in: [[CS Fundamentals/06 - Distributed Systems/Distributed Transactions - Saga Pattern and Two-Phase Commit|Distributed Transactions: Saga Pattern & Two-Phase Commit]].

**Used in:** [[CS Fundamentals/06 - Distributed Systems/Distributed Transactions - Saga Pattern and Two-Phase Commit|Distributed Transactions: Saga Pattern & Two-Phase Commit]]
