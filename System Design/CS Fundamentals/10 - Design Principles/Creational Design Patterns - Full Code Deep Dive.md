---
title: "Creational Design Patterns — Full Code Deep Dive"
aliases: [Factory Method, Abstract Factory, Builder, Singleton, Prototype]
tags: [system-design, cs-fundamentals, design-principles, design-patterns, go, creational]
status: reference-quality
---

# Creational Design Patterns — Full Code Deep Dive

> [!abstract] What you'll be able to do after this chapter
> For every creational pattern except Factory Method (which already has a full LLD chapter), see the exact bad code, feel exactly why it breaks, and see the exact refactor that fixes it — real, compilable-shaped Go, not pseudocode.

> [!info] Companion to the cheat sheet
> [[CS Fundamentals/10 - Design Principles/Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] gives you the one-paragraph recognition version of all 23 GoF patterns. This chapter (and its two Structural/Behavioral siblings) gives you the thing the cheat sheet explicitly said it wouldn't: a real "before" and "after" for the patterns that don't have a full LLD case study built around them.

---

## Factory Method — recap only

Already has a full, real case study: [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]]'s `PricingStrategyFactory`. **Layman:** ordering "a coffee" without specifying the machine — the barista (factory) picks the concrete steps. **When to use:** the exact concrete type isn't known until runtime, and callers should depend only on the abstract type. Nothing to add here that chapter doesn't already cover in full.

---

## Abstract Factory

> [!example] Layman
> Ordering a "meal combo" where burger, fries, and drink are all picked to match one theme — the whole family swaps together, never piece by piece.

**When to use:** you need to guarantee several related objects come from the same consistent family — a UI toolkit's light vs. dark theme, where a light button must never end up paired with a dark checkbox.

```go
// BEFORE — caller assembles the family by hand; nothing stops a mismatch
type LightButton struct{}
type DarkButton struct{}
type LightCheckbox struct{}
type DarkCheckbox struct{}

func RenderUIBad(dark bool) {
	if dark {
		btn := DarkButton{}
		chk := LightCheckbox{} // BUG: mismatched theme — compiles fine, looks wrong at runtime
		_ = btn
		_ = chk
	}
}
```

```go
// AFTER — Abstract Factory guarantees a matched family, by construction
type Button interface{ Render() string }
type Checkbox interface{ Render() string }

type UIFactory interface {
	CreateButton() Button
	CreateCheckbox() Checkbox
}

type LightButton struct{}
func (LightButton) Render() string { return "light button" }
type LightCheckbox struct{}
func (LightCheckbox) Render() string { return "light checkbox" }

type LightFactory struct{}
func (LightFactory) CreateButton() Button     { return LightButton{} }
func (LightFactory) CreateCheckbox() Checkbox { return LightCheckbox{} }

type DarkButton struct{}
func (DarkButton) Render() string { return "dark button" }
type DarkCheckbox struct{}
func (DarkCheckbox) Render() string { return "dark checkbox" }

type DarkFactory struct{}
func (DarkFactory) CreateButton() Button     { return DarkButton{} }
func (DarkFactory) CreateCheckbox() Checkbox { return DarkCheckbox{} }

func RenderUI(f UIFactory) {
	btn := f.CreateButton()
	chk := f.CreateCheckbox()
	fmt.Println(btn.Render(), chk.Render()) // ALWAYS matched — the factory enforces it
}
```

> [!warning] Tradeoffs
> Adding a new product to the family (a `Scrollbar`) means touching **every** concrete factory, not just one. **Go note:** less common than in classic OOP — Go's composition-over-inheritance style often reaches for a simple `Config` struct or a set of constructor functions grouped by a shared prefix instead of a full Abstract Factory hierarchy.

---

## Builder

> [!example] Layman
> Ordering a custom sandwich one layer at a time, instead of being forced to list all 15 possible ingredients — most left blank — on one giant order form.

**When to use:** a constructor would otherwise need many optional parameters, most of which are only occasionally set.

```go
// BEFORE — telescoping constructor, unreadable and error-prone at the call site
func NewServerBad(port int, tls bool, cert string, timeoutSec int, maxConns int, logLevel string) *Server {
	return &Server{port, tls, cert, time.Duration(timeoutSec) * time.Second, maxConns, logLevel}
}

srv := NewServerBad(8080, true, "cert.pem", 30, 100, "info")
// what's 30? what's 100? order matters and is invisible at the call site
```

```go
// AFTER — functional options, Go's idiomatic Builder
type Server struct {
	port    int
	tls     bool
	cert    string
	timeout time.Duration
}

type ServerOption func(*Server)

func WithPort(p int) ServerOption       { return func(s *Server) { s.port = p } }
func WithTLS(cert string) ServerOption  { return func(s *Server) { s.tls = true; s.cert = cert } }
func WithTimeout(d time.Duration) ServerOption { return func(s *Server) { s.timeout = d } }

func NewServer(opts ...ServerOption) *Server {
	s := &Server{port: 80, timeout: 30 * time.Second} // sane defaults
	for _, opt := range opts {
		opt(s)
	}
	return s
}

srv := NewServer(WithPort(8080), WithTLS("cert.pem"), WithTimeout(30*time.Second))
// self-documenting at the call site, any subset of options, any order
```

