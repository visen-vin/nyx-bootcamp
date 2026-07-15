# Day 7 — Next Permutation

## Problem Statement
Implement **next permutation**, which rearranges numbers into the lexicographically next greater permutation of numbers. If such arrangement is not possible, it must rearrange it as the lowest possible order (i.e., sorted in ascending order). The replacement must be **in-place** and use only constant extra memory.

**Example 1:**
```
Input:  nums = [1,2,3]
Output: [1,3,2]
```

**Example 2:**
```
Input:  nums = [3,2,1]
Output: [1,2,3]
```

**Example 3:**
```
Input:  nums = [1,1,5]
Output: [1,5,1]
```

---

## Approach 1 — Brute Force (Generate All Permutations)

**Idea:** Generate every possible permutation of the array, sort them lexicographically, find the current arrangement, and return the next one. If the current arrangement is the last, return the first.

**Complexity:**
- **Time:** O(n × n!) — there are n! permutations and each permutation takes O(n) to generate or copy.
- **Space:** O(n × n!) — storing all permutations.

**Code:**
```python
from itertools import permutations

def nextPermutation_bruteforce(nums):
    """
    Brute-force: generate all permutations, sort, find next.
    """
    sorted_nums = sorted(nums)
    perms = sorted(set(permutations(sorted_nums)))  # set to handle duplicates
    n = len(perms)
    for i in range(n):
        if list(perms[i]) == nums:
            if i + 1 < n:
                nums[:] = list(perms[i + 1])
            else:
                nums[:] = list(perms[0])
            return
```

> ⚠️ This approach is **not feasible** for large arrays (n > 8 or so). It exists mainly to verify correctness of the optimal approach on small inputs.

---

## Approach 2 — Optimal (Single-Pass Reverse)

**Idea (O(n) time, O(1) space):**

The next permutation can be found by observing the following pattern:

1. **Find the first decreasing element** from the right — this is the "pivot" we need to change.  
   Traverse from right to left and find index `i` such that `nums[i] < nums[i+1]`.  
   If no such `i` exists, the array is in descending order — reverse the whole thing (it's the last permutation).

2. **Find the smallest element larger than `nums[i]`** from the right side (the suffix) — call that `j`.

3. **Swap `nums[i]` and `nums[j]`** — this pushes the pivot to the next higher value.

4. **Reverse the suffix** starting from `i+1` — the suffix after the pivot was in descending order (largest arrangement). Reversing it gives the smallest arrangement, which is the correct next state.

### Dry Run

```
nums = [1, 3, 5, 4, 2]

Step 1: Find first decreasing from right
        1, 3, 5, 4, 2
           ↑  (nums[1]=3 < nums[2]=5)
        i = 1

Step 2: Find smallest element > nums[i] in suffix [5,4,2]
        j = 2 (nums[2]=5)

Step 3: Swap nums[1] and nums[2]
        [1, 5, 3, 4, 2]

Step 4: Reverse suffix from i+1 = 2
        suffix [3,4,2] -> reverse -> [2,3,4]
        [1, 5, 2, 3, 4] ✅
```

### Complexity
- **Time:** O(n) — single pass to find the pivot, at most one more pass to find `j`, and one reverse pass over the suffix.
- **Space:** O(1) — in-place modifications only.

### Code
```python
def nextPermutation(nums):
    """
    Modifies nums in-place to the next lexicographical permutation.
    """
    n = len(nums)
    
    # Step 1: Find first decreasing element from the right
    i = n - 2
    while i >= 0 and nums[i] >= nums[i + 1]:
        i -= 1
    
    if i >= 0:
        # Step 2: Find the smallest element larger than nums[i] to the right
        j = n - 1
        while nums[j] <= nums[i]:
            j -= 1
        # Step 3: Swap
        nums[i], nums[j] = nums[j], nums[i]
    
    # Step 4: Reverse the suffix (if i == -1, this reverses the whole array)
    left, right = i + 1, n - 1
    while left < right:
        nums[left], nums[right] = nums[right], nums[left]
        left += 1
        right -= 1
```

### Handling Edge Cases

| Case | Input | Output |
|---|---|---|
| Single element | `[1]` | `[1]` |
| Two elements | `[1,2]` | `[2,1]` |
| Last permutation | `[3,2,1]` | `[1,2,3]` |
| Duplicates | `[1,1,5]` | `[1,5,1]` |
| Large input | `[1,2,3,...,100]` | `[1,2,3,...,100,99]` → works in O(n) |

### Why This Works (Intuition)

Think of the array as a number — we want the next larger number using the same digits. The rightmost part that can be incremented is the first dip from the right (because everything to the right is already in descending order = the largest arrangement for that suffix). Swapping the dip with the next larger digit in the suffix bumps the number up minimally. Reversing the suffix sets it to its smallest arrangement, giving the *next* permutation rather than a jump.
