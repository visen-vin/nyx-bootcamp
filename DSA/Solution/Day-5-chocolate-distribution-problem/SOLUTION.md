# Day 5 — Chocolate Distribution Problem

## Problem Statement

Given an array `arr` of `n` integers where each integer represents the number of chocolates in a packet, and an integer `m` representing the number of students, distribute the packets such that:

1. Each student gets **exactly one** packet.
2. The difference between the packet with the **most chocolates** and the packet with the **fewest chocolates** given to the students is **minimized**.

Return that minimum difference.

**Constraints:**
- `1 <= m <= n <= 10^5`
- `0 <= arr[i] <= 10^9`

---

## Approach 1: Brute Force

### Intuition
Generate every possible combination of `m` packets from `n` packets. For each combination, find the max and min values and compute the difference. Track the minimum difference across all combinations.

### Code (Python)

```python
from itertools import combinations

def chocolate_distribution_brute(arr, m):
    if m == 0 or len(arr) < m:
        return 0

    n = len(arr)
    min_diff = float('inf')

    for subset in combinations(arr, m):
        diff = max(subset) - min(subset)
        if diff < min_diff:
            min_diff = diff

    return min_diff
```

### Complexity

| Metric        | Value              |
|---------------|--------------------|
| **Time**      | O(C(n, m) * m)     |
| **Space**     | O(m)               |

The number of combinations grows factorially — completely infeasible for `n` up to `10^5`.

---

## Approach 2: Optimal (Sorting + Sliding Window)

### Intuition
If the array is sorted, any contiguous subarray of size `m` represents a valid distribution where the **minimum difference** between max and min in that window is simply `arr[i + m - 1] - arr[i]`. We don't need to consider non-contiguous subsets because the subarray property of sorted arrays guarantees that the min and max of any group are the smallest and largest elements in that group — and the elements between them just add more candy, never reduce the diff. By sliding the window of size `m` across the sorted array, we find the global minimum in O(n log n) time.

### Code (Python)

```python
def chocolate_distribution(arr, m):
    if m == 0 or len(arr) < m:
        return 0

    arr.sort()
    min_diff = float('inf')

    # Slide a window of size m across the sorted array
    for i in range(len(arr) - m + 1):
        diff = arr[i + m - 1] - arr[i]
        if diff < min_diff:
            min_diff = diff

    return min_diff
```

### Complexity

| Metric        | Value        |
|---------------|--------------|
| **Time**      | O(n log n)   |
| **Space**     | O(1)         |

**Why this works:** After sorting, the packets are ordered by chocolate count. For any subset of `m` packets, the smallest possible max-min difference is achieved by a **contiguous** segment of the sorted array — jumping gaps would only increase the spread. The sliding window checks every possible contiguous segment of length `m`.

---

## Test Cases

```python
def test():
    # Basic
    assert chocolate_distribution([7, 3, 2, 4, 9, 12, 56], 3) == 2  # subarray [2, 3, 4]
    # Single student
    assert chocolate_distribution([7, 3, 2], 1) == 0                # any packet alone
    # All same values
    assert chocolate_distribution([5, 5, 5, 5], 2) == 0
    # Large spread
    assert chocolate_distribution([1, 2, 3, 100, 101], 2) == 1      # subarray [100, 101]
    # m == n
    assert chocolate_distribution([10, 20, 30], 3) == 20
    # Edge: m == 0
    assert chocolate_distribution([1, 2, 3], 0) == 0

    print("All tests passed!")

test()
```

---

## Key Insight

The brute force approach enumerates combinations (exponential), while the optimal approach exploits **sorting + sliding window** to reduce the problem to a linear scan after sorting. Classic example of how a simple observation about ordering can collapse an exponential problem into a near-linear one.
