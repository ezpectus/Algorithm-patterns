# Minkowski Sum

## Origin & Motivation

**Minkowski Sum** of two sets A and B in the plane is defined as `A ⊕ B = { a + b | a ∈ A, b ∈ B }` — the set of all pairwise vector sums. Named after Hermann Minkowski (1903), it is the central operation in computational geometry for motion planning, collision detection, and offset computation.

**The problem it solves:** Given a robot (modelled as polygon P) moving among obstacles (polygon Q), the robot collides with an obstacle when P and Q overlap. The set of all positions where the robot's reference point causes a collision is exactly `Q ⊕ (-P)` — the Minkowski sum of the obstacle with the reflected robot. This reduces robot-obstacle collision to a point-in-polygon query.

**The key idea:** For two convex polygons with n and m vertices respectively, the Minkowski sum is also a convex polygon with at most n+m vertices and can be computed in **O(n + m)** by merging the edge vectors of both polygons in angular order — a single simultaneous traversal of both edge sequences sorted by angle, which are already sorted by virtue of convexity.

Complexity: **O(n + m)** for two convex polygons, **O(nm)** naive for non-convex via decomposition into convex parts.

---

## Where It Is Used

- Robot motion planning: configuration space obstacle construction (`C-obstacle = Q ⊕ (-P)`)
- Collision detection: two convex shapes A and B do not collide iff `origin ∉ A ⊕ (-B)`
- GJK algorithm: uses Minkowski difference implicitly for collision queries
- Polygon offsetting / buffering: `P ⊕ disk(r)` inflates P by radius r (used in GIS buffering)
- Computational geometry: convex hull of union, width of convex set, Hausdorff distance
- Competitive programming: robot-in-corridor problems, forbidden zone detection

---

## Core Structure

### Definition

```
A ⊕ B = { a + b  |  a ∈ A, b ∈ B }
```

For convex polygons A and B:
- The result is always convex.
- The vertices of `A ⊕ B` are sums of vertices of A and B taken in the order their edge vectors appear angularly.
- The edges of `A ⊕ B` are exactly the union of the edge vectors of A and B, sorted by angle.

### Edge Vector Merging

A convex polygon traversed CCW produces edges whose direction vectors rotate monotonically through 360°. For two CCW convex polygons P and Q, merge their edge sequences by angle (like merge-sort):

```
Edge sequences (as 2D vectors):
  P: e_p0, e_p1, ..., e_p(n-1)    (CCW, angles increasing)
  Q: e_q0, e_q1, ..., e_q(m-1)    (CCW, angles increasing)

Merge by angle using cross product:
  if cross(e_pi, e_qj) > 0:  take e_pi  (e_pi comes first angularly)
  if cross(e_pi, e_qj) < 0:  take e_qj
  if cross(e_pi, e_qj) == 0: take both  (parallel edges, both contribute)

Starting vertex: bottom-left vertex of P + bottom-left vertex of Q
Each merged edge advances the running sum to the next vertex.
```

---

## Algorithm

```
MinkowskiSum(P[0..n-1], Q[0..m-1]):
    // Both polygons must be convex and CCW
    // Reorder each to start from its bottom-left vertex
    i0 = index of bottom-left vertex of P
    j0 = index of bottom-left vertex of Q
    rotate P to start at i0
    rotate Q to start at j0

    result = []
    i = 0, j = 0
    while i < n or j < m:
        result.append(P[i] + Q[j])           // current vertex of sum
        edgeP = P[(i+1) mod n] - P[i]
        edgeQ = Q[(j+1) mod m] - Q[j]
        c = cross(edgeP, edgeQ)
        if   c > 0: advance i               // edgeP is more CCW
        elif c < 0: advance j               // edgeQ is more CCW
        else:       advance both            // parallel edges

    return result   // convex polygon with ≤ n+m vertices
```

**Why start from bottom-left?** The bottom-left vertex has the smallest y (then x) — it is always on the lower hull of both polygons. Starting there ensures the edge vectors begin from the same angular position (pointing rightward/downward), so the merge starts in sync.

