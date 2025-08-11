# Two Pointers Pattern - C# Guide

This guide covers **4 classic problems** that can be solved using the **Two Pointers** technique.
We will explain each problem, show its C# solution, and analyze the time and space complexity.

---

## 1. Two Sum II - Input Array Is Sorted

**Problem:**  
Given a 1-indexed array of integers `numbers` that is already sorted in non-decreasing order,
find two numbers such that they add up to a specific target number.  
Return the indices of the two numbers, added by one as required.

**Approach (Two Pointers, Detailed):**
A brute-force approach would be to check all possible pairs of numbers,  
which would require **O(n²)** time — too slow for large inputs.

With Two Pointers:
- Initialize `left` at the beginning of the array and `right` at the end.
- Since the array is sorted, we can use the sum comparison to decide how to move pointers:
    - If `numbers[left] + numbers[right]` equals the target → return their indices.
    - If the sum is less than the target → increment `left` (to increase the sum).
    - If the sum is greater than the target → decrement `right` (to decrease the sum).
- Continue until the correct pair is found.

Why this works:
- Sorting property guarantees that moving `left` increases the sum and moving `right` decreases it.
- We avoid nested loops and check each element at most once.
- This makes the time complexity **O(n)** and space complexity **O(1)**,  
  which is optimal for this problem.


**C# Code:**

```csharp
public int[] TwoSum(int[] numbers, int target) {
    int left = 0, right = numbers.Length - 1;
    while (left < right) {
        int sum = numbers[left] + numbers[right];
        if (sum == target) {
            return new int[] { left + 1, right + 1 };
        } else if (sum < target) {
            left++;
        } else {
            right--;
        }
    }
    return new int[0];
}
```
## Complexity:

- Time: O(n)
- Space: O(1)

## 2. Container With Most Water
**Problem:**
You are given n non-negative integers height, where each represents a point at coordinate (i, height[i]).
Find two lines that together with the x-axis form a container, such that the container holds the most water.

**Approach (Two Pointers, Detailed):**
The naive solution would be to check all possible pairs of lines and compute the area —  
this would take O(n²) time, which is inefficient for large arrays.

With Two Pointers:
- Place one pointer at the beginning (`left`) and one at the end (`right`).
- Calculate the area formed by the lines at these pointers.
- To maximize the area, we always move the pointer pointing to the **shorter line** inward.
  This is because moving the taller line inward cannot increase the height,
  and it reduces the width — so it will never produce a better result.
- Continue until `left` meets `right`.

This approach guarantees that we explore all possible *maximum area candidates*
in O(n) time without redundant checks.


C# Code:

```csharp

public int MaxArea(int[] height) {
    int left = 0, right = height.Length - 1, maxArea = 0;
    while (left < right) {
        int area = Math.Min(height[left], height[right]) * (right - left);
        maxArea = Math.Max(maxArea, area);
        if (height[left] < height[right]) {
            left++;
        } else {
            right--;
        }
    }
    return maxArea;
}
```
### Complexity:

- Time: O(n)
- Space: O(1)

## 3. Valid Palindrome
**Problem:**
 Given a string s, determine if it is a palindrome, considering only alphanumeric characters and ignoring cases.

**Approach (Two Pointers, Detailed):**
A naive approach would be to create a filtered string with only lowercase alphanumeric characters
and then check if it reads the same forward and backward.  
While this works, it requires extra memory proportional to the string length (O(n) space).

With Two Pointers:
- Use two indices: `left` starting at the beginning, `right` starting at the end.
- Skip all characters that are not letters or digits (e.g., punctuation, spaces).
- Compare the characters at `left` and `right` in lowercase form.
- If they ever differ, return `false`.
- Move both pointers toward the center and continue until they meet.

Why this works:
- We avoid creating a new filtered string, checking in-place instead.
- This reduces extra space to **O(1)** and processes the string in a single pass — **O(n)** time.
- The method works even for very large strings because it doesn’t rely on extra copies of data.

### C# Code:

```csharp

public bool IsPalindrome(string s) {
    int left = 0, right = s.Length - 1;
    while (left < right) {
        while (left < right && !char.IsLetterOrDigit(s[left])) left++;
        while (left < right && !char.IsLetterOrDigit(s[right])) right--;
        if (char.ToLower(s[left]) != char.ToLower(s[right])) return false;
        left++;
        right--;
    }
    return true;
}
```
### Complexity:

- Time: O(n)
- Space: O(1)

## 4. 3Sum
**Problem:**
  Given an integer array nums, return all the triplets [nums[i], nums[j], nums[k]] such that i != j, i != k, and j != k, and nums[i] + nums[j] + nums[k] == 0.

**Approach (Two Pointers, Detailed):**
The brute-force method would be to try all triplets using three nested loops,
which takes O(n³) time — far too slow for large arrays.

Optimized strategy with Two Pointers:
1. **Sort the array** — this allows us to use the two-pointer method effectively.
2. Fix one number at position `i` (outer loop).
3. Use two pointers (`left = i+1` and `right = n-1`) to find the remaining two numbers
   such that their sum equals `-nums[i]`.
4. If the sum is too small, move `left` forward; if it's too large, move `right` backward.
5. Skip duplicates for both `i`, `left`, and `right` to avoid repeating triplets.

Why this works:
- Sorting enables ordered traversal and efficient sum adjustments.
- Two Pointers reduce the inner search from O(n²) per fixed element to O(n),
  giving an overall O(n²) solution.
- This is much faster than checking all triplets without ordering.


### C# Code:

```csharp
public IList<IList<int>> ThreeSum(int[] nums) {
    Array.Sort(nums);
    var res = new List<IList<int>>();
    for (int i = 0; i < nums.Length - 2; i++) {
        if (i > 0 && nums[i] == nums[i - 1]) continue;
        int left = i + 1, right = nums.Length - 1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                res.Add(new List<int> { nums[i], nums[left], nums[right] });
                while (left < right && nums[left] == nums[left + 1]) left++;
                while (left < right && nums[right] == nums[right - 1]) right--;
                left++;
                right--;
            } else if (sum < 0) {
                left++;
            } else {
                right--;
            }
        }
    }
    return res;
}
```
### Complexity:

- Time: O(n^2)
- Space: O(1) (excluding output storage)

## Conclusion
The **Two Pointers** technique is a powerful optimization that replaces nested loops
with a single pass from both ends toward the center. It shines in problems where:

- The data is sorted or can be sorted without breaking the problem constraints.
- You need to find pairs, triplets, or subsequences with certain properties.
- Traversal from both ends can eliminate large portions of the search space quickly.

**Advantages:**
- Significant time complexity reduction (often from O(n²) to O(n) or O(n²) instead of O(n³)).
- Constant extra space usage in most cases.
- Simple to implement once the logic is understood.

**Limitations:**
- Requires ordered data or constraints that allow sorting.
- Not suitable if the array must remain in its original order for the solution.
- May not help if all elements need to be checked without a relational pattern.

When used appropriately, Two Pointers can turn an otherwise slow brute-force approach
into an elegant, efficient solution that scales well for large inputs.





---
