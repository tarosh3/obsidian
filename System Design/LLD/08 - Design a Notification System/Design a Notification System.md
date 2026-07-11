---
title: "Design a Notification System (LLD)"
aliases: [Design a Notification System, Notification System LLD]
tags: [system-design, lld, case-study, golang, design-patterns, observer-pattern, chain-of-responsibility]
status: reference-quality
---

# Design a Notification System (LLD)

> [!abstract] What you'll be able to do after this chapter
> Combine Observer with a Chain-of-Responsibility-style filter chain — implemented as an idiomatic Go slice instead of a linked-node structure — and know precisely which CoR variant applies here vs the Logger Framework chapter's deliberate deviation.

> [!info] Distinct from the HLD version
> [[HLD/04 - Design a Notification Service/Design a Notification Service|The HLD chapter]] covers distributed infrastructure — Kafka, fan-out at scale, per-provider rate limiting. This chapter is the **in-process object design**: how a single event notifies multiple interested parties, each with its own delivery rules.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a notification component where various events (new comment, new follower, price drop) can trigger notifications to interested subscribers across multiple channels — extensible for new event types and channels without modifying existing code."

## Step 2 — Requirement clarification

Multiple subscribers can care about the **same** event. Each notification may need filtering (e.g. respect do-not-disturb hours, per-type mute settings) before actually being delivered. Adding a new event type or channel must not require editing existing code.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~8 min) — Observer alone, no filtering yet.**
> ```go
> // Observer — the Observer pattern. Publisher never knows who's
> // subscribed or how many observers exist.
> type Observer interface {
>     OnEvent(event Event)
> }
>
> type EventPublisher struct {
>     observers []Observer
> }
>
> func (p *EventPublisher) Subscribe(o Observer) {
>     p.observers = append(p.observers, o)
> }
>
> func (p *EventPublisher) Publish(event Event) {
>     for _, o := range p.observers {
>         o.OnEvent(event)
>     }
> }
> ```
> **Pattern used: Observer.** One event type, one observer that just prints — get this firing end-to-end before adding filtering or multiple channels.
>
> **Checkpoint 2 (~5 min) — a second observer, prove it fans out.**
> ```go
> type ConsoleObserver struct{ Name string }
>
> func (c *ConsoleObserver) OnEvent(event Event) {
>     fmt.Printf("[%s] got event: %s\n", c.Name, event.Type)
> }
>
> publisher.Subscribe(&ConsoleObserver{Name: "email-service"})
> publisher.Subscribe(&ConsoleObserver{Name: "sms-service"})
> publisher.Publish(Event{Type: NewComment})
> // both fire — zero coupling between them
> ```
> No new pattern — just proving Observer's actual payoff (many independent subscribers, zero coupling between them).
>
> **Checkpoint 3 (~10 min) — add delivery filtering, once "respect do-not-disturb hours" comes up.**
> ```go
> // DeliveryFilter — a Chain of Responsibility variant, contrasted
> // with the Logger Framework chapter: the FIRST filter that rejects
> // genuinely stops delivery, unlike Logger's all-handlers-act version.
> type DeliveryFilter interface {
>     ShouldDeliver(event Event) bool
> }
>
> // FilterChain: an ordered SLICE, not a linked chain of node objects
> // — the more idiomatic Go expression of the same intent.
> type FilterChain struct {
>     filters []DeliveryFilter
> }
>
> func (c *FilterChain) ShouldDeliver(event Event) bool {
>     for _, f := range c.filters {
>         if !f.ShouldDeliver(event) {
>             return false // first rejection stops the chain
>         }
>     }
>     return true
> }
>
> type MuteFilter struct{ mutedTypes map[EventType]bool }
>
> func (f *MuteFilter) ShouldDeliver(event Event) bool {
>     return !f.mutedTypes[event.Type]
> }
> ```
> **Pattern used: Chain of Responsibility (first-rejection-stops variant) composed with Observer.** Say explicitly why this variant differs from the Logger chapter's — the two patterns solve genuinely different problems ("who's interested" vs. "should this specific delivery happen right now").
>
> **Checkpoint 4 (remaining time, or if asked) — wire a `ChannelObserver` combining both.**
> ```go
> // ChannelObserver composes Observer + the filter chain — a concrete
> // Observer that delivers to one channel only after filters pass.
> type ChannelObserver struct {
>     channel Channel
>     filters *FilterChain
> }
>
> func (o *ChannelObserver) OnEvent(event Event) {
>     if !o.filters.ShouldDeliver(event) {
>         return
>     }
>     o.channel.Send(event)
> }
> ```
> No new pattern — composition of what already exists.
>
> **If you're short on time:** stop after Checkpoint 2. A working Observer with multiple independent subscribers is a complete answer to "how do multiple parties get notified" — describe the filter chain verbally as how you'd add delivery rules on top.

