---
title: "Structural Design Patterns — Full Code Deep Dive"
aliases: [Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy]
tags: [system-design, cs-fundamentals, design-principles, design-patterns, go, structural]
status: reference-quality
---

# Structural Design Patterns — Full Code Deep Dive

> [!abstract] What you'll be able to do after this chapter
> For every one of the 7 structural patterns, see a complete, runnable "without the pattern" program, the exact same problem solved "with the pattern," and a one-line trick for telling this pattern apart from the one it's most often confused with.

> [!info] Companion chapter
> Part of a 3-chapter set with [[CS Fundamentals/10 - Design Principles/03 - Creational Design Patterns - Full Code Deep Dive|Creational]] and [[CS Fundamentals/10 - Design Principles/05 - Behavioral Design Patterns - Full Code Deep Dive|Behavioral]] Design Patterns. See [[CS Fundamentals/10 - Design Principles/02 - Design Patterns Cheat Sheet|the Design Patterns Cheat Sheet]] for all 23 at a glance. Composite also has two full systems built around it — [[LLD/09 - Design Chess/Design Chess|Design Chess]] and [[LLD/16 - Design a File System/Design a File System|Design a File System]] — if you want the deeper, multi-requirement version.

---

## Adapter

> [!example] Layman
> A physical travel plug adapter — doesn't change your laptop charger or the foreign wall socket, just sits between them so two incompatible shapes connect.

**Trigger:** "make this 3rd-party or legacy interface match what my code expects"
**When to use:** integrating a third-party SDK or legacy code whose interface doesn't match what your code expects, and you can't or shouldn't modify the original.

> [!tip] Identification trick
> You control the interface you *want*; you do **not** control the interface you're bridging to (a 3rd-party SDK, legacy code). If you control both sides, you don't need Adapter — just fix the signature directly.

**Without the pattern:**

```go
package main

import "fmt"

// Pretend this is a third-party SDK we don't control
type StripeClient struct{}

func (StripeClient) CreateCharge(amountCents int) error {
	fmt.Println("stripe charged", amountCents, "cents")
	return nil
}

func CheckoutBad(stripeClient *StripeClient, amountCents int) error {
	return stripeClient.CreateCharge(amountCents)
	// switching providers means rewriting every call site that touches this
}

func main() {
	client := &StripeClient{}
	_ = CheckoutBad(client, 999)
}

// Output:
// stripe charged 999 cents
```

**With Adapter:**

```go
package main

import "fmt"

// Two third-party SDKs, each with a DIFFERENT shape — we don't control either
type StripeClient struct{}

func (StripeClient) CreateCharge(amountCents int) error {
	fmt.Println("stripe charged", amountCents, "cents")
	return nil
}

type RazorpayClient struct{}

func (RazorpayClient) MakePayment(amountPaise int) error {
	fmt.Println("razorpay paid", amountPaise, "paise")
	return nil
}

// Our own, consistent interface
type PaymentGateway interface {
	Charge(amountCents int) error
}

type StripeAdapter struct{ client *StripeClient }

func (s *StripeAdapter) Charge(amountCents int) error {
	return s.client.CreateCharge(amountCents)
}

type RazorpayAdapter struct{ client *RazorpayClient }

func (r *RazorpayAdapter) Charge(amountCents int) error {
	return r.client.MakePayment(amountCents) // adapts the DIFFERENT method name/shape
}

func Checkout(gw PaymentGateway, amountCents int) error {
	return gw.Charge(amountCents) // works with ANY provider behind the interface
}

func main() {
	var gw PaymentGateway = &StripeAdapter{client: &StripeClient{}}
	_ = Checkout(gw, 999)

	gw = &RazorpayAdapter{client: &RazorpayClient{}} // swap provider, Checkout() unchanged
	_ = Checkout(gw, 999)
}

// Output:
// stripe charged 999 cents
// razorpay paid 999 paise
```

> [!warning] Tradeoffs
> An extra indirection layer for a provider you'll genuinely never swap is pure overhead. Earns its keep specifically at integration boundaries with volatile or external dependencies — exactly [[HLD/17 - Design a Payment System/Design a Payment System|the Payment System chapter's]] provider-agnostic design.

---

## Bridge

> [!example] Layman
> A universal TV remote whose buttons work identically no matter which TV brand's internal signal protocol is on the other end.

**Trigger:** "two independent dimensions of variation are multiplying into a type explosion"
**When to use:** notification urgency × delivery channel, where you don't want `N × M` concrete types for every combination.

