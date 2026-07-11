---
title: "Backpressure"
aliases: [Backpressure]
tags: [system-design, glossary]
---

# Backpressure

> [!info] Definition
> A mechanism letting a slow consumer signal a fast producer to slow down — instead of the producer blindly overwhelming the consumer (unbounded queue growth, memory exhaustion, or crashes).

In Go, a **bounded/buffered channel** is the simplest backpressure primitive — once the channel's buffer fills, a `send` blocks until the consumer catches up, naturally throttling the producer. Without backpressure, a fast producer + slow consumer pairing silently builds an ever-growing queue until something runs out of memory.

> [!example] Layman's terms
> A factory conveyor belt that stops feeding new parts in when the packing station at the end is backed up — instead of parts piling up on the floor uncontrolled.

> [!tip] Related full chapters
> [[CS Fundamentals/Distributed Systems/Resilience Patterns|Resilience Patterns]] (bulkheads, timeout budgets — the neighboring resilience techniques) · [[CS Fundamentals/Messaging & Streaming/Kafka Internals|Kafka Internals]] (a consumer falling behind is exactly a backpressure signal, visible as consumer lag)
