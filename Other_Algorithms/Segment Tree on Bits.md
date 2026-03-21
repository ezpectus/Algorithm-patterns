# Segment Tree on Bits — Bitwise Segment Trees

## Origin & Motivation

A Segment Tree on Bits is not a single algorithm but a family of techniques that store and aggregate **bitsets** or **bitwise values** inside segment tree nodes. The idea is to exploit 64-bit word-level parallelism: instead of storing a single scalar per node, each node holds a bitset of width `W`, and all merge operations reduce to bitwise AND / OR / XOR / popcount over machine words, making them **64x faster** than scalar equivalents.

Complexity: **O(n * W/64)** build, **O(W/64 * log n)** per update/query, where `W` is the bitset width.  
When W = 64 the per-operation cost collapses to O(log n) with a tiny constant.

---

## Where It's Used

- Range OR / AND / XOR aggregation over bitmasks
- Counting set bits in a union/intersection of ranges (popcount queries)
- 2D problems reduced to bitset rows (segment tree of bitsets)
- DP optimizations where transitions are bitset shifts (knapsack in O(n²/64))
- Graph reachability in layers (BFS over adjacency bitsets)
- Competitive programming: "how many distinct values in range", "range bitwise AND != 0"

---

## Variants

| Variant | Node stores | Merge | Typical query |
|---|---|---|---|
| Bitwise OR tree | uint64 or bitset | a \| b | Range OR |
| Bitwise AND tree | uint64 or bitset | a & b | Range AND |
| Bitwise XOR tree | uint64 or bitset | a ^ b | Range XOR |
| Popcount tree | uint64 | popcount(a \| b) or sum | Count set bits in range union |
| Bitset seg tree | bitset\<W\> | custom | Range union / intersection |
| Seg tree of bitsets | bitset per node | OR/AND | 2D membership, DP |

---

## Core Idea

### Range Bitwise OR / AND / XOR

Standard segment tree where the node value is the aggregate bitwise operation over its range. Build, update, and query are identical to a scalar segment tree — only the stored type and merge operator change.

```
node[v] = node[2v] OP node[2v+1]    // OP in {|, &, ^}
```

### Segment Tree of Bitsets (2D membership)

Each node at position `[l, r]` stores a `bitset<W>` representing which "columns" (second dimension) have at least one set element in rows `[l, r]`. Querying which columns appear in a row range reduces to OR-ing O(log n) bitsets, each of size W/64 words — total cost O(W/64 * log n).

### Popcount over range union

Each leaf holds a bitmask. The query "how many distinct bits are set across A[l..r]" computes the OR of all bitmasks in the range, then applies popcount. A segment tree stores per-node OR aggregates, so the range OR is answered in O(log n) and popcount applied once.

### DP with bitset seg tree (knapsack layer trick)

Store DP states as bitsets. A segment tree over items allows range-updates of the form `dp |= (dp << w[i])` to be applied in bulk, reducing naive O(n²) knapsack to O(n² / 64).

---

## Complexity Analysis

| Operation | Scalar seg tree | Bitset seg tree (width W) |
|---|---|---|
| Build | O(n) | O(n * W/64) |
| Point update | O(log n) | O(W/64 * log n) |
| Range query | O(log n) | O(W/64 * log n) |
| Space | O(n) | O(n * W/64) |

For W = 64: identical asymptotic to scalar, constant factor ~1.  
For W = 1024: 16x more work per node but handles 1024-bit bitmasks atomically.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// -------------------------------------------------------
// 1. Range Bitwise OR / AND / XOR  (scalar uint64)
// -------------------------------------------------------
template<typename T, T(*OP)(T,T), T IDENTITY>
struct SegTree {
    int n;
    vector<T> tree;

    SegTree(int n) : n(n), tree(4 * n, IDENTITY) {}

    void build(const vector<T>& A, int v, int l, int r) {
        if (l == r) { tree[v] = A[l]; return; }
        int mid = (l + r) / 2;
        build(A, 2*v, l, mid);
        build(A, 2*v+1, mid+1, r);
        tree[v] = OP(tree[2*v], tree[2*v+1]);
    }

    void update(int v, int l, int r, int pos, T val) {
        if (l == r) { tree[v] = val; return; }
        int mid = (l + r) / 2;
        if (pos <= mid) update(2*v, l, mid, pos, val);
        else            update(2*v+1, mid+1, r, pos, val);
        tree[v] = OP(tree[2*v], tree[2*v+1]);
    }

