---
title: "Gossip Protocol"
aliases: [Gossip Protocol, Epidemic Protocol]
tags: [system-design, glossary]
---

# Gossip Protocol

> [!info] Definition
> A decentralized way for nodes in a cluster to spread information: each node periodically shares its known state with a few randomly chosen peers, who then do the same — information spreads "epidemic-style" across the whole cluster without any central coordinator.

Used by Cassandra and DynamoDB-style databases to propagate cluster membership and node-health info. Tradeoff: no single point of failure and scales well with cluster size, but propagation is **eventually consistent** — it takes a few rounds for information to reach every node, not instant.

> [!example] Layman's terms
> Like a rumor spreading through an office — no one announces it to everyone at once, but each person who hears it tells a couple of others, and within a short time everyone knows, without anyone being "in charge" of the spreading.

**Used in:** [[HLD/05 - Consistency, Availability & Consensus/Index|HLD 05 - Consistency, Availability & Consensus]]
