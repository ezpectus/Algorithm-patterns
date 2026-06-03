# 2D Fenwick Tree (Binary Indexed Tree)

## Origin & Motivation

**2D Fenwick Tree** (2D Binary Indexed Tree) is the natural extension of the 1D BIT introduced by Peter Fenwick in 1994, lifted to work on a 2D grid of `n × m` cells. It answers prefix sum queries and supports point updates on a 2D array in **O(log n · log m)** time with an extremely compact implementation — just a 2D array and two nested loops, each controlled by the classic `i += i & -i` / `i -= i & -i` trick.

**The problem it solves:** Given a 2D grid that receives point updates (add a value to cell `(x, y)`), answer rectangle sum queries `sum(x1, y1, x2, y2)` efficiently. The naive approach is O(1) update but O(nm) query, or O(nm) build + O(1) query for a static prefix-sum table. The 2D Fenwick Tree gives **O(log n · log m)** for both, using only O(nm) space.

**The key idea:** Every cell `(i, j)` in the BIT stores the sum of a rectangular block whose dimensions in each axis are determined by the lowest set bit of `i` and `j` respectively — exactly the 1D BIT principle applied twice. A prefix query `[1..x][1..y]` decomposes into O(log n · log m) such blocks; an update propagates to O(log n · log m) cells.

Complexity: **O(nm)** build, **O(log n · log m)** update and prefix query, **O(nm)** space.

---

## Where It Is Used

- 2D grid point updates with rectangle sum queries (competitive programming standard)
- Image processing: summed area tables with dynamic updates
- Counting inversions in 2D (offline algorithms)
- Computational geometry: dominance counting, 2D order statistics
- Game grids and simulations where cells are updated and rectangular regions summed

---

## Core Structure

### 1D BIT Recap

In 1D a BIT node `i` stores the sum of `a[i - lowbit(i) + 1 .. i]` where `lowbit(i) = i & -i`. A prefix query `sum(x)` walks `x → x - lowbit(x) → ...` down to 0. An update at `x` walks `x → x + lowbit(x) → ...` up to `n`.

### 2D Extension

In 2D each node `(i, j)` stores the sum of the rectangle `[i - lowbit(i)+1 .. i] × [j - lowbit(j)+1 .. j]`. The outer loop runs over the x-axis exactly as in 1D; for each outer index, the inner loop runs over the y-axis exactly as in 1D:

```
update(x, y, delta):
    for i = x; i <= n; i += i & -i          // outer BIT walk up
        for j = y; j <= m; j += j & -j      // inner BIT walk up
            tree[i][j] += delta

prefix(x, y):                                // sum [1..x][1..y]
    for i = x; i > 0; i -= i & -i           // outer BIT walk down
        for j = y; j > 0; j -= j & -j       // inner BIT walk down
            s += tree[i][j]
    return s
```

### Rectangle Query via Inclusion-Exclusion

```
sum(x1, y1, x2, y2) =
    prefix(x2, y2)
  - prefix(x1-1, y2)
  - prefix(x2, y1-1)
  + prefix(x1-1, y1-1)
```

This is the standard 2D inclusion-exclusion identity. Each call to `prefix` costs O(log n · log m), so the total rectangle query costs O(log n · log m).

---

## Range Update, Range Query (4-BIT Trick)

To support rectangle updates `add(x1,y1,x2,y2,v)` (add `v` to every cell in the rectangle) AND rectangle sum queries, maintain **four** BITs `B1, B2, B3, B4` and use the following algebraic identity.

Define `f(x,y) = B1·xy - B2·y - B3·x + B4`. Then:

```
prefix_sum(x, y) = f(x,y)

range_add(x1,y1,x2,y2,v):
    // updates to all four BITs so that prefix_sum gives correct totals
    add B1 at (x1,y1):+v, (x1,y2+1):-v, (x2+1,y1):-v, (x2+1,y2+1):+v
    add B2 at corners with weights (x1-1), x2
    add B3 at corners with weights (y1-1), y2
    add B4 at corners with weights (x1-1)(y1-1), x2·y2, etc.

range_sum(x1,y1,x2,y2) = prefix(x2,y2) - prefix(x1-1,y2)
                        - prefix(x2,y1-1) + prefix(x1-1,y1-1)
```

