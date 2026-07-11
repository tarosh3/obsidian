---
title: "Thread-Safety"
aliases: [Thread-Safe, Concurrency-Safe]
tags: [system-design, glossary]
---

# Thread-Safety

> [!info] Definition
> A property of code/data structures: they behave correctly when accessed by multiple threads (or goroutines) at the same time, with no [[Race Condition|race conditions]] — regardless of how the operations interleave.

In Go, a plain `struct` is **not** thread-safe by default — concurrent reads/writes to its fields from multiple goroutines are undefined behavior unless explicitly protected (usually with a `sync.Mutex` or `sync.RWMutex`, see [[LLD/07 - Concurrency in LLD (Go)/Index|Concurrency in LLD]]). "Is this thread-safe?" is one of the most common LLD follow-up questions once any design involves shared state accessed concurrently.

**Used in:** [[LLD/07 - Concurrency in LLD (Go)/Index|LLD 07 - Concurrency in LLD (Go)]]
