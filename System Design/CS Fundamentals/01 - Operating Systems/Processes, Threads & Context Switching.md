---
title: "Processes, Threads & Context Switching"
aliases: [Processes vs Threads, Context Switching]
tags: [system-design, cs-fundamentals, operating-systems]
status: reference-quality
---

# Processes, Threads & Context Switching

> [!abstract] What you'll be able to do after this chapter
> Explain precisely what a process and a thread each own, why a thread context switch is cheaper than a process one (with the actual mechanism, not just "it's lighter"), and connect this straight to why Go's goroutines are cheaper still.

---

## 1. Why processes exist

Early computers ran one program at a time, start to finish, before loading the next (batch processing). The CPU sat idle constantly — every time a program waited on disk or network I/O, nothing else could run. The OS introduced the **process** to solve two problems at once: **time-sharing** (let the CPU switch between programs so I/O waits on one don't stall everything) and **isolation** (one program's bugs/crashes shouldn't corrupt another's memory).

A **process** is a running instance of a program with its own **virtual address space** (code, heap, stack, data), its own file descriptors, and an OS-tracked **Process Control Block (PCB)** — the PID, saved register state, memory maps, open file handles, and scheduling metadata. The hardware's **MMU (Memory Management Unit)** enforces isolation: one process genuinely cannot read or write another's memory without going through the OS explicitly (shared memory, pipes, etc.).

## 2. Why threads exist — the problem processes didn't solve

Processes solved isolation, but isolation has a cost: if a web server wants to handle 1,000 concurrent connections, spawning 1,000 *processes* means 1,000 separate address spaces, each needing its own copy of shared data (like an in-memory cache) — either duplicated per process (wasteful) or shared via slow, explicit IPC (inter-process communication).

A **thread** is a unit of execution *within* a process — it shares the process's address space (code, heap, global data) but has its own **stack** and its own saved CPU register state (program counter, stack pointer). Multiple threads in one process can read and write the same memory directly, which is fast — and is exactly why [[Glossary/Race Condition|race conditions]] and the need for [[Glossary/Thread-Safety|thread-safety]] exist at all: shared mutable memory, accessed concurrently, without coordination.

## 3. What a context switch actually does

A **context switch** happens when the CPU scheduler stops running one thread and starts running another. Mechanically:
1. Save the currently-running thread's CPU register state (including the program counter — "where was it in its code") into its Thread Control Block.
2. Load the next thread's previously-saved register state into the CPU.
3. Resume execution at exactly where that next thread left off.

```mermaid
sequenceDiagram
    participant CPU
    participant TCB_A as Thread A's saved state
    participant TCB_B as Thread B's saved state

    Note over CPU: Thread A running
    CPU->>CPU: Timer interrupt fires
    CPU->>TCB_A: save registers, program counter
    CPU->>TCB_B: load registers, program counter
    Note over CPU: Thread B now running, resumes exactly where it left off
```

## 4. Why a thread switch is cheaper than a process switch

This is the actual mechanism, not just a claim:

