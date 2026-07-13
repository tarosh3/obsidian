---
title: "Design YouTube (LLD)"
aliases: [Design YouTube LLD, YouTube View Count LLD]
tags: [system-design, lld, case-study, go, concurrency, observer-pattern]
status: reference-quality
---

# Design YouTube (LLD)

> [!abstract] What you'll be able to do after this chapter
> Recognize a "lost update" as a genuinely different race-condition shape from the check-then-act double-claim bug already fixed in earlier chapters, fix it with an atomic counter instead of a mutex, and model per-user view dedup within a time window.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design the core object model for YouTube — upload videos, record views, like videos, subscribe to channels, comment on videos."

## Step 2 — Requirements

**Functional:** upload a video, record a view, like a video, subscribe/unsubscribe to a channel, comment on a video.

**Non-functional:** view counting must stay correct under many concurrent viewers. Rewatching the same video repeatedly in a short window shouldn't inflate the view count arbitrarily — a real product rule, not just a performance concern. Subscribers should be notified when a channel uploads a new video.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~6 min) — the buggy view counter, deliberately.**
> ```go
> type Video struct {
>     ID    string
>     Views int
> }
>
> func (v *Video) RecordView() {
>     v.Views++ // <-- NOT atomic: read, increment, write as 3 separate steps
> }
> ```
> **Pattern used: none — a live bug, on purpose.** Narrate it precisely: *"this is a **lost update**, not a check-then-act race — two goroutines can both read the same `Views` value before either writes, and one increment silently disappears. That's a different bug shape than the seat-booking double-claim from earlier chapters, even though both are 'concurrency bugs.'"*
>
> **Checkpoint 2 (~8 min) — fix with an atomic counter, not a mutex.**
> ```go
> import "sync/atomic"
>
> type Video struct {
>     ID    string
>     views int64 // accessed ONLY via atomic ops
> }
>
> func (v *Video) RecordView() {
>     atomic.AddInt64(&v.views, 1)
> }
>
> func (v *Video) ViewCount() int64 {
>     return atomic.LoadInt64(&v.views)
> }
> ```
> **Pattern used: none — a concurrency-primitive fix, not a design pattern.** Say explicitly why atomic beats a mutex here: a single-field increment is exactly what `sync/atomic` is for — lighter weight than acquiring/releasing a full mutex for one instruction's worth of work.
>
> **Checkpoint 3 (~10 min) — view dedup within a cooldown window.**
> ```go
> type ViewTracker struct {
>     mu       sync.Mutex
>     lastView map[string]map[string]time.Time // lastView[videoID][userID]
>     cooldown time.Duration
> }
>
> func (t *ViewTracker) RecordView(video *Video, userID string) {
>     t.mu.Lock()
>     defer t.mu.Unlock()
>     if t.lastView[video.ID] == nil {
>         t.lastView[video.ID] = make(map[string]time.Time)
>     }
>     if last, seen := t.lastView[video.ID][userID]; seen && time.Since(last) < t.cooldown {
>         return // rewatched too soon — don't double count
>     }
>     t.lastView[video.ID][userID] = time.Now()
>     video.RecordView()
> }
> ```
> This is a real, easy-to-forget product rule — say it out loud even if not asked: *"otherwise a user refreshing the page 50 times counts as 50 views."*
>
> **Checkpoint 4 (remaining time, or if asked) — subscriptions via Observer, kept brief (reused pattern).**
> ```go
> // SubscriberObserver — the SAME Observer pattern already used for
> // Notification System, Task Scheduler, and Google Meet. Kept brief
> // here since it's the 4th application, not new content.
> type SubscriberObserver interface {
>     OnNewVideo(channelID, videoTitle string)
> }
>
> type Channel struct {
>     ID          string
>     subscribers []SubscriberObserver
> }
>
> func (c *Channel) Subscribe(o SubscriberObserver) {
>     c.subscribers = append(c.subscribers, o)
> }
>
> func (c *Channel) UploadVideo(video *Video) {
>     for _, s := range c.subscribers {
>         s.OnNewVideo(c.ID, video.Title)
>     }
> }
> ```
> **Pattern used: Observer (reused).** Don't over-explain this one — say "same pattern as Notification System" and move on to whatever the interviewer wants to probe deeper.
>
> **If you're short on time:** stop after Checkpoint 3. Correctly fixing the lost-update bug AND handling view dedup is the real substance of this question — subscriptions are a straightforward reuse of a pattern you've already proven you know.

## Step 3 — Why `Views++` breaks, precisely

