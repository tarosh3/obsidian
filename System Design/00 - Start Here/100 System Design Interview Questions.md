---
title: "100 System Design Interview Questions"
aliases: [System Design Rapid Fire, Common Interview Questions, System Design FAQ]
tags: [system-design, interview-prep, cross-cutting]
status: reference-quality
---

# 100 System Design Interview Questions

> [!abstract] What this file is, and why it's separate from every other chapter
> Every HLD/LLD chapter in this handbook already ends with an "Interview Q&A" section — but those are **specific** to that one case study. This file is different: it's the set of questions that recur across **almost every** system design interview, regardless of which system is being designed — "how do you handle a million requests," "what if a server fails," "two people booking the same seat." Drill this the night before an interview; read the full chapters for depth on any answer that feels thin.

> [!tip] How to use this
> Each question is collapsed — click to expand. Every answer has three parts: the **technical answer**, a **layman analogy** (explain it to a non-engineer, the same way you'd explain it to yourself at 2am before you've had coffee), and a **concrete example** linked to the full chapter. Where a full chapter exists, it's linked — go there for the complete derivation.

---

## 1. Scale & Traffic

> [!question]- 1. How do you handle a system that suddenly needs to serve a million requests per second?
> No single lever — combine horizontally scaling the stateless app tier behind a load balancer, caching reads in front of the database, and sharding/replicating the database so no single node is the bottleneck. Identify which layer breaks *first* under load (usually the database) and scale that layer specifically rather than uniformly scaling everything.
> **Layman:** like a single cashier suddenly facing a stadium of customers — you open more checkout counters (more servers), let regulars grab pre-bagged items without waiting (cache), and split the warehouse into sections so no single storeroom gets swarmed (sharding).
> **Example:** [[HLD/06 - Design Twitter - News Feed/Design Twitter - News Feed|Twitter's News Feed chapter]] — the real fix for feed-read-at-scale was fan-out-on-write (precompute feeds ahead of time), not "add more servers," since the bottleneck was fan-out computation, not raw request-handling capacity.

> [!question]- 2. Vertical vs. horizontal scaling — when do you pick which?
> Vertical (bigger machine) is simpler, zero data-distribution complexity, but hits a hard ceiling and stays a single point of failure. Horizontal (more machines) has no ceiling and gives redundancy for free, but requires the system to be designed for distribution (stateless services, partitioned data) from the start.
> **Layman:** vertical is hiring one super-strong employee to do more work; horizontal is hiring more employees. The super-employee eventually can't get any stronger; more employees can keep growing, but now need a manager to coordinate them.
> **Example:** a single-node Postgres can scale vertically for a while; past a point (write throughput exceeds one machine), horizontal sharding becomes unavoidable.

> [!question]- 3. How do you scale reads differently from writes?
> Reads scale via replication and caching — many copies of the same data. Writes are harder: they must eventually land somewhere consistent, so the fix is sharding (partition writes across many primaries) or offloading to async processing where possible.
> **Layman:** reads are like photocopying a poster — make as many copies as you like. Writes are like actually painting the original — only one canvas holds the "true" version at a time, so more painters means more coordination, not just more copies.
> **Example:** [[HLD/23 - Design an E-commerce System/Design an E-commerce System|E-commerce's search index]] — reads hit a horizontally-replicated Elasticsearch cluster; writes flow through one ordered CDC pipeline.

> [!question]- 4. What's the standard fix for a sudden traffic spike (flash sale, viral post)?
> Combine caching hot content, rate limiting/admission control to protect backends, and horizontal auto-scaling — but auto-scaling lags (new instances take time to spin up), so caching and admission control matter more for the first few seconds than scaling does.
> **Layman:** Black Friday at a store — keep popular items already up front (cache), put a bouncer at the door instead of letting everyone flood in at once (rate limit), and call in extra staff (auto-scale) even though organizing that takes a few minutes.
> **Example:** [[HLD/15 - Design a Ticket Booking System/Design a Ticket Booking System|Ticket Booking's waiting room]] admits users in a controlled trickle specifically because scaling alone can't absorb 100K simultaneous users against 500 seats fast enough.

> [!question]- 5. How do you reduce load on your primary database?
> Cache reads (cache-aside by default), add read replicas, move non-critical writes to async processing, archive cold data out of hot tables.
> **Layman:** instead of asking the librarian for the same popular book over and over, keep a copy on the front desk so most people never bother the librarian at all.
> **Example:** a "like count" doesn't need a synchronous DB write per click — batch/aggregate and flush periodically.

> [!question]- 6. When do you introduce caching, and where in the stack?
> When data is read far more often than it changes, and recomputing/refetching is expensive. Where: closest to the consumer that still keeps the cache reasonably fresh — CDN edge for static assets, Redis for computed/DB-backed data, sometimes local in-process cache for extremely hot, small data.
> **Layman:** keep milk in your kitchen fridge (close, fast), not in a warehouse across town (origin server) — but only for things you actually use often; no point keeping rarely-touched stuff nearby.
> **Example:** YouTube video metadata cached at the CDN edge; a personalized recommendation list cached in Redis since it's expensive to compute but changes faster than static assets.

> [!question]- 7. What happens when your cache is down?
> Depends on the fail-open/fail-closed decision (see Reliability section) — for most read-through caches, requests fall through to the database directly, taking the *full* load the cache was absorbing, which can cascade into a database outage.
> **Layman:** if the express checkout lane closes, everyone floods to the regular checkout lane — and if the regular lane wasn't built to handle everyone at once, it jams too.
> **Example:** exactly why [[HLD/03 - Design a Distributed Cache (build Redis)/Design a Distributed Cache (build Redis)|the Distributed Cache chapter]] and [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|Rate Limiter chapter]] both discuss fail-open behavior explicitly — the cache's own outage shouldn't take down what it protects.

> [!question]- 8. How do you prevent a cache stampede (thundering herd) on cache expiry?
> A popular key expiring causes many concurrent requests to all miss simultaneously and hit the database at once. Fix: request coalescing (only one request actually goes to the DB, others wait for its result) or staggered TTLs with jitter.
> **Layman:** a popular item runs out on a shelf and 500 people simultaneously ask the stock room for a refill — instead, put ONE person in charge of asking the stock room, everyone else just waits on that one answer.
> **Example:** Go's `singleflight` package, referenced directly in [[CS Fundamentals/Caching/Caching Strategies|Caching Strategies]].

> [!question]- 9. How would you design for a system whose load is extremely spiky (e.g., 100x traffic at 12pm daily)?
> Predictable spikes get pre-scaled ahead of time (scheduled scaling), not left to reactive auto-scaling which lags the spike's start. Unpredictable spikes need aggressive caching and admission control as the first line of defense since scaling reaction time can't be trusted.
> **Layman:** a restaurant staffs up BEFORE the lunch rush starts, not after customers are already lined up out the door.
> **Example:** [[HLD/16 - Design a Food Delivery System/Design a Food Delivery System|Food Delivery's]] lunch/dinner rush explicitly budgets for a predictable 5-10x peak, distinct from steady-state average.

> [!question]- 10. What's the difference between scaling the application layer and scaling the database layer?
> The app layer is usually stateless and trivially horizontally scalable — add more identical instances behind a load balancer. The database holds state, so scaling means partitioning (sharding) or replicating, both introducing real consistency and operational complexity the app layer doesn't have.
> **Layman:** adding more waiters is easy — just hire more, they're interchangeable. Adding more kitchens is hard — you have to decide which ingredients go in which kitchen.
> **Example:** adding a 10th API server is a non-event; adding a 10th database shard needs a real resharding plan — [[CS Fundamentals/Distributed Systems/Sharding & Partitioning|Sharding & Partitioning]].

> [!question]- 11. How do you handle a "hot" API endpoint that a small number of clients hit disproportionately?
> Per-client rate limiting isolates the heavy client from affecting others; if the load is legitimate (a celebrity account, a viral resource), that specific resource may need its own dedicated caching or sharding treatment rather than the general solution.
> **Layman:** one incredibly popular restaurant table getting reserved nonstop while others sit empty — give it its own dedicated host, or a stricter limit on how often that ONE table can be booked.
> **Example:** Twitter's celebrity-follower fan-out problem — handled with a hybrid push/pull model specifically for high-follower accounts, not the general fan-out-on-write path.

> [!question]- 12. If your system needs to grow 10x in the next year, what would you change today?
> Identify which component has the least scaling headroom under the current design and address that first — usually the database (hardest to reshard later) — rather than uniformly future-proofing everything. Also make sure the app layer is genuinely stateless now, since retrofitting statelessness later is much harder.
> **Layman:** if you know your shop will get 10x busier, you don't repaint the walls first — you check whether the front door is wide enough for the crowd.
> **Example:** deliberately a judgment question — interviewers want you to identify the actual bottleneck, not recite a scaling checklist.

## 2. Failure & Reliability

> [!question]- 13. What happens if one of your application servers fails?
> The load balancer's health checks detect it (missed heartbeats/failed probe) and stop routing traffic to it; since the app layer is stateless, any other server picks up the next request with no data loss.
> **Layman:** one cashier goes on break — the "next available cashier" sign just sends the next customer to a different open lane; nobody's order is lost.
> **Example:** standard practice across every HLD chapter in this book — the app tier is designed stateless specifically so this is a non-event.

> [!question]- 14. What happens if the load balancer itself fails?
> A single LB is itself a single point of failure — production deployments run multiple LB instances reached via DNS round-robin or anycast.
> **Layman:** if the "next available cashier" sign itself breaks, customers don't know where to go — so stores actually have two or three such signs, not just one.
> **Example:** covered directly in [[CS Fundamentals/Networking/Load Balancing|Load Balancing]] — a commonly-missed detail candidates forget.

> [!question]- 15. What happens if your primary database fails?
> A leader-follower setup fails over by promoting a replica — automatically (via a consensus coordinator like Raft/etcd) or via a runbook. Real gap: writes not yet replicated at the moment of failure are lost unless synchronous replication was used (at a latency cost).
> **Layman:** the head chef collapses — the sous chef takes over, but any order the head chef hadn't yet told the sous chef about is lost, unless every order was double-written in real time (which slows the kitchen down).
> **Example:** the direct real-world instance of the tradeoff in [[CS Fundamentals/Distributed Systems/CAP Theorem & PACELC|CAP Theorem & PACELC]].

> [!question]- 16. How do you detect that a server or node is down?
> Heartbeats/health checks — a node periodically reports "I'm alive," or is periodically polled; missing a threshold of consecutive heartbeats marks it unhealthy. Too sensitive causes false positives (flapping); too lax delays real detection.
> **Layman:** a lifeguard doing roll call — if someone doesn't answer a few times in a row, assume they're not there anymore. Too strict causes false alarms for someone who just stepped away; too lax lets a real problem go unnoticed too long.
> **Example:** [[Glossary/Gossip Protocol|Gossip Protocol]] — nodes propagate liveness info to each other rather than all polling one central checker, avoiding that checker becoming its own single point of failure.

> [!question]- 17. What's the difference between fail-open and fail-closed, and when do you choose each?
> Fail-open: when a protection layer breaks, default to *allowing* traffic through. Fail-closed: default to *blocking*. Choose fail-open when the protected system tolerates unprotected traffic better than an outage (a rate limiter's own Redis dying). Choose fail-closed when letting unsafe traffic through is worse (an auth check failing).
> **Layman:** if a metal detector breaks at a casual event, you might just wave people through (fail-open) rather than shut the whole event down. At an airport, you'd shut the gate instead (fail-closed) — the cost of letting something bad through is much higher there.
> **Example:** [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|Rate Limiter]] fails open on its own dependency — better to let traffic through unthrottled briefly than reject everything because Redis is down.

> [!question]- 18. How do you handle a network partition between two data centers?
> Depends on the CAP choice already made. A CP system (Raft/Paxos-based) stops accepting writes on the minority side to preserve consistency. An AP system keeps accepting writes on both sides and reconciles once the partition heals.
> **Layman:** two offices of the same company lose their phone line to each other — the bigger office (majority) keeps making decisions; the smaller, cut-off office (minority) pauses rather than risk making conflicting decisions nobody can reconcile later.
> **Example:** [[CS Fundamentals/Distributed Systems/Consensus (Raft & Paxos)|the Consensus chapter]] walks through exactly what happens on each side of a Raft cluster partition.

> [!question]- 19. What is a single point of failure, and how do you find them in a design?
> Any component whose failure takes down the whole system because nothing else can take over. Find them by tracing every request path end-to-end and asking "if this one box died right now, what breaks" — including easy-to-forget ones like the load balancer or DNS.
> **Layman:** a bridge that's the ONLY way into town — doesn't matter how good the roads inside town are if the bridge collapses. Walk every road on the map and ask "if this one thing breaks, is the town cut off?"
> **Example:** a common trap — candidates design a redundant app/database tier and forget the load balancer in front is a single instance.

> [!question]- 20. What's graceful degradation, and how would you design for it?
> When a non-critical dependency fails, the system keeps serving its core function with reduced quality rather than failing entirely — requires identifying "core" vs. "nice to have" ahead of time.
> **Layman:** if the restaurant's dessert-menu printer breaks, you still serve full dinners — you just can't show dessert options for a bit. The kitchen keeps running.
> **Example:** if a recommendation service is down, an e-commerce site should still let users search and check out — recommendations degrade, not the whole site.

> [!question]- 21. What's a retry storm, and how do you prevent it?
> When a dependency starts failing, every caller retrying immediately *adds to* the load on the already-struggling dependency, worsening the original problem. Prevented by exponential backoff with jitter and a circuit breaker.
> **Layman:** a struggling waiter drops a tray, and instead of waiting a moment, everyone immediately re-orders at once — burying the already-overwhelmed waiter further. Waiting a random, growing amount of time before re-ordering gives them room to recover.
> **Example:** full mechanism and Go implementation in [[CS Fundamentals/Distributed Systems/Resilience Patterns|Resilience Patterns]].

> [!question]- 22. How do you handle a "poison pill" — a malformed message that keeps crashing your consumer?
> Without a limit, the consumer retries the same bad message forever, blocking everything behind it. Fix: a max-retry count, after which the message moves to a dead-letter queue for separate investigation.
> **Layman:** one jammed envelope stuck in the mail sorting machine, and the machine keeps trying to process that SAME envelope forever, blocking every letter behind it. Set it aside after a few tries and keep the rest of the mail moving.
> **Example:** [[CS Fundamentals/Messaging & Streaming/RabbitMQ Internals|RabbitMQ Internals']] dead-letter-exchange discussion.

> [!question]- 23. If a downstream service you depend on becomes very slow (not fully down), what do you do?
> A slow dependency is often worse than a dead one — naive code waits, piling up threads/connections waiting on it, exhausting the *caller's* own resources. Fix: a strict timeout, a circuit breaker, and a bulkhead isolating its resource pool from healthy dependencies.
> **Layman:** worse than a closed shop is a painfully slow one — customers queue up waiting, tying up your own staff who could be helping elsewhere. Set a maximum wait time, stop sending people there once it's clearly too slow, and don't let it block staff who could help with something else.
> **Example:** the exact three-pattern combination in [[CS Fundamentals/Distributed Systems/Resilience Patterns|Resilience Patterns]].

> [!question]- 24. How do you design a system to survive an entire data center/region going down?
> Multi-region deployment with data replicated across regions, plus a mechanism (DNS failover, global load balancer) to redirect traffic. The hard part isn't detecting the outage — it's handling in-flight requests and data that hadn't yet replicated to the surviving region.
> **Layman:** a company with a backup office in another city — if the main office floods, work shifts to the backup, but paperwork that hadn't been faxed over yet before the flood is the tricky part.
> **Example:** [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter|the Multi-Region Rate Limiter chapter]] is a concrete instance of exactly this tradeoff, applied to one component.

## 3. Concurrency & Race Conditions

> [!question]- 25. Two people try to book the last seat/ticket at the exact same time — how do you prevent double-booking?
> The classic check-then-act race: checking "is it available" and "book it" as two separate steps lets both requests pass the check before either commits. Fix: make check-and-claim a *single atomic operation* — a DB row-level lock with conditional update, or a Redis Lua script.
> **Layman:** two people call to book the last hotel room at the same second — if the front desk checks "is it free?" and THEN marks it booked as two separate steps, both callers can hear "yes it's free" before either booking lands. The desk needs to check-and-claim in one uninterruptible motion.
> **Example:** full depth with Go code in [[LLD/06 - Design BookMyShow - Seat Booking/Design BookMyShow - Seat Booking|BookMyShow LLD]] — reused directly in Uber's driver matching and the Ticket Booking HLD chapter.

> [!question]- 26. Two people try to withdraw the last $10 from a shared account at the same time — same problem?
> Yes, structurally identical to seat-booking — an atomic "check balance and debit" operation (not two separate read-then-write steps) is the fix.
> **Layman:** two family members with the same joint account card both try to withdraw the last $10 at two different ATMs at once — the bank must treat "check balance, then subtract" as one atomic step, not two.
> **Example:** same fix pattern as [[LLD/11 - Design an ATM/Design an ATM|the ATM LLD chapter's]] balance-check-and-withdraw logic.

> [!question]- 27. How do you implement a distributed lock, and what can go wrong with it?
> Typically Redis (`SETNX` + TTL) or a consensus system (ZooKeeper/etcd). What goes wrong: a process holding the lock can pause (GC pause, slow I/O) longer than the TTL — the lock expires and is granted to someone else, so *two* processes now believe they hold it.
> **Layman:** a bathroom key at a coffee shop that only one person can hold — the danger is if the current key-holder falls asleep in there, the shop might assume they left and hand the key to someone else, and now two people think they have exclusive access.
> **Example:** the fencing-token fix — full detail in [[HLD/18 - Design a Distributed Lock Service/Design a Distributed Lock Service|Distributed Lock Service]].

> [!question]- 28. What's a "check-then-act" race condition, generically?
> Any sequence checking a condition then acting on it, with a gap during which another process can invalidate the check. The generic fix is always the same shape: collapse check and act into one atomic operation.
> **Layman:** "look both ways, then cross" only works if nothing changes in the gap between looking and stepping. The fix everywhere is the same: don't leave a gap.
> **Example:** seat booking, bank withdrawal, inventory decrement (E-commerce), driver assignment (Uber) — literally the same bug pattern reused across unrelated domains.

> [!question]- 29. A user double-clicks "Pay Now" — how do you prevent a duplicate charge?
> Idempotency keys — the client generates a unique key per logical operation; the server checks if it already processed that key and returns the original result instead of re-processing.
> **Layman:** writing "reference #1234" on a check — if the bank somehow receives the check twice, they recognize it's the same payment and don't cash it twice.
> **Example:** built in full depth in [[HLD/17 - Design a Payment System/Design a Payment System|the Payment System chapter]] — Stripe's actual idempotency-key design is the reference model.

> [!question]- 30. What is idempotency, and why does it matter beyond "double-click" protection?
> An idempotent operation produces the same result no matter how many times it's applied. It matters because network retries are unavoidable at scale (a timeout doesn't tell you if the original request succeeded) — without idempotency, a *safe* retry becomes a dangerous double-execution.
> **Layman:** pressing an elevator call button five times because you're not sure it registered doesn't send five elevators — pressing it repeatedly has the same effect as pressing it once.
> **Example:** any at-least-once delivery system (Kafka, most queues) requires idempotent consumers, since delivery guarantees alone don't prevent duplicate processing.

> [!question]- 31. How do you generate unique IDs across many servers without collisions or a central bottleneck?
> A centralized auto-increment counter becomes a bottleneck/SPOF. Fix: Snowflake IDs — each ID embeds a timestamp, machine ID, and local sequence number, letting any server generate globally unique, roughly time-sortable IDs with zero coordination.
> **Layman:** instead of one central ticket-number dispenser everyone has to queue at, give each branch office its own numbered ticket books stamped with a branch code and timestamp — no two branches ever hand out the same number, and nobody calls a central office first.
> **Example:** full bit-layout implementation in [[LLD/20 - Design a Distributed ID Generator/Design a Distributed ID Generator|Distributed ID Generator]].

> [!question]- 32. How would you handle two concurrent updates to the same row/record?
> Pessimistic locking (lock the row for the read-modify-write, blocking others) or optimistic concurrency control (read a version, write back only if unchanged, retry on conflict). Optimistic wins under low contention; pessimistic is safer under high contention.
> **Layman:** two editors trying to edit the same paragraph of a shared document at once — either lock the paragraph so only one edits at a time (pessimistic), or let both try, and whoever saves last checks "did anyone change this since I started" and redoes it if so (optimistic).
> **Example:** MVCC (Postgres, MySQL InnoDB) is a sophisticated form of optimistic concurrency — [[CS Fundamentals/Databases/SQL Query Execution Deep Dive|SQL Query Execution Deep Dive]].

> [!question]- 33. What's a race condition, precisely, and how is it different from a general bug?
> A bug whose existence and behavior depend on the specific *timing/interleaving* of concurrent operations — it may pass every test run under light load and fail only under real concurrent traffic.
> **Layman:** a magic trick that only glitches if two things happen at exactly the wrong moment relative to each other — works every time you test it slowly, and mysteriously breaks only under a real, busy crowd.
> **Example:** the seat-booking bug is invisible in a single-threaded test and only manifests under actual concurrent requests to the same seat.

> [!question]- 34. How do you make operations across multiple services atomic (all succeed or all fail)?
> Two-Phase Commit (strong atomicity, blocks if the coordinator fails mid-transaction) or the Saga pattern (each step has a compensating action to undo it if a later step fails — eventually consistent, no cross-service locking for the whole duration).
> **Layman:** booking a flight + hotel + rental car together — if the car booking fails after the flight and hotel already went through, you either hold everything hostage until all three confirm together (2PC, risky if one party goes silent), or let each book independently and CANCEL the ones that succeeded if a later one fails (Saga).
> **Example:** [[HLD/23 - Design an E-commerce System/Design an E-commerce System|E-commerce's checkout saga]] — inventory reservation, payment, order creation, each with a real compensating-action story.

> [!question]- 35. If optimistic locking keeps losing the race under high contention, what do you do?
> A raw infinite-retry loop can starve indefinitely. Cap retries with backoff; if contention is consistently high on one record, that's a signal that record needs a different concurrency strategy (e.g., a serialized-write queue for just that key).
> **Layman:** if two people keep trying to grab the same popular parking spot and keep bumping into each other, endlessly retrying wastes everyone's time — eventually it's better to just make one of them wait in a line for that specific spot.
> **Example:** the same underlying idea as the hot-shard-key fix in [[CS Fundamentals/Distributed Systems/Sharding & Partitioning|Sharding & Partitioning]] and [[CS Fundamentals/Databases/MongoDB Internals|MongoDB Internals]].

## 4. Consistency & Data

> [!question]- 36. Strong vs. eventual consistency — when do you pick which?
> Strong consistency (every read sees the latest write) is needed when stale data causes real harm — a bank balance, a seat's availability at booking time. Eventual consistency (reads may briefly see stale data) is fine, and much cheaper, when brief staleness doesn't matter — a like count, a feed.
> **Layman:** your bank balance must be exactly right the instant you check it (strong). A "500K likes" counter being off by a few for a minute doesn't hurt anyone (eventual) — the looser rule is cheaper and faster.
> **Example:** BookMyShow's browsing seat map (eventually consistent) vs. the actual booking action (strongly consistent) — a deliberate split within one feature.

> [!question]- 37. How do you keep a cache and its database in sync?
> No perfect answer. Cache-aside (read from cache, on miss read DB and populate) tolerates brief staleness after a write. Write-through updates cache and DB synchronously, closing that gap at the cost of write latency.
> **Layman:** a whiteboard copy of today's specials in the front window vs. the master menu in the kitchen — update the whiteboard right away (write-through, slower to update) or let it go stale for a bit until someone refreshes it (cache-aside, faster but momentarily wrong).
> **Example:** full comparison table in [[CS Fundamentals/Caching/Caching Strategies|Caching Strategies]].

> [!question]- 38. What happens if a write succeeds on the primary but fails to replicate to a read replica?
> A read hitting that lagging replica returns stale data — this is concretely what "eventual consistency" means in a replicated system. If unacceptable, that read must go to the primary directly, at the cost of adding load back to it.
> **Layman:** telling one friend big news, and them not yet having told a second friend — if you ask the second friend, they'll give you old information until the news catches up to them.
> **Example:** replication lag is a real, monitorable metric — mentioned in several chapters' production-experience sections.

> [!question]- 39. When is a stale read actually OK, and how do you decide?
> Ask: "if a user sees data a few seconds/minutes old, does anything break?" If no (follower count, dashboard metric), stale reads are a fine, deliberate tradeoff. If yes (account balance, seat availability at purchase), it isn't.
> **Layman:** a stale sports score being 10 seconds old bothers nobody. A stale bank balance saying you have money you already spent is a real problem.
> **Example:** exactly PACELC's "Else" branch, applied per use case rather than as one global policy.

> [!question]- 40. What's a "hot" key/partition, and why is it a consistency *and* a performance problem?
> One key/record receiving disproportionate traffic — even with data evenly hash-distributed, a single celebrity user/product/room concentrates all its traffic on one shard, since hashing distributes *different* keys evenly but can't split *one* key's traffic.
> **Layman:** splitting customers evenly across 10 checkout lanes works great — until one single celebrity customer needs 1,000 items rung up, and whichever lane they're in is now swamped no matter how evenly everyone ELSE was spread out.
> **Example:** [[CS Fundamentals/Databases/MongoDB Internals|MongoDB Internals]] — fixed by adding entropy to the key (sub-sharding a hot key into several sub-keys).

> [!question]- 41. How do you handle conflicting writes to the same data from two different regions?
> Needs an explicit conflict-resolution strategy: last-write-wins (simple, can silently lose a legitimate write), vector clocks (detect the conflict, defer resolution), or CRDTs (mathematically designed to merge conflicting updates without losing information).
> **Layman:** two co-authors on opposite sides of the world both edit the same sentence of a shared doc while offline, then reconnect — the system needs a rule for which edit "wins," or a way to flag "hey, you two both changed this, please look."
> **Example:** the harder, real-world version of the multiplayer collaborative-editing problem mentioned as a follow-up in [[LLD/15 - Design a Text Editor/Design a Text Editor|the Text Editor LLD chapter]].

> [!question]- 42. What's the difference between read-your-own-writes consistency and full strong consistency?
> Read-your-own-writes guarantees a user always sees *their own* recent write immediately (others might briefly see stale data) — a cheaper, often-sufficient middle ground achieved by routing a user's own reads to the primary right after they write.
> **Layman:** you post a comment and it shows up on YOUR screen right away, even if your friend across the world doesn't see it for another second — the system makes sure you never see your own action get "undone" from your point of view.
> **Example:** a comment you just posted appears instantly on your own screen, even if it takes a moment to propagate to everyone else's feed.

> [!question]- 43. How would you design a "seen it already" / deduplication check at scale, without storing every ID forever?
> A Bloom filter — a compact probabilistic structure answering "definitely not seen" or "possibly seen" with no false negatives, using far less memory than a full ID set, at the cost of a small, tunable false-positive rate.
> **Layman:** a bouncer with a rough mental list of "definitely haven't seen this face before" vs. "maybe I have" — cheap, small memory, occasionally double-checks someone who wasn't actually there before, but never lets in someone who was already flagged.
> **Example:** [[Glossary/Bloom Filter|Bloom Filter]] and the cache-penetration-prevention use case in [[CS Fundamentals/Caching/Caching Strategies|Caching Strategies]].

> [!question]- 44. How do you count something "roughly" at massive scale without exact bookkeeping?
> Purpose-built probabilistic structures: HyperLogLog for approximate *distinct* counts, Count-Min Sketch for approximate *frequency* counts — both trade a small, quantifiable error for fixed, tiny memory regardless of true cardinality.
> **Layman:** instead of literally writing down every single visitor's name to count "how many different people visited" (an expensive notebook), use a clever shortcut that gives you a very-close-enough number using a tiny scrap of paper.
> **Example:** full treatment, including the "can't delete data" limitation, in [[HLD/24 - Design an Analytics Aggregation System/Design an Analytics Aggregation System|the Analytics Aggregation System chapter]].

> [!question]- 45. What's a quorum, and why does R + W > N guarantee consistency?
> With N replicas, requiring R for a read and W for a write to agree, if R + W > N, any read set and any write set are mathematically guaranteed to overlap by at least one replica — a read can never completely miss the most recent write.
> **Layman:** a group decision needs enough overlap between "who voted last time" and "who's voting now" that at least one person in the new vote definitely remembers the last outcome — that overlap is what quorum guarantees mathematically.
> **Example:** [[Glossary/Quorum (R + W over N)|Quorum]] and the tunable-consistency discussion in [[CS Fundamentals/Databases/Cassandra Internals|Cassandra Internals]].

## 5. Database Design

> [!question]- 46. SQL or NoSQL — how do you decide?
> SQL when data is relational, needs strong ACID transactions, and the schema is relatively stable. NoSQL when you need to scale writes horizontally beyond a single relational primary, the access pattern is simple/key-based, or the schema needs to evolve flexibly.
> **Layman:** SQL is a well-organized filing cabinet with strict labeled folders (great when relationships between things matter). NoSQL is more like sticky notes on a giant wall — fast to add anywhere, less structured, better when you just need speed and flexibility over millions of notes.
> **Example:** Splitwise (genuinely relational — debts, groups, users) vs. a chat message store (simple key-based access, needs horizontal scale — Cassandra fits better).

> [!question]- 47. How do you shard a database, and what actually breaks when you do?
> Pick a shard key (see [[CS Fundamentals/Distributed Systems/Sharding & Partitioning|Sharding & Partitioning]] for range/hash/directory tradeoffs). What breaks: cross-shard joins/transactions lose single-node ACID guarantees, and any query not filtering by the shard key scatter-gathers across every shard.
> **Layman:** splitting one giant filing cabinet into ten smaller cabinets, one per last-name range — fast if you know exactly whose file you want, but "find everyone born in March" now means checking all ten cabinets instead of one.
> **Example:** sharding `users` by `user_id` makes "get this user" fast, but "find all users in this city" now queries every shard unless a secondary index exists.

> [!question]- 48. How do you run a schema migration on a live, massive table without downtime?
> Never a single blocking `ALTER TABLE`. Add the new column as nullable (fast, metadata-only), backfill in small batches in the background, switch application code to the new column, then drop the old one.
> **Layman:** renovating a busy restaurant's kitchen without closing — add the new stove first, test it quietly, get everyone cooking on it, THEN remove the old stove — never rip everything out at once mid-dinner-rush.
> **Example:** a standard staged-migration discipline — a strong signal of real production experience.

> [!question]- 49. What is replication lag, and why does it matter operationally?
> The delay between a write landing on the primary and appearing on a replica. Any read routed to a lagging replica can return stale data — unbounded growth in lag signals the replica can't keep up with write volume, a real capacity problem.
> **Layman:** a rumor spreading through an office — if it's taking unusually long to reach the far corner, that corner keeps acting on old information longer than expected.
> **Example:** called out explicitly as a "what to monitor" item across multiple HLD chapters.

> [!question]- 50. How do you back up a huge production database without taking it offline?
> Snapshot-based backups (storage-layer copy-on-write, near-instant) or backing up from a dedicated replica rather than the primary, so backup I/O doesn't compete with live traffic.
> **Layman:** photocopying a book by scanning a duplicate copy on a separate table, not by yanking the original out of the reader's hands mid-page.
> **Example:** standard cloud practice (e.g. AWS RDS snapshots) — worth naming even briefly.

> [!question]- 51. Why are B+ Trees used for indexes instead of a plain binary search tree or hash table?
> B+ Trees are shallow and wide (high fan-out), designed around disk-page reads — each node read is one disk I/O, so a shallow tree means very few I/Os across millions of rows. A BST is far deeper; a hash index can't support range queries or sorted scans.
> **Layman:** a phone book organized so you can find any name in 3-4 flips of the page, versus a phone book listed in the order people signed up, where you'd have to check every single page.
> **Example:** full internals, including why leaf nodes are linked for range scans, in [[CS Fundamentals/Databases/Indexes & B+ Trees|Indexes & B+ Trees]].

> [!question]- 52. What's the difference between a clustered and a non-clustered index?
> A clustered index determines the physical storage order of rows — only one per table (usually the primary key). A non-clustered index is a separate structure pointing back to the row's location — a table can have many.
> **Layman:** clustered = the book's pages are physically printed in alphabetical order (only one way to sort the actual pages). Non-clustered = a separate index-card box pointing to which page each name is on — you can have many different index boxes, sorted different ways, without reprinting the book.
> **Example:** covered with the leftmost-prefix-rule follow-up in [[CS Fundamentals/Databases/Indexes & B+ Trees|Indexes & B+ Trees]].

> [!question]- 53. What happens internally when you run a slow SQL query, and how do you speed it up?
> The query goes through parse → plan → optimize → execute; `EXPLAIN ANALYZE` reveals the chosen plan — commonly a slow query is doing a full table scan because no usable index exists, or an index isn't used due to a type mismatch or a function wrapped around the indexed column.
> **Layman:** asking "why did it take so long to find this book?" — usually the assistant walked every single shelf instead of using the card catalog that would've pointed straight to it.
> **Example:** full pipeline breakdown in [[CS Fundamentals/Databases/SQL Query Execution Deep Dive|SQL Query Execution Deep Dive]].

> [!question]- 54. Why would a database choose a full table scan even when an index exists?
> The query planner estimates cost — if the filter matches a large fraction of rows, sequential scanning can genuinely be cheaper than the random I/O of jumping through an index for that many matches. A deliberate optimizer decision, not a bug.
> **Layman:** if you need almost every book in the library anyway, flipping through the card catalog for each one is actually slower than just walking the shelves once — sometimes skipping the shortcut is genuinely faster.
> **Example:** exactly the nuance separating "I understand index internals" from a shallow "just add an index" answer.

> [!question]- 55. How would you design a schema to support "undo" of a financial transaction, weeks later?
> Never physically delete/overwrite financial records — model transactions as an append-only ledger; an "undo" is a *new*, opposite transaction referencing the original, not a mutation of history. Preserves a full, immutable audit trail.
> **Layman:** an accountant never erases a line in the ledger — a mistake gets a NEW correcting entry added below it, so anyone auditing later can see exactly what happened and when.
> **Example:** the same principle underlies [[HLD/17 - Design a Payment System/Design a Payment System|the Payment System's]] transaction model.

## 6. Caching

> [!question]- 56. What's the difference between cache-aside, write-through, and write-back?
> Cache-aside: read from cache, on miss read DB and populate; writes go to DB, cache invalidated/expires. Write-through: every write hits cache and DB synchronously, always in sync, slower writes. Write-back: writes hit cache first, async-flushed to DB later — fastest writes, risks data loss if cache fails before flushing.
> **Layman:** cache-aside = check your notebook first, only walk to the archive room if it's not written down yet. Write-through = every time you file something in the archive, you also immediately copy it into your notebook. Write-back = you jot it in your notebook first and walk it to the archive later in a batch — fastest for you, risky if your notebook gets lost before you file it.
> **Example:** full comparison in [[CS Fundamentals/Caching/Caching Strategies|Caching Strategies]].

> [!question]- 57. LRU vs. LFU eviction — when does each make more sense?
> LRU (evict least-recently-used) assumes recent access predicts near-future access — good general default. LFU (evict least-frequently-used) is better when some items are persistently popular regardless of recent gaps — LRU can wrongly evict a globally popular item just because it wasn't touched in the last few seconds.
> **Layman:** LRU throws out whatever you haven't touched in the longest time (clearing out the fridge item sitting untouched longest). LFU throws out whatever you use LEAST OFTEN overall, even if you happened to grab it recently once.
> **Example:** [[LLD/14 - Design an LRU Cache (pluggable eviction)/Design an LRU Cache (pluggable eviction)|the pluggable-eviction LRU Cache chapter]], designed so the policy can be swapped without changing core logic.

> [!question]- 58. What's cache penetration, and how is it different from a cache stampede?
> Penetration: repeated requests for a key that *doesn't exist* (can never be cached, every request hits the DB) — often from bad input or attack. Stampede: many concurrent requests for a key that *did* exist but just expired. Different fixes: penetration → Bloom filter; stampede → request coalescing.
> **Layman:** penetration is someone repeatedly asking the librarian for a book that doesn't exist in the library at all. Stampede is a hundred people all asking for the SAME real book at the exact moment it gets returned and re-shelved.
> **Example:** both covered side by side in [[CS Fundamentals/Caching/Caching Strategies|Caching Strategies]].

> [!question]- 59. How do you decide what TTL to set on a cached item?
> Balance staleness tolerance against hit rate and backend load — shorter TTL means fresher data but more misses; longer TTL means better hit rate but more staleness risk. For rarely-changing data, a long TTL plus explicit invalidation on write beats relying on a short TTL alone.
> **Layman:** how long you trust a "specials board" before walking to the kitchen to double-check it's still accurate — trust it too long and you might order something no longer available; check too often and you're annoying the kitchen.
> **Example:** [[HLD/15 - Design a Ticket Booking System/Design a Ticket Booking System|Ticket Booking's]] seat-map browsing cache deliberately uses a short TTL because staleness there has a real, if minor, cost.

> [!question]- 60. Why is Redis single-threaded for command execution, and how is it still fast?
> Single-threaded execution avoids locking overhead for concurrent access to shared structures — no parallel execution means no internal race condition to guard against. It stays fast because in-memory ops are already quick, and an event loop (not blocking I/O) handles many concurrent connections despite one execution thread.
> **Layman:** one chef in a small kitchen doing one dish at a time, perfectly, with zero risk of two cooks colliding over the same pan — fast because each dish is quick, not because there are many chefs.
> **Example:** full internals in [[CS Fundamentals/Caching/Redis Internals|Redis Internals]].

> [!question]- 61. What's the difference between Redis and Memcached, and when would you pick each?
> Memcached: simpler, pure key-value, multi-threaded, great raw throughput for simple caching. Redis: rich data structures (sorted sets, lists, hashes) plus persistence, usable as more than a cache.
> **Layman:** Memcached is a simple sticky-note board (just note and retrieve). Redis is a whole filing system with folders, sorted lists, and the ability to keep a permanent copy — more useful, a bit heavier.
> **Example:** [[HLD/19 - Design a Real-Time Leaderboard/Design a Real-Time Leaderboard|the Real-Time Leaderboard chapter]] specifically needs Redis's sorted sets — Memcached couldn't support this at all.

> [!question]- 62. How would you cache a personalized (per-user) result without caching everything separately per user?
> Depends how much of the result is shared across users — cache the shared/expensive-to-compute portion once, layer personalization on top at request time, rather than caching the fully-assembled personalized response per user.
> **Layman:** keep one shared "menu" photocopied for everyone (the expensive-to-print part), and hand-write each customer's personal notes on top at the table, rather than reprinting an entirely custom menu booklet for every customer.
> **Example:** a product page's price/description is identical for everyone and cacheable once; "you might also like" is genuinely personalized and shouldn't be.

> [!question]- 63. What happens to your database if your cache hit rate suddenly drops (e.g., after a cold restart)?
> A flood of requests that would normally hit the cache now all hit the database simultaneously — a self-inflicted cache-stampede-at-scale. Real deployments warm the cache before cutting traffic over to a freshly restarted instance.
> **Layman:** if the express lane suddenly reopens completely empty, everyone floods to the regular counter at once until the express lane restocks — better to quietly restock it BEFORE reopening to the public.
> **Example:** a real, common production incident pattern worth naming explicitly.

## 7. API & Communication Design

> [!question]- 64. REST vs. gRPC vs. GraphQL — when do you pick which?
> REST: simple, cacheable, universal — good public-API default. gRPC: binary, low overhead, strongly-typed contracts — best for internal service-to-service where performance matters. GraphQL: client specifies exactly what fields it needs in one request, avoiding over-fetching and multiple round-trips.
> **Layman:** REST is ordering from a fixed menu (simple, everyone understands it). gRPC is a private phone line between kitchen and warehouse (fast, but both ends need to speak the exact same shorthand). GraphQL is telling the waiter exactly which parts of the meal you want, so you don't get side dishes you didn't ask for.
> **Example:** an internal service mesh often uses gRPC; a public API serving diverse frontends benefits from GraphQL.

> [!question]- 65. What's the N+1 query problem in GraphQL, and how do you fix it?
> Fetching a list, then separately fetching related data for *each* item one at a time, turns 1 logical request into N+1 actual queries. Fixed with a DataLoader-style batching layer — collect individual lookups within one tick and issue one batched query.
> **Layman:** asking for a list of 50 students, then asking the office separately "who's this student's teacher?" 50 times in a row, instead of asking once "give me all 50 students AND their teachers together."
> **Example:** fetching 50 posts and each author naively — 1 query for posts, then 50 for authors, instead of 1 + 1 batched.

> [!question]- 66. Synchronous vs. asynchronous communication between services — tradeoffs?
> Synchronous: simpler to reason about, but couples the caller's availability to the callee's — if the callee is slow/down, the caller blocks. Asynchronous (via a queue): decouples the two, at the cost of added complexity and eventual consistency.
> **Layman:** calling someone and waiting on hold until they answer (sync — you're stuck if they're busy) vs. leaving a voicemail and going about your day, trusting they'll get to it (async — you're free, but no instant answer).
> **Example:** the exact reasoning behind every "decouple via Kafka" fix reused throughout this book — Notification Service, Log Aggregation.

> [!question]- 67. How do you version an API without breaking existing clients?
> Never change the meaning or shape of an existing field/endpoint in place — add new fields as additive/optional, or introduce a new versioned endpoint (`/v2/...`) for breaking changes, keeping the old version alive until clients migrate.
> **Layman:** adding a new dish to a restaurant's menu is fine, nobody's confused. Renaming an existing dish to mean something totally different without warning confuses regulars — so you print a whole NEW menu (v2) instead and keep the old one available until people switch over.
> **Example:** the "additive changes are safe, breaking changes need a new version" rule, stated explicitly rather than just "use versioning."

> [!question]- 68. What do you do if a downstream API doesn't support idempotency keys but you need to retry safely?
> Build your own idempotency layer on your side (track "have I already successfully called this?" before retrying), or accept the risk if the operation is naturally idempotent already (a "set to X" write vs. an "increment by X" write).
> **Layman:** if the pizza place doesn't have a system to recognize "wait, didn't you already call this order in?", YOU have to keep your own notebook of "did I already place this call" before calling again.
> **Example:** distinguishing naturally-idempotent PUT-style operations from non-idempotent POST-style ones without external help.

> [!question]- 69. Push vs. pull — when do you pick which for delivering updates to a client?
> Push (WebSocket, server-sent events) when updates are frequent/real-time-sensitive. Pull (polling) when updates are infrequent, open connections at scale are expensive, or the client tolerates delay.
> **Layman:** push is the newspaper delivered to your doorstep the moment it's printed. Pull is walking to the store to check if today's paper is out yet.
> **Example:** WhatsApp/Google Meet use push; an hourly-updated analytics dashboard is fine with periodic pull.

> [!question]- 70. How do you handle a client slower than the server producing data (backpressure)?
> Without a "slow down" signal, the producer keeps pushing and the consumer's buffer grows unbounded. A bounded buffer/channel with blocking sends, or a protocol-level backpressure signal, lets the consumer throttle the producer naturally.
> **Layman:** a conveyor belt feeding boxes to a packer — if the packer falls behind, the belt should slow down or stop feeding more boxes, not just keep piling boxes on the floor until they topple over.
> **Example:** [[Glossary/Backpressure|Backpressure]]'s Go-channel example.

> [!question]- 71. How would you design pagination for an API returning a huge, constantly-changing list (a feed)?
> Offset-based pagination (`LIMIT/OFFSET`) breaks when the list changes between page requests — items shift, causing skipped/duplicated results. Cursor-based pagination (fetch strictly after "the last item you saw") is stable under concurrent inserts/deletes.
> **Layman:** "give me the next 20 emails after this specific one" stays correct even as new emails arrive. "Give me items 21-40" breaks the moment new items get inserted at the top, shifting everything's position underneath you.
> **Example:** [[HLD/06 - Design Twitter - News Feed/Design Twitter - News Feed|Twitter's News Feed]] uses exactly this cursor-based approach.

## 8. Security & Auth

> [!question]- 72. How do you authenticate a user's request as it flows through multiple internal microservices?
> Authenticate once at the edge (API gateway), issue a short-lived, verifiable token (JWT) carrying identity/claims, pass it along the call chain — internal services trust the gateway's verification rather than each re-authenticating original credentials.
> **Layman:** showing your ID once at the building's front desk, getting a visitor badge, and every office inside trusts the badge instead of asking to see your ID all over again at every door.
> **Example:** full flow in [[CS Fundamentals/Networking/API Gateway|API Gateway]] and [[CS Fundamentals/Security/Authentication & Authorization|Authentication & Authorization]].

> [!question]- 73. How do you prevent a DDoS attack from taking your system down?
> Layered defense: rate limiting at the edge, a CDN/edge network absorbing and filtering volumetric traffic before it reaches origin, and auto-scaling as a last line of defense for legitimate spikes that resemble an attack.
> **Layman:** a nightclub with a bouncer capping how fast people enter (rate limiting), a big outdoor holding area to absorb a crowd before it reaches the actual door (CDN/edge), and the ability to open more entrances if a real, legitimate crowd shows up (auto-scale).
> **Example:** [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter|Multi-Region Rate Limiter]] and [[CS Fundamentals/Networking/CDN Internals|CDN Internals]] both directly support this.

> [!question]- 74. Why rate-limit per-user *and* per-IP, rather than just one?
> Per-user alone doesn't stop an unauthenticated flood (no identity to key on yet). Per-IP alone punishes legitimate users sharing an IP (corporate/mobile NAT) for one abuser's behavior. Both, at different layers, catches different abuse without over-punishing shared-IP traffic.
> **Layman:** limiting "one troublemaker" isn't enough if you don't know who they are yet, and limiting "one house" isn't fair if a whole family living there gets blamed for one member's bad behavior — you need both angles.
> **Example:** a common, specific follow-up to [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|the base Rate Limiter chapter]].

> [!question]- 75. How would you securely store a user's password?
> Never plaintext or reversible encryption — hash with a slow, purpose-built algorithm (bcrypt/scrypt/Argon2), deliberately expensive to brute-force, with a per-user random salt against rainbow tables.
> **Layman:** never write passwords on a sticky note anyone could read — instead run them through a deliberately slow, one-way blender (hashing) so even if someone steals the blended result, they can't easily un-blend it back into the original.
> **Example:** suggesting plain SHA-256 (fast, not designed for passwords) is a real red flag interviewers listen for.

> [!question]- 76. What's the difference between authentication and authorization, precisely?
> Authentication: "who are you" — verifying identity. Authorization: "what are you allowed to do" — an independent question; a system can perfectly authenticate someone and still correctly deny them a specific resource.
> **Layman:** showing your ID proves who you are (authentication). Whether the bouncer lets you into the VIP section is a totally separate decision (authorization) — being a verified real person doesn't automatically mean you're on the list.
> **Example:** full RBAC vs. ABAC treatment in [[CS Fundamentals/Security/Authentication & Authorization|Authentication & Authorization]].

> [!question]- 77. How do you handle a leaked/compromised access token?
> If tokens are short-lived JWTs, the damage window is naturally bounded; the real lever is revoking the associated *refresh* token immediately (server-side, the one piece of state actually tracked), preventing further access tokens once the current one expires.
> **Layman:** if someone steals your hotel room key card, the hotel can deactivate that specific card immediately at the front desk — you don't need to change the locks on every door in the hotel.
> **Example:** full refresh-token rotation and theft-detection scheme in [[CS Fundamentals/Security/Authentication & Authorization|Authentication & Authorization]].

> [!question]- 78. How would you design rate limiting to stop credential-stuffing / brute-force logins specifically?
> A stricter, security-specific limit than general API rate limiting — per-account *and* per-IP together, escalating delays or CAPTCHA after a few failures. One of the real cases worth paying for strict, synchronous global coordination rather than the relaxed eventual-sync default.
> **Layman:** after a few wrong door-code attempts, the door starts making you wait longer between tries, or asks for extra proof you're really you — stricter than the general "don't let too many people through the lobby" rule.
> **Example:** directly the "when strict global coordination is worth the latency cost" follow-up from [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter|Multi-Region Rate Limiter]].

## 9. Monitoring & Operations

> [!question]- 79. How do you know your system is down before your users tell you?
> Active monitoring/alerting on key health signals (error rate, latency percentiles, availability checks) paging on-call automatically when a threshold is breached — faster detection than user complaints surfacing it.
> **Layman:** a smoke detector going off before the room fills with visible smoke — you want the alarm to catch the problem before a person has to notice and report it.
> **Example:** the premise of [[HLD/20 - Design a Log Aggregation and Monitoring System/Design a Log Aggregation and Monitoring System|the Log Aggregation / Monitoring System chapter]], including the "monitoring the monitor" meta-requirement.

> [!question]- 80. What metrics would you track for any given system?
> Request rate (QPS), error rate, latency percentiles (P50/P90/P99 — not just average), resource utilization, and a domain-specific business metric reflecting actual user-facing success.
> **Layman:** a car dashboard — speed (request rate), warning lights (error rate), how long the trip is actually taking (latency), and fuel/engine temp (resource usage) — you don't just watch one gauge.
> **Example:** [[Glossary/Latency Percentiles (P50, P90, P99)|Latency Percentiles]] — P99 often matters more than average for real user experience.

> [!question]- 81. Why do you care about P99 latency instead of just average latency?
> Average can look healthy while a meaningful fraction of users experience terrible latency — P99 reveals that tail, which matters because at scale, 1% of a large system's traffic is still a large absolute number of bad experiences hidden by a healthy-looking average.
> **Layman:** if 99 people get their food in 2 minutes and 1 person waits 45 minutes, the "average" wait sounds fine — but that one person had a genuinely terrible experience, and at a busy restaurant, "1 in 100" is a lot of angry customers over a full day.
> **Example:** any fan-out system — one slow shard/replica out of many drags the P99 of the whole request down while the average barely moves.

> [!question]- 82. How do you debug a sudden latency spike in production?
> Check what changed first (recent deploy, config, traffic pattern shift), then narrow down which layer is actually slow using distributed tracing or per-hop metrics — a spike is rarely uniform across every component.
> **Layman:** if your commute suddenly takes twice as long, you check what changed today — construction, an accident, weather — rather than randomly guessing, then figure out WHICH stretch of the route is the slow part.
> **Example:** the exact reasoning [[HLD/20 - Design a Log Aggregation and Monitoring System/Design a Log Aggregation and Monitoring System|the Log Aggregation chapter's]] observability design exists to support.

> [!question]- 83. What's your rollback strategy if a new deploy breaks production?
> Fast, automated rollback to the last known-good version, ideally triggered by automated post-deploy health-check failure, paired with canary deployment (small traffic percentage first) so a bad deploy's blast radius is limited.
> **Layman:** painting a house — test the new color on a small hidden wall first (canary), and have a can of the old color ready to repaint immediately if it looks awful, rather than painting the whole house first and hoping.
> **Example:** explicitly separating "how do you detect it's bad" from "how do you recover" — both matter.

> [!question]- 84. What's the difference between SLA, SLO, and SLI?
> SLI: the actual measured metric ("99.95% of requests succeeded last month"). SLO: the internal target ("we aim for 99.9%"). SLA: the external, often contractual commitment, usually set looser than the internal SLO to leave margin.
> **Layman:** SLI is your actual grade on a test (94%). SLO is the grade you're personally aiming for (95%). SLA is the grade you PROMISED your parents you'd get (90%) — set a little lower than your real target, so an off day doesn't break the promise.
> **Example:** full definitions in [[Glossary/SLA vs SLO vs SLI|SLA vs SLO vs SLI]].

> [!question]- 85. How would you design an on-call alerting system to avoid alert fatigue?
> Alert only on symptoms requiring human action (user-facing impact, SLO burn rate), not every internal anomaly — too many low-signal alerts cause real ones to get ignored.
> **Layman:** a smoke detector that also beeps for burnt toast every single day — eventually people just take the battery out, and then it's silent during an actual fire.
> **Example:** related to, but distinct from, the alert-rule-indexing *efficiency* problem in [[HLD/20 - Design a Log Aggregation and Monitoring System/Design a Log Aggregation and Monitoring System|Log Aggregation]] — this question is about alert *quality*.

> [!question]- 86. How do you monitor a system you don't own but depend on heavily (a third-party API)?
> Track your *own* observed success rate/latency calling it (you can't instrument their internals), combined with a circuit breaker so a degrading dependency is handled automatically, not just observed.
> **Layman:** you can't see inside your supplier's factory, but you CAN track how often their deliveries to YOU are late or broken — and have a backup plan ready for when they clearly are.
> **Example:** monitoring and resilience as two sides of the same problem — [[CS Fundamentals/Distributed Systems/Resilience Patterns|Resilience Patterns]].

## 10. Estimation & Numbers

> [!question]- 87. How would you estimate storage requirements for a new system?
> `(number of records) × (average size per record) × (replication factor)`, plus a growth buffer — the skill being tested is showing the work with reasonable stated assumptions, not landing on a "correct" number.
> **Layman:** figuring out how many moving boxes you need by estimating (rooms) × (average stuff per room) × (a few extra boxes just in case) — nobody expects the exact number, just a reasonable, well-reasoned guess.
> **Example:** every HLD chapter's Step 3 does exactly this, with the math shown.

> [!question]- 88. How do you estimate QPS given daily active users?
> `(DAU) × (avg actions/user/day) ÷ 86,400` gives average QPS; apply a peak multiplier (commonly 2-10x depending on spikiness) to get peak QPS — the number that actually determines capacity.
> **Layman:** if a stadium holds 50,000 people and everyone buys popcorn once during a 2-hour game, that's roughly 50,000 purchases spread over 7,200 seconds — then multiply up for the rush right after halftime, which is way busier than the average moment.
> **Example:** this calculation, with average-vs-peak made explicit, appears in nearly every HLD chapter's Step 3.

> [!question]- 89. What are the "latency numbers every engineer should know," roughly?
> Memory access ~100ns, SSD random read ~100μs (1,000x slower than memory), same-datacenter round trip ~0.5ms, cross-region round trip ~100-150ms (bounded by the speed of light, not improvable by engineering). Core lesson: in-memory ops are ~1,000-1,000,000x faster than a network round trip — why caching and minimizing hops matter so much.
> **Layman:** grabbing something from your own pocket (memory) is instant; walking to your car's trunk (disk) takes a bit; mailing a letter overseas and getting a reply (cross-region network) takes real, unavoidable time no matter how efficient the post office is, because it has to physically travel.
> **Example:** why [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter|Multi-Region Rate Limiter]] treats cross-region sync latency as a hard physical constraint, not a tunable parameter.

> [!question]- 90. How would you estimate the number of servers needed for a given QPS?
> `(peak QPS) ÷ (QPS one server can handle)` gives a rough count, plus a redundancy margin (never run at exactly 100% theoretical capacity) — again, the reasoning shown matters more than the exact number.
> **Layman:** figuring out how many checkout lanes a store needs by dividing expected shoppers-per-hour by how many one lane can ring up per hour, then adding a couple of spare lanes just in case.
> **Example:** standard practice used throughout every HLD chapter's estimation step.

> [!question]- 91. If asked to estimate something with numbers you don't know, what do you do?
> State reasonable, clearly-labeled assumptions and show the arithmetic explicitly — interviewers test structured estimation reasoning, not memorized trivia.
> **Layman:** "how many piano tuners are in Chicago" style reasoning — you don't know the real number, but you can reason your way to a sensible ballpark out loud, step by step, and that reasoning is the actual point.
> **Example:** exactly the Fermi-estimation approach modeled in every HLD chapter's Step 3.

> [!question]- 92. Why does back-of-envelope estimation matter if the "real" numbers will differ in production?
> It reveals *which* architectural decisions actually matter before any code is written — e.g., discovering peak QPS is 10x average tells you admission control matters more than raw server count.
> **Layman:** knowing your move will need "roughly 40 boxes, mostly for the kitchen" tells you to rent a bigger truck and pack the kitchen first — the precise final count matters less than knowing where the real weight is.
> **Example:** [[HLD/15 - Design a Ticket Booking System/Design a Ticket Booking System|Ticket Booking's]] estimation step (peak-to-average ratio) drives its entire subsequent design.

> [!question]- 93. How would you estimate bandwidth requirements for a video streaming service?
> `(concurrent viewers) × (average bitrate, depending on the adaptive-bitrate tier mix)` gives instantaneous bandwidth — dominated by raw video bytes over the wire, which is why CDN edge caching is the single highest-leverage optimization here.
> **Layman:** the number of people simultaneously watching a live TV broadcast times how much "picture quality" each one is streaming — which is why local relay towers (CDN) matter so much more than upgrading the original broadcast studio.
> **Example:** [[HLD/09 - Design YouTube - Netflix/Design YouTube - Netflix|the YouTube/Netflix chapter's]] estimation step.

## 11. Trade-off & Judgment Meta-Questions

> [!question]- 94. "Why did you choose X over Y here?"
> The interviewer is testing whether you understand this as a genuine tradeoff, not a fact to recite — name the *specific* dimension you optimized for (latency, consistency, cost, simplicity) and what you deliberately gave up.
> **Layman:** like explaining why you chose the highway over the scenic route — not because one is objectively "better," but because you were optimizing for time over view, and you should be able to say that plainly.
> **Example:** every "three real options, not one correct answer" table in this book's HLD chapters exists to train exactly this kind of answer.

> [!question]- 95. "What would you do differently with more time or a bigger budget?"
> A specific answer beats a vague one — name the *one* simplification you made under real constraints, and why.
> **Layman:** "I'd have used a nicer, sturdier material here, but given the deadline I used the cheaper one that still holds — here's exactly what I'd upgrade first."
> **Example:** [[LLD/19 - Design a Stock Exchange/Design a Stock Exchange|the Stock Exchange chapter]] explicitly names its own simplification (sorted slices vs. a production heap-based book) as exactly this kind of honest, stated scope limit.

> [!question]- 96. "How would this design change at 10x the scale? 100x?"
> Identify which component breaks *first* at each multiplier, rather than assuming uniform scaling — a single database fine at 10x may need sharding at 100x; a synchronous call chain fine at 10x may need to go async at 100x.
> **Layman:** a corner store's setup works fine as a small chain, but running it like a national chain needs a completely different supply system — figure out which part breaks first as the crowd grows, not just "add more of everything."
> **Example:** the reasoning connecting [[HLD/02 - Design a Rate Limiter/Design a Rate Limiter|the base Rate Limiter]] to its [[HLD/14 - Design a Multi-Region Rate Limiter/Design a Multi-Region Rate Limiter|Multi-Region follow-up]] — a new deployment shape introduces genuinely new problems, not just "more of the same."

> [!question]- 97. "What's the biggest risk in this design?"
> Name a real, specific failure mode you've already reasoned about — a single point of failure not fully mitigated, a consistency window during one specific operation — rather than a generic "scalability" non-answer.
> **Layman:** like a contractor honestly telling you "the roof will hold in normal weather, but I'm less confident about it in a severe storm" — specific, not a vague "nothing's perfect."
> **Example:** point to the exact moment in your design where you made a deliberate tradeoff and name the residual risk honestly.

> [!question]- 98. "How would you test this system?"
> Beyond unit/integration tests: load testing (does it hold at estimated peak QPS), chaos testing (does it survive a random server/AZ failure gracefully), and correctness testing for concurrency-sensitive logic specifically (needs concurrent scenarios, not sequential ones).
> **Layman:** test-driving a new car for (1) does it drive normally, (2) does it still work if a tire blows out, (3) is it actually fast enough for the highway — three different questions, three different tests.
> **Example:** worth separating "does it work," "does it survive failure," and "is it fast enough" as three distinct concerns.

> [!question]- 99. "If you had to cut scope to ship in half the time, what would you cut first, and why?"
> Cut whatever's furthest from the core user-facing requirement, and say so explicitly — e.g., real-time seat-map updates can ship later than the atomic booking guarantee itself, since the guarantee prevents actual harm.
> **Layman:** if you have to finish a house renovation in half the time, you finish the plumbing and electrical (safety-critical) before you worry about the paint color (nice-to-have).
> **Example:** distinguishing "correctness-critical" from "nice-to-have" scope is exactly the judgment being tested.

> [!question]- 100. "Walk me through what happens end-to-end for a single request in your design."
> The single most common closing question — trace one concrete request hop by hop (client → LB → gateway → service → cache/DB → response), naming every component touched and what could go wrong at each hop.
> **Layman:** literally following one customer's order from the moment they walk in the door to the moment they leave with their food, checking every single stop along the way — often you spot a broken step you'd never notice just describing the kitchen in the abstract.
> **Example:** deliberately the last question here because it's often the actual final question in a real interview — a design that "sounds right" component-by-component often reveals a gap only when traced end-to-end for one real request.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] — every question above links back to the full chapter it's drawn from. Read the chapter for the complete derivation; use this file for rapid-fire drilling.*
