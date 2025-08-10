# Sliding Window — algorithm template

> Full analysis of the **Sliding Window** pattern — theory, examples, pseudocode, implementations, typical errors, complexity assessment, and memorization tips.

---

## Introduction

Sliding Window is one of the most universal and frequently used patterns for tasks related to sequences: arrays and strings. The point is to support the window (subsegment) on the sequence and move/stretch it, carefully updating the aggregated state (sum, symbol counter, number of unique ones, etc.) without recalculating from zero.

The pattern allows you to reduce the complexity of virtual solutions from O(n²) to linear O(n) in typical problems.

---

## When to apply

* Aggregate calculation for all subsegments: sum, maximum, minimum length satisfying the condition, number of unique elements in the window, etc.
* Tasks "substring/subarray with a condition" — for example: "find the maximum sum of length k", "find the minimum length of the subarray whose sum is ≥ S", "find the length of the largest substring without repeating characters".
* Any tasks where work with **continuous (contiguous)** segments is required.

If the problem allows for separate elements (optional continuity), the second approach (hash + two-sum, frequency tables, etc.) is more often suitable.

---

## Types of windows

1. **Fixed-size window (фиксированный размер окна)** — the window has a constant length k. Moves to the right one step at a time. Frequent tasks: the sum of k consecutive elements, the median in the window, the maxima of the sliding window.

2. **Variable-size window (пременный размер окна)** — the window changes its length in the process (usually through two pointers `left` and `right`). It is used when there is a condition that must be fulfilled within the window (for example, the sum ≥ S, or the number of unique ones ≤ K). Usually we use two pointers and move `right`, expanding until the condition is fulfilled/stops being true, then move `left`.

---

## General scheme (pseudocode)

### Fixed-size window

```
left = 0
window_sum = 0
for right in 0..n-1:
    window_sum += arr[right]
    if right - left + 1 == k:
        answer = max(answer, window_sum)  // пример
        window_sum -= arr[left]
        left += 1
```

### Variable-size window (typical template for searching for minimum length with sum ≥ S)

```
left = 0
window_sum = 0
min_len = INF
for right in 0..n-1:
    window_sum += arr[right]
    while window_sum >= S:
        min_len = min(min_len, right - left + 1)
        window_sum -= arr[left]
        left += 1
```

---

## Typical tasks and transformation into a pattern

1. **Fixed sliding window**: "maximum k consecutive sum" → clear fixed-window.
2. **Longest substring without repeating characters** → variable-size: expand `right', take character frequencies into account; If a repetition is found, move `left' until there are no repetitions left in the window.
3. **Minimum size subarray sum** → variable-size: expand `right` until the sum < S, then shrink `left` until the sum ≥ S.
4. **Longest subarray with at most K distinct elements** → variable-size with frequency dictionary; we move `left' when the number of unique > K.

---

## Implementation: C# — basic templates

### Fixed-size window (sum of k elements)

```csharp
public int MaxSumFixedWindow(int[] arr, int k) {
    if(arr == null || arr.Length < k) return 0;
    int n = arr.Length;
    int sum = 0;
    for(int i = 0; i < k; i++) sum += arr[i];
    int max = sum;
    for(int i = k; i < n; i++){
        sum += arr[i] - arr[i - k];
        if(sum > max) max = sum;
    }
    return max;
}
```

### Variable-size window: minimum length of subsequence with sum ≥ S

```csharp
public int MinSubArrayLen(int S, int[] nums) {
    int n = nums.Length;
    int left = 0, sum = 0, minLen = int.MaxValue;
    for(int right = 0; right < n; right++) {
        sum += nums[right];
        while(sum >= S) {
            minLen = Math.Min(minLen, right - left + 1);
            sum -= nums[left++];
        }
    }
    return minLen == int.MaxValue ? 0 : minLen;
}
```

### Longest substring without repeating characters (string) — C\#

```csharp
public int LengthOfLongestSubstring(string s) {
    int n = s.Length;
    var last = new int[128]; // или Dictionary<char,int>
    for(int i = 0; i < 128; i++) last[i] = -1;
    int left = 0, maxLen = 0;
    for(int right = 0; right < n; right++) {
        char c = s[right];
        if(last[c] >= left) {
            left = last[c] + 1;
        }
        last[c] = right;
        maxLen = Math.Max(maxLen, right - left + 1);
    }
    return maxLen;
}
```

---

## Variations and hybrids

* Sliding Window + HashMap (freq table) — when you need to store the frequencies of elements inside the window (for example, "at most K distinct").
* Sliding Window + Deque — for "max in sliding window" problems (a monotonic queue is used) — provides O(n).
* Sliding Window + Prefix sums — sometimes it's easier to maintain a prefix and look for borders (less often) — but often this leads to a binary search on the left border.

---

## Complexity

Fixed-size: O(n) in time, O(1) in memory (if input data is not taken into account). Sometimes O(k) due to additional structures.

Variable-size: O(n) in time — each pointer (left and right) moves a maximum of n times; O(m) in memory, where m is the size of additional structures (hashmap/deque), usually O(1) or O(alphabet).

Typical mistakes and how to avoid them

State recalculation — do not use nested loops where the aggregate can be maintained. Update state on left/right shift.

Incorrect initialization of boundaries — check left/right boundaries and exit conditions.

Repeats are not taken into account when working with strings - when using the last[char] array, remember the initial value (usually -1).

An error with the while condition — in variable-window, the while(condition) construct should decrease the window, not increase it — make sure that the condition is inverted correctly.

Inefficiency — at the maximum in the window, use deque instead of recalculation.


## Useful techniques and "micro-patterns"

"Shift of the sum": when removing arr[left] from the sum, make sum -= arr[left++] in one line.

Use left and right as inclusive intervals: window = [left, right].

For ASCII/UTF-8 characters, an int[256] / int[128] array is often sufficient instead of Dictionary.

For tasks with the maximum element in the window, there is a monotonic queue (Deque<int> with indices).


## Examples of problems for practice (with tips)

Maximum Sum Subarray of Size K — fixed window, O(n).

Minimum Size Subarray Sum — variable window, O(n).

Longest Substring Without Repeating Characters — variable window + last occurrence table.

Longest Substring with At Most K Distinct Characters — variable window + freq map.

Sliding Window Maximum — deque (monotonic queue).

(These tasks are on LeetCode/Codeforces/HackerRank; I recommend doing them in order from simple to complex.)


## Memorization tips

If you see "substring", "subarray", "following fragment" - the first candidate should be Sliding Window.

Solve 5–10 problems in a row on this pattern - neural connections are formed faster.

When comparing solutions, pay attention to: how the state is updated upon shift and what structures are used (array, hashmap, deque).

## Conclusion

Sliding Window is a key tool in a programmer's arsenal, especially when preparing for interviews and programming competitions. Having mastered fixed/variable windows, monotonous queues and hybrids with hash tables, you will be able to solve a large layer of problems faster and more reliably.
