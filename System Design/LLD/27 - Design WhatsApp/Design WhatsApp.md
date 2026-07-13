---
title: "Design WhatsApp (LLD)"
aliases: [Design WhatsApp LLD, Read Receipts LLD, Group Chat LLD]
tags: [system-design, lld, case-study, go]
status: reference-quality
---

# Design WhatsApp (LLD)

> [!abstract] What you'll be able to do after this chapter
> Model per-recipient read receipts correctly (a group message needs one status PER participant, not one global bool), and make the same "is this really a State pattern, or just a guarded enum" judgment call already exercised in the Vending Machine and BookMyShow chapters — applied to message status this time.

> [!info] Distinct from the generic Chat Application chapter
> [[LLD/18 - Design a Chat Application/Design a Chat Application|The Chat Application chapter]] taught Mediator — how messages get *routed* between users without direct references. This chapter assumes routing is solved and focuses on a genuinely different problem: modeling 1-to-1 vs. group conversations uniformly, and tracking per-recipient read-receipt state correctly.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design the core object model for WhatsApp — 1-to-1 and group chats, with message read receipts (sent/delivered/read) tracked correctly for every recipient."

## Step 2 — Requirements

**Functional:** send a message in a 1-to-1 chat or a group chat, track read-receipt status per recipient, list a conversation's message history.

**Non-functional:** a **group** message needs a **per-participant** read status — not one global bool, since some group members may have read it while others haven't. Status must only move **forward** (Sent → Delivered → Read), never backward.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~6 min) — the buggy single-bool version, deliberately.**
> ```go
> type Message struct {
>     ID   string
>     Text string
>     read bool // breaks for groups — WHO read it?
> }
> ```
> **Pattern used: none — a live modeling bug, on purpose.** Narrate it: *"this works for a 1-to-1 chat, but a group message read by 2 of 5 members can't be represented by one bool — I need status per recipient."*
>
> **Checkpoint 2 (~10-12 min) — fix with per-recipient status, as a guarded enum, not full State.**
> ```go
> type MessageStatus int
>
> const (
>     Sent MessageStatus = iota
>     Delivered
>     Read
> )
>
> type Message struct {
>     ID       string
>     SenderID string
>     Text     string
>     status   map[string]MessageStatus // per RECIPIENT — the actual fix
> }
>
> // MarkDelivered/MarkRead are guarded transitions, NOT the full State
> // pattern from earlier chapters — same judgment call as BookMyShow's
> // Available/Held/Booked: no rich per-status BEHAVIOR differs here,
> // just a legality/ordering check. A full State pattern would be
> // over-engineering for two forward-only transitions.
> func (m *Message) MarkDelivered(participantID string) error {
>     current, ok := m.status[participantID]
>     if !ok {
>         return ErrNotARecipient
>     }
>     if current >= Delivered {
>         return nil // already delivered or read — no-op, not an error
>     }
>     m.status[participantID] = Delivered
>     return nil
> }
> ```
> **Pattern used: none — a guarded enum.** Say the judgment call out loud unprompted: *"I could model this as full State with a Status interface per message-status, but the transitions here don't have different behavior, just an ordering rule — a guarded enum is the simpler, equally correct choice."*
>
> **Checkpoint 3 (~8 min) — `MarkRead` with enforced ordering, plus `IsReadByAll`.**
> ```go
> func (m *Message) MarkRead(participantID string) error {
>     current, ok := m.status[participantID]
>     if !ok {
>         return ErrNotARecipient
>     }
>     if current == Sent {
>         return ErrCannotSkipDelivered // must be delivered before read
>     }
>     m.status[participantID] = Read
>     return nil
> }
>
> // IsReadByAll — a GROUP message is only "fully read" once every
> // participant individually reaches Read.
> func (m *Message) IsReadByAll() bool {
>     for _, s := range m.status {
>         if s != Read {
>             return false
>         }
>     }
>     return true
> }
> ```
> This checkpoint is where the real signal is — the forward-only enforcement (`current == Sent` blocking a jump straight to `Read`) is a genuine correctness rule, not decoration.
>
> **Checkpoint 4 (remaining time, or if asked) — unify 1-to-1 and group via one `Conversation` interface.**
> ```go
> // Conversation — implemented by BOTH DirectConversation and
> // GroupConversation, so calling code never branches on chat type.
> type Conversation interface {
>     SendMessage(senderID, text string) *Message
>     Participants() []string
> }
> ```
> **No new pattern** — a small shared interface, same spirit as Instagram's `Likeable`/`Commentable` but applied here to unify two chat shapes instead of splitting content capabilities.
>
> **If you're short on time:** stop after Checkpoint 3. Per-recipient status with correctly enforced forward-only transitions IS the actual substance of this question — unifying 1-to-1 and group behind one interface is a clean but secondary addition.

