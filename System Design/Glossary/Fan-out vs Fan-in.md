---
title: "Fan-out vs Fan-in"
aliases: [Fan-out, Fan-in]
tags: [system-design, glossary]
---

# Fan-out vs Fan-in

> [!info] Definition
> **Fan-out**: one input triggers many downstream outputs (one event → many recipients/workers). **Fan-in**: many inputs converge into one output (many producers → one consumer/aggregator).

**Fan-out example:** a celebrity posts once, and that single write needs to be delivered ("fanned out") into millions of followers' feeds — the "celebrity problem" in [[HLD/06 - Design Twitter - News Feed/Design Twitter - News Feed|the Twitter News Feed chapter]]. **Fan-in example:** metrics from thousands of servers all flowing into one aggregation/monitoring pipeline.

Both directions carry a scaling risk: fan-out can overwhelm downstream consumers (mitigated with async processing/[[CS Fundamentals/05 - Messaging & Streaming/Kafka Internals|queues]]), fan-in can overwhelm the single aggregator (mitigated with sharded aggregation or [[Backpressure]]).

**Used in:** [[HLD/06 - Design Twitter - News Feed/Design Twitter - News Feed|Design Twitter - News Feed]] · [[HLD/20 - Design a Log Aggregation and Monitoring System/Design a Log Aggregation and Monitoring System|Design a Log Aggregation and Monitoring System]] · [[CS Fundamentals/07 - Architecture and Deployment Patterns/Event-Driven Architecture|Event-Driven Architecture]]
