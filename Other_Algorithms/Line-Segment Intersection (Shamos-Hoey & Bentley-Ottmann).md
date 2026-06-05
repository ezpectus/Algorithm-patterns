# Line-Segment Intersection (Shamos-Hoey & Bentley-Ottmann)

## Origin & Motivation

**Line-segment intersection** is one of the foundational problems in computational geometry. Given a set of `n` line segments in the plane, two classical algorithms address it at different levels:

**Shamos-Hoey (1976)** — answers the yes/no question: *do any two segments intersect?* in **O(n log n)** time, which is optimal since the output is a single bit but the input has Ω(n log n) information-theoretic complexity under comparison-based models.

**Bentley-Ottmann (1979)** — enumerates *all* `k` intersection points in **O((n + k) log n)** time, improving on the naive O(n²) brute force when `k` is small. It introduced the **sweep line paradigm** that became the backbone of most computational geometry algorithms.

**The problem they solve:** The naive approach checks all O(n²) pairs. For n = 10⁵ segments that is 10¹⁰ checks — infeasible. Both algorithms exploit the plane-sweep structure: a vertical line moves left to right, maintaining only the segments it currently crosses (the *status*). Two segments can only intersect after they become neighbors in the status — so only O(n + k) neighbor pairs ever need checking.

Complexity: Shamos-Hoey **O(n log n)**, Bentley-Ottmann **O((n + k) log n)**, both **O(n)** space.

---

## Where It Is Used

- GIS and map overlay: detecting road/polygon intersections
- CAD/CAM: collision detection between geometric primitives
- VLSI design rule checking: wire intersection detection
- Robot motion planning: obstacle boundary intersections
- Computer graphics: ray casting, polygon clipping (Sutherland-Hodgman uses related sweep ideas)
- Computational geometry libraries: CGAL, Boost.Geometry, Clipper

---

## Core Concepts

### Sweep Line and Event Queue

Both algorithms share the same skeleton:

```
Event queue Q  — priority queue ordered by x (then y for ties)
Status S       — balanced BST of segments ordered by their
                 y-coordinate at the current sweep line x

Events:
  LEFT(s)         — left endpoint of segment s
  RIGHT(s)        — right endpoint of segment s
  INTERSECTION(s,t,p) — intersection point p of segments s and t
```

The sweep line moves from left to right, processing events in order. At each event the status changes — segments are inserted, removed, or swapped — and new intersection events may be discovered.

### Key Invariant

At any sweep line position `x`, the status `S` orders segments by their y-value at `x`. Two segments `s` and `t` can only intersect *after* they become adjacent in `S`. This means:

- When inserting `s`: check `s` against its immediate neighbors in `S`.
- When removing `s`: check the two segments that become new neighbors.
- When processing an intersection of `s` and `t`: swap their order in `S`, then check each against its new outer neighbor.

### Cross Product Test

The fundamental geometric primitive for both algorithms:

```
cross(a, b) = a.x * b.y - a.y * b.x

Segments [a,b] and [c,d] intersect iff:
  cross(d-c, a-c) and cross(d-c, b-c) have opposite signs  (proper crossing)
  OR one of a,b,c,d lies on the other segment               (degenerate)
```

---

## Shamos-Hoey Algorithm

Detects whether *any* intersection exists. Never adds intersection events to the queue — exits as soon as one is found.

```
ShamasHoey(segments):
    orient each segment left-to-right
    build event queue Q with LEFT/RIGHT events only
    S = empty BST (ordered by y at sweep x)

    for each event e in Q (left to right):
        if e = LEFT(s):
            insert s into S
            above = successor(s) in S
            below = predecessor(s) in S
            if intersects(s, above): return TRUE
            if intersects(s, below): return TRUE

        if e = RIGHT(s):
            above = successor(s) in S
            below = predecessor(s) in S
            if above and below exist and intersects(above, below):
                return TRUE
            remove s from S

    return FALSE
```

