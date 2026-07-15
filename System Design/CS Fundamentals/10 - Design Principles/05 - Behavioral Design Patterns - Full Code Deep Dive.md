---
title: "Behavioral Design Patterns — Full Code Deep Dive"
aliases: [Memento, Template Method, Visitor, Iterator, Interpreter]
tags: [system-design, cs-fundamentals, design-principles, design-patterns, go, behavioral]
status: reference-quality
---

# Behavioral Design Patterns — Full Code Deep Dive

> [!abstract] What you'll be able to do after this chapter
> For the 5 behavioral patterns without a full LLD chapter, see the exact bad code, feel exactly why it breaks, and see the exact refactor. The other 6 already have full case studies — recapped and cross-linked here, not re-derived.

> [!info] Companion chapter
> Part of a 3-chapter set with [[CS Fundamentals/10 - Design Principles/Creational Design Patterns - Full Code Deep Dive|Creational]] and [[CS Fundamentals/10 - Design Principles/Structural Design Patterns - Full Code Deep Dive|Structural]] Design Patterns. See [[CS Fundamentals/10 - Design Principles/Design Patterns Cheat Sheet|the Design Patterns Cheat Sheet]] for all 23 at a glance.

---

## Already fully covered — recap only

> [!success] 6 patterns with real, full case studies — nothing to re-derive
> **Strategy:** [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Parking Lot]], [[LLD/04 - Design a Rate Limiter/Design a Rate Limiter|Rate Limiter]], [[LLD/03 - Design an Elevator System/Design an Elevator System|Elevator]], [[LLD/14 - Design an LRU Cache (pluggable eviction)/Design an LRU Cache (pluggable eviction)|LRU Cache]]. **Observer:** [[LLD/08 - Design a Notification System/Design a Notification System|Notification System]], [[LLD/22 - Design Google Meet/Design Google Meet|Google Meet]]. **State:** [[LLD/02 - Design a Vending Machine/Design a Vending Machine|Vending Machine]], [[LLD/03 - Design an Elevator System/Design an Elevator System|Elevator]]. **Command:** [[LLD/11 - Design an ATM/Design an ATM|ATM]], [[LLD/15 - Design a Text Editor/Design a Text Editor|Text Editor]]. **Chain of Responsibility:** [[LLD/07 - Design a Logger Framework/Design a Logger Framework|Logger Framework]], [[LLD/08 - Design a Notification System/Design a Notification System|Notification System]]. **Mediator:** [[LLD/18 - Design a Chat Application/Design a Chat Application|Chat Application]].

---

## Memento

> [!example] Layman
> A video game save file — a full snapshot of exactly where you were, restorable later, without the save file needing to understand *how* the game works internally.

**When to use:** you need undo/checkpoint capability without breaking the object's own encapsulation.

```go
// BEFORE — no clean way to undo; caller would have to manually remember old state
type Editor struct{ Content string }

func (e *Editor) Type(text string) { e.Content += text } // no way back, ever
```

```go
// AFTER — Memento: snapshot state externally, restore without exposing internals
type EditorMemento struct{ content string } // opaque to everyone but Editor itself

type Editor struct{ Content string }

func (e *Editor) Type(text string)         { e.Content += text }
func (e *Editor) Save() *EditorMemento     { return &EditorMemento{content: e.Content} }
func (e *Editor) Restore(m *EditorMemento) { e.Content = m.content }

editor := &Editor{}
editor.Type("Hello")
checkpoint := editor.Save()
editor.Type(" World")
editor.Restore(checkpoint) // back to "Hello"
```

> [!warning] Tradeoffs — a decision this book actually made, worth knowing precisely
> Snapshotting the **entire** state on every checkpoint is correct but wasteful for large objects — exactly why [[LLD/15 - Design a Text Editor/Design a Text Editor|the Text Editor chapter]] rejected Memento and chose Command-based delta storage instead: storing only the small delta each command needs to reverse itself is far cheaper than a full-document snapshot per keystroke. Memento is the right fit when full-state snapshots are cheap or infrequent — not always.

---

## Template Method

> [!example] Layman
> A recipe card with fixed steps (preheat, mix, bake, cool) where only the specific ingredients change per recipe — the order and structure stay identical every time.

**When to use:** several algorithms share the same overall sequence of steps but differ in one or two specific steps.

