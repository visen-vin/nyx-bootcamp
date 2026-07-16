# Day 8 — Best Time to Buy and Sell Stock

**LeetCode 121** — [Solve it here](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

---

## Problem

You are given an array `prices` where `prices[i]` is the price of a given stock on the `i`-th day.

You want to maximize your profit by choosing a **single day** to buy one stock and choosing a **different day in the future** to sell that stock.

Return the maximum profit you can achieve. If no profit is possible, return `0`.

### Example

```
Input:  prices = [7, 1, 5, 3, 6, 4]
Output: 5

Explanation: Buy on day 2 (price = 1), sell on day 5 (price = 6), profit = 6 - 1 = 5.
```

```
Input:  prices = [7, 6, 4, 3, 1]
Output: 0

Explanation: No transaction yields a positive profit, so return 0.
```

---

## 1. Brute-Force Approach

Try every possible pair of buy day and sell day. Compute the profit for
each and keep the maximum.

```python
def max_profit_bruteforce(prices):
    n = len(prices)
    max_profit = 0
    for i in range(n):
        for j in range(i + 1, n):
            profit = prices[j] - prices[i]
            if profit > max_profit:
                max_profit = profit
    return max_profit
```

### Complexity

| Metric | Value |
|--------|-------|
| Time   | **O(n²)** — nested loops over all `(i, j)` pairs |
| Space  | **O(1)** — constant extra space |

**Why not use this?** For `n` up to 10⁵ (or more), O(n²) is too slow.

---

## 2. Optimal Approach — One Pass (Kadane-inspired)

Scan left-to-right once, keeping track of the **minimum price seen so far**.
At each day, compute the profit if we sold *today*:
`prices[i] - min_so_far`. Track the maximum across all days.

```python
def max_profit(prices):
    min_price = float('inf')
    max_profit = 0

    for price in prices:
        if price < min_price:
            min_price = price          # new best day to buy
        else:
            profit = price - min_price
            if profit > max_profit:
                max_profit = profit    # better selling day found

    return max_profit
```

### Complexity

| Metric | Value |
|--------|-------|
| Time   | **O(n)** — single pass over the array |
| Space  | **O(1)** — two integer variables |

This is the textbook solution for LeetCode 121.

---

## Key Insight

You only care about the *lowest price before today* — you don't need to
check every future day (brute force does that wastefully). By tracking
the minimum on the fly, each day you can immediately answer: *"Should I
sell today?"* The answer depends only on the cheapest price that came
before.

```
prices  =  [7,  1,  5,  3,  6,  4]
min_so_far  7   1   1   1   1   1
profit       0   0   4   2   5   3
                          ↑ answer = 5
```