**Why it works:** If any intersection exists at point `p`, then just before the sweep line reaches `p`, the two intersecting segments must be adjacent in `S` (can be proven by induction on the event structure). The algorithm checks every adjacency change, so it cannot miss the first intersection.

---

## Bentley-Ottmann Algorithm

Enumerates all `k` intersections. Adds intersection events to Q when two segments become adjacent.

```
BentleyOttmann(segments):
    orient each segment left-to-right
    build Q with LEFT/RIGHT events
    S = empty BST

    while Q not empty:
        e = Q.pop()

        if e = LEFT(s):
            insert s into S
            above = successor(s),  below = predecessor(s)
            schedule_if_future(intersect(s, above))
            schedule_if_future(intersect(s, below))

        if e = RIGHT(s):
            above = successor(s),  below = predecessor(s)
            schedule_if_future(intersect(above, below))
            remove s from S

        if e = INTERSECTION(s, t, p):
            output p
            swap s and t in S   // their order reverses after crossing
            // s was below t, now s is above t
            new_above = successor(t_new_position)
            new_below = predecessor(s_new_position)
            schedule_if_future(intersect(t, new_above))
            schedule_if_future(intersect(s, new_below))

schedule_if_future(s, t):
    if s and t intersect at point p with p.x > sweep_x:
        if (s,t) not already in Q: add INTERSECTION(s,t,p) to Q
```

**The swap step** is critical: after two segments cross, their vertical order reverses, so their new outer neighbors must be checked for future intersections.

---

## Complexity Analysis

### Shamos-Hoey O(n log n)

- Event queue: 2n events, each processed in O(log n) (BST insert/delete + two `intersects` checks).
- Total: O(n log n).
- Space: O(n) for queue and status.

### Bentley-Ottmann O((n + k) log n)

- Events: 2n endpoint events + at most k intersection events = O(n + k) total.
- Each event: O(log n) BST operations + O(log n) queue operations.
- Total: O((n + k) log n).
- Space: O(n + k) — the queue can hold O(n + k) events.

**Note:** when k = O(n²) (all pairs intersect), Bentley-Ottmann is O(n² log n) — not better than brute force. The algorithm is beneficial when k is subquadratic.

---

## Complexity Summary

| Algorithm | Time | Space | Output |
|---|---|---|---|
| Brute force | O(n²) | O(1) | All k intersections |
| Shamos-Hoey | O(n log n) | O(n) | Yes/No |
| Bentley-Ottmann | O((n+k) log n) | O(n+k) | All k intersection points |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ld = long double;
const ld EPS = 1e-9;

// ================================================================
// GEOMETRY PRIMITIVES
// ================================================================
struct P {
    ld x, y;
    P(ld x=0, ld y=0) : x(x), y(y) {}
    P operator-(P o) const { return {x-o.x, y-o.y}; }
    P operator+(P o) const { return {x+o.x, y+o.y}; }
    P operator*(ld t) const { return {x*t, y*t}; }
    bool operator<(P o) const {
        if (abs(x-o.x) > EPS) return x < o.x;
        return y < o.y - EPS;
    }
    bool operator==(P o) const { return abs(x-o.x)<EPS && abs(y-o.y)<EPS; }
};

ld cross(P a, P b) { return a.x*b.y - a.y*b.x; }
ld dot  (P a, P b) { return a.x*b.x + a.y*b.y; }
int sign(ld v)     { return v>EPS ? 1 : v<-EPS ? -1 : 0; }

bool on_seg(P p, P a, P b) {
    return sign(cross(b-a, p-a)) == 0
        && sign(dot(p-a, b-a)) >= 0
        && sign(dot(p-b, a-b)) >= 0;
}

// Do segments [a,b] and [c,d] intersect?
bool seg_intersect(P a, P b, P c, P d) {
    int d1=sign(cross(d-c,a-c)), d2=sign(cross(d-c,b-c));
    int d3=sign(cross(b-a,c-a)), d4=sign(cross(b-a,d-a));
    if (d1*d2 < 0 && d3*d4 < 0) return true;
    if (!d1 && on_seg(a,c,d)) return true;
    if (!d2 && on_seg(b,c,d)) return true;
    if (!d3 && on_seg(c,a,b)) return true;
    if (!d4 && on_seg(d,a,b)) return true;
    return false;
}

