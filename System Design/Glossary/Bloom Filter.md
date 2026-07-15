---
title: "Bloom Filter"
aliases: [Bloom Filter]
tags: [system-design, glossary]
---

# Bloom Filter

> [!info] Quick definition
> A probabilistic data structure answering "have I possibly seen this before?" — **no false negatives** (if it says "no," the item is definitely not present) but **occasional false positives** (if it says "yes," it's *probably* present, but might not be). Extremely space-efficient vs storing the actual full set.

Works by hashing an item through several hash functions and setting bits in a bit array; checking membership hashes the same way and checks if all those bits are set. Used to avoid expensive lookups for known-absent keys (e.g. "check the Bloom filter before querying the database — if it says no, skip the query entirely").

> [!tip] This is a glossary stub
> Full explanation lives in: [[CS Fundamentals/06 - Distributed Systems/Bloom Filter and Probabilistic Membership|Bloom Filter & Probabilistic Membership]].

**Used in:** [[CS Fundamentals/06 - Distributed Systems/Bloom Filter and Probabilistic Membership|Bloom Filter & Probabilistic Membership]] · [[HLD/12 - Design a Web Crawler/Design a Web Crawler|Design a Web Crawler]] (URL dedup) · [[CS Fundamentals/03 - Databases/Cassandra Internals|Cassandra Internals]] (SSTable read path)
