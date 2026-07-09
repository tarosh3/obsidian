---
title: "1929. Concatenation of Array"
aliases: [Concatenation of Array, LeetCode 1929]
tags: [dsa, leetcode, arrays, index-mapping, easy]
difficulty: Easy
leetcode: 1929
---

# 1929. Concatenation of Array
`Easy` · **Pattern:** Index mapping via modulo (not a two-pointer problem — noted below)

> [!question] Problem
> Given an integer array `nums` of length `n`, create an array `ans` of length `2n` where `ans[i] == nums[i]` and `ans[i + n] == nums[i]` for `0 <= i < n` (0-indexed). Specifically, `ans` is the concatenation of two `nums` arrays. Return the array `ans`.
>
> **Example 1:**
> ```
> Input: nums = [1,2,1]
> Output: [1,2,1,1,2,1]
> Explanation: The array ans is formed as follows:
> ans = [nums[0],nums[1],nums[2],nums[0],nums[1],nums[2]]
>     = [1,2,1,1,2,1]
> ```
>
> **Example 2:**
> ```
> Input: nums = [1,3,2,1]
> Output: [1,3,2,1,1,3,2,1]
> ```
>
> **Constraints:**
> - `n == nums.length`
> - `1 <= n <= 1000`
> - `1 <= nums[i] <= 1000`

---

## 🧩 Pattern this follows

> [!warning] Not actually a two-pointer problem
> This one's a warm-up/filler problem — worth solving fast and moving on, but the real pattern here is **index mapping with modulo (`i % n`)**, not converging pointers. Flagging that honestly rather than forcing a two-pointer narrative onto it.

## 💻 My Solution (C++)

```cpp
class Solution {
public:
    vector<int> getConcatenation(vector<int>& nums) {
        int n = nums.size();
        vector<int> ans(2 * n);

        for (int i = 0; i < 2 * n; i++) {
            ans[i] = nums[i % n];
        }
        return ans;
    }
};
```

## 🔍 Walkthrough

1. `n` is the original length; `ans` is pre-sized to `2n` so there's no repeated resizing/`push_back` overhead.
2. Loop `i` across the *full* `2n` range of the output. `nums[i % n]` is the trick: for `i` in `[0, n)`, `i % n == i`, so it just copies `nums` directly. For `i` in `[n, 2n)`, `i % n` wraps back around to `[0, n)`, replaying `nums` a second time.
3. One pass, one array, no separate "copy the second half" step needed.

## ⏱️ Complexity

| | Complexity | Why |
|---|---|---|
| **Time** | O(n) | Exactly `2n` writes, one pass |
| **Space** | O(n) | The output array itself (not counted as "extra" since it's the required return value) |

## 🚀 Tricks & Similar Problems

> [!success] `i % n` as a "wrap around and repeat" primitive
> Whenever a problem says "repeat this array/string," or you need **circular indexing** (like a circular buffer, or rotating through a fixed-size list), `index % length` is the general tool — it turns an ever-growing index into one that loops back into valid bounds automatically.