```go
// BEFORE — Chess and Tic-Tac-Toe each hand-roll an almost-identical game loop
func PlayChessBad() {
	for {
		if checkChessWin() { break }
		move := getChessMove()
		applyChessMove(move)
	}
}
func PlayTicTacToeBad() {
	for {
		if checkTTTWin() { break } // IDENTICAL structure, different specifics, copy-pasted
		move := getTTTMove()
		applyTTTMove(move)
	}
}
```

```go
// AFTER — Template Method: one shared skeleton, games plug in only what varies
type Game interface {
	CheckWin() bool
	GetMove() Move
	ApplyMove(Move)
}

func Play(g Game) { // the skeleton — written ONCE, identical for every game
	for {
		if g.CheckWin() { break }
		move := g.GetMove()
		g.ApplyMove(move)
	}
}

type Chess struct{}
func (c *Chess) CheckWin() bool   { return checkChessWin() }
func (c *Chess) GetMove() Move    { return getChessMove() }
func (c *Chess) ApplyMove(m Move) { applyChessMove(m) }

Play(&Chess{}) // the same Play() works for Chess, Tic-Tac-Toe, anything implementing Game
```

> [!warning] Tradeoffs
> Go has no class inheritance, so this leans on interfaces + composition rather than the classic GoF subclass-overrides-a-hook-method version — works well, but a slightly different flavor. **Where this shape appears implicitly:** the exact shared skeleton across [[LLD/09 - Design Chess/Design Chess|Chess]] and [[LLD/10 - Design Tic-Tac-Toe with AI/Design Tic-Tac-Toe with AI|Tic-Tac-Toe]].

---

## Visitor

> [!example] Layman
> A museum tour guide who visits different exhibit types and knows exactly what to say about each, without the exhibits needing to know how to describe themselves to every possible visitor.

**When to use:** adding a new operation across a whole family of related types, without modifying any of those types.

```go
// BEFORE — adding a new operation means editing Circle, Square, AND every future shape
type Circle struct{ Radius float64 }
func (c Circle) Area() float64 { return 3.14159 * c.Radius * c.Radius }

type Square struct{ Side float64 }
func (s Square) Area() float64 { return s.Side * s.Side }
// want to add Describe()? edit Circle, edit Square, edit every shape that will ever exist
```

```go
// AFTER — Visitor: new operations added WITHOUT touching Circle or Square again
type Shape interface{ Accept(v Visitor) }

type Circle struct{ Radius float64 }
func (c Circle) Accept(v Visitor) { v.VisitCircle(c) }

type Square struct{ Side float64 }
func (s Square) Accept(v Visitor) { v.VisitSquare(s) }

type Visitor interface {
	VisitCircle(Circle)
	VisitSquare(Square)
}

// A brand-new operation — zero changes to Circle or Square themselves
type DescribeVisitor struct{}
func (DescribeVisitor) VisitCircle(c Circle) { fmt.Printf("circle r=%.1f\n", c.Radius) }
func (DescribeVisitor) VisitSquare(s Square) { fmt.Printf("square s=%.1f\n", s.Side) }

shapes := []Shape{Circle{Radius: 2}, Square{Side: 3}}
for _, s := range shapes {
	s.Accept(DescribeVisitor{})
}
```

> [!warning] Tradeoffs — a real, honest Go limitation
> Visitor leans on method overloading (`Visit(Circle)`, `Visit(Square)` dispatched by the compiler on argument type) — a feature Go doesn't have. Go's version needs differently-named methods (`VisitCircle`, `VisitSquare`) or a type switch, partially defeating the pattern's original elegance. Worth naming honestly, not forcing Visitor into Go as if it were free.

---

## Iterator

> [!example] Layman
> Flipping through a photo album page by page — you don't need to know whether photos are stored by date or by album; "next" always works the same way.

**When to use:** sequential access to a collection's elements, without exposing how the collection is actually structured internally.

```go
// BEFORE — caller must know it's a linked list internally to traverse it at all
type Node struct {
	Value int
	Next  *Node
}

func PrintAllBad(head *Node) {
	for n := head; n != nil; n = n.Next { // caller entangled with the internal structure
		fmt.Println(n.Value)
	}
}
```

```go
// AFTER — Iterator hides the internal structure behind HasNext()/Next()
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
```

> [!tip] Go-native idiom — worth naming as baked into the language
> `for range mySlice`, `for range myMap`, `for range myChannel` — and a goroutine pushing values onto a channel that callers `range` over — are essentially built-in Iterator implementations. A hand-rolled `HasNext()/Next()` is rarely needed for Go's own collection types, only for custom traversal logic (a tree, a graph) the language doesn't natively iterate.

