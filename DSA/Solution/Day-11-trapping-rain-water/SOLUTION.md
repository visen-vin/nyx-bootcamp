# Day 11 — Trapping Rain Water

## Problem

Given `n` non-negative integers representing an elevation map where the width of each bar is 1, compute how much water it can trap after raining.

```
Input:  height = [0,1,0,2,1,0,1,3,2,1,2,1]
Output: 6
```

![rainwater](https://assets.leetcode.com/static_assets/public/images/rainwatertrap.png)

---

## Approach 1 — Brute Force (O(n²) time, O(1) space)

**Idea:** For every bar, find the tallest bar to its left and the tallest bar to its right. The water this bar can hold is `min(maxLeft, maxRight) - height[i]`. Sum over all bars.

```python
def trap_bruteforce(height: list[int]) -> int:
    n = len(height)
    total = 0
    for i in range(n):
        max_left = max(height[:i+1])   # tallest including current
        max_right = max(height[i:])    # tallest including current
        total += min(max_left, max_right) - height[i]
    return total
```

| Complexity | Value |
|---|--------|
| Time  | O(n²) — for each of n bars we scan left and right O(n) |
| Space | O(1)  — no extra arrays |

**Why it works (but slowly):** The water level at a bar is determined by the shorter of the two tallest walls on either side. This is the definitional brute force, good for verifying correctness.

---

## Approach 2 — Pre-computed Prefix/Suffix Max Arrays (O(n) time, O(n) space)

**Idea:** Instead of recomputing `max_left` and `max_right` for every bar, pre-compute two auxiliary arrays:
- `prefix_max[i]` = tallest bar from index 0 to i
- `suffix_max[i]` = tallest bar from index i to n-1

Then water at `i` = `min(prefix_max[i], suffix_max[i]) - height[i]`.

```python
def trap_prefix_suffix(height: list[int]) -> int:
    n = len(height)
    if n == 0:
        return 0

    prefix_max = [0] * n
    suffix_max = [0] * n

    prefix_max[0] = height[0]
    for i in range(1, n):
        prefix_max[i] = max(prefix_max[i-1], height[i])

    suffix_max[n-1] = height[n-1]
    for i in range(n-2, -1, -1):
        suffix_max[i] = max(suffix_max[i+1], height[i])

    total = 0
    for i in range(n):
        total += min(prefix_max[i], suffix_max[i]) - height[i]

    return total
```

| Complexity | Value |
|---|--------|
| Time  | O(n) — three linear passes |
| Space | O(n) — two auxiliary arrays of length n |

**Tradeoff:** Much faster than brute force but uses extra memory.

---

## Approach 3 — Two-Pointer (Optimal, **O(n) time, O(1) space**)

**Idea:** Instead of storing prefix/suffix arrays, maintain two pointers (`left`, `right`) and two running maximums (`left_max`, `right_max`). At each step:

- If `height[left] < height[right]`, the water at the left pointer is **bounded by `left_max`** (because there's at least a wall of height `height[right]` on the right). Compute water at `left`, move `left` forward.
- Else, symmetric for the right pointer.

This works because the smaller of the two sides determines the water level, and we always process the **more constrained** side.

```python
def trap(height: list[int]) -> int:
    n = len(height)
    if n <= 2:
        return 0

    left, right = 0, n - 1
    left_max = height[left]
    right_max = height[right]
    total = 0

    while left < right:
        if height[left] < height[right]:
            # left side is the limiting side
            left_max = max(left_max, height[left])
            total += left_max - height[left]
            left += 1
        else:
            # right side is the limiting side
            right_max = max(right_max, height[right])
            total += right_max - height[right]
            right -= 1

    return total
```

| Complexity | Value |
|---|--------|
| Time  | O(n) — single pass, each element visited once |
| Space | O(1) — just a few integer variables |

**Why this is optimal:** Every element must be visited at least once to determine how much water it holds — that gives a lower bound of Ω(n). The two-pointer approach achieves exactly O(n) with no extra memory.

---

## Summary

| Approach | Time  | Space | Notes |
|---|---|---|---|
| Brute Force       | O(n²) | O(1) | Simple, good for verification |
| Prefix/Suffix Max | O(n)  | O(n) | Fast, easy to understand |
| Two-Pointer       | O(n)  | O(1) | **Optimal** — efficient and elegant |

**Takeaway:** The two-pointer solution is the one to use in an interview. Start with the brute force for clarity, then refactor to prefix/suffix arrays, and finally optimize to the two-pointer approach.
