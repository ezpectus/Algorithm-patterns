# Segment Tree Beats — Advanced Range Update Engine

---

## Origin & Motivation

**Segment Tree Beats** was introduced in **2016** by **Ruyi “jiry_2” Ji** to solve problems involving **range updates with conditional constraints**, such as:

* `chmin`: set all values in a range to `min(A[i], x)`  
* `chmax`: set all values in a range to `max(A[i], x)`  
* Maintain **range sum**, **range max**, **range min**

Traditional **segment trees with lazy propagation** fail here because these operations are **non-linear** and **non-commutative**.

**Segment Tree Beats** overcomes this by:
* **Delaying updates** only when necessary  
* **Recursing deeper** only when needed  
* Using **second max** to decide when to stop

> “Beats” refers to how it **beats standard segment trees in capability**, not speed.

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Range chmin/chmax/sum queries | Yes |
| Conditional range updates | Yes |
| Game state compression | Yes |
| Competitive programming | CF 438D, CF 438E |
| Data analytics with bounded constraints | Yes |

---

## When to Use Segment Tree Beats

| Operation Type | Use Beats | Use Lazy Segment Tree |
|----------------|-----------|------------------------|
| Range add/set | No | Yes |
| Range chmin/chmax | Yes | No |
| Range sum + chmin | Yes | No |
| Conditional updates | Yes | No |
| Simple point updates | No | Yes |

---

## Core Idea

Each node stores:

| Field | Meaning |
|------|--------|
| `max1` | maximum value in segment |
| `max2` | **second maximum** |
| `count_max1` | number of elements equal to `max1` |
| `sum` | sum of segment |

**To apply `chmin(x)`:**

1. `if x >= max1` → do nothing  
2. `if x < max2` → recurse to children  
3. `else` → update `max1 = x`, adjust `sum`

This **avoids unnecessary recursion** and ensures **correctness**.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const int N = 100005;
const long long INF = 1LL << 60;

struct Node {
    long long max1, max2, sum;
    int count;
    Node() : max1(-INF), max2(-INF), sum(0), count(0) {}
};

Node tree[4 * N];
int A[N];

void build(int v, int l, int r) {
    if (l + 1 == r) {
        tree[v] = {A[l], -INF, A[l], 1};
        return;
    }
    int m = (l + r) / 2;
    build(2*v, l, m);
    build(2*v+1, m, r);
    merge(v);
}

void merge(int v) {
    Node &L = tree[2*v], &R = tree[2*v+1], &T = tree[v];
    T.sum = L.sum + R.sum;
    T.max1 = max(L.max1, R.max1);
    T.max2 = -INF;
    T.count = 0;

    if (L.max1 == T.max1) T.count += L.count;
    if (R.max1 == T.max1) T.count += R.count;

    if (L.max1 != T.max1) T.max2 = max(T.max2, L.max1);
    if (R.max1 != T.max1) T.max2 = max(T.max2, R.max1);
    T.max2 = max(T.max2, max(L.max2, R.max2));
}

void push(int v, int l, int r) {
    if (l + 1 == r) return;
    for (int u : {2*v, 2*v+1}) {
        if (tree[u].max1 > tree[v].max1) {
            tree[u].sum -= (tree[u].max1 - tree[v].max1) * tree[u].count;
            tree[u].max1 = tree[v].max1;
        }
    }
}

void chmin(int v, int l, int r, int ql, int qr, long long x) {
    if (r <= ql || qr <= l || tree[v].max1 <= x) return;
    if (ql <= l && r <= qr && tree[v].max2 < x) {
        tree[v].sum -= (tree[v].max1 - x) * tree[v].count;
        tree[v].max1 = x;
        return;
    }
    push(v, l, r);
    int m = (l + r) / 2;
    chmin(2*v, l, m, ql, qr, x);
    chmin(2*v+1, m, r, ql, qr, x);
    merge(v);
}

long long query_sum(int v, int l, int r, int ql, int qr) {
    if (r <= ql || qr <= l) return 0;
    if (ql <= l && r <= qr) return tree[v].sum;
    push(v, l, r);
    int m = (l + r) / 2;
    return query_sum(2*v, l, m, ql, qr) + query_sum(2*v+1, m, r, ql, qr);
}

long long query_max(int v, int l, int r, int ql, int qr) {
    if (r <= ql || qr <= l) return -INF;
    if (ql <= l && r <= qr) return tree[v].max1;
    push(v, l, r);
    int m = (l + r) / 2;
    return max(query_max(2*v, l, m, ql, qr), query_max(2*v+1, m, r, ql, qr));
}
```

## Implementation (C#)
```csharp
using System;