## Step 3 — The bad first draft (kept brief — you've seen this shape 6 times now)

A single `NotificationManager` with a giant `switch` on event type, hardcoding channel-sending logic inline per branch. Adding a new event type *or* a new channel means editing the same sprawling method — the same Open/Closed violation as every prior chapter's bad draft.

## Step 4 — Refactor: Observer + a filter chain — expressed idiomatically

**Observer** handles "who's interested" — an `EventPublisher` fires events without knowing or caring who's subscribed or how many observers exist. **A filter chain** handles "should this actually be delivered right now."

> [!tip] The filter chain here is a SLICE, not a linked list of objects — and it stops at the first rejection
> The [[LLD/07 - Design a Logger Framework/Design a Logger Framework|Logger Framework chapter]] deliberately deviated from textbook Chain of Responsibility — every handler acted, regardless of earlier ones. Here, the **opposite** applies: the first filter that says "don't deliver" genuinely **should** stop the chain — the textbook "first handler wins" version. Which variant is correct depends entirely on the problem's actual semantics, not a fixed rule to memorize. And rather than replicating a linked-node structure (idiomatic in languages built around inheritance), an **ordered slice, iterated in sequence**, satisfies the same Chain of Responsibility *intent* far more idiomatically in Go.

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: event.go
// ============================================================
package notification

type EventType string

const (
	NewComment  EventType = "NEW_COMMENT"
	NewFollower EventType = "NEW_FOLLOWER"
	PriceDrop   EventType = "PRICE_DROP"
)

type Event struct {
	Type    EventType
	UserID  string
	Payload map[string]string
}
```

```go
// ============================================================
// FILE: observer.go
// ============================================================
package notification

// Observer is any component that reacts to a fired event —
// EventPublisher never knows who's listening or how many exist.
type Observer interface {
	OnEvent(event Event)
}
```

```go
// ============================================================
// FILE: publisher.go
// ============================================================
package notification

import "sync"

// EventPublisher is the Subject — event sources call Publish; every
// registered Observer gets notified, fully decoupled from how many
// observers exist or what any of them do.
type EventPublisher struct {
	mu        sync.Mutex
	observers []Observer
}

func NewEventPublisher() *EventPublisher {
	return &EventPublisher{}
}

func (p *EventPublisher) Subscribe(o Observer) {
	p.mu.Lock()
	defer p.mu.Unlock()
	p.observers = append(p.observers, o)
}

// Publish copies the observer list under lock, then releases the
// lock BEFORE calling any observer — this avoids holding the lock
// during potentially slow (or reentrant, e.g. an observer that
// calls Subscribe/Publish itself) callbacks.
func (p *EventPublisher) Publish(event Event) {
	p.mu.Lock()
	observersCopy := append([]Observer{}, p.observers...)
	p.mu.Unlock()

	for _, o := range observersCopy {
		o.OnEvent(event)
	}
}
```

```go
// ============================================================
// FILE: filter.go
// ============================================================
package notification

// DeliveryFilter decides whether a notification should actually be
// delivered right now (e.g. do-not-disturb hours, mute settings).
type DeliveryFilter interface {
	ShouldDeliver(event Event) bool
}

// FilterChain: an ordered slice, not a linked chain of objects — see
// Step 4 for why that's the more idiomatic Go expression of the
// same Chain of Responsibility intent. The first filter to reject
// stops evaluation.
type FilterChain struct {
	filters []DeliveryFilter
}

func NewFilterChain(filters ...DeliveryFilter) *FilterChain {
	return &FilterChain{filters: filters}
}

