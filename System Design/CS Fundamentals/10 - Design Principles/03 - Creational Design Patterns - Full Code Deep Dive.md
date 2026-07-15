---
title: "Creational Design Patterns — Full Code Deep Dive"
aliases: [Factory Method, Abstract Factory, Builder, Singleton, Prototype]
tags: [system-design, cs-fundamentals, design-principles, design-patterns, go, creational]
status: reference-quality
---

# Creational Design Patterns — Full Code Deep Dive

> [!abstract] What you'll be able to do after this chapter
> For every one of the 5 creational patterns, see a complete, runnable "without the pattern" program, the exact same problem solved "with the pattern," and a one-line trick for telling this pattern apart from the one it's most often confused with.

> [!info] Companion to the cheat sheet
> [[CS Fundamentals/10 - Design Principles/02 - Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] gives you the one-paragraph recognition version of all 23 GoF patterns. This chapter (and its two Structural/Behavioral siblings) gives you real, complete Go programs for every single one — including Factory Method, which also has a full system built around it in [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]] if you want the deeper, multi-requirement version.

---

## Factory Method

> [!example] Layman
> Ordering "a coffee" without specifying the machine — the barista (factory) picks the concrete steps.

**Trigger:** "create the right type without the caller choosing the concrete class"
**When to use:** the exact concrete type isn't known until runtime, and callers should depend only on the abstract type.

> [!tip] Identification trick
> If the requirement is "give me a `Foo`, I don't care exactly which kind" → Factory Method. If it's "give me a whole *matching set* of `Foo` + `Bar` + `Baz` that must stay consistent with each other" → that's the next pattern, Abstract Factory, not this one.

**Without the pattern:**

```go
package main

import "fmt"

type EmailNotifier struct{}

func (EmailNotifier) Send(msg string) { fmt.Println("email:", msg) }

type SMSNotifier struct{}

func (SMSNotifier) Send(msg string) { fmt.Println("sms:", msg) }

func main() {
	kind := "sms"
	msg := "server down"

	// caller must know about every concrete type and branch itself
	if kind == "email" {
		n := EmailNotifier{}
		n.Send(msg)
	} else if kind == "sms" {
		n := SMSNotifier{}
		n.Send(msg)
	}
	// adding "push" later means finding and editing every call site that does this branching
}

// Output:
// sms: server down
```

**With Factory Method:**

```go
package main

import "fmt"

type Notifier interface {
	Send(msg string)
}

type EmailNotifier struct{}

func (EmailNotifier) Send(msg string) { fmt.Println("email:", msg) }

type SMSNotifier struct{}

func (SMSNotifier) Send(msg string) { fmt.Println("sms:", msg) }

type PushNotifier struct{}

func (PushNotifier) Send(msg string) { fmt.Println("push:", msg) }

// Factory Method: one function owns "which concrete type to build"
func NewNotifier(kind string) Notifier {
	switch kind {
	case "email":
		return EmailNotifier{}
	case "sms":
		return SMSNotifier{}
	case "push":
		return PushNotifier{}
	default:
		return nil
	}
}

func main() {
	n := NewNotifier("sms")
	n.Send("server down")
	// caller never imports EmailNotifier/SMSNotifier directly — only Notifier + NewNotifier
}

// Output:
// sms: server down
```

> [!info] Full system-design depth
> [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]] builds a real `PricingStrategyFactory` inside a full multi-requirement system — read it once the shape above is comfortable.

> [!warning] Tradeoffs
> A new concrete type means editing the factory function — fine, that's the *one* place allowed to know about every concrete type; everywhere else only depends on the interface.

---

## Abstract Factory

> [!example] Layman
> Ordering a "meal combo" where burger, fries, and drink are all picked to match one theme — the whole family swaps together, never piece by piece.

**Trigger:** "guarantee several related objects come from the same consistent family"
**When to use:** a UI toolkit's light vs. dark theme, where a light button must never end up paired with a dark checkbox.

> [!tip] Identification trick
> Count how many related objects need to be created *together* and stay consistent. One object → Factory Method. A whole matched family (2 or more objects that must never be mismatched) → Abstract Factory.

**Without the pattern:**

```go
package main

import "fmt"

type LightButton struct{}

func (LightButton) Render() string { return "light button" }

type DarkButton struct{}

func (DarkButton) Render() string { return "dark button" }

type LightCheckbox struct{}

func (LightCheckbox) Render() string { return "light checkbox" }

type DarkCheckbox struct{}

func (DarkCheckbox) Render() string { return "dark checkbox" }

func RenderUIBad(dark bool) {
	if dark {
		btn := DarkButton{}
		chk := LightCheckbox{} // BUG: mismatched theme — compiles fine, wrong at runtime
		fmt.Println(btn.Render(), chk.Render())
	}
}

func main() {
	RenderUIBad(true) // nothing in the type system prevents this mismatch
}

// Output:
// dark button light checkbox
```

**With Abstract Factory:**

```go
package main

import "fmt"

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

func main() {
	RenderUI(DarkFactory{}) // impossible to get a mismatched pair — the factory is the family
}

// Output:
// dark button dark checkbox
```

