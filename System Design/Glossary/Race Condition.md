---
title: "Race Condition"
aliases: [Race Condition, Data Race]
tags: [system-design, glossary]
---

# Race Condition

> [!info] Definition
> A bug where the correctness of a program depends on the timing/ordering of concurrent operations — two or more threads/goroutines access shared state, and the outcome differs depending on which one "wins" the race.

Classic example: two goroutines both read a counter's current value, both increment it locally, both write back — one increment is silently lost, because both reads happened before either write. The fix is always some form of synchronization (mutex, atomic operation, or channel-based serialization) around the shared state.

> [!example] Layman's terms
> Two people editing the same shared spreadsheet cell at the exact same moment, both starting from the same "before" value — whoever saves last wins, and the other person's change vanishes without either of them noticing.

**Used in:** [[LLD/07 - Concurrency in LLD (Go)/Index|LLD 07 - Concurrency in LLD (Go)]]
