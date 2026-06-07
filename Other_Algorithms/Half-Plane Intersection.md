# Half-Plane Intersection

## Origin & Motivation

**Half-Plane Intersection** computes the region of the plane satisfying a set of linear inequalities simultaneously — the intersection of `n` half-planes. Each half-plane is the set of points on one side of a directed line. The intersection is a convex polygon (possibly empty, unbounded, or infinite).

**The problem it solves:** Linear programming in 2D, feasibility of convex constraints, and many classical geometry problems reduce to half-plane intersection. The naive approach clips a polygon by each half-plane one at a time in O(n) each, giving O(n²) total. The **sorted incremental algorithm** (Shamos 1978, refined by various authors) solves it in **O(n log n)** by sorting half-planes by angle and maintaining a deque of candidates.

**The key idea:** Sort half-planes by the angle of their boundary line. Process them in angular order. Maintain a deque of half-planes whose pairwise boundary intersections form the current feasible polygon frontier. When adding a new half-plane, trim the back and front of the deque — any intersection point now outside the new half-plane's boundary is infeasible and its half-plane is discarded.

Complexity: **O(n log n)** (dominated by sort), **O(n)** space.

---

## Where It Is Used

- 2D linear programming (minimize `c·x` subject to `Ax ≤ b`)
- Feasibility testing: does a set of linear constraints have a solution?
- Convex polygon intersection (each edge of a convex polygon defines a half-plane)
- Voronoi diagrams: each Voronoi cell is a half-plane intersection
- Robot visibility and motion planning (field-of-view constraints)
- Competitive programming: "find the region satisfying all given inequalities"

---

## Core Structure

### Half-Plane Definition

A half-plane is defined by a directed line `a → b`. The half-plane contains all points **on the left** of this direction:

```
Half-plane H(a, b) = { q : cross(b-a, q-a) >= 0 }

cross(b-a, q-a) > 0  →  q is strictly left  (inside)
cross(b-a, q-a) = 0  →  q is on boundary    (inside, boundary)
cross(b-a, q-a) < 0  →  q is strictly right (outside)
```

Equivalently, each half-plane corresponds to a linear inequality `ax + by ≤ c`.

### Angular Order

Sorting half-planes by the angle of their direction vector `atan2(d.y, d.x)` groups parallel half-planes together. Among parallel half-planes (same angle), only the most restrictive one matters — the one whose boundary line is furthest in the normal direction.

```
Among parallel half-planes with same angle:
  keep the one where the OTHER half-plane's defining point
  is most to the LEFT — i.e. sign(cross(a.d, b.p - a.p)) > 0
  means a is more restrictive than b.
```

---

## Algorithm

```
HalfPlaneIntersection(half_planes[0..n-1]):

    // Step 1: Sort by angle, break ties keeping most restrictive
    sort half_planes by angle
    remove dominated parallel duplicates (keep most restrictive per angle)

    // Step 2: Deque-based incremental construction
    dq  = deque of half-planes   (current candidate set)
    pts = deque of intersection points  (pts[i] = isect(dq[i], dq[i+1]))

    push half_planes[0] and half_planes[1] to dq
    push isect(dq[0], dq[1]) to pts

    for i = 2 to n-1:
        // Trim back: last intersection no longer in new half-plane
        while pts not empty AND pts.back() NOT in half_planes[i]:
            dq.pop_back(), pts.pop_back()
        // Trim front: first intersection no longer in new half-plane
        while pts not empty AND pts.front() NOT in half_planes[i]:
            dq.pop_front(), pts.pop_front()
        dq.push_back(half_planes[i])
        pts.push_back(isect(dq[-2], dq[-1]))

    // Step 3: Circular cleanup (wrap-around between first and last)
    while pts not empty AND pts.back()  NOT in dq.front(): pop back
    while pts not empty AND pts.front() NOT in dq.back():  pop front

    if dq.size() < 3: return EMPTY

    // Step 4: Collect final vertices
    pts.push_back(isect(dq.front(), dq.back()))
    return pts as convex polygon
```

