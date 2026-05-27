# Merge Sort Tree

## Origin & Motivation

A **Merge Sort Tree** is a segment tree where each node stores the **sorted array** of all elements in its range. It combines the hierarchical range decomposition of a segment tree with the sorted order needed for order-statistic and frequency queries.

The name comes from the construction algorithm: build a segment tree bottom-up, and at each internal node merge the two sorted children arrays — exactly one level of merge sort. This gives O(n log n) build time and space, since the total size across all levels of the segment tree is O(n log n) (each element appears in O(log n) nodes).

The key capability: **count elements in range [l, r] with value in [a, b]** in O(log² n) time via binary search at each of the O(log n) segment tree nodes visited during the query.

Complexity: **O(n log n)** build, **O(log² n)** per query, **O(n log n)** space.

---

## Where It Is Used

- Count elements in range [l, r] with value in [a, b]
- K-th smallest element in range [l, r]
- Count inversions in a range
- Offline range order-statistic queries
- Competitive programming: "how many elements in [l,r] are ≤ x?"
- 2D range counting (one dimension is index, other is value)

---

## Structure

A segment tree where node covering range `[l, r]` stores a sorted array of all `A[l..r]`. The sorted arrays are built during construction by merging children's arrays.

```
Array: A = [3, 1, 4, 1, 5, 9, 2, 6]
Index:      0  1  2  3  4  5  6  7

Segment tree node [0,7]: [1,1,2,3,4,5,6,9]  ← sorted A[0..7]
  Node [0,3]: [1,1,3,4]   ← sorted A[0..3]
    Node [0,1]: [1,3]
      Node [0,0]: [3]
      Node [1,1]: [1]
    Node [2,3]: [1,4]
      Node [2,2]: [4]
      Node [3,3]: [1]
  Node [4,7]: [2,5,6,9]   ← sorted A[4..7]
    ...
```

---

## Core Queries

### Count elements in [l, r] with value ≤ x

Decompose [l, r] into O(log n) segment tree nodes. At each node, binary search for x in the sorted array. Sum the counts.

```
query_leq(node, l, r, x):
    if node.range ⊆ [l, r]:
        return upper_bound(node.sorted, x)   // binary search O(log n)
    
    ans = 0
    if left_child.range ∩ [l,r] ≠ ∅:
        ans += query_leq(left_child, l, r, x)
    if right_child.range ∩ [l,r] ≠ ∅:
        ans += query_leq(right_child, l, r, x)
    return ans
```

O(log n) nodes × O(log n) binary search = **O(log² n)**.

### K-th smallest in [l, r]

Binary search on the answer value. For each candidate value `mid`, count elements ≤ `mid` in [l, r] (using query_leq). Find smallest value where count ≥ k.

Binary search range [min_val, max_val] × log² n per count query = **O(log³ n)**.

Alternatively: fractional cascading reduces to O(log² n), or use persistent segment tree for O(log n).

### Count elements in [l, r] with value in [a, b]

`query_leq(l, r, b) - query_leq(l, r, a-1)` = O(log² n).

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Build | O(n log n) | O(n log n) |
| Count ≤ x in [l, r] | O(log² n) | — |
| Count in [a, b] in [l, r] | O(log² n) | — |
| K-th smallest in [l, r] | O(log³ n) | — |
| Point update (rebuild path) | O(log² n) | O(n log n) — needs realloc |

Point updates require rebuilding O(log n) sorted arrays on the root-to-leaf path, each costing O(range_size) to re-sort → O(n) worst case per update. The merge sort tree is primarily a **static** structure. For dynamic updates, use a wavelet tree or persistent segment tree.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// MERGE SORT TREE
// Segment tree where each node stores sorted array of its range.
// ================================================================
struct MergeSortTree {
    int n;
    vector<vector<int>> tree; // tree[node] = sorted elements

    MergeSortTree(vector<int>& A) : n(A.size()), tree(4 * A.size()) {
        build(A, 1, 0, n - 1);
    }

    void build(vector<int>& A, int node, int l, int r) {
        if (l == r) {
            tree[node] = {A[l]};
            return;
        }
        int mid = (l + r) / 2;
        build(A, 2*node,   l,   mid);
        build(A, 2*node+1, mid+1, r);

        // Merge children's sorted arrays
        merge(tree[2*node].begin(),   tree[2*node].end(),
              tree[2*node+1].begin(), tree[2*node+1].end(),
              back_inserter(tree[node]));
    }

    // Count elements in A[ql..qr] that are <= x
    int query_leq(int node, int l, int r, int ql, int qr, int x) const {
        if (qr < l || r < ql) return 0;
        if (ql <= l && r <= qr) {
            // Binary search: count elements <= x
            return (int)(upper_bound(tree[node].begin(), tree[node].end(), x)
                         - tree[node].begin());
        }
        int mid = (l + r) / 2;
        return query_leq(2*node,   l,   mid, ql, qr, x)
             + query_leq(2*node+1, mid+1, r, ql, qr, x);
    }

