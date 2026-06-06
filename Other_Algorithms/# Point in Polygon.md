

# Point in Polygon

## Origin & Motivation

**Point in Polygon (PIP)** is one of the most fundamental queries in computational geometry: given a polygon and a query point, determine whether the point lies inside, outside, or on the boundary of the polygon. It is a building block for almost every spatial algorithm — clipping, collision detection, GIS, ray tracing, mesh generation.

Three algorithms cover the full practical range:

**Ray Casting (Jordan curve theorem, 1887)** — shoot a ray from the query point in any direction (conventionally rightward) and count how many polygon edges it crosses. Odd = inside, Even = outside. Works for any simple polygon, O(n) per query.

**Winding Number** — compute the total angle the polygon winds around the query point. Nonzero = inside, zero = outside. More robust than ray casting for self-intersecting (non-simple) polygons, same O(n) cost.

**Binary Search on Convex Polygon** — for convex polygons given in sorted angular order, binary search locates the triangle of the fan decomposition containing the point in O(log n).

Complexity: Ray Casting / Winding Number **O(n)** per query, Convex Binary Search **O(log n)** per query, all **O(1)** preprocessing.

---

## Where It Is Used

- GIS and mapping: is this GPS coordinate inside a country/region polygon?
- Game development: collision detection, fog-of-war regions, area triggers
- Computer graphics: ray tracing (determine medium a ray is in), polygon fill
- Robotics: is the robot inside a forbidden zone?
- Mesh generation: classify points relative to boundary polygons
- Computational geometry: preprocessing step for almost every polygon operation

---

## Core Concepts

### Simple vs Non-Simple Polygon

A **simple polygon** has no self-intersections. Ray casting and winding number both work on simple polygons. The winding number also correctly handles **self-intersecting** polygons (the "even-odd" vs "nonzero" fill rule used in SVG and PDF).

### Boundary Convention

All three implementations here return three distinct values:

```
 1  = strictly inside
 0  = on boundary (edge or vertex)
-1  = strictly outside
```

---

## Ray Casting Algorithm

Shoots a horizontal ray rightward from `q`. An edge is counted if the ray properly crosses it. The standard formulation avoids double-counting vertices by using a strict inequality on one endpoint:

```
RayCast(q, polygon[0..n-1]):
    crossings = 0
    for each edge (a, b):
        if q lies on edge (a,b): return BOUNDARY

        // Count crossing using upward-crossing rule
        if a.y ≤ q.y:
            if b.y > q.y AND cross(b-a, q-a) > 0:
                crossings++
        else:
            if b.y ≤ q.y AND cross(b-a, q-a) < 0:
                crossings++

    return (crossings % 2 == 1) ? INSIDE : OUTSIDE
```

**Why the upward-crossing rule?** By counting only upward crossings (a.y ≤ q.y < b.y) we handle the vertex-on-ray case deterministically: a vertex exactly on the ray is counted exactly once regardless of the polygon shape.

**Cross product test** instead of explicit line-ray intersection avoids computing the actual x-intercept and is numerically more stable:

```
cross(b-a, q-a) > 0  ⟺  q is to the left of directed edge a→b
                      ⟺  the ray rightward from q crosses a→b from below
```

---

## Winding Number Algorithm

The winding number `w(q, P)` counts how many times the polygon winds around `q`:

```
WindingNumber(q, polygon[0..n-1]):
    w = 0
    for each edge (a, b):
        if q lies on edge (a,b): return 0  // boundary

        if a.y ≤ q.y:
            if b.y > q.y AND cross(b-a, q-a) > 0:
                w++      // upward crossing, q is left of edge → CCW wind
        else:
            if b.y ≤ q.y AND cross(b-a, q-a) < 0:
                w--      // downward crossing, q is right of edge → CW wind

    return w   // 0 = outside, nonzero = inside
```

**Relation to ray casting:** For simple polygons `w ∈ {0, 1, -1}` and `|w| == crossings % 2` — both give the same inside/outside classification. For self-intersecting polygons, winding number distinguishes regions wound multiple times (winding number = 2 → doubly inside under nonzero fill rule).

---

## Convex Polygon — O(log n) Binary Search

For a convex polygon with vertices in CCW order, decompose into a fan of triangles from vertex 0:

