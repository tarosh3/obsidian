---
title: "Design a Task Scheduler / Cron"
aliases: [Design a Task Scheduler, Design Cron, Cron LLD]
tags: [system-design, lld, case-study, golang, design-patterns, strategy-pattern, observer-pattern]
status: reference-quality
---

# Design a Task Scheduler / Cron

> [!abstract] What you'll be able to do after this chapter
> Combine Strategy (for computing "when's next") with Observer (for reacting to outcomes), and see why each due task runs in its own goroutine — a direct, necessary application of everything the Concurrency in LLD chapter established.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a task scheduler like cron — register tasks with a recurring schedule, have the scheduler trigger them at the right times, and support reacting to success/failure (e.g. alerting on repeated failures)."

## Step 2 — Requirement clarification

Support multiple schedule *shapes* (fixed interval, daily at a specific time) without touching the scheduler's core loop for each new one. Notify interested observers on task outcome. A slow task must never block the scheduler from checking or triggering *other* tasks.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~8 min) — one task, fixed interval, sequential loop, no goroutines.**
> ```go
> type Task struct {
>     Interval time.Duration
>     Run      func() error
>     nextRun  time.Time
> }
>
> for {
>     time.Sleep(time.Second)
>     if time.Now().After(task.nextRun) {
>         task.Run() // sequential — fine for ONE task
>         task.nextRun = time.Now().Add(task.Interval)
>     }
> }
> ```
> **Pattern used: none.** One task, one schedule shape, blocking execution — get the tick-check-run cycle correct before anything else.
>
> **Checkpoint 2 (~8 min) — multiple schedule shapes, refactor into Strategy.**
> ```go
> // Schedule — Strategy applied to TIME COMPUTATION, not an action.
> // The scheduler's loop only ever asks "what's next," never HOW.
> type Schedule interface {
>     NextRunTime(after time.Time) time.Time
> }
>
> type IntervalSchedule struct{ Interval time.Duration }
> func (s IntervalSchedule) NextRunTime(after time.Time) time.Time { return after.Add(s.Interval) }
> ```
> **Pattern used: Strategy.** Add `DailyAtSchedule` as a second implementation once this compiles — same interface, genuinely different computation.
>
> **Checkpoint 3 (~8 min) — Observer for outcomes.**
> ```go
> type TaskObserver interface {
>     OnSuccess(taskName string)
>     OnFailure(taskName string, err error)
> }
> ```
> **Pattern used: Observer.** Scheduler and Task never know what an observer does with a result — log it, page someone, increment a metric.
>
> **Checkpoint 4 (remaining time — this is the actual correctness point) — one goroutine per due task.**
> ```go
> // Each due task runs in its OWN goroutine — a slow task must never
> // block the loop from checking/triggering OTHER tasks.
> for _, t := range due {
>     go s.runTask(t, observersCopy)
> }
> ```
> **No new pattern — pure concurrency correctness.** Say explicitly why this matters: without it, a 10-second task delays every other task's due-check for those same 10 seconds if the scheduler's tick interval is, say, 1 second.
>
> **If you're short on time:** stop after Checkpoint 2. A scheduler with swappable `Schedule` strategies, even running tasks sequentially, is a strong, complete answer — describe Observer and the per-task-goroutine fix verbally as the next two layers, and be ready to explain WHY sequential execution is a real bug waiting to happen.

## Step 3 — The bad first draft (kept brief)

A `Scheduler` whose core loop branches on a schedule-type string ("interval" vs "daily") inline, computing "is this task due" differently per branch — the same Open/Closed shape covered repeatedly by now; adding a weekly or cron-expression schedule means editing the loop itself.

## Step 4 — Refactor: Strategy for "when's next," Observer for outcomes

**`Schedule`** becomes the abstraction — `NextRunTime(after time.Time) time.Time` — the scheduler's core loop only ever asks "what's your next run time," never *how* that's computed. This is Strategy applied to **time computation**, rather than an action, the same pattern shape reused in a new context.

**`TaskObserver`** gets notified after every execution (`OnSuccess`/`OnFailure`) — the scheduler and the task itself never know or care what an observer actually does with that information (log it, page someone, record a metric).

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: schedule.go
// ============================================================
package scheduler

import "time"

// Schedule replaced the schedule-type if/else chain from the bad
// first draft — the scheduler's loop only ever calls NextRunTime.
type Schedule interface {
	NextRunTime(after time.Time) time.Time
}

// IntervalSchedule runs a task every fixed duration.
type IntervalSchedule struct {
	Interval time.Duration
}

func (s IntervalSchedule) NextRunTime(after time.Time) time.Time {
	return after.Add(s.Interval)
}

// DailyAtSchedule runs once per day at a specific hour:minute.
type DailyAtSchedule struct {
	Hour, Minute int
}

func (s DailyAtSchedule) NextRunTime(after time.Time) time.Time {
	next := time.Date(after.Year(), after.Month(), after.Day(), s.Hour, s.Minute, 0, 0, after.Location())
	if !next.After(after) {
		next = next.AddDate(0, 0, 1) // already passed today — tomorrow instead
	}
	return next
}
```

```go
// ============================================================
// FILE: task.go
// ============================================================
package scheduler

