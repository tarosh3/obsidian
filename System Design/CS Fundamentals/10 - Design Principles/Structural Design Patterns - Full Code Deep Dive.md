---
title: "Structural Design Patterns — Full Code Deep Dive"
aliases: [Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy]
tags: [system-design, cs-fundamentals, design-principles, design-patterns, go, structural]
status: reference-quality
---

# Structural Design Patterns — Full Code Deep Dive

> [!abstract] What you'll be able to do after this chapter
> For every structural pattern except Composite (which already has full LLD chapters), see the exact bad code, feel exactly why it breaks, and see the exact refactor that fixes it.

> [!info] Companion chapter
> Part of a 3-chapter set with [[CS Fundamentals/10 - Design Principles/Creational Design Patterns - Full Code Deep Dive|Creational]] and [[CS Fundamentals/10 - Design Principles/Behavioral Design Patterns - Full Code Deep Dive|Behavioral]] Design Patterns. See [[CS Fundamentals/10 - Design Principles/Design Patterns Cheat Sheet|the Design Patterns Cheat Sheet]] for the one-paragraph recognition version of all 23.

---

## Composite — recap only

Already has two full case studies: [[LLD/09 - Design Chess/Design Chess|Design Chess]] (polymorphic move logic) and [[LLD/16 - Design a File System/Design a File System|Design a File System]] (the canonical files-and-directories case). **Layman:** a folder in a file system — "how big is this?" works identically whether it's one file or a thousand nested folders. Nothing to add here that those chapters don't already cover in full.

---

## Adapter

> [!example] Layman
> A physical travel plug adapter — doesn't change your laptop charger or the foreign wall socket, just sits between them so two incompatible shapes connect.

**When to use:** integrating a third-party SDK or legacy code whose interface doesn't match what your code expects, and you can't or shouldn't modify the original.

```go
// BEFORE — business logic directly coupled to one provider's specific SDK shape
func CheckoutBad(stripeClient *stripe.Client, amountCents int) error {
	_, err := stripeClient.Charges.New(&stripe.ChargeParams{Amount: amountCents})
	return err // switching providers means rewriting every call site that touches this
}
```

```go
// AFTER — Adapter isolates each provider's shape behind one consistent interface
type PaymentGateway interface {
	Charge(amountCents int) error
}

type StripeAdapter struct{ client *stripe.Client }
func (s *StripeAdapter) Charge(amountCents int) error {
	_, err := s.client.Charges.New(&stripe.ChargeParams{Amount: amountCents})
	return err
}

type RazorpayAdapter struct{ client *razorpay.Client }
func (r *RazorpayAdapter) Charge(amountCents int) error {
	_, err := r.client.Payments.Create(razorpay.PaymentRequest{AmountPaise: amountCents})
	return err
}

func Checkout(gw PaymentGateway, amountCents int) error {
	return gw.Charge(amountCents) // works with ANY provider behind the interface
}
```

> [!warning] Tradeoffs
> An extra indirection layer for a provider you'll genuinely never swap is pure overhead. Earns its keep specifically at integration boundaries with volatile or external dependencies — exactly [[HLD/17 - Design a Payment System/Design a Payment System|the Payment System chapter's]] provider-agnostic design.

---

## Bridge

> [!example] Layman
> A universal TV remote whose buttons work identically no matter which TV brand's internal signal protocol is on the other end.

**When to use:** two independent dimensions of variation exist (notification urgency × delivery channel) and you don't want `N × M` concrete types for every combination.

```go
// BEFORE — combinatorial explosion: one type per (kind x channel) pair
type UrgentEmailNotification struct{}
type UrgentSMSNotification struct{}
type ReminderEmailNotification struct{}
type ReminderSMSNotification struct{} // 2x2 = 4 types, grows MULTIPLICATIVELY with each new channel
```

```go
// AFTER — Bridge: notification kind and delivery channel vary independently
type Sender interface{ Send(msg string) error }

type EmailSender struct{}
func (EmailSender) Send(msg string) error { fmt.Println("email:", msg); return nil }

type SMSSender struct{}
func (SMSSender) Send(msg string) error { fmt.Println("sms:", msg); return nil }

type Notifier struct {
	sender Sender // the "bridge" to whichever implementation is plugged in
}

func (n *Notifier) NotifyUrgent(msg string) error   { return n.sender.Send("[URGENT] " + msg) }
func (n *Notifier) NotifyReminder(msg string) error { return n.sender.Send("[reminder] " + msg) }

n := &Notifier{sender: SMSSender{}}
n.NotifyUrgent("server down") // 2 kinds x N senders, never 2xN TYPES
```