**Why a deque?** The feasible region is a convex polygon. As we add half-planes in angular order, infeasible vertices appear at both ends of the current chain (not in the middle), so a deque lets us trim both ends efficiently in O(1) amortized per element.

---

## Key Invariant

At each step, the deque holds a sequence of half-planes `H[head..tail]` such that:
- Each consecutive pair's boundary intersection is feasible with respect to all half-planes processed so far.
- The intersections form the right "chain" of the current feasible polygon.

Adding `H[i]` eliminates any intersection points that violate `H[i]`, trimming the chain from both ends.

---

## Complexity Summary

| Step | Time |
|---|---|
| Sort by angle | O(n log n) |
| Deque processing | O(n) — each half-plane pushed/popped at most once |
| Total | O(n log n) |
| Space | O(n) |

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
    P operator+(P b) const { return {x+b.x, y+b.y}; }
    P operator-(P b) const { return {x-b.x, y-b.y}; }
    P operator*(ld t) const { return {x*t, y*t}; }
};
ld cross(P a, P b) { return a.x*b.y - a.y*b.x; }

// Half-plane: left of directed line a → b
// { q : cross(b-a, q-a) >= 0 }
struct Line {
    P p, d;   // point on line and direction
    ld ang;   // angle of direction for sorting
    Line() {}
    Line(P a, P b) : p(a), d(b-a), ang(atan2l(b.y-a.y, b.x-a.x)) {}
};

// Is q strictly on the left of line l?
bool onLeft(Line l, P q) { return cross(l.d, q-l.p) > EPS; }

// Intersection of two lines (boundary lines of half-planes)
P lineIsect(Line a, Line b) {
    ld t = cross(b.d, a.p-b.p) / cross(a.d, b.d);
    return a.p + a.d*t;
}

// ================================================================
// HALF-PLANE INTERSECTION — O(n log n)
// Input:  vector of half-planes (left of each directed line a→b)
// Output: convex polygon (CCW) or empty if infeasible/unbounded
// Note:   add a bounding box first to ensure bounded result
// ================================================================
vector<P> halfPlaneIntersection(vector<Line> L) {
    int n = L.size();

    // Sort by angle; among parallel, keep most restrictive
    sort(L.begin(), L.end(), [](const Line& a, const Line& b) {
        if (fabsl(a.ang - b.ang) > EPS) return a.ang < b.ang;
        return cross(a.d, b.p - a.p) > 0; // a more restrictive than b
    });

    // Remove dominated half-planes with same angle
    {
        vector<Line> tmp;
        for (auto& l : L)
            if (tmp.empty() || fabsl(tmp.back().ang - l.ang) > EPS)
                tmp.push_back(l);
        L = tmp;
        n = L.size();
    }
    if (n < 3) return {};

    deque<Line> dq;
    deque<P>    pts;

    dq.push_back(L[0]);
    dq.push_back(L[1]);
    pts.push_back(lineIsect(L[0], L[1]));

    for (int i = 2; i < n; i++) {
        // Trim back: intersection at back is outside L[i]
        while (!pts.empty() && !onLeft(L[i], pts.back())) {
            dq.pop_back(); pts.pop_back();
        }
        // Trim front: intersection at front is outside L[i]
        while (!pts.empty() && !onLeft(L[i], pts.front())) {
            dq.pop_front(); pts.pop_front();
        }
        dq.push_back(L[i]);
        pts.push_back(lineIsect(dq[dq.size()-2], dq.back()));
    }

    // Circular cleanup: wrap-around between front and back
    while (!pts.empty() && !onLeft(dq.front(), pts.back())) {
        dq.pop_back(); pts.pop_back();
    }
    while (!pts.empty() && !onLeft(dq.back(), pts.front())) {
        dq.pop_front(); pts.pop_front();
    }

    if ((int)dq.size() < 3) return {};

    pts.push_back(lineIsect(dq.front(), dq.back()));
    return vector<P>(pts.begin(), pts.end());
}