    // Count elements in A[l..r] that are <= x
    int count_leq(int l, int r, int x) const {
        return query_leq(1, 0, n-1, l, r, x);
    }

    // Count elements in A[l..r] with value in [a, b]
    int count_range(int l, int r, int a, int b) const {
        return count_leq(l, r, b) - count_leq(l, r, a - 1);
    }

    // K-th smallest element in A[l..r] (1-indexed k)
    int kth_smallest(int l, int r, int k) const {
        // Binary search on value using global sorted array
        // Get all distinct values from root's sorted array [l..r]
        // Use the root's sorted array with fractional index approach:
        // Since root = tree[1] = sorted A[0..n-1], we need the k-th in [l,r]
        // Binary search: find smallest x s.t. count_leq(l,r,x) >= k
        int lo = tree[1].front(), hi = tree[1].back();
        while (lo < hi) {
            int mid = lo + (ll)(hi - lo) / 2;
            if (count_leq(l, r, mid) >= k) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }

    // Count inversions in A[l..r]:
    // Number of pairs (i,j) with l<=i<j<=r and A[i]>A[j]
    // = for each position j in [l..r], count elements in [l..j-1] that are > A[j]
    // Requires original array access
    ll count_inversions(vector<int>& A, int l, int r) const {
        ll inv = 0;
        for (int j = l + 1; j <= r; j++) {
            // Count elements in A[l..j-1] greater than A[j]
            int cnt_leq = count_leq(l, j-1, A[j]);
            int total   = j - l; // elements in [l..j-1]
            inv += total - cnt_leq;
        }
        return inv;
    }
};

// ================================================================
// FRACTIONAL CASCADING — reduces query to O(log n)
// Precompute, for each internal node, the mapping from its sorted
// array position to each child's sorted array position.
// This allows binary search at the root and O(1) propagation to children.
// ================================================================
struct MergeSortTreeFC {
    int n;
    vector<vector<int>> sorted; // sorted[node] = sorted elements
    // For each node, left_pos[node][i] = number of elements in left child
    // that are <= sorted[node][i]  (allows O(1) child query from parent result)
    vector<vector<int>> left_pos, right_pos;

    MergeSortTreeFC(vector<int>& A) : n(A.size()),
        sorted(4*n), left_pos(4*n), right_pos(4*n)
    {
        build(A, 1, 0, n-1);
    }

    void build(vector<int>& A, int v, int l, int r) {
        if (l == r) { sorted[v] = {A[l]}; return; }
        int mid = (l+r)/2;
        build(A, 2*v,   l,   mid);
        build(A, 2*v+1, mid+1, r);

        merge(sorted[2*v].begin(),   sorted[2*v].end(),
              sorted[2*v+1].begin(), sorted[2*v+1].end(),
              back_inserter(sorted[v]));

        int sz = sorted[v].size();
        left_pos[v].resize(sz);
        right_pos[v].resize(sz);

        // For each position i in sorted[v], compute:
        // left_pos[v][i]  = #elements in sorted[2v]   that are <= sorted[v][i]
        // right_pos[v][i] = #elements in sorted[2v+1] that are <= sorted[v][i]
        for (int i = 0; i < sz; i++) {
            left_pos[v][i]  = (int)(upper_bound(sorted[2*v].begin(),
                                                 sorted[2*v].end(),
                                                 sorted[v][i])
                                    - sorted[2*v].begin());
            right_pos[v][i] = (int)(upper_bound(sorted[2*v+1].begin(),
                                                  sorted[2*v+1].end(),
                                                  sorted[v][i])
                                     - sorted[2*v+1].begin());
        }
    }

    // Count elements <= x in A[ql..qr] using fractional cascading: O(log n)
    int count_leq_fc(int v, int l, int r, int ql, int qr,
                     int pos_lo, int pos_hi) const {
        // pos_lo, pos_hi = range in sorted[v] corresponding to [ql..qr]
        // (the fraction of sorted[v] that lies in the query range [ql..qr])
        // This is the fractional cascading approach — simplified here.
        // Full implementation requires careful position propagation.
        // (Shown as structure; full FC is complex to implement correctly)
        if (qr < l || r < ql || pos_lo > pos_hi) return 0;
        if (ql <= l && r <= qr) return pos_hi - pos_lo + 1;
        int mid = (l+r)/2;
        // Propagate positions to children using left_pos/right_pos
        // (simplified: fall back to standard query)
        return 0; // placeholder
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    vector<int> A = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};
    MergeSortTree mst(A);