---

## Interpreter

> [!example] Layman
> A translator who understands a specific set of grammar rules well enough to convert spoken sentences into actions.

**When to use:** rare in general application code, common when parsing a small domain-specific expression — recurrence rules, search query syntax, a rule engine's condition language.

```go
// BEFORE — recurrence-rule parsing hardcoded as brittle, growing string matching
func ExpandBad(rule string) []string {
	if rule == "daily" {
		return everyDay()
	}
	if rule == "weekly-mon-wed" {
		return []string{"Mon", "Wed"} // every new rule = another if-branch, forever
	}
	return nil
}
```

```go
// AFTER — Interpreter: a small grammar evaluated as a real, composable expression tree
type Expr interface{ Evaluate() []string }

type OnDaysExpr struct{ Days []string }
func (e OnDaysExpr) Evaluate() []string { return e.Days }

type UntilExpr struct {
	Inner  Expr
	Cutoff string
}
func (e UntilExpr) Evaluate() []string { return filterBefore(e.Inner.Evaluate(), e.Cutoff) }

// "every Monday and Wednesday until March" becomes a composed tree, not a string match:
rule := UntilExpr{Inner: OnDaysExpr{Days: []string{"Mon", "Wed"}}, Cutoff: "2026-03-01"}
occurrences := rule.Evaluate()
```

> [!warning] Tradeoffs
> Per [[HLD/21 - Design Google Calendar/Design Google Calendar|the Google Calendar chapter's]] own scope choice: most real systems implement the *effect* of Interpreter (expanding a recurrence rule) without building a general-purpose grammar/parser framework around it — a reasonable, deliberate scope decision, not a compromise.

---

## Interview Q&A

> [!info] Leveled by seniority
> **Beginner:** "How do you tell Strategy and State apart, since both involve swappable behavior objects?" — Strategy is chosen by the *caller* and stays fixed for an object's lifetime; State is driven by the object's *own internal transitions* and changes itself over time. **Intermediate:** "Why did the Text Editor chapter reject Memento for undo/redo?" — full-state snapshotting on every keystroke is correct but wasteful; Command-based delta storage is far cheaper — a real, deliberate tradeoff, not a default choice. **Senior:** "A codebase needs to add 3 new operations across 10 existing shape types without touching those types — what pattern, and what's the Go-specific catch?" — expects Visitor, plus the honest Go limitation: no method overloading, so `VisitX` methods need distinct names per type. **Staff:** "Design an undo/redo system for a drawing app with very large canvases, where Memento's full-snapshot cost is genuinely prohibitive." — expects Command-based delta storage (the exact fix the Text Editor chapter already applied), storing only what changed per action rather than the whole canvas. **Architect:** "When does Interpreter's overhead (building a real grammar/parser) actually pay for itself vs. just hardcoding a few cases?" — expects recognizing the break-even point: a small, fixed, unlikely-to-grow set of rules doesn't justify a parser; a genuinely extensible, user-facing expression language (search syntax, rule engines) does.

## Summary / Cheat Sheet

- **Strategy, Observer, State, Command, Chain of Responsibility, Mediator:** full chapters — see the recap section above for links.
- **Memento:** external snapshot + restore without breaking encapsulation — full-state cost is real; Command's delta storage can be cheaper.
- **Template Method:** one shared algorithm skeleton, varying steps plugged in — Go does this via interfaces, not subclassing.
- **Visitor:** add new operations across a type family without touching the types — Go's lack of overloading is a real, honest limitation here.
- **Iterator:** hides a collection's internal structure behind sequential access — Go's `range` already *is* Iterator for built-in types.
- **Interpreter:** a small grammar evaluated as an expression tree — rare outside genuine DSL parsing; most systems implement the effect without a full framework.

---
*Related: [[CS Fundamentals/00 - Learning Path|CS Fundamentals Learning Path]] · [[CS Fundamentals/10 - Design Principles/Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] · [[CS Fundamentals/10 - Design Principles/Creational Design Patterns - Full Code Deep Dive|Creational Design Patterns]] · [[CS Fundamentals/10 - Design Principles/Structural Design Patterns - Full Code Deep Dive|Structural Design Patterns]] · [[LLD/15 - Design a Text Editor/Design a Text Editor|Design a Text Editor]]*
