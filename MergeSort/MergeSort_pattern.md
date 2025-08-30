# 🧠 Merge Sort — Algorithm Template

## 📘 Introduction

Merge Sort is a classic **divide-and-conquer** sorting algorithm that recursively splits an array into halves, sorts each half, and merges them back together in sorted order.

It guarantees **O(n log n)** time complexity and is **stable**, making it ideal for:

- Sorting **linked lists**
- Handling **large datasets**
- Avoiding worst-case behavior of quicksort

---

## 🔍 Core Idea

- **Divide** the array into two halves  
- **Conquer** each half recursively  
- **Merge** the sorted halves into a single sorted array

---

## 🧱 When to Apply Merge Sort

Use Merge Sort when:

- ✅ You need guaranteed **O(n log n)** performance  
- ✅ Stability matters (e.g., preserving relative order of equal elements)  
- ✅ You're sorting **linked lists** or large arrays  
- ✅ You want to avoid worst-case behavior of quicksort (e.g., already sorted input)

---
# 🧩 Merge Sort — Pseudocode (Annotated)

```text
function mergeSort(arr):
    if length of arr <= 1:
        return arr
    # Base case: arrays of size 0 or 1 are already sorted

    mid = length of arr / 2
    # Divide: find the midpoint to split the array

    left = mergeSort(arr[0:mid])
    right = mergeSort(arr[mid:end])
    # Recursively sort both halves

    return merge(left, right)
    # Conquer: merge the sorted halves
```
```text
function merge(left, right):
    result = []
    # Initialize result container

    while left and right:
        if left[0] <= right[0]:
            result.append(left.pop(0))
        else:
            result.append(right.pop(0))
        # Compare front elements and append the smaller one
        # This preserves stability (equal elements retain order)

    append remaining elements of left and right to result
    # One side may still have elements — flush them into result

    return result
```


## 🧬 Merge Sort — C# Implementation (With Inline Commentary)
```csharp
public int[] MergeSort(int[] arr) {
    if (arr.Length <= 1) return arr;
    // Base case: single-element arrays are already sorted

    int mid = arr.Length / 2;
    // Divide: find midpoint

    int[] left = MergeSort(arr.Take(mid).ToArray());
    int[] right = MergeSort(arr.Skip(mid).ToArray());
    // Recursively sort left and right halves

    return Merge(left, right);
    // Merge the sorted halves
}


private int[] Merge(int[] left, int[] right) {
    List<int> result = new List<int>();
    // Use a dynamic list to accumulate sorted elements

    int i = 0, j = 0;
    // Two pointers to traverse left and right arrays

    while (i < left.Length && j < right.Length) {
        if (left[i] <= right[j]) {
            result.Add(left[i++]);
        } else {
            result.Add(right[j++]);
        }
        // Compare current elements and add the smaller one
        // Advance the pointer of the chosen side
    }

    while (i < left.Length) result.Add(left[i++]);
    while (j < right.Length) result.Add(right[j++]);
    // Flush remaining elements from either side

    return result.ToArray();
    // Convert result list back to array
}
```

# 🧠 Engineering Signals

| **Signal**              | **Explanation** |
|------------------------|-----------------|
| 🔁 **Recursion Depth**  | Controlled by array size. Each recursive call halves the input, ensuring logarithmic depth. Base case (`length <= 1`) guarantees termination. |
| ⚖️ **Merge Stability**  | Using `<=` ensures that equal elements retain their original relative order — critical for stable sorting in problems where stability matters. |
| 🧠 **Memory Tradeoff**  | `ToArray()` and `List<int>` simplify implementation but introduce extra memory usage. For large datasets, consider in-place merging or switch to linked list variant for better space efficiency. |
| 🔗 **Linked List Variant** | Arrays require slicing and copying; linked lists allow splitting via slow/fast pointers and merging via pointer rewiring — no shifting or copying needed. |
| 🔍 **Inversion Counting** | During merge, every time a right element is placed before a left one, it "skips" over remaining left elements. Each such skip is an inversion — useful for problems like **Leetcode 493: Reverse Pairs**. |

---

# 🧠 Merge Sort on Linked Lists

Merge Sort is especially efficient for **linked lists** due to their pointer-based structure and lack of random access.

## ✅ Why It Works Well

- **Splitting**: Use **slow/fast pointers** to find the midpoint in O(n) time.
- **Merging**: No need to shift elements — just rewire `next` pointers.
- **Space Efficiency**: Can be done **in-place** with O(1) extra space, unlike array-based merge which requires auxiliary arrays.

## 🧩 Use Cases

| **Problem**                        | **Why Merge Sort Fits** |
|-----------------------------------|--------------------------|
| Sort List (Leetcode 148)          | Linked list + stable sort + O(n log n) |
| Merge K Sorted Lists (Leetcode 23)| Merge logic is reusable; divide-and-conquer variant scales well |

---

# 📊 Complexity Analysis

| **Metric**         | **Value**     | **Reasoning** |
|--------------------|---------------|----------------|
| ⏱️ Time (array)     | O(n log n)    | Each recursive level splits the array; merging takes linear time |
| 🧠 Space (array)    | O(n)          | Requires auxiliary arrays during merge |
| 🔗 Space (list)     | O(1)          | Linked list variant can be done in-place with pointer manipulation |

---

# ❌ Common Mistakes in Merge Sort

| ❌ Mistake                    | ⚠️ Consequence                     |
|------------------------------|-----------------------------------|
| Forgetting base case         | Infinite recursion                |
| Incorrect merge logic        | Duplicates or missing elements    |
| Not handling edge cases      | Crashes or incorrect results      |
| Inefficient array slicing    | Extra memory usage                |
| Mixing up indices during merge | Off-by-one errors, unstable sort |

---

# 🧠 Memorization Tips

- **Merge Sort = Divide + Conquer + Merge**
- Always check base case: `length <= 1`
- Use **two pointers** to merge efficiently
- For **linked lists**, use **slow/fast pointer** to split
- Merge is where **stability** is preserved — compare `<=`, not `<`

---

# 🧩 Merge Sort Training Progression

| ✅ Stage                          | 🔍 Focus                                      |
|----------------------------------|-----------------------------------------------|
| Merge Sort on arrays             | Basic recursion + merge logic                 |
| Merge Sort on linked lists       | Pointer manipulation + in-place merge         |
| Merge Sort with custom comparators | Sorting by key, object fields               |
| Merge Sort with inversion counting | Count how many right elements skip left     |
| Merge Sort with index tracking   | Track original positions (e.g., count smaller after self) |

---

# 🧠 Denis-style Insight

Merge Sort isn’t just a sorting algorithm — it’s a **recursive layout engine**.  
It teaches you how to:

- 🔁 Control recursion depth with architectural clarity  
- 🧭 Manage pointer-based merging with precision  
- ⚖️ Preserve stability and order across recursive layers  
- 📊 Benchmark divide-and-conquer strategies with predictable complexity

Once internalized, Merge Sort becomes your go-to for:

- ✅ Sorting with guarantees  
- ✅ Linked list manipulation  
- ✅ Structural recursion templates  
- ✅ Inversion-based counting problems  
- ✅ Index-aware transformations

---

# ✅ Conclusion

Merge Sort is a foundational algorithm that blends:

- **Recursion** → for structural decomposition  
- **Pointer control** → for efficient merging  
- **Architectural clarity** → for predictable performance

Mastering Merge Sort means mastering:

- Stable sorting  
- Divide-and-conquer intuition  
- Recursive engineering patterns

It’s not just about sorting — it’s about **how you split**, **how you merge**, and **how you control structure**.



---
