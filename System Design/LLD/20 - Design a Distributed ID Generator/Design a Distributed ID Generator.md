---
title: "Design a Distributed ID Generator (Snowflake-style)"
aliases: [Design Snowflake, Distributed ID Generator LLD]
tags: [system-design, lld, case-study, golang, distributed-systems]
status: reference-quality
---

# Design a Distributed ID Generator (Snowflake-style)

> [!abstract] What you'll be able to do after this chapter
> Implement the actual Twitter Snowflake bit-layout — not just describe it — closing the loop on the ID-generation problem [[HLD/01 - Design TinyURL (URL Shortener)/Design TinyURL|the TinyURL chapter]] named but only partially solved.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a distributed unique ID generator producing roughly time-ordered, globally unique 64-bit IDs across many machines, with no coordination needed per ID."

## Step 2 — Requirement clarification

No central coordination per ID generated. IDs roughly sortable by generation time. Many machines generating concurrently, IDs still guaranteed globally unique.

## Step 3 — The bad first draft, and why TinyURL's fix was only partial

A single shared counter (Redis `INCR` or a DB sequence) works, but requires a network round-trip **per ID generated** — the exact bottleneck [[HLD/01 - Design TinyURL (URL Shortener)/Design TinyURL|the TinyURL chapter]] named and partially mitigated via **range allocation** (handing each server a pre-fetched block of IDs). Range allocation reduces round-trips but doesn't eliminate the shared-counter dependency entirely — this chapter builds the fully-local alternative that dependency was pointing toward.

## Step 4 — Refactor: the Snowflake bit layout

A 64-bit ID composed of: 1 unused sign bit + **41 bits timestamp** (milliseconds since a custom epoch, ~69 years of range) + **10 bits machine ID** (up to 1,024 distinct machines) + **12 bits sequence number** (up to 4,096 IDs per machine per millisecond).

> [!tip] Every ID is generated ENTIRELY locally — no network call, ever
> The machine-ID segment makes collisions between *different* machines structurally impossible — two machines can never produce the same ID, since their machine-ID bits differ. The sequence number handles multiple IDs generated on the *same* machine within the *same* millisecond. No coordination, no shared state, no network round-trip per ID — a genuinely different approach from both the naive counter and TinyURL's range-allocation mitigation.

```mermaid
graph LR
    subgraph "64-bit ID"
    A["1 bit<br/>unused (sign)"] --> B["41 bits<br/>timestamp (ms since custom epoch)"]
    B --> C["10 bits<br/>machine ID"]
    C --> D["12 bits<br/>sequence"]
    end
```

---

## Step 5 — Complete, compilable Go implementation

```go
// ============================================================
// FILE: snowflake.go
// ============================================================
package idgen

import (
	"errors"
	"sync"
	"time"
)

const (
	timestampBits = 41
	machineIDBits = 10
	sequenceBits  = 12

	maxMachineID = -1 ^ (-1 << machineIDBits) // 1023
	maxSequence  = -1 ^ (-1 << sequenceBits)  // 4095

	machineIDShift = sequenceBits
	timestampShift = sequenceBits + machineIDBits
)

// customEpoch is an arbitrary reference point (not Unix epoch) that
// maximizes the useful range of the 41-bit timestamp field, since
// dates before this system existed never need representing.
var customEpoch = time.Date(2024, 1, 1, 0, 0, 0, 0, time.UTC).UnixMilli()

var ErrInvalidMachineID = errors.New("idgen: machine ID out of range")

// SnowflakeGenerator produces IDs entirely locally — no network
// round-trip per ID, unlike a shared counter — while still
// guaranteeing global uniqueness via the embedded machine ID.
type SnowflakeGenerator struct {
	mu            sync.Mutex
	machineID     int64
	lastTimestamp int64
	sequence      int64
}

func NewSnowflakeGenerator(machineID int64) (*SnowflakeGenerator, error) {
	if machineID < 0 || machineID > maxMachineID {
		return nil, ErrInvalidMachineID
	}
	return &SnowflakeGenerator{machineID: machineID, lastTimestamp: -1}, nil
}

// NextID generates one ID. Only concurrent calls on THIS SAME
// generator instance serialize against each other — a call on a
// different machine's generator never contends with this one at all,
// since there's no shared state between machines to begin with.
func (g *SnowflakeGenerator) NextID() int64 {
	g.mu.Lock()
	defer g.mu.Unlock()

	now := time.Now().UnixMilli() - customEpoch

	if now == g.lastTimestamp {
		// Same millisecond as the last ID — increment the sequence,
		// wrapping and waiting for the next millisecond if exhausted.
		g.sequence = (g.sequence + 1) & maxSequence
		if g.sequence == 0 {
			for now <= g.lastTimestamp {
				now = time.Now().UnixMilli() - customEpoch
			}
		}
	} else {
		g.sequence = 0
	}

	g.lastTimestamp = now

	return (now << timestampShift) | (g.machineID << machineIDShift) | g.sequence
}
```

```go
// ============================================================
// FILE: main.go  (adjust import path to your module name)
// ============================================================
package main

import (
	"fmt"

	idgen "example.com/idgen"
)

func main() {
	gen, err := idgen.NewSnowflakeGenerator(5) // this machine is worker #5
	if err != nil {
		panic(err)
	}

	for i := 0; i < 5; i++ {
		fmt.Println(gen.NextID())
	}
}
```

---

## 🎯 Interview follow-up Q&A

> [!quote]- "What happens if the same machine generates more than 4,096 IDs within one millisecond?"
> The sequence counter wraps to 0, and `NextID` busy-waits (spinning `time.Now()` calls) until the clock actually advances to the next millisecond — guaranteeing no two IDs from this machine ever share both the same timestamp and sequence value, at the cost of a brief spin under extreme burst load.

> [!quote]- "What happens if the system clock moves backward (NTP adjustment, VM migration)?"
> This is a genuine, known Snowflake weakness — if `now` is ever less than `lastTimestamp`, the current implementation doesn't detect it, which could theoretically produce a duplicate or out-of-order ID. A production-hardened version would explicitly check for clock regression and either refuse to generate IDs until the clock catches up, or maintain a small buffer of "borrowed" future time — worth naming as a real edge case this simplified version doesn't handle, rather than claiming false completeness.

> [!quote]- "Why 41 bits for timestamp specifically, not more or fewer?"
> It's a deliberate three-way tradeoff against machine-ID bits and sequence bits, all sharing a fixed 63-bit budget — 41 bits gives ~69 years of range (more than enough for any realistic system lifetime), while leaving enough remaining bits to support a meaningful number of machines (1,024) and a meaningful per-machine-per-millisecond throughput (4,096) simultaneously.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/01 - Design TinyURL (URL Shortener)/Design TinyURL|Design TinyURL]]*