Each of the four BITs uses the same O(log n · log m) inner operations. Total: O(log n · log m) per update and query.

---

## Complexity Summary

| Variant | Build | Update | Query | Space |
|---|---|---|---|---|
| Point update, prefix sum | O(nm) | O(log n · log m) | O(log n · log m) | O(nm) |
| Point update, range sum | O(nm) | O(log n · log m) | O(log n · log m) | O(nm) |
| Range update, range sum | O(nm) | O(log n · log m) | O(log n · log m) | O(4nm) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// 2D FENWICK TREE — Point Update, Range Sum Query
// 1-indexed. Cells: (1..n) x (1..m)
// ================================================================
struct Fenwick2D {
    int n, m;
    vector<vector<ll>> tree;

    Fenwick2D(int n, int m) : n(n), m(m), tree(n+1, vector<ll>(m+1, 0)) {}

    // Add delta to cell (x, y)
    void update(int x, int y, ll delta) {
        for (int i = x; i <= n; i += i & -i)
            for (int j = y; j <= m; j += j & -j)
                tree[i][j] += delta;
    }

    // Prefix sum [1..x][1..y]
    ll prefix(int x, int y) const {
        ll s = 0;
        for (int i = x; i > 0; i -= i & -i)
            for (int j = y; j > 0; j -= j & -j)
                s += tree[i][j];
        return s;
    }

    // Rectangle sum [x1..x2][y1..y2]
    ll query(int x1, int y1, int x2, int y2) const {
        return prefix(x2, y2)
             - prefix(x1-1, y2)
             - prefix(x2, y1-1)
             + prefix(x1-1, y1-1);
    }
};

// ================================================================
// 2D FENWICK TREE — Range Update, Range Query (4-BIT method)
// add(x1,y1,x2,y2,v): add v to every cell in rectangle
// sum(x1,y1,x2,y2):   sum of all cells in rectangle
// ================================================================
struct Fenwick2D_RURQ {
    int n, m;
    vector<vector<ll>> B1, B2, B3, B4;

    Fenwick2D_RURQ(int n, int m) : n(n), m(m),
        B1(n+2, vector<ll>(m+2, 0)),
        B2(n+2, vector<ll>(m+2, 0)),
        B3(n+2, vector<ll>(m+2, 0)),
        B4(n+2, vector<ll>(m+2, 0)) {}

    void _add(vector<vector<ll>>& B, int x, int y, ll v) {
        for (int i = x; i <= n; i += i & -i)
            for (int j = y; j <= m; j += j & -j)
                B[i][j] += v;
    }
    ll _sum(vector<vector<ll>>& B, int x, int y) {
        ll s = 0;
        for (int i = x; i > 0; i -= i & -i)
            for (int j = y; j > 0; j -= j & -j)
                s += B[i][j];
        return s;
    }

    // Add v to every cell in [x1..x2][y1..y2]
    void range_add(int x1, int y1, int x2, int y2, ll v) {
        _add(B1, x1,   y1,   v);     _add(B1, x1,   y2+1, -v);
        _add(B1, x2+1, y1,  -v);     _add(B1, x2+1, y2+1,  v);

        _add(B2, x1,   y1,   v*(x1-1));   _add(B2, x1,   y2+1, -v*(x1-1));
        _add(B2, x2+1, y1,  -v*x2);       _add(B2, x2+1, y2+1,  v*x2);

        _add(B3, x1,   y1,   v*(y1-1));   _add(B3, x1,   y2+1, -v*y2);
        _add(B3, x2+1, y1,  -v*(y1-1));   _add(B3, x2+1, y2+1,  v*y2);

        _add(B4, x1,   y1,   v*(x1-1)*(y1-1));
        _add(B4, x1,   y2+1, -v*(x1-1)*y2);
        _add(B4, x2+1, y1,  -v*x2*(y1-1));
        _add(B4, x2+1, y2+1, v*x2*y2);
    }

