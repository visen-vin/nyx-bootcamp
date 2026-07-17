# Day 9 — Repeat and Missing Number Array

## Problem Statement

Given an array of size **N** containing integers from **1 to N** (inclusive), one number appears **twice** (the repeating number) and one number is **missing** from the range. Find the repeating and missing numbers.

Return them as `[repeating, missing]`.

### Example

```
Input:  [3, 1, 3]
Output: [3, 2]     (3 is repeated, 2 is missing)

Input:  [4, 3, 6, 2, 1, 1]
Output: [1, 5]     (1 is repeated, 5 is missing)
```

---

## Approach 1 — Brute Force (O(N²) time, O(1) space)

Count the frequency of every number from 1 to N by scanning the array for each number.

### Algorithm

1. For each number `i` from 1 to N:
   - Scan the array and count how many times `i` appears.
   - If count == 2 → `repeating = i`
   - If count == 0 → `missing = i`
2. Return `[repeating, missing]`.

### Code

```python
def find_repeat_and_missing_bruteforce(arr):
    n = len(arr)
    repeating = -1
    missing = -1

    for i in range(1, n + 1):
        count = 0
        for num in arr:
            if num == i:
                count += 1
        if count == 2:
            repeating = i
        elif count == 0:
            missing = i

    return [repeating, missing]
```

### Complexity

- **Time:** O(N²) — for each of the N numbers, we scan the array.
- **Space:** O(1) — only a few variables.

---

## Approach 2 — Optimal (Mathematical — O(N) time, O(1) space)

Use the difference between actual sums and expected sums to derive the missing and repeating numbers.

### Intuition

Let:
- `S = 1 + 2 + ... + N = N*(N+1)/2` (sum of first N natural numbers)
- `P = 1² + 2² + ... + N² = N*(N+1)*(2N+1)/6` (sum of squares)

Let `arr_sum` = actual sum of the array, `arr_sq_sum` = actual sum of squares.

If `R` = repeating number and `M` = missing number:

1. `arr_sum - S = R - M`  →  `diff = R - M`
2. `arr_sq_sum - P = R² - M² = (R - M)(R + M)`  →  `sum = R + M`

From (1) and (2):
- `R = (diff + sum) / 2`
- `M = sum - R`

### Algorithm

1. Compute `n = len(arr)`
2. Compute `expected_sum = n*(n+1)//2` and `expected_sq_sum = n*(n+1)*(2*n+1)//6`
3. Compute `actual_sum = sum(arr)` and `actual_sq_sum = sum(x*x for x in arr)`
4. `diff = actual_sum - expected_sum`  (= R - M)
5. `sq_diff = actual_sq_sum - expected_sq_sum`  (= R² - M²)
6. `sum_rm = sq_diff // diff`  (= R + M)
7. `repeating = (diff + sum_rm) // 2`
8. `missing = sum_rm - repeating`
9. Return `[repeating, missing]`

### Code

```python
def find_repeat_and_missing_optimal(arr):
    n = len(arr)

    # Expected sums
    expected_sum = n * (n + 1) // 2
    expected_sq_sum = n * (n + 1) * (2 * n + 1) // 6

    # Actual sums
    actual_sum = sum(arr)
    actual_sq_sum = sum(x * x for x in arr)

    # diff = R - M
    diff = actual_sum - expected_sum
    # sq_diff = R² - M² = (R - M)(R + M)
    sq_diff = actual_sq_sum - expected_sq_sum

    # R + M = sq_diff / diff
    sum_rm = sq_diff // diff

    repeating = (diff + sum_rm) // 2
    missing = sum_rm - repeating

    return [repeating, missing]
```

### Complexity

- **Time:** O(N) — two passes through the array (sum and sum of squares).
- **Space:** O(1) — only a handful of integer variables.

> **Note on large inputs:** The sum of squares can overflow 32-bit integers for large N (e.g., N = 10⁵ → sum of squares ≈ 3.3 × 10¹⁴, well within Python's big ints). If using a 32-bit language like C++/Java, use `long long` or `long`.

---

## Approach 3 — XOR-based Alternative (O(N) time, O(1) space)

XOR all array elements with 1..N. The result is `R ^ M` (since paired numbers cancel). Find a set bit in this XOR to partition both sets, then XOR each partition to isolate R and M. Finally determine which is repeating and which is missing with one more pass.

Excellent for languages without big integers; Python's big ints make the mathematical approach equally clean.

---

## Summary

| Approach | Time   | Space | Notes                        |
|----------|--------|-------|------------------------------|
| Brute    | O(N²)  | O(1)  | Simple, but too slow for N>~10³ |
| Optimal  | O(N)   | O(1)  | Mathematical — clean & fast    |
| XOR      | O(N)   | O(1)  | No overflow concerns           |