```
ConvexPIP(q, polygon[0..n-1]):   // polygon is convex, CCW, n ≥ 3
    o = polygon[0]

    // Check if q is in the angular range of the fan
    if cross(polygon[1]-o, q-o) < 0:   return OUTSIDE
    if cross(polygon[n-1]-o, q-o) > 0: return OUTSIDE

    // Binary search for the sector containing q
    lo = 1, hi = n-1
    while hi - lo > 1:
        mid = (lo+hi)/2
        if cross(polygon[mid]-o, q-o) ≥ 0: lo = mid
        else: hi = mid

    // q is in triangle (o, polygon[lo], polygon[lo+1])
    c = cross(polygon[lo+1] - polygon[lo], q - polygon[lo])
    if c < 0:  return OUTSIDE
    if c == 0: return BOUNDARY
    return INSIDE
```

**Why O(log n)?** The fan from vertex 0 partitions the polygon into n-2 triangles. Binary search over the angles finds the right triangle in O(log n). The final cross product test places q relative to the triangle's outer edge in O(1).

**Limitation:** Only works for convex polygons given in consistent (CCW) order. Use ray casting or winding number for general polygons.

---

## Complexity Summary

| Algorithm | Polygon type | Query | Preprocessing | Space |
|---|---|---|---|---|
| Ray Casting | Any simple | O(n) | O(1) | O(n) |
| Winding Number | Simple or self-intersecting | O(n) | O(1) | O(n) |
| Convex Binary Search | Convex only | O(log n) | O(1) | O(n) |
| Point location (DCEL/trapezoidal) | Any simple | O(log n) | O(n log n) | O(n) |

For repeated queries on the same **non-convex** polygon: build a trapezoidal decomposition (O(n log n), O(n)) to get O(log n) per query. For competitive programming O(n) per query is almost always sufficient.

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
    P operator-(P o) const { return {x-o.x, y-o.y}; }
    P operator+(P o) const { return {x+o.x, y+o.y}; }
    P operator*(ld t) const { return {x*t, y*t}; }
};

ld cross(P a, P b) { return a.x*b.y - a.y*b.x; }
ld dot  (P a, P b) { return a.x*b.x + a.y*b.y; }
int sign(ld v)     { return v>EPS ? 1 : v<-EPS ? -1 : 0; }

bool on_segment(P q, P a, P b) {
    return sign(cross(b-a, q-a)) == 0
        && sign(dot(q-a, b-a)) >= 0
        && sign(dot(q-b, a-b)) >= 0;
}

// ================================================================
// 1. RAY CASTING — O(n), any simple polygon
// Returns: 1 = inside, 0 = on boundary, -1 = outside
// ================================================================
int ray_cast(P q, const vector<P>& poly) {
    int n = (int)poly.size();
    int crossings = 0;
    for (int i = 0; i < n; i++) {
        P a = poly[i], b = poly[(i+1) % n];

        // Boundary check
        if (on_segment(q, a, b)) return 0;

        // Upward-crossing rule: count edges crossing the horizontal ray rightward
        if (a.y <= q.y) {
            if (b.y > q.y && sign(cross(b-a, q-a)) > 0) crossings++;
        } else {
            if (b.y <= q.y && sign(cross(b-a, q-a)) < 0) crossings++;
        }
    }
    return (crossings % 2 == 1) ? 1 : -1;
}

// ================================================================
// 2. WINDING NUMBER — O(n), works for self-intersecting polygons
// Returns winding number (0 = outside, nonzero = inside)
// Returns INT_MIN if q is on the boundary
// ================================================================
int winding_number(P q, const vector<P>& poly) {
    int n = (int)poly.size();
    int w = 0;
    for (int i = 0; i < n; i++) {
        P a = poly[i], b = poly[(i+1) % n];
        if (on_segment(q, a, b)) return 0; // on boundary → winding = 0 sentinel

        if (a.y <= q.y) {
            if (b.y > q.y && sign(cross(b-a, q-a)) > 0) w++;
        } else {
            if (b.y <= q.y && sign(cross(b-a, q-a)) < 0) w--;
        }
    }
    return w;
}

// Convenience: same -1/0/1 convention as ray_cast
int pip_winding(P q, const vector<P>& poly) {
    int w = winding_number(q, poly);
    if (w == 0) {
        // Distinguish outside from boundary (winding_number returns 0 for both)
        // Re-use ray_cast boundary check for boundary detection
        int n = (int)poly.size();
        for (int i = 0; i < n; i++) {
            P a=poly[i], b=poly[(i+1)%n];
            if (on_segment(q,a,b)) return 0;
        }
        return -1;
    }
    return 1;
}

