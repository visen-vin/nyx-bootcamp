# Day 4 — Contains Duplicate

**Language:** Python 3  
**Question:** Given an integer array `nums`, return `true` if any value appears **at least twice** in the array, and return `false` if every element is distinct.

---

## Approach 1 — Brute Force (Naive)

**Idea:** Compare every pair of elements. If any two are equal, return `true`. Otherwise `false`.

```python
def contains_duplicate_brute(nums):
    n = len(nums)
    for i in range(n):
        for j in range(i + 1, n):
            if nums[i] == nums[j]:
                return True
    return False
```

**Complexity:**
- **Time:** O(n²) — two nested loops over the array.
- **Space:** O(1) — no extra storage.

**Why not use this?** For n up to 10⁵, 10¹⁰ comparisons is far too slow. Interviewers will expect better.

---

## Approach 2 — Hash Set (Optimal)

**Idea:** Walk through the array once. For each element, check if it's already in a set. If yes → duplicate found. If no → add it to the set and continue.

```python
def contains_duplicate(nums):
    seen = set()
    for num in nums:
        if num in seen:
            return True
        seen.add(num)
    return False
```

**Complexity:**
- **Time:** O(n) — single pass; set lookups are O(1) average.
- **Space:** O(n) — the set stores up to n elements in the worst case.

**Why this is optimal:** You have to look at each element at least once (Ω(n)), so O(n) is the lower bound. The hash set trades O(n) memory for linear time — a standard and completely acceptable trade-off for this problem.

---

## Summary

| Approach | Time   | Space | When to Use                     |
|----------|--------|-------|----------------------------------|
| Brute    | O(n²)  | O(1)  | Tiny arrays, no memory allowed   |
| Hash Set | O(n)   | O(n)  | Every other case (preferred)     |