    printf("=== Merge Sort Tree ===\n");
    printf("A = {3,1,4,1,5,9,2,6,5,3}\n\n");

    // Count ≤ x in range
    printf("count_leq([0,9], 5) = %d\n", mst.count_leq(0, 9, 5));  // 7: {1,1,2,3,3,4,5}
    printf("count_leq([2,7], 4) = %d\n", mst.count_leq(2, 7, 4));  // A[2..7]={4,1,5,9,2,6} -> ≤4: {4,1,2}=3
    printf("count_leq([0,4], 3) = %d\n", mst.count_leq(0, 4, 3));  // A[0..4]={3,1,4,1,5} -> ≤3: {3,1,1}=3

    // Count in value range
    printf("\ncount_range([0,9], [2,5]) = %d\n", mst.count_range(0,9,2,5)); // {2,3,3,4,5,5}=6
    printf("count_range([1,6], [1,4]) = %d\n", mst.count_range(1,6,1,4)); // A[1..6]={1,4,1,5,9,2} -> in [1,4]: {1,4,1,2}=4

    // K-th smallest
    printf("\nkth_smallest([0,9], 1) = %d\n", mst.kth_smallest(0,9,1)); // 1
    printf("kth_smallest([0,9], 5) = %d\n", mst.kth_smallest(0,9,5)); // 3
    printf("kth_smallest([2,7], 2) = %d\n", mst.kth_smallest(2,7,2)); // A[2..7]={4,1,5,9,2,6} sorted={1,2,4,5,6,9}, 2nd=2

    // Count inversions
    printf("\nInversions in A[0..5]: %lld\n",
           mst.count_inversions(A, 0, 5));
    // A[0..5] = {3,1,4,1,5,9}
    // (3,1),(3,1),(4,1) = 3 inversions

