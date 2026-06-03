# 2D Segment Tree

## Origin & Motivation

**2D Segment Tree** is a nested segment tree structure that answers rectangle range queries and supports point (or range) updates on a 2D grid. The outer segment tree covers rows; each of its nodes contains a complete inner segment tree covering columns for that row range. The structure generalises the 1D segment tree to two dimensions while preserving the O(log n · log m) query and update complexity.

**The problem it solves:** A 2D Fenwick Tree handles point updates and rectangle sum queries very efficiently but cannot support arbitrary aggregations (min, max, GCD, custom monoids) or lazy range updates in a clean way. The 2D Segment Tree handles all of these: **any associative operation** (sum, min, max, XOR, …) can be placed in the inner trees, and lazy propagation can be added to both levels for range update + range query.

**The key idea:** The outer segment tree partitions rows into O(log n) disjoint segments; each outer node `vx` owns an inner segment tree over columns that stores the aggregate of all cells in row-range `[lx..rx]` × `[1..m]`. A rectangle query decomposes into O(log n) outer nodes, and each inner query costs O(log m), giving O(log n · log m) total.

Complexity: **O(nm)** space, **O(nm log n)** build (merging columns at each outer level), **O(log n · log m)** point update and rectangle query.

---

## Where It Is Used

- 2D range minimum / maximum queries with point updates
- Persistent or fractional-cascading range trees (offline 2D queries)
- Competitive programming: rectangle aggregation with non-invertible operations (min/max where Fenwick doesn't work)
- Computational geometry: 2D stabbing queries, axis-aligned rectangle intersection
- Image processing with custom aggregation (e.g. count of pixels above threshold in a rectangle)

---

## Core Structure

### Two-Level Nesting

```
outer segment tree: covers rows [1..n]
    outer node vx covers row range [lx..rx]
    each outer node vx owns:
        inner segment tree: covers columns [1..m]
            inner node vy covers column range [ly..ry]
            stores aggregate of all a[i][j] for i in [lx..rx], j in [ly..ry]
```

The total number of inner nodes is `O(n log m)` per outer level times `O(log n)` outer levels = `O(nm log n / n · log n)` — or more simply, each of the `O(n)` cells contributes to `O(log n)` outer nodes and `O(log m)` inner nodes, giving `O(nm log n)` total nodes when inner trees are stored separately per outer node. For a static build this can be reduced to `O(nm)` by sharing column arrays.

### Simplified Implementation: Outer × Inner Arrays

The cleanest implementation stores, for each outer node `vx`, a flat array `rows[vx]` of size `4m` representing the inner segment tree for that outer node's column dimension:

```
rows[vx][inner_node] = aggregate over {a[i][j] : i in outer[vx], j in inner[inner_node]}
```

Updates push through the outer tree (visiting O(log n) outer nodes) and within each outer node push through the inner tree (O(log m) inner nodes). Total: O(log n · log m).

---

## Operations

### Point Update

```
update(x, y, val):
    update_outer(vx=1, lx=1, rx=n, x, y, val)

update_outer(vx, lx, rx, x, y, val):
    update_inner(rows[vx], vy=1, ly=1, ry=m, y, val)   // update this outer node's inner tree
    if lx == rx: return                                   // leaf outer node
    mid = (lx+rx)/2
    if x <= mid: update_outer(2*vx,   lx,  mid, x, y, val)
    else:        update_outer(2*vx+1, mid+1,rx, x, y, val)

update_inner(row, vy, ly, ry, y, val):
    row[vy] += val                                        // update this inner node
    if ly == ry: return
    mid = (ly+ry)/2
    if y <= mid: update_inner(row, 2*vy,   ly,  mid, y, val)
    else:        update_inner(row, 2*vy+1, mid+1,ry, y, val)
    row[vy] = row[2*vy] + row[2*vy+1]                   // pull
```

### Rectangle Query

```
query(x1, y1, x2, y2):
    return query_outer(vx=1, lx=1, rx=n, x1, x2, y1, y2)

query_outer(vx, lx, rx, ql, qr, yql, yqr):
    if ql > rx or qr < lx: return 0                     // out of range
    if ql <= lx and rx <= qr:                            // fully covered
        return query_inner(rows[vx], 1, 1, m, yql, yqr)
    mid = (lx+rx)/2
    return query_outer(2*vx,   lx,  mid, ql, qr, yql, yqr)
         + query_outer(2*vx+1, mid+1,rx, ql, qr, yql, yqr)

query_inner(row, vy, ly, ry, ql, qr):
    if ql > ry or qr < ly: return 0
    if ql <= ly and ry <= qr: return row[vy]
    mid = (ly+ry)/2
    return query_inner(row, 2*vy,   ly,  mid, ql, qr)
         + query_inner(row, 2*vy+1, mid+1,ry, ql, qr)
```

---

## Complexity Summary

| Operation | Time | Space |
|---|---|---|
| Build (all zeros) | O(nm) | O(nm) per outer level × O(log n) = O(nm log n) simplified to O(nm) with flat arrays |
| Point update | O(log n · log m) | — |
| Rectangle query | O(log n · log m) | — |
| Range update + range query (with lazy) | O(log n · log m) | O(nm log n) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// 2D SEGMENT TREE — Point Update, Rectangle Sum Query
// Outer tree covers rows [1..n], inner trees cover columns [1..m].
// Each outer node vx has its own inner segment tree: rows[vx][1..4m].
// ================================================================
struct SegTree2D {
    int n, m;
    vector<vector<ll>> rows; // rows[outer_node][inner_seg_node]

    SegTree2D(int n, int m) : n(n), m(m) {}

    // Must call before any updates/queries
    void init() { rows.assign(4*n+4, vector<ll>(4*m+4, 0)); }

    // ---- Inner (column) segment tree ----
    void inner_update(vector<ll>& row, int vy, int ly, int ry, int y, ll val) {
        if (ly == ry) { row[vy] += val; return; }
        int mid = (ly + ry) / 2;
        if (y <= mid) inner_update(row, 2*vy,   ly,   mid, y, val);
        else          inner_update(row, 2*vy+1, mid+1, ry, y, val);
        row[vy] = row[2*vy] + row[2*vy+1];
    }

    ll inner_query(vector<ll>& row, int vy, int ly, int ry, int ql, int qr) {
        if (ql > ry || qr < ly) return 0;
        if (ql <= ly && ry <= qr) return row[vy];
        int mid = (ly + ry) / 2;
        return inner_query(row, 2*vy,   ly,   mid, ql, qr)
             + inner_query(row, 2*vy+1, mid+1, ry, ql, qr);
    }

    // ---- Outer (row) segment tree ----
    void outer_update(int vx, int lx, int rx, int x, int y, ll val) {
        inner_update(rows[vx], 1, 1, m, y, val); // update this outer node's inner tree
        if (lx == rx) return;
        int mid = (lx + rx) / 2;
        if (x <= mid) outer_update(2*vx,   lx,   mid, x, y, val);
        else          outer_update(2*vx+1, mid+1, rx,  x, y, val);
    }

    ll outer_query(int vx, int lx, int rx, int ql, int qr, int yql, int yqr) {
        if (ql > rx || qr < lx) return 0;
        if (ql <= lx && rx <= qr)
            return inner_query(rows[vx], 1, 1, m, yql, yqr);
        int mid = (lx + rx) / 2;
        return outer_query(2*vx,   lx,   mid, ql, qr, yql, yqr)
             + outer_query(2*vx+1, mid+1, rx,  ql, qr, yql, yqr);
    }

    // ---- Public API ----
    void update(int x, int y, ll val) {
        outer_update(1, 1, n, x, y, val);
    }

    ll query(int x1, int y1, int x2, int y2) {
        return outer_query(1, 1, n, x1, x2, y1, y2);
    }
};

// ================================================================
// 2D SEGMENT TREE — Min query variant
// Swap sum → min in inner_query and the pull line.
// ================================================================
struct SegTree2D_Min {
    int n, m;
    vector<vector<ll>> rows;
    static const ll INF = 1e18;

    SegTree2D_Min(int n, int m) : n(n), m(m) {}
    void init() { rows.assign(4*n+4, vector<ll>(4*m+4, INF)); }

    void inner_update(vector<ll>& row, int vy, int ly, int ry, int y, ll val) {
        if (ly == ry) { row[vy] = val; return; }
        int mid = (ly + ry) / 2;
        if (y <= mid) inner_update(row, 2*vy,   ly,   mid, y, val);
        else          inner_update(row, 2*vy+1, mid+1, ry, y, val);
        row[vy] = min(row[2*vy], row[2*vy+1]);
    }

    ll inner_query(vector<ll>& row, int vy, int ly, int ry, int ql, int qr) {
        if (ql > ry || qr < ly) return INF;
        if (ql <= ly && ry <= qr) return row[vy];
        int mid = (ly + ry) / 2;
        return min(inner_query(row, 2*vy,   ly,   mid, ql, qr),
                   inner_query(row, 2*vy+1, mid+1, ry, ql, qr));
    }

    void outer_update(int vx, int lx, int rx, int x, int y, ll val) {
        inner_update(rows[vx], 1, 1, m, y, val);
        if (lx == rx) return;
        int mid = (lx + rx) / 2;
        if (x <= mid) outer_update(2*vx,   lx,   mid, x, y, val);
        else          outer_update(2*vx+1, mid+1, rx,  x, y, val);
    }

    ll outer_query(int vx, int lx, int rx, int ql, int qr, int yql, int yqr) {
        if (ql > rx || qr < lx) return INF;
        if (ql <= lx && rx <= qr)
            return inner_query(rows[vx], 1, 1, m, yql, yqr);
        int mid = (lx + rx) / 2;
        return min(outer_query(2*vx,   lx,   mid, ql, qr, yql, yqr),
                   outer_query(2*vx+1, mid+1, rx,  ql, qr, yql, yqr));
    }

    void update(int x, int y, ll val) { outer_update(1, 1, n, x, y, val); }
    ll   query(int x1, int y1, int x2, int y2) { return outer_query(1,1,n,x1,x2,y1,y2); }
};

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- Sum demo ----
    {
        printf("=== 2D Segment Tree — Sum ===\n");
        SegTree2D st(5, 5); st.init();
        st.update(1,1,1); st.update(2,2,3);
        st.update(3,3,5); st.update(5,5,7);
        printf("query(1,1,5,5)=%lld  (expect 16)\n", st.query(1,1,5,5));
        printf("query(2,2,3,3)=%lld  (expect 8)\n",  st.query(2,2,3,3));
        printf("query(1,1,1,1)=%lld  (expect 1)\n",  st.query(1,1,1,1));
    }

    // ---- Min demo ----
    {
        printf("\n=== 2D Segment Tree — Min ===\n");
        SegTree2D_Min st(4, 4); st.init();
        st.update(1,1,9); st.update(2,3,4);
        st.update(3,2,7); st.update(4,4,2);
        printf("min(1,1,4,4)=%lld  (expect 2)\n", st.query(1,1,4,4));
        printf("min(1,1,3,3)=%lld  (expect 4)\n", st.query(1,1,3,3));
        printf("min(1,1,1,1)=%lld  (expect 9)\n", st.query(1,1,1,1));
    }

    // ---- Stress: Sum ----
    {
        srand(42); int errors = 0;
        for (int trial = 0; trial < 300; trial++) {
            int n=15, m=15;
            SegTree2D st(n,m); st.init();
            vector<vector<ll>> grid(n+1, vector<ll>(m+1, 0));
            for (int i = 0; i < 50; i++) {
                int x=rand()%n+1, y=rand()%m+1; ll v=rand()%20-5;
                st.update(x,y,v); grid[x][y]+=v;
            }
            for (int q = 0; q < 50; q++) {
                int x1=rand()%n+1,y1=rand()%m+1;
                int x2=rand()%n+1,y2=rand()%m+1;
                if(x1>x2)swap(x1,x2); if(y1>y2)swap(y1,y2);
                ll brute=0;
                for(int i=x1;i<=x2;i++) for(int j=y1;j<=y2;j++) brute+=grid[i][j];
                if(st.query(x1,y1,x2,y2)!=brute) errors++;
            }
        }
        printf("\nSum stress 300 trials: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Stress: Min ----
    {
        srand(77); int errors = 0;
        for (int trial = 0; trial < 300; trial++) {
            int n=15, m=15;
            SegTree2D_Min st(n,m); st.init();
            vector<vector<ll>> grid(n+1, vector<ll>(m+1, 1e17));
            for (int i = 0; i < 50; i++) {
                int x=rand()%n+1, y=rand()%m+1; ll v=rand()%100;
                st.update(x,y,v); grid[x][y]=v;
            }
            for (int q = 0; q < 50; q++) {
                int x1=rand()%n+1,y1=rand()%m+1;
                int x2=rand()%n+1,y2=rand()%m+1;
                if(x1>x2)swap(x1,x2); if(y1>y2)swap(y1,y2);
                ll brute=1e17;
                for(int i=x1;i<=x2;i++) for(int j=y1;j<=y2;j++) brute=min(brute,grid[i][j]);
                if(st.query(x1,y1,x2,y2)!=brute) errors++;
            }
        }
        printf("Min  stress 300 trials: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

// ================================================================
// 2D SEGMENT TREE — C#, Point Update, Rectangle Sum Query
// ================================================================
public class SegTree2D
{
    private readonly int _n, _m;
    private readonly long[][] _rows; // _rows[outer_node][inner_node]

    public SegTree2D(int n, int m)
    {
        _n = n; _m = m;
        _rows = new long[4*n+4][];
        for (int i = 0; i < _rows.Length; i++)
            _rows[i] = new long[4*m+4];
    }

    // ---- Inner segment tree ----
    private void InnerUpdate(long[] row, int vy, int ly, int ry, int y, long val)
    {
        if (ly == ry) { row[vy] += val; return; }
        int mid = (ly+ry)/2;
        if (y <= mid) InnerUpdate(row, 2*vy,   ly,   mid, y, val);
        else          InnerUpdate(row, 2*vy+1, mid+1, ry, y, val);
        row[vy] = row[2*vy] + row[2*vy+1];
    }

    private long InnerQuery(long[] row, int vy, int ly, int ry, int ql, int qr)
    {
        if (ql > ry || qr < ly) return 0;
        if (ql <= ly && ry <= qr) return row[vy];
        int mid = (ly+ry)/2;
        return InnerQuery(row, 2*vy,   ly,   mid, ql, qr)
             + InnerQuery(row, 2*vy+1, mid+1, ry, ql, qr);
    }

    // ---- Outer segment tree ----
    private void OuterUpdate(int vx, int lx, int rx, int x, int y, long val)
    {
        InnerUpdate(_rows[vx], 1, 1, _m, y, val);
        if (lx == rx) return;
        int mid = (lx+rx)/2;
        if (x <= mid) OuterUpdate(2*vx,   lx,   mid, x, y, val);
        else          OuterUpdate(2*vx+1, mid+1, rx,  x, y, val);
    }

    private long OuterQuery(int vx, int lx, int rx, int ql, int qr, int yql, int yqr)
    {
        if (ql > rx || qr < lx) return 0;
        if (ql <= lx && rx <= qr)
            return InnerQuery(_rows[vx], 1, 1, _m, yql, yqr);
        int mid = (lx+rx)/2;
        return OuterQuery(2*vx,   lx,   mid, ql, qr, yql, yqr)
             + OuterQuery(2*vx+1, mid+1, rx,  ql, qr, yql, yqr);
    }

    public void Update(int x, int y, long val) => OuterUpdate(1,1,_n,x,y,val);
    public long Query(int x1, int y1, int x2, int y2) => OuterQuery(1,1,_n,x1,x2,y1,y2);

    public static void Main()
    {
        var st = new SegTree2D(5,5);
        st.Update(1,1,1); st.Update(2,2,3); st.Update(3,3,5); st.Update(5,5,7);
        Console.WriteLine($"sum(1,1,5,5)={st.Query(1,1,5,5)} (expect 16)");
        Console.WriteLine($"sum(2,2,3,3)={st.Query(2,2,3,3)} (expect 8)");
        Console.WriteLine($"sum(1,1,1,1)={st.Query(1,1,1,1)} (expect 1)");
    }
}
```

---

## 2D Segment Tree vs 2D Fenwick Tree

| Property | 2D Fenwick Tree | 2D Segment Tree |
|---|---|---|
| Code size | ~20 lines | ~60 lines |
| Constant factor | Very small | Moderate |
| Supported operations | Invertible only (sum, XOR) | Any monoid (min, max, GCD, …) |
| Range update + range query | Possible with 4 BITs (sum only) | Possible with lazy propagation |
| Memory | O(nm) | O(nm log n) with separate inner trees |
| Build from existing array | O(nm) | O(nm log n) |
| Best for | Competitive programming, sum queries | Non-invertible ops, more flexibility |

---

## Pitfalls

- **Inner tree must be re-initialised for each outer node** — when using separate `rows[vx]` arrays, all outer nodes share the same column range `[1..m]` but completely independent data. Never share or alias inner arrays between different outer nodes.
- **Outer update visits O(log n) outer nodes, not just one** — unlike a 2D Fenwick Tree which only updates cells along the BIT chain, the 2D segment tree `outer_update` updates the inner tree of every outer ancestor of `x` (O(log n) of them). This is intentional — each outer node stores the aggregate for its entire row range, so all ancestors must reflect the change.
- **Inner pull must happen after recursive calls** — after updating left or right child of an inner node, `row[vy] = row[2*vy] + row[2*vy+1]` must be called on the way back up. Forgetting the pull leaves ancestors stale and makes queries return wrong totals.
- **Memory scales with O(nm log n)** — storing a full 4m-size inner tree per outer node means `4n * 4m = 16nm` cells total for the simplified version, or up to `4n log n * 4m` for a proper outer tree. For `n=m=1000` this is 16 million longs (128 MB) — manageable, but pre-compute sizes before allocating.
- **Outer query passes the ROW range to inner but column range stays fixed** — the outer query decomposes by rows (`ql..qr`) and passes the full column query `(yql, yqr)` unchanged to the inner tree. Passing row bounds into the inner tree instead is a common mistake that produces wrong results.
- **Min initialisation must use +∞, not 0** — for a min-segment tree the identity element is `+∞` (no element), not 0. Initialising all cells to 0 makes every min query return 0 even for regions containing no updates. Use `numeric_limits<ll>::max()` or a large constant.
- **1-indexed externally, internally consistent** — this implementation uses 1-based indexing for both dimensions. Passing 0-based coordinates off by one is the most common runtime bug — wrap the public API if your input is 0-based.

---

## Conclusion

The 2D Segment Tree is the **most flexible 2D range structure**:

- By nesting two segment trees it inherits all the power of the 1D version: any associative operation works as the aggregate, lazy propagation can be added for range updates, and persistent variants are possible.
- The trade-off versus the 2D Fenwick Tree is code complexity and memory: O(nm log n) space instead of O(nm), and a larger constant factor per operation.
- For sum queries with point updates, the 2D Fenwick Tree is strictly better in practice. Choose the 2D Segment Tree when you need min/max/GCD, range updates with lazy tags, or a non-invertible monoid.

**Key takeaway:** the outer tree determines which row-range nodes are visited (O(log n)); for each such node the inner tree answers a column-range query in O(log m). Every update must propagate to all O(log n) outer ancestors so their inner trees remain consistent — this is the fundamental difference from the 1D case and the source of the O(log n · log m) complexity.