import "time"

type Task struct {
	Name     string
	Schedule Schedule
	Run      func() error
	nextRun  time.Time
}
```

```go
// ============================================================
// FILE: observer.go
// ============================================================
package scheduler

// TaskObserver is notified after every task execution — Scheduler
// and Task never know or care what an observer does with this.
type TaskObserver interface {
	OnSuccess(taskName string)
	OnFailure(taskName string, err error)
}
```

```go
// ============================================================
// FILE: scheduler.go
// ============================================================
package scheduler

import (
	"sync"
	"time"
)

type Scheduler struct {
	mu        sync.Mutex
	tasks     []*Task
	observers []TaskObserver
	stopCh    chan struct{}
}

func NewScheduler() *Scheduler {
	return &Scheduler{stopCh: make(chan struct{})}
}

func (s *Scheduler) Register(task *Task) {
	s.mu.Lock()
	defer s.mu.Unlock()
	task.nextRun = task.Schedule.NextRunTime(time.Now())
	s.tasks = append(s.tasks, task)
}

func (s *Scheduler) Subscribe(o TaskObserver) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.observers = append(s.observers, o)
}

// Start runs the scheduling loop until Stop is called.
func (s *Scheduler) Start(tickInterval time.Duration) {
	ticker := time.NewTicker(tickInterval)
	defer ticker.Stop()

	for {
		select {
		case <-s.stopCh:
			return
		case now := <-ticker.C:
			s.checkAndRunDueTasks(now)
		}
	}
}

func (s *Scheduler) Stop() {
	close(s.stopCh)
}

func (s *Scheduler) checkAndRunDueTasks(now time.Time) {
	s.mu.Lock()
	var due []*Task
	for _, t := range s.tasks {
		if !now.Before(t.nextRun) {
			due = append(due, t)
			t.nextRun = t.Schedule.NextRunTime(now)
		}
	}
	observersCopy := append([]TaskObserver{}, s.observers...)
	s.mu.Unlock()

	// Each due task runs in its OWN goroutine — a slow task must
	// never block the loop from checking/triggering other tasks,
	// directly applying the Concurrency in LLD chapter's principles.
	for _, t := range due {
		go s.runTask(t, observersCopy)
	}
}

func (s *Scheduler) runTask(t *Task, observers []TaskObserver) {
	err := t.Run()
	for _, o := range observers {
		if err != nil {
			o.OnFailure(t.Name, err)
		} else {
			o.OnSuccess(t.Name)
		}
	}
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"errors"
	"fmt"
	"time"

	scheduler "example.com/scheduler"
)

type ConsoleObserver struct{}

func (c *ConsoleObserver) OnSuccess(taskName string) {
	fmt.Printf("✅ %s succeeded\n", taskName)
}
func (c *ConsoleObserver) OnFailure(taskName string, err error) {
	fmt.Printf("❌ %s failed: %v\n", taskName, err)
}

func main() {
	sched := scheduler.NewScheduler()
	sched.Subscribe(&ConsoleObserver{})

	sched.Register(&scheduler.Task{
		Name:     "cleanup-temp-files",
		Schedule: scheduler.IntervalSchedule{Interval: 2 * time.Second},
		Run: func() error {
			fmt.Println("running cleanup...")
			return nil
		},
	})

	sched.Register(&scheduler.Task{
		Name:     "flaky-sync-job",
		Schedule: scheduler.IntervalSchedule{Interval: 3 * time.Second},
		Run: func() error {
			return errors.New("upstream service timeout")
		},
	})

	go sched.Start(1 * time.Second)

	time.Sleep(7 * time.Second)
	sched.Stop()
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "Why does each due task run in its own goroutine instead of executing sequentially inside the loop?"
> A task that takes 10 seconds to run would otherwise delay every *other* task's due-check for those same 10 seconds if run inline — with the scheduler ticking, say, every second, that's several missed check cycles. Running each due task in its own goroutine keeps the scheduling loop itself fast and responsive, decoupled entirely from how long any individual task takes.

> [!quote]- "How would you add a cron-expression-style schedule (e.g. `0 */6 * * *`)?"
> One new struct implementing `Schedule`, parsing the cron expression and computing `NextRunTime` accordingly. Zero changes to `Scheduler` — it never knew or cared how `IntervalSchedule` or `DailyAtSchedule` computed their answers either.

> [!quote]- "What happens if `Stop()` is called while a task is still running in its goroutine?"
> `Stop()` only closes `stopCh`, which ends the scheduling *loop* — it doesn't cancel in-flight task goroutines. A production version would likely pass a `context.Context` into `Run` and cancel it on `Stop`, letting well-behaved tasks observe cancellation and exit cleanly — worth naming as a real gap in this simplified version if pushed on it.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/08 - Design a Notification System/Design a Notification System|Design a Notification System]]*
