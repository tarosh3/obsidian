---
title: "Idempotency"
aliases: [Idempotent, Idempotency]
tags: [system-design, glossary]
---

# Idempotency

> [!info] Definition
> An operation is idempotent if performing it multiple times has the **exact same effect** as performing it once. Critical for anything that might get retried — network calls, message processing, payment APIs.

`PUT /users/5 {name: "Tarosh"}` is idempotent — running it 5 times leaves the same end state. `POST /charge {amount: 500}` is **not** idempotent by default — running it twice charges the user twice. The standard fix: attach a unique **idempotency key** to the request; the server checks if it's already processed that key before, and if so, returns the original result instead of re-executing.

> [!quote] Interview framing
> "Your payment consumer crashed mid-processing and restarted — did the user get charged twice?" is one of the most common messaging-system interview questions — the correct answer is always "no, because the write is idempotent via a dedup key," not "we made sure it never crashes."

> [!tip] This is a glossary stub
> Full explanation lives in: [[CS Fundamentals/06 - Distributed Systems/Idempotency|Idempotency]].

**Used in:** [[CS Fundamentals/06 - Distributed Systems/Idempotency|Idempotency]] · [[CS Fundamentals/06 - Distributed Systems/Resilience Patterns|Resilience Patterns]] · [[HLD/17 - Design a Payment System/Design a Payment System|Design a Payment System]]
