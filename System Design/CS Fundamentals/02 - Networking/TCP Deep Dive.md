---
title: "TCP Deep Dive"
aliases: [TCP, Transmission Control Protocol, TCP Handshake]
tags: [system-design, cs-fundamentals, networking, tcp]
status: reference-quality
---

# TCP Deep Dive

> [!abstract] What you'll be able to do after this chapter
> Draw the 3-way handshake and the 4-way close from memory with a justification for each step, explain flow control vs congestion control as genuinely different mechanisms, and diagnose a real production Nagle's-algorithm latency bug.

---

## 1. Why TCP exists

Raw IP packet delivery gives you none of what most applications actually need: packets can be **lost**, **duplicated**, or **arrive out of order** — IP is a "best effort" protocol with zero guarantees. TCP sits on top of IP and adds exactly four things: **reliability** (retransmit lost data), **ordering** (deliver bytes to the application in the order they were sent, even if packets arrived out of order), **flow control** (don't overwhelm a slow receiver), and **congestion control** (don't overwhelm the network itself).

## 2. The three-way handshake — and why three, not two

```mermaid
sequenceDiagram
    participant C as Client
    participant S as Server
    C->>S: SYN (seq=x)
    S->>C: SYN-ACK (seq=y, ack=x+1)
    C->>S: ACK (ack=y+1)
    Note over C,S: Connection established, both sides confirmed
```

Each side needs to confirm two things: "I can send you data" and "I received your confirmation that you can send me data." A 2-way handshake can't close that gap — the second party would have no way to know their SYN-ACK actually arrived before sending real data. The third message (`ACK`) is what confirms the server's `SYN-ACK` was received, giving both sides mutual confirmation before any application data flows.

> [!bug] Worth naming explicitly in an interview
> Even the 3-way handshake doesn't perfectly solve this in an absolute theoretical sense (the "Two Generals Problem" proves no finite handshake can guarantee both sides have *provably* synchronized state over an unreliable channel) — it's a practical, good-enough solution, not a mathematical guarantee. Naming this shows real depth beyond "SYN, SYN-ACK, ACK."

## 3. Reliability — sequence numbers & acknowledgment

Every byte sent gets a sequence number. The receiver acknowledges the next byte it expects (`ACK` = "I have everything up through here"). If the sender doesn't see an ACK within a timeout, it retransmits. **Fast retransmit** is a real optimization worth knowing: three duplicate ACKs for the same sequence number signal a specific packet was likely lost (later packets *did* arrive, confirming they're being acked with the same "still waiting for X" number) — the sender retransmits immediately, without waiting for the full timeout.

## 4. Flow control vs congestion control — genuinely different mechanisms

> [!warning] The most common thing candidates conflate
> These solve two *different* problems and use *different* mechanisms — treating them as the same concept is a real gap interviewers probe for.

**Flow control** protects the **receiver**. The receiver advertises a **window size** — how many unacknowledged bytes it's willing to buffer. The sender must never have more than that outstanding at once. Purely about not overwhelming *this specific receiver's* buffer.

**Congestion control** protects the **network** itself, independent of what any single receiver can handle. TCP maintains a **congestion window (cwnd)**, starting small and growing via **slow start** (roughly doubling every round-trip) until packet loss is detected, then switching to **congestion avoidance** (linear growth) — and on loss, cutting the window multiplicatively (commonly halving it). This produces the well-known TCP "sawtooth" throughput pattern.

```mermaid
graph LR
    A["cwnd small<br/>(slow start: exponential growth)"] --> B["loss detected"]
    B --> C["cwnd halved<br/>(congestion avoidance: linear growth)"]
    C --> D["loss detected again"]
    D --> E["cwnd halved again"]
```

The **actual send window** is `min(flow-control window, congestion window)` — the sender is bounded by whichever constraint is tighter at any given moment.

## 5. Connection teardown & TIME_WAIT

