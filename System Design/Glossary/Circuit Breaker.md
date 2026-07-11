---
title: "Circuit Breaker"
aliases: [Circuit Breaker]
tags: [system-design, glossary]
---

# Circuit Breaker

> [!info] Quick definition
> A pattern that stops calling a failing downstream service after too many failures, preventing **cascading failure** — instead of every request piling up waiting on a doomed call, the breaker "trips" and fails fast.

Three states: **Closed** (normal, calls pass through) → too many failures → **Open** (calls fail immediately, no network call made at all) → after a cooldown → **Half-Open** (lets a few test calls through to check if the dependency recovered) → success returns to Closed, failure returns to Open.

> [!example] Layman's terms
> An actual electrical circuit breaker: instead of letting a short circuit keep drawing dangerous current and starting a fire, it trips and cuts power — you fix the problem, then flip it back on.

> [!tip] Full chapter
> Complete state-machine explanation, a real Go implementation, and how it composes with retry/bulkhead/timeout: [[CS Fundamentals/Distributed Systems/Resilience Patterns|Resilience Patterns (Circuit Breaker, Retry, Bulkhead)]].
