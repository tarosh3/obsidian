---
title: "Behavioral Design Patterns — Full Code Deep Dive"
aliases: [Strategy, Observer, State, Command, Chain of Responsibility, Mediator, Memento, Template Method, Visitor, Iterator, Interpreter]
tags: [system-design, cs-fundamentals, design-principles, design-patterns, go, behavioral]
status: reference-quality
---

# Behavioral Design Patterns — Full Code Deep Dive

> [!abstract] What you'll be able to do after this chapter
> For every one of the 11 behavioral patterns, see a complete, runnable "without the pattern" program, the exact same problem solved "with the pattern," and a one-line trick for telling this pattern apart from the one it's most often confused with — this is the largest category, and also where the classic confusions (Strategy vs. State, Observer vs. Mediator) actually live.

> [!info] Companion chapter
> Part of a 3-chapter set with [[CS Fundamentals/10 - Design Principles/03 - Creational Design Patterns - Full Code Deep Dive|Creational]] and [[CS Fundamentals/10 - Design Principles/04 - Structural Design Patterns - Full Code Deep Dive|Structural]] Design Patterns. See [[CS Fundamentals/10 - Design Principles/02 - Design Patterns Cheat Sheet|the Design Patterns Cheat Sheet]] for all 23 at a glance. 6 of the 11 below also have full systems built around them — linked at the end of each section — if you want the deeper, multi-requirement version.

---

## Strategy

> [!example] Layman
> Choosing a route on a GPS — walking, driving, and transit are three interchangeable strategies for the same "get me from A to B" goal, swapped based on context.

**Trigger:** "swap this algorithm / pricing rule / sort order at runtime"
**When to use:** 3 or more interchangeable ways to do the same computation, chosen by the caller.

> [!tip] Identification trick vs. State
> Who decides which implementation runs? The **caller**, once, from outside, and it stays fixed → Strategy. The **object itself**, driven by its own internal transitions over time → State (below). This is the single most commonly confused pair in the whole set — always ask "who's driving the choice."

**Without the pattern:**

```go
package main

import "fmt"

func CalculateTotalBad(base float64, vehicleType string) float64 {
	if vehicleType == "ev" {
		return base * 0.8
	} else if vehicleType == "motorcycle" {
		return base * 0.5
	}
	return base // every new vehicle type = another branch in this one function, forever
}

func main() {
	fmt.Println(CalculateTotalBad(100, "ev"))
}

// Output:
// 80
```

**With Strategy:**

```go
package main

import "fmt"

type PricingStrategy interface {
	Calculate(base float64) float64
}

type FlatRate struct{}

func (FlatRate) Calculate(base float64) float64 { return base }

type EVDiscount struct{}

func (EVDiscount) Calculate(base float64) float64 { return base * 0.8 }

type MotorcycleDiscount struct{}

func (MotorcycleDiscount) Calculate(base float64) float64 { return base * 0.5 }

type Checkout struct{ strategy PricingStrategy }

func (c *Checkout) Total(base float64) float64 { return c.strategy.Calculate(base) }

func main() {
	checkout := &Checkout{strategy: EVDiscount{}}
	fmt.Println(checkout.Total(100)) // swap strategy at runtime, ZERO changes to Checkout
}

// Output:
// 80
```

> [!info] Full system-design depth
> The most-reused pattern in this handbook: [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Parking Lot]] (pricing), [[LLD/04 - Design a Rate Limiter/Design a Rate Limiter|Rate Limiter]] (token bucket vs. sliding window), [[LLD/03 - Design an Elevator System/Design an Elevator System|Elevator]] (dispatch), [[LLD/14 - Design an LRU Cache (pluggable eviction)/Design an LRU Cache (pluggable eviction)|LRU Cache]] (eviction policy), [[LLD/19 - Design a Stock Exchange/Design a Stock Exchange|Stock Exchange]] (order matching).

> [!warning] Tradeoffs
> Caller must pick the right strategy. Not worth it for 1-2 variants — earns its keep at 3+.

---

## Observer

> [!example] Layman
> A newsletter — the publisher doesn't know or care who's subscribed or what they'll do with the email; it just broadcasts to whoever signed up.

