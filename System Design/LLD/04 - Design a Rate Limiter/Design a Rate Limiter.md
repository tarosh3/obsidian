---
title: "Design a Rate Limiter (LLD)"
aliases: [Design a Rate Limiter LLD, LLD Rate Limiter]
tags: [system-design, lld, case-study, golang, design-patterns, strategy-pattern]
status: reference-quality
---

# Design a Rate Limiter (LLD)

> [!abstract] What you'll be able to do after this chapter
> Implement both token bucket *and* the sliding-window-counter approximation from scratch in Go, swap between them via Strategy without touching calling code, and connect this directly to the distributed version already covered in the HLD chapter.

> [!info] This is the single-machine, pluggable-component version
> The [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|HLD chapter]] covers rate limiting *across a distributed fleet* (the Redis/Lua-script problem). This chapter is the algorithm-level component you'd actually write and drop into a service — same core algorithms, without the distributed-state problem layered on top.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Implement a rate limiter component with an `Allow(clientID) bool` method that a service can call before processing each request. It should support multiple limiting algorithms."

## Step 2 — Requirement clarification

Support swapping the underlying algorithm (token bucket for one API, a stricter sliding window for another) **without changing any calling code.** Thread-safe under concurrent calls from multiple goroutines.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> This is a shorter problem than most LLD questions — you can afford to build the "wrong" version deliberately before refactoring.
>
> **Checkpoint 1 (~10-12 min) — one algorithm, hardcoded, no interface. Get it fully correct before thinking about a second algorithm at all.**
> ```go
> type RateLimiter struct {
>     mu       sync.Mutex
>     buckets  map[string]*bucket
>     rate     float64
>     capacity float64
> }
>
> type bucket struct {
>     tokens     float64
>     lastRefill time.Time
> }
>
> func (r *RateLimiter) Allow(clientID string) bool {
>     r.mu.Lock()
>     defer r.mu.Unlock()
>     b, ok := r.buckets[clientID]
>     if !ok {
>         b = &bucket{tokens: r.capacity, lastRefill: time.Now()}
>         r.buckets[clientID] = b
>     }
>     elapsed := time.Since(b.lastRefill).Seconds()
>     b.tokens = math.Min(r.capacity, b.tokens+elapsed*r.rate)
>     b.lastRefill = time.Now()
>     if b.tokens < 1 {
>         return false
>     }
>     b.tokens--
>     return true
> }
> ```
> **Pattern used: none.** A single, correct algorithm beats a half-built interface with two broken implementations behind it.
>
> **Checkpoint 2 (~10 min) — once asked "what about a stricter limiter for payments," refactor into Strategy.**
> ```go
> // LimiterStrategy — the Strategy pattern. Replaces the single
> // hardcoded algorithm from Checkpoint 1.
> type LimiterStrategy interface {
>     Allow(clientID string) bool
> }
>
> // TokenBucketStrategy — Checkpoint 1's Allow logic, unchanged, now
> // behind the interface.
> type TokenBucketStrategy struct {
>     mu       sync.Mutex
>     rate     float64
>     capacity float64
>     buckets  map[string]*bucket
> }
>
> func (t *TokenBucketStrategy) Allow(clientID string) bool {
>     t.mu.Lock()
>     defer t.mu.Unlock()
>     b, ok := t.buckets[clientID]
>     if !ok {
>         b = &bucket{tokens: t.capacity, lastRefill: time.Now()}
>         t.buckets[clientID] = b
>     }
>     elapsed := time.Since(b.lastRefill).Seconds()
>     b.tokens = math.Min(t.capacity, b.tokens+elapsed*t.rate)
>     b.lastRefill = time.Now()
>     if b.tokens < 1 {
>         return false
>     }
>     b.tokens--
>     return true
> }
>
> // RateLimiter — now a thin wrapper. Dependency Inversion: it depends
> // on LimiterStrategy, never a concrete algorithm.
> type RateLimiter struct {
>     strategy LimiterStrategy
> }
>
> func (r *RateLimiter) Allow(clientID string) bool {
>     return r.strategy.Allow(clientID)
> }
> ```
> **Pattern used: Strategy.**
>
> **Checkpoint 3 (~10 min) — implement the second algorithm.**
> ```go
> // SlidingWindowCounterStrategy — same interface, genuinely different
> // math: weights the previous window's count by how much of it still
> // "overlaps" now, instead of storing a raw per-request timestamp log.
> type SlidingWindowCounterStrategy struct {
>     mu         sync.Mutex
>     limit      int
>     windowSize time.Duration
>     counters   map[string]*windowCounter
> }
>
> type windowCounter struct {
>     currentWindowStart time.Time
>     currentCount       int
>     previousCount      int
> }
>
> func (s *SlidingWindowCounterStrategy) Allow(clientID string) bool {
>     s.mu.Lock()
>     defer s.mu.Unlock()
>     now := time.Now()
>     wc, ok := s.counters[clientID]
>     if !ok {
>         wc = &windowCounter{currentWindowStart: now}
>         s.counters[clientID] = wc
>     }
>     elapsed := now.Sub(wc.currentWindowStart)
>     if elapsed >= s.windowSize {
>         wc.previousCount = wc.currentCount
>         wc.currentCount = 0
>         wc.currentWindowStart = now
>         elapsed = 0
>     }
>     fraction := float64(elapsed) / float64(s.windowSize)
>     weighted := float64(wc.previousCount)*(1-fraction) + float64(wc.currentCount)
>     if weighted >= float64(s.limit) {
>         return false
>     }
>     wc.currentCount++
>     return true
> }
> ```
>
> **Checkpoint 4 (remaining time, or if asked) — the distributed jump.** No new code needed — describe verbally: the `map[string]*bucket` here is local, in-process memory; [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|the HLD chapter]] replaces it with Redis state and turns the check-refill-decrement sequence into one atomic Lua script, since multiple *servers* (not just goroutines) can now race on the same client's bucket.
>
> **If you're short on time:** stop after Checkpoint 1. A single, fully correct token-bucket limiter is a complete, working answer — describe the Strategy extraction verbally as how you'd support a second algorithm without touching this code.

