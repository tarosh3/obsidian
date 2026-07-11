---
title: "Design a Chat Application (LLD-level)"
aliases: [Design a Chat Application LLD, Mediator Pattern Chat]
tags: [system-design, lld, case-study, go, mediator-pattern]
status: reference-quality
---

# Design a Chat Application (LLD-level)

> [!abstract] What you'll be able to do after this chapter
> Implement the Mediator pattern correctly, and state precisely how it differs from Observer even though both involve a "room" object coordinating multiple participants.

> [!info] Distinct from the HLD chapter and from Google Meet's LLD
> [[HLD/07 - Design WhatsApp - Chat System/Design WhatsApp - Chat System|The WhatsApp HLD chapter]] covers the distributed infrastructure (message delivery, fan-out, offline queuing). [[LLD/22 - Design Google Meet/Design Google Meet|Google Meet's LLD chapter]] uses a room-scoped **Observer** to broadcast state *changes*. This chapter is deliberately different: it's about **Mediator** — centralizing how peer objects *communicate*, not how they're notified of state changes. Worth holding these apart precisely; they look similar on the surface.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a chat application's core messaging object model — users send messages to a room, without needing direct references to every other user in that room."

## Step 2 — Requirements

Users join a chat room. A sent message reaches all other members. Users shouldn't need to hold direct references to every other user — avoiding tight, N-way coupling.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~6 min) — the buggy N-to-N version, deliberately.**
> ```go
> type User struct {
>     Name   string
>     Others []*User // every other member — kept in sync manually
> }
>
> func (u *User) SendMessage(msg string) {
>     for _, other := range u.Others {
>         other.Receive(u, msg)
>     }
> }
> ```
> **Pattern used: none.** Works for 2-3 hardcoded users. Narrate: *"every user needs a reference to every other user — that's O(N²) references to maintain, and users are coupled directly to each other instead of through anything shared."*
>
> **Checkpoint 2 (~10-12 min) — refactor into Mediator.**
> ```go
> // ChatMediator — the Mediator pattern. Replaces every User needing
> // a direct reference to every other User.
> type ChatMediator interface {
>     SendMessage(from *User, message string)
>     Join(user *User)
> }
>
> type User struct {
>     Name     string
>     mediator ChatMediator // the ONLY reference a User ever holds
> }
>
> func (u *User) Send(message string) {
>     u.mediator.SendMessage(u, message)
> }
> ```
> **Pattern used: Mediator.** This is the checkpoint that matters — get `ChatRoom.Join` and `ChatRoom.SendMessage` (looping its own member list, excluding the sender) working for 3 users before anything else.
>
> **Checkpoint 3 (~5 min) — prove it with 3 users, one message.** Join Alice/Bob/Carol, `alice.Send(...)`, confirm Bob and Carol receive it and Alice doesn't. This is a cheap, convincing demo of the whole design.
>
> **Checkpoint 4 (remaining time, or if asked) — Mediator vs. Observer, and private messages.** No code needed for the distinction — state it precisely (Step 5): Observer broadcasts a state *change*; Mediator centralizes *how peers communicate*. If asked for 1-to-1 messaging, add `SendPrivate(from, to *User, message string)` to the same mediator — the room stays the one place that knows how to route any shape of message.
>
> **If you're short on time:** stop after Checkpoint 2 with 2 users proven working. The Mediator refactor itself, demonstrated even minimally, is the entire point of this question.

## Step 3 — The bad first draft

Each `User` holds a list of **every other user** directly, and `SendMessage` loops over that list calling each one's `ReceiveMessage`:

```go
// The bad draft — every User references every other User directly
type User struct {
    Name    string
    Others  []*User // every other member — must be kept in sync manually
}

func (u *User) SendMessage(msg string) {
    for _, other := range u.Others {
        other.Receive(u, msg)
    }
}
```

Works for a tiny, static group. Breaks structurally: every `User` must be updated whenever membership changes, and users are coupled **directly to each other** rather than through any shared coordinating object. N users require managing N-1 references *each* — an O(N²) reference-management problem. This is distinct from, though related to, the O(N²) *bandwidth* problem the WhatsApp/Meet chapters already solved at the infrastructure level — here it's an **object-design coupling** problem, not a bandwidth one.

## Step 4 — Refactor: the Mediator pattern

`ChatRoom` becomes the single object every `User` communicates *through*. A `User` sends a message **to the ChatRoom**, and the ChatRoom — not the sending user — is responsible for delivering it to every other member. Users never hold references to each other at all, only to the shared mediator.

```go
// FILE: mediator.go
package chat

// ChatMediator is the abstraction — it replaces every User needing a
// direct reference to every other User.
type ChatMediator interface {
    SendMessage(from *User, message string)
    Join(user *User)
    Leave(user *User)
}
```

```go
// FILE: user.go
package chat

import "fmt"

// User only ever holds a reference to the ChatMediator — never to
// any other User directly. This is the actual point of Mediator:
// centralizing "who talks to whom" in one place instead of scattering
// N-1 references across every participant.
type User struct {
    Name     string
    mediator ChatMediator
}

func NewUser(name string, mediator ChatMediator) *User {
    return &User{Name: name, mediator: mediator}
}

func (u *User) Send(message string) {
    u.mediator.SendMessage(u, message)
}

func (u *User) Receive(from *User, message string) {
    fmt.Printf("[%s received from %s]: %s\n", u.Name, from.Name, message)
}
```

```go
// FILE: chat_room.go
package chat

import "sync"

// ChatRoom is the concrete Mediator — it decides how a message from
// one User reaches every other member, without any User needing to
// know how many other members exist or who they are.
type ChatRoom struct {
    mu      sync.Mutex
    members []*User
}

func NewChatRoom() *ChatRoom {
    return &ChatRoom{}
}

func (r *ChatRoom) Join(user *User) {
    r.mu.Lock()
    defer r.mu.Unlock()
    r.members = append(r.members, user)
}

func (r *ChatRoom) Leave(user *User) {
    r.mu.Lock()
    defer r.mu.Unlock()
    for i, m := range r.members {
        if m == user {
            r.members = append(r.members[:i], r.members[i+1:]...)
            return
        }
    }
}

// SendMessage copies the member list under lock, then delivers
// outside the lock — the same safe pattern used by the Notification
// Service's EventPublisher and Google Meet's Room: never hold a lock
// while calling into arbitrary receiver code.
func (r *ChatRoom) SendMessage(from *User, message string) {
    r.mu.Lock()
    membersCopy := append([]*User{}, r.members...)
    r.mu.Unlock()

    for _, m := range membersCopy {
        if m != from {
            m.Receive(from, message)
        }
    }
}
```

```go
// FILE: main.go
package main

import (
    chat "example.com/chat"
)

func main() {
    room := chat.NewChatRoom()

    alice := chat.NewUser("Alice", room)
    bob := chat.NewUser("Bob", room)
    carol := chat.NewUser("Carol", room)

    room.Join(alice)
    room.Join(bob)
    room.Join(carol)

    alice.Send("Hey everyone!") // Bob and Carol receive it; Alice does not receive her own message
}
```

---

## Step 5 — Mediator vs. Observer, precisely

> [!warning] These are genuinely different responsibilities, easy to conflate
> - **Observer** (used in [[LLD/22 - Design Google Meet/Design Google Meet|Google Meet's Room]] and [[LLD/08 - Design a Notification System/Design a Notification System|the Notification System's EventPublisher]]): a subject broadcasts **state changes** to subscribers that registered interest. The subject doesn't care what the observers do with the notification.
> - **Mediator** (used here): centralizes **how peer objects interact** with each other, replacing direct references between them. The mediator isn't broadcasting a state change — it's actively routing a communication between two specific parties (or one-to-many, as here) that would otherwise need to know about each other directly.
>
> A `ChatRoom` and a Meet `Room` look similar — both are "a room object coordinating multiple participants" — but their responsibilities are different: Meet's `Room` broadcasts *that something changed* (someone muted, someone joined). This `ChatRoom` centralizes *the act of one user talking to the others*. Worth being able to state this distinction precisely rather than treating "room" as a single mental pattern.

## Interview Q&A

> [!question]- Could you implement chat with Observer instead of Mediator?
> You could make `ChatRoom` an Observer subject where a "message sent" event is the state change being broadcast — and at the mechanics level the resulting code would look almost identical to what's above. The distinction is about **intent**: Observer's contract is "notify me when X changes," open-ended and passive. Mediator's contract is "route this interaction for me," an active coordination responsibility. In an interview, naming *both* framings and explaining why Mediator is the more precise fit here (users are explicitly relying on the room to route sends, not just subscribing to a feed) is a stronger answer than picking one silently.

> [!question]- How would you extend this to private (1-to-1) messages within the same room?
> Add a `SendPrivate(from, to *User, message string)` method on the mediator that delivers to exactly one recipient instead of iterating all members — the mediator remains the single place that knows how to route any shape of message, which is exactly why centralizing this logic in one object pays off as requirements grow.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/22 - Design Google Meet/Design Google Meet|Design Google Meet]] · [[LLD/08 - Design a Notification System/Design a Notification System|Design a Notification System]] · [[HLD/07 - Design WhatsApp - Chat System/Design WhatsApp - Chat System|WhatsApp HLD]]*
