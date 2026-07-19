# Day 10 — Kth Largest Element in an Array

**Problem:** Given an integer array `nums` and an integer `k`, return the *kth largest element* in the array.

Note that it is the kth largest element in the sorted order, not the kth distinct element.

---

## Approach 1 — Brute Force (Sorting)

Sort the array in descending order, then pick the element at index `k-1`.

```python
def find_kth_largest_brute(nums: list[int], k: int) -> int:
    nums.sort(reverse=True)
    return nums[k - 1]
```

**Time Complexity:** O(n log n) — dominated by sorting.  
**Space Complexity:** O(1) or O(n) depending on the sort implementation (Python's Timsort uses O(n) auxiliary space in the worst case).

---

## Approach 2 — Optimal (Quickselect / Hoare's Selection Algorithm)

Quickselect is a selection algorithm based on the partition step of Quicksort. It repeatedly partitions the array around a pivot until the pivot lands at the (k-1)th position (0-indexed for kth largest). Since we want the kth *largest*, we partition in descending order.

Average case is O(n), which is better than O(n log n) sorting.

```python
import random

def find_kth_largest_optimal(nums: list[int], k: int) -> int:
    # Convert to kth largest → kth smallest index for easier partitioning
    # kth largest = nums[len - k] in ascending order
    # But we can also partition in descending order.
    # We'll work with 0-indexed target: index = k - 1 in descending order.
    target = k - 1
    left, right = 0, len(nums) - 1

    while left <= right:
        pivot_idx = random.randint(left, right)
        pivot_idx = partition_desc(nums, left, right, pivot_idx)

        if pivot_idx == target:
            return nums[pivot_idx]
        elif pivot_idx < target:
            left = pivot_idx + 1
        else:
            right = pivot_idx - 1

    return -1  # should never reach here


def partition_desc(arr: list[int], left: int, right: int, pivot_idx: int) -> int:
    """Partition arr[left..right] around pivot (descending order).
    Returns the final index of the pivot.
    """
    pivot_val = arr[pivot_idx]
    # Move pivot to end
    arr[pivot_idx], arr[right] = arr[right], arr[pivot_idx]

    store_idx = left
    for i in range(left, right):
        if arr[i] > pivot_val:  # descending: larger elements go left
            arr[store_idx], arr[i] = arr[i], arr[store_idx]
            store_idx += 1

    # Move pivot to its final place
    arr[right], arr[store_idx] = arr[store_idx], arr[right]
    return store_idx
```

**Time Complexity:** O(n) average, O(n²) worst case (rare with random pivot).  
**Space Complexity:** O(1) — in-place partitioning, no extra data structures.

---

## Approach 3 — Alternative Optimal (Min-Heap of size k)

Use a min-heap of size k. Iterate through the array; keep the k largest elements in the heap. The top of the heap is the kth largest.

```python
import heapq

def find_kth_largest_heap(nums: list[int], k: int) -> int:
    heap = nums[:k]
    heapq.heapify(heap)  # min-heap of size k

    for x in nums[k:]:
        if x > heap[0]:
            heapq.heapreplace(heap, x)

    return heap[0]
```

**Time Complexity:** O(n log k) — each heap operation is O(log k).  
**Space Complexity:** O(k) — the heap stores k elements.

---

### Summary

| Approach    | Time         | Space   | Notes                              |
|-------------|-------------|---------|------------------------------------|
| Brute force | O(n log n)  | O(1)/O(n)| Simple, always correct            |
| Quickselect | O(n) avg    | O(1)    | Mutates input, fast average       |
| Min-heap    | O(n log k)  | O(k)    | No mutation, good for streaming   |

For k small relative to n, the heap approach can be very practical. Quickselect is the asymptotic winner.
