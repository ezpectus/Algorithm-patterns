# Fractional Cascading

## Origin & Motivation

**Fractional Cascading** is a data structure technique introduced by Chazelle and Guibas in 1986 that eliminates redundant binary searches across a sequence of related sorted lists. The name comes from the construction: "cascade" pointers from each list to the next, and include a "fraction" (every other element) of each list into the previous one.

**The problem it solves:** Given k sorted lists L_1, ..., L_k, answer a query "find x in each L_i" efficiently. The naive approach does k independent binary searches for O(k log n) total. Fractional cascading reduces this to **O(log n + k)** — one binary search in the first list, then O(1) per subsequent list using precomputed pointers.

**The key idea:** Augment each list L_i by merging in every other element from L_{i+1} (the "cascade"). After one binary search on the augmented L_1, each subsequent position can be found in O(1) by following pointers. The "fraction" (every other element) limits the size blowup to O(1) per element across all lists.

Complexity: **O(n k)** build (n = max list size), **O(log n + k)** per query, **O(n k)** space (reduced to **O(n log n)** when applied to segment trees).

---

## Where It Is Used

- Segment trees with range queries requiring binary search at each node (e.g., Merge Sort Tree with FC achieves O(log n) instead of O(log² n))
- Computational geometry: stabbing queries, interval trees
- Multi-level range trees (2D/3D orthogonal range search)
- Compiler optimization: multi-level lookup tables
- Competitive programming: range k-th element in O(n log n + q log n)

---

## Core Construction

### Without Fractional Cascading

Query "find x in L_1, ..., L_k" requires k binary searches:
```
for i = 1 to k:
    binary_search(L_i, x)   // O(log |L_i|)
Total: O(k log n)
```

### With Fractional Cascading

**Build augmented lists M_1, ..., M_k:**

```
M_k = L_k

For i = k-1 down to 1:
    M_i = merge(L_i, every_other_element(M_{i+1}))
    
    For each element in M_i:
        left_ptr[e]  = position of e in M_i    (trivial)
        right_ptr[e] = position of e in M_{i+1} (predecessor position)
```

**Query x across all lists:**

```
pos = binary_search(M_1, x)   // One binary search: O(log n)
report position in L_1         // Check if M_1[pos] came from L_1

For i = 2 to k:
    pos = right_ptr[i-1][pos]  // Jump to M_i in O(1)
    report position in L_i     // Check if M_i[pos] came from L_i
```

---

## Why Every Other Element

Including **every** element of M_{i+1} into M_i would double the size at each level: M_1 would have O(2^k * n) elements — exponential blowup.

Including **every other** element: |M_i| ≤ |L_i| + |M_{i+1}|/2.

Let S_i = |M_i|. Then S_i ≤ n + S_{i+1}/2:
```
S_k     = n
S_{k-1} ≤ n + n/2
S_{k-2} ≤ n + n/2 + n/4
...
S_1     ≤ 2n
```

Each list has size O(n). Total space O(k * n). For a segment tree with k = O(log n): total O(n log n).

---

## Complexity Analysis

| Operation | Naive | With FC |
|---|---|---|
| Build (k lists of size n) | O(k n) | O(k n) |
| Query (find x in all k lists) | O(k log n) | O(log n + k) |
| Space | O(k n) | O(k n) |

Applied to segment tree (k = O(log n), n total elements):

| Structure | Build | Query | Space |
|---|---|---|---|
| Merge Sort Tree (naive) | O(n log n) | O(log² n) | O(n log n) |
| Merge Sort Tree + FC | O(n log n) | O(log n) | O(n log n) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// FRACTIONAL CASCADING — basic structure for k sorted lists
// ================================================================
struct FractionalCascading {
    int k; // number of lists
    struct AugElement {
        int val;
        bool from_original; // true if this element came from L_i (not cascaded)
        int right_ptr;      // index in M_{i+1} of predecessor of this element
    };

    vector<vector<AugElement>> M; // augmented lists M_1..M_k