// ================================================================
// 3. CONVEX POLYGON BINARY SEARCH — O(log n)
// polygon must be convex and given in CCW order
// Returns: 1 = inside, 0 = on boundary, -1 = outside
// ================================================================
int pip_convex(P q, const vector<P>& poly) {
    int n = (int)poly.size();
    if (n == 1) return (sign(cross(q-poly[0], q-poly[0])) == 0) ? 0 : -1;
    if (n == 2) return on_segment(q, poly[0], poly[1]) ? 0 : -1;

    P o = poly[0];

    // Check angular bounds of the fan
    int s1 = sign(cross(poly[1]-o,   q-o));
    int sn = sign(cross(poly[n-1]-o, q-o));
    if (s1 < 0 || sn > 0) return -1;

    // Binary search for the sector
    int lo = 1, hi = n-1;
    while (hi - lo > 1) {
        int mid = (lo + hi) / 2;
        if (sign(cross(poly[mid]-o, q-o)) >= 0) lo = mid;
        else hi = mid;
    }

    // Place q relative to the outer edge of triangle (o, poly[lo], poly[lo+1])
    int c = sign(cross(poly[lo+1] - poly[lo], q - poly[lo]));
    if (c < 0) return -1; // outside the triangle
    if (c == 0) return 0; // on the outer edge

    // Also check if on the fan edges from o
    if (s1 == 0) return 0;
    if (sn == 0) return 0;
    return 1;
}