public class SegmentTreeBeats {
    const int N = 100005;
    const long INF = 1L << 60;
    
    class Node {
        public long max1 = -INF, max2 = -INF, sum = 0;
        public int count = 0;
    }

    Node[] tree = new Node[4 * N];
    int[] A = new int[N];

    public SegmentTreeBeats() {
        for (int i = 0; i < 4 * N; i++) tree[i] = new Node();
    }

    public void Build(int[] arr, int n) {
        Array.Copy(arr, A, n);
        Build(1, 0, n);
    }

    void Build(int v, int l, int r) {
        if (l + 1 == r) {
            tree[v].max1 = A[l];
            tree[v].sum = A[l];
            tree[v].count = 1;
            return;
        }
        int m = (l + r) / 2;
        Build(2*v, l, m);
        Build(2*v+1, m, r);
        Merge(v);
    }

    void Merge(int v) {
        var L = tree[2*v]; var R = tree[2*v+1]; var T = tree[v];
        T.sum = L.sum + R.sum;
        T.max1 = Math.Max(L.max1, R.max1);
        T.max2 = -INF;
        T.count = 0;

        if (L.max1 == T.max1) T.count += L.count;
        if (R.max1 == T.max1) T.count += R.count;

        if (L.max1 != T.max1) T.max2 = Math.Max(T.max2, L.max1);
        if (R.max1 != T.max1) T.max2 = Math.Max(T.max2, R.max1);
        T.max2 = Math.Max(T.max2, Math.Max(L.max2, R.max2));
    }

    void Push(int v, int l, int r) {
        if (l + 1 == r) return;
        foreach (int u in new[] {2*v, 2*v+1}) {
            if (tree[u].max1 > tree[v].max1) {
                tree[u].sum -= (tree[u].max1 - tree[v].max1) * tree[u].count;
                tree[u].max1 = tree[v].max1;
            }
        }
    }

    public void Chmin(int ql, int qr, long x, int n) {
        Chmin(1, 0, n, ql, qr, x);
    }

    void Chmin(int v, int l, int r, int ql, int qr, long x) {
        if (r <= ql || qr <= l || tree[v].max1 <= x) return;
        if (ql <= l && r <= qr && tree[v].max2 < x) {
            tree[v].sum -= (tree[v].max1 - x) * tree[v].count;
            tree[v].max1 = x;
            return;
        }
        Push(v, l, r);
        int m = (l + r) / 2;
        Chmin(2*v, l, m, ql, qr, x);
        Chmin(2*v+1, m, r, ql, qr, x);
        Merge(v);
    }

    public long QuerySum(int ql, int qr, int n) {
        return QuerySum(1, 0, n, ql, qr);
    }

    long QuerySum(int v, int l, int r, int ql, int qr) {
        if (r <= ql || qr <= l) return 0;
        if (ql <= l && r <= qr) return tree[v].sum;
        Push(v, l, r);
        int m = (l + r) / 2;
        return QuerySum(2*v, l, m, ql, qr) + QuerySum(2*v+1, m, r, ql, qr);
    }

    public long QueryMax(int ql, int qr, int n) {
        return QueryMax(1, 0, n, ql, qr);
    }

    long QueryMax(int v, int l, int r, int ql, int qr) {
        if (r <= ql || qr <= l) return -INF;
        if (ql <= l && r <= qr) return tree[v].max1;
        Push(v, l, r);
        int m = (l + r) / 2;
        return Math.Max(QueryMax(2*v, l, m, ql, qr), QueryMax(2*v+1, m, r, ql, qr));
    }
}
```
##  Complexity Analysis

| Operation           | Time Complexity             |
|---------------------|-----------------------------|
| Build               |  O(n)               |
| Range chmin/chmax   |  O(log^2 n)  amortized |
| Range sum query     |  O(\log n)              |
| Space               |  O(n)                   |

---

## Pitfalls

- Requires careful tracking of `max2` and `count`  
- `Push()` must preserve invariants  
- Not suitable for **range add/set** — use lazy segment tree instead  
- Complex to debug — test on **small cases first**


## Conclusion
Segment Tree Beats is the advanced range update engine:

- Handles conditional updates like chmin/chmax
- Maintains full segment statistics
- Amortized near-logarithmic performance
- Ideal for problems where lazy propagation fails


## Key takeaway:
- Segment Tree Beats extends the power of segment trees to non-linear, conditional updates — a must-have in your advanced data structure arsenal.


---

