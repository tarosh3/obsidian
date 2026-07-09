---
title: 30-Day Interview Prep Plan
aliases: [Prep Plan, 30 Day Plan]
tags: [interview-prep, dsa, lld, hld, golang]
start: 2026-07-07
end: 2026-08-05
target: Tier-2 with ease, Tier-1 stretch
status: in-progress
---

# 🎯 30-Day Interview Prep Plan

> [!abstract] Overview
> **Start:** Tue, 7 Jul 2026 (Day 1) → **End:** Wed, 5 Aug 2026 (Day 30)
> **Target:** crack any Tier-2 with ease, Tier-1 if possible.
> **Profile:** 4 YOE, Go/backend/distributed systems.
> **Languages:** DSA in **C++**, LLD in **Go**, real coding in Go.
> **Budget:** ~80 hrs (22 weekdays × 2 hrs + 8 weekend days × 4.5 hrs).

> [!info]- What changed from v1 (click to expand)
> Completed original Days 1–4 (through Fri 3 Jul), then lost momentum. This version **restarts the count from today** and **removes everything already covered**.
> **Already done (not repeated):** Arrays & Hashing · Two Pointers · Sliding Window · Go basics + goroutines/channels · SOLID · Creational patterns.
> **Silver lining:** 30 days for ~26 days of content = ~4 extra buffer days, added to the mock/polish phase. This nudges Tier-1 from "stretch" toward "reachable."

> [!tip] Language split & Go LLD idioms
> DSA = **C++** (coding rounds). LLD = **Go** — plays well at Go-heavy targets (Razorpay, Cashfree, Juspay, most fintech/GCC backends) and doubles as Go fluency reactivation.
> **Go has no classes/inheritance.** Model LLD with **structs + interfaces + composition (embedding)**:
> - Strategy / State → interface with swappable impls
> - Factory → constructor func returning an interface
> - Singleton → package var + `sync.Once`
> - Observer → slice of subscriber interfaces
> - Concurrency → `sync.Mutex` / `sync.RWMutex` + channels
> Where an item says "class hierarchy," read it as "interface + concrete structs."

> [!warning] Non-negotiable rules
> 1. **No AI while practicing** — blank editor, autocomplete off.
> 2. **Re-solve, don't just solve** — anything you needed a hint on gets re-solved cold 2 days later.
> 3. **Protect the time** — minimum viable day is 1 hr (DSA block only). Never zero.

