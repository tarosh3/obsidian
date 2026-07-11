---
title: "Heartbeat (Health Check)"
aliases: [Heartbeat, Health Check]
tags: [system-design, glossary]
---

# Heartbeat (Health Check)

> [!info] Definition
> A periodic "I'm alive" signal a node sends (or that gets polled from it) to prove it's still functioning — absence of a heartbeat within an expected window triggers failure detection.

Load balancers use health checks to stop routing traffic to a dead server. Cluster/consensus systems (like [[Consistent Hashing|sharded]] or [[Raft (Consensus)|Raft-based]] systems) use missed heartbeats to trigger leader re-election or failover. The tuning tradeoff: check too often and you waste resources/bandwidth; check too rarely and failure detection is slow, leaving traffic routed to a dead node longer.

**Used in:** [[HLD/05 - Consistency, Availability & Consensus/Index|HLD 05 - Consistency, Availability & Consensus]], [[HLD/07 - Scalability & Resilience Patterns/Index|HLD 07 - Scalability & Resilience Patterns]]