> [!tip] Identification trick
> Count the independent dimensions of variation. One dimension → Strategy. Two or more dimensions that would otherwise multiply into `N × M` types → Bridge.

**Without the pattern:**

```go
package main

import "fmt"

type UrgentEmailNotification struct{}

func (UrgentEmailNotification) Send(msg string) { fmt.Println("URGENT email:", msg) }

type UrgentSMSNotification struct{}

func (UrgentSMSNotification) Send(msg string) { fmt.Println("URGENT sms:", msg) }

type ReminderEmailNotification struct{}

func (ReminderEmailNotification) Send(msg string) { fmt.Println("reminder email:", msg) }

type ReminderSMSNotification struct{}

func (ReminderSMSNotification) Send(msg string) { fmt.Println("reminder sms:", msg) }

// 2x2 = 4 types, grows MULTIPLICATIVELY with every new channel (add push -> 6, add slack -> 8...)

func main() {
	n := UrgentSMSNotification{}
	n.Send("server down")
}

// Output:
// URGENT sms: server down
```

**With Bridge:**

```go
package main

import "fmt"

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

func main() {
	n := &Notifier{sender: SMSSender{}}
	_ = n.NotifyUrgent("server down") // 2 kinds x N senders, never 2xN TYPES

	n2 := &Notifier{sender: EmailSender{}}
	_ = n2.NotifyReminder("standup in 5 min")
}

// Output:
// sms: [URGENT] server down
// email: [reminder] standup in 5 min
```

> [!warning] Tradeoffs
> Adds indirection for a problem that only exists once there are genuinely 2+ independent dimensions of variation — premature if there's only ever going to be one channel.

---

## Composite

> [!example] Layman
> A folder in a file system — "how big is this?" works identically whether it's one file or a thousand nested folders.

**Trigger:** "treat a single object and a group of them through the exact same interface"
**When to use:** a tree of objects where "the whole" and "one piece" should behave identically to the caller.

> [!tip] Identification trick
> Ask: "does this need to work the same whether it's ONE item or a GROUP of items, recursively?" If the caller shouldn't have to special-case "is this a leaf or a branch" → Composite.

**Without the pattern:**

```go
package main

import "fmt"

type File struct {
	Name string
	Size int
}

type Folder struct {
	Name    string
	Files   []File
	Folders []Folder
}

// caller must special-case "is this a file or a folder" every time it needs a size
func TotalSizeBad(f Folder) int {
	total := 0
	for _, file := range f.Files {
		total += file.Size
	}
	for _, sub := range f.Folders {
		total += TotalSizeBad(sub) // recursion only works because we hardcoded the folder case
	}
	return total
}

func main() {
	root := Folder{
		Name:  "root",
		Files: []File{{"a.txt", 10}, {"b.txt", 20}},
		Folders: []Folder{
			{Name: "docs", Files: []File{{"c.txt", 5}}},
		},
	}
	fmt.Println(TotalSizeBad(root))
}

// Output:
// 35
```

**With Composite:**

```go
package main

import "fmt"

// File and Folder both implement the SAME interface
type FileSystemNode interface {
	Size() int
}

type File struct {
	Name string
	size int
}

func (f File) Size() int { return f.size }

type Folder struct {
	Name     string
	Children []FileSystemNode // files AND folders, held uniformly
}

func (f Folder) Size() int {
	total := 0
	for _, child := range f.Children {
		total += child.Size() // works identically whether child is a File or a Folder
	}
	return total
}

func main() {
	root := Folder{
		Name: "root",
		Children: []FileSystemNode{
			File{Name: "a.txt", size: 10},
			File{Name: "b.txt", size: 20},
			Folder{Name: "docs", Children: []FileSystemNode{
				File{Name: "c.txt", size: 5},
			}},
		},
	}
	fmt.Println(root.Size()) // caller never special-cases file vs folder
}

// Output:
// 35
```