> [!note] Resources (don't fragment)
> - **DSA:** NeetCode 150 (free) → solve on LeetCode in C++. Videos only when stuck.
> - **LLD:** Refactoring Guru (has Go examples) · Grokking the LLD Interview · search "design patterns in Go".
> - **HLD:** System Design Primer (GitHub) · Grokking the System Design Interview · ByteByteGo.
> - **Mocks:** Pramp / interviewing.io or self-timed.
> - **Tracker:** Problem | Pattern | Date | Confidence (1–5) | Re-solve date.

---

## 📌 Phase 1 — Confirm + core patterns (Days 1–7 · 7–13 Jul)

### Day 1 — Tue 7 Jul — 2 hrs
- [ ] **DSA (1h30):** Retention check (30 min) — re-solve 3 covered problems cold (Arrays/Hashing, Two Pointers, Sliding Window). Then **Stack** (1h): Valid Parentheses, Min Stack, Daily Temperatures, Car Fleet. #dsa
- [ ] **LLD (0h30):** Structural + Behavioral patterns overview in Go (interfaces + embedding) — Decorator, Adapter, Observer, Strategy, State. One-line Go sketch each. #lld

- *Goal: covered stuff confirmed; monotonic-stack pattern back.*

### Day 2 — Wed 8 Jul — 2 hrs
- [ ] **DSA (1h15):** Binary Search — Binary Search, Search Rotated Array, Find Min in Rotated, Koko Eating Bananas. #dsa
- [ ] **LLD (0h45):** Implement Observer + Strategy in Go from scratch. #lld

- *Goal: all binary-search variants (incl. on answer space) fluent.*

### Day 3 — Thu 9 Jul — 2 hrs
- [ ] **DSA (1h15):** Linked List Part 1 — Reverse List, Merge Two Lists, Reorder List, Remove Nth Node. #dsa
- [ ] **HLD (0h45):** Fundamentals Part 1 — scalability, load balancing (L4/L7), caching, CDN. One-page summary. #hld

- *Goal: pointer manipulation restarted; HLD vocab begun.*

### Day 4 — Fri 10 Jul — 2 hrs
- [ ] **DSA (1h15):** Linked List Part 2 + LRU Cache — Cycle, Add Two Numbers, then LRU Cache carefully. #dsa
- [ ] **LLD (0h45):** Pattern consolidation — re-implement any pattern you fumbled; map pattern → problem. #lld

- *Goal: LRU logic solid; pattern toolkit ready.*

### Day 5 — Sat 11 Jul — 4.5 hrs 🟢 weekend
- [ ] **DSA (2h):** Trees Part 1+2 — Invert, Max Depth, Diameter, Balanced, Same Tree; Level Order, Right Side View, Validate BST, Kth Smallest. #dsa
- [ ] **LLD (1h15):** **Problem 1 — LRU Cache (full).** Struct + `map` + hand-rolled DLL, `sync.Mutex`. Go. #lld
- [ ] **HLD (1h15):** Fundamentals Part 2 — sharding, replication, CAP, consistency, SQL vs NoSQL. #hld

- *Goal: tree recursion + BFS/DFS automatic; data-layer tradeoffs articulate-able.*

### Day 6 — Sun 12 Jul — 4.5 hrs 🟢 weekend
- [ ] **DSA (2h):** Tries + Heaps — Implement Trie, Add/Search Word; Kth Largest, Last Stone Weight, K Closest Points, Task Scheduler. #dsa
- [ ] **LLD (1h):** **Problem 2 — Parking Lot.** Vehicle as interface + structs, Factory (constructors) + Strategy. Go. #lld
- [ ] **HLD (1h):** Message queues / Kafka deep-dive (your strength). Draft "partition leader dies mid-rebalance" answer. #hld
- [ ] **Revision (0h30):** Re-solve 2 problems from Days 1–3 cold.

- *Goal: trie + heap fluent; Kafka talking points sharp.*

### Day 7 — Mon 13 Jul — 2 hrs
- [ ] **DSA (1h15):** Backtracking — Subsets, Combination Sum, Permutations, Word Search. #dsa
- [ ] **HLD (0h45):** Case study — TinyURL (refine). API, encoding, schema, estimation, cache. 35 min. #hld

- *Goal: backtracking template down; one polished HLD case.*

> [!success] Phase 1 checkpoint
> Stack, binary search, linked list, trees, tries, heaps, backtracking done. 2 LLD problems (Go). HLD fundamentals + Kafka + TinyURL covered.

---

## 📌 Phase 2 — Core depth (Days 8–14 · 14–20 Jul)

### Day 8 — Tue 14 Jul — 2 hrs
- [ ] **DSA (1h15):** Priority Queue (harder) — Design Twitter, Find Median from Data Stream, Merge K Sorted Lists, Reorganize String. #dsa
- [ ] **LLD (0h45):** **Problem 3 — Rate Limiter.** `RateLimiter` interface, TokenBucket/SlidingWindow structs, `sync.Mutex`. Go. *Targets your concurrency gap.* #lld

- *Goal: heap-with-comparator fluent; concurrency in code.*

### Day 9 — Wed 15 Jul — 2 hrs
- [ ] **DSA (1h15):** Graphs Part 1 (BFS/DFS) — Number of Islands, Clone Graph, Max Area, Pacific Atlantic, Rotting Oranges. #dsa
- [ ] **HLD (0h45):** Distributed Rate Limiter — token/leaky bucket, Redis INCR + Lua, cross-node races. *Your weak spot.* #hld

- *Goal: grid/graph traversal automatic; atomicity reasoning sharp.*

### Day 10 — Thu 16 Jul — 2 hrs
- [ ] **DSA (1h15):** Graphs Part 2 — Course Schedule I & II (topo), Redundant Connection (Union-Find), Connected Components. #dsa
- [ ] **LLD (0h45):** **Problem 4 — Vending Machine / Tic-Tac-Toe.** State via `State` interface + per-state structs. Go. #lld

- *Goal: topo sort + Union-Find solid; State pattern internalized.*

### Day 11 — Fri 17 Jul — 2 hrs
- [ ] **DSA (1h15):** Advanced Graphs — Min Cost to Connect Points (MST), Network Delay (Dijkstra), Cheapest Flights K Stops. #dsa
- [ ] **HLD (0h45):** Case study — News Feed / Twitter. Fan-out write vs read, celebrity problem. 35 min. #hld

- *Goal: Dijkstra/MST recalled; feed-system pattern learned.*

### Day 12 — Sat 18 Jul — 4.5 hrs 🟢 weekend
- [ ] **DSA (2h):** Graph mixed + re-solve — Graph Valid Tree, Word Ladder, Surrounded Regions + re-solve 3 weak graph problems. #dsa
- [ ] **LLD (1h15):** **Problem 5 — BookMyShow / seat booking.** Concurrency, `sync.Mutex`/`RWMutex`, double-booking prevention. Go. *Concurrency again.* #lld
- [ ] **HLD (1h15):** Case study — Chat system (WhatsApp). WebSockets, delivery/ordering, groups. 40 min. #hld

- *Goal: graphs mastered; booking-concurrency handled.*

### Day 13 — Sun 19 Jul — 4.5 hrs 🟢 weekend
- [ ] **DSA (2h):** 1-D DP Part 1+2 — Climbing Stairs, House Robber I & II, Longest Palindromic Substring, Decode Ways, Coin Change. #dsa
- [ ] **LLD (1h):** **Problem 6 — Logger / Notification.** Chain of Responsibility (handler interfaces) + Observer. Go. #lld
- [ ] **HLD (1h):** Case study — Notification system. Queue-based fan-out (Kafka), retries, idempotency. 35 min. #hld
- [ ] **Revision (0h30):** Re-solve 2 DP problems cold.

- *Goal: 1-D DP fluent; idempotency reasoning sharp.*

### Day 14 — Mon 20 Jul — 2 hrs
- [ ] **DSA (1h15):** 1-D DP (harder) — Word Break, LIS, Partition Equal Subset Sum, Max Product Subarray. #dsa
- [ ] **HLD (0h45):** Case study — Distributed cache. Consistent hashing, eviction, replication, invalidation. 35 min. #hld

- *Goal: trickier 1-D DP confronted; caching internals clear.*

> [!success] Phase 2 checkpoint
> Graphs, advanced graphs, 1-D DP done. 6 LLD problems (Go). 6 HLD case studies. Concurrency hit 3×.

---

## 📌 Phase 3 — Hard patterns + HLD breadth (Days 15–21 · 21–27 Jul)

### Day 15 — Tue 21 Jul — 2 hrs
- [ ] **DSA (1h15):** 2-D DP Part 1 — Unique Paths, LCS, Coin Change II. #dsa
- [ ] **LLD (0h45):** **Problem 7 — Splitwise.** Balance graph (`map[string]map[string]float64`) + simplify-debts. Go. #lld

- *Goal: 2-D DP grid setup automatic.*

### Day 16 — Wed 22 Jul — 2 hrs
- [ ] **DSA (1h15):** 2-D DP Part 2 — Edit Distance, Target Sum, Interleaving String. #dsa
- [ ] **HLD (0h45):** Case study — Uber / proximity. Geohash/QuadTree, matching, real-time location. 35 min. #hld

- *Goal: harder 2-D DP confronted.*

### Day 17 — Thu 23 Jul — 2 hrs
- [ ] **DSA (1h15):** Greedy — Maximum Subarray, Jump Game I & II, Gas Station. #dsa
- [ ] **LLD (0h45):** **Problem 8 — Elevator.** State (elevator states) + Strategy (scheduling interface). Go. #lld

- *Goal: greedy fluent; LLD breadth complete.*

### Day 18 — Fri 24 Jul — 2 hrs
- [ ] **DSA (1h15):** Intervals — Insert Interval, Merge Intervals, Non-overlapping, Meeting Rooms II. #dsa
- [ ] **HLD (0h45):** Mine your own experience — write 3 case studies (Kafka pipeline, multi-tenant SaaS, Beckn). Rehearse each in 5 min. #hld

- *Goal: intervals fluent; personal experience weaponized.*

### Day 19 — Sat 25 Jul — 4.5 hrs 🟢 weekend
- [ ] **DSA (2h):** Math/Bit + weak-spot revision — Reverse Integer, Single Number, Number of 1 Bits, Counting Bits + re-solve 4 lowest-confidence. #dsa
- [ ] **LLD (1h15):** Cold mock — unseen problem (Food Delivery, Chess, ATM). 45 min in Go: requirements → interface/struct design → core code. #lld
- [ ] **HLD (1h15):** Case study — YouTube/Netflix. Upload pipeline, transcoding, CDN, storage. 40 min. #hld

- *Goal: full DSA coverage complete; first cold LLD attempt.*

### Day 20 — Sun 26 Jul — 4.5 hrs 🟢 weekend
- [ ] **DSA (2h):** Timed mixed set — 3 mediums, 45-min cap. Pull from lowest-confidence patterns. #dsa
- [ ] **LLD (1h):** Re-solve timed — one earlier problem from blank file, 45 min. #lld
- [ ] **HLD (1h):** Case study — Web crawler / search. BFS at scale, dedup, politeness. 35 min. #hld
- [ ] **Revision (0h30):** Update tracker; list 10 weakest problems for Phase 4.

- *Goal: interview-pace execution; weak-spots identified.*

### Day 21 — Mon 27 Jul — 2 hrs
- [ ] **DSA (1h30):** Timed mixed set — 2 mediums + 1 hard, 60 min. #dsa
- [ ] **HLD (0h30):** Full mock — cold case: requirements → estimation → high-level → deep-dive → bottlenecks. #hld

- *Goal: hard-problem composure under time.*

> [!success] Phase 3 checkpoint
> All DSA patterns covered + drilled. 8 LLD problems (Go) + 1 cold mock. 10+ HLD cases incl. your own work. Everything timed.

---

## 📌 Phase 4 — Mock & polish (Days 22–30 · 28 Jul – 5 Aug)

### Day 22 — Tue 28 Jul — 2 hrs
- [ ] **DSA (1h15):** Weak-spot drilling — re-solve 3 lowest-confidence cold. #dsa
- [ ] **LLD (0h45):** Full cold mock — unseen problem, 45 min in Go, self-critique interface/struct design. #lld

- *Goal: attack a cold LLD with structure.*

### Day 23 — Wed 29 Jul — 2 hrs
- [ ] **DSA (1h30):** Timed set — 2 mediums + 1 hard, 60 min. #dsa
- [ ] **HLD (0h30):** Mock on your own system — present Kafka pipeline as formal HLD; prep deep-dive follow-ups. #hld

- *Goal: turn real experience into a rehearsed answer.*

### Day 24 — Thu 30 Jul — 2 hrs
- [ ] **DSA (1h):** Full mock round — 2 problems, 45 min, talk out loud. #dsa
- [ ] **HLD (1h):** Full mock — cold case, 45 min. Record yourself; watch for rambling. #hld

- *Goal: real-round simulation; communication tightened.*

### Day 25 — Fri 31 Jul — 2 hrs
- [ ] **DSA (1h):** Two mock rounds — 2×(2 problems, ~45 min), different patterns. #dsa
- [ ] **LLD (1h):** Full mock + review — cold problem, 45 min, then critique. #lld

- *Goal: back-to-back stamina building.*

### Day 26 — Sat 1 Aug — 4.5 hrs 🟢 weekend
- [ ] **DSA (2h):** Two full mock rounds — 2×(2 problems, 45 min). #dsa
- [ ] **LLD (1h15):** Full mock + review. #lld
- [ ] **HLD (1h15):** Full mock, then list every deep-dive an interviewer could push + prep answers. #hld

- *Goal: endurance for real 3–5 hr loops.*

### Day 27 — Sun 2 Aug — 4.5 hrs 🟢 weekend
- [ ] **DSA (1h30):** Weak-spot final pass — re-solve 6 lowest-confidence one last time. #dsa
- [ ] **LLD (1h):** Pattern→problem cheat-sheet — one page: pattern + problems + Go template. #lld
- [ ] **HLD (1h30):** Rehearse 5 best cases — TinyURL, Distributed Rate Limiter, Chat, News Feed, your Kafka system. Each 35 min. #hld

- *Goal: everything consolidated into quick-reference form.*

### Day 28 — Mon 3 Aug — 2 hrs
- [ ] **DSA (1h15):** Light mixed set — 2–3 mediums, relaxed. Stay warm. #dsa
- [ ] **HLD (0h45):** Behavioral / STAR — stories: team exits + extra workload, hard technical decision, production incident. #hld

- *Goal: stay sharp; behavioral covered.*

### Day 29 — Tue 4 Aug — 2 hrs
- [ ] **DSA (1h):** Final mock — 2 problems, 45 min. #dsa
- [ ] **Review (1h):** Tracker sweep — re-read approaches for 1–2 confidence problems; skim LLD cheat-sheet + HLD notes.

- *Goal: confidence pass. Taper begins.*

### Day 30 — Wed 5 Aug — 2 hrs (light)
- [ ] **Review only (1h30):** Skim DSA templates, LLD pattern→problem map, top-5 HLD cases, your own-experience stories. No new problems.
- [ ] **Logistics (0h30):** Resume polished, LinkedIn updated, referral list ready, applications queued.

- *Goal: Ready. Rested. Applying.*

---

## ✅ What "ready" looks like at Day 30

> [!check] Tier-2 (guaranteed bar)
> DSA — easy < 10 min, medium < 25 min, cold, in C++ · LLD — clean interface/struct design + core code in Go in 45 min · standard HLD case with structure and depth.

> [!check] Tier-1 (stretch — now reachable)
> Most mediums < 20 min, viable approach on hards · concurrency/atomicity follow-ups without freezing · HLD deep-dives backed by your real Kafka/multi-tenant/Beckn experience.

---

## 🛟 If workload eats your time

> [!warning] Fallback rules
> - **1-hr day:** DSA block only; pick up LLD/HLD on the weekend.
> - **Zero-hr day:** max twice in 30 days; re-solve 1 problem on your phone during commute.
> - **Weekends are the buffer:** lose 3+ weekday hours → reclaim Saturday. Never let DSA go untouched >2 days.

> [!tip] Apply in parallel — from ~Day 8 (14 Jul)
> Pipelines take 2–4 weeks to reach onsite, so early applications land interviews as your prep peaks. Target GCCs + fintech/infra (domain fit), activate referrals, use Wellfound/Cutshort. Don't wait until Day 30.

---

*Related: [[Golang]] · [[Interview Schedule Planning]]*
