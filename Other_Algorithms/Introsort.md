# Introsort — Hybrid Sorting King  
*O(n log n) Worst-Case — Quick + Heap + Insertion*

---

## Origin & Motivation  
**Introsort** (Introspective Sort) was created by **David Musser** in **1997** to solve **QuickSort’s fatal flaw**:  
> **O(n²) worst-case on sorted/reverse-sorted data**

**Goal**:  
- **Fast as QuickSort** in average case  
- **Guaranteed O(n log n)** worst-case  
- **No extra space** (in-place)  
- **Used in `std::sort` (C++ STL)**

**Idea**:  
> **Start with QuickSort → if recursion too deep → switch to HeapSort**

---

## Where It’s Used  

| Domain | Use |
|-------|-----|
| **C++ STL** | `std::sort`, `std::stable_sort` (fallback) |
| **Production code** | When you need **fast + safe** sort |
| **Competitive Programming** | Rare — but **reliable** |
| **Real-time systems** | No pathological cases |

---

## When to Use Introsort  

| Requirement | Introsort | QuickSort | Timsort |
|-----------|-----------|-----------|---------|
| **Worst-case O(n log n)** | Yes | No | Yes |
| **In-place** | Yes | Yes | No |
| **Fast on random** | Yes | Yes | Yes |
| **Fast on nearly sorted** | No | No | Yes |
| **Stable** | No | No | Yes |
| **Simple to implement** | No | Yes | No |

> **Use Introsort when**:  
> - You need **guaranteed performance**  
> - You **can’t risk O(n²)**  
> - You want **in-place**  
> - You don’t need **stability**

---

## Core Idea — Step by Step  

### **1. Start with QuickSort**
```text
partition → recurse left → recurse right
```
2. Track recursion depth
   ```max_depth = 2 * log₂(n)```
4. If depth > max_depth → switch to HeapSort
```heapsort(array, low, high)```

3. Final polish: Insertion Sort on small chunks
```if (size < 16) → insertion_sort```


## Full Implementation (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

class Introsort {
private:
    static const int INSERTION_THRESHOLD = 16;

    void insertionSort(vector<int>& arr, int low, int high) {
        for (int i = low + 1; i <= high; ++i) {
            int key = arr[i];
            int j = i - 1;
            while (j >= low && arr[j] > key) {
                arr[j + 1] = arr[j];
                --j;
            }
            arr[j + 1] = key;
        }
    }

    void heapify(vector<int>& arr, int n, int i, int low) {
        int largest = i;
        int left = 2 * i + 1;
        int right = 2 * i + 2;

        if (left < n && arr[low + left] > arr[low + largest])
            largest = left;
        if (right < n && arr[low + right] > arr[low + largest])
            largest = right;

        if (largest != i) {
            swap(arr[low + i], arr[low + largest]);
            heapify(arr, n, largest, low);
        }
    }

    void heapSort(vector<int>& arr, int low, int high) {
        int n = high - low + 1;
        for (int i = n / 2 - 1; i >= 0; --i)
            heapify(arr, n, i, low);
        for (int i = n - 1; i > 0; --i) {
            swap(arr[low], arr[low + i]);
            heapify(arr, i, 0, low);
        }
    }

    int partition(vector<int>& arr, int low, int high) {
        int pivot = arr[high];
        int i = low - 1;
        for (int j = low; j < high; ++j) {
            if (arr[j] <= pivot) {
                ++i;
                swap(arr[i], arr[j]);
            }
        }
        swap(arr[i + 1], arr[high]);
        return i + 1;
    }

    void quicksort(vector<int>& arr, int low, int high, int depthLimit) {
        while (low < high) {
            int size = high - low + 1;

            if (size <= INSERTION_THRESHOLD) {
                insertionSort(arr, low, high);
                return;
            }

            if (depthLimit == 0) {
                heapSort(arr, low, high);
                return;
            }

            int pi = partition(arr, low, high);
            int leftSize = pi - low;
            int rightSize = high - pi;

            if (leftSize < rightSize) {
                quicksort(arr, low, pi - 1, depthLimit - 1);
                low = pi + 1;
            } else {
                quicksort(arr, pi + 1, high, depthLimit - 1);
                high = pi - 1;
            }
        }
    }

public:
    void sort(vector<int>& arr) {
        int n = arr.size();
        if (n <= 1) return;
        int depthLimit = 2 * (int)log2(n);
        quicksort(arr, 0, n - 1, depthLimit);
    }
};

// === USAGE ===
int main() {
    vector<int> arr = {64, 34, 25, 12, 22, 11, 90, 88, 1, 100};
    Introsort is;
    is.sort(arr);
    
    cout << "Sorted: ";
    for (int x : arr) cout << x << " ";
    cout << endl;
    
    return 0;
}
```

## Full Implementation (C#)
```cpp
using System;
using System.Collections.Generic;

