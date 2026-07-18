# Day 10 — Kth Largest Element in an Array

## Problem

Given an integer array `nums` and an integer `k`, return the **kth largest element** in the array.

> Note: It is the kth largest element in **sorted order**, not the kth distinct element.

---

## Approach 1 — Brute Force (Sorting)

**Idea:** Sort the array in descending order and pick the element at index `k-1`.

### Complexity
- **Time:** O(n log n) — dominated by sorting.
- **Space:** O(1) or O(n) depending on the sort implementation (Python's Timsort uses O(n) auxiliary space in the worst case).

### Code

```python
def findKthLargest_bruteforce(nums: list[int], k: int) -> int:
    nums.sort(reverse=True)
    return nums[k - 1]
```

That's it — one line. Simple but not optimal.

---

## Approach 2 — Optimal (Quickselect / Hoare's Selection Algorithm)

**Idea:** Use the **Quickselect** algorithm — a variant of Quicksort that only recurses into the partition containing the kth largest element, giving average O(n) time.

Since we want the **kth largest**, we can equivalently look for the **(n-k)th smallest** (0-indexed). Pick a pivot, partition the array, and recurse into only one side.

### Complexity
- **Time:** O(n) average, O(n²) worst-case (if pivot choices are unlucky — mitigated by random pivot).
- **Space:** O(1) extra (in-place partitioning), O(log n) recursion stack average.

### Code

```python
import random

def findKthLargest(nums: list[int], k: int) -> int:
    # kth largest == (n - k)th smallest (0-indexed)
    target_idx = len(nums) - k

    def quickselect(left: int, right: int) -> int:
        # Standard random pivot
        pivot_idx = random.randint(left, right)
        pivot_val = nums[pivot_idx]

        # Move pivot to the end
        nums[pivot_idx], nums[right] = nums[right], nums[pivot_val]

        # Partition: elements < pivot go left, > pivot go right
        store = left
        for i in range(left, right):
            if nums[i] < pivot_val:
                nums[store], nums[i] = nums[i], nums[store]
                store += 1

        # Move pivot to its final position
        nums[right], nums[store] = nums[store], nums[right]

        # Decide which side to recurse into
        if store == target_idx:
            return nums[store]
        elif target_idx < store:
            return quickselect(left, store - 1)
        else:
            return quickselect(store + 1, right)

    return quickselect(0, len(nums) - 1)
```

---

## Approach 3 — Optimal (Min-Heap of Size k)

**Idea:** Maintain a min-heap of size `k` over the array. After processing all elements, the smallest element in this heap (the root) is the kth largest.

### Complexity
- **Time:** O(n log k) — each heap operation is O(log k), and we do n operations.
- **Space:** O(k) for the heap.

### Code

```python
import heapq

def findKthLargest_heap(nums: list[int], k: int) -> int:
    heap = []
    for num in nums:
        heapq.heappush(heap, num)
        if len(heap) > k:
            heapq.heappop(heap)
    return heap[0]
```

---

## Summary

| Approach        | Time         | Space   | Notes                                     |
|-----------------|--------------|---------|-------------------------------------------|
| Sorting         | O(n log n)   | O(1)†   | Simple, good for small n                  |
| Quickselect     | O(n) avg     | O(log n)| Optimal average — in-place               |
| Min-Heap        | O(n log k)   | O(k)    | Good when k << n, no recursion            |

† Python's Timsort uses O(n) auxiliary space in the worst case.

LeetCode's expected optimal solution is **Quickselect** (or the heap approach).