> [!warning] Tradeoffs
> Adds indirection for a problem that only exists once there are genuinely 2+ independent dimensions of variation — premature if there's only ever going to be one channel.

---

## Decorator

> [!example] Layman
> Stacking pizza toppings — each topping wraps the base pizza with an added layer, any combination, without a separate pre-made type for every possible combo.

**When to use:** combinable, optional behavior layered on a base object — the alternative (subclassing every combination) explodes combinatorially.

```go
// BEFORE — one type per combination, explodes as options grow
type Coffee struct{}
type CoffeeWithMilk struct{}
type CoffeeWithMilkAndSugar struct{}
type CoffeeWithMilkSugarAndExtraShot struct{} // and so on, combinatorially
```

```go
// AFTER — Decorator: each option wraps the previous, stackable in any combination
type Beverage interface {
	Cost() float64
	Description() string
}

type Coffee struct{}
func (Coffee) Cost() float64       { return 2.00 }
func (Coffee) Description() string { return "Coffee" }

type MilkDecorator struct{ Beverage }
func (m MilkDecorator) Cost() float64       { return m.Beverage.Cost() + 0.50 }
func (m MilkDecorator) Description() string { return m.Beverage.Description() + " + Milk" }

type SugarDecorator struct{ Beverage }
func (s SugarDecorator) Cost() float64       { return s.Beverage.Cost() + 0.25 }
func (s SugarDecorator) Description() string { return s.Beverage.Description() + " + Sugar" }

order := SugarDecorator{MilkDecorator{Coffee{}}}
fmt.Println(order.Description(), order.Cost()) // "Coffee + Milk + Sugar", 2.75
// any combination, any order, ZERO new types needed
```

> [!warning] Tradeoffs
> A long decorator chain becomes hard to read in a stack trace — each layer adds a frame. Real, production use: [[LLD/13 - Design a Food Delivery System/Design a Food Delivery System|the Food Delivery chapter's]] `ChainedPricingStrategy` stacking multiple promos this exact way.

---

## Facade

> [!example] Layman
> A hotel concierge — ask for "a nice dinner reservation," and the concierge handles the restaurant, transport, and table confirmation without you touching those systems yourself.

**When to use:** a subsystem has many components with a complex internal protocol, and most callers only need a small, simple slice of it exposed.

```go
// BEFORE — caller must know and orchestrate every subsystem directly
func PlaceOrderBad(inv *InventoryService, pay *PaymentService, ship *ShippingService, notif *NotificationService, item, userID string) error {
	if err := inv.Reserve(item); err != nil { return err }
	if err := pay.Charge(userID, 999); err != nil { return err }
	if err := ship.Schedule(item, userID); err != nil { return err }
	return notif.Send(userID, "Order placed")
}
```

```go
// AFTER — Facade hides the orchestration behind one simple call
type OrderFacade struct {
	inventory    *InventoryService
	payment      *PaymentService
	shipping     *ShippingService
	notification *NotificationService
}

func (f *OrderFacade) PlaceOrder(item, userID string) error {
	if err := f.inventory.Reserve(item); err != nil { return err }
	if err := f.payment.Charge(userID, 999); err != nil { return err }
	if err := f.shipping.Schedule(item, userID); err != nil { return err }
	return f.notification.Send(userID, "Order placed")
}

// caller now just does:
facade.PlaceOrder("widget", "user-42")
```

> [!warning] Tradeoffs
> Doesn't remove complexity, relocates it into the Facade. Good when most callers want the simple path and don't need per-subsystem control. **System-design-level example:** [[CS Fundamentals/02 - Networking/API Gateway|API Gateway]] is architecturally a Facade applied at the whole-system level.

---

## Flyweight

> [!example] Layman
> A print shop keeping one master stencil for a common design, reused to stamp thousands of copies, instead of hand-drawing it fresh each time.

**When to use:** a huge number of similar objects would otherwise each redundantly store identical, large, immutable data.

```go
// BEFORE — every chess piece duplicates its own copy of shared move-rule data
type Piece struct {
	Color     string
	Position  string
	MoveRules []Move // heavy, IDENTICAL for every Bishop, duplicated per instance
}
```

```go
// AFTER — Flyweight: shared, immutable data reused across every instance of that type
type MoveRuleSet struct{ Rules []Move } // heavy, built ONCE

var bishopRules = &MoveRuleSet{Rules: buildBishopRules()} // shared across every Bishop
var rookRules = &MoveRuleSet{Rules: buildRookRules()}

type Piece struct {
	Color    string
	Position string
	Rules    *MoveRuleSet // just a pointer — zero duplication
}

whiteBishop := Piece{Color: "white", Position: "c1", Rules: bishopRules}
blackBishop := Piece{Color: "black", Position: "c8", Rules: bishopRules} // same pointer
```

> [!warning] Tradeoffs
> Only worth it when the shared data is genuinely immutable *and* genuinely large relative to instance count — adds indirection for a memory saving that doesn't matter at small scale. A natural extension of [[LLD/09 - Design Chess/Design Chess|the Chess chapter's]] shared move-logic design.