> [!warning] Tradeoffs
> Every node type in the tree must implement the shared interface, even ones where an operation is meaningless (a `File.Add(child)` doesn't really make sense) — usually solved with a no-op or a panic on the leaf type, a real, minor wart worth naming honestly.

---

## Decorator

> [!example] Layman
> Stacking pizza toppings — each topping wraps the base pizza with an added layer, any combination, without a separate pre-made type for every possible combo.

**Trigger:** "stack optional behaviors in any combination"
**When to use:** combinable, optional behavior layered on a base object — the alternative (subclassing every combination) explodes combinatorially.

> [!tip] Identification trick vs. Proxy
> Does the wrapper **add new behavior/responsibility** (a new cost, a new description)? → Decorator. Does the wrapper **control access** to existing behavior without adding new responsibility (caching, lazy-load, permission check)? → Proxy, the next-but-one pattern below.

**Without the pattern:**

```go
package main

import "fmt"

type Coffee struct{}

func (Coffee) Cost() float64       { return 2.00 }
func (Coffee) Description() string { return "Coffee" }

type CoffeeWithMilk struct{}

func (CoffeeWithMilk) Cost() float64       { return 2.50 }
func (CoffeeWithMilk) Description() string { return "Coffee + Milk" }

type CoffeeWithMilkAndSugar struct{}

func (CoffeeWithMilkAndSugar) Cost() float64       { return 2.75 }
func (CoffeeWithMilkAndSugar) Description() string { return "Coffee + Milk + Sugar" }

// one TYPE per combination — CoffeeWithSugar, CoffeeWithMilkAndExtraShot... explodes

func main() {
	order := CoffeeWithMilkAndSugar{}
	fmt.Println(order.Description(), order.Cost())
}

// Output:
// Coffee + Milk + Sugar 2.75
```

**With Decorator:**

```go
package main

import "fmt"

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

func main() {
	order := SugarDecorator{MilkDecorator{Coffee{}}}
	fmt.Println(order.Description(), order.Cost()) // any combination, ZERO new types needed
}

// Output:
// Coffee + Milk + Sugar 2.75
```

> [!warning] Tradeoffs
> A long decorator chain becomes hard to read in a stack trace — each layer adds a frame. Real, production use: [[LLD/13 - Design a Food Delivery System/Design a Food Delivery System|the Food Delivery chapter's]] `ChainedPricingStrategy` stacking multiple promos this exact way.

---

## Facade

> [!example] Layman
> A hotel concierge — ask for "a nice dinner reservation," and the concierge handles the restaurant, transport, and table confirmation without you touching those systems yourself.

**Trigger:** "hide a complex, multi-step subsystem behind one simple call"
**When to use:** a subsystem has many components with a complex internal protocol, and most callers only need a small, simple slice of it exposed.

> [!tip] Identification trick vs. Adapter
> Adapter changes the **shape of one interface** so it matches what you expect. Facade simplifies **several different subsystems** into one easy entry point. If you're gluing 2 mismatched shapes → Adapter. If you're hiding an orchestration across 3+ subsystems → Facade.

**Without the pattern:**

```go
package main

import "fmt"

type InventoryService struct{}

func (InventoryService) Reserve(item string) error { fmt.Println("reserved", item); return nil }

type PaymentService struct{}

func (PaymentService) Charge(userID string, cents int) error {
	fmt.Println("charged", userID, cents)
	return nil
}

type ShippingService struct{}

func (ShippingService) Schedule(item, userID string) error {
	fmt.Println("shipping", item, "to", userID)
	return nil
}

type NotificationService struct{}

func (NotificationService) Send(userID, msg string) error {
	fmt.Println("notified", userID, ":", msg)
	return nil
}

func PlaceOrderBad(inv *InventoryService, pay *PaymentService, ship *ShippingService, notif *NotificationService, item, userID string) error {
	if err := inv.Reserve(item); err != nil {
		return err
	}
	if err := pay.Charge(userID, 999); err != nil {
		return err
	}
	if err := ship.Schedule(item, userID); err != nil {
		return err
	}
	return notif.Send(userID, "Order placed")
}

func main() {
	inv, pay, ship, notif := &InventoryService{}, &PaymentService{}, &ShippingService{}, &NotificationService{}
	_ = PlaceOrderBad(inv, pay, ship, notif, "widget", "user-42")
	// every caller must wire up and orchestrate all 4 services correctly, every time
}

// Output:
// reserved widget
// charged user-42 999
// shipping widget to user-42
// notified user-42 : Order placed
```

**With Facade:**

```go
package main

import "fmt"

type InventoryService struct{}

func (InventoryService) Reserve(item string) error { fmt.Println("reserved", item); return nil }

type PaymentService struct{}

func (PaymentService) Charge(userID string, cents int) error {
	fmt.Println("charged", userID, cents)
	return nil
}

type ShippingService struct{}

func (ShippingService) Schedule(item, userID string) error {
	fmt.Println("shipping", item, "to", userID)
	return nil
}

type NotificationService struct{}

func (NotificationService) Send(userID, msg string) error {
	fmt.Println("notified", userID, ":", msg)
	return nil
}

// Facade — hides the 4-service orchestration behind one simple call
type OrderFacade struct {
	inventory    *InventoryService
	payment      *PaymentService
	shipping     *ShippingService
	notification *NotificationService
}

func NewOrderFacade() *OrderFacade {
	return &OrderFacade{&InventoryService{}, &PaymentService{}, &ShippingService{}, &NotificationService{}}
}

func (f *OrderFacade) PlaceOrder(item, userID string) error {
	if err := f.inventory.Reserve(item); err != nil {
		return err
	}
	if err := f.payment.Charge(userID, 999); err != nil {
		return err
	}
	if err := f.shipping.Schedule(item, userID); err != nil {
		return err
	}
	return f.notification.Send(userID, "Order placed")
}

func main() {
	facade := NewOrderFacade()
	_ = facade.PlaceOrder("widget", "user-42") // caller just calls ONE method
}

// Output:
// reserved widget
// charged user-42 999
// shipping widget to user-42
// notified user-42 : Order placed
```

> [!warning] Tradeoffs
> Doesn't remove complexity, relocates it into the Facade. Good when most callers want the simple path and don't need per-subsystem control. **System-design-level example:** [[CS Fundamentals/02 - Networking/API Gateway|API Gateway]] is architecturally a Facade applied at the whole-system level.

---

## Flyweight

> [!example] Layman
> A print shop keeping one master stencil for a common design, reused to stamp thousands of copies, instead of hand-drawing it fresh each time.

**Trigger:** "share large, immutable data across many instances to save memory"
**When to use:** a huge number of similar objects would otherwise each redundantly store identical, large, immutable data.

> [!tip] Identification trick
> Ask: "am I about to create thousands of near-identical objects, and is most of their data actually IDENTICAL and IMMUTABLE across every instance?" If yes → Flyweight. If the shared thing is small, this pattern isn't worth its own indirection.

**Without the pattern:**

```go
package main

import "fmt"

type MoveRule string

type Piece struct {
	Color     string
	Position  string
	MoveRules []MoveRule // heavy, IDENTICAL for every Bishop, duplicated per instance
}

func buildBishopRules() []MoveRule {
	return []MoveRule{"diagonal-nw", "diagonal-ne", "diagonal-sw", "diagonal-se"} // pretend: large
}

func main() {
	whiteBishop := Piece{Color: "white", Position: "c1", MoveRules: buildBishopRules()} // own copy
	blackBishop := Piece{Color: "black", Position: "c8", MoveRules: buildBishopRules()} // own copy AGAIN
	fmt.Println(len(whiteBishop.MoveRules), len(blackBishop.MoveRules), "-- two separate slices in memory")
}

// Output:
// 4 4 -- two separate slices in memory
```

**With Flyweight:**

```go
package main

import "fmt"

type MoveRule string

type MoveRuleSet struct{ Rules []MoveRule } // heavy, built ONCE

func buildBishopRules() *MoveRuleSet {
	return &MoveRuleSet{Rules: []MoveRule{"diagonal-nw", "diagonal-ne", "diagonal-sw", "diagonal-se"}}
}

var bishopRules = buildBishopRules() // shared across EVERY Bishop, built exactly once

type Piece struct {
	Color    string
	Position string
	Rules    *MoveRuleSet // just a pointer -- zero duplication
}

func main() {
	whiteBishop := Piece{Color: "white", Position: "c1", Rules: bishopRules}
	blackBishop := Piece{Color: "black", Position: "c8", Rules: bishopRules} // same pointer
	fmt.Println(whiteBishop.Rules == blackBishop.Rules, "-- one shared MoveRuleSet in memory")
}

// Output:
// true -- one shared MoveRuleSet in memory
```

> [!warning] Tradeoffs
> Only worth it when the shared data is genuinely immutable *and* genuinely large relative to instance count — adds indirection for a memory saving that doesn't matter at small scale. A natural extension of [[LLD/09 - Design Chess/Design Chess|the Chess chapter's]] shared move-logic design.

---

## Proxy

> [!example] Layman
> A receptionist who screens calls before connecting you — same eventual access, but with a controlled gate in front of it.

**Trigger:** "add caching, lazy-loading, or a permission check without changing the object itself"
**When to use:** adding access control, lazy initialization, logging, or caching around an object without changing the object itself or its callers.

> [!tip] Identification trick vs. Decorator
> Same shape (both wrap an object behind the same interface) — the question is intent. Controls **access** to something that already works? → Proxy. Adds genuinely **new behavior** on top? → Decorator.

**Without the pattern:**

```go
package main

import "fmt"

type RealImageLoader struct{}

func (RealImageLoader) Load(path string) string {
	fmt.Println("reading full resolution from disk:", path) // expensive, runs EVERY call
	return "image-bytes:" + path
}

func main() {
	loader := RealImageLoader{}
	loader.Load("photo.jpg") // expensive
	loader.Load("photo.jpg") // expensive AGAIN -- identical work repeated
}

// Output:
// reading full resolution from disk: photo.jpg
// reading full resolution from disk: photo.jpg
```

**With Proxy:**

```go
package main

import "fmt"

type ImageLoader interface{ Load(path string) string }

type RealImageLoader struct{}

func (RealImageLoader) Load(path string) string {
	fmt.Println("reading full resolution from disk:", path)
	return "image-bytes:" + path
}

type CachingImageProxy struct {
	real  RealImageLoader
	cache map[string]string
}

func (p *CachingImageProxy) Load(path string) string {
	if data, ok := p.cache[path]; ok {
		fmt.Println("served from cache:", path)
		return data // real loader never touched
	}
	data := p.real.Load(path)
	p.cache[path] = data
	return data
}

func main() {
	var loader ImageLoader = &CachingImageProxy{cache: map[string]string{}}
	loader.Load("photo.jpg") // expensive, first time
	loader.Load("photo.jpg") // cheap, served from proxy's own cache
}

// Output:
// reading full resolution from disk: photo.jpg
// served from cache: photo.jpg
```

> [!warning] Tradeoffs
> A caching proxy needs the same invalidation discipline as any cache (per [[CS Fundamentals/04 - Caching/Caching Strategies|Caching Strategies]]). **System-design-level example:** a cache-aside layer is architecturally a **caching proxy**; [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|a rate limiter in front of a service]] is architecturally a **protection proxy**.

