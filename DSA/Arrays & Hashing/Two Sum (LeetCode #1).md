---
title: "1. Two Sum"
aliases: [Two Sum, LeetCode 1]
tags: [dsa, leetcode, arrays-hashing, hash-map, easy]
difficulty: Easy
leetcode: 1
---

# 1. Two Sum
`Easy` · **Pattern:** Hash Map (complement lookup, one pass)

> [!question] Problem
> Given an array of integers `nums` and an integer `target`, return indices of the two numbers such that they add up to `target`.
> You may assume that each input has **exactly one solution**, and you may not use the *same* element twice. You can return the answer in any order.
>
> **Example 1:**
> ```
> Input: nums = [2,7,11,15], target = 9
> Output: [0,1]
> Explanation: nums[0] + nums[1] == 9, so return [0, 1].
> ```
>
> **Example 2:**
> ```
> Input: nums = [3,2,4], target = 6
> Output: [1,2]
> ```
>
> **Example 3:**
> ```
> Input: nums = [3,3], target = 6
> Output: [0,1]
> ```
>
> **Constraints:**
> - `2 <= nums.length <= 10^4`
> - `-10^9 <= nums[i] <= 10^9`
> - `-10^9 <= target <= 10^9`
> - Only one valid answer exists.

---

## 🧩 Pattern this follows

> [!tip] "Have I seen the complement?" — the canonical hash-map pattern
> The brute force is checking every pair (O(n²)). The insight: for each `nums[i]`, you don't need to search the *whole array* for `target - nums[i]` — you just need to know **have I already seen it?** A hash map turns that question into O(1). This "seen it before?" pattern is the single most common opener across the entire Arrays & Hashing chapter.

### 🖼️ Visualizing it

Trace of `nums=[2,7,11,15], target=9` — the map grows one entry at a time until a complement is found.

```mermaid
sequenceDiagram
    participant Loop as i (scanning nums)
    participant Map as mp (value→index)
    Loop->>Map: i=0, nums[0]=2 — look up 9-2=7? not found
    Loop->>Map: insert 2→0
    Loop->>Map: i=1, nums[1]=7 — look up 9-7=2? found (index 0)!
    Map-->>Loop: return [0, 1]
```

## 💻 My Solution (C++)

```cpp
class Solution {
public:
    vector<int> twoSum(vector<int>& nums, int target) {
        unordered_map<int, int> mp;

        for (int i = 0; i < nums.size(); i++) {
            if (mp.find(target - nums[i]) != mp.end()) {
                return {mp[target - nums[i]], i};
            }
            mp[nums[i]] = i;
        }

        return {};
    }
};
```

## 🔍 Walkthrough

1. `mp` maps **value → index** of every number seen *so far*.
2. For each `i`, first check: is `target - nums[i]` (the complement needed to hit `target`) already in `mp`? If yes, that earlier index plus the current `i` is the answer — return immediately.
3. If not found, **insert** `nums[i]` into `mp` (`mp[nums[i]] = i`) so future iterations can find it as a complement.
4. Checking *before* inserting is what guarantees the same element is never used twice — `nums[i]` isn't in the map yet when you check for its own complement (unless a duplicate value at an earlier index already put it there, which is exactly the case Example 3 tests).

## ⏱️ Complexity

| | Complexity | Why |
|---|---|---|
| **Time** | O(n) | Single pass; each hash map lookup/insert is O(1) average |
| **Space** | O(n) | Map can hold up to n-1 entries before the answer is found |

> [!bug] Watch out
> Check-then-insert order matters. If you insert `nums[i]` into `mp` **before** checking for the complement, `nums[i]` could match itself when `target == 2 * nums[i]` (e.g. `nums=[3,3], target=6` would incorrectly pair index `0` with itself). This solution avoids that by checking first — worth explaining out loud in an interview, it's a common "found the bug" moment.

## 🚀 Tricks & Similar Problems

> [!success] The "complement in a map" pattern generalizes
> Anytime a problem says "find a pair/subarray that sums/relates to X," ask: *"can I reframe this as, for each element, do I already have what I need in a map?"* Variants: **Two Sum II** (sorted array → two pointers instead), **3Sum** (fix one element, Two-Sum the rest), **Subarray Sum Equals K** (prefix sums in a map).
> **Similar pattern:** [[Contains Duplicate (LeetCode #217)]] (same "have I seen this?" hash lookup, simpler check).
