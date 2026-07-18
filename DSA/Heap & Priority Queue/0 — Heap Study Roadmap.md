---
title: "Heap / Priority Queue — Study Roadmap"
aliases: [Heap Roadmap, Priority Queue Roadmap, Heap MOC]
tags: [dsa, heap, priority-queue, roadmap, moc, index]
difficulty: Reference
---

# 🔺 Heap / Priority Queue — Study Roadmap
`Reference` · Recommended order to learn/revise, grouped by **difficulty + similarity**.

> [!abstract] How to use this
> Obsidian sorts files alphabetically, so the folder listing isn't in learning order. **Follow the sequence below.** Each row links to the full note; your 6 solutions are copied **verbatim**.

---

## 1️⃣ Foundations — heap as a simulator `Easy`

Get comfortable with `priority_queue`: default **max**-heap, and `greater<int>` for a **min**-heap.

| # | Note | Core idea |
|---|---|---|
| 1046 | [[Last Stone Weight (LeetCode #1046)]] | max-heap: repeatedly smash the two heaviest |
| 703 | [[Kth Largest Element in a Stream (LeetCode #703)]] | size-`k` **min**-heap; root = `k`th largest |

## 2️⃣ Top-K pattern — a fixed-size heap of size `k` `Medium`

The single most-tested heap idea. **Opposite** heap type to what intuition says.

| # | Note | Core idea |
|---|---|---|
| 215 | [[Kth Largest Element in an Array (LeetCode #215)]] | size-`k` **min**-heap for the `k` largest |
| 973 | [[K Closest Points to Origin (LeetCode #973)]] | size-`k` **max**-heap for the `k` smallest distances |

## 3️⃣ Advanced — greedy & the two-heap trick `Medium → Hard`

| # | Note | Core idea |
|---|---|---|
| 621 | [[Task Scheduler (LeetCode #621)]] | greedy: fill cooldown around the hottest task (math formula) |
| 295 | [[Find Median from Data Stream (LeetCode #295)]] | two heaps: max-heap (low half) + min-heap (high half) |

---

## 🔑 Pattern cheat-sheet

> [!tip] The rules that unlock every heap problem
> - **`priority_queue<int>` = MAX-heap** (default). **`priority_queue<int, vector<int>, greater<int>>` = MIN-heap.** Memorize both spellings.
> - **Top-K, the counter-intuitive rule:**
>   - `k` **largest** → keep a **min-heap of size `k`**; the root is the `k`th largest. → [[Kth Largest Element in an Array (LeetCode #215)]], [[Kth Largest Element in a Stream (LeetCode #703)]]
>   - `k` **smallest / closest** → keep a **max-heap of size `k`**; the root is the worst kept. → [[K Closest Points to Origin (LeetCode #973)]]
>   - Capping at `k` makes each op `O(log k)` → overall **`O(n log k)`**, beating an `O(n log n)` sort.
> - **"Always take the current extreme, process, reinsert"** → plain heap simulation. → [[Last Stone Weight (LeetCode #1046)]]
> - **"Running median / balance two halves"** → **two heaps** (max-heap below + min-heap above), roots straddle the median. → [[Find Median from Data Stream (LeetCode #295)]]
> - **Comparisons:** compare **squared** distances (skip `sqrt`); watch **overflow** (`long long` for `x²+y²`); store an **index** in the heap to keep the payload small.
> - **Alternative to a size-`k` heap:** **Quickselect** finds the `k`th element in **O(n) average** — mention it, but the heap is safer to code.

> [!note] On #621 Task Scheduler
> Your solution solves it with a **greedy math formula** (`O(1)` space), not an actual heap — it lives in this folder because it's the classic heap *topic* and the max-heap simulation is the alternate approach. Both reach the same answer.