    T query(int v, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return IDENTITY;
        if (ql <= l && r <= qr) return tree[v];
        int mid = (l + r) / 2;
        return OP(query(2*v, l, mid, ql, qr),
                  query(2*v+1, mid+1, r, ql, qr));
    }

    void build(const vector<T>& A) { build(A, 1, 0, n-1); }
    void update(int pos, T val)     { update(1, 0, n-1, pos, val); }
    T    query(int l, int r)        { return query(1, 0, n-1, l, r); }
};

uint64_t op_or (uint64_t a, uint64_t b) { return a | b; }
uint64_t op_and(uint64_t a, uint64_t b) { return a & b; }
uint64_t op_xor(uint64_t a, uint64_t b) { return a ^ b; }

// -------------------------------------------------------
// 2. Segment Tree of Bitsets  (fixed width W)
// -------------------------------------------------------
template<size_t W>
struct BitsetSegTree {
    int n;
    vector<bitset<W>> tree;

    BitsetSegTree(int n) : n(n), tree(4 * n) {}

    void build(const vector<bitset<W>>& A, int v, int l, int r) {
        if (l == r) { tree[v] = A[l]; return; }
        int mid = (l + r) / 2;
        build(A, 2*v, l, mid);
        build(A, 2*v+1, mid+1, r);
        tree[v] = tree[2*v] | tree[2*v+1];
    }

    void update(int v, int l, int r, int pos, const bitset<W>& val) {
        if (l == r) { tree[v] = val; return; }
        int mid = (l + r) / 2;
        if (pos <= mid) update(2*v, l, mid, pos, val);
        else            update(2*v+1, mid+1, r, pos, val);
        tree[v] = tree[2*v] | tree[2*v+1];
    }

    // Range OR: returns bitset of all bits set in any A[l..r]
    bitset<W> query(int v, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return bitset<W>();
        if (ql <= l && r <= qr) return tree[v];
        int mid = (l + r) / 2;
        return query(2*v, l, mid, ql, qr)
             | query(2*v+1, mid+1, r, ql, qr);
    }

    void       build(const vector<bitset<W>>& A) { build(A, 1, 0, n-1); }
    void       update(int pos, const bitset<W>& val) { update(1, 0, n-1, pos, val); }
    bitset<W>  query(int l, int r)  { return query(1, 0, n-1, l, r); }

    // Count distinct set bits in union of A[l..r]
    int popcount_query(int l, int r) {
        return (int)query(l, r).count();
    }
};

// -------------------------------------------------------
// 3. Popcount range query (how many 1-bits in range XOR)
// -------------------------------------------------------
// Special case: store uint64 XOR aggregate, answer popcount of range XOR
struct XorPopcountTree {
    int n;
    vector<uint64_t> tree;

    XorPopcountTree(int n, const vector<uint64_t>& A) : n(n), tree(4*n, 0) {
        build(A, 1, 0, n-1);
    }

    void build(const vector<uint64_t>& A, int v, int l, int r) {
        if (l == r) { tree[v] = A[l]; return; }
        int mid = (l + r) / 2;
        build(A, 2*v, l, mid);
        build(A, 2*v+1, mid+1, r);
        tree[v] = tree[2*v] ^ tree[2*v+1];
    }

    uint64_t query_xor(int v, int l, int r, int ql, int qr) {
        if (qr < l || r < ql) return 0;
        if (ql <= l && r <= qr) return tree[v];
        int mid = (l + r) / 2;
        return query_xor(2*v, l, mid, ql, qr)
             ^ query_xor(2*v+1, mid+1, r, ql, qr);
    }

