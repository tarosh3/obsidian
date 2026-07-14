---
title: "ACID"
aliases: [ACID]
tags: [system-design, glossary]
---

# ACID

> [!info] Quick definition
> Traditional relational-database transaction guarantees: **A**tomicity (a transaction fully happens or not at all), **C**onsistency (a transaction moves the DB from one valid state to another, respecting constraints), **I**solation (concurrent transactions don't see each other's partial work), **D**urability (once committed, it survives a crash).

The strong-consistency counterpart to [[BASE]]'s eventual-consistency tradeoff — SQL databases are typically built around ACID; many NoSQL databases relax some or all of these in exchange for scale/availability.

> [!tip] Full chapter
> All four guarantees explained precisely, plus the part that actually matters in interviews — isolation levels, and the exact difference between a dirty read, a non-repeatable read, and a phantom read: [[CS Fundamentals/03 - Databases/ACID & Isolation Levels|ACID & Isolation Levels]].