    // Prefix sum [1..x][1..y]
    ll prefix(int x, int y) {
        return _sum(B1,x,y)*x*y
             - _sum(B2,x,y)*y
             - _sum(B3,x,y)*x
             + _sum(B4,x,y);
    }

    // Rectangle sum [x1..x2][y1..y2]
    ll range_sum(int x1, int y1, int x2, int y2) {
        return prefix(x2,y2) - prefix(x1-1,y2)
             - prefix(x2,y1-1) + prefix(x1-1,y1-1);
    }
};

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- PURQ demo ----
    {
        printf("=== 2D Fenwick PURQ ===\n");
        Fenwick2D fw(5, 5);
        fw.update(1,1,1); fw.update(2,2,3);
        fw.update(3,3,5); fw.update(5,5,7);
        printf("sum(1,1,5,5)=%lld  (expect 16)\n", fw.query(1,1,5,5));
        printf("sum(2,2,3,3)=%lld  (expect 8)\n",  fw.query(2,2,3,3));
        printf("sum(1,1,2,2)=%lld  (expect 4)\n",  fw.query(1,1,2,2));
    }

    // ---- RURQ demo ----
    {
        printf("\n=== 2D Fenwick RURQ ===\n");
        Fenwick2D_RURQ fw(4, 4);
        fw.range_add(1,1,4,4,1);   // all cells +1
        fw.range_add(2,2,3,3,2);   // inner 2x2 +2
        printf("sum(1,1,4,4)=%lld  (expect 24)\n", fw.range_sum(1,1,4,4));
        printf("sum(2,2,3,3)=%lld  (expect 12)\n", fw.range_sum(2,2,3,3));
        printf("sum(1,1,1,1)=%lld  (expect 1)\n",  fw.range_sum(1,1,1,1));
    }

    // ---- Stress: PURQ ----
    {
        srand(42); int errors = 0;
        for (int trial = 0; trial < 300; trial++) {
            int n=12, m=12;
            Fenwick2D fw(n,m);
            vector<vector<ll>> grid(n+1, vector<ll>(m+1, 0));
            for (int i = 0; i < 60; i++) {
                int x=rand()%n+1, y=rand()%m+1; ll v=rand()%20-5;
                fw.update(x,y,v); grid[x][y]+=v;
            }
            for (int q = 0; q < 60; q++) {
                int x1=rand()%n+1, y1=rand()%m+1;
                int x2=rand()%n+1, y2=rand()%m+1;
                if(x1>x2)swap(x1,x2); if(y1>y2)swap(y1,y2);
                ll brute=0;
                for(int i=x1;i<=x2;i++) for(int j=y1;j<=y2;j++) brute+=grid[i][j];
                if(fw.query(x1,y1,x2,y2)!=brute) errors++;
            }
        }
        printf("\nPURQ stress 300 trials: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Stress: RURQ ----
    {
        srand(99); int errors = 0;
        for (int trial = 0; trial < 300; trial++) {
            int n=12, m=12;
            Fenwick2D_RURQ fw(n,m);
            vector<vector<ll>> grid(n+2, vector<ll>(m+2, 0));
            for (int i = 0; i < 40; i++) {
                int x1=rand()%n+1, y1=rand()%m+1;
                int x2=rand()%n+1, y2=rand()%m+1;
                if(x1>x2)swap(x1,x2); if(y1>y2)swap(y1,y2);
                ll v=rand()%10-3;
                fw.range_add(x1,y1,x2,y2,v);
                for(int a=x1;a<=x2;a++) for(int b=y1;b<=y2;b++) grid[a][b]+=v;
            }
            for (int q = 0; q < 60; q++) {
                int x1=rand()%n+1, y1=rand()%m+1;
                int x2=rand()%n+1, y2=rand()%m+1;
                if(x1>x2)swap(x1,x2); if(y1>y2)swap(y1,y2);
                ll brute=0;
                for(int i=x1;i<=x2;i++) for(int j=y1;j<=y2;j++) brute+=grid[i][j];
                if(fw.range_sum(x1,y1,x2,y2)!=brute) errors++;
            }
        }
        printf("RURQ stress 300 trials: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;

// ================================================================
// 2D FENWICK TREE — C#
// ================================================================
public class Fenwick2D
{
    private readonly int _n, _m;
    private readonly long[,] _tree;

    public Fenwick2D(int n, int m)
    {
        _n = n; _m = m;
        _tree = new long[n+1, m+1];
    }

    public void Update(int x, int y, long delta)
    {
        for (int i = x; i <= _n; i += i & -i)
            for (int j = y; j <= _m; j += j & -j)
                _tree[i,j] += delta;
    }

    public long Prefix(int x, int y)
    {
        long s = 0;
        for (int i = x; i > 0; i -= i & -i)
            for (int j = y; j > 0; j -= j & -j)
                s += _tree[i,j];
        return s;
    }

    public long Query(int x1, int y1, int x2, int y2)
        => Prefix(x2,y2) - Prefix(x1-1,y2) - Prefix(x2,y1-1) + Prefix(x1-1,y1-1);
}

public class Fenwick2D_RURQ
{
    private readonly int _n, _m;
    private readonly long[,] _b1, _b2, _b3, _b4;

    public Fenwick2D_RURQ(int n, int m)
    {
        _n = n; _m = m;
        _b1 = new long[n+2, m+2]; _b2 = new long[n+2, m+2];
        _b3 = new long[n+2, m+2]; _b4 = new long[n+2, m+2];
    }

    private void Add(long[,] B, int x, int y, long v)
    {
        for (int i = x; i <= _n; i += i & -i)
            for (int j = y; j <= _m; j += j & -j)
                B[i,j] += v;
    }
    private long Sum(long[,] B, int x, int y)
    {
        long s = 0;
        for (int i = x; i > 0; i -= i & -i)
            for (int j = y; j > 0; j -= j & -j)
                s += B[i,j];
        return s;
    }

    public void RangeAdd(int x1, int y1, int x2, int y2, long v)
    {
        Add(_b1,x1,y1,v);      Add(_b1,x1,y2+1,-v);
        Add(_b1,x2+1,y1,-v);   Add(_b1,x2+1,y2+1,v);

        Add(_b2,x1,y1,v*(x1-1));    Add(_b2,x1,y2+1,-v*(x1-1));
        Add(_b2,x2+1,y1,-v*x2);     Add(_b2,x2+1,y2+1,v*x2);

        Add(_b3,x1,y1,v*(y1-1));    Add(_b3,x1,y2+1,-v*y2);
        Add(_b3,x2+1,y1,-v*(y1-1)); Add(_b3,x2+1,y2+1,v*y2);

        Add(_b4,x1,y1,v*(x1-1)*(y1-1));
        Add(_b4,x1,y2+1,-v*(x1-1)*y2);
        Add(_b4,x2+1,y1,-v*x2*(y1-1));
        Add(_b4,x2+1,y2+1,v*x2*y2);
    }

    private long Prefix(int x, int y)
        => Sum(_b1,x,y)*x*y - Sum(_b2,x,y)*y - Sum(_b3,x,y)*x + Sum(_b4,x,y);

    public long RangeSum(int x1, int y1, int x2, int y2)
        => Prefix(x2,y2) - Prefix(x1-1,y2) - Prefix(x2,y1-1) + Prefix(x1-1,y1-1);

    public static void Main()
    {
        // PURQ demo
        var fw = new Fenwick2D(5,5);
        fw.Update(1,1,1); fw.Update(2,2,3); fw.Update(3,3,5); fw.Update(5,5,7);
        Console.WriteLine($"PURQ sum(1,1,5,5)={fw.Query(1,1,5,5)} (expect 16)");
        Console.WriteLine($"PURQ sum(2,2,3,3)={fw.Query(2,2,3,3)} (expect 8)");

        // RURQ demo
        var rq = new Fenwick2D_RURQ(4,4);
        rq.RangeAdd(1,1,4,4,1);  // all cells +1
        rq.RangeAdd(2,2,3,3,2);  // inner 2x2 +2
        Console.WriteLine($"RURQ sum(1,1,4,4)={rq.RangeSum(1,1,4,4)} (expect 24)");
        Console.WriteLine($"RURQ sum(2,2,3,3)={rq.RangeSum(2,2,3,3)} (expect 12)");
    }
}
```

---

## The 4-BIT RURQ Derivation

To understand why four BITs are needed for range-update range-query, expand the rectangle sum in terms of prefix sums. Let `P(x,y) = sum of a[i][j] for i≤x, j≤y`. After a range update `add(r1,c1,r2,c2,v)` the prefix sums change by:

```
ΔP(x,y) = v · |[1..x]∩[r1..r2]| · |[1..y]∩[c1..c2]|
```

Expanding this product for all four boundary cases produces four terms, each a product of `x` or a constant in `x`, times `y` or a constant in `y`. Maintaining one BIT per term gives the O(log n · log m) update and query:

```
P(x,y) = B1[x][y]·x·y  - B2[x][y]·y  - B3[x][y]·x  + B4[x][y]

where B1,B2,B3,B4 accumulate the four coefficient types
```

---

## Pitfalls

- **1-indexed only** — the BIT update/query loops use `i & -i` which gives 0 at `i=0`, causing an infinite loop. Always use 1-based indexing; shift your 0-based input by +1.
- **Rectangle query needs four prefix calls** — a single prefix call gives `[1..x][1..y]`; a rectangle `[x1..x2][y1..y2]` requires four prefix calls with inclusion-exclusion. Missing one term silently returns wrong sums.
- **RURQ `range_add` updates 16 BIT cells** — each of the four BITs receives four corner updates (4×4=16 total). Forgetting any corner or sign corrupts all subsequent prefix sums. Write each BIT's four lines together, not interleaved.
- **RURQ weights use pre-subtracted corners** — the B2 update at `(x1,y1)` uses weight `v*(x1-1)`, not `v*x1`. The `-1` is essential; using `x1` shifts the sums by one column worth. Same for B3 with `(y1-1)` and B4 with `(x1-1)*(y1-1)`.
- **Grid boundaries for RURQ** — the `range_add` writes to index `x2+1` and `y2+1`, so the BIT arrays must be sized `(n+2) × (m+2)`, not `(n+1) × (m+1)`. Out-of-bounds writes silently corrupt memory.
- **Overflow with large grids** — for an `n×m` grid with values up to `V` and up to `Q` updates, the maximum prefix sum is `n·m·V·Q`. Use `long long` (C++) or `long` (C#) throughout; `int` overflows for `n=m=1000`, `V=10^6`.

---

## Conclusion

The 2D Fenwick Tree is the **go-to structure for dynamic rectangle sum queries on a 2D grid**:

- The implementation is minimal — two nested loops with the bit-trick — making it the fastest constant-factor option in practice.
- Point update + range query (PURQ) needs one BIT and four prefix calls per rectangle query.
- Range update + range query (RURQ) needs four BITs with sixteen corner updates per rectangle update, but keeps the same O(log n · log m) complexity.
- For static data (no updates), a 2D prefix sum table answers rectangle queries in O(1) with O(nm) build — the Fenwick Tree is only needed when updates arrive online.

**Key takeaway:** the 2D BIT is two nested 1D BITs. Every concept from 1D — indexing, update propagation, prefix decomposition, inclusion-exclusion for range queries — applies unchanged in each dimension independently.
