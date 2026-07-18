---
title: "Trees — Study Roadmap"
aliases: [Trees Roadmap, Trees Study Order, Trees MOC]
tags: [dsa, tree, binary-tree, bst, roadmap, moc, index]
difficulty: Reference
---

# 🌳 Trees — Study Roadmap
`Reference` · Recommended order to learn/revise, grouped by **difficulty + similarity** so each note builds on the last.

> [!abstract] How to use this
> Obsidian sorts files alphabetically, so the folder listing isn't in learning order. **Follow the sequence below instead.** Notes marked ⭐ are extra foundations I added to fill gaps in the pasted set — do them at the point shown, not skipped. Each row links to the full note.

---

## 1️⃣ Foundations — recursion & the height template `Easy`

The "recurse both children, combine" skeleton. Everything else is a variation of these.

| # | Note | Core idea |
|---|---|---|
| 104 | [[Maximum Depth of Binary Tree (LeetCode #104)]] | `1 + max(left, right)` — the base template |
| 226 | [[Invert Binary Tree (LeetCode #226)]] | swap children at every node |
| 100 ⭐ | [[Same Tree (LeetCode #100)]] | parallel DFS over two trees |
| 572 | [[Subtree of Another Tree (LeetCode #572)]] | "Same Tree" run at every node |
| 543 | [[Diameter of Binary Tree (LeetCode #543)]] | height DFS + global max (bent path) |
| 110 | [[Balanced Binary Tree (LeetCode #110)]] | height DFS + `-1` sentinel |

## 2️⃣ BFS / level-order family `Medium → Hard`

The `qSize`-snapshot queue loop and its uses.

| # | Note | Core idea |
|---|---|---|
| 102 ⭐ | [[Binary Tree Level Order Traversal (LeetCode #102)]] | the BFS template — freeze `qSize` per level |
| 199 | [[Binary Tree Right Side View (LeetCode #199)]] | keep the last node of each level |
| 297 | [[Serialize and Deserialize Binary Tree (LeetCode #297)]] | BFS to string with `null` markers, and back |

## 3️⃣ "State flows down the path" `Medium`

Carry accumulated info **downward** as a parameter.

| # | Note | Core idea |
|---|---|---|
| 1448 | [[Count Good Nodes in Binary Tree (LeetCode #1448)]] | push max-so-far down; count `val >= max` |

## 4️⃣ BST family — exploit the ordering `Medium`

Left < node < right, and in-order = sorted.

| # | Note | Core idea |
|---|---|---|
| 98 | [[Validate Binary Search Tree (LeetCode #98)]] | carry `(min, max)` bounds down |
| 230 | [[Kth Smallest Element in a BST (LeetCode #230)]] | in-order traversal + counter |
| 701 | [[Insert into a Binary Search Tree (LeetCode #701)]] | descend by `< / >`, plant at the null |
| 235 | [[Lowest Common Ancestor of a Binary Search Tree (LeetCode #235)]] | the split point is the LCA |
| 236 ⭐ | [[Lowest Common Ancestor of a Binary Tree (LeetCode #236)]] | general-tree LCA (no ordering → search both) |

## 5️⃣ Advanced — DP-on-tree & construction `Medium → Hard`

Return one quantity, record another; rebuild from traversals.

| # | Note | Core idea |
|---|---|---|
| 124 | [[Binary Tree Maximum Path Sum (LeetCode #124)]] | return one-sided gain, record bent path |
| 105 | [[Construct Binary Tree from Preorder and Inorder Traversal (LeetCode #105)]] | preorder = roots, inorder = split |

---

## 🔑 Pattern cheat-sheet

> [!tip] Recognize the shape, pick the tool
> - **"Combine children on the way up"** (height, diameter, balance, max path sum) → **post-order** DFS, often with a **global** accumulator: [[Maximum Depth of Binary Tree (LeetCode #104)]] → [[Diameter of Binary Tree (LeetCode #543)]] → [[Binary Tree Maximum Path Sum (LeetCode #124)]].
> - **"Carry state down the path"** (good nodes, BST bounds) → **pre-order** DFS with a **parameter**: [[Count Good Nodes in Binary Tree (LeetCode #1448)]], [[Validate Binary Search Tree (LeetCode #98)]].
> - **"By level / shape"** (level order, right view, serialize) → **BFS** with the `qSize` snapshot: [[Binary Tree Level Order Traversal (LeetCode #102)]].
> - **"BST"** → use `left < node < right`; **in-order = sorted**: [[Kth Smallest Element in a BST (LeetCode #230)]], [[Lowest Common Ancestor of a Binary Search Tree (LeetCode #235)]].
> - **"Rebuild from traversals"** → preorder/postorder give roots, inorder splits sides: [[Construct Binary Tree from Preorder and Inorder Traversal (LeetCode #105)]].

> [!note] ⭐ = added by me to fill gaps
> [[Same Tree (LeetCode #100)]], [[Binary Tree Level Order Traversal (LeetCode #102)]], and [[Lowest Common Ancestor of a Binary Tree (LeetCode #236)]] weren't in your pasted set but are prerequisites the other notes lean on. Your 14 solutions are copied **verbatim**; these 3 use clean reference implementations.
