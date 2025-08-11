# Two Pointers — Algorithm Pattern

> Full breakdown of the **Two Pointers** pattern — theory, examples, pseudocode, implementations, common pitfalls, complexity analysis, and memorization tips.

---

## Introduction

The **Two Pointers** pattern is a fundamental technique in algorithm design, widely used for solving problems involving arrays, strings, or linked lists.  
The idea is to use two indices (pointers) that traverse the data structure in a coordinated way, often from opposite ends or at different speeds, to achieve optimal time complexity — frequently reducing O(n²) solutions to O(n).

This approach is especially powerful when the data is **sorted** or when a **contiguous** relationship between elements is important.

---

## When to Use

* Searching for pairs or triplets that satisfy a condition (e.g., sum to a target value).
* Removing or skipping duplicates in sorted data.
* Merging two sorted lists or arrays.
* Partitioning data based on a pivot condition.
* Detecting palindromes or mirrored patterns.
* Linked list cycle detection (fast and slow pointer variant).

---

## Common Variants

1. **Opposite-direction pointers**  
   Start from both ends of the array and move towards each other — often used for sum problems or palindrome checks.

2. **Same-direction pointers (fast and slow)**  
   One pointer moves faster than the other — useful for detecting cycles, skipping duplicates, or finding middle nodes.

3. **Multiple pointers**  
   Extends the idea to three or more pointers — used for problems like "3Sum" or merging multiple sorted lists.

---

## Generic Pseudocode

### Opposite-direction

```
left = 0
right = n - 1

while left < right:
   if condition(arr[left], arr[right]):
// process pair
   left++
   right--
  else if shouldMoveLeft():
    left++
else:
   right--
```

### Fast and slow
```
slow = head
fast = head

while fast != null and fast.next != null:
    slow = slow.next
    fast = fast.next.next
if slow == fast:
   // cycle detected
     break

```

---

## Example Problems

### 1. Two Sum II — Input Array is Sorted

```csharp
public int[] TwoSumSorted(int[] numbers, int target) {
    int left = 0, right = numbers.Length - 1;
    while (left < right) {
        int sum = numbers[left] + numbers[right];
        if (sum == target) {
            return new int[] { left + 1, right + 1 }; // 1-based index
        } else if (sum < target) {
            left++;
        } else {
            right--;
        }
    }
    return new int[0];
}
```
### Complexity:
- Time: O(n) — each pointer moves at most n steps.
- Space: O(1).


## 2. Remove Duplicates from Sorted Array
```csharp
public int RemoveDuplicates(int[] nums) {
    if (nums.Length == 0) return 0;
    int slow = 0;
    for (int fast = 1; fast < nums.Length; fast++) {
        if (nums[fast] != nums[slow]) {
            slow++;
            nums[slow] = nums[fast];
        }
    }
    return slow + 1;
}
```

### Complexity:
- Time: O(n)
- Space: O(1)


## 3. Valid Palindrome (ignoring non-alphanumeric)
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


# Hybrid Uses

 - Two Pointers + Binary Search — move one pointer, binary search for the other.
- Two Pointers + Sliding Window — useful for variable-length subarrays with constraints.
- Two Pointers + HashMap — when additional lookups are needed for non-sorted data.

## Complexity

- Time: O(n) in most typical problems — each pointer usually moves in one direction only.
- Space: O(1) extra space, unless using auxiliary structures.

### Common Mistakes

- Infinite loop — forgetting to move one of the pointers in all branches.
- Wrong pointer initialization — starting in the wrong position for the given problem.
- Skipping potential answers — moving both pointers when only one should be moved.
- Not leveraging sorted order — applying two pointers to unsorted data without preprocessing.

## Memorization Tips

- If the array is sorted and you need a pair/triplet — think Two Pointers first.
- For linked lists — remember "fast/slow pointer" for cycle detection or middle finding.
- If you need to compare from both ends — Two Pointers is your go-to.

## Practice Problems

- Two Sum II — Input Array Is Sorted (LeetCode #167)
- Remove Duplicates from Sorted Array (LeetCode #26)
- Valid Palindrome (LeetCode #125)
- 3Sum (LeetCode #15)
- Linked List Cycle (LeetCode #141)

### Conclusion

Two Pointers is a core algorithmic pattern that allows you to solve a wide variety of problems with optimal time and space complexity.
By recognizing scenarios where two indices can converge or move in tandem, you can dramatically reduce runtime from quadratic to linear.



---


