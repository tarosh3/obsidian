---
title: "Chapter 1 — Why Go (and when not to use it)"
aliases: [Chapter 1, Why Go]
tags: [golang, interview-prep, theory, chapter-1]
chapter: 1
status: notes-ready
---

# Why Go — and When Not To
*The interview opener. A crisp, tradeoff-aware answer here signals seniority instantly.*

> [!abstract] One-line answer
> Go was built for **networked, concurrent, multicore server software**. It gives Python-like productivity with C-like performance, first-class concurrency, and a single static binary that's trivial to deploy — at the cost of some expressiveness (late generics, verbose errors).

---

## 1. Why Google built it (the origin story)
Created at Google in **2007** (Griesemer, Pike, Thompson), open-sourced **2009**, v1.0 in **2012**. It was a reaction to three real pains of large-scale C++/Java server development:

- **Slow builds** — huge C++ codebases took minutes to compile, killing the feedback loop.
- **Hard concurrency** — hardware went multicore, but threads/locks were painful and error-prone.
- **Runaway complexity** — dependency and code-scale sprawl across thousands of engineers.

Go's answer: keep the **performance of a compiled language**, add **fast builds**, **built-in concurrency**, and **ruthless simplicity**.

---

## 2. The design pillars (memorize these)

| Pillar | What it means |
|---|---|
| **Simplicity** | ~25 keywords, small spec, "one obvious way." Optimized for *reading* code, not writing it. |
| **Fast compilation** | Compiles to native code in seconds — near-scripting feedback loop. |
| **Built-in concurrency** | Goroutines + channels as language features, not a library bolt-on. |
| **Single static binary** | No runtime/VM/deps to ship. One file to deploy — container gold. |
| **Garbage collected** | Low-latency concurrent GC. Memory safety without manual management. |
| **Batteries + tooling** | Strong stdlib, `gofmt`, `go test`, `go vet`, race detector built in. |

> [!quote] Core mantra
> **"Don't communicate by sharing memory; share memory by communicating."**
> Prefer passing data over channels to guarding shared state with locks. This is Go's concurrency worldview in one sentence — drop it in the interview.

---

## 3. Go vs the world (the money table)

| Language | Its strength over Go | Go's edge |
|---|---|---|
| **Java** | Mature ecosystem, rich JVM, huge libraries. | Faster startup, static binary, lower memory, simpler, no JVM warmup. |
| **Python** | Dev speed, ML/data-science ecosystem. | 10–100× faster, real parallelism (no GIL), static typing, one binary. |
| **C++** | Raw peak performance, fine-grained control, SIMD. | Memory-safe (GC), far simpler, compiles in seconds, safe concurrency. |
| **Rust** | Top performance + memory safety with no GC. | Much simpler, faster to learn, faster builds; GC removes borrow-checker friction. |
| **Node.js** | Async I/O, JS ecosystem, shared front/back language. | True multicore parallelism, static typing, one binary, better CPU-bound work. |

> [!tip] Memory hook
> Go's sweet spot = **"C's speed, Python's ease, built-in concurrency, one binary."**
> **Concurrency is not parallelism** (Pike): concurrency is *structuring* independent work; parallelism is *running* it at once. Go gives you the concurrency primitives; the runtime handles parallelism.

---

## 4. Right tool vs wrong tool

**Go SHINES for:**
- Network services, APIs, microservices
- Cloud infra & DevOps (Docker, Kubernetes are Go)
- CLI tools & developer tooling
- High-throughput, concurrent backends
- Anything needing dead-simple deploys

**Go is the WRONG choice for:**
- Heavy CPU / numeric / SIMD → C++, Rust
- ML & data science → Python
- Rich desktop / mobile GUI
- Hard real-time (GC pauses, tiny but present)
- Deeply generic-heavy domain modeling

> [!warning] Honest tradeoffs (name these, don't hide them)
> **Verbose error handling** (`if err != nil` everywhere) · **generics arrived late** (v1.18, 2022) · no exhaustive enums / rich metaprogramming · **GC makes it unfit for hard real-time**.
> Naming cons shows maturity — pretending Go is perfect reads as junior.

---

## 5. Say this in the interview

> [!quote]- "Why Go?"
> "For server-side, concurrent, networked systems it hits a rare sweet spot: compiled performance, a fast build/deploy loop, goroutines and channels as first-class concurrency, and a single static binary. I trade away some expressiveness — verbose errors, late generics — for simplicity and operability, which is the right trade for services at scale."

> [!quote]- "Why not Rust / Python / Java?"
> Anchor each on the **one axis** that matters: **Rust** = simplicity vs peak control; **Python** = performance + real parallelism; **Java** = startup, memory, deploy simplicity.

---
*Chapter 1 of 15 · Go Theory Interview Curriculum*

*Related: [[Index]] · Next → [[Chapter 2 - Execution Model, Tooling & Terminology]]*
