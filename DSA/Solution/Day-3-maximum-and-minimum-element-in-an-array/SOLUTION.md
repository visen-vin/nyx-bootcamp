# Maximum and Minimum Element in an Array

Find the maximum and minimum elements in a given unsorted array.

## 1. Brute-Force Approach — Linear Scan

### Intuition
Simply traverse the array once, keeping track of the minimum and maximum seen so far.

### Code

```python
def find_min_max(arr):
    if not arr:
        return None, None

    min_val = arr[0]
    max_val = arr[0]

    for x in arr[1:]:
        if x < min_val:
            min_val = x
        if x > max_val:
            max_val = x

    return min_val, max_val
```

### Complexity Analysis

| Metric | Value |
|--------|-------|
| Time Complexity | **O(n)** — single pass through the array |
| Space Complexity | **O(1)** — constant extra space |

This is already optimal in terms of time complexity (you must look at every element), but we can **optimize the number of comparisons**, which matters when comparisons are expensive (e.g., large structs).

---

## 2. Optimal Approach — Tournament Method (Divide & Conquer)

### Intuition
We can reduce the number of comparisons from `2n` to `~3n/2` by processing elements in pairs.

**How it works:**
1. Split the array into pairs.
2. Compare the two elements in each pair — the smaller gets compared with the current minimum, the larger with the current maximum.
3. This saves 1 comparison per pair compared to the brute-force approach.

For an array of size `n`:
- **n is even:** `3n/2 - 2` comparisons
- **n is odd:** `3n/2` comparisons (floor division)

This is provably optimal — you can't find both min and max in fewer than `⌈3n/2⌉ - 2` comparisons.

### Code

```python
def find_min_max_optimal(arr):
    if not arr:
        return None, None

    n = len(arr)

    # Initialize min and max
    if n % 2 == 0:
        # Even length: compare first pair
        if arr[0] < arr[1]:
            min_val, max_val = arr[0], arr[1]
        else:
            min_val, max_val = arr[1], arr[0]
        i = 2
    else:
        # Odd length: initialize with first element
        min_val = max_val = arr[0]
        i = 1

    # Process remaining elements in pairs
    while i < n - 1:
        if arr[i] < arr[i + 1]:
            local_min, local_max = arr[i], arr[i + 1]
        else:
            local_min, local_max = arr[i + 1], arr[i]

        # Compare local min with global min, local max with global max
        if local_min < min_val:
            min_val = local_min
        if local_max > max_val:
            max_val = local_max

        i += 2

    return min_val, max_val
```

### Complexity Analysis

| Metric | Value |
|--------|-------|
| Time Complexity | **O(n)** — still linear time |
| Comparisons | **⌈3n/2⌉ - 2** — provably optimal |
| Space Complexity | **O(1)** — iterative, no recursion overhead |

### Comparison Count Table

| n | Brute-Force (2n comparisons) | Tournament Method (⌈3n/2⌉ - 2) |
|---|------------------------------|--------------------------------|
| 2 | 4 | 1 |
| 5 | 10 | 6 |
| 10 | 20 | 13 |
| 100 | 200 | 148 |
| 10⁶ | 2×10⁶ | ~1.5×10⁶ |

---

## Summary

- **Brute-force** is simple and perfectly fine for everyday use — O(n) time, O(1) space.
- **Tournament method** is the optimal comparison-minimized approach, useful when comparisons are expensive.
- Both run in **linear time**; the optimization is about constant factors.
- **Edge cases handled:** empty array → returns `(None, None)`, single-element array, duplicates, all equal elements.
