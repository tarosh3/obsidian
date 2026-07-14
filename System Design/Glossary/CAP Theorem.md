---
title: "CAP Theorem"
aliases: [CAP, CAP Theorem]
tags: [system-design, glossary]
---

# CAP Theorem

> [!info] Quick definition
> During a network **P**artition, a distributed system must choose between **C**onsistency (every node sees the same data) and **A**vailability (the system keeps responding) — you cannot have both at once. Named for the three letters: Consistency, Availability, Partition tolerance.

Partition tolerance isn't really optional in a real distributed system (networks *will* partition eventually), so in practice CAP reduces to a **CP vs AP** choice once a partition happens. See also [[PACELC Theorem|PACELC]], which extends this to cover the far more common "no partition, everything's fine" case CAP doesn't address.

> [!tip] This is a glossary stub
> Full explanation with real examples, tradeoffs, and interview Q&A: [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC|CAP Theorem & PACELC]].

**Used in:** [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC|CAP Theorem & PACELC]], [[HLD/01 - Design TinyURL (URL Shortener)/Design TinyURL|Design TinyURL]]