A **process** switch changes the entire virtual address space — every virtual-to-physical address mapping the CPU was relying on becomes invalid. The **TLB (Translation Lookaside Buffer)**, a small hardware cache of recent address translations, must be flushed (or, on hardware supporting address-space-tagged TLB entries, the old process's entries are simply no longer useful). Right after the switch, the new process suffers a burst of **TLB misses** as its mappings get re-cached from scratch — genuinely expensive. On top of that, the CPU's L1/L2 caches, warm with the old process's data and instructions, get evicted and refilled for the new process — **cache locality is lost.**

A **thread** switch within the *same* process changes none of that — the address space is identical, so the TLB and CPU caches stay valid. The switch is just swapping register state and stack pointer — a fraction of the cost.

> [!tip] This is exactly why goroutines are cheaper still
> Go's goroutines take this one step further: the Go runtime schedules goroutines **in user space**, without involving the kernel at all for most switches — no syscall, no kernel-level TCB save/restore. See [[Golang/Golang Theory/Chapter 5 - Goroutines & the Runtime Scheduler|Golang Theory Chapter 5]] for the full GMP model — the "cheap to switch" property described there is a direct consequence of the mechanics in this section, just moved one layer higher, out of the kernel's hands entirely.

## 5. Process states & the scheduler's role

```mermaid
stateDiagram-v2
    [*] --> New
    New --> Ready: admitted
    Ready --> Running: scheduler dispatches
    Running --> Ready: preempted (time slice expired)
    Running --> Waiting: blocks on I/O
    Waiting --> Ready: I/O completes
    Running --> Terminated: exits
    Terminated --> [*]
```

A **preemptive** scheduler can forcibly interrupt a running thread (via a timer interrupt) once its time slice expires, guaranteeing no single thread monopolizes the CPU. A **cooperative** scheduler relies on threads voluntarily yielding — simpler to reason about, but one badly-behaved thread can starve everything else (full scheduling algorithm depth is its own future chapter).

## 6. Alternatives & when NOT to use threads

**Multi-process instead of multi-threaded** (e.g. Chrome's one-process-per-tab): trades the memory/IPC overhead of separate address spaces for genuine fault isolation — a crashing tab can't corrupt the rest of the browser. The right call whenever isolation/fault-containment matters more than cheap communication — running untrusted plugin/extension code is a textbook case.

**CPU-bound parallel work in Python**: CPython's Global Interpreter Lock (GIL) means only one thread executes Python bytecode at a time, *regardless of core count* — threads help with I/O-bound concurrency (waiting frees the GIL) but give **zero** speedup for CPU-bound work. The fix is multiprocessing (separate processes, separate GILs, real parallelism), not more threads — a genuinely common "when NOT to use threads" interview trap.

## 7. Scaling: 1 thread to thousands

```mermaid
flowchart TD
    A["1 thread<br/>no switching overhead at all"] --> B["A handful of threads<br/>context-switch overhead<br/>negligible"]
    B --> C["Thousands of threads<br/>switch overhead AND memory<br/>(thread stacks) become real costs —<br/>the exact C10K problem"]
    C --> D["Many-core machines<br/>at high thread counts, CONTENTION<br/>on shared data structures becomes<br/>the bottleneck, not switching itself"]
```

At thread counts in the thousands, both the memory cost (each thread's stack, typically 1-8MB) and the accumulated context-switch overhead become real, measurable costs — precisely the motivation behind [[CS Fundamentals/01 - Operating Systems/I-O Models - Blocking, Non-Blocking, and Async|the I/O Models chapter's]] event-loop alternative. On many-core machines specifically, once thread count is high enough to keep every core busy, the bottleneck often shifts away from switching cost entirely and toward **contention** — many threads competing for the same locks or shared data structures, a genuinely different problem than switching overhead, requiring different fixes (finer-grained locking, lock-free structures, or reducing shared mutable state).

## 8. Failure scenarios

> [!bug] What actually happens
> - **A thread crashes:** takes down the **entire process**, since all threads share one address space — a single bad pointer dereference in one thread can corrupt or crash everything, unlike a process crash which stays isolated to that process alone. This is a real, meaningful reason to choose multi-process over multi-threaded for fault containment (Section 6).
> - **A deadlock between threads:** two threads each hold a lock the other needs — the process hangs rather than crashing, often requiring a hung-process investigation (thread dumps, stack traces) to diagnose, since there's no crash log pointing directly at the cause.
> - **A thread leak:** threads spawned but never cleaned up accumulate over time, eventually exhausting the OS's thread limit for the process — a slow-building failure that can look like an unrelated resource exhaustion until traced back to its actual source.

## 9. Monitoring

> [!info] What to watch
> **Thread count per process** — a steadily climbing count with no corresponding load increase is the direct signal of a thread leak. **Context-switch rate** — a high rate relative to available cores signals oversubscription (more runnable threads than cores can service concurrently). **CPU time per thread** — the direct tool for identifying one runaway thread consuming disproportionate CPU among many.

## 10. Common mistakes

> [!warning] Real, recurring errors
> 1. **Spawning a thread per request with no bound** — under load, this becomes an accidental self-inflicted resource-exhaustion event, since nothing caps how many threads can pile up.
> 2. **Using threads for CPU-bound parallelism in CPython without accounting for the GIL** — Section 6 covers this precisely; threads help I/O-bound concurrency, not CPU-bound work, in CPython specifically.
> 3. **Not using a thread pool** — paying full thread-creation cost repeatedly instead of reusing a bounded set of already-created threads, a real, avoidable overhead under any nontrivial request volume.

---

## 🎯 Interview follow-up Q&A

> [!info] Leveled by seniority
> **Beginner:** "What's the difference between a process and a thread?" — Section 1-2: isolated address space vs. shared address space within one process. **Intermediate:** "Why is a thread switch cheaper than a process switch?" — Section 4, TLB/cache validity preserved within the same address space. **Senior:** "A production service is hanging with no crash log — how do you diagnose it?" — expects suspecting a deadlock (Section 8) and reaching for thread dumps/stack traces as the actual diagnostic tool, not assuming a crash that never happened. **Staff:** "Design the concurrency model for a service handling both many concurrent I/O-bound requests and occasional CPU-heavy background jobs." — expects an event-loop or thread-pool model for the I/O-bound majority (per I/O Models), with CPU-heavy work explicitly isolated onto separate worker threads/processes so it can't stall the request-handling path. **Architect:** "How would you decide between a multi-threaded and multi-process architecture for a new service handling untrusted third-party plugin code?" — expects Section 6's answer prioritized correctly: fault isolation from a genuinely untrusted component outweighs the IPC/memory overhead cost, favoring multi-process specifically for that reason.

> [!quote]- "What's the actual difference between a process and a thread?"
> A process has its own isolated virtual address space, enforced by the MMU; a thread shares its parent process's address space with all sibling threads, owning only its own stack and register state.
>
> **Follow-up: "Why is a thread context switch cheaper than a process one?"**
> A process switch invalidates the CPU's address-translation cache (TLB) and evicts CPU cache locality, since the entire address space changed. A thread switch within the same process leaves the address space untouched — only registers and stack pointer need swapping.

> [!quote]- "How does the OS decide when to preempt a running thread?"
> A hardware timer fires an interrupt at a fixed interval (the scheduling quantum); the interrupt handler invokes the scheduler, which may choose to context-switch to a different ready thread rather than resuming the interrupted one — this is what makes preemptive scheduling preemptive, as opposed to relying on threads yielding voluntarily.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[Golang/Golang Theory/Chapter 5 - Goroutines & the Runtime Scheduler|Golang Theory Ch5 — Goroutines]] · [[Glossary/Thread-Safety|Thread-Safety]] · [[Glossary/Race Condition|Race Condition]]*
