# Day 12 — Product of Array Except Self

**Problem:** Given an integer array `nums`, return an array `answer` such that `answer[i]` is equal to the product of all the elements of `nums` except `nums[i]`.

Must solve **without division** and in **O(n)** time.

---

## 1. Brute-Force Approach

### Idea
For each index `i`, iterate over the entire array and multiply all elements except `nums[i]`.

### Code

```python
def product_except_self_bruteforce(nums):
    n = len(nums)
    result = [1] * n

    for i in range(n):
        product = 1
        for j in range(n):
            if i != j:
                product *= nums[j]
        result[i] = product

    return result
```

### Complexity
- **Time:** O(n²) — nested loops, n elements each computing n−1 multiplications.
- **Space:** O(1) extra (output array not counted as extra space per convention).

---

## 2. Optimal Approach (Prefix & Suffix Products)

### Idea
Use two pass technique:
1. **Left-to-right pass:** `prefix[i]` = product of all elements to the left of `i`.
2. **Right-to-left pass:** `suffix[i]` = product of all elements to the right of `i`.
3. Combine: `answer[i] = prefix[i] * suffix[i]`.

We can avoid two extra arrays by computing the answer in the same array:
- First pass: store left products directly in `answer`.
- Second pass: keep a running `right_product` and multiply into `answer`.

### Code

```python
def product_except_self(nums):
    n = len(nums)
    answer = [1] * n

    # Left pass: answer[i] = product of everything to the left of i
    left = 1
    for i in range(n):
        answer[i] = left
        left *= nums[i]

    # Right pass: multiply by product of everything to the right of i
    right = 1
    for i in range(n - 1, -1, -1):
        answer[i] *= right
        right *= nums[i]

    return answer
```

### Walkthrough (nums = [1, 2, 3, 4])

| i | Left pass (answer after step) | Right pass (answer after step) |
|---|-------------------------------|--------------------------------|
| 0 | answer = [1, 1, 1, 1], left = 1 | (i=3) answer[3]= 1×1 = 1, right=4 |
| 1 | answer = [1, 1, 2, 1], left = 2 | (i=2) answer[2]= 2×4 = 8, right=12 |
| 2 | answer = [1, 1, 2, 6], left = 6 | (i=1) answer[1]= 1×12 = 12, right=24 |
| 3 | answer = [1, 1, 2, 6], left = 24 | (i=0) answer[0]= 1×24 = 24, right=24 |

**Final answer:** `[24, 12, 8, 6]` ✅

### Complexity
- **Time:** O(n) — two linear passes.
- **Space:** O(1) extra (output array not counted as extra space).

---

## Edge Cases
- **Single element `[x]`:** result is `[1]` (product of zero other elements).
- **Array with zeros:** handled correctly by multiplication — no division used, so zeros don't cause division-by-zero errors.
- **All zeros:** unchanged handling — each position gets product of everything else, which will usually be zero unless n=1.