## Step 3 — Why the single-bool version breaks

> [!bug] One bool can't represent "read by some, not others"
> `Message.read bool` implicitly assumes exactly one reader. The moment a group has 5 participants, "read" needs to answer "read by *whom*" — a single bool has already lost the information needed to answer that, no matter how the rest of the code is written around it. This is a modeling bug, not an extensibility gap — the fix isn't a new pattern, it's a different *data shape*.

## Step 4 — Why a guarded enum, not the State pattern

> [!tip] The same judgment call, exercised a third time
> [[LLD/02 - Design a Vending Machine/Design a Vending Machine|The Vending Machine chapter]] and [[LLD/06 - Design BookMyShow - Seat Booking/Design BookMyShow - Seat Booking|BookMyShow]] both made this call explicitly: State earns its cost when different states have genuinely different *behavior* for the same action. Here, `MarkDelivered` and `MarkRead` don't behave differently depending on current status in any rich way — they just enforce "don't go backward, don't skip a step." A guarded enum with ordering checks is simpler, equally correct, and the right call — reflexively reaching for full State every time a value has named stages would be over-engineering.

---

## Step 5 — Complete, compilable Go implementation

```go
// FILE: message.go
package whatsapp

import "errors"

var (
    ErrNotARecipient       = errors.New("whatsapp: user is not a recipient of this message")
    ErrCannotSkipDelivered = errors.New("whatsapp: cannot mark read before delivered")
)

type MessageStatus int

const (
    Sent MessageStatus = iota
    Delivered
    Read
)

type Message struct {
    ID       string
    SenderID string
    Text     string
    status   map[string]MessageStatus // per-recipient status — the group-chat fix
}

func NewMessage(id, senderID, text string, recipients []string) *Message {
    status := make(map[string]MessageStatus)
    for _, r := range recipients {
        status[r] = Sent
    }
    return &Message{ID: id, SenderID: senderID, Text: text, status: status}
}

func (m *Message) MarkDelivered(participantID string) error {
    current, ok := m.status[participantID]
    if !ok {
        return ErrNotARecipient
    }
    if current >= Delivered {
        return nil
    }
    m.status[participantID] = Delivered
    return nil
}

func (m *Message) MarkRead(participantID string) error {
    current, ok := m.status[participantID]
    if !ok {
        return ErrNotARecipient
    }
    if current == Sent {
        return ErrCannotSkipDelivered
    }
    m.status[participantID] = Read
    return nil
}

func (m *Message) StatusFor(participantID string) (MessageStatus, bool) {
    s, ok := m.status[participantID]
    return s, ok
}

func (m *Message) IsReadByAll() bool {
    for _, s := range m.status {
        if s != Read {
            return false
        }
    }
    return true
}
```

