# Closest Pair of Points (Divide & Conquer)

## Origin & Motivation

**Closest Pair of Points** asks: given `n` points in the plane, find the two with minimum Euclidean distance. The brute-force approach checks all O(n²) pairs. The **Divide and Conquer** algorithm by Shamos and Hoey (1975) achieves **O(n log n)** — optimal under the algebraic decision-tree model.

**The problem it solves:** Collision proximity detection, clustering initialization, nearest-neighbor search, and computational geometry preprocessing all need the closest pair. With n = 10⁶ points, O(n²) is 10¹² operations — infeasible. O(n log n) is 20 × 10⁶ — instant.

**The key idea:** Split the point set by median x-coordinate. Recursively find the closest pair in each half. The answer is either entirely within one half, or it crosses the dividing line. The crucial insight is that any crossing pair must lie in a strip of width `2d` around the midline (where `d` is the minimum from both halves) and within that strip the **7-point rule** guarantees at most 7 comparisons per point — making the strip check O(n) per level.

Complexity: **O(n log n)** time, **O(n)** space.

---

## Where It Is Used

- Computational geometry: preprocessing for Delaunay triangulation, Voronoi diagrams
- Collision detection: find the two closest objects in a simulation
- Clustering: initialization for k-means (find well-separated seeds)
- Network design: find closest pair of nodes as a starting edge
- Genomics: closest pair in high-dimensional sequence space (reduced to 2D projections)
- Competitive programming: classic divide-and-conquer template problem

---

## Core Structure

### Divide Step

Split `n` points at median x-coordinate into left half `L` and right half `R`:

```
Points sorted by x: [p0, p1, ..., p_{n/2-1} | p_{n/2}, ..., p_{n-1}]
                                               ↑ midline x = pts[n/2].x

d_L = ClosestPair(L)   // recursion on left half
d_R = ClosestPair(R)   // recursion on right half
d   = min(d_L, d_R)    // best within either half
```

### Conquer Step — The Strip

Any pair that crosses the midline and has distance < `d` must have both points within horizontal distance `d` of the midline. Collect the **strip**:

```
strip = { p : |p.x - midline.x| < d }
```

Within the strip, sort points by y. For each point in the strip, check only points within vertical distance `d` above it. The **7-point lemma** proves at most 7 such points exist:

```
Why at most 7 comparisons per strip point:
  The d×2d rectangle around any strip point (width 2d, height d)
  can contain at most 8 points total (4 on each side of midline)
  with pairwise distance ≥ d (by definition of d_L and d_R).
  In practice the bound is 7 (the point itself is one of the 8).
```

This makes the strip check O(n) per level → total O(n log n).

### Merge Step

After recursion, both halves are y-sorted (maintained via merge-sort style merge). This allows the strip to be built from an already-y-sorted array in O(n) rather than O(n log n).

---

## Algorithm

```
ClosestPair(pts[lo..hi]):                        // pts sorted by x initially
    if hi - lo <= 3:
        brute_force(pts[lo..hi])                 // O(1) base case
        sort pts[lo..hi] by y                    // prepare for merge
        return min distance found

    mid = (lo + hi) / 2
    mx  = pts[mid].x                             // midline x-coordinate

    d_L = ClosestPair(pts[lo..mid])              // left half
    d_R = ClosestPair(pts[mid..hi])              // right half
    d   = min(d_L, d_R)

    merge pts[lo..mid] and pts[mid..hi] by y     // O(n) merge-sort step

    // Build strip: points within d of midline, already y-sorted
    strip = [ p for p in pts[lo..hi] if |p.x - mx| < d ]

    // Check strip: at most 7 comparisons per point
    for i in 0..len(strip)-1:
        for j in i+1..len(strip)-1:
            if strip[j].y - strip[i].y >= d: break   // 7-point bound
            d = min(d, dist(strip[i], strip[j]))

    return d
```

The y-sort is maintained via a merge step on the way back up (like merge sort), so each level costs O(n) for the merge + O(n) for the strip check = O(n) per level × O(log n) levels = **O(n log n)**.

---

## The 7-Point Lemma (Strip Bound)

Consider a `2d × d` rectangle (width `2d` centered on midline, height `d`). Divide it into 8 cells of size `d/2 × d/2`:

```
|---d---|---d---|
|  1   |  5   |  ← height d/2
|  2   |  6   |
|  3   |  7   |  ← height d/2
|  4   |  8   |
```

Each cell has diagonal `d/2 · √2 = d/√2 < d`. Since all points in the left half have pairwise distance ≥ `d_L ≥ d`, at most one point from the left half fits in each left cell. Similarly for the right. So at most **8 points total** can fit in the rectangle — including the query point itself, at most **7 other points** need to be checked.

