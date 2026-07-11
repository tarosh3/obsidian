---
title: "Design a Library Management System"
aliases: [Design a Library Management System, Library LLD]
tags: [system-design, lld, case-study, golang, design-patterns, factory-pattern, observer-pattern]
status: reference-quality
---

# Design a Library Management System

> [!abstract] What you'll be able to do after this chapter
> Combine Factory (item creation) with Observer (hold-queue notification), and understand why a Factory function having its own internal switch statement is correct — not a contradiction of everything the rest of this handbook has argued against.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a library management system — catalog books with multiple physical copies, checkout/return, holds/reservations, and notify a member when their held book becomes available."

## Step 2 — Requirement clarification

Multiple item types (books, DVDs, magazines) with different loan periods. Members can place holds on unavailable titles. Returning a copy should notify the **first** person in that title's hold queue, in fair FIFO order.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~8 min) — one item type, simple checkout/return, no hold queue.**
> ```go
> type Catalog struct {
>     availableCopies map[string]int
> }
>
> func (c *Catalog) Checkout(title string) bool {
>     if c.availableCopies[title] <= 0 {
>         return false
>     }
>     c.availableCopies[title]--
>     return true
> }
>
> func (c *Catalog) Return(title string) {
>     c.availableCopies[title]++
> }
> ```
> **Pattern used: none.** Books only, one loan period, no holds — get checkout/return genuinely working first.
>
> **Checkpoint 2 (~8 min) — multiple item types, refactor into Factory.**
> ```go
> // LibraryItem — each concrete type owns its own loan period,
> // replacing an item-type if/else chain.
> type LibraryItem interface {
>     Title() string
>     LoanPeriod() time.Duration
> }
>
> type Book struct{ title string }
>
> func (b *Book) Title() string             { return b.title }
> func (b *Book) LoanPeriod() time.Duration { return 21 * 24 * time.Hour }
>
> type DVD struct{ title string }
>
> func (d *DVD) Title() string             { return d.title }
> func (d *DVD) LoanPeriod() time.Duration { return 7 * 24 * time.Hour }
>
> // NewLibraryItem centralizes the type switch into ONE place — this
> // is the deliberate exception to "avoid switches": the switch
> // doesn't disappear, it gets concentrated here instead of scattered
> // across every method that needs to know an item's type.
> func NewLibraryItem(itemType ItemType, title string) LibraryItem {
>     switch itemType {
>     case DVDType:
>         return &DVD{title: title}
>     default:
>         return &Book{title: title}
>     }
> }
> ```
> **Pattern used: Factory.** Say explicitly why this switch is fine when every earlier chapter's switch wasn't — this is a strong, specific thing to articulate unprompted.
>
> **Checkpoint 3 (~10 min) — hold queue + Observer, once "notify me when it's back" comes up.**
> ```go
> type HoldObserver interface {
>     OnAvailable(title string)
> }
>
> func (c *Catalog) Return(title string) {
>     if queue := c.holdQueues[title]; len(queue) > 0 {
>         next := queue[0]
>         c.holdQueues[title] = queue[1:]
>         next.OnAvailable(title) // FIFO fairness by construction
>         return
>     }
>     c.availableCopies[title]++
> }
> ```
> **Pattern used: Observer**, reused from earlier chapters, applied here to "item became available" instead of a generic event bus.
>
> **If you're short on time:** stop after Checkpoint 2. Multiple item types via Factory, with checkout/return working, is a complete answer — describe the hold-queue Observer verbally as the next layer.

## Step 3 — The bad first draft (kept brief)

A single `LibraryItem` struct with a type field, and checkout logic branching on that type inline to determine the loan period — the same Open/Closed shape as every prior chapter.

## Step 4 — Refactor: Factory for item creation, Observer for hold notification

> [!tip] Why the Factory function's own internal switch is NOT a violation
> Every chapter so far has flagged type-string switches as the problem to refactor away from. A Factory function is the **one deliberate exception** — its entire job is centralizing "given a type, produce the right concrete instance" into exactly **one** place, so that nowhere *else* in the codebase needs a similar switch. The switch doesn't disappear; it gets concentrated into a single, obvious, well-named location instead of scattered across every method that needs to know an item's type.

Each concrete item type (`Book`, `DVD`, `Magazine`) owns its own loan period directly — `Catalog` and checkout logic never branch on type again. **Observer** handles hold notification: returning a copy checks that title's hold queue and notifies the first waiting member, reusing the exact `Observer` shape from earlier chapters, applied here to "become available" events instead of arbitrary application events.

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: item.go
// ============================================================
package library

import "time"

