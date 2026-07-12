# Solution: Chocolate Distribution Problem

> **Problem:** Given an array `arr` of `n` integers where each value represents the number of chocolates in a packet, and an integer `m` (number of students), distribute packets such that:
> - Each student gets **exactly one** packet.
> - The **difference** between the packet with the most chocolates and the packet with the least chocolates given to the students is **minimized**.
>
> Return that minimum difference.

---

## Approach 1: Brute Force

**Idea:** Generate all combinations of `m` packets from `n` packets, compute the max-min difference for each, and track the minimum.

### Code

```python
from itertools import combinations

def find_min_diff_brute(arr, m):
    n = len(arr)
    if m > n:
        return -1  # Not enough packets

    min_diff = float("inf")

    for combo in combinations(arr, m):
        diff = max(combo) - min(combo)
        min_diff = min(min_diff, diff)

    return min_diff
```

### Complexity

| Metric | Value |
|--------|-------|
| **Time** | O(n! / (m! × (n-m)!)) — combinatorial explosion; completely impractical for moderate `n`. |
| **Space** | O(m) for each combination tuple (ignoring recursion/intermediate storage). |

---

## Approach 2: Optimal (Sorting + Sliding Window)

**Idea:** Sort the array. Once sorted, any contiguous subarray of size `m` gives the smallest possible spread for that set of elements. Slide a window of size `m` over the sorted array and take the minimum of `arr[i+m-1] - arr[i]`.

### Reasoning

If you sort the packets, the most similar-sized packets will be neighbours. So the optimal `m` packets must be a contiguous block in the sorted order — otherwise, swapping a far-out packet for a nearer one would only reduce the difference.

### Code

```python
def find_min_diff(arr, m):
    n = len(arr)
    if m > n or m == 0:
        return -1

    arr.sort()
    min_diff = float("inf")

    for i in range(n - m + 1):
        diff = arr[i + m - 1] - arr[i]
        min_diff = min(min_diff, diff)

    return min_diff
```

### Complexity

| Metric | Value |
|--------|-------|
| **Time** | O(n log n) — dominated by sorting. The single pass over the array is O(n). |
| **Space** | O(1) extra (or O(n) if sorting uses extra space — Python's Timsort is O(n) auxiliary space in the worst case). |

---

## Example

```
arr = [7, 3, 2, 4, 9, 12, 56], m = 3

Sorted: [2, 3, 4, 7, 9, 12, 56]

Windows:
  [2, 3, 4] → 4 - 2 = 2   ← minimum
  [3, 4, 7] → 7 - 3 = 4
  [4, 7, 9] → 9 - 4 = 5
  [7, 9, 12] → 12 - 7 = 5
  [9, 12, 56] → 56 - 9 = 47

Result: 2
```

---

## Edge Cases

| Case | Expected |
|------|----------|
| `m = 1` | Always `0` (one packet → no difference). |
| `m = n` | `max(arr) - min(arr)` — the whole array's spread. |
| `m > n` | Return `-1` — impossible to distribute. |
| Duplicates | Works fine — sorting groups equal values together, difference becomes `0`. |
| Single element | `m = 1` returns `0`. |