// ================================================================
// BOUNDING BOX — 4 half-planes enclosing [-B, B] x [-B, B]
// Always add this to ensure bounded intersection result
// ================================================================
vector<Line> boundingBox(ld B) {
    return {
        Line({-B,-B}, { B,-B}),  // bottom edge  → above y=-B
        Line({ B,-B}, { B, B}),  // right edge   → left  of x=B
        Line({ B, B}, {-B, B}),  // top edge     → below y=B
        Line({-B, B}, {-B,-B}),  // left edge    → right of x=-B
    };
}

// ================================================================
// AREA of convex polygon (shoelace)
// ================================================================
ld area(const vector<P>& poly) {
    ld s = 0; int n = poly.size();
    for (int i = 0; i < n; i++) s += cross(poly[i], poly[(i+1)%n]);
    return fabsl(s) / 2;
}

// ================================================================
// BRUTE FORCE: Sutherland-Hodgman clipping (for verification)
// ================================================================
vector<P> clipByHP(vector<P> poly, Line h) {
    vector<P> res; int n = poly.size();
    for (int i = 0; i < n; i++) {
        P a = poly[i], b = poly[(i+1)%n];
        bool ca = onLeft(h,a) || fabsl(cross(h.d,a-h.p)) <= EPS;
        bool cb = onLeft(h,b) || fabsl(cross(h.d,b-h.p)) <= EPS;
        if (ca) res.push_back(a);
        if (ca != cb) {
            ld t = cross(h.d, a-h.p) / cross(h.d, a-b);
            res.push_back(a + (b-a)*t);
        }
    }
    return res;
}
vector<P> bruteHPI(vector<Line> hs, ld B = 50) {
    vector<P> poly = {{-B,-B},{B,-B},{B,B},{-B,B}};
    for (auto& h : hs) poly = clipByHP(poly, h);
    return poly;
}