---

## Interview Q&A

> [!info] Leveled by seniority
> **Beginner:** "What's the core difference between Decorator and Proxy — both wrap an object behind the same interface?" — Decorator *adds* new behavior/responsibility; Proxy *controls access* to existing behavior without adding new responsibility. **Intermediate:** "When would you choose Bridge over just subclassing?" — when there are genuinely 2+ independent dimensions of variation; subclassing alone forces `N x M` types, Bridge keeps them independent. **Senior:** "A Decorator chain has grown to 6 layers deep and a bug report is hard to trace — assess it." — expects recognizing the real, named tradeoff (stack-trace depth vs. combinability) rather than treating Decorator as free composability with no cost. **Staff:** "Design the integration layer for a system needing to support 5 different third-party shipping providers with different APIs." — expects Adapter at the integration boundary, one consistent `ShippingProvider` interface, isolating each provider's SDK-specific shape from the rest of the system. **Architect:** "How do you decide whether a Facade is hiding complexity well or just relocating it badly?" — expects checking whether callers who *do* need finer-grained control still have an escape hatch to the underlying subsystems, or whether the Facade has become a leaky, all-or-nothing bottleneck.

## Summary / Cheat Sheet

| Pattern | Trick | Complete code above |
|---|---|---|
| Adapter | You don't control the interface you're bridging to | ✅ |
| Bridge | 2+ independent dimensions of variation, avoids `N x M` types | ✅ |
| Composite | Works the same for one item or a group, recursively | ✅ |
| Decorator | Wraps to *add* new behavior, stackable in any combination | ✅ |
| Facade | Simplifies 3+ subsystems into one entry point | ✅ |
| Flyweight | Thousands of near-identical, immutable objects — share the data | ✅ |
| Proxy | Wraps to *control access* — caching, lazy-load, permission check | ✅ |

---
*Related: [[CS Fundamentals/00 - Learning Path|CS Fundamentals Learning Path]] · [[CS Fundamentals/10 - Design Principles/02 - Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] · [[CS Fundamentals/10 - Design Principles/03 - Creational Design Patterns - Full Code Deep Dive|Creational Design Patterns]] · [[CS Fundamentals/10 - Design Principles/05 - Behavioral Design Patterns - Full Code Deep Dive|Behavioral Design Patterns]]*