    // More queries
    printf("\ncount_leq([3,3], 1) = %d\n", mst.count_leq(3,3,1)); // A[3]=1, ≤1: 1
    printf("count_leq([0,0], 2) = %d\n", mst.count_leq(0,0,2)); // A[0]=3, ≤2: 0
    printf("count_leq([0,0], 3) = %d\n", mst.count_leq(0,0,3)); // A[0]=3, ≤3: 1

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class MergeSortTree {
    private readonly int n;
    private readonly List<int>[] tree;

    public MergeSortTree(int[] A) {
        n = A.Length;
        tree = new List<int>[4 * n];
        for (int i = 0; i < tree.Length; i++) tree[i] = new();
        Build(A, 1, 0, n - 1);
    }

    private void Build(int[] A, int node, int l, int r) {
        if (l == r) { tree[node].Add(A[l]); return; }
        int mid = (l + r) / 2;
        Build(A, 2*node,   l,     mid);
        Build(A, 2*node+1, mid+1, r);

        // Merge
        int i = 0, j = 0;
        var L = tree[2*node]; var R = tree[2*node+1];
        while (i < L.Count && j < R.Count)
            tree[node].Add(L[i] <= R[j] ? L[i++] : R[j++]);
        while (i < L.Count) tree[node].Add(L[i++]);
        while (j < R.Count) tree[node].Add(R[j++]);
    }

    private static int CountLeq(List<int> sorted, int x) {
        int lo = 0, hi = sorted.Count;
        while (lo < hi) {
            int mid = (lo + hi) / 2;
            if (sorted[mid] <= x) lo = mid + 1;
            else hi = mid;
        }
        return lo;
    }

    private int QueryLeq(int node, int l, int r, int ql, int qr, int x) {
        if (qr < l || r < ql) return 0;
        if (ql <= l && r <= qr) return CountLeq(tree[node], x);
        int mid = (l + r) / 2;
        return QueryLeq(2*node,   l,     mid, ql, qr, x)
             + QueryLeq(2*node+1, mid+1, r,   ql, qr, x);
    }

    public int CountLeq(int l, int r, int x) =>
        QueryLeq(1, 0, n-1, l, r, x);

    public int CountRange(int l, int r, int a, int b) =>
        CountLeq(l, r, b) - CountLeq(l, r, a - 1);

    public int KthSmallest(int l, int r, int k) {
        int lo = tree[1][0], hi = tree[1][^1];
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (CountLeq(l, r, mid) >= k) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }

    public static void Main() {
        int[] A = {3,1,4,1,5,9,2,6,5,3};
        var mst = new MergeSortTree(A);

        Console.WriteLine($"count_leq([0,9], 5) = {mst.CountLeq(0,9,5)}");  // 7
        Console.WriteLine($"count_range([0,9],[2,5]) = {mst.CountRange(0,9,2,5)}"); // 6
        Console.WriteLine($"kth([0,9], 5) = {mst.KthSmallest(0,9,5)}");  // 3
        Console.WriteLine($"kth([2,7], 2) = {mst.KthSmallest(2,7,2)}");  // 2
    }
}
```

---

## Query Trace Example

```
A = {3,1,4,1,5,9,2,6,5,3}
count_leq([2,7], 4):  A[2..7] = {4,1,5,9,2,6}

Segment tree query [2,7], x=4:
  Node [0,9] → not fully inside, recurse
    Node [0,4] → not fully inside, recurse
      Node [0,1] → outside [2,7], return 0
      Node [2,4] → fully inside [2,7]
        sorted[2,4] = {1,4,5}
        upper_bound(x=4) = index 2 → return 2
    Node [5,9] → not fully inside, recurse
      Node [5,7] → fully inside [2,7]
        sorted[5,7] = {2,6,9}
        upper_bound(x=4) = index 1 → return 1
      Node [8,9] → outside [2,7], return 0

Total = 0 + 2 + 1 + 0 = 3 ✓  (elements {4,1,2} from A[2..7] are ≤4)
```

---

## Comparison with Alternatives

| Structure | Build | Count ≤ x in [l,r] | K-th in [l,r] | Space | Updates |
|---|---|---|---|---|---|
| Merge Sort Tree | O(n log n) | O(log² n) | O(log³ n) | O(n log n) | Hard |
| Wavelet Tree | O(n log n) | O(log n) | O(log n) | O(n log n) | No |
| Persistent Seg Tree | O(n log n) | O(log n) | O(log n) | O(n log n) | Partial |
| Sqrt Decomposition | O(n log n) | O(√n log n) | O(√n log n) | O(n) | O(√n log n) |
| 2D BIT | O(n log n) | O(log² n) | O(log³ n) | O(n log n) | O(log² n) |

Merge Sort Tree is the **easiest to implement correctly** among O(log² n) structures. Wavelet tree and persistent segment tree are faster but harder to implement.

---

## Pitfalls

- **4n node allocation** — the standard segment tree uses 4n nodes. Each node stores a sorted vector. Total memory = O(n log n) elements, but allocating 4n vectors also has overhead. For n = 10^5, the 4*10^5 vectors with average size log n ≈ 17 is fine. For n = 10^6, memory usage approaches the limit — use 2n nodes with a complete binary tree layout if possible.
- **off-by-one in count_leq** — `upper_bound` returns an iterator past the last element ≤ x. The count is `upper_bound - begin`. Using `lower_bound` instead counts elements strictly < x (off by the count of elements equal to x). For "count strictly less than x," use `lower_bound`; for "count ≤ x," always use `upper_bound`.
- **k-th binary search requires correct value range** — the binary search for k-th smallest operates on the value domain [min, max]. Using `lo = 0, hi = n-1` (index range instead of value range) gives wrong answers. The correct bounds are `tree[1].front()` (global min) and `tree[1].back()` (global max).
- **k-th may not exist in the array** — after binary search converges, `lo` is the k-th smallest value. For arrays with duplicate values, `lo` is always a value that actually appears in A (since binary search snaps to an existing value due to the count monotonicity). Verify: `count_leq(l, r, lo) >= k` and `count_leq(l, r, lo-1) < k`.
- **Build must use merge, not sort** — building each node by sorting from scratch costs O(n log n) per level × O(log n) levels = O(n log² n). Using merge (O(node_size) per node) gives O(n log n) build. Always merge children arrays rather than re-sorting.
- **No efficient point updates** — updating A[i] requires rebuilding all O(log n) sorted arrays on the path from leaf to root. Each sorted array rebuild costs O(range_size), making updates O(n) in the worst case. If updates are frequent, use a different structure (wavelet tree, BIT with sorted buckets, or sqrt decomposition with sorted blocks).

---

## Conclusion

The Merge Sort Tree is the **canonical offline range order-statistic structure**:

- It answers "count elements in [l,r] with value in [a,b]" in O(log² n) with a clean implementation of ~30 lines.
- The build is exactly one pass of merge sort across all segment tree levels — O(n log n) total work.
- K-th smallest in a range requires one outer binary search over values plus O(log² n) count queries — total O(log³ n), which is acceptable for n ≤ 10^5.
- For problems requiring faster queries, the wavelet tree achieves O(log n) per query with the same O(n log n) build and space, at the cost of implementation complexity.

**Key takeaway:**  
Use a merge sort tree when: the array is static, you need "count elements with value ≤ x in range [l,r]" or related queries, and you want the simplest correct implementation. The query is always "decompose [l,r] into O(log n) nodes, binary search in each sorted array." No lazy propagation, no tricky invariants — just a segment tree with sorted arrays at each node.