**Trigger:** "one event, many independent listeners react"
**When to use:** a state change needs to notify an open-ended, possibly-growing set of interested parties.

> [!tip] Identification trick vs. Mediator
> Observer is **one-to-many broadcast** — the subject doesn't know or care who's listening. Mediator (below) is **many-to-many coordination** — peers talk *through* a central hub that actively routes messages between specific parties. If it's "notify everyone interested," it's Observer; if it's "route this specific message to that specific peer," it's Mediator.

**Without the pattern:**

```go
package main

import "fmt"

type EmailSvc struct{}

func (EmailSvc) Notify(order string) { fmt.Println("email: order placed", order) }

type SMSSvc struct{}

func (SMSSvc) Notify(order string) { fmt.Println("sms: order placed", order) }

type Analytics struct{}

func (Analytics) Track(order string) { fmt.Println("analytics: tracked", order) }

func PlaceOrderBad(order string, email *EmailSvc, sms *SMSSvc, an *Analytics) {
	email.Notify(order) // OrderService hardcoded to know about every interested party
	sms.Notify(order)   // adding Slack notifications means editing THIS function again
	an.Track(order)
}

func main() {
	PlaceOrderBad("order-1", &EmailSvc{}, &SMSSvc{}, &Analytics{})
}

// Output:
// email: order placed order-1
// sms: order placed order-1
// analytics: tracked order-1
```

**With Observer:**

```go
package main

import "fmt"

type Observer interface{ Notify(event string) }

type EmailSvc struct{}

func (EmailSvc) Notify(order string) { fmt.Println("email: order placed", order) }

type SMSSvc struct{}

func (SMSSvc) Notify(order string) { fmt.Println("sms: order placed", order) }

type Analytics struct{}

func (Analytics) Notify(order string) { fmt.Println("analytics: tracked", order) }

type EventBus struct{ observers []Observer }

func (b *EventBus) Subscribe(o Observer) { b.observers = append(b.observers, o) }
func (b *EventBus) Publish(event string) {
	for _, o := range b.observers {
		o.Notify(event) // bus doesn't know or care WHO is listening or how many
	}
}

func main() {
	bus := &EventBus{}
	bus.Subscribe(EmailSvc{})
	bus.Subscribe(SMSSvc{})
	bus.Subscribe(Analytics{})
	bus.Publish("order-1") // adding a new listener never touches PlaceOrder logic
}

// Output:
// email: order placed order-1
// sms: order placed order-1
// analytics: tracked order-1
```

> [!info] Full system-design depth
> [[LLD/08 - Design a Notification System/Design a Notification System|Notification System]] (global `EventPublisher`) and [[LLD/22 - Design Google Meet/Design Google Meet|Google Meet]] (room-**scoped** Observer) — deliberately contrasted for different broadcast scope.

> [!warning] Tradeoffs
> Notification order usually isn't guaranteed — say so if it matters for the prompt.

---

## State

> [!example] Layman
> A traffic light — the exact same "signal change" event produces different behavior depending on whether the light is currently red, yellow, or green; the light doesn't recompute logic from scratch each time, it just knows what state it's in.

**Trigger:** "behavior changes based on internal status/lifecycle, not by caller choice"
**When to use:** an object's own transitions (placed → shipped → delivered) should drive its behavior, and an if/else on a status field is starting to miss cases.

> [!tip] Identification trick vs. Strategy
> Same shape as Strategy (a swappable object behind an interface) — the difference is *who* drives the swap. State changes **itself**, automatically, as a direct consequence of the object's own lifecycle. Strategy is chosen once, externally, and stays fixed.

**Without the pattern:**

```go
package main

import "fmt"

type Order struct{ status string }

func (o *Order) Advance() {
	if o.status == "placed" {
		o.status = "shipped"
	} else if o.status == "shipped" {
		o.status = "delivered"
	} else if o.status == "delivered" {
		// terminal -- easy to forget this branch entirely as statuses grow
	}
}

func main() {
	o := &Order{status: "placed"}
	o.Advance()
	o.Advance()
	fmt.Println(o.status)
}

// Output:
// delivered
```

**With State:**