## Step 3 — The bad first draft

```go
type RateLimiter struct {
	mu       sync.Mutex
	buckets  map[string]*bucket
	rate     float64
	capacity float64
}

type bucket struct {
	tokens     float64
	lastRefill time.Time
}

func (r *RateLimiter) Allow(clientID string) bool {
	r.mu.Lock()
	defer r.mu.Unlock()
	b, ok := r.buckets[clientID]
	if !ok {
		b = &bucket{tokens: r.capacity, lastRefill: time.Now()}
		r.buckets[clientID] = b
	}
	elapsed := time.Since(b.lastRefill).Seconds()
	b.tokens = min(r.capacity, b.tokens+elapsed*r.rate)
	b.lastRefill = time.Now()
	if b.tokens < 1 {
		return false
	}
	b.tokens--
	return true
}
```

## Step 4 — Why it breaks

> [!bug] Requirement: "the payments API needs a stricter sliding-window limiter; the general API is fine with token bucket's burst tolerance."
> There is exactly one algorithm baked directly into `RateLimiter`. Satisfying this means either duplicating the entire struct into a near-identical `RateLimiter2`, or bolting an `algorithm string` field onto `RateLimiter` and adding an `if/else` inside `Allow` — the same **Open/Closed Principle** violation shape as the pricing logic in [[LLD/01 - Design a Parking Lot/Design a Parking Lot|the Parking Lot chapter]] and the state checks in [[LLD/02 - Design a Vending Machine/Design a Vending Machine|the Vending Machine chapter]] — this pattern of failure keeps recurring for the exact same underlying reason: behavior welded directly into a concrete type instead of pulled behind an interface.

## Step 5 — Refactor: Strategy

`LimiterStrategy` becomes the abstraction — one method, `Allow(clientID) bool`. `TokenBucketStrategy` and `SlidingWindowCounterStrategy` each implement it independently. `RateLimiter` becomes a thin wrapper holding whichever strategy was injected at construction — swapping algorithms per API is a **constructor argument**, not a code change.

---

## Step 6 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: strategy.go
// ============================================================
package ratelimiter

// LimiterStrategy is the abstraction that replaced the single
// hardcoded algorithm from the bad first draft.
type LimiterStrategy interface {
	Allow(clientID string) bool
}
```

```go
// ============================================================
// FILE: token_bucket.go
// ============================================================
package ratelimiter

import (
	"math"
	"sync"
	"time"
)

// TokenBucketStrategy allows bursts up to `capacity`, refilling at
// `rate` tokens/second — same algorithm discussed in the HLD chapter,
// implemented here as a standalone, embeddable component.
type TokenBucketStrategy struct {
	mu       sync.Mutex
	rate     float64
	capacity float64
	buckets  map[string]*tokenBucket
}

type tokenBucket struct {
	tokens     float64
	lastRefill time.Time
}

func NewTokenBucketStrategy(rate, capacity float64) *TokenBucketStrategy {
	return &TokenBucketStrategy{
		rate:     rate,
		capacity: capacity,
		buckets:  make(map[string]*tokenBucket),
	}
}

func (t *TokenBucketStrategy) Allow(clientID string) bool {
	t.mu.Lock()
	defer t.mu.Unlock()

	b, ok := t.buckets[clientID]
	if !ok {
		b = &tokenBucket{tokens: t.capacity, lastRefill: time.Now()}
		t.buckets[clientID] = b
	}

	now := time.Now()
	elapsed := now.Sub(b.lastRefill).Seconds()
	b.tokens = math.Min(t.capacity, b.tokens+elapsed*t.rate)
	b.lastRefill = now

	if b.tokens < 1 {
		return false
	}
	b.tokens--
	return true
}
```

```go
// ============================================================
// FILE: sliding_window.go
// ============================================================
package ratelimiter

