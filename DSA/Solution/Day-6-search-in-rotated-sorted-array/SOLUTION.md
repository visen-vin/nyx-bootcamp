# Day 6 — Search in Rotated Sorted Array

**Problem:** Given an integer array `nums` sorted in ascending order (with **distinct** values) that has been rotated at an unknown pivot, and a `target` integer, return the index of `target` if it exists, or `-1` otherwise.

**Constraints:** `1 <= nums.length <= 5000`, `-10⁴ <= nums[i], target <= 10⁴`

---

## Approach 1 — Brute Force (Linear Scan)

**Idea:** Iterate through the entire array and compare each element with the target. The simplest possible solution.

```python
def search(nums: list[int], target: int) -> int:
    for i, num in enumerate(nums):
        if num == target:
            return i
    return -1
```

**Time complexity:** O(n) — single pass over `n` elements.

**Space complexity:** O(1) — only a loop variable.

**Why this matters:** It always works regardless of rotation, but it doesn't exploit the sorted-and-rotated property. Fine for small inputs, wasteful for large ones.

---

## Approach 2 — Optimal (Modified Binary Search)

**Idea:** Even though the array is rotated, one half of any mid-point split is always **normally sorted**. We can use modified binary search:

1. Find `mid`. At least one of `[left..mid]` or `[mid..right]` is fully sorted.
2. Check if `target` lies in the sorted half. If yes, binary search there. Otherwise, search the other half.
3. This eliminates half the array each iteration — classic binary search with one extra check.

```python
def search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1

    while left <= right:
        mid = (left + right) // 2

        if nums[mid] == target:
            return mid

        # Left half is sorted
        if nums[left] <= nums[mid]:
            if nums[left] <= target < nums[mid]:
                right = mid - 1
            else:
                left = mid + 1
        # Right half is sorted
        else:
            if nums[mid] < target <= nums[right]:
                left = mid + 1
            else:
                right = mid - 1

    return -1
```

**Time complexity:** O(log n) — each iteration halves the search space.

**Space complexity:** O(1) — only integer pointers.

**Key insight:** The condition `nums[left] <= nums[mid]` (or `<` for distinct values) tells us which side is normally sorted. From there it's standard binary search logic.

---

## Edge Cases to Watch

- **Single element array** — works with both approaches.
- **Target is the pivot element** — the binary search condition handles it.
- **Target not present** — both approaches return `-1`.
- **Array not rotated** (i.e., normal sorted array) — the modified binary search still works because `nums[left] <= nums[mid]` is true for the entire array, so it behaves like standard binary search.