    // Build from k sorted lists
    FractionalCascading(vector<vector<int>>& lists) : k(lists.size()), M(lists.size()) {
        // Build from right to left
        // M[k-1] = lists[k-1] with all from_original=true
        {
            int i = k - 1;
            for (int x : lists[i])
                M[i].push_back({x, true, -1}); // no next list
        }

        for (int i = k - 2; i >= 0; i--) {
            // Merge lists[i] with every other element of M[i+1]
            // Collect every-other element from M[i+1]
            vector<int> cascaded;
            for (int j = 0; j < (int)M[i+1].size(); j += 2)
                cascaded.push_back(M[i+1][j].val);

            // Merge lists[i] and cascaded into M[i]
            int p = 0, q = 0;
            while (p < (int)lists[i].size() && q < (int)cascaded.size()) {
                if (lists[i][p] <= cascaded[q])
                    M[i].push_back({lists[i][p++], true, -1});
                else
                    M[i].push_back({cascaded[q++], false, -1});
            }
            while (p < (int)lists[i].size())
                M[i].push_back({lists[i][p++], true, -1});
            while (q < (int)cascaded.size())
                M[i].push_back({cascaded[q++], false, -1});

            // Compute right_ptr: for each element e in M[i],
            // right_ptr = largest j in M[i+1] s.t. M[i+1][j].val <= e.val
            // (predecessor position in M[i+1])
            int ptr = -1;
            int mi1_size = M[i+1].size();
            // Since M[i] is sorted and every other element of M[i+1] is in M[i],
            // we can compute right_ptr with a two-pointer sweep
            int j = 0; // pointer into M[i+1]
            for (int idx = 0; idx < (int)M[i].size(); idx++) {
                // Advance j while M[i+1][j].val <= M[i][idx].val
                while (j < mi1_size && M[i+1][j].val <= M[i][idx].val)
                    j++;
                // j is now the first index in M[i+1] with val > M[i][idx].val
                // right_ptr = j - 1 (predecessor), or -1 if none
                M[i][idx].right_ptr = j - 1;
            }
        }
    }

    // Query: find predecessor of x in each of the k original lists
    // Returns for each list i: index in lists[i] of the largest element <= x
    // (-1 if no such element)
    // Returned as positions in M[i] (caller checks from_original flag)
    vector<int> query(int x) {
        vector<int> result(k, -1);

        // Binary search in M[0]
        int pos = -1;
        {
            int lo = 0, hi = (int)M[0].size() - 1;
            while (lo <= hi) {
                int mid = (lo + hi) / 2;
                if (M[0][mid].val <= x) { pos = mid; lo = mid + 1; }
                else hi = mid - 1;
            }
        }

        // Record result for list 0
        // Find rightmost position with from_original=true and val <= x
        {
            int p = pos;
            while (p >= 0 && !M[0][p].from_original) p--;
            result[0] = p; // index in M[0] (from_original positions correspond to L[0])
        }

        // Follow pointers through M[1..k-1]
        for (int i = 1; i < k; i++) {
            // pos was predecessor position in M[i-1]
            // right_ptr[i-1][pos] gives predecessor position in M[i]
            int next_pos = (pos >= 0) ? M[i-1][pos].right_ptr : -1;

            // Adjust: right_ptr gives predecessor of M[i-1][pos].val in M[i]
            // but we want predecessor of x, which may be higher
            // We need to check next_pos and next_pos+1 (at most 2 checks)
            if (next_pos + 1 < (int)M[i].size() && M[i][next_pos+1].val <= x)
                next_pos++;
            pos = next_pos;

            // Record result for list i
            int p = pos;
            while (p >= 0 && !M[i][p].from_original) p--;
            result[i] = p;
        }
        return result;
    }
};

// ================================================================
// SEGMENT TREE WITH FRACTIONAL CASCADING
// Each node stores sorted array of its range elements.
// FC reduces count_leq query from O(log² n) to O(log n).
// ================================================================
struct MergeSortTreeFC {
    int n;
    vector<vector<int>> sorted;    // sorted[v] = sorted elements at node v
    // FC pointers: for each node v and position i in sorted[v],
    // left_fc[v][i]  = predecessor position in sorted[2v]
    // right_fc[v][i] = predecessor position in sorted[2v+1]
    vector<vector<int>> left_fc, right_fc;

    MergeSortTreeFC(vector<int>& A) : n(A.size()),
        sorted(4*n), left_fc(4*n), right_fc(4*n)
    {
        build(A, 1, 0, n-1);
    }