In the y-sorted strip, the inner loop breaks as soon as `strip[j].y - strip[i].y ≥ d` — this is exactly the rectangle height bound.

---

## Complexity Summary

| Algorithm | Time | Space |
|---|---|---|
| Brute force | O(n²) | O(1) |
| Divide & Conquer | O(n log n) | O(n) |
| Randomized incremental | O(n) expected | O(n) |

The D&C version is the standard deterministic optimal algorithm. The O(n) randomized version exists but has larger constants and is rarely used in practice.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ld = long double;

struct P {
    ld x, y;
    int id; // original index
};

ld dist(P a, P b) {
    return sqrtl((a.x-b.x)*(a.x-b.x) + (a.y-b.y)*(a.y-b.y));
}

// Global best pair (updated during recursion)
pair<P,P> best_pair;
ld        best_dist;

// ================================================================
// CLOSEST PAIR — Divide & Conquer O(n log n)
// pts[lo..hi) must be sorted by x before the first call.
// After each call, pts[lo..hi) is sorted by y (merge-sort style).
// tmp: scratch array of same size as pts.
// ================================================================
ld closest(vector<P>& pts, vector<P>& tmp, int lo, int hi) {
    // Base case: brute force 2 or 3 points
    if (hi - lo <= 3) {
        ld d = 1e18;
        for (int i = lo; i < hi; i++)
            for (int j = i+1; j < hi; j++) {
                ld dd = dist(pts[i], pts[j]);
                if (dd < d) {
                    d = dd;
                    if (dd < best_dist) { best_dist = dd; best_pair = {pts[i], pts[j]}; }
                }
            }
        // Sort this small range by y for the merge on the way back up
        sort(pts.begin()+lo, pts.begin()+hi, [](P a, P b){ return a.y < b.y; });
        return d;
    }

    int mid = (lo + hi) / 2;
    ld mx = pts[mid].x; // midline x-coordinate

    ld d = min(closest(pts, tmp, lo, mid),
               closest(pts, tmp, mid, hi));

    // Merge both halves by y (both already y-sorted from recursion)
    merge(pts.begin()+lo,  pts.begin()+mid,
          pts.begin()+mid, pts.begin()+hi,
          tmp.begin()+lo,
          [](P a, P b){ return a.y < b.y; });
    copy(tmp.begin()+lo, tmp.begin()+hi, pts.begin()+lo);

    // Build strip: points within horizontal distance d of midline
    // pts[lo..hi) is now y-sorted, so strip is also y-sorted
    vector<P> strip;
    for (int i = lo; i < hi; i++)
        if (fabsl(pts[i].x - mx) < d)
            strip.push_back(pts[i]);

    // Check strip: at most 7 comparisons per point (7-point lemma)
    int s = strip.size();
    for (int i = 0; i < s; i++)
        for (int j = i+1; j < s && strip[j].y - strip[i].y < d; j++) {
            ld dd = dist(strip[i], strip[j]);
            if (dd < d) {
                d = dd;
                if (dd < best_dist) { best_dist = dd; best_pair = {strip[i], strip[j]}; }
            }
        }

    return d;
}

// ================================================================
// PUBLIC API
// Returns {pair of closest points, distance}
// ================================================================
struct Result { P a, b; ld distance; };

Result closest_pair(vector<P> pts) {
    int n = pts.size();
    if (n < 2) return {{},{},1e18};
    sort(pts.begin(), pts.end(), [](P a, P b){
        return a.x < b.x || (a.x == b.x && a.y < b.y);
    });
    best_dist = 1e18;
    best_pair = {};
    vector<P> tmp(n);
    closest(pts, tmp, 0, n);
    return {best_pair.first, best_pair.second, best_dist};
}

// ================================================================
// Brute force O(n²) for verification
// ================================================================
Result brute_force(const vector<P>& pts) {
    ld bd = 1e18; P ba={}, bb={};
    int n = pts.size();
    for (int i = 0; i < n; i++)
        for (int j = i+1; j < n; j++) {
            ld d = dist(pts[i], pts[j]);
            if (d < bd) { bd=d; ba=pts[i]; bb=pts[j]; }
        }
    return {ba, bb, bd};
}

