---
title: "Design a Food Delivery System"
aliases: [Design a Food Delivery System, Zomato LLD, Swiggy LLD]
tags: [system-design, lld, case-study, golang, design-patterns, state-pattern, strategy-pattern]
status: reference-quality
---

# Design a Food Delivery System

> [!abstract] What you'll be able to do after this chapter
> Recognize that delivery-partner assignment is structurally the exact same Strategy shape as elevator dispatch, reuse that same shape a SECOND time for promo/discount pricing, and wire State and Observer together in one object — real-time order tracking driven directly off state transitions, not a bolted-on side system.

> [!info] Distinct from the HLD version
> [[HLD/16 - Design a Food Delivery System/Design a Food Delivery System|The HLD chapter]] covers geospatial restaurant discovery, two-leg delivery-partner matching at scale, and continuous surge pricing. This chapter is the in-process order object model — lifecycle, discount pricing, and live status notification.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a food delivery system — a customer places an order (with a promo code applied), it's matched with a delivery partner, its status is tracked in real time through preparation, pickup, and delivery."

## Step 2 — Requirement clarification

Order lifecycle: placed → preparing → ready → picked up → delivered. Cancellation rules depend on current state (can't cancel after pickup). Extensible partner-assignment logic. **Extensible discount/promo pricing** applied to the order subtotal. **Real-time status updates** pushed to the customer's app as the order progresses — not polled.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> Three patterns live in this chapter — build them one at a time, in the order the requirements naturally surface them.
>
> **Checkpoint 1 (~6-8 min) — order lifecycle, string status, flat price, no assignment.**
> ```go
> type Order struct {
>     status   string // "PLACED", "PREPARING", etc.
>     subtotal int64
> }
>
> func (o *Order) Prepare() { o.status = "PREPARING" }
> func (o *Order) FinalAmount() int64 { return o.subtotal } // no discounts yet
> ```
> **Pattern used: none.** Place → prepare → deliver, working end to end, before any pattern enters.
>
> **Checkpoint 2 (~8 min) — refactor lifecycle into State (kept brief — reused pattern, don't over-explain).**
> ```go
> type OrderState interface {
>     Name() string
>     Prepare(o *Order)
>     PickUp(o *Order)
>     Cancel(o *Order) error
> }
>
> type PreparingState struct{}
>
> func (s *PreparingState) Name() string     { return "PREPARING" }
> func (s *PreparingState) Prepare(o *Order) {}
> func (s *PreparingState) PickUp(o *Order)  { o.setState(&PickedUpState{}) }
> func (s *PreparingState) Cancel(o *Order) error { return nil } // restaurant absorbs the cost
>
> type PickedUpState struct{}
>
> func (s *PickedUpState) Name() string          { return "PICKED_UP" }
> func (s *PickedUpState) Prepare(o *Order)      {}
> func (s *PickedUpState) PickUp(o *Order)       {}
> func (s *PickedUpState) Cancel(o *Order) error { return ErrCannotCancelAfterPickup }
> ```
> **Pattern used: State**, same skeleton as Vending Machine/Elevator/ATM. `PickedUpState.Cancel` returns an error, `PreparingState.Cancel` doesn't — before pickup, cancelling only wastes prep effort; after, a partner is already en route. `PlacedState`/`ReadyState`/`DeliveredState` are the same shape again (Step 5).
>
> **Checkpoint 3 (~8 min) — partner assignment via Strategy, fully implemented. Recognize the shape before writing it.**
> ```go
> // AssignmentStrategy — structurally the SAME shape as the Elevator
> // chapter's SchedulingStrategy: match a request to the nearest
> // available resource. Swap "elevator" for "delivery partner."
> type AssignmentStrategy interface {
>     Assign(partners []*DeliveryPartner) *DeliveryPartner
> }
>
> type NearestPartnerStrategy struct{}
>
> func (s NearestPartnerStrategy) Assign(partners []*DeliveryPartner) *DeliveryPartner {
>     var best *DeliveryPartner
>     for _, p := range partners {
>         if best == nil || p.CurrentDistanceFromRestaurant < best.CurrentDistanceFromRestaurant {
>             best = p
>         }
>     }
>     return best
> }
> ```
> **Pattern used: Strategy (2nd application in this chapter).** Say the recognition out loud — that transfer *is* the skill being tested, not re-deriving Strategy from scratch a second time.
>
> **Checkpoint 4 (~8 min) — a SECOND, independent Strategy for pricing, fully implemented.**
> ```go
> // PricingStrategy — the SAME shape as Parking Lot's PricingStrategy
> // and Splitwise's SplitStrategy. Independent of AssignmentStrategy —
> // that's WHY they're two separate interfaces, not one bloated one.
> type PricingStrategy interface {
>     FinalAmount(subtotal int64) int64
> }
>
> type PercentageDiscountStrategy struct {
>     Percentage       float64
>     MaxDiscountCents int64
> }
>
> func (s PercentageDiscountStrategy) FinalAmount(subtotal int64) int64 {
>     discount := int64(float64(subtotal) * s.Percentage / 100)
>     if discount > s.MaxDiscountCents {
>         discount = s.MaxDiscountCents
>     }
>     final := subtotal - discount
>     if final < 0 {
>         return 0
>     }
>     return final
> }
> ```
> **Pattern used: Strategy (independent 2nd instance).** `NoDiscountStrategy`/`FlatDiscountStrategy` are the same shape again (Step 5).
>
> **Checkpoint 5 (remaining time, or if asked) — wire Observer directly into `setState`.**
> ```go
> // setState is the SINGLE funnel every transition passes through —
> // notification lives here, not duplicated across every state method.
> func (o *Order) setState(s OrderState) {
>     o.currentState = s
>     for _, obs := range o.observers {
>         obs.OnStatusChanged(o.ID, s.Name())
>     }
> }
> ```
> **Pattern used: Observer, composed WITH State** — the strongest "aha" in this chapter: real-time tracking isn't a bolted-on side system, it's the state machine's own transition point doing double duty.
>
> **If you're short on time:** stop after Checkpoint 3. Working lifecycle (State) plus partner assignment (Strategy #1) is a complete, demoable core — describe pricing and the Observer wiring verbally as the next two layers.

## Step 3 — The bad first draft (kept brief)

Monolithic `Order` struct with a string status field and inline `if/else` state checks — the same shape as [[LLD/02 - Design a Vending Machine/Design a Vending Machine|Vending Machine]] and [[LLD/11 - Design an ATM/Design an ATM|ATM]] by now — plus hardcoded "assign nearest partner" logic with zero abstraction.

## Step 4 — Refactor: State for the lifecycle, Strategy for assignment

State handling is kept brief here — you've seen this exact skeleton three times now. The genuinely new content is partner assignment:

> [!tip] Recognize the repetition — that recognition IS the skill
> `AssignmentStrategy` below is **structurally identical** to [[LLD/03 - Design an Elevator System/Design an Elevator System|the Elevator chapter's `SchedulingStrategy`]] — matching a request to the best available resource by proximity. Just swap "elevator" for "delivery partner." Once Strategy clicks for one "which resource handles this" problem, most others reduce to the exact same shape — that transfer is the actual point of learning the pattern once, properly, rather than memorizing it per-problem.

**A second, independent Strategy for pricing.** `PricingStrategy` computes the final payable amount from an order's subtotal — the identical shape to [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Parking Lot's `PricingStrategy`]] and [[LLD/05 - Design Splitwise/Design Splitwise|Splitwise's `SplitStrategy`]], applied here to promo codes and discount campaigns. Two completely unrelated Strategy interfaces coexist on the same `Order` — assignment and pricing vary **independently** of each other, which is exactly why they're two separate abstractions instead of one bloated interface trying to do both.

**Observer, wired directly into the State transitions.** Rather than a separate system that polls order status, `Order.setState` — the single funnel every state transition passes through, regardless of which state triggered it — is where observer notification actually lives. This is the first chapter in this handbook where State and Observer are **composed together** in one object, not just placed side by side as independent systems.

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: order_state.go
// ============================================================
package fooddelivery

type OrderState interface {
	Name() string // reported to observers on every transition
	Prepare(o *Order)
	MarkReady(o *Order)
	PickUp(o *Order)
	Deliver(o *Order)
	Cancel(o *Order) error
}

type PlacedState struct{}

func (s *PlacedState) Name() string          { return "PLACED" }
func (s *PlacedState) Prepare(o *Order)      { o.setState(&PreparingState{}) }
func (s *PlacedState) MarkReady(o *Order)    {}
func (s *PlacedState) PickUp(o *Order)       {}
func (s *PlacedState) Deliver(o *Order)      {}
func (s *PlacedState) Cancel(o *Order) error { return nil }

type PreparingState struct{}

func (s *PreparingState) Name() string          { return "PREPARING" }
func (s *PreparingState) Prepare(o *Order)      {}
func (s *PreparingState) MarkReady(o *Order)    { o.setState(&ReadyState{}) }
func (s *PreparingState) PickUp(o *Order)       {}
func (s *PreparingState) Deliver(o *Order)      {}
func (s *PreparingState) Cancel(o *Order) error { return nil } // restaurant absorbs the cost

type ReadyState struct{}

func (s *ReadyState) Name() string          { return "READY" }
func (s *ReadyState) Prepare(o *Order)      {}
func (s *ReadyState) MarkReady(o *Order)    {}
func (s *ReadyState) PickUp(o *Order)       { o.setState(&PickedUpState{}) }
func (s *ReadyState) Deliver(o *Order)      {}
func (s *ReadyState) Cancel(o *Order) error { return nil }

type PickedUpState struct{}

func (s *PickedUpState) Name() string          { return "PICKED_UP" }
func (s *PickedUpState) Prepare(o *Order)      {}
func (s *PickedUpState) MarkReady(o *Order)    {}
func (s *PickedUpState) PickUp(o *Order)       {}
func (s *PickedUpState) Deliver(o *Order)      { o.setState(&DeliveredState{}) }
func (s *PickedUpState) Cancel(o *Order) error { return ErrCannotCancelAfterPickup }

type DeliveredState struct{}

func (s *DeliveredState) Name() string          { return "DELIVERED" }
func (s *DeliveredState) Prepare(o *Order)      {}
func (s *DeliveredState) MarkReady(o *Order)    {}
func (s *DeliveredState) PickUp(o *Order)       {}
func (s *DeliveredState) Deliver(o *Order)      {}
func (s *DeliveredState) Cancel(o *Order) error { return ErrCannotCancelAfterPickup }
```

```go
// ============================================================
// FILE: pricing_strategy.go
// ============================================================
package fooddelivery

// PricingStrategy computes the final payable amount from an order's
// subtotal — the same Strategy shape as Parking Lot's
// PricingStrategy and Splitwise's SplitStrategy, applied here to
// promo codes and discount campaigns. All amounts in cents.
type PricingStrategy interface {
	FinalAmount(subtotal int64) int64
}

// NoDiscountStrategy — the default, no promo applied.
type NoDiscountStrategy struct{}

func (s NoDiscountStrategy) FinalAmount(subtotal int64) int64 { return subtotal }

// FlatDiscountStrategy knocks a fixed amount off, never below zero.
type FlatDiscountStrategy struct {
	DiscountCents int64
}

func (s FlatDiscountStrategy) FinalAmount(subtotal int64) int64 {
	final := subtotal - s.DiscountCents
	if final < 0 {
		return 0
	}
	return final
}

// PercentageDiscountStrategy applies a percentage off, capped at a
// maximum — the real-world "20% off, up to $5" promo shape.
type PercentageDiscountStrategy struct {
	Percentage       float64 // e.g. 20 for 20%
	MaxDiscountCents int64
}

func (s PercentageDiscountStrategy) FinalAmount(subtotal int64) int64 {
	discount := int64(float64(subtotal) * s.Percentage / 100)
	if discount > s.MaxDiscountCents {
		discount = s.MaxDiscountCents
	}
	final := subtotal - discount
	if final < 0 {
		return 0
	}
	return final
}
```

```go
// ============================================================
// FILE: order_observer.go
// ============================================================
package fooddelivery

import "fmt"

// OrderObserver is notified on every status transition — the
// mechanism a customer's app uses for real-time order tracking,
// instead of polling for status.
type OrderObserver interface {
	OnStatusChanged(orderID string, status string)
}

type ConsoleOrderObserver struct{}

func (c *ConsoleOrderObserver) OnStatusChanged(orderID string, status string) {
	fmt.Printf("[order %s] status: %s\n", orderID, status)
}
```

```go
// ============================================================
// FILE: order.go
// ============================================================
package fooddelivery

import "errors"

var ErrCannotCancelAfterPickup = errors.New("fooddelivery: cannot cancel after pickup")

type Order struct {
	ID              string
	currentState    OrderState
	AssignedPartner *DeliveryPartner
	Subtotal        int64
	Pricing         PricingStrategy
	observers       []OrderObserver
}

func NewOrder(id string, subtotal int64, pricing PricingStrategy) *Order {
	return &Order{ID: id, currentState: &PlacedState{}, Subtotal: subtotal, Pricing: pricing}
}

func (o *Order) Subscribe(observer OrderObserver) {
	o.observers = append(o.observers, observer)
}

// setState is the SINGLE funnel every transition passes through,
// regardless of which state triggered it — exactly why notification
// lives here instead of being duplicated in every state's method.
func (o *Order) setState(s OrderState) {
	o.currentState = s
	for _, obs := range o.observers {
		obs.OnStatusChanged(o.ID, s.Name())
	}
}

func (o *Order) FinalAmount() int64 {
	return o.Pricing.FinalAmount(o.Subtotal)
}

func (o *Order) Prepare()      { o.currentState.Prepare(o) }
func (o *Order) MarkReady()    { o.currentState.MarkReady(o) }
func (o *Order) PickUp()       { o.currentState.PickUp(o) }
func (o *Order) Deliver()      { o.currentState.Deliver(o) }
func (o *Order) Cancel() error { return o.currentState.Cancel(o) }
```

```go
// ============================================================
// FILE: delivery_partner.go
// ============================================================
package fooddelivery

type DeliveryPartner struct {
	ID                            string
	CurrentDistanceFromRestaurant float64 // km — simplified from real lat/long
}
```

```go
// ============================================================
// FILE: assignment_strategy.go
// ============================================================
package fooddelivery

// AssignmentStrategy is structurally the SAME shape as the Elevator
// chapter's SchedulingStrategy — see Step 4.
type AssignmentStrategy interface {
	Assign(partners []*DeliveryPartner) *DeliveryPartner
}

type NearestPartnerStrategy struct{}

func (s NearestPartnerStrategy) Assign(partners []*DeliveryPartner) *DeliveryPartner {
	var best *DeliveryPartner
	for _, p := range partners {
		if best == nil || p.CurrentDistanceFromRestaurant < best.CurrentDistanceFromRestaurant {
			best = p
		}
	}
	return best
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"

	fooddelivery "example.com/fooddelivery"
)

func main() {
	// $30.00 subtotal, "20% off up to $5" promo applied.
	pricing := fooddelivery.PercentageDiscountStrategy{Percentage: 20, MaxDiscountCents: 500}
	order := fooddelivery.NewOrder("order-1", 3000, pricing)
	order.Subscribe(&fooddelivery.ConsoleOrderObserver{})

	fmt.Println("Final amount after discount:", order.FinalAmount(), "cents") // 2500 — capped, not 2400

	partners := []*fooddelivery.DeliveryPartner{
		{ID: "partner-A", CurrentDistanceFromRestaurant: 2.5},
		{ID: "partner-B", CurrentDistanceFromRestaurant: 0.8},
	}
	strategy := fooddelivery.NearestPartnerStrategy{}
	order.AssignedPartner = strategy.Assign(partners)
	fmt.Println("Assigned partner:", order.AssignedPartner.ID)

	order.Prepare()   // observer fires: PREPARING
	order.MarkReady() // observer fires: READY
	order.PickUp()    // observer fires: PICKED_UP

	if err := order.Cancel(); err != nil {
		fmt.Println("cancel correctly rejected:", err)
	}

	order.Deliver() // observer fires: DELIVERED
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "How would you extend `AssignmentStrategy` to also consider partner rating, not just distance?"
> A new struct implementing `AssignmentStrategy` — e.g. `WeightedAssignmentStrategy` combining distance and rating into one score — swapped in at construction time. `Order` never needs to change; it never knew which concrete strategy it was using in the first place.

> [!quote]- "Why does `PreparingState.Cancel` still allow cancellation, but `PickedUpState.Cancel` doesn't?"
> This encodes a real business rule directly into the type system: before pickup, cancelling only wastes food-prep effort (the restaurant absorbs that cost) — after pickup, a delivery partner is physically already en route, making cancellation operationally much costlier and effectively disallowed. The state machine isn't just tracking status, it's enforcing exactly which real-world actions remain valid at each point.

> [!quote]- "Why does `setState` notify observers instead of each individual state method calling notification itself?"
> `setState` is the one place every transition passes through no matter which state or which action triggered it — putting notification there means it's structurally impossible to add a new transition that forgets to notify. Duplicating the notification call across all five states' methods would work today, but silently break the instant a sixth state gets added and someone forgets to wire it in.

> [!quote]- "How would you add a stacked promo — a percentage discount AND a flat referral credit, both applied to the same order?"
> Compose two `PricingStrategy` values with a small wrapper, e.g. `ChainedPricingStrategy{ First, Second PricingStrategy }` whose `FinalAmount` feeds the subtotal through `First`, then feeds *that* result through `Second` — `Order` still only ever calls one `PricingStrategy.FinalAmount`, unaware it's actually two strategies chained together underneath.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/16 - Design a Food Delivery System/Design a Food Delivery System|HLD version]] · [[LLD/03 - Design an Elevator System/Design an Elevator System|Design an Elevator System]] · [[LLD/02 - Design a Vending Machine/Design a Vending Machine|Design a Vending Machine]] · [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]]*
