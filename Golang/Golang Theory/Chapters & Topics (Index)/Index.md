---
title: Go Theory Mastery — Interview Curriculum
aliases: [Go Theory Index, Chapters & Topics]
tags: [golang, interview-prep, theory, index]
status: in-progress
chapters-total: 15
---

# 🧠 Go Theory Mastery — Interview Curriculum
*(Metro Edition)*

> [!abstract] Format
> **15 chapters, theory-only.** Read on the metro (~1 hr/day).
> **How we'll run it:** ask for notes on a chapter → read → ask to be quizzed interview-style → one question at a time, rated (good / wrong / model answer), difficulty ramps up → next chapter.
> **Practice tasks** are coded separately — the chapter notes themselves stay pure theory.

> [!warning] Ordering rule
> Chapters are ordered by **dependency** — do them in sequence. **Concurrency (Ch 5–8) is the core** — give it the most passes.

---

## 📊 Progress Tracker

| # | Chapter | Status | Notes |
|---|---|---|---|
| 1 | [[Chapter 1 - Why Go\|Why Go (and when *not* to use it)]] | ✅ Notes done | |
| 2 | [[Chapter 2 - Execution Model, Tooling & Terminology\|Execution model, tooling & core terminology]] | ✅ Notes done | |
| 3 | [[Chapter 3 - Types, Memory & Data Structures\|Types, memory & data structures]] | ✅ Notes done | self-generated |
| 4 | [[Chapter 4 - Interfaces & Method Sets\|Interfaces & method sets]] | ✅ Notes done | self-generated |
| 5 | [[Chapter 5 - Goroutines & the Runtime Scheduler\|Goroutines & the runtime scheduler]] | ✅ Notes done | 🔥 core |
| 6 | [[Chapter 6 - Channels\|Channels]] | ⬜ Not started | 🔥 core |
| 7 | [[Chapter 7 - Mutexes, Locks & sync-atomic\|Mutexes, locks & the sync/atomic toolkit]] | ⬜ Not started | 🔥 core |
| 8 | [[Chapter 8 - Memory Model & Data Races\|The Go memory model & data races]] | ⬜ Not started | 🔥 core |
| 9 | [[Chapter 9 - Context & Cancellation\|context & cancellation]] | ⬜ Not started | |
| 10 | [[Chapter 10 - Errors, Panic, Defer, Recover\|Errors, panic, defer, recover]] | ⬜ Not started | |
| 11 | [[Chapter 11 - Memory Management, Escape Analysis & GC\|Memory management, escape analysis & the GC]] | ⬜ Not started | |
| 12 | [[Chapter 12 - Performance & Profiling Theory\|Performance & profiling theory]] | ⬜ Not started | |
| 13 | [[Chapter 13 - Generics\|Generics (1.18+)]] | ⬜ Not started | |
| 14 | [[Chapter 14 - Idioms, Project Structure & Stdlib\|Idioms, project structure & stdlib essentials]] | ⬜ Not started | |
| 15 | [[Chapter 15 - Classic Gotchas & Trick Questions\|Classic gotchas & trick questions]] | ⬜ Not started | rapid-fire round |

> [!tip] Legend
> ⬜ Not started · 🟡 In progress · ✅ Notes done · 🎯 Quizzed & solid

---

## Chapter 1 — Why Go (and when *not* to use it)
The classic opener. Design philosophy (simplicity, fast compilation, built-in concurrency, static binaries), what problem Go was built to solve at Google, and honest tradeoffs.
- Go vs Java, Python, C++, Rust, Node — on concurrency model, GC, performance, ergonomics, ecosystem.
- When Go is the wrong choice (heavy CPU/SIMD, rich generics-era before 1.18, GUI, ML).
- "Share memory by communicating" as a design stance.

> [!info] Why it matters
> Almost every interview opens here; a crisp, tradeoff-aware answer signals seniority.

> [!example] Practice task
> Write a 90-second spoken pitch for "why we chose Go for this service" and a 90-second "why we didn't."

---