// ================================================================
// Usage + stress test
// ================================================================
int main() {
    // ---- Triangle ----
    {
        printf("=== Triangle ===\n");
        vector<Line> hs = {
            Line({0,0},{4,0}),  // above y=0
            Line({4,0},{0,4}),  // below hypotenuse
            Line({0,4},{0,0}),  // right of x=0
        };
        auto r = halfPlaneIntersection(hs);
        printf("Vertices (%d), area=%.4Lf\n", (int)r.size(), area(r));
        for (auto& p : r) printf("  (%.3Lf, %.3Lf)\n", p.x, p.y);
    }
    // ---- Square clipped by diagonal ----
    {
        printf("\n=== Square [0,4]^2 + diagonal cut ===\n");
        vector<Line> hs = {
            Line({0,0},{4,0}), Line({4,0},{4,4}),
            Line({4,4},{0,4}), Line({0,4},{0,0}),
            Line({0,4},{4,0}), // diagonal: keeps lower-right triangle
        };
        auto r = halfPlaneIntersection(hs);
        printf("Vertices (%d), area=%.4Lf (expect 8.0)\n", (int)r.size(), area(r));
        for (auto& p : r) printf("  (%.3Lf, %.3Lf)\n", p.x, p.y);
    }
    // ---- Bounded example with bbox ----
    {
        printf("\n=== Bounded: bbox + two half-planes ===\n");
        auto hs = boundingBox(5);
        hs.push_back(Line({0,0},{3,1}));  // above this line
        hs.push_back(Line({3,0},{0,2}));  // above this line
        auto r = halfPlaneIntersection(hs);
        printf("Vertices (%d), area=%.4Lf\n", (int)r.size(), area(r));
    }
    // ---- Stress: O(n log n) vs brute force ----
    {
        srand(42); int errors = 0;
        for (int t = 0; t < 3000; t++) {
            auto hs = boundingBox(15);
            int n = 2 + rand()%7;
            for (int i = 0; i < n; i++) {
                P a = {ld(rand()%20-10), ld(rand()%20-10)};
                P b = {ld(rand()%20-10), ld(rand()%20-10)};
                if (fabsl(cross(b-a, b-a)) < EPS) continue;
                hs.push_back(Line(a, b));
            }
            auto fast  = halfPlaneIntersection(hs);
            auto slow  = bruteHPI(hs, 20);
            if (fabsl(area(fast) - area(slow)) > 0.05) errors++;
        }
        printf("\nStress 3000 trials: %s\n",
               errors==0 ? "OK" : ("FAIL " + to_string(errors)).c_str());
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

public static class HalfPlaneIntersection
{
    const double EPS = 1e-9;

    public struct Pt
    {
        public double X, Y;
        public Pt(double x, double y) { X=x; Y=y; }
        public static Pt operator+(Pt a,Pt b)=>new Pt(a.X+b.X,a.Y+b.Y);
        public static Pt operator-(Pt a,Pt b)=>new Pt(a.X-b.X,a.Y-b.Y);
        public static Pt operator*(Pt a,double t)=>new Pt(a.X*t,a.Y*t);
    }

    static double Cross(Pt a,Pt b)=>a.X*b.Y-a.Y*b.X;

    // Half-plane: left of directed line A→B
    public struct Line
    {
        public Pt P,D; public double Ang;
        public Line(Pt a,Pt b){P=a;D=new Pt(b.X-a.X,b.Y-a.Y);Ang=Math.Atan2(b.Y-a.Y,b.X-a.X);}
    }

    static bool OnLeft(Line l,Pt q)=>Cross(l.D,q-l.P)>EPS;

    static Pt LineIsect(Line a,Line b){
        double t=Cross(b.D,a.P-b.P)/Cross(a.D,b.D);
        return a.P+a.D*t;
    }

    // Half-plane intersection — O(n log n)
    public static Pt[] Compute(Line[] input)
    {
        var L=input.ToList();
        L.Sort((a,b)=>{
            if(Math.Abs(a.Ang-b.Ang)>EPS) return a.Ang.CompareTo(b.Ang);
            return Cross(a.D,b.P-a.P)>0?-1:1;
        });
        // deduplicate same angle
        var tmp=new List<Line>();
        foreach(var l in L)
            if(tmp.Count==0||Math.Abs(tmp[^1].Ang-l.Ang)>EPS) tmp.Add(l);
        L=tmp; int n=L.Count;
        if(n<3) return Array.Empty<Pt>();

        var dq=new LinkedList<Line>();
        var pts=new LinkedList<Pt>();
        dq.AddLast(L[0]); dq.AddLast(L[1]);
        pts.AddLast(LineIsect(L[0],L[1]));

        for(int i=2;i<n;i++){
            while(pts.Count>0&&!OnLeft(L[i],pts.Last.Value)){dq.RemoveLast();pts.RemoveLast();}
            while(pts.Count>0&&!OnLeft(L[i],pts.First.Value)){dq.RemoveFirst();pts.RemoveFirst();}
            dq.AddLast(L[i]);
            var arr=dq.ToArray();
            pts.AddLast(LineIsect(arr[^2],arr[^1]));
        }
        while(pts.Count>0&&!OnLeft(dq.First.Value,pts.Last.Value)){dq.RemoveLast();pts.RemoveLast();}
        while(pts.Count>0&&!OnLeft(dq.Last.Value,pts.First.Value)){dq.RemoveFirst();pts.RemoveFirst();}
        if(dq.Count<3) return Array.Empty<Pt>();
        pts.AddLast(LineIsect(dq.First.Value,dq.Last.Value));
        return pts.ToArray();
    }

    // Bounding box [-B,B]x[-B,B]
    public static Line[] BoundingBox(double B)=>new[]{
        new Line(new Pt(-B,-B),new Pt( B,-B)),
        new Line(new Pt( B,-B),new Pt( B, B)),
        new Line(new Pt( B, B),new Pt(-B, B)),
        new Line(new Pt(-B, B),new Pt(-B,-B)),
    };

    static double Area(Pt[]p){
        double s=0;int n=p.Length;
        for(int i=0;i<n;i++)s+=Cross(p[i],p[(i+1)%n]);
        return Math.Abs(s)/2;
    }

    public static void Main()
    {
        // Triangle
        var tri=new Line[]{
            new Line(new Pt(0,0),new Pt(4,0)),
            new Line(new Pt(4,0),new Pt(0,4)),
            new Line(new Pt(0,4),new Pt(0,0)),
        };
        var r=Compute(tri);
        Console.WriteLine($"Triangle: {r.Length} vertices, area={Area(r):F4}");
        foreach(var p in r) Console.WriteLine($"  ({p.X:F3}, {p.Y:F3})");

        // Square + diagonal
        var sq=new Line[]{
            new Line(new Pt(0,0),new Pt(4,0)),new Line(new Pt(4,0),new Pt(4,4)),
            new Line(new Pt(4,4),new Pt(0,4)),new Line(new Pt(0,4),new Pt(0,0)),
            new Line(new Pt(0,4),new Pt(4,0)),
        };
        r=Compute(sq);
        Console.WriteLine($"Square+diag: {r.Length} vertices, area={Area(r):F4} (expect 8)");
    }
}
```

---

## Bounding Box Requirement

The algorithm returns an empty result for unbounded intersections (e.g. two parallel half-planes) unless a bounding box is added. Always prepend four bounding-box half-planes:

```
BoundingBox(B):
    Line({-B,-B}, { B,-B})  // y >= -B  (above bottom edge)
    Line({ B,-B}, { B, B})  // x <=  B  (left of right edge)
    Line({ B, B}, {-B, B})  // y <=  B  (below top edge)
    Line({-B, B}, {-B,-B})  // x >= -B  (right of left edge)

Choose B large enough that the true intersection fits inside [-B,B]^2.
```

---

## Pitfalls

- **`onLeft` uses strict inequality** — the deque cleanup condition `!onLeft(L[i], pt)` must be strict (`cross > EPS`), not `>=`. Using `>= 0` keeps boundary points that are exactly on the new half-plane's edge, which can cause the deque to not trim enough and produce an incorrect polygon.
- **Parallel half-planes: keep most restrictive** — when two half-planes have the same angle, only the more restrictive one contributes. The sort comparator must place the more restrictive one first (`cross(a.d, b.p - a.p) > 0`), and the deduplication must keep only the first (= most restrictive) per angle group.
- **Wrap-around cleanup is mandatory** — after the main loop, the intersection point between `dq.front()` and `dq.back()` is not yet computed. The circular cleanup trims both ends against each other's intersection. Skipping it produces a polygon with extra spurious vertices or an incorrect final edge.
- **Fewer than 3 planes in deque = empty** — if after all cleanup the deque has fewer than 3 half-planes, the intersection is empty or a single point/line segment. Return an empty vector in this case; attempting to collect vertices from a 2-element deque gives wrong results.
- **Unbounded intersection without bounding box** — if the half-planes do not bound the region in all directions, the algorithm may return a degenerate result or crash (division by zero in `lineIsect` for parallel boundary lines). Always add a bounding box.
- **`lineIsect` undefined for parallel lines** — `cross(a.d, b.d) ≈ 0` when two half-planes are nearly parallel, causing near-division by zero. After deduplication of same-angle half-planes this should not occur in the deque, but floating-point near-parallel pairs can still arise. Guard with `if (|cross(a.d, b.d)| < EPS) handle_parallel`.
- **C# `LinkedList` is slow** — the C# implementation uses `LinkedList<T>` for O(1) front/back operations. In practice `LinkedList` has large constant factors. For performance-critical code prefer `Deque` from a library or simulate with two stacks.

---

## Conclusion

Half-Plane Intersection is the **fundamental algorithm for 2D linear feasibility and convex region intersection**:

- The O(n log n) sort + O(n) deque sweep is asymptotically optimal (sorting n half-planes requires Ω(n log n)).
- The deque maintains the current "right chain" of the feasible region in angular order; each new half-plane trims infeasible vertices from both ends in amortized O(1) per element.
- Always add a bounding box before calling the algorithm — it ensures the intersection is bounded and avoids degenerate parallel-line intersections.
- The result is a convex polygon in CCW order; its area, centroid, and containment queries all run in O(n) or O(log n) using standard convex polygon algorithms.

**Key takeaway:** sort half-planes by the angle of their boundary direction, deduplicate parallel ones keeping the most restrictive, then run the deque sweep. The deque always holds a consistent chain — adding a new half-plane can only invalidate vertices at the ends of the chain, never in the middle, because angles increase monotonically.