> [!warning] Tradeoffs
> More boilerplate (one function per option) than a plain struct literal — not worth it for a 2-3-field struct. Builder/functional-options earns its keep once optional parameters genuinely multiply.

---

## Singleton

> [!example] Layman
> A country's central bank — not "hard to duplicate," but genuinely *shouldn't* have more than one, since two would issue conflicting currency decisions.

**When to use:** exactly one instance must exist, globally accessible — a shared, stateless config or logger being the legitimate case.

```go
// BEFORE — global var, race condition on first concurrent access
var config *Config

func GetConfigBad() *Config {
	if config == nil {
		config = loadConfig() // RACE: two goroutines can both see nil, both load, both assign
	}
	return config
}
```

```go
// AFTER — sync.Once guarantees exactly-once init, safe under concurrency
var (
	config     *Config
	configOnce sync.Once
)

func GetConfig() *Config {
	configOnce.Do(func() {
		config = loadConfig() // guaranteed to run exactly once, even under concurrent first access
	})
	return config
}
```

> [!warning] Tradeoffs — the controversial part, stated honestly
> Hidden global state makes unit tests unable to swap in a fake instance, and can mask a real concurrency bug if the "single" instance isn't actually thread-safe internally once other code starts mutating it. A shared, stateless config or logger is a legitimate use; a mutable global "god object" usually isn't.

---

## Prototype

> [!example] Layman
> Photocopying a filled-out form instead of writing a blank one from memory each time, then correcting only the fields that differ.

**When to use:** building an object from scratch is expensive (heavy initialization, deep config), but a similar object already exists to copy from.

```go
// BEFORE — every new similar object pays the full, expensive init cost again
type Document struct {
	Sections []string
	Styles   map[string]string
}

func NewLegalTemplate() *Document {
	return &Document{Sections: parseTemplate(), Styles: loadStyles()} // expensive: disk + parsing
}

doc1 := NewLegalTemplate() // expensive
doc2 := NewLegalTemplate() // expensive AGAIN — nearly identical to doc1
```

```go
// AFTER — build the expensive template once, clone it cheaply thereafter
func (d *Document) Clone() *Document {
	stylesCopy := make(map[string]string, len(d.Styles))
	for k, v := range d.Styles {
		stylesCopy[k] = v
	}
	sectionsCopy := append([]string(nil), d.Sections...)
	return &Document{Sections: sectionsCopy, Styles: stylesCopy}
}

template := NewLegalTemplate() // pay the expensive cost ONCE
doc1 := template.Clone()       // cheap
doc2 := template.Clone()       // cheap
doc2.Sections[0] = "Custom clause" // only doc2 changes — template itself untouched
```

> [!warning] Tradeoffs
> Every mutable field (slice, map, pointer) must be explicitly deep-copied in `Clone()`, or clones silently share hidden state — a real, easy-to-get-wrong detail. **Go note:** Go's value semantics (`newObj := *existingObj` for a shallow copy) make this pattern almost invisible for simple structs — an explicit `Clone()` is needed only when nested pointers/slices/maps make a shallow copy unsafe.

---

## Interview Q&A

> [!info] Leveled by seniority
> **Beginner:** "What's the difference between Factory Method and Abstract Factory?" — Factory Method creates one product; Abstract Factory creates a whole *family* of related products that must stay consistent with each other. **Intermediate:** "Why is Go's functional-options pattern considered Builder, not something new?" — it constructs a complex object step by step, with each `With...` function acting as one builder step, avoiding both a bloated constructor and Go's lack of default parameters. **Senior:** "A team's Singleton config object is making unit tests flaky under `go test -race` — diagnose it." — expects checking for a non-atomic init pattern (a bare `if x == nil` check instead of `sync.Once`), the exact race the Singleton section above fixes. **Staff/Architect:** "When would you actively avoid Singleton even though 'exactly one instance' is technically true?" — expects recognizing that testability usually matters more than the marginal safety of enforcing single-instance-ness at the type level; dependency injection of a single, shared instance (created once at startup, passed explicitly) gets the same practical benefit without the hidden global state.

## Summary / Cheat Sheet

- **Factory Method:** full chapter — see [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Parking Lot]].
- **Abstract Factory:** guarantees a *matched family* of related objects — the fix for accidentally mixing themes/families.
- **Builder:** step-by-step construction for many optional parameters — Go's idiom is functional options.
- **Singleton:** exactly one instance, made safe under concurrency via `sync.Once` — controversial; weigh testability before reaching for it.
- **Prototype:** clone an existing, already-built object instead of paying expensive init again — must deep-copy mutable fields.

---
*Related: [[CS Fundamentals/00 - Learning Path|CS Fundamentals Learning Path]] · [[CS Fundamentals/10 - Design Principles/Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] · [[CS Fundamentals/10 - Design Principles/Structural Design Patterns - Full Code Deep Dive|Structural Design Patterns]] · [[CS Fundamentals/10 - Design Principles/Behavioral Design Patterns - Full Code Deep Dive|Behavioral Design Patterns]] · [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]]*