    void build(vector<int>& A, int v, int l, int r) {
        if (l == r) {
            sorted[v] = {A[l]};
            left_fc[v]  = {-1};
            right_fc[v] = {-1};
            return;
        }
        int mid = (l + r) / 2;
        build(A, 2*v,   l,   mid);
        build(A, 2*v+1, mid+1, r);

        // Merge children
        merge(sorted[2*v].begin(),   sorted[2*v].end(),
              sorted[2*v+1].begin(), sorted[2*v+1].end(),
              back_inserter(sorted[v]));

        int sz = sorted[v].size();
        left_fc[v].resize(sz);
        right_fc[v].resize(sz);

        // Two-pointer: for each position in sorted[v], find predecessor
        // position in sorted[2v] and sorted[2v+1]
        {
            int j = 0;
            for (int i = 0; i < sz; i++) {
                while (j < (int)sorted[2*v].size() && sorted[2*v][j] <= sorted[v][i])
                    j++;
                left_fc[v][i] = j - 1; // predecessor in left child
            }
        }
        {
            int j = 0;
            for (int i = 0; i < sz; i++) {
                while (j < (int)sorted[2*v+1].size() && sorted[2*v+1][j] <= sorted[v][i])
                    j++;
                right_fc[v][i] = j - 1;
            }
        }
    }

    // Count elements <= x in A[ql..qr] using FC: O(log n)
    // pos_in_v = current position (predecessor of x) in sorted[v]
    int query_leq(int v, int l, int r, int ql, int qr, int x, int pos_v) const {
        if (qr < l || r < ql) return 0;
        if (ql <= l && r <= qr) {
            // pos_v is the predecessor position of x in sorted[v]
            return pos_v + 1; // count of elements <= x
        }

        int mid = (l + r) / 2;
        int ans = 0;

        // Propagate FC pointers to children
        // left child position: left_fc[v][pos_v] (if pos_v >= 0)
        int left_pos  = (pos_v >= 0) ? left_fc[v][pos_v]  : -1;
        int right_pos = (pos_v >= 0) ? right_fc[v][pos_v] : -1;

        // Adjust: check if the next position also qualifies
        // (Two adjacent elements in a child may both be ≤ x)
        if (left_pos + 1 < (int)sorted[2*v].size() &&
            sorted[2*v][left_pos+1] <= x) left_pos++;
        if (right_pos + 1 < (int)sorted[2*v+1].size() &&
            sorted[2*v+1][right_pos+1] <= x) right_pos++;

        ans += query_leq(2*v,   l,   mid, ql, qr, x, left_pos);
        ans += query_leq(2*v+1, mid+1, r, ql, qr, x, right_pos);
        return ans;
    }

    // Public interface: count elements in A[ql..qr] that are <= x
    int count_leq(int ql, int qr, int x) const {
        if (sorted[1].empty()) return 0;
        // Initial binary search in root
        int pos = (int)(upper_bound(sorted[1].begin(), sorted[1].end(), x)
                        - sorted[1].begin()) - 1;
        return query_leq(1, 0, n-1, ql, qr, x, pos);
    }

    // Count elements in A[ql..qr] with value in [a,b]
    int count_range(int ql, int qr, int a, int b) const {
        return count_leq(ql, qr, b) - count_leq(ql, qr, a-1);
    }

    // K-th smallest in A[ql..qr]
    int kth_smallest(int ql, int qr, int k) const {
        int lo = sorted[1].front(), hi = sorted[1].back();
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (count_leq(ql, qr, mid) >= k) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // --- Basic Fractional Cascading ---
    {
        printf("=== Fractional Cascading — Basic ===\n");
        // 4 sorted lists
        vector<vector<int>> lists = {
            {1, 3, 5, 7, 9},
            {2, 4, 6, 8, 10},
            {1, 4, 7, 10, 13},
            {3, 6, 9, 12, 15}
        };

        FractionalCascading fc(lists);

        // Query: find predecessor of 6 in each list
        int x = 6;
        printf("Predecessor of %d in each list:\n", x);
        auto result = fc.query(x);
        for (int i = 0; i < (int)lists.size(); i++) {
            int pos = result[i]; // position in M[i] with from_original
            if (pos < 0) printf("  L[%d]: none\n", i);
            else         printf("  L[%d]: val=%d\n", i, fc.M[i][pos].val);
        }
        // L[0]: 5, L[1]: 6, L[2]: 4, L[3]: 6
    }

    // --- Merge Sort Tree with FC ---
    {
        printf("\n=== Merge Sort Tree + FC ===\n");
        vector<int> A = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};
        MergeSortTreeFC mst(A);

        printf("A = {3,1,4,1,5,9,2,6,5,3}\n");
        printf("count_leq([0,9], 5) = %d\n", mst.count_leq(0,9,5));   // 7
        printf("count_leq([2,7], 4) = %d\n", mst.count_leq(2,7,4));   // 3
        printf("count_range([0,9],[2,5]) = %d\n", mst.count_range(0,9,2,5)); // 6
        printf("kth([0,9], 5) = %d\n", mst.kth_smallest(0,9,5));      // 3
        printf("kth([2,7], 2) = %d\n", mst.kth_smallest(2,7,2));      // 2
    }

