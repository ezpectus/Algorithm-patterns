# HeapSort — In-Place Sorting Pyramid  
*Guaranteed O(n log n) — No Recursion, No Extra Memory*

---

## Origin & Motivation
**HeapSort** was introduced by **J. W. J. Williams** in 1964 as a sorting algorithm that:

- **Guarantees O(n log n)** in the **worst case**
- Works **in-place** — no extra memory
- Avoids recursion using **iterative heap operations**

Unlike **QuickSort**, which can degrade to **O(n²)**, **HeapSort is always stable** in complexity.  
The phrase **“Pyramid. Guarantee. No recursion.”** captures its essence:

- **Pyramid (heap)** — the core data structure
- **Guarantee** — worst-case O(n log n)
- **No recursion** — implemented with loops and array indexing

---

## Where It’s Used

| Domain | Use Case |
|-------|----------|
| **System programming** | Reliable sorting without degradation risk |
| **Embedded systems** | Limited memory → needs in-place |
| **Education** | Teaching heaps and sorting |
| **Competitive programming** | When QuickSort may be unsafe |
| **Libraries** | Fallback sort (e.g., `std::sort` in C++) |

---

## When to Use HeapSort

| Requirement | Use HeapSort | Use Other Sort |
|------------|--------------|----------------|
| Worst-case O(n log n) | Yes | QuickSort may degrade |
| In-place (no extra memory) | Yes | MergeSort needs O(n) |
| Stability (same keys keep order) | No | MergeSort/TimSort |
| Simplicity | Yes | QuickSort simpler |
| Recursive implementation | No | QuickSort/MergeSort |

---

## Core Idea
HeapSort has **two phases**:

1. **Build max-heap** from array (bottom-up, iterative)
2. **Extract max** repeatedly:
   - Swap root with last element
   - Reduce heap size
   - Restore max-heap property (**sift-down**)

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

void heapify(vector<int>& a, int n, int i) {
    int largest = i;
    int l = 2 * i + 1;
    int r = 2 * i + 2;

    if (l < n && a[l] > a[largest]) largest = l;
    if (r < n && a[r] > a[largest]) largest = r;

    if (largest != i) {
        swap(a[i], a[largest]);
        heapify(a, n, largest);
    }
}

void heapSort(vector<int>& a) {
    int n = a.size();
    
    // Build max heap
    for (int i = n / 2 - 1; i >= 0; i--)
        heapify(a, n, i);

    // Extract elements one by one
    for (int i = n - 1; i > 0; i--) {
        swap(a[0], a[i]);
        heapify(a, i, 0);
    }
}
```

## Implementation (C#)

```csharp
public class HeapSort
{
    private static void Heapify(int[] arr, int n, int i)
    {
        int largest = i;
        int left = 2 * i + 1;
        int right = 2 * i + 2;

        if (left < n && arr[left] > arr[largest]) largest = left;
        if (right < n && arr[right] > arr[largest]) largest = right;

        if (largest != i)
        {
            int temp = arr[i];
            arr[i] = arr[largest];
            arr[largest] = temp;

            Heapify(arr, n, largest);
        }
    }

    public static void Sort(int[] arr)
    {
        int n = arr.Length;

        // Build max heap
        for (int i = n / 2 - 1; i >= 0; i--)
            Heapify(arr, n, i);

        // Extract elements
        for (int i = n - 1; i > 0; i--)
        {
            int temp = arr[0];
            arr[0] = arr[i];
            arr[i] = temp;

            Heapify(arr, i, 0);
        }
    }
}
```

## Complexity Analysis

| Operation             | Time Complexity     | Explanation |
|-----------------------|---------------------|-----------|
| **Build heap**        | **O(n)**            | Bottom-up heap construction visits each level once |
| **Extract max (n times)** | **O(n log n)**  | Each extraction performs `sift-down` up to height log n |
| **Total**             | **O(n log n)**      | Guaranteed in **all cases** |
| **Space**             | **O(1)**            | **In-place** — no extra array, only swaps |

> **Note:** Building heap is **linear**, not O(n log n) — key optimization!

---

## Pitfalls

- **Not stable**  
  > Equal elements **may change order** after sorting  
  > Example: `[3a, 1, 3b]` → may become `[1, 3b, 3a]`

- **Slower than QuickSort on random data**  
  > ~2–3× slower due to poor cache locality and more swaps  
  > QuickSort: better spatial locality, fewer comparisons

- **Requires careful indexing**  
  > 0-based array vs 1-based heap logic  
  > Common bugs:  
  > - `left = 2*i + 1`, `right = 2*i + 2`  
  > - Loop bounds: `i = n/2 - 1` down to `0`

---

## Conclusion
**HeapSort** is a **reliable, pyramid-based sorting algorithm** with **ironclad guarantees**:

- **Guarantee**: **worst-case always O(n log n)** — no degradation  
- **In-place**: **zero extra memory** — ideal for constrained environments  
- **No recursion**: **fully iterative** — safe for deep arrays, no stack overflow  

**Key takeaway:**  
> **HeapSort is the go-to algorithm when you need predictable performance and minimal memory usage — even if it sacrifices average-case speed.**

It’s not the fastest, but it’s **bulletproof**.

---

## Fichka Library Entry
**Category:** `Sorting / Heap`  
**Pattern:** `In-place O(n log n) sort with heap — build + extract max`  
**Tags:** `guaranteed`, `in-place`, `no-recursion`, `stable-complexity`


---
