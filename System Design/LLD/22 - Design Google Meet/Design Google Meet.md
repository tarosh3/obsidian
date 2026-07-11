---
title: "Design Google Meet (LLD)"
aliases: [Design Google Meet LLD, Meeting Room LLD]
tags: [system-design, lld, case-study, golang, design-patterns, observer-pattern]
status: reference-quality
---

# Design Google Meet (LLD)

> [!abstract] What you'll be able to do after this chapter
> Implement a room-SCOPED Observer (events only reach participants in the same room, not a global broadcast) and a real permission check woven directly into a state-changing action, not bolted on separately.

> [!info] Distinct from the HLD version
> [[HLD/22 - Design Google Meet/Design Google Meet|The HLD chapter]] covers SFU/MCU infrastructure, simulcast, and NAT traversal — the actual media transport. This chapter is the **in-process meeting-room object model**: participants, mute state, and room-scoped event broadcasting.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design the core meeting-room object model — participants join/leave, per-participant mute/camera state, broadcasting state changes to everyone else in the room in real time, with host-only actions."

## Step 2 — Requirement clarification

Join/leave a room. Per-participant mute/video state. Broadcast state changes to all **other** participants in the **same** room — never globally. A host role with elevated permissions (mute-all, remove participant).

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~6-8 min) — Join/Leave only, no broadcasting at all.**
> ```go
> type Room struct {
>     participants map[string]*Participant
> }
>
> func (r *Room) Join(participantID string, isHost bool) {
>     r.participants[participantID] = &Participant{ID: participantID, IsHost: isHost}
> }
> ```
> **Pattern used: none.** Prove participants can join/leave and the map reflects it correctly before adding any notification.
>
> **Checkpoint 2 (~5 min) — bad draft, kept brief.** Each state-changing method (`Mute`, `Join`, `Leave`) loops participants and pushes updates inline, duplicated per method. Narrate the coupling rather than fully building it: *"if I add a call-recording service that also needs every state change, I'm editing every one of these methods individually."*
>
> **Checkpoint 3 (~10-12 min) — refactor into room-SCOPED Observer. State the scoping distinction unprompted.**
> ```go
> // RoomObserver — Observer, but SCOPED to one Room, not global.
> // An event in Room A must never reach an observer of Room B.
> type RoomObserver interface {
>     OnRoomEvent(event RoomEvent)
> }
>
> type Room struct {
>     participants map[string]*Participant
>     observers    []RoomObserver // owned per-Room, not shared globally
> }
>
> func notifyAll(observers []RoomObserver, event RoomEvent) {
>     for _, o := range observers {
>         o.OnRoomEvent(event)
>     }
> }
> ```
> **Pattern used: Observer (room-scoped variant).** Say explicitly how this differs from the Notification System chapter's single global `EventPublisher` — that's the single strongest thing to volunteer in this question.
>
> **Checkpoint 4 (remaining time, or if asked) — host-only `MuteAll`, permission check woven into the action.**
> ```go
> func (r *Room) MuteAll(requesterID string) error {
>     requester, ok := r.participants[requesterID]
>     if !ok || !requester.IsHost {
>         return ErrNotHost // permission check IS the first line, not bolted on after
>     }
>     // mute everyone else, notify
> }
> ```
> **No new pattern** — but this is the second real signal in the chapter: authorization living directly inside the action it guards, not as a separate middleware-style layer floating outside the object model.
>
> **If you're short on time:** stop after Checkpoint 3. Join/Leave/Mute with room-scoped Observer notification working, and the scoping distinction stated clearly, is a complete answer — describe `MuteAll`'s permission check verbally as the next addition.

## Step 3 — The bad first draft (kept brief)

A `Room` struct with a flat participant list, where every state-changing method (`Mute`, `Join`, `Leave`) directly loops over all participants and pushes an update **inline within the same method that changed the state**. Tightly couples "what changed" with "how everyone finds out" — adding a new kind of listener (e.g. a call-recording service that also needs every state change) means editing every single state-mutation method individually.

## Step 4 — Refactor: room-scoped Observer