public class Introsort
{
    private const int INSERTION_THRESHOLD = 16;

    private void InsertionSort(List<int> arr, int low, int high)
    {
        for (int i = low + 1; i <= high; i++)
        {
            int key = arr[i];
            int j = i - 1;
            while (j >= low && arr[j] > key)
            {
                arr[j + 1] = arr[j];
                j--;
            }
            arr[j + 1] = key;
        }
    }

    private void Heapify(List<int> arr, int n, int i, int low)
    {
        int largest = i;
        int left = 2 * i + 1;
        int right = 2 * i + 2;

        if (left < n && arr[low + left] > arr[low + largest])
            largest = left;
        if (right < n && arr[low + right] > arr[low + largest])
            largest = right;

        if (largest != i)
        {
            (arr[low + i], arr[low + largest]) = (arr[low + largest], arr[low + i]);
            Heapify(arr, n, largest, low);
        }
    }

    private void HeapSort(List<int> arr, int low, int high)
    {
        int n = high - low + 1;
        for (int i = n / 2 - 1; i >= 0; i--)
            Heapify(arr, n, i, low);
        for (int i = n - 1; i > 0; i--)
        {
            (arr[low], arr[low + i]) = (arr[low + i], arr[low]);
            Heapify(arr, i, 0, low);
        }
    }

    private int Partition(List<int> arr, int low, int high)
    {
        int pivot = arr[high];
        int i = low - 1;
        for (int j = low; j < high; j++)
        {
            if (arr[j] <= pivot)
            {
                i++;
                (arr[i], arr[j]) = (arr[j], arr[i]);
            }
        }
        (arr[i + 1], arr[high]) = (arr[high], arr[i + 1]);
        return i + 1;
    }

    private void Quicksort(List<int> arr, int low, int high, int depthLimit)
    {
        while (low < high)
        {
            int size = high - low + 1;

            if (size <= INSERTION_THRESHOLD)
            {
                InsertionSort(arr, low, high);
                return;
            }

            if (depthLimit == 0)
            {
                HeapSort(arr, low, high);
                return;
            }

            int pi = Partition(arr, low, high);
            int leftSize = pi - low;
            int rightSize = high - pi;

            if (leftSize < rightSize)
            {
                Quicksort(arr, low, pi - 1, depthLimit - 1);
                low = pi + 1;
            }
            else
            {
                Quicksort(arr, pi + 1, high, depthLimit - 1);
                high = pi - 1;
            }
        }
    }

    public void Sort(List<int> arr)
    {
        int n = arr.Count;
        if (n <= 1) return;
        int depthLimit = 2 * (int)Math.Log2(n);
        Quicksort(arr, 0, n - 1, depthLimit);
    }
}

// === USAGE ===
class Program
{
    static void Main()
    {
        var arr = new List<int> { 64, 34, 25, 12, 22, 11, 90, 88, 1, 100 };
        var intro = new Introsort();
        intro.Sort(arr);

        Console.Write("Sorted: ");
        foreach (int x in arr) Console.Write(x + " ");
        Console.WriteLine();
    }
}
```

## Complexity Analysis  

| Metric | **Value** | **Explanation** |
|--------|-----------|------------------|
| **Time (Average)** | **O(n log n)** | Behaves like **QuickSort** on random data |
| **Time (Worst)** | **O(n log n)** | **HeapSort fallback** kicks in when recursion gets too deep |
| **Space** | **O(log n)** | Only **recursion stack** — in-place sorting |
| **Stable** | **No** | Does **not preserve** order of equal elements |

> **Guaranteed O(n log n)** — **no O(n²) ever**

---

## Pitfalls & Fixes  

| **Issue** | **Fix / Mitigation** |
|----------|-----------------------|
| **Recursion depth too high** | Limit to `2 * log₂(n)` — prevents stack overflow |
| **Small arrays slow** | Use **Insertion Sort** for `n < 16` — cache-friendly |
| **Bad pivot choice** | Use **last element** (simple) or **median-of-3** (better) |
| **Not stable** | Use `std::stable_sort` or **Timsort** if stability needed |

---

## Conclusion  
**Introsort is the **practical king of sorting**:**

- **Fast** like **QuickSort**  
- **Safe** like **HeapSort**  
- **In-place** — no extra memory  
- **Used in `std::sort` (C++ STL)**  

> **Real-world champion** — balances speed, safety, and simplicity.

**Key takeaway:**  
> **When you need sorting in production — use Introsort.**  
> **No O(n²). No extra space. No excuses.**

---