func (c *FilterChain) ShouldDeliver(event Event) bool {
	for _, f := range c.filters {
		if !f.ShouldDeliver(event) {
			return false
		}
	}
	return true
}
```

```go
// ============================================================
// FILE: filters_impl.go
// ============================================================
package notification

import "time"

// DoNotDisturbFilter blocks delivery during configured quiet hours.
type DoNotDisturbFilter struct {
	quietStart, quietEnd int
	now                  func() time.Time // injected for testability
}

func NewDoNotDisturbFilter(quietStart, quietEnd int) *DoNotDisturbFilter {
	return &DoNotDisturbFilter{quietStart: quietStart, quietEnd: quietEnd, now: time.Now}
}

func (f *DoNotDisturbFilter) ShouldDeliver(event Event) bool {
	hour := f.now().Hour()
	if f.quietStart < f.quietEnd {
		return !(hour >= f.quietStart && hour < f.quietEnd)
	}
	// Quiet hours span midnight, e.g. 22 -> 7.
	return !(hour >= f.quietStart || hour < f.quietEnd)
}

// MuteFilter blocks delivery for event types the user has muted.
type MuteFilter struct {
	mutedTypes map[EventType]bool
}

func NewMuteFilter(mutedTypes ...EventType) *MuteFilter {
	m := make(map[EventType]bool)
	for _, t := range mutedTypes {
		m[t] = true
	}
	return &MuteFilter{mutedTypes: m}
}

func (f *MuteFilter) ShouldDeliver(event Event) bool {
	return !f.mutedTypes[event.Type]
}
```

```go
// ============================================================
// FILE: channel.go
// ============================================================
package notification

import "fmt"

type Channel interface {
	Send(event Event) error
}

type ConsoleChannel struct{}

func (c *ConsoleChannel) Send(event Event) error {
	fmt.Printf("[notify %s] user=%s payload=%v\n", event.Type, event.UserID, event.Payload)
	return nil
}
```

```go
// ============================================================
// FILE: channel_observer.go
// ============================================================
package notification

import "fmt"

// ChannelObserver is a concrete Observer delivering matching events
// to one channel, after passing them through its filter chain —
// composing Observer and the filter chain together.
type ChannelObserver struct {
	channel Channel
	filters *FilterChain
}

func NewChannelObserver(channel Channel, filters *FilterChain) *ChannelObserver {
	return &ChannelObserver{channel: channel, filters: filters}
}

func (o *ChannelObserver) OnEvent(event Event) {
	if !o.filters.ShouldDeliver(event) {
		return
	}
	if err := o.channel.Send(event); err != nil {
		fmt.Printf("failed to send notification: %v\n", err)
	}
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"

	notification "example.com/notification"
)

func main() {
	publisher := notification.NewEventPublisher()

	filters := notification.NewFilterChain(
		notification.NewMuteFilter(notification.PriceDrop),
	)
	observer := notification.NewChannelObserver(&notification.ConsoleChannel{}, filters)
	publisher.Subscribe(observer)

	publisher.Publish(notification.Event{
		Type: notification.NewComment, UserID: "user-1",
		Payload: map[string]string{"comment": "Nice post!"},
	})
	publisher.Publish(notification.Event{
		Type: notification.PriceDrop, UserID: "user-1",
		Payload: map[string]string{"item": "Shoes"},
	})

	fmt.Println("Done — PriceDrop should have been muted and not printed above.")
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "Why copy the observer slice under lock instead of just iterating `p.observers` directly?"
> If `Publish` iterated the live slice while holding the lock, a slow observer callback would hold up every other goroutine trying to `Subscribe` for its entire duration — and if any observer's `OnEvent` tried to call `Subscribe` or `Publish` itself, it would deadlock against the still-held lock. Copying under lock, then releasing before iterating, avoids both problems.

> [!quote]- "How would you add a new filter — say, rate-limiting notifications per user to at most 10/hour?"
> One new struct implementing `DeliveryFilter`, added to the `NewFilterChain(...)` call. Zero changes to `EventPublisher`, `ChannelObserver`, or any existing filter.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/07 - Design a Logger Framework/Design a Logger Framework|Design a Logger Framework]] · [[HLD/04 - Design a Notification Service/Design a Notification Service|HLD version — Design a Notification Service]]*
