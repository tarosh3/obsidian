---
title: "Saga Pattern"
aliases: [Saga, Saga Pattern]
tags: [system-design, glossary]
---

# Saga Pattern

> [!info] Quick definition
> A way to handle a distributed transaction as a **sequence of local transactions**, each with a **compensating "undo" action** — if a later step fails, previously completed steps are undone by running their compensations, instead of one big atomic commit/rollback like [[Two-Phase Commit (2PC)|2PC]].

E.g. "book flight → book hotel → charge card" — if charging the card fails, the Saga runs "cancel hotel" and "cancel flight" as compensating actions. Eventually consistent (there's a window where some steps are done and others aren't) but scales far better than 2PC and doesn't hold cross-service locks.

> [!tip] This is a glossary stub
> Full explanation lives in: [[CS Fundamentals/06 - Distributed Systems/Distributed Transactions - Saga Pattern and Two-Phase Commit|Distributed Transactions: Saga Pattern & Two-Phase Commit]].

**Used in:** [[CS Fundamentals/06 - Distributed Systems/Distributed Transactions - Saga Pattern and Two-Phase Commit|Distributed Transactions: Saga Pattern & Two-Phase Commit]] · [[CS Fundamentals/07 - Architecture and Deployment Patterns/Event-Driven Architecture|Event-Driven Architecture]] · [[HLD/23 - Design an E-commerce System/Design an E-commerce System|Design an E-commerce System]]