import (
	"sync"
	"time"
)

// SlidingWindowCounterStrategy approximates a true sliding window
// cheaply: it weights the previous fixed window's count by how much
// of it still "overlaps" the current moment, avoiding the memory
// cost of a full request-timestamp log (the "sliding window log"
// approach named in the HLD chapter).
type SlidingWindowCounterStrategy struct {
	mu         sync.Mutex
	limit      int
	windowSize time.Duration
	counters   map[string]*windowCounter
}

type windowCounter struct {
	currentWindowStart time.Time
	currentCount       int
	previousCount      int
}

func NewSlidingWindowCounterStrategy(limit int, windowSize time.Duration) *SlidingWindowCounterStrategy {
	return &SlidingWindowCounterStrategy{
		limit:      limit,
		windowSize: windowSize,
		counters:   make(map[string]*windowCounter),
	}
}

func (s *SlidingWindowCounterStrategy) Allow(clientID string) bool {
	s.mu.Lock()
	defer s.mu.Unlock()

	now := time.Now()
	wc, ok := s.counters[clientID]
	if !ok {
		wc = &windowCounter{currentWindowStart: now}
		s.counters[clientID] = wc
	}

	elapsed := now.Sub(wc.currentWindowStart)
	if elapsed >= s.windowSize {
		windowsPassed := int(elapsed / s.windowSize)
		if windowsPassed == 1 {
			// Exactly one window boundary crossed — the just-finished
			// window becomes the new "previous" for weighting.
			wc.previousCount = wc.currentCount
		} else {
			// More than one window has fully elapsed with no
			// activity — the old count is too stale to weight at all.
			wc.previousCount = 0
		}
		wc.currentCount = 0
		wc.currentWindowStart = wc.currentWindowStart.Add(time.Duration(windowsPassed) * s.windowSize)
		elapsed = now.Sub(wc.currentWindowStart)
	}

	fractionElapsed := float64(elapsed) / float64(s.windowSize)
	weightedCount := float64(wc.previousCount)*(1-fractionElapsed) + float64(wc.currentCount)

	if weightedCount >= float64(s.limit) {
		return false
	}
	wc.currentCount++
	return true
}
```

```go
// ============================================================
// FILE: limiter.go
// ============================================================
package ratelimiter

// RateLimiter is the public component a service actually holds and
// calls — a thin wrapper delegating to whichever LimiterStrategy was
// injected. This is the Dependency Inversion half of the refactor:
// RateLimiter depends on the interface, never a concrete strategy.
type RateLimiter struct {
	strategy LimiterStrategy
}

func NewRateLimiter(strategy LimiterStrategy) *RateLimiter {
	return &RateLimiter{strategy: strategy}
}

func (r *RateLimiter) Allow(clientID string) bool {
	return r.strategy.Allow(clientID)
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"
	"time"

	ratelimiter "example.com/ratelimiter"
)

func main() {
	// General API: tolerates bursts up to 10, refilling at 5/sec.
	generalAPILimiter := ratelimiter.NewRateLimiter(
		ratelimiter.NewTokenBucketStrategy(5, 10),
	)

	// Payments API: stricter, no burst tolerance — max 3 per second.
	paymentsAPILimiter := ratelimiter.NewRateLimiter(
		ratelimiter.NewSlidingWindowCounterStrategy(3, time.Second),
	)

	for i := 0; i < 12; i++ {
		fmt.Println("general:", generalAPILimiter.Allow("user-1"))
	}
	for i := 0; i < 5; i++ {
		fmt.Println("payments:", paymentsAPILimiter.Allow("user-1"))
	}
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "How would you extend this to work across multiple servers, not just one process?"
> This is exactly the jump [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|the HLD chapter]] makes: the `map[string]*tokenBucket` here is local, in-process memory — swap it for state stored in Redis, and the check-refill-decrement sequence for a single client's bucket needs to become one **atomic Redis Lua script** instead of a local mutex-protected map access, since multiple *servers* (not just goroutines) can now race on the same client's bucket.

> [!quote]- "Why does `SlidingWindowCounterStrategy` track a `previousCount` at all instead of just `currentCount`?"
> Without it, the algorithm degrades to a plain fixed-window counter — reintroducing the exact boundary-burst flaw named in the HLD chapter's algorithm comparison. Weighting in the previous window's count, proportional to how much of it still "overlaps" the present moment, is what makes this a genuine *sliding* window approximation rather than a fixed one.

> [!quote]- "What happens under this implementation if a client makes zero requests for a long time, then comes back?"
> `windowsPassed` will be `≥ 2`, taking the `else` branch — `previousCount` resets to `0` rather than carrying forward a now-meaningless old count, and the client effectively starts fresh with a full new window's allowance. This is the correct behavior: a long-idle client shouldn't be penalized by stale historical activity.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|HLD version — Design a Rate Limiter]] · [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]]*
