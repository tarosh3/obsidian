---
title: "Raft (Consensus)"
aliases: [Raft, Raft Consensus, Consensus Algorithm]
tags: [system-design, glossary]
---

# Raft (Consensus)

> [!info] Quick definition
> A protocol letting a cluster of nodes agree on a single value (typically "who's the leader" and "what's in the replicated log") despite node failures — nodes elect a leader via randomized election timeouts, the leader replicates log entries to followers, and an entry is committed once a **majority** acknowledges it.

Powers real systems like etcd, Consul, and CockroachDB's leader election. The majority requirement is exactly what prevents [[Split-Brain]] — a minority partition can never elect its own leader, because it can't reach a majority.

> [!tip] Full chapter
> Leader election mechanics, log replication, the safety proof behind the election restriction, and Raft vs. Paxos: [[CS Fundamentals/06 - Distributed Systems/Consensus (Raft & Paxos)|Consensus (Raft & Paxos)]].

**Used in:** [[HLD/18 - Design a Distributed Lock Service/Design a Distributed Lock Service|Design a Distributed Lock Service]]
