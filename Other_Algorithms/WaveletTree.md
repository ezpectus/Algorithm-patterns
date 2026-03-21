# Wavelet Tree — Frequency Queries and Order Statistics

## Origin & Motivation

Wavelet Tree was introduced by Grossi, Gupta, and Vitter in 2003 as a compressed representation of a sequence supporting a wide range of queries. It decomposes the alphabet recursively into two halves (like a segment tree over the value domain), storing a bitvector at each node indicating whether each element goes left or right. This structure answers queries that would otherwise require O(n) time, reducing them to O(log sigma) by traversing the tree.

Complexity: **O(n log sigma)** space (bits), **O(log sigma)** per query.  
`sigma` — alphabet size, `n` — sequence length.

---

## Where It's Used

- Compressed suffix arrays and FM-Index (Occ queries in O(log sigma))
- Range frequency and order statistics on sequences
- Computational geometry (dominance counting, range rank)
- Data compression pipelines
- Document retrieval systems

---

## Supported Queries

| Query | Description | Time |
|---|---|---|
| `rank(v, i)` | Count of value v in A[0..i] | O(log sigma) |
| `select(v, k)` | Position of k-th occurrence of v | O(log sigma) |
| `kth(l, r, k)` | k-th smallest value in A[l..r] | O(log sigma) |
| `count_range(l, r, a, b)` | Count of values in [a,b] in A[l..r] | O(log sigma) |
| `quantile(l, r, k)` | Same as kth | O(log sigma) |

---

## Core Idea

Map the value domain `[0, sigma)` to the leaves of a complete binary tree. At each internal node, partition elements into left child (value < mid) and right child (value >= mid), storing a bitvector `B` of length n where `B[i] = 0` means element goes left, `B[i] = 1` means right.

A **rank/select structure** is built on each bitvector, answering:
- `rank0(i)` — number of 0s in B[0..i]
- `rank1(i)` — number of 1s in B[0..i]

To answer `kth(l, r, k)` (k-th smallest in range):
```
At each node with value range [lo, hi], mid = (lo+hi)/2:
    cnt = rank0(r) - rank0(l-1)   // elements going left in [l,r]
    if k <= cnt:
        recurse into left child, map [l,r] to left-child coordinates
    else:
        k -= cnt
        recurse into right child, map [l,r] to right-child coordinates
Return lo when lo == hi
```

Mapping indices between levels uses the rank structure: if going left, new index = rank0(i); if going right, new index = rank1(i) + (size of left subtree at this level).

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Build | O(n log sigma) | O(n log sigma) bits |
| rank(v, i) | O(log sigma) | — |
| select(v, k) | O(log sigma) | — |
| kth(l, r, k) | O(log sigma) | — |
| count_range(l, r, a, b) | O(log sigma) | — |

With a rank/select structure using O(n) bits and O(1) rank queries (e.g., popcount-based), all per-query constants are small. Total space is O(n log sigma) bits ≈ n log sigma / 8 bytes.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// Simple rank structure over a bitvector using prefix popcount
struct Bitvec {
    int n;
    vector<uint64_t> data;
    vector<int> prefix; // prefix[i] = popcount of data[0..i-1] words

    Bitvec() : n(0) {}
    Bitvec(int n) : n(n), data((n + 63) / 64, 0), prefix((n + 63) / 64 + 1, 0) {}

    void set(int i, int v) {
        if (v) data[i / 64] |= (1ULL << (i % 64));
    }

    void build() {
        prefix[0] = 0;
        for (int i = 0; i < (int)data.size(); i++)
            prefix[i + 1] = prefix[i] + __builtin_popcountll(data[i]);
    }

    // Number of 1s in [0, i] (inclusive)
    int rank1(int i) const {
        if (i < 0) return 0;
        int w = i / 64, b = i % 64;
        return prefix[w] + __builtin_popcountll(data[w] & ((2ULL << b) - 1));
    }

    // Number of 0s in [0, i] (inclusive)
    int rank0(int i) const {
        return (i + 1) - rank1(i);
    }
};

struct WaveletTree {
    int lo, hi; // value range [lo, hi)
    int n;
    vector<Bitvec> bv;  // one bitvector per tree level
    vector<int> nodeL;  // left boundary of value range per node (BFS order)
    vector<int> nodeR;  // right boundary
    int levels;

    // A — input array, values in [0, sigma)
    WaveletTree(vector<int> A, int sigma) {
        n = A.size();
        lo = 0; hi = sigma;
        levels = 0;
        int s = sigma;
        while (s > 1) { s = (s + 1) / 2; levels++; }

        // Build level by level (iterative)
        // At each level, A is rearranged by stable partition (left=0, right=1)
        bv.resize(levels, Bitvec(n));

        vector<int> cur = A;
        for (int lv = 0; lv < levels; lv++) {
            int mid_val = 1 << (levels - lv - 1); // value threshold at this level
            // Actually use value midpoint relative to current range — simpler with
            // recursive approach; here we use a flat coordinate-compressed version.
            // Each element's bit at level lv is bit (levels-1-lv) of its value.
            bv[lv] = Bitvec(n);
            for (int i = 0; i < n; i++) {
                int bit = (cur[i] >> (levels - 1 - lv)) & 1;
                bv[lv].set(i, bit);
            }
            bv[lv].build();
            // Stable partition: 0s first, then 1s
            vector<int> left, right;
            for (int x : cur)
                ((x >> (levels - 1 - lv)) & 1 ? right : left).push_back(x);
            cur.clear();
            cur.insert(cur.end(), left.begin(), left.end());
            cur.insert(cur.end(), right.begin(), right.end());
        }
    }