    // --- Performance comparison ---
    {
        printf("\n=== FC vs Naive comparison ===\n");
        int n = 1000;
        vector<int> A(n);
        for (int i = 0; i < n; i++) A[i] = rand() % 1000;

        MergeSortTreeFC mst(A);

        // Verify against brute force
        int errors = 0;
        for (int q = 0; q < 100; q++) {
            int l = rand() % n, r = rand() % n;
            if (l > r) swap(l, r);
            int x = rand() % 1000;

            int fc_ans = mst.count_leq(l, r, x);
            int brute  = 0;
            for (int i = l; i <= r; i++) if (A[i] <= x) brute++;
            if (fc_ans != brute) errors++;
        }
        printf("Verification: %s (%d errors in 100 random tests)\n",
               errors == 0 ? "OK" : "FAIL", errors);
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class MergeSortTreeFC {
    private readonly int n;
    private readonly List<int>[] sorted;
    private readonly List<int>[] leftFC, rightFC;

    public MergeSortTreeFC(int[] A) {
        n = A.Length;
        sorted  = new List<int>[4*n];
        leftFC  = new List<int>[4*n];
        rightFC = new List<int>[4*n];
        for (int i = 0; i < 4*n; i++) {
            sorted[i]  = new(); leftFC[i]  = new(); rightFC[i] = new();
        }
        Build(A, 1, 0, n-1);
    }

    private void Build(int[] A, int v, int l, int r) {
        if (l == r) {
            sorted[v].Add(A[l]);
            leftFC[v].Add(-1); rightFC[v].Add(-1);
            return;
        }
        int mid = (l+r)/2;
        Build(A, 2*v, l, mid); Build(A, 2*v+1, mid+1, r);

        // Merge
        int i=0, j=0;
        var L=sorted[2*v]; var R=sorted[2*v+1];
        while(i<L.Count&&j<R.Count) sorted[v].Add(L[i]<=R[j]?L[i++]:R[j++]);
        while(i<L.Count) sorted[v].Add(L[i++]);
        while(j<R.Count) sorted[v].Add(R[j++]);

        int sz=sorted[v].Count;

        // FC pointers: two-pointer sweep
        {
            int p=0;
            for(int idx=0;idx<sz;idx++){
                while(p<L.Count&&L[p]<=sorted[v][idx]) p++;
                leftFC[v].Add(p-1);
            }
        }
        {
            int p=0;
            for(int idx=0;idx<sz;idx++){
                while(p<R.Count&&R[p]<=sorted[v][idx]) p++;
                rightFC[v].Add(p-1);
            }
        }
    }

    private int QueryLeq(int v, int l, int r, int ql, int qr, int x, int posV) {
        if(qr<l||r<ql) return 0;
        if(ql<=l&&r<=qr) return posV+1;

        int mid=(l+r)/2;
        int lp = posV>=0 ? leftFC[v][posV]  : -1;
        int rp = posV>=0 ? rightFC[v][posV] : -1;

        if(lp+1<sorted[2*v].Count  &&sorted[2*v][lp+1]  <=x) lp++;
        if(rp+1<sorted[2*v+1].Count&&sorted[2*v+1][rp+1]<=x) rp++;

        return QueryLeq(2*v,l,mid,ql,qr,x,lp)
             + QueryLeq(2*v+1,mid+1,r,ql,qr,x,rp);
    }

    public int CountLeq(int ql, int qr, int x) {
        if(sorted[1].Count==0) return 0;
        int pos = sorted[1].BinarySearch(x);
        if(pos<0) pos=~pos-1; else while(pos+1<sorted[1].Count&&sorted[1][pos+1]==x) pos++;
        return QueryLeq(1,0,n-1,ql,qr,x,pos);
    }

    public int CountRange(int ql, int qr, int a, int b) =>
        CountLeq(ql,qr,b)-CountLeq(ql,qr,a-1);

    public int KthSmallest(int ql, int qr, int k) {
        int lo=sorted[1][0], hi=sorted[1][^1];
        while(lo<hi){int mid=lo+(hi-lo)/2;if(CountLeq(ql,qr,mid)>=k)hi=mid;else lo=mid+1;}
        return lo;
    }

    public static void Main() {
        int[] A={3,1,4,1,5,9,2,6,5,3};
        var mst=new MergeSortTreeFC(A);
        Console.WriteLine($"count_leq([0,9],5)={mst.CountLeq(0,9,5)}");    // 7
        Console.WriteLine($"count_leq([2,7],4)={mst.CountLeq(2,7,4)}");    // 3
        Console.WriteLine($"count_range([0,9],[2,5])={mst.CountRange(0,9,2,5)}"); // 6
        Console.WriteLine($"kth([0,9],5)={mst.KthSmallest(0,9,5)}");       // 3
    }
}
```

---

## FC Pointer Correctness — Two-Element Check

The FC pointer `right_ptr[v][pos]` gives the predecessor position in the child for the value `sorted[v][pos]`. When querying for x (which may be larger than `sorted[v][pos]`), the child's predecessor of x could be one position further:

```
sorted[v][pos] = some value s ≤ x
right_ptr gives predecessor of s in child, not predecessor of x

Since x ≥ s, the predecessor of x in child is:
  right_ptr[v][pos]       if child[right_ptr+1] > x
  right_ptr[v][pos] + 1   if child[right_ptr+1] ≤ x

The difference is at most 1 because:
  Every other element of child was merged into parent.
  So between consecutive elements of parent sourced from the child,
  there is at most one "missed" child element.

This is the critical correctness property of fractional cascading:
the adjustment is always at most 1 step.
```

---

## Pitfalls

- **FC pointer gives predecessor of sorted[v][pos], not of x** — after propagating `right_ptr[v][pos]` to a child, the result is the predecessor of the value `sorted[v][pos]`, which may be smaller than x. Always check whether `child[ptr+1] ≤ x` and advance by 1 if so. This ±1 adjustment is always sufficient because every other child element is in the parent.
- **Initial binary search must be in the root** — fractional cascading requires exactly one binary search at the entry point (the root or the first list). All subsequent positions are derived in O(1). Performing binary searches at intermediate nodes defeats the purpose and restores the O(log² n) complexity.
- **Empty range query: pos = -1 is valid** — when no element ≤ x exists, the predecessor position is -1. Both `left_fc[v][-1]` and `right_fc[v][-1]` are undefined. Guard all FC pointer accesses with `if (pos >= 0)`, using -1 to mean "zero elements ≤ x in this subtree."
- **FC build must use two-pointer, not binary search** — computing `left_fc[v][i]` for all i requires a sweep: both `sorted[v]` and `sorted[2v]` are sorted, so a two-pointer in O(sz) total is correct. Using binary search for each i gives O(sz log sz) per node — summing to O(n log² n) total build time instead of O(n log n).
- **FC requires the child arrays to be a subset of the parent** — the FC invariant holds only when the parent's sorted array is the merge of its children's sorted arrays. If any non-standard merge or filtering is applied, the ±1 guarantee breaks and the propagation is wrong.
- **BinarySearch in C# returns bitwise complement for misses** — `List<int>.BinarySearch(x)` returns the bitwise complement `~insertionPoint` when x is not found. The predecessor index is `~result - 1` (not `~result`). Always verify the initial position satisfies `sorted[1][pos] ≤ x` and `sorted[1][pos+1] > x` before starting the FC traversal.

---

## Complexity Summary

| Structure | Build | Query | Space |
|---|---|---|---|
| k lists, naive | O(kn) | O(k log n) | O(kn) |
| k lists, FC | O(kn) | O(log n + k) | O(kn) |
| Seg tree, naive | O(n log n) | O(log² n) | O(n log n) |
| Seg tree + FC | O(n log n) | O(log n) | O(n log n) |

---

## Conclusion

Fractional Cascading is the **canonical technique for eliminating redundant binary searches across related sorted lists**:

- By merging every other element of L_{i+1} into L_i and precomputing predecessor pointers, subsequent searches after the first cost O(1) each — total O(log n + k) for k lists.
- Applied to a segment tree, it reduces Merge Sort Tree queries from O(log² n) to O(log n) without changing build time or space (both remain O(n log n)).
- The ±1 adjustment at each level is the essential correctness detail: FC pointers track the predecessor of `sorted[v][pos]`, not of the query value x, requiring at most one additional comparison per level.
- In competitive programming, the most common application is the segment tree with sorted arrays — FC gives O(log n) count queries, matching the performance of wavelet trees and persistent segment trees with simpler pointer structure.

**Key takeaway:**  
FC is a two-pass construction: build sorted arrays via merge, then sweep two pointers to compute left/right FC pointers in O(n log n) total. The query is one binary search at the root, then O(1) pointer propagation per level with a single ±1 check. The complexity gain (log² n → log n) is significant for n up to 10^6, where log²(10^6) ≈ 400 vs log(10^6) ≈ 20.