    // Number of set bits in XOR of A[l..r]
    int popcount(int l, int r) {
        return __builtin_popcountll(query_xor(1, 0, n-1, l, r));
    }
};

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    // --- Scalar OR tree ---
    vector<uint64_t> A = {0b1010, 0b0101, 0b1100, 0b0011};
    SegTree<uint64_t, op_or, 0ULL> orTree(4);
    orTree.build(A);
    printf("Range OR [0,3]: %llx\n", orTree.query(0, 3));  // 0b1111 = 0xf

    // --- AND tree (identity = all 1s) ---
    SegTree<uint64_t, op_and, ~0ULL> andTree(4);
    andTree.build(A);
    printf("Range AND [0,1]: %llx\n", andTree.query(0, 1)); // 0b0000

    // --- Bitset segment tree (W=8 for demo) ---
    vector<bitset<8>> B(4);
    B[0] = 0b10100001;
    B[1] = 0b01010010;
    B[2] = 0b11000100;
    B[3] = 0b00001000;
    BitsetSegTree<8> bst(4);
    bst.build(B);
    printf("Distinct bits in [0,2]: %d\n", bst.popcount_query(0, 2)); // popcount of OR

    // --- XOR popcount ---
    XorPopcountTree xpt(4, A);
    printf("Popcount of XOR [0,3]: %d\n", xpt.popcount(0, 3));

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections;
using System.Collections.Generic;
using System.Numerics;

// -------------------------------------------------------
// 1. Generic bitwise segment tree (ulong, single word)
// -------------------------------------------------------
public class BitwiseSegTree {
    public enum Op { Or, And, Xor }

    private readonly int n;
    private readonly ulong[] tree;
    private readonly Op op;
    private readonly ulong identity;

    public BitwiseSegTree(int n, Op op) {
        this.n  = n;
        this.op = op;
        identity = op == Op.And ? ulong.MaxValue : 0UL;
        tree = new ulong[4 * n];
        Array.Fill(tree, identity);
    }

    private ulong Merge(ulong a, ulong b) => op switch {
        Op.Or  => a | b,
        Op.And => a & b,
        Op.Xor => a ^ b,
        _      => throw new InvalidOperationException()
    };

    public void Build(ulong[] A, int v = 1, int l = 0, int r = -1) {
        if (r == -1) r = n - 1;
        if (l == r) { tree[v] = A[l]; return; }
        int mid = (l + r) / 2;
        Build(A, 2*v, l, mid);
        Build(A, 2*v+1, mid+1, r);
        tree[v] = Merge(tree[2*v], tree[2*v+1]);
    }

    public void Update(int pos, ulong val, int v = 1, int l = 0, int r = -1) {
        if (r == -1) r = n - 1;
        if (l == r) { tree[v] = val; return; }
        int mid = (l + r) / 2;
        if (pos <= mid) Update(pos, val, 2*v, l, mid);
        else            Update(pos, val, 2*v+1, mid+1, r);
        tree[v] = Merge(tree[2*v], tree[2*v+1]);
    }

    public ulong Query(int ql, int qr, int v = 1, int l = 0, int r = -1) {
        if (r == -1) r = n - 1;
        if (qr < l || r < ql) return identity;
        if (ql <= l && r <= qr) return tree[v];
        int mid = (l + r) / 2;
        return Merge(Query(ql, qr, 2*v, l, mid),
                     Query(ql, qr, 2*v+1, mid+1, r));
    }

    public int PopcountQuery(int l, int r) =>
        BitOperations.PopCount(Query(l, r));
}

// -------------------------------------------------------
// 2. Segment Tree of BitArrays  (variable width W)
// -------------------------------------------------------
public class BitArraySegTree {
    private readonly int n, W;
    private readonly ulong[][] tree; // tree[node] = ulong array of ceil(W/64) words

    public BitArraySegTree(int n, int W) {
        this.n = n;
        this.W = W;
        int words = (W + 63) / 64;
        tree = new ulong[4 * n][];
        for (int i = 0; i < tree.Length; i++)
            tree[i] = new ulong[words];
    }

    private ulong[] OrMerge(ulong[] a, ulong[] b) {
        var res = new ulong[a.Length];
        for (int i = 0; i < a.Length; i++) res[i] = a[i] | b[i];
        return res;
    }

    private bool IsZero(ulong[] a) {
        foreach (var w in a) if (w != 0) return false;
        return true;
    }

    public void Build(ulong[][] A, int v = 1, int l = 0, int r = -1) {
        if (r == -1) r = n - 1;
        if (l == r) { Array.Copy(A[l], tree[v], A[l].Length); return; }
        int mid = (l + r) / 2;
        Build(A, 2*v, l, mid);
        Build(A, 2*v+1, mid+1, r);
        tree[v] = OrMerge(tree[2*v], tree[2*v+1]);
    }