// Intersection point of lines through [a,b] and [c,d]
bool line_isect(P a, P b, P c, P d, P& out) {
    ld A1=b.y-a.y, B1=a.x-b.x, C1=A1*a.x+B1*a.y;
    ld A2=d.y-c.y, B2=c.x-d.x, C2=A2*c.x+B2*c.y;
    ld det = A1*B2 - A2*B1;
    if (abs(det) < EPS) return false;
    out = {(C1*B2-C2*B1)/det, (A1*C2-A2*C1)/det};
    return true;
}

struct Seg { P a, b; int id; };

// ================================================================
// SHAMOS-HOEY — O(n log n) any-intersection detection
// ================================================================
static ld SX; // current sweep x coordinate

ld seg_y(const Seg* s, ld x) {
    if (abs(s->b.x - s->a.x) < EPS) return max(s->a.y, s->b.y);
    ld t = (x - s->a.x) / (s->b.x - s->a.x);
    return s->a.y + t*(s->b.y - s->a.y);
}

struct CmpSH {
    bool operator()(const Seg* a, const Seg* b) const {
        ld ya = seg_y(a, SX), yb = seg_y(b, SX);
        if (abs(ya-yb) > EPS) return ya < yb;
        return a->id < b->id;
    }
};

bool shamos_hoey(vector<Seg> segs) {
    for (auto& s : segs) if (s.b < s.a) swap(s.a, s.b);

    struct Ev {
        ld x; int t; Seg* s;
        bool operator<(const Ev& o) const {
            if (abs(x-o.x) > EPS) return x < o.x;
            return t < o.t;
        }
    };
    vector<Ev> evs;
    for (auto& s : segs) {
        evs.push_back({s.a.x, 0, &s});
        evs.push_back({s.b.x, 1, &s});
    }
    sort(evs.begin(), evs.end());

    set<Seg*, CmpSH> A;
    for (auto& e : evs) {
        SX = e.x; Seg* s = e.s;
        if (e.t == 0) { // left endpoint — insert
            auto [it, ok] = A.insert(s);
            auto nxt = next(it);
            if (nxt != A.end() &&
                seg_intersect(s->a,s->b,(*nxt)->a,(*nxt)->b)) return true;
            if (it != A.begin() &&
                seg_intersect(s->a,s->b,(*prev(it))->a,(*prev(it))->b)) return true;
        } else { // right endpoint — remove
            auto it = A.find(s);
            if (it != A.end()) {
                auto nxt = next(it);
                if (it != A.begin() && nxt != A.end() &&
                    seg_intersect((*prev(it))->a,(*prev(it))->b,
                                  (*nxt)->a,(*nxt)->b)) return true;
                A.erase(it);
            }
        }
    }
    return false;
}

// ================================================================
// BENTLEY-OTTMANN — O((n+k) log n) all intersections
// ================================================================
struct BentleyOttmann {
    struct Ev {
        ld x, y; int type; // 0=left, 1=intersect, 2=right
        int id1, id2;
        bool operator>(const Ev& o) const {
            if (abs(x-o.x) > EPS) return x > o.x;
            if (abs(y-o.y) > EPS) return y > o.y;
            return type > o.type;
        }
    };

    vector<Seg>* segs;
    ld sx = 0;

    ld yat(int id) const {
        auto& s = (*segs)[id];
        if (abs(s.b.x-s.a.x) < EPS) return (s.a.y+s.b.y)/2.0L;
        ld t = max((ld)0, min((ld)1, (sx-s.a.x)/(s.b.x-s.a.x)));
        return s.a.y + t*(s.b.y-s.a.y);
    }

    struct Cmp {
        BentleyOttmann* bo;
        bool operator()(int a, int b) const {
            ld ya = bo->yat(a), yb = bo->yat(b);
            if (abs(ya-yb) > EPS) return ya < yb;
            return a < b;
        }
    };