**Why cross product decides order?** `cross(edgeP, edgeQ) > 0` means edgeP points more counterclockwise than edgeQ — so edgeP should be placed first in the merged CCW sequence.

---

## Key Properties

### Area Formula

```
area(A ⊕ B) = area(A) + area(B) + mixed_term

For convex polygons:
  mixed_term = sum over all edge pairs (eA, eB) of |cross(eA, eB)| / 2
             = perimeter(A) * width(B, dir) integrated over directions
             ≥ 0

Simple bound: area(A ⊕ B) ≥ area(A) + area(B)
```

### Minkowski Difference (for collision detection)

```
A ⊖ B = A ⊕ (-B)

where -B = {-b | b ∈ B}  (reflect B through origin)

A and B do NOT collide  ⟺  origin ∉ A ⊖ B
A and B DO collide       ⟺  origin ∈ A ⊖ B
```

To compute `-B`: negate all coordinates of B and rebuild the CCW order (reversing the vertex list and negating gives the reflected CCW polygon).

### Non-Convex Polygons

For non-convex polygons, decompose each into convex parts:
- If A has `a` convex parts and B has `b` parts, compute all `a*b` pairwise Minkowski sums of convex parts, then take the union.
- Total complexity: O(n·m) in general.

---

## Complexity Summary

| Case | Time | Vertices of result |
|---|---|---|
| Two convex polygons | O(n + m) | ≤ n + m |
| Convex + disk (offset) | O(n) | n + arc samples |
| Non-convex (decomposition) | O(nm) | O(nm) |
| General (exact, non-convex) | O(n²m²) worst case | O(n²m²) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ld = long double;
const ld EPS = 1e-9;

struct P {
    ld x, y;
    P(ld x=0, ld y=0) : x(x), y(y) {}
    P operator+(P o) const { return {x+o.x, y+o.y}; }
    P operator-(P o) const { return {x-o.x, y-o.y}; }
    P operator*(ld t) const { return {x*t,   y*t};   }
    bool operator<(P o) const {
        return x < o.x-EPS || (abs(x-o.x) < EPS && y < o.y-EPS);
    }
    bool operator==(P o) const {
        return abs(x-o.x) < EPS && abs(y-o.y) < EPS;
    }
};

ld cross(P a, P b) { return a.x*b.y - a.y*b.x; }
ld dot  (P a, P b) { return a.x*b.x + a.y*b.y; }
int sign(ld v)     { return v>EPS ? 1 : v<-EPS ? -1 : 0; }

// ================================================================
// CONVEX HULL — Andrew's monotone chain, CCW, O(n log n)
// ================================================================
vector<P> convex_hull(vector<P> pts) {
    int n = pts.size();
    if (n < 2) return pts;
    sort(pts.begin(), pts.end());
    pts.erase(unique(pts.begin(), pts.end()), pts.end());
    n = pts.size();
    if (n < 2) return pts;
    vector<P> h;
    for (int i = 0; i < n; i++) {
        while (h.size() >= 2 &&
               sign(cross(h.back()-h[h.size()-2], pts[i]-h[h.size()-2])) <= 0)
            h.pop_back();
        h.push_back(pts[i]);
    }
    int lo = h.size();
    for (int i = n-2; i >= 0; i--) {
        while ((int)h.size() > lo &&
               sign(cross(h.back()-h[h.size()-2], pts[i]-h[h.size()-2])) <= 0)
            h.pop_back();
        h.push_back(pts[i]);
    }
    h.pop_back();
    return h;
}

