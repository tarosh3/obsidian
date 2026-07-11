---
title: "Design Google Calendar (LLD)"
aliases: [Design Google Calendar LLD, Calendar LLD, RRULE]
tags: [system-design, lld, case-study, golang, design-patterns, strategy-pattern]
status: reference-quality
---

# Design Google Calendar (LLD)

> [!abstract] What you'll be able to do after this chapter
> Implement recurrence-rule expansion via Strategy, correctly handle per-occurrence exceptions, and know a real, easy-to-miss Go gotcha: why `time.Time` should never be used directly as a map key.

> [!info] Distinct from the HLD version
> [[HLD/21 - Design Google Calendar/Design Google Calendar|The HLD chapter]] covers free-busy indexing, timezone/DST correctness at scale, and shared-resource conflict prevention. This chapter is the **core event/recurrence object model**.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design the core event and recurrence object model for a calendar service — support single and recurring events, per-occurrence exceptions, and conflict detection."

## Step 2 — Requirement clarification

Create single or recurring events. Cancel/modify one occurrence without affecting the series. Detect scheduling conflicts for a shared resource.

## Step 3 — The bad first draft (kept brief)

An `Event` struct with an `isRecurring bool` and ad-hoc fields (`interval int`, `unit string`), with recurrence-expansion logic **duplicated inline** at every call site that needs occurrences (e.g. "today's events" and "this week's events" each re-implementing the same expansion). Adding a new recurrence pattern means finding and editing every duplicate.

## Step 4 — Refactor: Strategy for recurrence rules

`RecurrenceRule` — `Occurrences(seriesStart, from, to time.Time) []time.Time` — implemented independently by `DailyRule` and `WeeklyOnDaysRule`. Expansion logic lives in exactly one place per rule type, computed once, never duplicated across call sites.

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: recurrence.go
// ============================================================
package calendar

import "time"

// RecurrenceRule replaced the duplicated expansion logic from the
// bad first draft — each rule type owns its own expansion.
type RecurrenceRule interface {
	Occurrences(seriesStart, from, to time.Time) []time.Time
}

// DailyRule repeats every N days, optionally until a bound.
type DailyRule struct {
	IntervalDays int
	Until        time.Time // zero value means no end
}

func (r DailyRule) Occurrences(seriesStart, from, to time.Time) []time.Time {
	var result []time.Time
	cursor := seriesStart
	for !cursor.After(to) {
		if !r.Until.IsZero() && cursor.After(r.Until) {
			break
		}
		if !cursor.Before(from) {
			result = append(result, cursor)
		}
		cursor = cursor.AddDate(0, 0, r.IntervalDays)
	}
	return result
}

// WeeklyOnDaysRule repeats on specific weekdays every week.
type WeeklyOnDaysRule struct {
	Weekdays []time.Weekday
	Until    time.Time
}

func (r WeeklyOnDaysRule) Occurrences(seriesStart, from, to time.Time) []time.Time {
	var result []time.Time
	weekdaySet := make(map[time.Weekday]bool)
	for _, d := range r.Weekdays {
		weekdaySet[d] = true
	}

	cursor := seriesStart
	for !cursor.After(to) {
		if !r.Until.IsZero() && cursor.After(r.Until) {
			break
		}
		if !cursor.Before(from) && weekdaySet[cursor.Weekday()] {
			result = append(result, cursor)
		}
		cursor = cursor.AddDate(0, 0, 1)
	}
	return result
}
```

```go
// ============================================================
// FILE: event.go
// ============================================================
package calendar

import "time"

type RSVPStatus int

const (
	Pending RSVPStatus = iota
	Accepted
	Declined
)

type Attendee struct {
	UserID string
	Status RSVPStatus
}

type Event struct {
	ID         string
	Title      string
	Start, End time.Time
	Rule       RecurrenceRule // nil for a one-off event
	Attendees  []*Attendee

	// exceptions is keyed by Unix() timestamp, NOT time.Time itself —
	// see the bug callout below for why.
	exceptions map[int64]bool
}

func NewEvent(id, title string, start, end time.Time, rule RecurrenceRule) *Event {
	return &Event{ID: id, Title: title, Start: start, End: end, Rule: rule, exceptions: make(map[int64]bool)}
}

// CancelOccurrence marks one instance of a recurring series as
// cancelled WITHOUT affecting the rest of the series.
func (e *Event) CancelOccurrence(date time.Time) {
	e.exceptions[date.Unix()] = true
}