// ================================================================
// Usage + stress test
// ================================================================
int main() {
    // ---- Demo ----
    {
        printf("=== Closest Pair Demo ===\n");
        vector<P> pts = {{2,3,0},{12,30,1},{40,50,2},{5,1,3},{12,10,4},{3,4,5}};
        auto r  = closest_pair(pts);
        auto r2 = brute_force(pts);
        printf("D&C:   (%.1Lf,%.1Lf)-(%.1Lf,%.1Lf)  dist=%.6Lf\n",
               r.a.x,r.a.y,r.b.x,r.b.y,r.distance);
        printf("Brute: (%.1Lf,%.1Lf)-(%.1Lf,%.1Lf)  dist=%.6Lf\n",
               r2.a.x,r2.a.y,r2.b.x,r2.b.y,r2.distance);
    }

    // ---- Edge cases ----
    {
        printf("\n=== Edge Cases ===\n");
        // Collinear points
        vector<P> col = {{0,0,0},{1,0,1},{2,0,2},{3,0,3},{4,0,4}};
        auto r = closest_pair(col);
        printf("Collinear: dist=%.4Lf (expect 1.0)\n", r.distance);

        // Duplicate points
        vector<P> dup = {{1,1,0},{1,1,1},{5,5,2}};
        r = closest_pair(dup);
        printf("Duplicates: dist=%.4Lf (expect 0.0)\n", r.distance);

        // Two points
        vector<P> two = {{0,0,0},{3,4,1}};
        r = closest_pair(two);
        printf("Two points: dist=%.4Lf (expect 5.0)\n", r.distance);
    }

    // ---- Stress test ----
    {
        printf("\n=== Stress Test 2000 trials ===\n");
        srand(42); int errors = 0;
        for (int t = 0; t < 2000; t++) {
            int n = 2 + rand()%30;
            vector<P> pts(n);
            for (int i = 0; i < n; i++)
                pts[i] = {ld(rand()%200 - 100), ld(rand()%200 - 100), i};
            auto r1 = closest_pair(pts);
            auto r2 = brute_force(pts);
            if (fabsl(r1.distance - r2.distance) > 1e-6) errors++;
        }
        printf("Result: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Performance ----
    {
        printf("\n=== Performance n=1,000,000 ===\n");
        srand(99);
        int n = 1000000;
        vector<P> pts(n);
        for (int i = 0; i < n; i++)
            pts[i] = {ld(rand()%1000000), ld(rand()%1000000), i};
        auto t0 = chrono::high_resolution_clock::now();
        auto r = closest_pair(pts);
        auto t1 = chrono::high_resolution_clock::now();
        double ms = chrono::duration<double,milli>(t1-t0).count();
        printf("n=1M: dist=%.4Lf  time=%.1f ms\n", r.distance, ms);
    }
    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class ClosestPair
{
    public struct Pt { public double X, Y; public int Id; }

    static double Dist(Pt a, Pt b) =>
        Math.Sqrt((a.X-b.X)*(a.X-b.X) + (a.Y-b.Y)*(a.Y-b.Y));

    public struct Result { public Pt A, B; public double Distance; }

    static Pt _bestA, _bestB;
    static double _bestDist;

    // ================================================================
    // CLOSEST PAIR — Divide & Conquer O(n log n)
    // ================================================================
    static double Closest(Pt[] pts, Pt[] tmp, int lo, int hi)
    {
        if (hi - lo <= 3)
        {
            double d = double.MaxValue;
            for (int i = lo; i < hi; i++)
                for (int j = i+1; j < hi; j++) {
                    double dd = Dist(pts[i], pts[j]);
                    if (dd < d) { d=dd; if (dd<_bestDist){_bestDist=dd;_bestA=pts[i];_bestB=pts[j];} }
                }
            Array.Sort(pts, lo, hi-lo, Comparer<Pt>.Create((a,b)=>a.Y.CompareTo(b.Y)));
            return d;
        }

        int mid = (lo+hi)/2;
        double mx = pts[mid].X;

        double dist = Math.Min(Closest(pts,tmp,lo,mid), Closest(pts,tmp,mid,hi));

        // Merge by Y
        int i1=lo, i2=mid, k=lo;
        while (i1<mid && i2<hi)
            tmp[k++] = pts[i1].Y <= pts[i2].Y ? pts[i1++] : pts[i2++];
        while (i1<mid) tmp[k++]=pts[i1++];
        while (i2<hi)  tmp[k++]=pts[i2++];
        Array.Copy(tmp, lo, pts, lo, hi-lo);

        // Strip check
        var strip = new List<Pt>();
        for (int i=lo; i<hi; i++)
            if (Math.Abs(pts[i].X - mx) < dist) strip.Add(pts[i]);

        int s = strip.Count;
        for (int i=0; i<s; i++)
            for (int j=i+1; j<s && strip[j].Y-strip[i].Y<dist; j++) {
                double dd = Dist(strip[i], strip[j]);
                if (dd < dist) {
                    dist = dd;
                    if (dd < _bestDist) { _bestDist=dd; _bestA=strip[i]; _bestB=strip[j]; }
                }
            }
        return dist;
    }

    public static Result Solve(Pt[] input)
    {
        var pts = (Pt[])input.Clone();
        Array.Sort(pts, (a,b) => a.X!=b.X ? a.X.CompareTo(b.X) : a.Y.CompareTo(b.Y));
        _bestDist = double.MaxValue;
        var tmp = new Pt[pts.Length];
        Closest(pts, tmp, 0, pts.Length);
        return new Result { A=_bestA, B=_bestB, Distance=_bestDist };
    }

    // Brute force O(n²)
    public static Result BruteForce(Pt[] pts)
    {
        double bd = double.MaxValue; Pt ba=default, bb=default;
        for (int i=0; i<pts.Length; i++)
            for (int j=i+1; j<pts.Length; j++) {
                double d=Dist(pts[i],pts[j]);
                if (d<bd){bd=d;ba=pts[i];bb=pts[j];}
            }
        return new Result{A=ba,B=bb,Distance=bd};
    }

    public static void Main()
    {
        var pts = new Pt[]{ new(){X=2,Y=3,Id=0}, new(){X=12,Y=30,Id=1},
                            new(){X=40,Y=50,Id=2}, new(){X=5,Y=1,Id=3},
                            new(){X=12,Y=10,Id=4}, new(){X=3,Y=4,Id=5} };
        var r  = Solve(pts);
        var r2 = BruteForce(pts);
        Console.WriteLine($"D&C:   ({r.A.X},{r.A.Y})-({r.B.X},{r.B.Y})  dist={r.Distance:F6}");
        Console.WriteLine($"Brute: ({r2.A.X},{r2.A.Y})-({r2.B.X},{r2.B.Y})  dist={r2.Distance:F6}");
    }
}
```

---

## Why the Merge Step Matters

A naive implementation sorts the strip from scratch at each recursive level, making the strip check O(n log n) per level instead of O(n) — blowing the total to O(n log² n). The merge-sort style approach maintains y-order as a side effect of recursion:

```
After each recursive call, pts[lo..hi) is y-sorted.
The parent merges two y-sorted halves in O(n).
The strip is built from the y-sorted array in O(n).
→ Total work per level: O(n). Levels: O(log n). Total: O(n log n).

Naive (re-sort strip each time):
→ Strip sort per level: O(n log n). Levels: O(log n). Total: O(n log² n).
```

Both O(n log n) and O(n log² n) pass for n ≤ 10⁵ in competitive programming, but the merge version is the correct theoretical O(n log n).

---

## Pitfalls

- **Base case must sort by y** — the recursion relies on both halves being y-sorted when it merges them on the way back up. If the base case does not y-sort its small range, the merge produces a wrong y-order and the strip check silently misses pairs.
- **Strip condition is strict: `< d`, not `<= d`** — the inner loop breaks when `strip[j].y - strip[i].y >= d`. Using `>` instead of `>=` allows an extra comparison that is guaranteed to fail (distance ≥ d by definition) but more importantly changes the termination condition, potentially missing degenerate cases where the minimum is achieved exactly at distance `d`.
- **Midline x from `pts[mid].x`, not average** — the midline must be exactly the x-coordinate that splits the sorted array, not an average or computed value. Using an average can place points in the wrong half and corrupt the strip width calculation.
- **Thread-unsafe global best** — the reference implementation uses global `best_dist` and `best_pair`. This is not thread-safe. For concurrent use, pass the best by reference or return it from the recursive function.
- **Integer overflow in squared distance** — if using integer coordinates and squared distances for comparison (to avoid `sqrt`), coordinates up to 10⁶ give squared distances up to 2×10¹² — overflows `int` and `unsigned int`. Use `long long` or `__int128` for squared distance comparisons.
- **Duplicate points give distance 0** — the algorithm correctly handles duplicates (two identical points have distance 0 and will be found). If the application requires distinct points, filter duplicates before calling.
- **Global state between calls** — `best_dist` is global and must be reset to `1e18` before each call. Forgetting to reset causes subsequent calls to use stale best distance from a previous run, potentially returning wrong results.

---

## Conclusion

Closest Pair of Points is the **canonical divide-and-conquer geometry algorithm**:

- Splitting by median x, recursing on each half, and checking only a thin strip around the midline reduces the problem from O(n²) to O(n log n).
- The 7-point lemma is the core geometric insight: within a `2d × d` strip rectangle, packing constraints from both halves limit the number of candidate neighbors to a constant — making the strip scan O(n) per level.
- The merge-sort style y-sort maintenance is the implementation detail that achieves the true O(n log n) bound rather than O(n log² n).

**Key takeaway:** after sorting by x once, the recursion simultaneously solves the subproblems and maintains a y-sorted order via merging. The strip check then costs only O(n) per level because it scans an already-sorted sequence with an early-termination condition derived from the current best distance.
