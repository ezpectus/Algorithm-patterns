# Voronoi Diagram — Topology of Nearest-Point Regions

---

## Origin & Motivation
The **Voronoi diagram** was introduced by Georgy Voronoi in 1908.  
It partitions the plane into regions, each corresponding to the nearest site from a given set of points.  
The idea: given a set of sites, each region contains all positions for which that site is the closest.  
It is fundamental in computational geometry, with applications in graphics, GIS, robotics, and data analysis.  
The dual structure is the **Delaunay triangulation**, which connects sites whose Voronoi cells share an edge.

---

## Core Idea
1. **Definition:**
   - Given a set of points P = {p1, p2, ..., pn} in the plane.  
   - The Voronoi region for a site pi is the set of all points closer to pi than to any other site.  

2. **Geometric structure:**
   - Each region is bounded by perpendicular bisectors between sites.  
   - Boundaries are line segments or rays where distances to two sites are equal.  
   - Voronoi vertices are points equidistant to three or more sites (circumcenters).  

3. **Topology:**
   - Regions form a mosaic covering of the plane.  
   - All regions are convex (in Euclidean geometry).  
   - The Voronoi diagram is the dual of the **Delaunay triangulation**.  

---

## Where It’s Used
- **Geography and cartography:** influence zones of cities, service distribution.  
- **Computer graphics:** texture generation, procedural landscapes.  
- **Robotics:** motion planning, space partitioning.  
- **Medicine:** modeling cellular structures.  
- **Data Science:** clustering, nearest-neighbor queries.  

---

## Algorithm Overview (Delaunay → Voronoi)
Voronoi diagrams are often constructed via the **Bowyer–Watson incremental Delaunay triangulation**, then converted into Voronoi cells.

1. **Initialization:**  
   - Start with a large “super-triangle” that contains all points.  

2. **Insert sites incrementally:**  
   - For each new site:  
     - Find all triangles whose circumcircle contains the site (“bad triangles”).  
     - Compute the boundary polygon of the cavity (edges not shared by two bad triangles).  
     - Remove bad triangles and re-triangulate the cavity by connecting the site to each boundary edge.  

3. **Cleanup:**  
   - After all sites are inserted, remove triangles that touch the super-triangle vertices.  

4. **Build Voronoi:**  
   - For each Delaunay triangle, compute the circumcenter → Voronoi vertex.  
   - For each site, collect adjacent Delaunay triangles and connect their circumcenters → Voronoi cell.  
   - For convex hull sites, Voronoi edges extend to infinity; in practice, clip to a bounding box.  

---

## Voronoi via Delaunay (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

struct Pt {
    double x, y;
};

struct Tri {
    int a, b, c;   // indices of points
    double cx, cy; // circumcenter
    double r2;     // squared circumradius
    bool valid = true;
};

static const double EPS = 1e-12;

double cross(const Pt& a, const Pt& b, const Pt& c) {
    return (b.x - a.x) * (c.y - a.y) - (b.y - a.y) * (c.x - a.x);
}

bool circumcircle(const Pt& a, const Pt& b, const Pt& c, double& cx, double& cy, double& r2) {
    double d = 2.0 * (a.x*(b.y-c.y) + b.x*(c.y-a.y) + c.x*(a.y-b.y));
    if (fabs(d) < EPS) return false; // collinear
    double a2 = a.x*a.x + a.y*a.y;
    double b2 = b.x*b.x + b.y*b.y;
    double c2 = c.x*c.x + c.y*c.y;
    cx = (a2*(b.y-c.y) + b2*(c.y-a.y) + c2*(a.y-b.y)) / d;
    cy = (a2*(c.x-b.x) + b2*(a.x-c.x) + c2*(b.x-a.x)) / d;
    r2 = (cx - a.x)*(cx - a.x) + (cy - a.y)*(cy - a.y);
    return true;
}

struct Edge {
    int u, v;
    bool operator<(const Edge& other) const {
        if (u != other.u) return u < other.u;
        return v < other.v;
    }
    bool operator==(const Edge& other) const {
        return u == other.u && v == other.v;
    }
};