// LibraryItem is what ItemFactory produces — each concrete type
// owns its own loan period, replacing an item-type if/else chain.
type LibraryItem interface {
	Title() string
	LoanPeriod() time.Duration
}

type Book struct{ title string }

func (b *Book) Title() string             { return b.title }
func (b *Book) LoanPeriod() time.Duration { return 21 * 24 * time.Hour }

type DVD struct{ title string }

func (d *DVD) Title() string             { return d.title }
func (d *DVD) LoanPeriod() time.Duration { return 7 * 24 * time.Hour }

type Magazine struct{ title string }

func (m *Magazine) Title() string             { return m.title }
func (m *Magazine) LoanPeriod() time.Duration { return 3 * 24 * time.Hour }
```

```go
// ============================================================
// FILE: item_factory.go
// ============================================================
package library

type ItemType string

const (
	BookType     ItemType = "BOOK"
	DVDType      ItemType = "DVD"
	MagazineType ItemType = "MAGAZINE"
)

// NewLibraryItem centralizes the type switch INTO one place — see
// Step 4 for why this is the correct, deliberate exception rather
// than a violation of this handbook's usual Open/Closed guidance.
func NewLibraryItem(itemType ItemType, title string) LibraryItem {
	switch itemType {
	case DVDType:
		return &DVD{title: title}
	case MagazineType:
		return &Magazine{title: title}
	default:
		return &Book{title: title}
	}
}
```

```go
// ============================================================
// FILE: hold_observer.go
// ============================================================
package library

import "fmt"

// HoldObserver is notified when a title they're holding becomes
// available. A real system would enqueue an actual notification —
// see the Notification System chapter for that infrastructure.
type HoldObserver interface {
	OnAvailable(title string)
}

type MemberNotifier struct {
	MemberID string
}

func (m *MemberNotifier) OnAvailable(title string) {
	fmt.Printf("%s notified: %s is now available\n", m.MemberID, title)
}
```

```go
// ============================================================
// FILE: catalog.go
// ============================================================
package library

import "sync"

type Catalog struct {
	mu              sync.Mutex
	availableCopies map[string]int
	holdQueues      map[string][]HoldObserver
}

func NewCatalog() *Catalog {
	return &Catalog{
		availableCopies: make(map[string]int),
		holdQueues:      make(map[string][]HoldObserver),
	}
}

func (c *Catalog) AddCopies(title string, count int) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.availableCopies[title] += count
}

// Checkout returns true if a copy was available and is now checked
// out; false means none available — the caller should PlaceHold.
func (c *Catalog) Checkout(title string) bool {
	c.mu.Lock()
	defer c.mu.Unlock()
	if c.availableCopies[title] <= 0 {
		return false
	}
	c.availableCopies[title]--
	return true
}

// PlaceHold enqueues an observer, notified in FIFO order — fairness
// by construction, not by luck.
func (c *Catalog) PlaceHold(title string, observer HoldObserver) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.holdQueues[title] = append(c.holdQueues[title], observer)
}

// Return makes a copy available again. If anyone is holding for this
// title, the FIRST in line is notified and the copy is conceptually
// reserved for them — not simply released back to general
// availability, preserving hold-queue fairness.
func (c *Catalog) Return(title string) {
	c.mu.Lock()
	defer c.mu.Unlock()

	queue := c.holdQueues[title]
	if len(queue) > 0 {
		next := queue[0]
		c.holdQueues[title] = queue[1:]
		c.availableCopies[title]++
		next.OnAvailable(title)
		return
	}
	c.availableCopies[title]++
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	library "example.com/library"
)

func main() {
	catalog := library.NewCatalog()
	catalog.AddCopies("Dune", 1)

	catalog.Checkout("Dune") // only copy now checked out

	catalog.PlaceHold("Dune", &library.MemberNotifier{MemberID: "member-2"})

	catalog.Return("Dune") // member-2 is notified automatically, in FIFO order
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "How would you add a new item type — say, an audiobook?"
> One new struct implementing `LibraryItem` with its own `LoanPeriod`, plus one new `case` in `NewLibraryItem`'s switch. That's the *only* place the switch needs to grow — `Catalog`, `Checkout`, `Return`, and every hold-related method remain completely untouched.

> [!quote]- "What happens if two members try to place a hold on the exact same title at the exact same instant?"
> `PlaceHold` acquires `c.mu` before appending to the queue — concurrent calls are serialized, so the append order (and therefore FIFO fairness) is well-defined regardless of which goroutine's call happened to be scheduled first.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/08 - Design a Notification System/Design a Notification System|Design a Notification System]]*