    // Count of value v in A[0..i] (0-indexed, inclusive)
    int rank(int v, int i) const {
        int lo = 0, hi = (1 << levels);
        int l = 0, r = i;
        for (int lv = 0; lv < levels; lv++) {
            int mid = (lo + hi) / 2;
            int zeros = bv[lv].rank0(n - 1); // total zeros at this level
            if (v < mid) {
                // go left
                r = bv[lv].rank0(r) - 1;
                l = bv[lv].rank0(l - 1); // rank0 with i=-1 returns 0
                if (l > 0) l = bv[lv].rank0(l - 1);
                hi = mid;
            } else {
                // go right
                int r1 = bv[lv].rank1(r);
                int l1 = l > 0 ? bv[lv].rank1(l - 1) : 0;
                l = zeros + l1;
                r = zeros + r1 - 1;
                lo = mid;
            }
        }
        return r - l + 1;
    }

    // k-th smallest (1-indexed) in A[l..r] (0-indexed range)
    int kth(int l, int r, int k) const {
        int lo = 0, hi = (1 << levels);
        for (int lv = 0; lv < levels; lv++) {
            int mid = (lo + hi) / 2;
            int zeros_total = bv[lv].rank0(n - 1);
            int cnt_left = bv[lv].rank0(r) - (l > 0 ? bv[lv].rank0(l - 1) : 0);
            if (k <= cnt_left) {
                // go left
                r = bv[lv].rank0(r) - 1;
                l = l > 0 ? bv[lv].rank0(l - 1) : 0;
                hi = mid;
            } else {
                // go right
                k -= cnt_left;
                int new_l = zeros_total + (l > 0 ? bv[lv].rank1(l - 1) : 0);
                int new_r = zeros_total + bv[lv].rank1(r) - 1;
                l = new_l;
                r = new_r;
                lo = mid;
            }
        }
        return lo;
    }

    // Count of values in [a, b] within A[l..r]
    int count_range(int l, int r, int a, int b) const {
        // count values < b+1 minus values < a
        return count_less(l, r, b + 1) - count_less(l, r, a);
    }

    // Count of values strictly less than v in A[l..r]
    int count_less(int l, int r, int v) const {
        if (v <= 0) return 0;
        int lo = 0, hi = (1 << levels);
        int result = 0;
        for (int lv = 0; lv < levels && lo < hi; lv++) {
            int mid = (lo + hi) / 2;
            int zeros_total = bv[lv].rank0(n - 1);
            int cnt_left = bv[lv].rank0(r) - (l > 0 ? bv[lv].rank0(l - 1) : 0);
            if (v <= mid) {
                // all right-subtree elements are >= mid >= v, go left only
                r = bv[lv].rank0(r) - 1;
                l = l > 0 ? bv[lv].rank0(l - 1) : 0;
                hi = mid;
            } else {
                // all left-subtree elements < mid < v, count them all
                result += cnt_left;
                int new_l = zeros_total + (l > 0 ? bv[lv].rank1(l - 1) : 0);
                int new_r = zeros_total + bv[lv].rank1(r) - 1;
                l = new_l;
                r = new_r;
                lo = mid;
            }
        }
        return result;
    }
};