Edge normEdge(int u, int v) {
    if (u > v) swap(u, v);
    return {u, v};
}

vector<Tri> delaunay(vector<Pt> pts) {
    int n = pts.size();
    // Build super-triangle large enough to contain all points
    double minx=pts[0].x, miny=pts[0].y, maxx=pts[0].x, maxy=pts[0].y;
    for (int i=1;i<n;i++){ minx=min(minx,pts[i].x); miny=min(miny,pts[i].y); maxx=max(maxx,pts[i].x); maxy=max(maxy,pts[i].y); }
    double dx = maxx - minx, dy = maxy - miny, dmax = max(dx, dy);
    double midx = (minx + maxx) / 2.0, midy = (miny + maxy) / 2.0;

    Pt pA{midx - 2*dmax, midy - dmax};
    Pt pB{midx,          midy + 2*dmax};
    Pt pC{midx + 2*dmax, midy - dmax};

    vector<Pt> P = pts;
    int A = P.size(); P.push_back(pA);
    int B = P.size(); P.push_back(pB);
    int C = P.size(); P.push_back(pC);

    vector<Tri> tris;
    Tri st;
    st.a = A; st.b = B; st.c = C;
    circumcircle(P[A], P[B], P[C], st.cx, st.cy, st.r2);
    tris.push_back(st);

    // Insert points incrementally
    for (int pi = 0; pi < n; ++pi) {
        Pt p = P[pi];
        vector<int> bad;
        for (int i = 0; i < (int)tris.size(); ++i) {
            Tri& t = tris[i]; if (!t.valid) continue;
            double dist2 = (p.x - t.cx)*(p.x - t.cx) + (p.y - t.cy)*(p.y - t.cy);
            if (dist2 <= t.r2 + 1e-10) { bad.push_back(i); t.valid = false; }
        }
        // Boundary edges: edges that belong to exactly one bad triangle
        map<Edge,int> edgeCount;
        auto addEdge = [&](int u, int v){
            Edge e = normEdge(u,v);
            edgeCount[e] += 1;
        };
        for (int bi : bad) {
            Tri& t = tris[bi];
            addEdge(t.a, t.b);
            addEdge(t.b, t.c);
            addEdge(t.c, t.a);
        }
        vector<Edge> boundary;
        for (auto& kv : edgeCount) {
            if (kv.second == 1) boundary.push_back(kv.first);
        }
        // Re-triangulate cavity
        for (auto e : boundary) {
            Tri nt;
            nt.a = e.u; nt.b = e.v; nt.c = pi;
            if (!circumcircle(P[nt.a], P[nt.b], P[nt.c], nt.cx, nt.cy, nt.r2)) continue;
            nt.valid = true;
            tris.push_back(nt);
        }
    }

    // Remove triangles touching super-triangle
    vector<Tri> out;
    for (auto& t : tris) {
        if (!t.valid) continue;
        if (t.a >= n || t.b >= n || t.c >= n) continue;
        out.push_back(t);
    }
    return out;
}

// Build Voronoi edges: for each pair of adjacent Delaunay triangles that share an edge,
// connect their circumcenters. Clip to bounding box if needed.
struct Segment { double x1,y1,x2,y2; };

vector<Segment> voronoiFromDelaunay(const vector<Pt>& pts, const vector<Tri>& tris) {
    // adjacency via shared edges
    map<Edge, vector<int>> edgeTris;
    for (int i=0;i<(int)tris.size();++i) {
        const Tri& t = tris[i];
        edgeTris[normEdge(t.a,t.b)].push_back(i);
        edgeTris[normEdge(t.b,t.c)].push_back(i);
        edgeTris[normEdge(t.c,t.a)].push_back(i);
    }
    vector<Segment> segs;
    for (auto& kv : edgeTris) {
        auto& v = kv.second;
        if (v.size() == 2) {
            const Tri& t1 = tris[v[0]];
            const Tri& t2 = tris[v[1]];
            segs.push_back({t1.cx, t1.cy, t2.cx, t2.cy});
        }
        // if v.size()==1, edge is on convex hull → Voronoi ray to infinity (clip if needed)
    }
    return segs;
}

