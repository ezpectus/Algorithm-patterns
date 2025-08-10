# Max Sum of Subarray of Size K — Sliding Window Pattern

## Problem Description
Given an integer array and a positive integer K, find the maximum possible sum of any contiguous subarray of size K.

## Example
- Input:
- arr = [2, 1, 5, 1, 3, 2], K = 3
- Output: 9

## Explanation:
- The subarray [5, 1, 3] has the maximum sum of 9.

# Constraints
 - 1 <= K <= len(arr)
- Array length can be up to 10^5
- Elements may be positive, negative, or zero

## Idea: Sliding Window
A naive approach would compute the sum for every possible subarray of size K — this takes O(N × K) time.

The Sliding Window technique optimizes this by:

- Calculating the sum of the first window of size K.
- As we move (or slide) the window forward by one position:
- Subtract the element that goes out of the window.
- Add the element that comes into the window.
- Keep track of the maximum sum found.
- This way, we avoid recomputing sums from scratch and reduce the time complexity to O(N).

## C# Implementation
```

using System;

public class Solution {
    public int MaxSumSubarray(int[] arr, int k) {
        int windowSum = 0, maxSum = int.MinValue;

        // Build initial window
        for (int i = 0; i < k; i++)
            windowSum += arr[i];
        
        maxSum = windowSum;

        // Slide the window
        for (int end = k; end < arr.Length; end++) {
            windowSum += arr[end] - arr[end - k];
            maxSum = Math.Max(maxSum, windowSum);
        }

        return maxSum;
    }
}

```
## C++ Implementation
```
#include <iostream>
#include <vector>
#include <algorithm>
using namespace std;

int maxSumSubarray(vector<int>& arr, int k) {
    int windowSum = 0, maxSum = INT_MIN;

    // Build initial window
    for (int i = 0; i < k; i++)
        windowSum += arr[i];
    
    maxSum = windowSum;

    // Slide the window
    for (int end = k; end < arr.size(); end++) {
        windowSum += arr[end] - arr[end - k];
        maxSum = max(maxSum, windowSum);
    }

    return maxSum;
}
```

## Java Implementation
```
class Solution {
    public int maxSumSubarray(int[] arr, int k) {
        int windowSum = 0, maxSum = Integer.MIN_VALUE;

        // Build initial window
        for (int i = 0; i < k; i++)
            windowSum += arr[i];
        
        maxSum = windowSum;

        // Slide the window
        for (int end = k; end < arr.length; end++) {
            windowSum += arr[end] - arr[end - k];
            maxSum = Math.max(maxSum, windowSum);
        }

        return maxSum;
    }
}
```

## Python Implementation
```
def max_sum_subarray(arr, k):
    window_sum = sum(arr[:k])
    max_sum = window_sum

    for end in range(k, len(arr)):
        window_sum += arr[end] - arr[end - k]
        max_sum = max(max_sum, window_sum)

    return max_sum

```

## Complexity Analysis
Time Complexity: O(N) — we traverse the array once, each element is added and removed from the sum exactly once.

Space Complexity: O(1) — we use only a few extra variables regardless of array size.

##Conclusion
This example demonstrates how the Sliding Window pattern turns an O(N*K) brute-force approach into an O(N) efficient solution by reusing computations.
It shows that identifying a "window" in problems involving contiguous elements can save a lot of processing time.
Mastering this pattern will help in solving a wide range of problems like maximum/minimum sums, averages, or substring problems in strings.

---
