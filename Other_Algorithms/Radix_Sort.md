# Radix Sort — Digit-Level Sorting Engine

---

## Origin & Motivation

**Radix Sort** dates back to the **19th century** and was formalized in the **1950s**.  
Unlike **comparison-based sorts** (e.g., quicksort, mergesort), Radix Sort **leverages the structure of numbers** — sorting them **digit by digit**, from **least to most significant (LSD)** or vice versa (MSD).

It achieves **linear time complexity** under **fixed digit constraints**, making it ideal for **integers, strings, and fixed-width keys**.

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Big Data | Sorting billions of integers |
| Counting Systems | Bucket-based digit grouping |
| Competitive Programming | Non-comparison sort challenges |
| Database Indexing | Fixed-length keys |
| Game Engines | Fast leaderboard updates |

---

## When to Use Radix Sort vs Alternatives

| Scenario | Radix Sort | QuickSort | MergeSort | Counting Sort |
|--------|------------|-----------|-----------|---------------|
| Large array of integers | Yes | Yes | Yes | Yes |
| Fixed-length strings | Yes | Yes | Yes | No |
| Floating-point numbers | No | Yes | Yes | No |
| Negative numbers | Warning (extra handling) | Yes | Yes | No |
| Comparison-based sorting | No | Yes | Yes | No |
| Stable sort required | Yes | No | Yes | Yes |

---

## Core Idea

* Treat each number as a **sequence of digits**  
* Sort by each **digit position** using a **stable sort** (typically **counting sort**)  
* Start from **least significant digit (LSD)** for integers  
* Repeat for **all digit positions**  

This **avoids comparisons entirely** and relies on **digit grouping**.

---

## Implementation (C++)

```cpp
void radixSort(vector<int>& nums) {
    int maxVal = *max_element(nums.begin(), nums.end());
    int exp = 1;
    int n = nums.size();
    vector<int> buffer(n);

    while (maxVal / exp > 0) {
        vector<int> count(10, 0);

        for (int i = 0; i < n; ++i)
            count[(nums[i] / exp) % 10]++;

        for (int i = 1; i < 10; ++i)
            count[i] += count[i - 1];

        for (int i = n - 1; i >= 0; --i) {
            int digit = (nums[i] / exp) % 10;
            buffer[--count[digit]] = nums[i];
        }

        nums = buffer;
        exp *= 10;
    }
}
```

## Implementation (C#)

```csharp
public static void RadixSort(int[] nums) {
    int maxVal = nums.Max();
    int exp = 1;
    int n = nums.Length;
    int[] buffer = new int[n];

    while (maxVal / exp > 0) {
        int[] count = new int[10];

        foreach (int num in nums)
            count[(num / exp) % 10]++;

        for (int i = 1; i < 10; i++)
            count[i] += count[i - 1];

        for (int i = n - 1; i >= 0; i--) {
            int digit = (nums[i] / exp) % 10;
            buffer[--count[digit]] = nums[i];
        }

        Array.Copy(buffer, nums, n);
        exp *= 10;
    }
}
```


## Complexity Analysis

| Metric | Value |
|--------|-------|
| **Time** | O(n·k) |
| **Space** | O(n + k) |
| **Stability** | Yes (via counting sort) |
| **Comparison-free** | Yes |

> *n*: number of elements  
> *k*: number of digits (logarithmic in value range)

---

## Why It’s Fast

* **Avoids comparisons entirely**  
* **Exploits digit structure** for grouping  
* **Linear time** for bounded integers  
* **Stable and predictable performance**

---

## Impact of Design Choices

| Choice | Effect |
|--------|--------|
| **LSD vs MSD** | LSD is simpler for integers |
| **Counting sort base** | Base 10 for decimal, base 256 for bytes |
| **Negative numbers** | Require offset or separate pass |
| **Stability** | Preserves relative order |

---

## Pitfalls

* Not **general-purpose**: only works on **digit-based data**  
* Requires **stable sort** per digit  
* **Memory overhead** from buckets  
* **Handling negatives** needs care  
* Not **cache-friendly** for large bases  

---

## Conclusion

Radix Sort is a **digit-level sorting engine**:

* **Linear-time performance** on bounded integers  
* Ideal for **large datasets** with **fixed-width keys**  
* **Elegant alternative** to comparison-based sorts  
* **Stable, predictable, and memory-efficient**  

> **Key takeaway**:  
> **Radix Sort shines when sorting large arrays of integers or strings with fixed length** — especially when comparison-based sorts hit their limits.


---