int main() {
    vector<Pt> pts = {{0,0},{1,0},{0.5,0.8},{2,0},{1.5,0.7}};
    auto tris = delaunay(pts);
    auto segs = voronoiFromDelaunay(pts, tris);
    cout << "Voronoi edges: " << segs.size() << "\n";
    for (auto& s : segs) {
        cout << fixed << setprecision(6)
             << s.x1 << " " << s.y1 << " -> " << s.x2 << " " << s.y2 << "\n";
    }
    return 0;
}
```

## Voronoi via Delaunay (C#)
```cpp
using System;
using System.Collections.Generic;

public struct Pt { public double x, y; public Pt(double x, double y){ this.x=x; this.y=y; } }
public struct Tri {
    public int a, b, c;
    public double cx, cy, r2;
    public bool valid;
}
public struct Edge : IEquatable<Edge> {
    public int u, v;
    public Edge(int u, int v){ this.u=u; this.v=v; }
    public bool Equals(Edge other) => u==other.u && v==other.v;
    public override int GetHashCode() => u*73856093 ^ v*19349663;
}
public struct Segment {
    public double x1,y1,x2,y2;
    public Segment(double x1,double y1,double x2,double y2){ this.x1=x1; this.y1=y1; this.x2=x2; this.y2=y2; }
}

public class VoronoiBuilder {
    const double EPS = 1e-12;

    static Edge NormEdge(int u, int v){ if (u>v){ int t=u; u=v; v=t; } return new Edge(u,v); }

    static bool Circumcircle(Pt a, Pt b, Pt c, out double cx, out double cy, out double r2){
        double d = 2.0 * (a.x*(b.y-c.y) + b.x*(c.y-a.y) + c.x*(a.y-b.y));
        if (Math.Abs(d) < EPS) { cx=cy=r2=0; return false; }
        double a2 = a.x*a.x + a.y*a.y;
        double b2 = b.x*b.x + b.y*b.y;
        double c2 = c.x*c.x + c.y*c.y;
        cx = (a2*(b.y-c.y) + b2*(c.y-a.y) + c2*(a.y-b.y)) / d;
        cy = (a2*(c.x-b.x) + b2*(a.x-c.x) + c2*(b.x-a.x)) / d;
        r2 = (cx - a.x)*(cx - a.x) + (cy - a.y)*(cy - a.y);
        return true;
    }