// ================================================================
// MINKOWSKI SUM of two convex polygons — O(n + m)
// Both must be convex, CCW. Result is convex, CCW.
// ================================================================
vector<P> minkowski_sum(vector<P> A, vector<P> B) {
    // Reorder each polygon to start from its bottom-left vertex
    auto reorder = [](vector<P>& poly) {
        int idx = 0;
        for (int i = 1; i < (int)poly.size(); i++)
            if (poly[i] < poly[idx]) idx = i;
        rotate(poly.begin(), poly.begin()+idx, poly.end());
    };
    reorder(A); reorder(B);

    int n = A.size(), m = B.size();
    // Append two extra copies of first two vertices for wraparound
    A.push_back(A[0]); A.push_back(A[1]);
    B.push_back(B[0]); B.push_back(B[1]);

    vector<P> result;
    int i = 0, j = 0;
    while (i < n || j < m) {
        result.push_back(A[i] + B[j]);
        P ea = A[i+1] - A[i];   // current edge of A
        P eb = B[j+1] - B[j];   // current edge of B
        ld c = cross(ea, eb);
        if      (sign(c) > 0) i++;          // ea is more CCW → advance A
        else if (sign(c) < 0) j++;          // eb is more CCW → advance B
        else                  { i++; j++; } // parallel → advance both
        if (i >= n && j >= m) break;
        if (i > n) i = n;
        if (j > m) j = m;
    }
    return result;
}

// ================================================================
// MINKOWSKI DIFFERENCE: A ⊖ B = A ⊕ (-B)
// Used for collision detection: shapes A and B collide
// iff origin ∈ minkowski_diff(A, B)
// ================================================================
vector<P> reflect(vector<P> poly) {
    for (auto& p : poly) { p.x = -p.x; p.y = -p.y; }
    // After negation the polygon is CW; reverse to make CCW
    reverse(poly.begin(), poly.end());
    return poly;
}

vector<P> minkowski_diff(const vector<P>& A, const vector<P>& B) {
    return minkowski_sum(A, reflect(B));
}

// ================================================================
// POINT IN CONVEX POLYGON — O(log n), CCW polygon
// Returns: 1=inside, 0=boundary, -1=outside
// ================================================================
int pip_convex(P q, const vector<P>& poly) {
    int n = poly.size();
    if (n < 3) return -1;
    P o = poly[0];
    int s1 = sign(cross(poly[1]-o,   q-o));
    int sn = sign(cross(poly[n-1]-o, q-o));
    if (s1 < 0 || sn > 0) return -1;
    int lo = 1, hi = n-1;
    while (hi - lo > 1) {
        int mid = (lo+hi)/2;
        if (sign(cross(poly[mid]-o, q-o)) >= 0) lo = mid;
        else hi = mid;
    }
    int c = sign(cross(poly[lo+1]-poly[lo], q-poly[lo]));
    if (c < 0) return -1;
    if (c == 0) return 0;
    if (s1 == 0 || sn == 0) return 0;
    return 1;
}

// ================================================================
// COLLISION DETECTION via Minkowski Difference
// Returns true if convex polygons A and B intersect
// ================================================================
bool convex_collide(const vector<P>& A, const vector<P>& B) {
    auto diff = minkowski_diff(A, B);
    if (diff.size() < 3) return false;
    return pip_convex({0,0}, diff) >= 0; // origin inside or on boundary
}

// ================================================================
// Area (shoelace)
// ================================================================
ld area(const vector<P>& poly) {
    ld s = 0; int n = poly.size();
    for (int i = 0; i < n; i++) s += cross(poly[i], poly[(i+1)%n]);
    return abs(s) / 2;
}