// Occurrences returns every concrete instance within [from, to],
// filtering cancelled exceptions. Expansion is delegated entirely to
// the Rule; exceptions are applied uniformly here regardless of
// which concrete Rule produced the raw occurrences.
func (e *Event) Occurrences(from, to time.Time) []time.Time {
	if e.Rule == nil {
		if !e.Start.Before(from) && !e.Start.After(to) {
			return []time.Time{e.Start}
		}
		return nil
	}

	raw := e.Rule.Occurrences(e.Start, from, to)
	var result []time.Time
	for _, occ := range raw {
		if !e.exceptions[occ.Unix()] {
			result = append(result, occ)
		}
	}
	return result
}
```

> [!bug] Why `exceptions` is keyed by `int64` (Unix timestamp), not `time.Time` directly
> `time.Time` is technically comparable with `==`, so using it as a map key *compiles* — but it's a real, known Go gotcha: two `time.Time` values representing the exact same instant can compare as **unequal** if they carry different monotonic-clock readings or different `*Location` pointers, even though they're semantically identical. A cancellation date constructed by the caller and an occurrence generated by rule expansion could easily diverge this way despite representing "the same moment," silently breaking the exception lookup. Keying by `.Unix()` (or `.UnixNano()` for sub-second precision) sidesteps the problem entirely by comparing a plain `int64`, not the `time.Time` struct's internal representation.

```go
// ============================================================
// FILE: conflict.go
// ============================================================
package calendar

import (
	"errors"
	"sync"
	"time"
)

var ErrConflict = errors.New("calendar: conflicting event already booked for this time")

// ResourceCalendar guards a single shared resource (e.g. a
// conference room) — the same atomic check-then-book discipline as
// the BookMyShow and Uber chapters, applied here to a time slot.
type ResourceCalendar struct {
	mu     sync.Mutex
	booked []struct{ start, end time.Time }
}

func (r *ResourceCalendar) TryBook(start, end time.Time) error {
	r.mu.Lock()
	defer r.mu.Unlock()

	for _, b := range r.booked {
		if start.Before(b.end) && b.start.Before(end) { // overlap check
			return ErrConflict
		}
	}
	r.booked = append(r.booked, struct{ start, end time.Time }{start, end})
	return nil
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"
	"time"

	calendar "example.com/calendar"
)

func main() {
	start := time.Date(2026, 7, 20, 10, 0, 0, 0, time.UTC)
	end := time.Date(2026, 7, 20, 10, 30, 0, 0, time.UTC)

	standup := calendar.NewEvent("evt-1", "Daily Standup", start, end,
		calendar.WeeklyOnDaysRule{
			Weekdays: []time.Weekday{time.Monday, time.Wednesday, time.Friday},
			Until:    time.Date(2026, 12, 31, 0, 0, 0, 0, time.UTC),
		},
	)

	standup.CancelOccurrence(time.Date(2026, 7, 22, 10, 0, 0, 0, time.UTC))

	occurrences := standup.Occurrences(
		time.Date(2026, 7, 20, 0, 0, 0, 0, time.UTC),
		time.Date(2026, 7, 27, 0, 0, 0, 0, time.UTC),
	)
	fmt.Println("Occurrences this week:", occurrences)

	room := &calendar.ResourceCalendar{}
	if err := room.TryBook(start, end); err != nil {
		fmt.Println("booking failed:", err)
	} else {
		fmt.Println("room booked successfully")
	}
	if err := room.TryBook(start, end); err != nil {
		fmt.Println("second booking correctly rejected:", err)
	}
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "How would you add a monthly recurrence rule (e.g. 'the 15th of every month')?"
> A new `MonthlyRule` implementing `RecurrenceRule`, with its own `Occurrences` expansion logic. Zero changes to `Event`, `ResourceCalendar`, or any existing rule.

> [!quote]- "Why key `exceptions` by Unix timestamp instead of just formatting the date as a string?"
> Either works correctly, but `int64` is cheaper to hash and compare than a string, and avoids any subtlety around date-formatting consistency (timezone in the formatted string, precision, etc.) between the two places dates get converted to keys — `Unix()` is a single, unambiguous canonical form.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/21 - Design Google Calendar/Design Google Calendar|HLD version]] · [[LLD/06 - Design BookMyShow - Seat Booking/Design BookMyShow - Seat Booking|Design BookMyShow]]*