## Chapter 2 — Execution model, tooling & core terminology
The vocabulary you're expected to use fluently.
- Compilation to a single static binary, cross-compilation, the toolchain (`go build/test/vet/mod`).
- Static typing, garbage collection, no runtime VM.
- Terms: package, module, exported vs unexported, `iota`, blank identifier, zero value, rune, byte, the `init` function, build tags.

> [!info] Why it matters
> Misusing terminology reads as "resume Go, not real Go."

> [!example] Practice task
> Explain each term in one sentence without looking.

---

## Chapter 3 — Types, memory & data structures
- Value vs reference semantics (what's copied on assignment/function call).
- Arrays vs slices: the slice header (ptr/len/cap), how `append` grows, aliasing surprises.
- Maps: unordered, not concurrency-safe, nil-map write panic.
- Strings vs `[]byte` vs `rune`; UTF-8 encoding.
- Structs, embedding, pointers, when to use a pointer vs value.

> [!info] Why it matters
> Slice/map internals are the most common "trick" theory questions.

> [!example] Practice task
> Predict output of 3 slice-aliasing/append snippets, then run them.

---

## Chapter 4 — Interfaces & method sets
- Interface internals: the (type, value) pair; what "nil interface" really means.
- The typed-nil gotcha (a non-nil interface holding a nil pointer).
- Pointer vs value receivers and how they affect method sets.
- Type assertions, type switches, `any`/empty interface.
- "Accept interfaces, return structs"; define interfaces at the consumer.

> [!info] Why it matters
> The typed-nil trap and receiver rules are staple senior questions.

> [!example] Practice task
> Reproduce the typed-nil-in-interface bug and explain it.

---

## Chapter 5 — Goroutines & the runtime scheduler 🔥
- What a goroutine actually is (not an OS thread); tiny growable stacks.
- The **GMP model**: Goroutines, Machines (OS threads), Processors (logical, `GOMAXPROCS`).
- Cooperative scheduling + asynchronous preemption (since 1.14); work-stealing.
- Goroutine lifecycle and leaks; why "start a goroutine, forget its exit" is a bug.

> [!info] Why it matters
> GMP is *the* Go-specific systems question. Owning it separates you instantly.

> [!example] Practice task
> Draw the GMP diagram from memory and explain what happens on a blocking syscall.

---

## Chapter 6 — Channels 🔥
- Buffered vs unbuffered semantics; blocking behavior.
- The channel axioms: nil channel blocks forever; recv on closed → zero + `ok=false`; send on closed **panics**; close of closed **panics**; closing is the sender's job.
- Directional channels, `select`, `default`, timeouts.
- Channel-based patterns (pipeline, fan-in/out, done-signal) — theory only.

> [!info] Why it matters
> The axioms are asked verbatim; the "why sender closes" reasoning shows depth.

> [!example] Practice task
> Code one program per axiom that demonstrates it.

---

## Chapter 7 — Mutexes, locks & the sync/atomic toolkit 🔥
- `sync.Mutex` vs `sync.RWMutex`; when RWMutex actually helps.
- `sync.Once`, `WaitGroup`, `Cond`, `sync.Pool`, `sync.Map`.
- `sync/atomic` and lock-free counters; check-then-act / TOCTOU races.
- Channels vs mutexes — how to choose; deadlock, livelock, starvation.

> [!info] Why it matters
> The "mutex or channel here?" judgment call is a senior signal; TOCTOU is a common trap.

> [!example] Practice task
> Implement a thread-safe cache first with Mutex, then RWMutex, then compare with atomic where possible.

---

## Chapter 8 — The Go memory model & data races 🔥
- Happens-before relationships; what synchronization actually guarantees.
- What a data race is vs a race condition (not the same thing).
- The race detector: what it can and can't prove.
- Why unsynchronized reads/writes are undefined even for "harmless" flags.

> [!info] Why it matters
> Distinguishing data race vs race condition is a favorite discriminator question.

> [!example] Practice task
> Write a data race, catch it with `-race`, then a race condition that `-race` won't catch.

---

## Chapter 9 — context & cancellation
- `context.Context` purpose: cancellation, deadlines, request-scoped values.
- `WithCancel` / `WithTimeout` / `WithDeadline` / `WithValue`; propagation trees.
- Rules: context as first param, don't store in structs, don't pass nil.
- How cancellation flows to goroutines and how leaks happen without it.

> [!info] Why it matters
> Every backend Go role expects fluent context reasoning.

> [!example] Practice task
> Build a cancellable pipeline where cancel propagates to all stages.

---

## Chapter 10 — Errors, panic, defer, recover
- The `error` interface; wrapping with `%w`, `errors.Is`, `errors.As`.
- Sentinel vs typed vs wrapped errors — when each.
- `defer` semantics (LIFO, arg evaluation time, defer-in-loop trap).
- `panic`/`recover`: when panic is legitimate, why it's not exceptions.

> [!info] Why it matters
> Error-handling philosophy questions test whether you think in Go or in Java/Python.

> [!example] Practice task
> Build an error chain and inspect it with `Is`/`As`.

---

## Chapter 11 — Memory management, escape analysis & the GC
- Stack vs heap; what forces a value to escape.
- Escape analysis (theory of `-gcflags=-m`).
- Go's GC: concurrent tri-color mark-and-sweep, non-generational, non-compacting.
- `GOGC`, `GOMEMLIMIT` (1.19+); GC pressure and how allocations drive it.

> [!info] Why it matters
> "Walk me through Go's GC" is a standard performance-round question.

> [!example] Practice task
> Find 3 escaping variables via `-gcflags=-m` and explain each.

---

## Chapter 12 — Performance & profiling theory
- Profiling types: CPU, heap, goroutine, block, mutex.
- Benchmarking concepts (`testing.B`, allocations/op, `benchstat`).
- Inlining, common allocation pitfalls, interface boxing cost, `[]byte`↔`string` conversions.
- The senior reflex: profile before optimizing.

> [!info] Why it matters
> Lets you talk performance concretely instead of hand-waving.

> [!example] Practice task
> Profile a hot function with pprof and narrate the flame graph.

---

## Chapter 13 — Generics (1.18+)
- Type parameters, constraints, `comparable`, constraint interfaces.
- When generics help vs when interfaces are better; the tradeoffs and costs.
- What Go deliberately left out and why.

> [!info] Why it matters
> Newer, and interviewers probe whether you use it judiciously.

> [!example] Practice task
> Write a generic `Map`/`Filter` and a generic constraint of your own.

---

## Chapter 14 — Idioms, project structure & stdlib essentials
- Composition over inheritance; useful zero values; functional-options pattern.
- Table-driven tests; `internal/` packages; `cmd/` layout — without over-engineering.
- Stdlib theory: `io.Reader/Writer`, `net/http` server model, `encoding/json` behavior.

> [!info] Why it matters
> Idiom fluency is what makes code review–style questions go smoothly.

> [!example] Practice task
> Refactor a small API to functional options + table-driven tests.

---

## Chapter 15 — Classic gotchas & trick questions
The rapid-fire round.
- Loop variable capture and the Go 1.22 per-iteration change.
- `nil` map writes, slice aliasing after `append`, `defer` in loops.
- Interface `nil` comparison (typed nil), map iteration order, closing channels twice.
- `time.After` in loops, unbuffered-channel deadlocks.

> [!info] Why it matters
> These are the fast filters interviewers use; knowing them cold is free points.

> [!example] Practice task
> Build a personal "gotcha flashcard" list — one line each, predicted output + reason.

---

## 🔁 How to use this

> [!tip] Workflow
> - **One chapter's notes at a time.** Read on the metro, then ask to be quizzed.
> - **Quiz loop:** question → answer → rated (good / wrong / model answer) → difficulty ramps → next question. When a chapter feels solid, move on.
> - **Weight Ch 5–8 heavily** — re-quiz them near the end before applications.
> - **Keep a one-line takeaway per chapter** as a revision sheet.

---

*Related: [[Golang]] · [[Interview Questions]]*