```go
package main

import "fmt"

type OrderState interface{ Next(o *Order) }

type Placed struct{}

func (Placed) Next(o *Order) { o.state = Shipped{}; o.status = "shipped" }

type Shipped struct{}

func (Shipped) Next(o *Order) { o.state = Delivered{}; o.status = "delivered" }

type Delivered struct{}

func (Delivered) Next(o *Order) { /* terminal, no-op -- the type itself enforces this */ }

type Order struct {
	state  OrderState
	status string
}

func NewOrder() *Order   { return &Order{state: Placed{}, status: "placed"} }
func (o *Order) Advance() { o.state.Next(o) }

func main() {
	o := NewOrder()
	o.Advance()
	o.Advance()
	fmt.Println(o.status) // each state OWNS its own valid transition -- impossible to skip one
}

// Output:
// delivered
```

> [!info] Full system-design depth
> [[LLD/02 - Design a Vending Machine/Design a Vending Machine|Vending Machine]], [[LLD/03 - Design an Elevator System/Design an Elevator System|Elevator]] (movement), [[LLD/11 - Design an ATM/Design an ATM|ATM]], [[LLD/13 - Design a Food Delivery System/Design a Food Delivery System|Food Delivery's]] order lifecycle.

> [!warning] Tradeoffs
> More types than a status enum with a switch statement — worth it once the transition logic itself gets nontrivial (guards, side effects per transition), not for a trivial 2-state flag.

---

## Command

> [!example] Layman
> A restaurant order ticket — the waiter doesn't cook on the spot; they write the order as a discrete, passable object that the kitchen executes later, and a cancelled ticket can be pulled back out.

**Trigger:** "undo/redo, or queue an action to run later"
**When to use:** a request/action needs to be treated as data — queued, logged, delayed, or reversed.

> [!tip] Identification trick vs. Memento
> Command records the **action** (and how to reverse it) as a small, targeted object. Memento (below) records the **entire state** as a snapshot. If undoing means "reverse this one operation," use Command. If undoing means "go back to exactly how everything looked," use Memento.

**Without the pattern:**

```go
package main

import "fmt"

type Account struct{ Balance int }

func WithdrawBad(a *Account, amount int) error {
	if a.Balance < amount {
		return fmt.Errorf("insufficient funds")
	}
	a.Balance -= amount
	return nil
	// no record of WHAT happened -- undoing later means manually re-adding amount,
	// and only if you remembered the exact amount and account elsewhere yourself
}

func main() {
	acc := &Account{Balance: 100}
	_ = WithdrawBad(acc, 30)
	fmt.Println(acc.Balance)
	// to undo: acc.Balance += 30 -- only works if "30" was tracked somewhere separately
}

// Output:
// 70
```

**With Command:**

```go
package main

import "fmt"

type Account struct{ Balance int }

type Command interface {
	Execute() error
	Undo()
}

type WithdrawCommand struct {
	account *Account
	amount  int
}

func (c *WithdrawCommand) Execute() error {
	if c.account.Balance < c.amount {
		return fmt.Errorf("insufficient funds")
	}
	c.account.Balance -= c.amount
	return nil
}

func (c *WithdrawCommand) Undo() {
	c.account.Balance += c.amount // the command KNOWS how to reverse itself
}

func main() {
	acc := &Account{Balance: 100}
	cmd := &WithdrawCommand{account: acc, amount: 30}

	_ = cmd.Execute()
	fmt.Println(acc.Balance) // 70

	cmd.Undo() // no need to remember "30" separately -- the command object already has it
	fmt.Println(acc.Balance) // 100
}

// Output:
// 70
// 100
```

> [!info] Full system-design depth
> [[LLD/11 - Design an ATM/Design an ATM|ATM]] (`WithdrawCommand.Undo()` for rare failure recovery) and [[LLD/15 - Design a Text Editor/Design a Text Editor|Text Editor]] (undo/redo as Command's *primary* feature, not a safety net).

> [!warning] Tradeoffs
> Every reversible action needs its own `Undo()` logic written and kept correct — real, ongoing maintenance cost as the set of commands grows.

---

## Chain of Responsibility

> [!example] Layman
> A customer complaint escalation line — the first-line agent tries to resolve it; if they can't, it passes to a supervisor, then a manager, each link deciding whether to handle it or forward it further.

**Trigger:** "pass a request along a chain of handlers, each deciding to act or forward it"
**When to use:** multiple potential handlers exist, and you don't know ahead of time which one will actually handle a given request.

> [!tip] Identification trick
> Ask: "is there a **sequence** of handlers where the request stops at the first one that can handle it, and each handler is unaware of the others beyond 'pass it on'?" If yes → Chain of Responsibility. If every handler always runs regardless of outcome → that's closer to Observer.

**Without the pattern:**

```go
package main

import "fmt"

func HandleComplaintBad(severity int) {
	if severity <= 1 {
		fmt.Println("front-line agent resolves it")
	} else if severity <= 3 {
		fmt.Println("supervisor resolves it")
	} else {
		fmt.Println("manager resolves it")
	}
	// adding a new escalation tier means editing this one growing if/else chain
}

func main() {
	HandleComplaintBad(4)
}

// Output:
// manager resolves it
```

**With Chain of Responsibility:**

```go
package main

import "fmt"

type Handler interface {
	SetNext(Handler)
	Handle(severity int)
}

type BaseHandler struct{ next Handler }

func (h *BaseHandler) SetNext(n Handler) { h.next = n }
func (h *BaseHandler) PassOn(severity int) {
	if h.next != nil {
		h.next.Handle(severity)
	}
}

type FrontLineAgent struct{ BaseHandler }

func (h *FrontLineAgent) Handle(severity int) {
	if severity <= 1 {
		fmt.Println("front-line agent resolves it")
		return
	}
	h.PassOn(severity) // doesn't handle it -- passes to the next link, unaware what that is
}

type Supervisor struct{ BaseHandler }

func (h *Supervisor) Handle(severity int) {
	if severity <= 3 {
		fmt.Println("supervisor resolves it")
		return
	}
	h.PassOn(severity)
}

type Manager struct{ BaseHandler }

func (h *Manager) Handle(severity int) { fmt.Println("manager resolves it") }

func main() {
	agent, supervisor, manager := &FrontLineAgent{}, &Supervisor{}, &Manager{}
	agent.SetNext(supervisor)
	supervisor.SetNext(manager)

	agent.Handle(4) // a new tier = one new type + one SetNext call, no existing code touched
}

// Output:
// manager resolves it
```

> [!info] Full system-design depth
> [[LLD/07 - Design a Logger Framework/Design a Logger Framework|Logger Framework]] ("all handlers act" variant) and [[LLD/08 - Design a Notification System/Design a Notification System|Notification System]] ("first rejection stops the chain" variant) — explicitly contrasted as two different variants of the same pattern.

> [!warning] Tradeoffs
> If the chain is wired incorrectly (a missing `SetNext`), a request silently falls off the end with no handler — a real, easy-to-miss failure mode worth testing explicitly.

---

## Mediator

> [!example] Layman
> An air traffic controller — pilots never talk directly to every other plane in the sky; they all talk to the tower, which coordinates who does what.

**Trigger:** "centralize how a group of peer objects communicate, so they never need direct references to each other"
**When to use:** objects would otherwise need direct references to many peers, creating a tangled many-to-many mess.

> [!tip] Identification trick vs. Observer
> Already covered above from Observer's side — repeated here since it's a two-way confusion. Mediator actively **routes** a specific message to a specific peer through a central hub; Observer **broadcasts** to anyone subscribed, with no notion of "peers talking to each other."

**Without the pattern:**

```go
package main

import "fmt"

type User struct {
	Name  string
	Peers []*User // each user must hold direct references to every other user to message them
}

func (u *User) Send(msg string) {
	for _, peer := range u.Peers {
		fmt.Println(u.Name, "->", peer.Name, ":", msg) // N users need N-1 references each
	}
}

func main() {
	alice := &User{Name: "Alice"}
	bob := &User{Name: "Bob"}
	alice.Peers = []*User{bob}
	bob.Peers = []*User{alice}

	alice.Send("hi")
}

// Output:
// Alice -> Bob : hi
```

**With Mediator:**

```go
package main

import "fmt"

type ChatRoom struct{ users map[string]*User }

func NewChatRoom() *ChatRoom          { return &ChatRoom{users: map[string]*User{}} }
func (r *ChatRoom) Register(u *User)  { r.users[u.Name] = u; u.room = r }
func (r *ChatRoom) Relay(from, to, msg string) {
	if target, ok := r.users[to]; ok {
		fmt.Println(from, "->", to, ":", msg)
		_ = target // in a real system: target.Receive(msg)
	}
}

type User struct {
	Name string
	room *ChatRoom // users only know the MEDIATOR, never each other directly
}

func (u *User) Send(to, msg string) { u.room.Relay(u.Name, to, msg) }

func main() {
	room := NewChatRoom()
	alice := &User{Name: "Alice"}
	bob := &User{Name: "Bob"}
	room.Register(alice)
	room.Register(bob)

	alice.Send("Bob", "hi") // adding Charlie later never touches Alice's or Bob's code
}

// Output:
// Alice -> Bob : hi
```

> [!info] Full system-design depth
> [[LLD/18 - Design a Chat Application/Design a Chat Application|Design a Chat Application]] — `ChatRoom` as Mediator, explicitly contrasted against Observer despite both involving a "room" coordinating multiple participants.

> [!warning] Tradeoffs
> The mediator itself can grow into a "god object" that knows too much about every peer's needs — worth watching as the number of peer interaction types grows.

---

## Memento

> [!example] Layman
> A video game save file — a full snapshot of exactly where you were, restorable later, without the save file needing to understand *how* the game works internally.

**Trigger:** "capture and externally store an object's internal state so it can be restored later, without violating its encapsulation"
**When to use:** you need undo/checkpoint capability where a full-state snapshot is cheap or infrequent.

> [!tip] Identification trick vs. Command
> Already covered above from Command's side. The quick test: is the *entire object* worth snapshotting wholesale (Memento), or is only a small, specific delta worth recording per action (Command)? Snapshot cost is what decides this — see the Tradeoffs note below.

**Without the pattern:**

```go
package main

import "fmt"

type Editor struct{ Content string }

func (e *Editor) Type(text string) { e.Content += text } // no way back, ever

func main() {
	e := &Editor{}
	e.Type("Hello")
	e.Type(" World")
	fmt.Println(e.Content)
	// no way to get back to "Hello" -- nothing recorded the earlier state
}

// Output:
// Hello World
```

**With Memento:**

```go
package main

import "fmt"

type EditorMemento struct{ content string } // opaque to everyone but Editor itself

type Editor struct{ Content string }

func (e *Editor) Type(text string)         { e.Content += text }
func (e *Editor) Save() *EditorMemento     { return &EditorMemento{content: e.Content} }
func (e *Editor) Restore(m *EditorMemento) { e.Content = m.content }

func main() {
	editor := &Editor{}
	editor.Type("Hello")
	checkpoint := editor.Save() // external snapshot, Editor's internals stay private
	editor.Type(" World")
	fmt.Println(editor.Content) // "Hello World"

	editor.Restore(checkpoint)
	fmt.Println(editor.Content) // back to "Hello"
}

// Output:
// Hello World
// Hello
```

> [!warning] Tradeoffs — a decision this book actually made, worth knowing precisely
> Snapshotting the **entire** state on every checkpoint is correct but wasteful for large objects — exactly why [[LLD/15 - Design a Text Editor/Design a Text Editor|the Text Editor chapter]] rejected Memento and chose Command-based delta storage instead: storing only the small delta each command needs to reverse itself is far cheaper than a full-document snapshot per keystroke. Memento is the right fit when full-state snapshots are cheap or infrequent — not always.

---

## Template Method

> [!example] Layman
> A recipe card with fixed steps (preheat, mix, bake, cool) where only the specific ingredients change per recipe — the order and structure stay identical every time.

**Trigger:** "one fixed algorithm skeleton, a few steps plugged in per case"
**When to use:** several algorithms share the same overall sequence of steps but differ in one or two specific steps.

> [!tip] Identification trick
> Ask: "do 2+ processes follow the EXACT SAME sequence of steps, differing only in what a couple of those steps actually do?" If the *skeleton* is shared and only the *details* vary → Template Method.

**Without the pattern:**

```go
package main

import "fmt"

func PlayChessBad() {
	moves := 0
	for moves < 2 {
		fmt.Println("chess: check win, get move, apply move")
		moves++
	}
}

func PlayTicTacToeBad() {
	moves := 0
	for moves < 2 {
		fmt.Println("tictactoe: check win, get move, apply move") // IDENTICAL structure, copy-pasted
		moves++
	}
}

func main() {
	PlayChessBad()
	PlayTicTacToeBad()
}

// Output:
// chess: check win, get move, apply move
// chess: check win, get move, apply move
// tictactoe: check win, get move, apply move
// tictactoe: check win, get move, apply move
```

**With Template Method:**

```go
package main

import "fmt"

type Game interface {
	CheckWin(moves int) bool
	GetMove() string
	ApplyMove(move string)
}

func Play(g Game) { // the skeleton -- written ONCE, identical for every game
	moves := 0
	for !g.CheckWin(moves) {
		move := g.GetMove()
		g.ApplyMove(move)
		moves++
	}
}

type Chess struct{}

func (Chess) CheckWin(moves int) bool { return moves >= 2 }
func (Chess) GetMove() string         { return "chess-move" }
func (Chess) ApplyMove(m string)      { fmt.Println("chess:", m) }

type TicTacToe struct{}

func (TicTacToe) CheckWin(moves int) bool { return moves >= 2 }
func (TicTacToe) GetMove() string         { return "tictactoe-move" }
func (TicTacToe) ApplyMove(m string)      { fmt.Println("tictactoe:", m) }

func main() {
	Play(Chess{})     // same Play() skeleton
	Play(TicTacToe{}) // only the varying steps differ
}

// Output:
// chess: chess-move
// chess: chess-move
// tictactoe: tictactoe-move
// tictactoe: tictactoe-move
```

> [!warning] Tradeoffs
> Go has no class inheritance, so this leans on interfaces + composition rather than the classic GoF subclass-overrides-a-hook-method version — works well, but a slightly different flavor. **Where this shape appears implicitly:** the exact shared skeleton across [[LLD/09 - Design Chess/Design Chess|Chess]] and [[LLD/10 - Design Tic-Tac-Toe with AI/Design Tic-Tac-Toe with AI|Tic-Tac-Toe]].

---

## Visitor

> [!example] Layman
> A museum tour guide who visits different exhibit types and knows exactly what to say about each, without the exhibits needing to know how to describe themselves to every possible visitor.

**Trigger:** "add a new operation across a whole family of related types without modifying any of those types"
**When to use:** the set of *operations* grows more often than the set of *types* — the opposite situation from where Strategy fits best.

> [!tip] Identification trick
> Ask: "am I adding a new OPERATION across many EXISTING types, without touching those types' source code?" If yes → Visitor. If instead you're adding a new *type* that must support existing operations, Visitor is often the wrong direction — a plain interface method is simpler.

**Without the pattern:**

```go
package main

import "fmt"

type Circle struct{ Radius float64 }

func (c Circle) Area() float64    { return 3.14159 * c.Radius * c.Radius }
func (c Circle) Describe() string { return fmt.Sprintf("circle r=%.1f", c.Radius) } // edits Circle itself

type Square struct{ Side float64 }

func (s Square) Area() float64    { return s.Side * s.Side }
func (s Square) Describe() string { return fmt.Sprintf("square s=%.1f", s.Side) } // edits Square itself

// every NEW operation (Describe, Perimeter, Export...) means touching every shape type again

func main() {
	fmt.Println(Circle{Radius: 2}.Describe())
	fmt.Println(Square{Side: 3}.Describe())
}

// Output:
// circle r=2.0
// square s=3.0
```

**With Visitor:**

```go
package main

import "fmt"

type Shape interface{ Accept(v Visitor) }

type Circle struct{ Radius float64 }

func (c Circle) Accept(v Visitor) { v.VisitCircle(c) }

type Square struct{ Side float64 }

func (s Square) Accept(v Visitor) { v.VisitSquare(s) }

type Visitor interface {
	VisitCircle(Circle)
	VisitSquare(Square)
}

// A brand-new operation -- ZERO changes to Circle or Square themselves
type DescribeVisitor struct{}

func (DescribeVisitor) VisitCircle(c Circle) { fmt.Printf("circle r=%.1f\n", c.Radius) }
func (DescribeVisitor) VisitSquare(s Square) { fmt.Printf("square s=%.1f\n", s.Side) }

func main() {
	shapes := []Shape{Circle{Radius: 2}, Square{Side: 3}}
	for _, s := range shapes {
		s.Accept(DescribeVisitor{})
	}
}

// Output:
// circle r=2.0
// square s=3.0
```

> [!warning] Tradeoffs — a real, honest Go limitation
> Visitor leans on method overloading (`Visit(Circle)`, `Visit(Square)` dispatched by the compiler on argument type) — a feature Go doesn't have. Go's version needs differently-named methods (`VisitCircle`, `VisitSquare`) or a type switch, partially defeating the pattern's original elegance. Worth naming honestly, not forcing Visitor into Go as if it were free.

---

## Iterator

> [!example] Layman
> Flipping through a photo album page by page — you don't need to know whether photos are stored by date or by album; "next" always works the same way.

**Trigger:** "sequential access to a collection without exposing how it's structured internally"
**When to use:** custom traversal logic (a tree, a graph, a linked list) that Go's built-in `range` doesn't already handle.

> [!tip] Identification trick — Go-specific
> Before reaching for a hand-rolled `HasNext()/Next()`, ask: "does `for range` already do this for me?" For slices, maps, channels — yes, always. Only build a real Iterator for custom structures `range` can't walk natively (a tree, a linked list, a graph).

**Without the pattern:**

```go
package main

import "fmt"

type Node struct {
	Value int
	Next  *Node
}

func PrintAllBad(head *Node) {
	for n := head; n != nil; n = n.Next { // caller entangled with the internal linked-list structure
		fmt.Println(n.Value)
	}
}

func main() {
	head := &Node{Value: 1, Next: &Node{Value: 2, Next: &Node{Value: 3}}}
	PrintAllBad(head)
}

// Output:
// 1
// 2
// 3
```

**With Iterator:**

```go
package main

import "fmt"

type Node struct {
	Value int
	Next  *Node
}

type Iterator interface {
	HasNext() bool
	Next() int
}

type ListIterator struct{ current *Node }

func (it *ListIterator) HasNext() bool { return it.current != nil }
func (it *ListIterator) Next() int {
	v := it.current.Value
	it.current = it.current.Next
	return v
}

func PrintAll(it Iterator) { // works for a linked list, array, or anything else implementing Iterator
	for it.HasNext() {
		fmt.Println(it.Next())
	}
}

func main() {
	head := &Node{Value: 1, Next: &Node{Value: 2, Next: &Node{Value: 3}}}
	PrintAll(&ListIterator{current: head}) // caller never touches Node/.Next directly
}

// Output:
// 1
// 2
// 3
```

> [!tip] Go-native idiom — worth naming as baked into the language
> `for range mySlice`, `for range myMap`, `for range myChannel` — and a goroutine pushing values onto a channel that callers `range` over — are essentially built-in Iterator implementations. A hand-rolled `HasNext()/Next()` is rarely needed for Go's own collection types.

---

## Interpreter

> [!example] Layman
> A translator who understands a specific set of grammar rules well enough to convert spoken sentences into actions.

**Trigger:** "evaluate a small grammar as a composable expression tree"
**When to use:** rare in general application code, common when parsing a small domain-specific expression — recurrence rules, search query syntax, a rule engine's condition language.

> [!tip] Identification trick
> Ask: "is this a real, potentially-growing GRAMMAR, or just 3-4 fixed cases?" If it's a small, fixed, unlikely-to-grow set of cases, a plain `switch` statement is the honest answer — building an expression tree for that is over-engineering. Interpreter earns its keep only once the grammar is genuinely extensible.

**Without the pattern:**

```go
package main

import "fmt"

func ExpandBad(rule string) []string {
	if rule == "daily" {
		return []string{"Mon", "Tue", "Wed", "Thu", "Fri", "Sat", "Sun"}
	}
	if rule == "weekly-mon-wed" {
		return []string{"Mon", "Wed"} // every new rule = another if-branch, forever
	}
	return nil
}

func main() {
	fmt.Println(ExpandBad("weekly-mon-wed"))
}

// Output:
// [Mon Wed]
```

**With Interpreter:**

```go
package main

import "fmt"

type Expr interface{ Evaluate() []string }

type OnDaysExpr struct{ Days []string }

func (e OnDaysExpr) Evaluate() []string { return e.Days }

type LimitExpr struct {
	Inner Expr
	Max   int
}

func (e LimitExpr) Evaluate() []string {
	days := e.Inner.Evaluate()
	if len(days) > e.Max {
		return days[:e.Max]
	}
	return days
}

func main() {
	// "Monday and Wednesday, but only the first occurrence" -- a composed expression tree,
	// not a growing string-match function
	rule := LimitExpr{Inner: OnDaysExpr{Days: []string{"Mon", "Wed"}}, Max: 1}
	fmt.Println(rule.Evaluate())
}

// Output:
// [Mon]
```

> [!warning] Tradeoffs
> Per [[HLD/21 - Design Google Calendar/Design Google Calendar|the Google Calendar chapter's]] own scope choice: most real systems implement the *effect* of Interpreter (expanding a recurrence rule) without building a general-purpose grammar/parser framework around it — a reasonable, deliberate scope decision, not a compromise.

---

## Interview Q&A

> [!info] Leveled by seniority
> **Beginner:** "How do you tell Strategy and State apart, since both involve swappable behavior objects?" — Strategy is chosen by the *caller* and stays fixed for an object's lifetime; State is driven by the object's *own internal transitions* and changes itself over time. **Intermediate:** "Why did the Text Editor chapter reject Memento for undo/redo?" — full-state snapshotting on every keystroke is correct but wasteful; Command-based delta storage is far cheaper — a real, deliberate tradeoff, not a default choice. **Senior:** "A codebase needs to add 3 new operations across 10 existing shape types without touching those types — what pattern, and what's the Go-specific catch?" — expects Visitor, plus the honest Go limitation: no method overloading, so `VisitX` methods need distinct names per type. **Staff:** "Design an undo/redo system for a drawing app with very large canvases, where Memento's full-snapshot cost is genuinely prohibitive." — expects Command-based delta storage (the exact fix the Text Editor chapter already applied), storing only what changed per action rather than the whole canvas. **Architect:** "When does Interpreter's overhead (building a real grammar/parser) actually pay for itself vs. just hardcoding a few cases?" — expects recognizing the break-even point: a small, fixed, unlikely-to-grow set of rules doesn't justify a parser; a genuinely extensible, user-facing expression language (search syntax, rule engines) does.

## Summary / Cheat Sheet

| Pattern | Trick | Complete code above |
|---|---|---|
| Strategy | Caller picks the implementation, once, from outside | ✅ |
| Observer | One-to-many broadcast — publisher doesn't know who's listening | ✅ |
| State | Object drives its own transitions, automatically | ✅ |
| Command | Turn an action into data — queue, log, undo it | ✅ |
| Chain of Responsibility | Sequence of handlers, stops at the first that handles it | ✅ |
| Mediator | Many-to-many coordination through one central hub | ✅ |
| Memento | Full-state external snapshot + restore | ✅ |
| Template Method | Same skeleton, only a few steps vary | ✅ |
| Visitor | New operations across many types, without touching those types | ✅ |
| Iterator | Sequential access without exposing internal structure — `range` in Go | ✅ |
| Interpreter | A real, growing grammar — not just 3-4 fixed cases | ✅ |

---
*Related: [[CS Fundamentals/00 - Learning Path|CS Fundamentals Learning Path]] · [[CS Fundamentals/10 - Design Principles/02 - Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] · [[CS Fundamentals/10 - Design Principles/03 - Creational Design Patterns - Full Code Deep Dive|Creational Design Patterns]] · [[CS Fundamentals/10 - Design Principles/04 - Structural Design Patterns - Full Code Deep Dive|Structural Design Patterns]] · [[LLD/15 - Design a Text Editor/Design a Text Editor|Design a Text Editor]]*