    public ulong[] Query(int ql, int qr, int v = 1, int l = 0, int r = -1) {
        if (r == -1) r = n - 1;
        if (qr < l || r < ql) return new ulong[(W + 63) / 64];
        if (ql <= l && r <= qr) return tree[v];
        int mid = (l + r) / 2;
        return OrMerge(Query(ql, qr, 2*v, l, mid),
                       Query(ql, qr, 2*v+1, mid+1, r));
    }

    public int PopcountQuery(int l, int r) {
        var result = Query(l, r);
        int cnt = 0;
        foreach (var w in result) cnt += BitOperations.PopCount(w);
        return cnt;
    }
}

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
public class Program {
    public static void Main() {
        // Scalar OR tree
        var orTree = new BitwiseSegTree(4, BitwiseSegTree.Op.Or);
        ulong[] A = { 0b1010, 0b0101, 0b1100, 0b0011 };
        orTree.Build(A);
        Console.WriteLine($"Range OR [0,3]: {orTree.Query(0, 3):X}");      // F
        Console.WriteLine($"Range AND [0,1]: ");

        var andTree = new BitwiseSegTree(4, BitwiseSegTree.Op.And);
        andTree.Build(A);
        Console.WriteLine($"{andTree.Query(0, 1):X}");                      // 0

        // Bitarray seg tree (W=8)
        int words = 1; // 8 bits = 1 ulong word
        var bst = new BitArraySegTree(4, 8);
        ulong[][] B = {
            new ulong[]{ 0b10100001 },
            new ulong[]{ 0b01010010 },
            new ulong[]{ 0b11000100 },
            new ulong[]{ 0b00001000 }
        };
        bst.Build(B);
        Console.WriteLine($"Distinct bits in [0,2]: {bst.PopcountQuery(0, 2)}");

        // XOR popcount
        var xorTree = new BitwiseSegTree(4, BitwiseSegTree.Op.Xor);
        xorTree.Build(A);
        Console.WriteLine($"Popcount of XOR [0,3]: {xorTree.PopcountQuery(0, 3)}");
    }
}
```

---

## Pitfalls

- **AND identity must be all-ones** — for a range AND tree, the identity element (returned for out-of-range nodes) must be `~0ULL` / `ulong.MaxValue`, not 0. Using 0 as identity corrupts every AND query that touches a boundary node.
- **Bitset identity for OR is all-zeros** — conversely, OR identity is the zero bitset. Mixing up the two identities when switching between OR and AND trees is a common mistake.
- **`std::bitset` size is compile-time** — in C++, `bitset<W>` requires W to be a compile-time constant. For runtime-width bitsets, use `vector<uint64_t>` of size `ceil(W/64)` and implement OR/AND manually over words. The C# `BitArraySegTree` above does exactly this.
- **Memory** — a bitset seg tree of depth log n with n leaves stores 4n nodes each holding W/64 words: total `4n * W/64 * 8` bytes. For n=100000, W=1000 this is ~50 MB. Plan accordingly.
- **Node reuse on lazy propagation** — lazy bitwise seg trees (e.g., range assign a bitmask, then query OR) require the lazy tag to be a bitmask too, and the push-down must OR/AND the tag into children rather than overwrite. Forgetting this produces wrong results that are hard to detect.
- **XOR range query is not XOR of all elements** — the aggregate stored in a node is `A[l] XOR A[l+1] XOR ... XOR A[r]`, not OR or AND. Querying XOR then taking popcount answers "how many bits toggle an odd number of times in the range", which is different from "how many distinct bits are set" (which needs OR).
- **Operator commutativity and associativity** — all three (OR, AND, XOR) are commutative and associative, so they are valid segment tree merge operators. Do not attempt to build a seg tree with non-associative operations using this template.

---

## Conclusion

Segment Tree on Bits exploits **word-level parallelism** to accelerate range aggregation:

- Range OR / AND / XOR queries run in **O(log n)** with a constant up to 64x smaller than scalar equivalents.
- Bitset seg trees extend this to W-bit aggregates in **O(W/64 * log n)**, enabling 2D membership, DP optimization, and distinct-bit counting.
- The structure is a drop-in replacement for any scalar seg tree — only the stored type and merge operator change.

**Key takeaway:**  
Whenever the merge operation is a bitwise operator and W fits in memory, prefer a bitset segment tree over per-bit scalar trees. The 64x word-level speedup is free and requires no algorithmic change — only a type substitution.