// ================================================================
// Usage
// ================================================================
int main() {
    // ---- Square ----
    {
        printf("=== Square [0,4]x[0,4] ===\n");
        vector<P> sq = {{0,0},{4,0},{4,4},{0,4}};
        auto test = [&](P q, const char* name) {
            printf("%-10s  ray=%2d  winding=%2d  convex=%2d\n",
                   name,
                   ray_cast(q, sq),
                   pip_winding(q, sq),
                   pip_convex(q, sq));
        };
        test({2,2},  "center");
        test({5,2},  "outside");
        test({0,0},  "corner");
        test({2,0},  "edge");
        test({-1,2}, "left");
    }

    // ---- Non-convex L-shape ----
    {
        printf("\n=== L-shape (non-convex) ===\n");
        vector<P> L = {{0,0},{4,0},{4,2},{2,2},{2,4},{0,4}};
        auto test = [&](P q, const char* name) {
            printf("%-6s  ray=%2d  winding=%2d\n",
                   name, ray_cast(q, L), pip_winding(q, L));
        };
        test({1,1}, "in1");
        test({3,3}, "out");
        test({3,1}, "in2");
        test({1,3}, "in3");
    }

    // ---- Triangle (convex) ----
    {
        printf("\n=== Triangle (convex, O(log n)) ===\n");
        vector<P> tri = {{0,0},{6,0},{3,5}};
        auto test = [&](P q, const char* name) {
            printf("%-8s  ray=%2d  convex=%2d\n",
                   name, ray_cast(q, tri), pip_convex(q, tri));
        };
        test({3,2},  "inside");
        test({3,0},  "edge");
        test({0,0},  "vertex");
        test({6,3},  "outside");
        test({-1,0}, "far out");
    }

    // ---- Stress ----
    {
        printf("\n=== Stress: ray_cast vs winding_number ===\n");
        srand(42); int errors = 0;
        for (int trial = 0; trial < 5000; trial++) {
            // Random convex hull as test polygon
            int n = 3 + rand()%8;
            vector<P> pts(n);
            for (auto& p : pts) p = {ld(rand()%20), ld(rand()%20)};
            sort(pts.begin(), pts.end(), [](P a,P b){
                return a.x<b.x||(a.x==b.x&&a.y<b.y); });
            // Convex hull (Andrew's monotone chain)
            vector<P> hull;
            for (auto& p : pts) {
                while (hull.size()>=2 &&
                       sign(cross(hull.back()-hull[hull.size()-2],p-hull[hull.size()-2]))<=0)
                    hull.pop_back();
                hull.push_back(p);
            }
            int lo = hull.size();
            for (int i=(int)pts.size()-2;i>=0;i--) {
                while ((int)hull.size()>lo &&
                       sign(cross(hull.back()-hull[hull.size()-2],pts[i]-hull[hull.size()-2]))<=0)
                    hull.pop_back();
                hull.push_back(pts[i]);
            }
            hull.pop_back();
            if ((int)hull.size() < 3) continue;

            for (int q = 0; q < 10; q++) {
                P qp = {ld(rand()%24)-2, ld(rand()%24)-2};
                int rc = ray_cast(qp, hull);
                int wn = pip_winding(qp, hull);
                int cv = pip_convex(qp, hull);
                // All three must agree on strictly inside vs outside
                if ((rc==1) != (wn==1)) errors++;
                if ((rc==1) != (cv==1)) errors++;
            }
        }
        printf("5000 trials: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }
    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class PointInPolygon
{
    const double EPS = 1e-9;

    public struct Pt { public double X, Y; public Pt(double x,double y){X=x;Y=y;} }

    static double Cross(Pt a, Pt b) => a.X*b.Y - a.Y*b.X;
    static double Dot  (Pt a, Pt b) => a.X*b.X + a.Y*b.Y;
    static int    Sign (double v)   => v>EPS?1:v<-EPS?-1:0;
    static Pt     Sub  (Pt a, Pt b) => new Pt(a.X-b.X, a.Y-b.Y);

    static bool OnSeg(Pt q, Pt a, Pt b) =>
        Sign(Cross(Sub(b,a),Sub(q,a)))==0 &&
        Sign(Dot(Sub(q,a),Sub(b,a)))>=0   &&
        Sign(Dot(Sub(q,b),Sub(a,b)))>=0;

    // ---- Ray Casting: 1=inside, 0=boundary, -1=outside ----
    public static int RayCast(Pt q, Pt[] poly)
    {
        int n=poly.Length, crossings=0;
        for (int i=0;i<n;i++){
            Pt a=poly[i], b=poly[(i+1)%n];
            if (OnSeg(q,a,b)) return 0;
            if (a.Y<=q.Y){
                if (b.Y>q.Y && Sign(Cross(Sub(b,a),Sub(q,a)))>0) crossings++;
            } else {
                if (b.Y<=q.Y && Sign(Cross(Sub(b,a),Sub(q,a)))<0) crossings++;
            }
        }
        return crossings%2==1 ? 1 : -1;
    }

    // ---- Winding Number: returns winding count (0=outside, 0=boundary sentinel) ----
    public static int WindingNumber(Pt q, Pt[] poly)
    {
        int n=poly.Length, w=0;
        for (int i=0;i<n;i++){
            Pt a=poly[i], b=poly[(i+1)%n];
            if (OnSeg(q,a,b)) return 0; // boundary
            if (a.Y<=q.Y){
                if (b.Y>q.Y && Sign(Cross(Sub(b,a),Sub(q,a)))>0) w++;
            } else {
                if (b.Y<=q.Y && Sign(Cross(Sub(b,a),Sub(q,a)))<0) w--;
            }
        }
        return w;
    }

    // Same -1/0/1 convention
    public static int PipWinding(Pt q, Pt[] poly)
    {
        int w = WindingNumber(q,poly);
        if (w==0){
            int n=poly.Length;
            for(int i=0;i<n;i++) if(OnSeg(q,poly[i],poly[(i+1)%n])) return 0;
            return -1;
        }
        return 1;
    }

    // ---- Convex Binary Search: O(log n), CCW polygon ----
    public static int PipConvex(Pt q, Pt[] poly)
    {
        int n=poly.Length;
        if (n<3) return -1;
        Pt o=poly[0];
        int s1=Sign(Cross(Sub(poly[1],o),Sub(q,o)));
        int sn=Sign(Cross(Sub(poly[n-1],o),Sub(q,o)));
        if (s1<0||sn>0) return -1;
        int lo=1,hi=n-1;
        while(hi-lo>1){
            int mid=(lo+hi)/2;
            if(Sign(Cross(Sub(poly[mid],o),Sub(q,o)))>=0) lo=mid; else hi=mid;
        }
        int c=Sign(Cross(Sub(poly[lo+1],poly[lo]),Sub(q,poly[lo])));
        if(c<0)  return -1;
        if(c==0) return 0;
        if(s1==0||sn==0) return 0;
        return 1;
    }

    public static void Main()
    {
        var sq = new Pt[]{ new(0,0),new(4,0),new(4,4),new(0,4) };
        Console.WriteLine("=== Square ===");
        Console.WriteLine($"center  rc={RayCast(new(2,2),sq)} wn={PipWinding(new(2,2),sq)} cv={PipConvex(new(2,2),sq)}");
        Console.WriteLine($"outside rc={RayCast(new(5,2),sq)} wn={PipWinding(new(5,2),sq)} cv={PipConvex(new(5,2),sq)}");
        Console.WriteLine($"corner  rc={RayCast(new(0,0),sq)} wn={PipWinding(new(0,0),sq)} cv={PipConvex(new(0,0),sq)}");
        Console.WriteLine($"edge    rc={RayCast(new(2,0),sq)} wn={PipWinding(new(2,0),sq)} cv={PipConvex(new(2,0),sq)}");

        var L = new Pt[]{ new(0,0),new(4,0),new(4,2),new(2,2),new(2,4),new(0,4) };
        Console.WriteLine("\n=== L-shape ===");
        Console.WriteLine($"in1  rc={RayCast(new(1,1),L)} wn={WindingNumber(new(1,1),L)}");
        Console.WriteLine($"out  rc={RayCast(new(3,3),L)} wn={WindingNumber(new(3,3),L)}");
    }
}
```

---

## Edge Cases and Degenerate Inputs

### Ray Through a Vertex

The classic pitfall: if the horizontal ray passes exactly through a vertex `v` shared by edges `(a,v)` and `(v,b)`, both edges are tested and the crossing might be counted 0 or 2 times instead of 1.

**Fix — upward-crossing rule:** count edge `(a,b)` only if `a.y ≤ q.y < b.y` (or the symmetric downward case). This makes each vertex counted at most once, always by the edge going upward through q.y:

```
Vertex v exactly on ray:
  edge (prev, v): b.y = v.y ≤ q.y → not counted (b.y not strictly > q.y)
  edge (v, next): a.y = v.y ≤ q.y → counted only if next.y > q.y
Result: vertex counted at most once — exactly what Jordan curve theorem requires.
```

### Horizontal Edges

A horizontal edge `(a,b)` with `a.y == b.y == q.y` would be tested as `a.y ≤ q.y` and `b.y ≤ q.y` simultaneously. The cross product `cross(b-a, q-a)` is zero for all points on the line, but the upward-crossing condition `b.y > q.y` is false — so horizontal edges are never counted. Horizontal edges on the ray must be handled separately by the boundary check.

### Point on Vertex

Always run the `on_segment` check first, before the crossing count. A vertex `poly[i]` satisfies `on_segment(q, poly[i-1], poly[i])` or `on_segment(q, poly[i], poly[i+1])`, so boundary detection catches it.

---

## Pitfalls

- **Ray through vertex counted twice** — without the upward-crossing rule, a ray passing through vertex `v` is counted by both edges meeting at `v`, giving an even count and incorrectly classifying inside points as outside. Always use `a.y ≤ q.y < b.y` (strict on one side).
- **Boundary not detected before counting crossings** — if `q` lies on an edge, the cross product test may return ±1 or 0 depending on floating-point precision. Always run the `on_segment` check first and return `BOUNDARY` immediately.
- **Convex binary search requires CCW order** — if the polygon is given in CW order, all cross products have reversed sign and the algorithm returns wrong results. Either reverse the polygon or negate the cross product comparisons.
- **Convex binary search fan from vertex 0** — vertex 0 must be an actual vertex of the convex hull, not an interior point. The algorithm decomposes the polygon into a fan of triangles from vertex 0; if vertex 0 is collinear with its neighbors the fan degenerate and some regions are missed.
- **Winding number and ray casting agree only for simple polygons** — for self-intersecting polygons, winding number implements the "nonzero fill rule" (any nonzero winding = inside) while ray casting implements "even-odd fill rule" (odd crossings = inside). The two disagree for regions wound an even number of times.
- **Integer coordinates vs floating point** — for polygons with integer coordinates use `long long` cross products to avoid all floating-point issues. The cross product of vectors with coordinates ≤ 10⁹ fits in `long long` (max ≈ 2·10¹⁸ < 9.2·10¹⁸). Using `double` introduces EPS-dependent edge cases near boundaries.
- **Polygon not closed explicitly** — implementations assume edge `(poly[n-1], poly[0])` closes the polygon. If the caller accidentally adds `poly[0]` again at the end, the closing edge is doubled and the parity of crossings is wrong.

---

## Conclusion

Point in Polygon is a **fundamental primitive** with three implementations suited to different contexts:

- **Ray Casting** is the default choice: simple to implement, O(n) per query, works for any simple polygon. The upward-crossing rule handles all vertex/edge degenerate cases with no special cases beyond the boundary check.
- **Winding Number** is preferred when the polygon may be self-intersecting or when the "nonzero fill rule" is semantically required (SVG fill, PostScript, PDF rendering).
- **Convex Binary Search** gives O(log n) per query for convex polygons — the right choice for repeated queries against a fixed convex region (e.g. game zone boundary, convex obstacle).

**Key takeaway:** the cross product `cross(b-a, q-a)` is the single primitive underlying all three algorithms. Its sign determines which side of a directed line the query point lies on; applied repeatedly around the polygon it encodes both the crossing count and the winding number without ever computing an explicit intersection point.