> [!tip] The key difference from the Notification System chapter's Observer
> [[LLD/08 - Design a Notification System/Design a Notification System|The Notification System chapter's]] `EventPublisher` is a single, global publisher. Here, **each `Room` owns its own observer list** — an event fired in Room A must never reach an observer subscribed only to Room B. This is a deliberate, meaningful difference in scope, not an arbitrary implementation choice — a meeting's participant events are inherently room-local.

Every state-changing method calls one shared internal `notifyAll`, rather than each duplicating its own broadcast loop.

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: participant.go
// ============================================================
package meet

type Participant struct {
	ID      string
	Muted   bool
	VideoOn bool
	IsHost  bool
}
```

```go
// ============================================================
// FILE: event.go
// ============================================================
package meet

type EventType string

const (
	ParticipantJoined EventType = "JOINED"
	ParticipantLeft   EventType = "LEFT"
	MuteChanged       EventType = "MUTE_CHANGED"
)

type RoomEvent struct {
	Type          EventType
	ParticipantID string
}
```

```go
// ============================================================
// FILE: observer.go
// ============================================================
package meet

// RoomObserver is notified of state changes in ONE specific room —
// a participant's own client UI, a call-recording service, and a
// live-transcription service can all subscribe independently.
type RoomObserver interface {
	OnRoomEvent(event RoomEvent)
}
```

```go
// ============================================================
// FILE: room.go
// ============================================================
package meet

import (
	"errors"
	"sync"
)

var (
	ErrParticipantNotFound = errors.New("meet: participant not found in room")
	ErrNotHost             = errors.New("meet: only the host can perform this action")
)

// Room's observers are notified only about events IN THIS ROOM —
// deliberately not a global event stream, see Step 4.
type Room struct {
	mu           sync.Mutex
	ID           string
	participants map[string]*Participant
	observers    []RoomObserver
}

func NewRoom(id string) *Room {
	return &Room{ID: id, participants: make(map[string]*Participant)}
}

func (r *Room) Subscribe(o RoomObserver) {
	r.mu.Lock()
	defer r.mu.Unlock()
	r.observers = append(r.observers, o)
}

func (r *Room) Join(participantID string, isHost bool) {
	r.mu.Lock()
	r.participants[participantID] = &Participant{ID: participantID, IsHost: isHost}
	observersCopy := append([]RoomObserver{}, r.observers...)
	r.mu.Unlock()

	notifyAll(observersCopy, RoomEvent{Type: ParticipantJoined, ParticipantID: participantID})
}

func (r *Room) Leave(participantID string) {
	r.mu.Lock()
	delete(r.participants, participantID)
	observersCopy := append([]RoomObserver{}, r.observers...)
	r.mu.Unlock()

	notifyAll(observersCopy, RoomEvent{Type: ParticipantLeft, ParticipantID: participantID})
}

func (r *Room) SetMuted(participantID string, muted bool) error {
	r.mu.Lock()
	p, ok := r.participants[participantID]
	if !ok {
		r.mu.Unlock()
		return ErrParticipantNotFound
	}
	p.Muted = muted
	observersCopy := append([]RoomObserver{}, r.observers...)
	r.mu.Unlock()

	notifyAll(observersCopy, RoomEvent{Type: MuteChanged, ParticipantID: participantID})
	return nil
}

// MuteAll is host-only — a real permission check woven directly
// into the action, not a separate, bolt-on authorization layer.
func (r *Room) MuteAll(requesterID string) error {
	r.mu.Lock()
	requester, ok := r.participants[requesterID]
	if !ok || !requester.IsHost {
		r.mu.Unlock()
		return ErrNotHost
	}
	var events []RoomEvent
	for id, p := range r.participants {
		if id != requesterID {
			p.Muted = true
			events = append(events, RoomEvent{Type: MuteChanged, ParticipantID: id})
		}
	}
	observersCopy := append([]RoomObserver{}, r.observers...)
	r.mu.Unlock()

	for _, e := range events {
		notifyAll(observersCopy, e)
	}
	return nil
}

func notifyAll(observers []RoomObserver, event RoomEvent) {
	for _, o := range observers {
		o.OnRoomEvent(event)
	}
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"

	meet "example.com/meet"
)

type ConsoleObserver struct{}

func (c *ConsoleObserver) OnRoomEvent(event meet.RoomEvent) {
	fmt.Printf("[room event] %s: %s\n", event.Type, event.ParticipantID)
}

func main() {
	room := meet.NewRoom("room-1")
	room.Subscribe(&ConsoleObserver{})

	room.Join("host-1", true)
	room.Join("guest-1", false)
	room.Join("guest-2", false)

	room.SetMuted("guest-1", true)

	if err := room.MuteAll("guest-2"); err != nil {
		fmt.Println("correctly rejected non-host mute-all:", err)
	}

	room.MuteAll("host-1")

	room.Leave("guest-1")
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "Why does each `Room` maintain its own observer list instead of one shared, global publisher?"
> Meeting events are inherently scoped to the room they happen in — a participant in Room B has no reason to ever receive Room A's join/leave/mute events. A single global publisher would either leak cross-room events to everyone, or need every subscriber to manually filter by room ID — pushing the scoping responsibility onto every observer instead of the publisher itself.

> [!quote]- "How would you add a 'remove participant' host-only action?"
> A new `Room.RemoveParticipant(requesterID, targetID string) error`, structurally identical to `MuteAll` — check `requester.IsHost`, mutate state under the lock, copy observers, notify after releasing the lock. No changes needed to `RoomObserver`, `Participant`, or any existing method.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/22 - Design Google Meet/Design Google Meet|HLD version]] · [[LLD/08 - Design a Notification System/Design a Notification System|Design a Notification System]]*