    set<int, Cmp>                               status{Cmp{this}};
    priority_queue<Ev,vector<Ev>,greater<Ev>>   pq;
    set<pair<int,int>>                          scheduled;
    vector<P>                                   result;

    void try_schedule(int i, int j) {
        if (i < 0 || j < 0) return;
        if (i > j) swap(i, j);
        if (scheduled.count({i,j})) return;
        auto& si = (*segs)[i]; auto& sj = (*segs)[j];
        if (!seg_intersect(si.a,si.b,sj.a,sj.b)) return;
        P pt;
        if (!line_isect(si.a,si.b,sj.a,sj.b,pt))
            pt = max(min(si.a,si.b), min(sj.a,sj.b));
        if (pt.x > sx + EPS) {
            scheduled.insert({i,j});
            pq.push({pt.x, pt.y, 1, i, j});
        }
    }

    int above(set<int,Cmp>::iterator it) {
        auto nx = next(it);
        return (nx != status.end()) ? *nx : -1;
    }
    int below(set<int,Cmp>::iterator it) {
        return (it != status.begin()) ? *prev(it) : -1;
    }

    vector<P> run(vector<Seg>& ss) {
        segs = &ss;
        for (auto& s : ss) if (s.b < s.a) swap(s.a, s.b);
        for (auto& s : ss) {
            pq.push({s.a.x, s.a.y, 0, s.id, -1});
            pq.push({s.b.x, s.b.y, 2, s.id, -1});
        }
        while (!pq.empty()) {
            auto e = pq.top(); pq.pop();
            sx = e.x;
            if (e.type == 0) {                    // ---- INSERT ----
                int id = e.id1;
                auto [it, ok] = status.insert(id);
                try_schedule(id, above(it));
                try_schedule(id, below(it));
            } else if (e.type == 2) {             // ---- REMOVE ----
                int id = e.id1;
                auto it = status.find(id);
                if (it != status.end()) {
                    int ab = above(it), bl = below(it);
                    try_schedule(ab, bl);
                    status.erase(it);
                }
            } else {                              // ---- SWAP ----
                int i = e.id1, j = e.id2;
                // report intersection
                auto& si = (*segs)[i]; auto& sj = (*segs)[j];
                P pt;
                if (!line_isect(si.a,si.b,sj.a,sj.b,pt))
                    pt = max(min(si.a,si.b), min(sj.a,sj.b));
                result.push_back(pt);
                // ensure i is the one currently below j
                if (yat(i) > yat(j)) swap(i, j);
                auto it_i = status.find(i);
                auto it_j = status.find(j);
                if (it_i == status.end() || it_j == status.end()) continue;
                int ab = above(it_j), bl = below(it_i);
                // swap: erase, nudge sx, re-insert
                status.erase(it_i); status.erase(it_j);
                sx += 2*EPS;
                status.insert(i); status.insert(j);
                sx = e.x;
                // i is now above j — check new outer neighbors
                try_schedule(i, ab);
                try_schedule(j, bl);
            }
        }
        return result;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // ---- Shamos-Hoey ----
    {
        printf("=== Shamos-Hoey ===\n");
        vector<Seg> yes = {{{0,0},{4,4},0}, {{0,4},{4,0},1}};
        vector<Seg> no  = {{{0,0},{1,0},0}, {{2,0},{3,0},1}, {{0,2},{1,2},2}};
        printf("Crossing X:    %s\n", shamos_hoey(yes) ? "YES":"NO"); // YES
        printf("No crossings:  %s\n", shamos_hoey(no)  ? "YES":"NO"); // NO
    }
    // ---- Bentley-Ottmann ----
    {
        printf("\n=== Bentley-Ottmann ===\n");
        // Three segments: two diagonals + one horizontal
        vector<Seg> segs = {
            {{0,0},{4,4},0},
            {{0,4},{4,0},1},
            {{-1,2},{5,2},2}
        };
        BentleyOttmann bo;
        auto pts = bo.run(segs);
        printf("Intersections found: %d\n", (int)pts.size());
        sort(pts.begin(), pts.end());
        for (auto& p : pts)
            printf("  (%.4Lf, %.4Lf)\n", p.x, p.y);
        // Expected: (2,2) [diagonals] and two points where horizontal crosses each diagonal
    }
    // ---- Stress: Shamos-Hoey vs brute force ----
    {
        srand(42); int errors = 0;
        for (int t = 0; t < 2000; t++) {
            int n = 2 + rand()%8;
            vector<Seg> s(n);
            for (int i = 0; i < n; i++) {
                s[i] = {{ld(rand()%20),ld(rand()%20)},
                        {ld(rand()%20),ld(rand()%20)}, i};
                if (s[i].a == s[i].b) s[i].b.x++;
            }
            // brute force
            bool exp = false;
            for (int i=0;i<n&&!exp;i++)
                for (int j=i+1;j<n&&!exp;j++)
                    exp = seg_intersect(s[i].a,s[i].b,s[j].a,s[j].b);
            if (shamos_hoey(s) != exp) errors++;
        }
        printf("\nShamos-Hoey stress 2000: %s\n",
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

// ================================================================
// Line-Segment Intersection — C#
// Shamos-Hoey + Bentley-Ottmann
// ================================================================
public static class LSI
{
    const double EPS = 1e-9;

    public struct Pt
    {
        public double X, Y;
        public Pt(double x, double y) { X=x; Y=y; }
        public static Pt operator-(Pt a, Pt b) => new Pt(a.X-b.X, a.Y-b.Y);
        public static Pt operator+(Pt a, Pt b) => new Pt(a.X+b.X, a.Y+b.Y);
        public int CompareTo(Pt o) {
            if (Math.Abs(X-o.X) > EPS) return X < o.X ? -1 : 1;
            return Math.Abs(Y-o.Y) < EPS ? 0 : Y < o.Y ? -1 : 1;
        }
    }

    public struct Seg { public Pt A, B; public int Id; }

    static double Cross(Pt a, Pt b) => a.X*b.Y - a.Y*b.X;
    static double Dot  (Pt a, Pt b) => a.X*b.X + a.Y*b.Y;
    static int    Sign (double v)   => v>EPS?1:v<-EPS?-1:0;

    static bool OnSeg(Pt p, Pt a, Pt b) =>
        Sign(Cross(b-a,p-a))==0 && Sign(Dot(p-a,b-a))>=0 && Sign(Dot(p-b,a-b))>=0;

    static bool Intersects(Pt a, Pt b, Pt c, Pt d) {
        int d1=Sign(Cross(d-c,a-c)), d2=Sign(Cross(d-c,b-c));
        int d3=Sign(Cross(b-a,c-a)), d4=Sign(Cross(b-a,d-a));
        if (d1*d2<0&&d3*d4<0) return true;
        return (!d1&&OnSeg(a,c,d))||(!d2&&OnSeg(b,c,d))||
               (!d3&&OnSeg(c,a,b))||(!d4&&OnSeg(d,a,b));
    }

    static bool LineIsect(Pt a,Pt b,Pt c,Pt d,out Pt p) {
        double A1=b.Y-a.Y,B1=a.X-b.X,C1=A1*a.X+B1*a.Y;
        double A2=d.Y-c.Y,B2=c.X-d.X,C2=A2*c.X+B2*c.Y;
        double det=A1*B2-A2*B1;
        if (Math.Abs(det)<EPS) { p=default; return false; }
        p=new Pt((C1*B2-C2*B1)/det,(A1*C2-A2*C1)/det);
        return true;
    }

    // ---- Shamos-Hoey ----
    static double _sx;
    static double SegY(Seg s, double x) {
        if (Math.Abs(s.B.X-s.A.X)<EPS) return Math.Max(s.A.Y,s.B.Y);
        double t=(x-s.A.X)/(s.B.X-s.A.X);
        return s.A.Y+t*(s.B.Y-s.A.Y);
    }

    class CmpSH : IComparer<Seg>
    {
        public int Compare(Seg a, Seg b) {
            double ya=SegY(a,_sx), yb=SegY(b,_sx);
            if (Math.Abs(ya-yb)>EPS) return ya<yb?-1:1;
            return a.Id.CompareTo(b.Id);
        }
    }

    public static bool ShamasHoey(Seg[] segs) {
        var s = (Seg[])segs.Clone();
        for (int i=0;i<s.Length;i++)
            if (s[i].B.CompareTo(s[i].A)<0) { var t=s[i].A; s[i].A=s[i].B; s[i].B=t; }

        var evs = new List<(double x,int t,int idx)>();
        for (int i=0;i<s.Length;i++) {
            evs.Add((s[i].A.X,0,i));
            evs.Add((s[i].B.X,1,i));
        }
        evs.Sort((a,b)=>{
            int c=a.x.CompareTo(b.x); return c!=0?c:a.t.CompareTo(b.t);});

        var A = new SortedSet<Seg>(new CmpSH());
        foreach (var (x,t,idx) in evs) {
            _sx = x;
            if (t==0) {
                A.Add(s[idx]);
                var nxt = A.GetViewBetween(s[idx],A.Max); nxt.Min.Equals(s[idx]);
                // check neighbors via Min/Max of views
                Seg? above=null, below=null;
                foreach (var seg in A) if (seg.Id==s[idx].Id) {
                    // find neighbors manually
                    break;
                }
                // Simplified: check all adjacent pairs on insert/remove
                // For production use, wrap in a list for O(log n) neighbor access
            } else {
                A.Remove(s[idx]);
            }
        }
        // Note: C# SortedSet doesn't expose O(1) predecessor/successor.
        // In practice use a sorted list or custom AVL for O(log n) neighbors.
        // The logic below uses a sorted list for clarity:
        return ShamasHoeyList(segs);
    }

    public static bool ShamasHoeyList(Seg[] segs) {
        var s = (Seg[])segs.Clone();
        for (int i=0;i<s.Length;i++)
            if (s[i].B.CompareTo(s[i].A)<0) { var t=s[i].A; s[i].A=s[i].B; s[i].B=t; }

        var evs = new List<(double x,int t,int idx)>();
        for (int i=0;i<s.Length;i++) {
            evs.Add((s[i].A.X,0,i)); evs.Add((s[i].B.X,1,i));
        }
        evs.Sort((a,b)=>{ int c=a.x.CompareTo(b.x); return c!=0?c:a.t.CompareTo(b.t); });

        // Use sorted list keyed by (y_at_sweep, id)
        var active = new SortedList<(double y,int id),int>();
        (double,int) Key(int idx) => (SegY(s[idx],_sx), s[idx].Id);

        foreach (var (x,t,idx) in evs) {
            _sx = x;
            if (t == 0) {
                var k = Key(idx);
                active[k] = idx;
                int pos = active.IndexOfKey(k);
                if (pos+1 < active.Count) {
                    int nb = active.Values[pos+1];
                    if (Intersects(s[idx].A,s[idx].B,s[nb].A,s[nb].B)) return true;
                }
                if (pos > 0) {
                    int nb = active.Values[pos-1];
                    if (Intersects(s[idx].A,s[idx].B,s[nb].A,s[nb].B)) return true;
                }
            } else {
                var k = Key(idx);
                int pos = active.IndexOfKey(k);
                if (pos > 0 && pos+1 < active.Count) {
                    int nb1=active.Values[pos-1], nb2=active.Values[pos+1];
                    if (Intersects(s[nb1].A,s[nb1].B,s[nb2].A,s[nb2].B)) return true;
                }
                active.Remove(k);
            }
        }
        return false;
    }

    public static void Main() {
        var yes = new Seg[]{ new(){A=new Pt(0,0),B=new Pt(4,4),Id=0},
                             new(){A=new Pt(0,4),B=new Pt(4,0),Id=1} };
        var no  = new Seg[]{ new(){A=new Pt(0,0),B=new Pt(1,0),Id=0},
                             new(){A=new Pt(2,0),B=new Pt(3,0),Id=1},
                             new(){A=new Pt(0,2),B=new Pt(1,2),Id=2} };
        Console.WriteLine("Crossing X:   " + ShamasHoeyList(yes)); // True
        Console.WriteLine("No crossings: " + ShamasHoeyList(no));  // False
    }
}
```

---

## Numerical Robustness — The Core Challenge

Both algorithms rely on comparing y-values of segments at the sweep line and on exact intersection computations. Floating-point arithmetic introduces errors that can cause:

```
Problem: segments s and t are adjacent in status but the
         computed intersection lies "in the past" (x < sweep_x)
         due to floating-point rounding → intersection missed

Problem: two segments computed as non-intersecting by cross product
         but visually touching at an endpoint → status corrupted

Common fixes:
  - Use exact arithmetic (CGAL uses exact kernel with lazy evaluation)
  - Use integer coordinates with exact integer cross products
  - Add epsilon guards: only schedule future intersections (x > sweep_x + EPS)
  - Symbolic perturbation (SOS) to make all coordinates algebraically distinct
```

For competitive programming, integer coordinates with `long long` cross products are sufficient. For production geometry libraries, exact arithmetic is required.

---

## Pitfalls

- **Status comparator must use current sweep x** — the BST ordering depends on `sweep_x` at the moment of comparison. Caching y-values leads to stale comparisons when `sweep_x` changes between the insert and a subsequent lookup. Always recompute `y_at(segment, sweep_x)` inside the comparator.
- **Intersection events must only be scheduled in the future** — adding an intersection at `x ≤ sweep_x` causes the algorithm to re-process already-handled events, potentially looping. Always check `intersection.x > sweep_x + EPS` before pushing to the queue.
- **Each pair scheduled at most once** — without a `scheduled` set, the same pair `(s, t)` gets pushed to the queue multiple times (once when they become adjacent, again after a swap), causing duplicate output and O(n²) queue size. Track every `(id1, id2)` pair added.
- **Segment endpoints coinciding with intersections** — when a segment endpoint falls exactly on another segment the degenerate case must be handled in the cross product test (`on_seg` check). Missing this makes segments appear non-intersecting when they share an endpoint.
- **Swap direction at intersection events** — at an intersection event for `(s, t)`, `s` was below `t` just before the event and becomes above after. The swap must use the y-values *just after* `sweep_x + EPS` to determine new order. Using y-values at exactly `sweep_x` gives the wrong order (they're equal at the crossing point).
- **Vertical segments** — a segment with `a.x == b.x` has undefined y-at-x for all sweep positions except its own x. Treat vertical segments specially: generate a single event at their x, check against all segments in the status at that sweep position.
- **C# SortedSet lacks O(log n) predecessor/successor** — `SortedSet<T>` in C# has no `GetPredecessor`/`GetSuccessor` API; iterating to find neighbors is O(n). For a production implementation use a custom balanced BST, `SortedList`, or wrap with index tracking.

---

## Conclusion

Shamos-Hoey and Bentley-Ottmann are the **canonical sweep-line algorithms for segment intersection**:

- Both maintain a status structure (segments ordered by y at current sweep x) and an event queue (endpoint and intersection events ordered by x).
- Shamos-Hoey stops at the first intersection found — O(n log n) regardless of k. Bentley-Ottmann reports all k intersections in O((n+k) log n).
- The key invariant is that intersecting segments must become adjacent in the status before their intersection is swept — so only O(n + k) adjacency checks are needed, not O(n²).
- The swap step in Bentley-Ottmann is the most subtle part: at an intersection the two segments exchange positions in the status, creating new adjacencies that must be checked for future intersections.

**Key takeaway:** the sweep line reduces a 2D problem to a sequence of 1D problems. The status at each sweep position is a 1D sorted structure; events drive transitions between states. This paradigm — event queue + sweep line status — is the foundation of polygon clipping, Voronoi diagram construction (Fortune's algorithm), and triangulation algorithms.
