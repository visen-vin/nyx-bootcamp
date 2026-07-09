# Maximum and Minimum Element in an Array

**Topic:** Arrays
**Link:** https://www.geeksforgeeks.org/maximum-and-minimum-in-an-array/

---

## Problem Statement

Given an array of integers, find the maximum and minimum element in the array.

**Example:**
```
Input:  arr = [3, 5, 4, 1, 9]
Output: min = 1, max = 9

Input:  arr = [22, 14, 8, 17, 35, 3]
Output: min = 3, max = 35
```

---

## Approach 1: Brute Force (Linear Scan)

The simplest approach: traverse the array once, keep track of the current minimum and maximum.

### Algorithm
1. Initialize `min` and `max` as the first element of the array.
2. Loop through the array from index 1 to n-1.
3. For each element:
   - If it's less than `min`, update `min`.
   - If it's greater than `max`, update `max`.
4. Return `(min, max)`.

### Code

```python
def find_min_max_brute(arr):
    if not arr:
        return None, None
    
    n = len(arr)
    min_val = arr[0]
    max_val = arr[0]
    
    for i in range(1, n):
        if arr[i] < min_val:
            min_val = arr[i]
        elif arr[i] > max_val:
            max_val = arr[i]
    
    return min_val, max_val
```

### Complexity

| Measure | Value |
|---------|-------|
| **Time** | O(n) — single pass through the array |
| **Space** | O(1) — only two extra variables |

**Comparisons:** 2×(n−1) in the worst case (each element triggers both comparisons).

---

## Approach 2: Optimal (Tournament Method / Pair Comparison)

This approach reduces the total number of comparisons by processing elements in pairs.

### Algorithm
1. If the array has only one element, that element is both min and max.
2. Initialize `min` and `max` based on the first two elements.
3. Process the remaining elements in pairs `(arr[i], arr[i+1])`:
   - Compare the two elements with each other (1 comparison).
   - Compare the smaller with `min` (1 comparison).
   - Compare the larger with `max` (1 comparison).
   - Total: 3 comparisons per pair, vs 4 in the brute-force approach.
4. If the array length is odd, handle the last remaining element separately.

### Code

```python
def find_min_max_optimal(arr):
    if not arr:
        return None, None
    
    n = len(arr)
    i = 0
    
    # Initialization
    if n % 2 == 0:
        # Even length: compare first two elements
        if arr[0] < arr[1]:
            min_val, max_val = arr[0], arr[1]
        else:
            min_val, max_val = arr[1], arr[0]
        i = 2
    else:
        # Odd length: first element is both min and max
        min_val = max_val = arr[0]
        i = 1
    
    # Process in pairs
    while i < n - 1:
        if arr[i] < arr[i + 1]:
            local_min, local_max = arr[i], arr[i + 1]
        else:
            local_min, local_max = arr[i + 1], arr[i]
        
        if local_min < min_val:
            min_val = local_min
        if local_max > max_val:
            max_val = local_max
        
        i += 2
    
    return min_val, max_val
```

### Complexity

| Measure | Value |
|---------|-------|
| **Time** | O(n) — single pass, but fewer comparisons |
| **Space** | O(1) — only a few extra variables |

**Comparisons:** 3⌊n/2⌋ comparisons in total (or 3⌊n/2⌋ + 1 if n is odd), which is roughly **25% fewer** than the brute-force approach.

---

## Comparison Summary

| Aspect | Brute Force | Optimal (Tournament) |
|--------|------------|---------------------|
| Time Complexity | O(n) | O(n) |
| Comparisons | ~2n | ~1.5n |
| Space | O(1) | O(1) |
| Code Simplicity | Very simple | Slightly more involved |
| When to Use | Small arrays, interview clarity | Large arrays, comparison-costly environments |

The optimal method doesn't change the big-O time complexity, but in competitive programming or embedded systems where each comparison is expensive (e.g., comparing large structs or reading from slow I/O), saving ~25% of comparisons is meaningful.