Closing a connection is (typically) a four-message exchange: `FIN → ACK → FIN → ACK`. After the final `ACK`, the side that initiated the close enters **TIME_WAIT** — a state where the connection lingers (not fully freed) for **2×MSL** (Maximum Segment Lifetime, historically ~30-60 seconds total).

> [!tip] Why TIME_WAIT exists — not just "cleanup"
> If a duplicate or delayed packet from the *old* connection is still in flight when a *new* connection happens to reuse the exact same source/destination IP+port pair, that stray packet could be misdelivered into the new connection's stream. TIME_WAIT ensures enough time passes for any such stragglers to die out in the network before the address pair can be reused.

**Real production consequence:** a host making very many short-lived outbound TCP connections (a proxy, a load balancer, a service hammering an external API without connection pooling) can accumulate thousands of sockets stuck in TIME_WAIT, eventually **exhausting available local ports** — a genuine, recurring production incident. Mitigation: connection pooling/reuse (avoid creating new connections per request at all), or `SO_REUSEADDR`/tuned TIME_WAIT settings where appropriate.

## 6. A real production gotcha: Nagle's Algorithm + Delayed ACK

**Nagle's algorithm** batches small outgoing writes into fewer, larger packets to reduce overhead — good for bulk throughput. **Delayed ACK** on the receiving side waits briefly (up to ~40ms) hoping to piggyback an ACK on outgoing data rather than sending a bare ACK packet. Combined, these two independently-reasonable optimizations can interact badly: a sender waiting for an ACK before sending its next small write, paired with a receiver delaying that ACK — produces a real, measurable **~40ms stall** on latency-sensitive small-message traffic.

> [!bug] The fix, worth knowing by name
> Setting `TCP_NODELAY` on the socket disables Nagle's algorithm — the standard fix for latency-sensitive applications sending small, frequent messages (RPC calls, interactive protocols) where throughput-optimized batching actively hurts.

## 7. TCP vs UDP — the core tradeoff (briefly, full UDP treatment is its own future chapter)

TCP's reliability/ordering guarantees come at a real cost: the handshake before any data flows, retransmission waits on loss, and **head-of-line blocking** — a single lost packet blocks *all* subsequently-arrived data from being delivered to the application until the gap is filled, even if later bytes already arrived intact. UDP has none of TCP's guarantees or overhead — the right choice when a late packet is *worse* than a dropped one (live video/audio, gaming) rather than needing every byte to eventually arrive correctly.

## 8. Scaling: 1 connection to millions

```mermaid
flowchart TD
    A["1 connection<br/>trivial"] --> B["Many concurrent connections<br/>on one server<br/>file descriptor limits,<br/>TIME_WAIT exhaustion risk"]
    B --> C["Massive scale<br/>epoll/io_uring required —<br/>see I/O Models chapter"]
    C --> D["Global, high-latency links<br/>standard TCP throughput bound by<br/>RTT x window size — window<br/>scaling needed for 'long fat pipes'"]
```