```go
// FILE: conversation.go
package whatsapp

import "fmt"

type Conversation interface {
    SendMessage(senderID, text string) *Message
    Participants() []string
    Messages() []*Message
}

var nextMessageID int

func generateMessageID() string {
    nextMessageID++
    return fmt.Sprintf("msg-%d", nextMessageID)
}

// sendMessage is shared by both conversation types below — avoids
// duplicating the identical "build recipient list, create message,
// append to history" logic in two places.
func sendMessage(participants []string, messages *[]*Message, senderID, text string) *Message {
    var recipients []string
    for _, p := range participants {
        if p != senderID {
            recipients = append(recipients, p)
        }
    }
    msg := NewMessage(generateMessageID(), senderID, text, recipients)
    *messages = append(*messages, msg)
    return msg
}

type DirectConversation struct {
    userA, userB string
    messages     []*Message
}

func NewDirectConversation(userA, userB string) *DirectConversation {
    return &DirectConversation{userA: userA, userB: userB}
}

func (c *DirectConversation) Participants() []string { return []string{c.userA, c.userB} }
func (c *DirectConversation) Messages() []*Message    { return c.messages }
func (c *DirectConversation) SendMessage(senderID, text string) *Message {
    return sendMessage(c.Participants(), &c.messages, senderID, text)
}

type GroupConversation struct {
    participants []string
    messages     []*Message
}

func NewGroupConversation(participants []string) *GroupConversation {
    return &GroupConversation{participants: participants}
}

func (c *GroupConversation) Participants() []string { return c.participants }
func (c *GroupConversation) Messages() []*Message     { return c.messages }
func (c *GroupConversation) SendMessage(senderID, text string) *Message {
    return sendMessage(c.participants, &c.messages, senderID, text)
}
```

```go
// FILE: main.go  (adjust import path to your module name)
package main

import (
    "fmt"

    whatsapp "example.com/whatsapp"
)

func main() {
    direct := whatsapp.NewDirectConversation("alice", "bob")
    msg1 := direct.SendMessage("alice", "hey bob!")
    msg1.MarkDelivered("bob")
    msg1.MarkRead("bob")
    status, _ := msg1.StatusFor("bob")
    fmt.Println("Direct message status for bob:", status) // Read

    group := whatsapp.NewGroupConversation([]string{"alice", "bob", "carol", "dave"})
    msg2 := group.SendMessage("alice", "team meeting at 3pm")

    msg2.MarkDelivered("bob")
    msg2.MarkRead("bob")
    msg2.MarkDelivered("carol") // carol hasn't read it yet

    fmt.Println("Read by all?", msg2.IsReadByAll()) // false — dave hasn't even gotten delivery

    if err := msg2.MarkRead("dave"); err != nil {
        fmt.Println("correctly rejected:", err) // ErrCannotSkipDelivered
    }
}
```

---

## Interview Q&A

> [!question]- Why not just store a `map[string]bool` (read or not) instead of a 3-state enum?
> That loses the sent/delivered distinction entirely — a real WhatsApp feature (single grey check vs. double grey check vs. double blue check) that users actually rely on. The 3-state enum captures a real product requirement a simple bool can't express, independent of the groups-vs-1-to-1 issue.

> [!question]- How would you extend this to support message deletion or editing?
> Add a `DeletedAt *time.Time` or `EditedAt *time.Time` field to `Message`, checked by conversation history rendering — a new, independent piece of state, not something that interacts with the read-receipt status machinery at all, which is a good sign the current design keeps concerns properly separated.

> [!question]- Why does `sendMessage` live as a shared free function instead of being duplicated in both `DirectConversation` and `GroupConversation`?
> Both types' `SendMessage` bodies were identical except for which participant list they passed in — duplicating that logic risks the two copies drifting apart over time (a bug fixed in one, forgotten in the other). Factoring it into one shared helper both types call keeps there being exactly one place that logic can go wrong.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/07 - Design WhatsApp - Chat System/Design WhatsApp - Chat System|HLD version]] · [[LLD/18 - Design a Chat Application/Design a Chat Application|Design a Chat Application]] (Mediator) · [[LLD/02 - Design a Vending Machine/Design a Vending Machine|Design a Vending Machine]] (guarded-enum judgment call)*
