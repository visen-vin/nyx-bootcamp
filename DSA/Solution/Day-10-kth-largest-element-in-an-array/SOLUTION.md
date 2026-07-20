# Day 10 — Kth-Largest Element in an Array

**Question:** Given an integer array `nums` and an integer `k`, return the *kth* largest element in the array. Note that it is the kth largest element in the **sorted order**, not the kth distinct element. You must solve it in **O(n)** time on average.

- [LeetCode 215 — Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/)

---

## Approach 1 — Brute Force (Sorting)

**Idea:** Sort the entire array in descending order and pick the element at index `k-1`.

```python
def find_kth_largest_brute(nums, k):
    nums.sort(reverse=True)
    return nums[k - 1]
```

### Complexity

| Metric | Value |
|---|---|
| **Time** | **O(n log n)** — dominated by sorting |
| **Space** | **O(1)** or **O(n)** depending on sort implementation (Python's Timsort uses O(n) auxiliary space in worst case) |

---

## Approach 2 — Optimal (QuickSelect, Hoare's Partition)

**Idea:** Use the QuickSelect algorithm — a variant of QuickSort that only recurses into the partition containing the kth largest element. On average this runs in **O(n)** time.

We look for the **(n - k)th** smallest element (0-indexed) so we don't have to reverse the comparison logic.

```python
import random

def find_kth_largest(nums, k):
    n = len(nums)
    target = n - k  # position of kth largest in sorted order (0-indexed)

    def quickselect(left, right):
        # Choose a random pivot to avoid worst-case O(n²) on sorted input
        pivot_idx = random.randint(left, right)
        pivot_val = nums[pivot_idx]

        # Move pivot to end
        nums[pivot_idx], nums[right] = nums[right], nums[pivot_idx]

        # Partition: move elements smaller than pivot to the left side
        store_idx = left
        for i in range(left, right):
            if nums[i] < pivot_val:
                nums[store_idx], nums[i] = nums[i], nums[store_idx]
                store_idx += 1

        # Restore pivot to its final position
        nums[right], nums[store_idx] = nums[store_idx], nums[right]

        # pivot is now at index store_idx
        if store_idx == target:
            return nums[store_idx]
        elif store_idx < target:
            return quickselect(store_idx + 1, right)
        else:
            return quickselect(left, store_idx - 1)

    return quickselect(0, n - 1)
```

### Complexity

| Metric | Value |
|---|---|
| **Time** | **O(n) average**, **O(n²) worst-case** — random pivot makes the worst-case vanishingly unlikely |
| **Space** | **O(1)** auxiliary (in-place partitioning) + **O(log n)** recursion stack on average |

---

## Approach 3 — Alternative Optimal (Min-Heap of size k)

**Idea:** Maintain a min-heap of size `k`. Push each element; if the heap exceeds size `k`, pop the smallest. At the end, the heap's top (root) is the kth largest element.

```python
import heapq

def find_kth_largest_heap(nums, k):
    heap = []
    for num in nums:
        heapq.heappush(heap, num)
        if len(heap) > k:
            heapq.heappop(heap)
    return heap[0]
```

### Complexity

| Metric | Value |
|---|---|
| **Time** | **O(n log k)** — each push/pop on a heap of size `k` is O(log k) |
| **Space** | **O(k)** — the heap stores at most `k` elements |

---

## Summary

| Approach | Time | Space | Notes |
|---|---|---|---|
| Sorting (Brute) | O(n log n) | O(1)–O(n) | Simplest, good enough for small inputs |
| QuickSelect (Optimal) | O(n) avg / O(n²) worst | O(log n) stack | Best average-case, in-place |
| Min-Heap of size k | O(n log k) | O(k) | Excellent when k << n |
