---
title: "Design an LRU Cache (as a system, pluggable eviction)"
aliases: [Design an LRU Cache LLD, Pluggable Cache Eviction]
tags: [system-design, lld, case-study, golang, design-patterns, strategy-pattern]
status: reference-quality
---

# Design an LRU Cache (as a system, pluggable eviction)

> [!abstract] What you'll be able to do after this chapter
> Wrap the exact LRU mechanics already mastered in the DSA vault behind a pluggable `EvictionPolicy` interface — so a cache can swap LRU for LFU without its `Get`/`Put` callers ever knowing.

> [!info] This builds directly on existing work, doesn't repeat it
> [[DSA/Linked List/LRU Cache (LeetCode #146)|The DSA LRU Cache note]] already covers the hashmap + doubly-linked-list mechanics in full depth. This chapter's value is different: wrapping that algorithm behind an `EvictionPolicy` abstraction so the *cache* doesn't hardcode one specific eviction algorithm — read the DSA note first if the raw mechanics aren't already second nature.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a generic, thread-safe cache component supporting pluggable eviction policies (LRU, LFU) without changing the cache's core `Get`/`Put` interface."

## Step 2 — Requirement clarification

`O(1)`-average `Get`/`Put`. Configurable eviction policy at construction time. Thread-safe under concurrent access.

## Step 3 — The bad first draft (kept brief)

A `Cache` struct with one eviction algorithm hardcoded directly inline — the same Open/Closed shape covered many times already. This chapter's value isn't re-teaching that shape; it's the pluggability wrapper itself.

## Step 4 — Refactor: `EvictionPolicy` as the abstraction

`EvictionPolicy` — `RecordAccess`, `RecordInsertion`, `Evict`, `Remove` — implemented independently by `LRUPolicy` and `LFUPolicy`. `Cache` owns the actual key/value storage and delegates every eviction *decision* to whichever policy it was constructed with.

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: eviction_policy.go
// ============================================================
package cache

// EvictionPolicy decides WHICH key to evict when full — the cache
// itself never encodes a specific eviction algorithm.
type EvictionPolicy interface {
	RecordAccess(key string)
	RecordInsertion(key string)
	Evict() (key string, ok bool)
	Remove(key string)
}
```

```go
// ============================================================
// FILE: lru_policy.go
// ============================================================
package cache

import "container/list"

// LRUPolicy wraps the exact hashmap + doubly-linked-list idea
// already covered in full in the DSA LRU Cache note — using Go's
// standard container/list instead of hand-rolling the linked list,
// since here the LEARNING goal is the pluggability wrapper, not
// re-deriving the linked-list mechanics again.
type LRUPolicy struct {
	order    *list.List
	elements map[string]*list.Element
}

func NewLRUPolicy() *LRUPolicy {
	return &LRUPolicy{order: list.New(), elements: make(map[string]*list.Element)}
}

func (p *LRUPolicy) RecordAccess(key string) {
	if elem, ok := p.elements[key]; ok {
		p.order.MoveToFront(elem)
	}
}

func (p *LRUPolicy) RecordInsertion(key string) {
	elem := p.order.PushFront(key)
	p.elements[key] = elem
}

func (p *LRUPolicy) Evict() (string, bool) {
	back := p.order.Back()
	if back == nil {
		return "", false
	}
	key := back.Value.(string)
	p.order.Remove(back)
	delete(p.elements, key)
	return key, true
}

func (p *LRUPolicy) Remove(key string) {
	if elem, ok := p.elements[key]; ok {
		p.order.Remove(elem)
		delete(p.elements, key)
	}
}
```

```go
// ============================================================
// FILE: lfu_policy.go
// ============================================================
package cache

// LFUPolicy evicts the least-frequently-used key. See the Caching
// Strategies chapter for the aging/staleness nuance a production
// LFU needs (an old, once-popular key can otherwise never be
// evicted) — this is the core frequency-tracking mechanics.
type LFUPolicy struct {
	frequency map[string]int
}

func NewLFUPolicy() *LFUPolicy {
	return &LFUPolicy{frequency: make(map[string]int)}
}

func (p *LFUPolicy) RecordAccess(key string)    { p.frequency[key]++ }
func (p *LFUPolicy) RecordInsertion(key string) { p.frequency[key] = 1 }

func (p *LFUPolicy) Evict() (string, bool) {
	var minKey string
	minFreq := -1
	for k, f := range p.frequency {
		if minFreq == -1 || f < minFreq {
			minFreq = f
			minKey = k
		}
	}
	if minFreq == -1 {
		return "", false
	}
	delete(p.frequency, minKey)
	return minKey, true
}

func (p *LFUPolicy) Remove(key string) {
	delete(p.frequency, key)
}
```

```go
// ============================================================
// FILE: cache.go
// ============================================================
package cache

import "sync"

type Cache struct {
	mu       sync.Mutex
	capacity int
	data     map[string]interface{}
	policy   EvictionPolicy
}

func NewCache(capacity int, policy EvictionPolicy) *Cache {
	return &Cache{capacity: capacity, data: make(map[string]interface{}), policy: policy}
}

func (c *Cache) Get(key string) (interface{}, bool) {
	c.mu.Lock()
	defer c.mu.Unlock()
	value, ok := c.data[key]
	if ok {
		c.policy.RecordAccess(key)
	}
	return value, ok
}

func (c *Cache) Put(key string, value interface{}) {
	c.mu.Lock()
	defer c.mu.Unlock()

	if _, exists := c.data[key]; exists {
		c.data[key] = value
		c.policy.RecordAccess(key)
		return
	}

	if len(c.data) >= c.capacity {
		if evictKey, ok := c.policy.Evict(); ok {
			delete(c.data, evictKey)
		}
	}

	c.data[key] = value
	c.policy.RecordInsertion(key)
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"

	cache "example.com/cache"
)

func main() {
	lru := cache.NewCache(2, cache.NewLRUPolicy())
	lru.Put("a", 1)
	lru.Put("b", 2)
	lru.Get("a")     // "a" is now most-recently-used
	lru.Put("c", 3)  // evicts "b" (least recently used), not "a"

	if _, ok := lru.Get("b"); !ok {
		fmt.Println("b was correctly evicted")
	}
	if v, ok := lru.Get("a"); ok {
		fmt.Println("a survived:", v)
	}
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "How would you switch this cache from LRU to LFU eviction in production, live?"
> You wouldn't, live — but at construction time, `NewCache(capacity, cache.NewLFUPolicy())` instead of `NewLRUPolicy()` is the entire change needed. `Cache.Get`/`Put` never reference either concrete policy by name, only the `EvictionPolicy` interface.

> [!quote]- "Why does `container/list` get used here instead of the hand-rolled doubly linked list from the DSA note?"
> Both are correct and `O(1)`; using the standard library here keeps this chapter's code focused on the *wrapper* being taught (pluggable eviction), rather than re-deriving linked-list mechanics already covered in full elsewhere — worth being explicit about this choice if asked, since hand-rolling it is equally valid and sometimes expected depending on what the interviewer is actually probing for.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[DSA/Linked List/LRU Cache (LeetCode #146)|DSA — LRU Cache]] · [[CS Fundamentals/Caching/Caching Strategies|Caching Strategies]]*
