---
title: "PACELC Theorem"
aliases: [PACELC]
tags: [system-design, glossary]
---

# PACELC Theorem

> [!info] Quick definition
> Extends [[CAP Theorem|CAP]]: if there's a **P**artition, choose **A**vailability or **C**onsistency (that's the "PAC" — same as CAP). **E**lse (no partition, normal operation), you still choose between **L**atency and **C**onsistency (the "ELC").

CAP only describes behavior *during* a partition — which is rare. PACELC covers the much more common "everything's fine" state too: even with no partition, replicating data for stronger consistency costs latency (you wait for more nodes to confirm). This is why it's considered the more complete, more practically useful theorem — and why bringing it up unprompted after CAP is a strong depth signal in an interview.

> [!tip] This is a glossary stub
> Full explanation, real-system classifications (DynamoDB/MongoDB/Cassandra), and interview Q&A: [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC|CAP Theorem & PACELC]].

**Used in:** [[CS Fundamentals/06 - Distributed Systems/CAP Theorem & PACELC|CAP Theorem & PACELC]]