At high connection counts on one machine, [[CS Fundamentals/01 - Operating Systems/I-O Models - Blocking, Non-Blocking, and Async|the I/O Models chapter's]] `epoll`-based event loop is what makes holding tens of thousands of TCP connections open on a handful of threads actually feasible — this chapter's mechanics (handshake, windows, retransmission) apply per-connection regardless of how many are held, but *how* a server holds many of them at once is that chapter's concern, not this one's. Over high-latency, high-bandwidth links (cross-region, satellite), a well-known real constraint appears: a connection's maximum throughput is bounded by `window size / RTT` — a large window is useless if RTT is high enough that acknowledgments trickle back slowly, the "long fat pipe" problem, addressed via TCP window scaling (negotiated during the handshake) to use a larger effective window than the original protocol's default allows.

## 9. Failure scenarios

> [!bug] What actually happens
> - **Packet loss mid-stream:** retransmission via timeout, or **fast retransmit** (Section 3) when three duplicate ACKs signal a specific loss — the faster path in practice.
> - **A peer crashes without sending `FIN`:** TCP alone doesn't notice for a long time without **keepalive** enabled — the connection can sit in a "believed open" state on the surviving side indefinitely unless the application layer sends its own heartbeats or TCP keepalive is explicitly configured.
> - **Sustained network congestion:** repeated loss triggers repeated congestion-window collapses — the sawtooth pattern (Section 4) becomes a genuine, sustained throughput ceiling rather than a brief blip.

## 10. Monitoring

> [!info] What to watch
> **Retransmission rate** — the direct, measurable signal of packet loss/network health on a given path. **Connection count and TIME_WAIT count** — the leading indicator before local port exhaustion (Section 5) becomes an actual incident. **RTT distribution** — rising values or growing variance signal network path degradation before it shows up as application-level timeouts.

## 11. Common mistakes

> [!warning] Real, recurring errors
> 1. **Not setting `TCP_NODELAY` for latency-sensitive RPC traffic** — Section 6's Nagle's-algorithm interaction causes a real, measurable ~40ms stall that's easy to misdiagnose as "the network is just slow."
> 2. **Not pooling/reusing connections** — Section 5's TIME_WAIT exhaustion is a direct, common consequence of creating a fresh short-lived connection per request instead of reusing one.
> 3. **Assuming TCP alone detects a dead peer** — without keepalive configured (at the TCP or application layer), a connection to a crashed peer can appear open far longer than expected.

---

## 🎯 Interview follow-up Q&A

> [!info] Leveled by seniority
> **Beginner:** "What does TCP add on top of raw IP?" — reliability, ordering, flow control, congestion control (Section 1). **Intermediate:** "Explain flow control vs. congestion control." — Section 4's precise distinction. **Senior:** "A service making many short-lived outbound calls to an external API is intermittently failing with 'cannot assign requested address' — diagnose it." — expects recognizing local port exhaustion from TIME_WAIT accumulation (Section 5) and connection pooling as the fix, not a network-layer red herring. **Staff:** "Design connection-handling for a service expecting to hold hundreds of thousands of concurrent long-lived connections (e.g. WebSockets)." — expects the epoll/event-loop answer from the I/O Models chapter, explicitly connected to per-connection TCP mechanics staying the same regardless of count. **Architect:** "How does TCP's behavior need to be tuned differently for a service primarily serving users across intercontinental links vs. one serving users in a single data center?" — expects window scaling and the "long fat pipe" reasoning from Section 8, not a generic "just add more bandwidth" answer.

> [!quote]- "Why does TCP use a 3-way handshake instead of 2 messages?"
> Both sides need to independently confirm "I can send" and "I've received your confirmation that you can send" before exchanging real data — a 2-way exchange leaves the initiator's confirmation-of-confirmation unverified.
>
> **Follow-up: "Does the 3-way handshake fully guarantee both sides have synchronized state?"**
> Not in an absolute theoretical sense — the Two Generals Problem shows no finite handshake over an unreliable channel can provide a mathematical guarantee. It's a practical, robust-in-practice solution, not a provable one.

> [!quote]- "What's the difference between flow control and congestion control?"
> Flow control protects the specific receiver from being overwhelmed, governed by its advertised window size. Congestion control protects the network itself from being overwhelmed, governed by the sender's dynamically-adjusted congestion window (slow start → congestion avoidance → multiplicative decrease on loss).
>
> **Follow-up: "What actually determines how much data can be in flight at any moment?"**
> The minimum of the flow-control window and the congestion window — whichever constraint is currently tighter.

> [!quote]- "What real production problem can excessive TIME_WAIT sockets cause?"
> A host opening many short-lived outbound connections without pooling can exhaust its available local ports, since each closed connection lingers in TIME_WAIT for roughly 2×MSL before its port can be reused.
>
> **Follow-up: "How would you fix it?"**
> Connection pooling/reuse to avoid creating new short-lived connections in the first place — addressing the root cause rather than tuning TIME_WAIT duration, which only reduces the symptom's window, not the underlying churn.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/01 - Design TinyURL (URL Shortener)/Design TinyURL|Design TinyURL]]*