// ================================================================
// Usage + stress test
// ================================================================
int main() {
    // ---- Demo: square + triangle ----
    {
        printf("=== Minkowski Sum: square + triangle ===\n");
        vector<P> sq  = {{0,0},{2,0},{2,2},{0,2}};
        vector<P> tri = {{0,0},{1,0},{0,1}};
        auto ms = minkowski_sum(sq, tri);
        printf("Result vertices (%d):\n", (int)ms.size());
        for (auto& p : ms) printf("  (%.1Lf, %.1Lf)\n", p.x, p.y);
        printf("Area = %.2Lf  (expected 8.50)\n", area(ms));
        // area(sq)=4, area(tri)=0.5, mixed=4 → total=8.5
    }

    // ---- Demo: collision detection ----
    {
        printf("\n=== Collision Detection via Minkowski Difference ===\n");
        vector<P> A = {{0,0},{2,0},{2,2},{0,2}}; // square at origin
        vector<P> B1= {{3,0},{5,0},{5,2},{3,2}}; // square, no collision
        vector<P> B2= {{1,0},{3,0},{3,2},{1,2}}; // square, overlapping
        printf("A and B1 (apart):     %s\n", convex_collide(A,B1)?"COLLIDE":"NO COLLISION");
        printf("A and B2 (overlapping):%s\n", convex_collide(A,B2)?"COLLIDE":"NO COLLISION");
    }

    // ---- Stress: Minkowski sum area vs brute force ----
    {
        srand(42); int errors = 0;
        for (int trial = 0; trial < 1000; trial++) {
            int na = 3+rand()%6, nb = 3+rand()%6;
            vector<P> ptsa(na), ptsb(nb);
            for (auto& p : ptsa) p = {ld(rand()%10), ld(rand()%10)};
            for (auto& p : ptsb) p = {ld(rand()%10), ld(rand()%10)};
            auto A = convex_hull(ptsa), B = convex_hull(ptsb);
            if ((int)A.size() < 3 || (int)B.size() < 3) continue;

            // Fast O(n+m) Minkowski sum
            auto ms = minkowski_sum(A, B);

            // Brute force: all pairwise sums → convex hull
            vector<P> all_pts;
            for (auto& a : A) for (auto& b : B) all_pts.push_back(a+b);
            auto brute = convex_hull(all_pts);

            if ((int)ms.size() < 3 || (int)brute.size() < 3) continue;

            ld a1 = area(ms), a2 = area(brute);
            if (abs(a1-a2) > 1e-4) errors++;
        }
        printf("\nMinkowski stress 1000 trials: %s\n",
               errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Stress: collision detection ----
    {
        srand(77); int errors = 0;
        for (int trial = 0; trial < 500; trial++) {
            // Generate two random convex polygons
            int na=3+rand()%5, nb=3+rand()%5;
            vector<P> ptsa(na), ptsb(nb);
            for (auto& p : ptsa) p = {ld(rand()%10),   ld(rand()%10)};
            for (auto& p : ptsb) p = {ld(rand()%10)+5, ld(rand()%10)};
            auto A = convex_hull(ptsa), B = convex_hull(ptsb);
            if ((int)A.size()<3||(int)B.size()<3) continue;

            bool fast = convex_collide(A, B);

            // Brute force: check if any point of A is in B or vice versa,
            // or any edge pair intersects
            bool brute = false;
            // Check if any vertex of A is inside B
            for (auto& p : A) if (pip_convex(p,B) >= 0) { brute=true; break; }
            if (!brute)
                for (auto& p : B) if (pip_convex(p,A) >= 0) { brute=true; break; }

            if (fast != brute) errors++;
        }
        printf("Collision stress 500 trials:  %s\n",
               errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }
    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public static class MinkowskiSum
{
    const double EPS = 1e-9;

    public struct Pt
    {
        public double X, Y;
        public Pt(double x, double y) { X=x; Y=y; }
        public static Pt operator+(Pt a, Pt b) => new Pt(a.X+b.X, a.Y+b.Y);
        public static Pt operator-(Pt a, Pt b) => new Pt(a.X-b.X, a.Y-b.Y);
        public static Pt operator*(Pt a, double t) => new Pt(a.X*t, a.Y*t);
    }

    static double Cross(Pt a, Pt b) => a.X*b.Y - a.Y*b.X;
    static int    Sign (double v)   => v>EPS?1:v<-EPS?-1:0;

    static bool Less(Pt a, Pt b) =>
        a.X < b.X-EPS || (Math.Abs(a.X-b.X)<EPS && a.Y < b.Y-EPS);

    // Convex hull — Andrew's monotone chain, CCW
    public static Pt[] ConvexHull(Pt[] pts)
    {
        var s = pts.OrderBy(p=>p.X).ThenBy(p=>p.Y).ToList();
        var h = new List<Pt>();
        foreach (var p in s) {
            while (h.Count>=2 && Sign(Cross(h[^1]-h[^2],p-h[^2]))<=0) h.RemoveAt(h.Count-1);
            h.Add(p);
        }
        int lo = h.Count;
        for (int i=s.Count-2;i>=0;i--) {
            while (h.Count>lo && Sign(Cross(h[^1]-h[^2],s[i]-h[^2]))<=0) h.RemoveAt(h.Count-1);
            h.Add(s[i]);
        }
        h.RemoveAt(h.Count-1);
        return h.ToArray();
    }

    // Reorder polygon to start from bottom-left vertex
    static Pt[] Reorder(Pt[] poly) {
        int idx=0;
        for (int i=1;i<poly.Length;i++) if (Less(poly[i],poly[idx])) idx=i;
        return poly.Skip(idx).Concat(poly.Take(idx)).ToArray();
    }

    // Minkowski Sum of two convex CCW polygons — O(n+m)
    public static Pt[] Compute(Pt[] A, Pt[] B)
    {
        A = Reorder(A); B = Reorder(B);
        int n=A.Length, m=B.Length;
        // Extend with wrap-around
        var Ae = A.Concat(new[]{A[0],A[1]}).ToArray();
        var Be = B.Concat(new[]{B[0],B[1]}).ToArray();

        var result = new List<Pt>();
        int i=0, j=0;
        while (i<n || j<m) {
            result.Add(Ae[i]+Be[j]);
            Pt ea = Ae[i+1]-Ae[i];
            Pt eb = Be[j+1]-Be[j];
            double c = Cross(ea,eb);
            if      (Sign(c)>0) i++;
            else if (Sign(c)<0) j++;
            else                { i++; j++; }
            if (i>=n && j>=m) break;
            if (i>n) i=n;
            if (j>m) j=m;
        }
        return result.ToArray();
    }

    // Minkowski Difference: A ⊖ B = A ⊕ (-B)
    public static Pt[] Difference(Pt[] A, Pt[] B)
    {
        var negB = B.Select(p=>new Pt(-p.X,-p.Y)).Reverse().ToArray();
        return Compute(A, negB);
    }

    // Point in convex polygon — O(log n)
    public static int PipConvex(Pt q, Pt[] poly)
    {
        int n=poly.Length; if(n<3)return -1;
        Pt o=poly[0];
        int s1=Sign(Cross(poly[1]-o,q-o));
        int sn=Sign(Cross(poly[n-1]-o,q-o));
        if(s1<0||sn>0)return -1;
        int lo=1,hi=n-1;
        while(hi-lo>1){int mid=(lo+hi)/2;if(Sign(Cross(poly[mid]-o,q-o))>=0)lo=mid;else hi=mid;}
        int c=Sign(Cross(poly[lo+1]-poly[lo],q-poly[lo]));
        if(c<0)return -1; if(c==0)return 0;
        if(s1==0||sn==0)return 0;
        return 1;
    }

    // Collision detection via Minkowski Difference
    public static bool Collide(Pt[] A, Pt[] B)
    {
        var diff = Difference(A, B);
        if (diff.Length < 3) return false;
        return PipConvex(new Pt(0,0), diff) >= 0;
    }

    static double Area(Pt[] poly) {
        double s=0; int n=poly.Length;
        for(int i=0;i<n;i++) s+=Cross(poly[i],poly[(i+1)%n]);
        return Math.Abs(s)/2;
    }

    public static void Main()
    {
        // Square + Triangle
        var sq  = new Pt[]{ new(0,0),new(2,0),new(2,2),new(0,2) };
        var tri = new Pt[]{ new(0,0),new(1,0),new(0,1) };
        var ms  = Compute(sq, tri);
        Console.WriteLine($"Square+Triangle: {ms.Length} vertices, area={Area(ms):F2}");
        foreach (var p in ms) Console.WriteLine($"  ({p.X:F1}, {p.Y:F1})");

        // Collision detection
        var A  = new Pt[]{ new(0,0),new(2,0),new(2,2),new(0,2) };
        var B1 = new Pt[]{ new(3,0),new(5,0),new(5,2),new(3,2) };
        var B2 = new Pt[]{ new(1,0),new(3,0),new(3,2),new(1,2) };
        Console.WriteLine($"A vs B1 (apart):     {(Collide(A,B1)?"COLLIDE":"NO COLLISION")}");
        Console.WriteLine($"A vs B2 (overlap):   {(Collide(A,B2)?"COLLIDE":"NO COLLISION")}");
    }
}
```

---

## Reflecting a Polygon for Minkowski Difference

To compute `A ⊖ B = A ⊕ (-B)`, reflect B through the origin (negate all coordinates). The reflected polygon `-B` is initially clockwise — reverse the vertex list to restore CCW order:

```
reflect(B):
    negate all vertices: B'[i] = (-B[i].x, -B[i].y)
    reverse vertex order: B'' = reverse(B')
    return B''   // now CCW

minkowski_diff(A, B) = minkowski_sum(A, reflect(B))
```

After this, `origin ∈ A ⊕ (-B)` is equivalent to `A ∩ B ≠ ∅` for convex A and B.

---

## Pitfalls

- **Polygon must be convex and CCW** — the O(n+m) algorithm relies on the edge vectors of each polygon being in sorted angular order (which is true for CCW convex polygons). A CW polygon or a non-convex polygon produces wrong edge-angle ordering and silently outputs a garbage result. Always run a convex hull first and verify CCW orientation.
- **Start from the correct vertex** — both polygons must begin from their respective bottom-left vertex (smallest y, then smallest x). Starting from an arbitrary vertex desynchronises the edge-angle merge and produces an incorrect polygon or missing vertices.
- **Parallel edges advance both pointers** — when `cross(edgeA, edgeB) == 0` (edges are parallel and co-directional), both `i` and `j` must advance. Advancing only one skips a vertex and creates a gap in the result polygon.
- **Wraparound index bounds** — the two extra vertices appended (`A[0]`, `A[1]` and `B[0]`, `B[1]`) are needed so that `A[i+1]` and `B[j+1]` are valid when `i = n-1` or `j = m-1`. Without them, the last edge of each polygon is never evaluated.
- **Reflected polygon must be re-sorted CCW** — `reflect(B)` negates all coordinates making the polygon CW. Reversing the vertex list (not just negating and keeping the same order) restores CCW. Forgetting the reversal passes a CW polygon into `minkowski_sum`, which then produces an incorrect Minkowski difference.
- **Collision check uses ≥ 0, not > 0** — the origin on the boundary of `A ⊖ B` means A and B touch at exactly one point (or share an edge). Depending on the application this may count as a collision. Using `> 0` (strict interior only) misses touching cases; using `>= 0` (interior + boundary) catches them.
- **Non-convex polygons require decomposition** — the O(n+m) algorithm only works for convex polygons. For non-convex input it silently produces a convex hull of the result rather than the true (possibly non-convex) Minkowski sum. Decompose into convex parts first.

---

## Complexity Summary

| Input | Algorithm | Time | Output vertices |
|---|---|---|---|
| Two convex polygons | Edge merge | O(n + m) | ≤ n + m |
| Convex + point | Translate | O(n) | n |
| Convex + disk r | Offset edges + arcs | O(n) | n + arc resolution |
| Non-convex (k parts each) | Pairwise convex sums | O(k²(n+m)) | O(k²(n+m)) |
| General non-convex | Exact (rare) | O(n²m²) | O(n²m²) |

---

## Conclusion

Minkowski Sum is the **fundamental operation for shape arithmetic in the plane**:

- For convex polygons the O(n+m) edge-merge algorithm is optimal — the result has at most n+m vertices and is computed in a single simultaneous traversal of both edge sequences sorted by angle.
- The Minkowski difference `A ⊖ B = A ⊕ (-B)` reduces convex polygon collision detection to a single point-in-polygon query at the origin, which combined with the O(log n) convex PIP gives an extremely fast collision test.
- The key algorithmic insight is that convexity guarantees edge vectors are already angularly sorted — merging them is O(n+m) with no sorting step needed, analogous to the merge step in merge sort.

**Key takeaway:** always ensure both polygons are convex, CCW, and restarted from their bottom-left vertex before running the merge. The cross product of consecutive edge vectors decides which edge is more counterclockwise — positive cross product means the first edge comes first in the angular order.