> [!warning] Tradeoffs
> Adding a new product to the family (a `Scrollbar`) means touching **every** concrete factory, not just one. **Go note:** less common than in classic OOP — Go's composition-over-inheritance style often reaches for a simple `Config` struct or a set of constructor functions grouped by a shared prefix instead of a full Abstract Factory hierarchy.

---

## Builder

> [!example] Layman
> Ordering a custom sandwich one layer at a time, instead of being forced to list all 15 possible ingredients — most left blank — on one giant order form.

**Trigger:** "too many optional constructor parameters"
**When to use:** a constructor would otherwise need many optional parameters, most of which are only occasionally set.

> [!tip] Identification trick
> Count the optional parameters a constructor would need. 4 or more optional params, most left at defaults most of the time → Builder. 2-3 fields → a plain struct literal is simpler; don't reach for Builder yet.

**Without the pattern:**

```go
package main

import (
	"fmt"
	"time"
)

type Server struct {
	port     int
	tls      bool
	cert     string
	timeout  time.Duration
	maxConns int
	logLevel string
}

func NewServerBad(port int, tls bool, cert string, timeoutSec int, maxConns int, logLevel string) *Server {
	return &Server{port, tls, cert, time.Duration(timeoutSec) * time.Second, maxConns, logLevel}
}

func main() {
	srv := NewServerBad(8080, true, "cert.pem", 30, 100, "info")
	// what's 30? what's 100? order matters and is invisible at the call site
	fmt.Printf("port=%d tls=%v timeout=%v\n", srv.port, srv.tls, srv.timeout)
}

// Output:
// port=8080 tls=true timeout=30s
```