// --- Usage ---
int main() {
    // Values must be in [0, sigma)
    vector<int> A = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};
    int sigma = 16; // next power of 2 >= max_value+1
    WaveletTree wt(A, sigma);

    // 3rd smallest in A[0..9]
    cout << "3rd smallest in whole array: " << wt.kth(0, 9, 3) << "\n"; // 2

    // Count values in [1,4] within A[2..7]
    cout << "Count [1,4] in A[2..7]: " << wt.count_range(2, 7, 1, 4) << "\n";

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class WaveletTree {
    private readonly int n;
    private readonly int levels;
    private readonly int[] zeros;  // zeros[lv] = total 0-bits at level lv
    private readonly int[][] prefix; // prefix[lv][i] = rank1 in [0..i-1] at level lv

    public WaveletTree(int[] A, int sigma) {
        n = A.Length;
        levels = 0;
        int s = sigma;
        while (s > 1) { s = (s + 1) / 2; levels++; }

        prefix = new int[levels][];
        zeros  = new int[levels];

        int[] cur = (int[])A.Clone();
        for (int lv = 0; lv < levels; lv++) {
            int shift = levels - 1 - lv;
            // Build prefix rank1 array (size n+1)
            prefix[lv] = new int[n + 1];
            prefix[lv][0] = 0;
            for (int i = 0; i < n; i++) {
                int bit = (cur[i] >> shift) & 1;
                prefix[lv][i + 1] = prefix[lv][i] + bit;
            }
            zeros[lv] = n - prefix[lv][n];

            // Stable partition
            var left  = new List<int>();
            var right = new List<int>();
            foreach (int x in cur)
                ((x >> shift & 1) == 0 ? left : right).Add(x);
            int idx = 0;
            foreach (int x in left)  cur[idx++] = x;
            foreach (int x in right) cur[idx++] = x;
        }
    }

    // rank1 in [0..i] at level lv (0-indexed inclusive)
    private int Rank1(int lv, int i) =>
        i < 0 ? 0 : prefix[lv][i + 1];

    // rank0 in [0..i] at level lv
    private int Rank0(int lv, int i) =>
        i < 0 ? 0 : (i + 1) - prefix[lv][i + 1];

    // k-th smallest (1-indexed) in A[l..r]
    public int Kth(int l, int r, int k) {
        int lo = 0, hi = 1 << levels;
        for (int lv = 0; lv < levels; lv++) {
            int cntLeft = Rank0(lv, r) - Rank0(lv, l - 1);
            if (k <= cntLeft) {
                r = Rank0(lv, r) - 1;
                l = Rank0(lv, l - 1);
                hi = (lo + hi) / 2;
            } else {
                k -= cntLeft;
                int newL = zeros[lv] + Rank1(lv, l - 1);
                int newR = zeros[lv] + Rank1(lv, r) - 1;
                l = newL; r = newR;
                lo = (lo + hi) / 2;
            }
        }
        return lo;
    }

    // Count of values strictly less than v in A[l..r]
    public int CountLess(int l, int r, int v) {
        if (v <= 0) return 0;
        int lo = 0, hi = 1 << levels, result = 0;
        for (int lv = 0; lv < levels && lo < hi; lv++) {
            int mid = (lo + hi) / 2;
            int cntLeft = Rank0(lv, r) - Rank0(lv, l - 1);
            if (v <= mid) {
                r = Rank0(lv, r) - 1;
                l = Rank0(lv, l - 1);
                hi = mid;
            } else {
                result += cntLeft;
                int newL = zeros[lv] + Rank1(lv, l - 1);
                int newR = zeros[lv] + Rank1(lv, r) - 1;
                l = newL; r = newR;
                lo = mid;
            }
        }
        return result;
    }

    // Count of values in [a, b] within A[l..r]
    public int CountRange(int l, int r, int a, int b) =>
        CountLess(l, r, b + 1) - CountLess(l, r, a);

    // --- Usage ---
    public static void Main() {
        int[] A = { 3, 1, 4, 1, 5, 9, 2, 6, 5, 3 };
        int sigma = 16;
        var wt = new WaveletTree(A, sigma);

        Console.WriteLine($"3rd smallest in A[0..9]: {wt.Kth(0, 9, 3)}");       // 2
        Console.WriteLine($"Count [1,4] in A[2..7]: {wt.CountRange(2, 7, 1, 4)}");
    }
}
```

---

## Pitfalls

- **sigma must be a power of 2** in the bit-level decomposition above. If your alphabet is not a power of 2, round sigma up to the next power of 2 or use a general midpoint split. Values outside `[0, sigma)` corrupt the bitvectors silently.
- **0-indexed vs 1-indexed rank** — `rank0(i)` returning the count in `[0..i]` inclusive is the most common source of off-by-one bugs. Be consistent: define and document whether the index is inclusive or exclusive before implementing.
- **Index mapping between levels** — when descending into the right child, the new left boundary is `zeros[lv] + rank1(lv, l-1)`, not just `rank1`. Forgetting to add `zeros[lv]` is the single most frequent bug.
- **k-th is 1-indexed** — `kth(l, r, 0)` is meaningless. Validate k in `[1, r-l+1]` before querying.
- **Space constant** — n*levels integers for prefix arrays is O(n log sigma) integers, not bits. For large n and sigma, replace prefix arrays with compact bitvectors (64-bit word popcount) to achieve the theoretical O(n log sigma) **bits**.
- **select query** — not shown above. Implement by binary searching on the prefix array at each level: O(log n * log sigma) naively, O(log sigma) with a proper select structure on each bitvector.
- **Persistent wavelet tree** — a common competitive-programming extension that supports offline range queries on subarrays defined by index ranges in an array of versions. Build it by storing left/right child pointers and creating new nodes only on the path modified per update.

---

## Conclusion

Wavelet Tree is a **universal range query structure** over sequences:

- Answers k-th order statistics, frequency, and range counting in **O(log sigma)** regardless of range size.
- Replaces merge-sort tree (O(log² n) per query) with a cleaner O(log sigma) structure.
- Serves as the rank/select backbone for FM-Index over large alphabets.
- Extends naturally to persistent and dynamic variants.

**Key takeaway:**  
When you need multiple different query types (rank, kth, count in value range) on a static sequence, Wavelet Tree unifies them all at O(log sigma) per query — far cheaper than maintaining separate structures for each query type.