---

## Proxy

> [!example] Layman
> A receptionist who screens calls before connecting you — same eventual access, but with a controlled gate in front of it.

**When to use:** adding access control, lazy initialization, logging, or caching around an object without changing the object itself or its callers.

```go
// BEFORE — every caller hits the expensive, real resource directly, no control
type RealImageLoader struct{}
func (RealImageLoader) Load(path string) []byte {
	return readFullResolutionFromDisk(path) // expensive, runs EVERY single call
}
```

```go
// AFTER — Proxy adds caching without changing the interface callers depend on
type ImageLoader interface{ Load(path string) []byte }

type CachingImageProxy struct {
	real  RealImageLoader
	cache map[string][]byte
}

func (p *CachingImageProxy) Load(path string) []byte {
	if data, ok := p.cache[path]; ok {
		return data // cached — real loader never touched
	}
	data := p.real.Load(path)
	p.cache[path] = data
	return data
}

var loader ImageLoader = &CachingImageProxy{cache: map[string][]byte{}}
loader.Load("photo.jpg") // expensive, first time
loader.Load("photo.jpg") // cheap, served from proxy's own cache
```

> [!warning] Tradeoffs
> A caching proxy needs the same invalidation discipline as any cache (per [[CS Fundamentals/04 - Caching/Caching Strategies|Caching Strategies]]). **System-design-level example:** a cache-aside layer is architecturally a **caching proxy**; [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|a rate limiter in front of a service]] is architecturally a **protection proxy**.

---

## Interview Q&A

> [!info] Leveled by seniority
> **Beginner:** "What's the core difference between Decorator and Proxy — both wrap an object behind the same interface?" — Decorator *adds* new behavior/responsibility; Proxy *controls access* to existing behavior without adding new responsibility. **Intermediate:** "When would you choose Bridge over just subclassing?" — when there are genuinely 2+ independent dimensions of variation; subclassing alone forces `N x M` types, Bridge keeps them independent. **Senior:** "A Decorator chain has grown to 6 layers deep and a bug report is hard to trace — assess it." — expects recognizing the real, named tradeoff (stack-trace depth vs. combinability) rather than treating Decorator as free composability with no cost. **Staff:** "Design the integration layer for a system needing to support 5 different third-party shipping providers with different APIs." — expects Adapter at the integration boundary, one consistent `ShippingProvider` interface, isolating each provider's SDK-specific shape from the rest of the system. **Architect:** "How do you decide whether a Facade is hiding complexity well or just relocating it badly?" — expects checking whether callers who *do* need finer-grained control still have an escape hatch to the underlying subsystems, or whether the Facade has become a leaky, all-or-nothing bottleneck.

## Summary / Cheat Sheet

- **Composite:** full chapters — see [[LLD/09 - Design Chess/Design Chess|Chess]] and [[LLD/16 - Design a File System/Design a File System|File System]].
- **Adapter:** wraps an incompatible interface to match what your code expects, without touching the original.
- **Bridge:** two independent dimensions of variation, kept independent — avoids `N x M` type explosion.
- **Decorator:** stackable, combinable behavior wrapped around a base object — avoids combinatorial subclassing.
- **Facade:** one simple entry point in front of a complex subsystem — relocates complexity, doesn't remove it.
- **Flyweight:** shared, immutable data reused across many instances — a real memory optimization, only worth it at scale.
- **Proxy:** a controlled stand-in for lazy-loading, caching, or access control — same interface, different behavior underneath.

---
*Related: [[CS Fundamentals/00 - Learning Path|CS Fundamentals Learning Path]] · [[CS Fundamentals/10 - Design Principles/Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] · [[CS Fundamentals/10 - Design Principles/Creational Design Patterns - Full Code Deep Dive|Creational Design Patterns]] · [[CS Fundamentals/10 - Design Principles/Behavioral Design Patterns - Full Code Deep Dive|Behavioral Design Patterns]]*