> [!bug] A lost update, not a double-claim
> `v.Views++` compiles to a read, an increment, and a write — three separate steps with no atomicity between them. Two goroutines can both read `Views == 100`, both compute `101`, and both write `101` — one increment vanishes. This is structurally different from [[LLD/06 - Design BookMyShow - Seat Booking/Design BookMyShow - Seat Booking|BookMyShow's]] check-then-act bug (where the danger is two actors both *succeeding* at claiming the same resource): here there's no claim or ownership at all, just a shared counter being updated unsafely. Naming this as a distinct race-condition category — not just pattern-matching "concurrency bug = seat booking bug" — is a real signal.

## Step 4 — Why atomic, not a mutex

> [!tip] The lightest correct tool for the job
> A `sync.Mutex` would also fix this (lock, increment, unlock), but for a single scalar field, `sync/atomic` is the more idiomatic, lower-overhead choice — no lock acquisition/release machinery for what's fundamentally one CPU instruction's worth of protected work. Reach for a mutex when protecting multiple related fields that must change together; reach for atomics when protecting one.

---

## Step 5 — Complete, compilable Go implementation

```go
// FILE: video.go
package youtube

import "sync/atomic"

type Video struct {
    ID        string
    ChannelID string
    Title     string
    views     int64
    likes     map[string]bool
    comments  []Comment
}

func NewVideo(id, channelID, title string) *Video {
    return &Video{ID: id, ChannelID: channelID, Title: title, likes: make(map[string]bool)}
}

func (v *Video) RecordView() {
    atomic.AddInt64(&v.views, 1)
}

func (v *Video) ViewCount() int64 {
    return atomic.LoadInt64(&v.views)
}

func (v *Video) Like(userID string) {
    v.likes[userID] = true
}

func (v *Video) LikeCount() int {
    return len(v.likes)
}
```

```go
// FILE: view_tracker.go
package youtube

import (
    "sync"
    "time"
)

// ViewTracker deduplicates views per (video, user) within a cooldown
// window, then delegates the actual count increment to Video's
// atomic counter.
type ViewTracker struct {
    mu       sync.Mutex
    lastView map[string]map[string]time.Time
    cooldown time.Duration
}

func NewViewTracker(cooldown time.Duration) *ViewTracker {
    return &ViewTracker{lastView: make(map[string]map[string]time.Time), cooldown: cooldown}
}

func (t *ViewTracker) RecordView(video *Video, userID string) {
    t.mu.Lock()
    defer t.mu.Unlock()

    if t.lastView[video.ID] == nil {
        t.lastView[video.ID] = make(map[string]time.Time)
    }
    if last, seen := t.lastView[video.ID][userID]; seen && time.Since(last) < t.cooldown {
        return
    }
    t.lastView[video.ID][userID] = time.Now()
    video.RecordView()
}
```

```go
// FILE: comment.go
package youtube

// Comment — the same shape as Instagram's Commentable content,
// not re-derived here.
type Comment struct {
    UserID string
    Text   string
}

func (v *Video) AddComment(userID, text string) {
    v.comments = append(v.comments, Comment{UserID: userID, Text: text})
}
```

```go
// FILE: channel.go
package youtube

// SubscriberObserver — the same Observer pattern already used for
// Notification System, Task Scheduler, and Google Meet.
type SubscriberObserver interface {
    OnNewVideo(channelID, videoTitle string)
}

type Channel struct {
    ID          string
    subscribers []SubscriberObserver
}

func NewChannel(id string) *Channel {
    return &Channel{ID: id}
}

func (c *Channel) Subscribe(o SubscriberObserver) {
    c.subscribers = append(c.subscribers, o)
}

func (c *Channel) UploadVideo(video *Video) {
    for _, s := range c.subscribers {
        s.OnNewVideo(c.ID, video.Title)
    }
}
```

```go
// FILE: main.go  (adjust import path to your module name)
package main

import (
    "fmt"
    "sync"
    "time"

    youtube "example.com/youtube"
)

type ConsoleSubscriber struct{ Name string }

func (c *ConsoleSubscriber) OnNewVideo(channelID, videoTitle string) {
    fmt.Printf("[%s] new upload from %s: %s\n", c.Name, channelID, videoTitle)
}

func main() {
    channel := youtube.NewChannel("gophers-channel")
    channel.Subscribe(&ConsoleSubscriber{Name: "alice"})
    channel.Subscribe(&ConsoleSubscriber{Name: "bob"})

    video := youtube.NewVideo("v1", channel.ID, "Intro to Go Concurrency")
    channel.UploadVideo(video) // both subscribers notified

    tracker := youtube.NewViewTracker(10 * time.Minute)

    // Simulate 100 concurrent viewers, each watching once — proves
    // the atomic counter has zero lost updates.
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(userNum int) {
            defer wg.Done()
            tracker.RecordView(video, fmt.Sprintf("user-%d", userNum))
        }(i)
    }
    wg.Wait()
    fmt.Println("View count after 100 concurrent unique viewers:", video.ViewCount()) // exactly 100

    // Same user rewatching immediately — deduped, not counted twice.
    tracker.RecordView(video, "user-0")
    fmt.Println("View count after immediate rewatch by user-0:", video.ViewCount()) // still 100
}
```

---

## Interview Q&A

> [!question]- Why does the concurrent-viewer test in `main.go` matter — what would a broken version look like?
> With the buggy `v.Views++` from Checkpoint 1, running 100 concurrent goroutines each calling `RecordView` would very likely produce a final count *less than* 100 — some increments get lost when two goroutines race on the same read-modify-write sequence. The atomic version guarantees exactly 100, every run, no exceptions.

> [!question]- How would you extend view counting to support "unique views" vs. "total plays" as two separate metrics?
> Two separate counters — a `sync/atomic` counter for total plays (increment on every `RecordView` call, no dedup), and the `ViewTracker`'s deduped count for unique views. They're genuinely different metrics answering different questions, and conflating them into one number loses information either way.

> [!question]- Why is this a good use of `sync/atomic` instead of `sync.Mutex`, precisely?
> `sync/atomic` protects a single, simple operation (increment, load) on one scalar field with minimal overhead — no lock object, no acquire/release pair. A mutex earns its cost when multiple related fields need to change together *atomically as a group* (like `ParkingSpot.Status` plus `heldBy` plus `holdExpiry` in the BookMyShow chapter) — a single counter doesn't have that multi-field coordination need.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/09 - Design YouTube - Netflix/Design YouTube - Netflix|HLD version]] · [[LLD/08 - Design a Notification System/Design a Notification System|Design a Notification System]] (Observer)*