    public static List<Tri> Delaunay(List<Pt> pts){
        int n = pts.Count;
        double minx=pts[0].x, miny=pts[0].y, maxx=pts[0].x, maxy=pts[0].y;
        for (int i=1;i<n;i++){ minx=Math.Min(minx,pts[i].x); miny=Math.Min(miny,pts[i].y); maxx=Math.Max(maxx,pts[i].x); maxy=Math.Max(maxy,pts[i].y); }
        double dx = maxx - minx, dy = maxy - miny, dmax = Math.Max(dx, dy);
        double midx = (minx + maxx)/2.0, midy = (miny + maxy)/2.0;

        Pt pA = new Pt(midx - 2*dmax, midy - dmax);
        Pt pB = new Pt(midx,           midy + 2*dmax);
        Pt pC = new Pt(midx + 2*dmax,  midy - dmax);

        var P = new List<Pt>(pts);
        int A = P.Count; P.Add(pA);
        int B = P.Count; P.Add(pB);
        int C = P.Count; P.Add(pC);

        var tris = new List<Tri>();
        Tri st = new Tri{ a=A, b=B, c=C, valid=true };
        Circumcircle(P[A], P[B], P[C], out st.cx, out st.cy, out st.r2);
        tris.Add(st);

        for (int pi=0; pi<n; ++pi){
            Pt p = P[pi];
            var bad = new List<int>();
            for (int i=0;i<tris.Count;i++){
                if (!tris[i].valid) continue;
                double dx2 = (p.x - tris[i].cx), dy2 = (p.y - tris[i].cy);
                double dist2 = dx2*dx2 + dy2*dy2;
                if (dist2 <= tris[i].r2 + 1e-10) { bad.Add(i); var t=tris[i]; t.valid=false; tris[i]=t; }
            }
            var edgeCount = new Dictionary<Edge,int>();
            void AddEdge(int u,int v){
                Edge e = NormEdge(u,v);
                edgeCount.TryGetValue(e, out int cnt);
                edgeCount[e] = cnt + 1;
            }
            foreach (int bi in bad){
                var t = tris[bi];
                AddEdge(t.a,t.b);
                AddEdge(t.b,t.c);
                AddEdge(t.c,t.a);
            }
            var boundary = new List<Edge>();
            foreach (var kv in edgeCount){
                if (kv.Value == 1) boundary.Add(kv.Key);
            }
            foreach (var e in boundary){
                Tri nt = new Tri{ a=e.u, b=e.v, c=pi, valid=true };
                if (!Circumcircle(P[nt.a], P[nt.b], P[nt.c], out nt.cx, out nt.cy, out nt.r2)) continue;
                tris.Add(nt);
            }
        }

        var outTris = new List<Tri>();
        for (int i=0;i<tris.Count;i++){
            var t = tris[i];
            if (!t.valid) continue;
            if (t.a>=n || t.b>=n || t.c>=n) continue;
            outTris.Add(t);
        }
        return outTris;
    }

    public static List<Segment> VoronoiFromDelaunay(List<Pt> pts, List<Tri> tris){
        var edgeTris = new Dictionary<Edge,List<int>>();
        for (int i=0;i<tris.Count;i++){
            var t = tris[i];
            Edge e1 = NormEdge(t.a,t.b), e2 = NormEdge(t.b,t.c), e3 = NormEdge(t.c,t.a);
            void Add(Edge e){
                if (!edgeTris.TryGetValue(e, out var list)){ list = new List<int>(); edgeTris[e]=list; }
                list.Add(i);
            }
            Add(e1); Add(e2); Add(e3);
        }
        var segs = new List<Segment>();
        foreach (var kv in edgeTris){
            var list = kv.Value;
            if (list.Count == 2){
                var t1 = tris[list[0]];
                var t2 = tris[list[1]];
                segs.Add(new Segment(t1.cx,t1.cy,t2.cx,t2.cy));
            }
            // if Count==1 → hull edge → Voronoi ray to infinity (clip in application)
        }
        return segs;
    }
}

public class Program {
    public static void Main(){
        var pts = new List<Pt>{ new Pt(0,0), new Pt(1,0), new Pt(0.5,0.8), new Pt(2,0), new Pt(1.5,0.7) };
        var tris = VoronoiBuilder.Delaunay(pts);
        var segs = VoronoiBuilder.VoronoiFromDelaunay(pts, tris);
        Console.WriteLine($"Voronoi edges: {segs.Count}");
        foreach (var s in segs)
            Console.WriteLine($"{s.x1:F6} {s.y1:F6} -> {s.x2:F6} {s.y2:F6}");
    }
}
```

## Complexity
- **Time:**  
  - Naive: O(n²).  
  - Delaunay-based: O(n log n).  
- **Space:** O(n), storing vertices and edges of the diagram.  

---

## Key Concepts Recap
- **Voronoi region:** set of points closest to one site.  
- **Boundaries:** perpendicular bisectors.  
- **Vertices:** points equidistant to three or more sites.  
- **Duality:** Voronoi ↔ Delaunay.  

---

✅ **Conclusion**  
The Voronoi diagram is a fundamental tool in topology and computational geometry:  
- A simple idea of “nearest site” becomes a powerful method for spatial analysis.  
- It connects geometry, algorithms, and practical applications.  
- In combination with Delaunay triangulation, it forms the basis of many algorithms in graphics, GIS, and data science.

---
