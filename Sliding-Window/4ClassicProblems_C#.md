# Sliding Window Pattern – C# Guide

This guide covers **4 classic problems** that can be solved using the **Sliding Window** technique.  
We will explain each problem, show its C# solution, and analyze the time and space complexity.

---

## 1. Maximum Sum of Subarray of Size K

**Problem:**  
Given an array of integers `nums` and an integer `k`, find the maximum sum of any contiguous subarray of size `k`.

**Approach (Sliding Window, Detailed):**  
A brute-force solution would check every subarray of size `k`, summing each — this takes **O(n·k)** time.

With Sliding Window:
- Start with a window of size `k` at the beginning of the array.
- Compute the sum of the first `k` elements.
- Then slide the window forward by one element at a time:
  - Subtract the element that’s leaving the window.
  - Add the new element that’s entering.
- Track the maximum sum seen so far.

Why this works:
- We avoid recomputing the entire sum for each window.
- Each element is added and removed once → **O(n)** time.

**C# Code:**

```csharp
public int MaxSumSubarray(int[] nums, int k) {
    int maxSum = 0, windowSum = 0;
    for (int i = 0; i < k; i++) {
        windowSum += nums[i];
    }
    maxSum = windowSum;

    for (int i = k; i < nums.Length; i++) {
        windowSum += nums[i] - nums[i - k];
        maxSum = Math.Max(maxSum, windowSum);
    }

    return maxSum;
}
```

### Complexity:

- Time: O(n)
- Space: O(1)

## 2. Longest Substring Without Repeating Characters
**Problem:** Given a string s, find the length of the longest substring without repeating characters.

**Approach (Sliding Window, Detailed):** 
Brute-force would check all substrings and verify uniqueness — O(n²) time.

With Sliding Window:

- Use a HashSet to track characters in the current window.
- Expand the window by moving right forward.
- If a duplicate is found, shrink the window from the left (left++) until the duplicate is removed.
- Track the maximum window size.

**Why this works:**

- Each character is added and removed at most once.
- Efficient in both time and space.

C# Code:

```csharp
public int LengthOfLongestSubstring(string s) {
    var seen = new HashSet<char>();
    int left = 0, maxLen = 0;

    for (int right = 0; right < s.Length; right++) {
        while (seen.Contains(s[right])) {
            seen.Remove(s[left]);
            left++;
        }
        seen.Add(s[right]);
        maxLen = Math.Max(maxLen, right - left + 1);
    }

    return maxLen;
}
```
### Complexity:
- Time: O(n)
- Space: O(n)

## 3. Minimum Size Subarray Sum
**Problem:** Given an array of positive integers nums and an integer target, 
find the minimal length of a contiguous subarray of which the sum is ≥ target.
Return 0 if no such subarray exists.

**Approach (Sliding Window, Detailed):** 
Brute-force would check all subarrays — O(n²) time.


With Sliding Window:

- Use two pointers left and right to define the window.
- Expand right to increase the sum.
- When the sum ≥ target, shrink left to find the minimal length.
- Track the smallest window that satisfies the condition.

Why this works:

- We only expand and contract the window as needed.
- Each element is processed at most twice.

C# Code:

```csharp
public int MinSubArrayLen(int target, int[] nums) {
    int left = 0, sum = 0, minLen = int.MaxValue;

    for (int right = 0; right < nums.Length; right++) {
        sum += nums[right];
        while (sum >= target) {
            minLen = Math.Min(minLen, right - left + 1);
            sum -= nums[left];
            left++;
        }
    }

    return minLen == int.MaxValue ? 0 : minLen;
}
```
### Complexity:

- Time: O(n)
- Space: O(1)

## 4. Find All Anagrams in a String
**Problem:**
Given two strings s and p, return all start indices of p's anagrams in s.

**Approach (Sliding Window, Detailed):**
Brute-force would generate all substrings of length p.Length and check if they are anagrams — O(n·k) time.

With Sliding Window:

- Use two frequency arrays or dictionaries: one for p, one for the current window in s.
- Slide a window of size p.Length across s.
- At each step, compare the frequency maps.
- If they match → record the starting index.

Why this works:

- Frequency comparison is constant time (fixed alphabet size).
- We avoid recomputing frequencies from scratch.

C# Code:

```csharp
public IList<int> FindAnagrams(string s, string p) {
    var result = new List<int>();
    if (s.Length < p.Length) return result;

    int[] pCount = new int[26];
    int[] sCount = new int[26];

    for (int i = 0; i < p.Length; i++) {
        pCount[p[i] - 'a']++;
        sCount[s[i] - 'a']++;
    }

    if (pCount.SequenceEqual(sCount)) result.Add(0);

    for (int i = p.Length; i < s.Length; i++) {
        sCount[s[i] - 'a']++;
        sCount[s[i - p.Length] - 'a']--;
        if (pCount.SequenceEqual(sCount)) result.Add(i - p.Length + 1);
    }

    return result;
}
```

### Complexity:
- Time: O(n)
- Space: O(1) (fixed alphabet size)

---

## 🧠 Conclusion

The **Sliding Window** technique is a powerful optimization strategy for problems involving **contiguous sequences** — especially when you're trying to compute **maximums, minimums, sums, counts, or uniqueness** over subarrays or substrings.

It replaces brute-force nested loops with a dynamic window that slides across the input, maintaining just enough state to answer the problem efficiently.

---

### ✅ Advantages

- **Time Complexity Reduction:**  
  Sliding Window often reduces time from **O(n²)** to **O(n)** by avoiding redundant recalculations.  
  For example, instead of summing every subarray from scratch, we update the sum incrementally as the window moves.

- **Constant or Linear Space Usage:**  
  Most implementations use **O(1)** or **O(k)** space, depending on whether you're tracking counts, sets, or frequency maps.  
  This makes it suitable for large-scale inputs where memory is a constraint.

- **Single-Pass Traversal:**  
  The input is usually scanned once, with each element entering and exiting the window at most once.  
  This leads to predictable performance and easy debugging.

- **Versatile Across Domains:**  
  Works for arrays, strings, and even streams — anywhere you need to process a moving segment of data.

- **Elegant and Readable:**  
  Once understood, Sliding Window solutions are often concise and expressive, making them ideal for interviews and production code.

---

### ⚠️ Limitations

- **Requires Contiguity:**  
  Sliding Window only works when the elements you're analyzing are **adjacent**.  
  If the problem involves non-contiguous combinations (e.g., arbitrary triplets), other techniques like hashing or two pointers may be better.

- **Edge Case Sensitivity:**  
  Shrinking and expanding the window must be handled carefully:
  - Off-by-one errors are common.
  - You must ensure the window always satisfies the problem constraints before recording results.

- **Fixed vs. Dynamic Window Size:**  
  Some problems require a **fixed-size window** (e.g., max sum of size k), while others need a **variable-size window** (e.g., smallest subarray with sum ≥ target).  
  Understanding which type applies is key to designing the correct logic.

- **Not Always Applicable:**  
  If the problem requires checking **all combinations**, or the data must remain unsorted and untouched, Sliding Window may not help.  
  For example, problems involving permutations or subsets often need backtracking or hashing instead.

---

### 🧩 When to Use Sliding Window

Use it when:
- You're scanning for **maximum/minimum/sum/count** in a **contiguous** segment.
- The problem involves **subarrays or substrings**, not arbitrary elements.
- You can **incrementally update** the state (e.g., sum, frequency map) as the window moves.
- You want to avoid recomputation and nested loops.

---

### 🔍 Example Use Cases

| Problem Type                        | Sliding Window Benefit                  |
|------------------------------------|-----------------------------------------|
| Max sum of subarray of size k      | Avoids recomputing sum for each window  |
| Longest substring without repeats  | Tracks uniqueness with dynamic window   |
| Minimum subarray with target sum   | Shrinks window to find optimal length   |
| Find anagrams in a string          | Compares frequency maps efficiently     |

---

### 🧠 Final Thought

Sliding Window is more than a trick — it's a mindset.  
It teaches you to think in terms of **stateful traversal**, where you maintain just enough information to make decisions without starting over.

Mastering it means you'll write faster, cleaner, and more scalable solutions to a wide range of problems.



---
