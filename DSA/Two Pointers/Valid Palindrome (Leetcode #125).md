---
title: "125. Valid Palindrome"
aliases: [Valid Palindrome, LeetCode 125]
tags: [dsa, leetcode, two-pointers, easy]
difficulty: Easy
leetcode: 125
---

# 125. Valid Palindrome
`Easy` · **Pattern:** Two Pointers (converging from both ends)

> [!question] Problem
> A phrase is a palindrome if, after converting all uppercase letters into lowercase letters and removing all non-alphanumeric characters, it reads the same forward and backward.
> Given a string `s`, return `true` if it is a palindrome, or `false` otherwise.
>
> **Example 1:**
> ```
> Input: s = "A man, a plan, a canal: Panama"
> Output: true
> Explanation: "amanaplanacanalpanama" is a palindrome.
> ```
>
> **Example 2:**
> ```
> Input: s = "race a car"
> Output: false
> Explanation: "raceacar" is not a palindrome.
> ```
>
> **Example 3:**
> ```
> Input: s = " "
> Output: true
> Explanation: s is an empty string "" after removing non-alphanumeric characters. An empty string reads the same forward and backward, so it is a palindrome.
> ```
>
> **Constraints:**
> - `1 <= s.length <= 2 * 10^5`
> - `s` consists only of printable ASCII characters.

---

## 🧩 Pattern this follows

> [!tip] Two pointers closing in from both ends
> A palindrome check is naturally symmetric — compare the outermost characters, then move inward. Instead of building a cleaned-up copy of the string first (extra `O(n)` space) and then checking it, walk **two pointers inward from both ends simultaneously**, skipping non-alphanumeric characters on the fly. This is the archetypal "converging two pointers" shape — same skeleton reused across nearly every problem in this chapter.

## 💻 My Solution (C++)

```cpp
class Solution {
public:
    bool isPalindrome(string s) {
        int left = 0;
        int right = s.size() - 1;

        while (left < right) {
            while (left < s.size() && !isalnum(s[left])) {
                left++;
            }
            while (right >= 0 && !isalnum(s[right])) {
                right--;
            }
            if (right < 0 || left > s.size() - 1) {
                return true;
            }

            char a = tolower(s[left]);
            char b = tolower(s[right]);
            if (a != b) {
                return false;
            }

            left++;
            right--;
        }

        return true;
    }
};
```

## 🔍 Walkthrough

1. `left` starts at index `0`, `right` starts at the last index.
2. Inside the loop, first **skip forward** past any non-alphanumeric character from the left (`!isalnum(s[left])`), and **skip backward** past any non-alphanumeric character from the right — this is what lets the pointers ignore spaces, commas, colons, etc. without ever building a cleaned string.
3. After skipping, guard against the pointers having crossed or run off the string entirely (`right < 0 || left > s.size() - 1`) — this covers strings that are *entirely* non-alphanumeric (like `" "`), where skipping alone would otherwise leave the pointers in an invalid state before the comparison below runs.
4. Compare the lowercased characters at `left` and `right`. Any mismatch → not a palindrome, return `false` immediately.
5. On a match, move both pointers inward (`left++`, `right--`) and repeat. Loop ends when `left >= right`, meaning every pair matched → `true`.

## ⏱️ Complexity

| | Complexity | Why |
|---|---|---|
| **Time** | O(n) | Each pointer moves inward at most `n/2` times total; every character is visited once |
| **Space** | O(1) | No extra string built — comparisons happen directly on the original `s` |

## 🚀 Tricks & Similar Problems

> [!success] Skip-then-guard-then-compare is the reusable shape
> The three-step body — **(1)** skip invalid characters from both sides, **(2)** guard against pointers crossing, **(3)** compare and advance — is the general template for any two-pointer string/array problem with a filtering condition layered on top of a symmetry check.
> **Similar pattern:** Palindrome Linked List (same converging-pointer idea after reversing half), Valid Palindrome II (this exact solution + "allow skipping one mismatch").