**With Builder (Go's idiom: functional options):**

```go
package main

import (
	"fmt"
	"time"
)

type Server struct {
	port    int
	tls     bool
	cert    string
	timeout time.Duration
}

type ServerOption func(*Server)

func WithPort(p int) ServerOption { return func(s *Server) { s.port = p } }
func WithTLS(cert string) ServerOption {
	return func(s *Server) { s.tls = true; s.cert = cert }
}
func WithTimeout(d time.Duration) ServerOption { return func(s *Server) { s.timeout = d } }

func NewServer(opts ...ServerOption) *Server {
	s := &Server{port: 80, timeout: 30 * time.Second} // sane defaults
	for _, opt := range opts {
		opt(s)
	}
	return s
}

func main() {
	srv := NewServer(WithPort(8080), WithTLS("cert.pem"), WithTimeout(30*time.Second))
	// self-documenting at the call site, any subset of options, any order
	fmt.Printf("port=%d tls=%v timeout=%v\n", srv.port, srv.tls, srv.timeout)
}

// Output:
// port=8080 tls=true timeout=30s
```

> [!warning] Tradeoffs
> More boilerplate (one function per option) than a plain struct literal — not worth it for a 2-3-field struct. Builder/functional-options earns its keep once optional parameters genuinely multiply.

---

## Singleton

> [!example] Layman
> A country's central bank — not "hard to make more of," but genuinely *shouldn't* have more than one, since two would issue conflicting currency decisions.

**Trigger:** "exactly one instance, shared everywhere, safe under concurrent access"
**When to use:** a shared, stateless config or logger being the legitimate case.

> [!tip] Identification trick
> Ask: "would having *two* of these actually produce an incorrect result, not just waste memory?" If yes → Singleton. If it's just expensive to build and you already have one lying around to copy → that's Prototype, not Singleton.

**Without the pattern:**

```go
package main

import (
	"fmt"
	"sync"
)

type Config struct{ Value string }

func loadConfig() *Config { return &Config{Value: "loaded"} }

var config *Config

func GetConfigBad() *Config {
	if config == nil {
		config = loadConfig() // RACE: two goroutines can both see nil, both load, both assign
	}
	return config
}

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_ = GetConfigBad() // run with `go run -race` to see the reported data race
		}()
	}
	wg.Wait()
	fmt.Println(GetConfigBad().Value)
}

// Output:
// loaded
//
// (functionally correct here because the race window is tiny and loadConfig is cheap —
// `go run -race` still flags a real, reportable data race on the "config" variable)
```

**With Singleton (`sync.Once`):**

```go
package main

import (
	"fmt"
	"sync"
)

type Config struct{ Value string }

func loadConfig() *Config { return &Config{Value: "loaded"} }

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

func main() {
	var wg sync.WaitGroup
	for i := 0; i < 5; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			_ = GetConfig() // safe under `go run -race` — init happens exactly once, guaranteed
		}()
	}
	wg.Wait()
	fmt.Println(GetConfig().Value)
}

// Output:
// loaded
```

> [!warning] Tradeoffs — the controversial part, stated honestly
> Hidden global state makes unit tests unable to swap in a fake instance, and can mask a real concurrency bug if the "single" instance isn't actually thread-safe internally once other code starts mutating it. A shared, stateless config or logger is a legitimate use; a mutable global "god object" usually isn't.

---

## Prototype

> [!example] Layman
> Photocopying a filled-out form instead of writing a blank one from memory each time, then correcting only the fields that differ.

**Trigger:** "clone an existing, expensive-to-build object instead of rebuilding"
**When to use:** building an object from scratch is expensive (heavy initialization, deep config), but a similar object already exists to copy from.

> [!tip] Identification trick
> Ask: "is building this from scratch measurably expensive, and do I already have a similar one sitting around to copy?" If yes → Prototype. If the concern is "there must never be more than one" rather than "building it is slow" → that's Singleton, not this.

**Without the pattern:**

```go
package main

import "fmt"

type Document struct {
	Sections []string
	Styles   map[string]string
}

func parseTemplate() []string        { return []string{"Header", "Body", "Footer"} } // pretend: expensive
func loadStyles() map[string]string  { return map[string]string{"font": "Times"} }   // pretend: expensive

func NewLegalTemplate() *Document {
	return &Document{Sections: parseTemplate(), Styles: loadStyles()} // expensive: disk + parsing
}

func main() {
	doc1 := NewLegalTemplate() // expensive
	doc2 := NewLegalTemplate() // expensive AGAIN — nearly identical to doc1
	doc2.Sections[0] = "Custom Header"
	fmt.Println(doc1.Sections[0], doc2.Sections[0])
}

// Output:
// Header Custom Header
```

**With Prototype:**

```go
package main

import "fmt"

type Document struct {
	Sections []string
	Styles   map[string]string
}

func parseTemplate() []string       { return []string{"Header", "Body", "Footer"} }
func loadStyles() map[string]string { return map[string]string{"font": "Times"} }

func NewLegalTemplate() *Document {
	return &Document{Sections: parseTemplate(), Styles: loadStyles()}
}

func (d *Document) Clone() *Document {
	stylesCopy := make(map[string]string, len(d.Styles))
	for k, v := range d.Styles {
		stylesCopy[k] = v
	}
	sectionsCopy := append([]string(nil), d.Sections...)
	return &Document{Sections: sectionsCopy, Styles: stylesCopy}
}

func main() {
	template := NewLegalTemplate()     // pay the expensive cost ONCE
	doc1 := template.Clone()           // cheap
	doc2 := template.Clone()           // cheap
	doc2.Sections[0] = "Custom Header" // only doc2 changes — template itself untouched
	fmt.Println(template.Sections[0], doc1.Sections[0], doc2.Sections[0])
}

// Output:
// Header Header Custom Header
```

> [!warning] Tradeoffs
> Every mutable field (slice, map, pointer) must be explicitly deep-copied in `Clone()`, or clones silently share hidden state — a real, easy-to-get-wrong detail. **Go note:** Go's value semantics (`newObj := *existingObj` for a shallow copy) make this pattern almost invisible for simple structs — an explicit `Clone()` is needed only when nested pointers/slices/maps make a shallow copy unsafe.

---

## Interview Q&A

> [!info] Leveled by seniority
> **Beginner:** "What's the difference between Factory Method and Abstract Factory?" — Factory Method creates one product; Abstract Factory creates a whole *family* of related products that must stay consistent with each other. **Intermediate:** "Why is Go's functional-options pattern considered Builder, not something new?" — it constructs a complex object step by step, with each `With...` function acting as one builder step, avoiding both a bloated constructor and Go's lack of default parameters. **Senior:** "A team's Singleton config object is making unit tests flaky under `go test -race` — diagnose it." — expects checking for a non-atomic init pattern (a bare `if x == nil` check instead of `sync.Once`), the exact race the Singleton section above fixes. **Staff/Architect:** "When would you actively avoid Singleton even though 'exactly one instance' is technically true?" — expects recognizing that testability usually matters more than the marginal safety of enforcing single-instance-ness at the type level; dependency injection of a single, shared instance (created once at startup, passed explicitly) gets the same practical benefit without the hidden global state.

## Summary / Cheat Sheet

| Pattern | Trick | Complete code above |
|---|---|---|
| Factory Method | One product, caller doesn't pick the concrete type | ✅ |
| Abstract Factory | A *matched family* of 2+ objects that must never mismatch | ✅ |
| Builder | 4+ optional constructor params → functional options in Go | ✅ |
| Singleton | Two instances would be *incorrect*, not just wasteful → `sync.Once` | ✅ |
| Prototype | Expensive to build from scratch, cheap to clone an existing one | ✅ |

---
*Related: [[CS Fundamentals/00 - Learning Path|CS Fundamentals Learning Path]] · [[CS Fundamentals/10 - Design Principles/02 - Design Patterns Cheat Sheet|Design Patterns Cheat Sheet]] · [[CS Fundamentals/10 - Design Principles/04 - Structural Design Patterns - Full Code Deep Dive|Structural Design Patterns]] · [[CS Fundamentals/10 - Design Principles/05 - Behavioral Design Patterns - Full Code Deep Dive|Behavioral Design Patterns]] · [[LLD/01 - Design a Parking Lot/Design a Parking Lot|Design a Parking Lot]]*
