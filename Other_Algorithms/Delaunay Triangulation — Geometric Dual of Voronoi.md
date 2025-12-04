# Delaunay Triangulation — Geometric Dual of Voronoi

---

## Origin & Motivation
The **Delaunay triangulation** was introduced by Boris Delaunay in 1934.  
It is a fundamental structure in computational geometry, closely related to the Voronoi diagram.  
Given a set of points in the plane, the Delaunay triangulation connects them into triangles such that no point lies inside the circumcircle of any triangle.  
This property maximizes the minimum angle of all triangles, avoiding skinny triangles and producing well-shaped meshes.

---

## Core Idea
- **Definition:** A triangulation of a set of points is Delaunay if no point lies inside the circumcircle of any triangle.  
- **Geometric properties:**
  - Maximizes the minimum angle (avoids sliver triangles).  
  - Dual to the Voronoi diagram: each edge in Delaunay corresponds to adjacent Voronoi cells.  
  - Unique for points in general position (no four points cocircular).  
- **Applications:** mesh generation, interpolation, terrain modeling, finite element methods, and computer graphics.

---

## Algorithm Overview
Several algorithms exist to construct Delaunay triangulations:

1. **Incremental (Bowyer–Watson):**
   - Start with a large super-triangle containing all points.  
   - Insert points one by one, removing triangles whose circumcircle contains the point.  
   - Re-triangulate the resulting cavity.  
   - Complexity: average O(n log n), worst-case O(n²).

2. **Divide and Conquer:**
   - Recursively split points, triangulate subsets, and merge.  
   - Complexity: O(n log n).

3. **Sweep Line (Fortune’s algorithm):**
   - Sweep a line across the plane, maintaining a beach line structure.  
   - Complexity: O(n log n).  
   - Often used for Voronoi diagrams, with Delaunay as the dual.

---



---

## Delaunay Triangulation (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

const double EPS = 1e-9;

struct Point {
    double x, y;
    int id;
    Point(double x = 0, double y = 0, int id = -1) : x(x), y(y), id(id) {}
    Point operator-(const Point& p) const { return {x - p.x, y - p.y}; }
    double cross(const Point& p) const { return x * p.y - y * p.x; }
    double dist2() const { return x * x + y * y; }
};

struct Triangle {
    int a, b, c;
    Triangle(int a = 0, int b = 0, int c = 0) : a(a), b(b), c(c) {}
    bool containsVertex(int v) const { return a == v || b == v || c == v; }
};

class DelaunayTriangulation {
private:
    vector<Point> points;
    vector<Triangle> triangles;
    vector<vector<int>> adj;  // adjacency list: triangle index → neighboring triangles

    // Check if point d is inside circumcircle of triangle abc
    bool inCircumcircle(const Point& a, const Point& b, const Point& c, const Point& d) {
        double ax = a.x - d.x, ay = a.y - d.y;
        double bx = b.x - d.x, by = b.y - d.y;
        double cx = c.x - d.x, cy = c.y - d.y;

        double det = ax * (by * (cx * cx + cy * cy) - cy * (bx * bx + by * by)) -
                     ay * (bx * (cx * cx + cy * cy) - cx * (bx * bx + by * by)) +
                     bx * (ay * (cx * cx + cy * cy) - cy * (ax * ax + ay * ay)) -
                     by * (ax * (cx * cx + cy * cy) - cx * (ax * ax + ay * ay));

        return det > EPS;
    }

    // Add super triangle (big enough to contain all points)
    void addSuperTriangle() {
        double minX = points[0].x, maxX = points[0].x;
        double minY = points[0].y, maxY = points[0].y;
        for (auto& p : points) {
            minX = min(minX, p.x); maxX = max(maxX, p.x);
            minY = min(minY, p.y); maxY = max(maxY, p.y);
        }
        double dx = maxX - minX + 1;
        double dy = maxY - minY + 1;
        double midX = (minX + maxX) / 2;
        double midY = (minY + maxY) / 2;

        points.push_back(Point(midX - 20 * dx, midY - dy, -1));
        points.push_back(Point(midX + 20 * dx, midY - dy, -1));
        points.push_back(Point(midX, midY + 20 * dy, -1));

        triangles.emplace_back(points.size() - 3, points.size() - 2, points.size() - 1);
    }

    // Find triangles that contain point p in their circumcircle
    vector<int> findBadTriangles(int p_idx) {
        vector<int> bad;
        for (int i = 0; i < triangles.size(); i++) {
            auto& t = triangles[i];
            if (inCircumcircle(points[t.a], points[t.b], points[t.c], points[p_idx])) {
                bad.push_back(i);
            }
        }
        return bad;
    }

    // Build polygon (cavity boundary) from bad triangles
    vector<pair<int, int>> getPolygon(const vector<int>& bad) {
        unordered_map<long long, int> edgeCount;
        vector<pair<int, int>> edges;

        for (int idx : bad) {
            auto& t = triangles[idx];
            vector<pair<int, int>> triEdges = {{t.a, t.b}, {t.b, t.c}, {t.c, t.a}};
            for (auto& e : triEdges) {
                if (e.first > e.second) swap(e.first, e.second);
                long long key = (long long)e.first * points.size() + e.second;
                if (++edgeCount[key] == 2) edgeCount[key] = -100; // shared edge
            }
        }

        for (auto& [key, cnt] : edgeCount) {
            if (cnt == 1) {
                int a = key / points.size();
                int b = key % points.size();
                edges.emplace_back(a, b);
            }
        }
        return edges;
    }

public:
    DelaunayTriangulation(vector<Point> pts) : points(move(pts)) {
        if (points.size() < 3) return;

        addSuperTriangle();
        int n = points.size();

        for (int i = 0; i < n - 3; i++) {
            vector<int> bad = findBadTriangles(i);
            if (bad.empty()) continue;

            vector<pair<int, int>> polygon = getPolygon(bad);

            // Remove bad triangles
            vector<Triangle> newTris;
            for (int j = 0; j < triangles.size(); j++) {
                bool isBad = false;
                for (int k : bad) if (k == j) { isBad = true; break; }
                if (!isBad) newTris.push_back(triangles[j]);
            }
            triangles = move(newTris);

            // Add new triangles from polygon + new point
            for (auto& e : polygon) {
                triangles.emplace_back(e.first, e.second, i);
            }
        }

        // Remove triangles with super vertices
        vector<Triangle> final;
        int superStart = n - 3;
        for (auto& t : triangles) {
            if (t.a >= superStart || t.b >= superStart || t.c >= superStart) continue;
            final.push_back(t);
        }
        triangles = move(final);
    }

    const vector<Triangle>& getTriangles() const { return triangles; }
};

// === TEST ===
int main() {
    vector<Point> pts = {
        {0, 0}, {10, 0}, {5, 10}, {3, 4}, {8, 6}, {2, 8}
    };

    DelaunayTriangulation dt(pts);
    auto tris = dt.getTriangles();

    cout << "Delaunay Triangulation (" << tris.size() << " triangles):\n";
    for (auto& t : tris) {
        cout << t.a << " " << t.b << " " << t.c << "\n";
    }

    return 0;
}
```




## Delaunay Triangulation (C#)
```cpp
using System;
using System.Collections.Generic;
using System.Linq;

public class Point
{
    public double X, Y;
    public int Id;
    public Point(double x, double y, int id = -1) { X = x; Y = y; Id = id; }
}

public class Triangle
{
    public int A, B, C;
    public Triangle(int a, int b, int c) { A = a; B = b; C = c; }
}

public class DelaunayTriangulation
{
    private List<Point> points;
    private List<Triangle> triangles = new List<Triangle>();

    public DelaunayTriangulation(List<Point> pts)
    {
        points = new List<Point>(pts);
        Triangulate();
    }

    private bool InCircumcircle(Point p, Point a, Point b, Point c)
    {
        double ax = a.X - p.X, ay = a.Y - p.Y;
        double bx = b.X - p.X, by = b.Y - p.Y;
        double cx = c.X - p.X, cy = c.Y - p.Y;

        double det = ax * (by * (cx * cx + cy * cy) - cy * (bx * bx + by * by)) -
                     ay * (bx * (cx * cx + cy * cy) - cx * (bx * bx + by * by)) +
                     bx * (ay * (cx * cx + cy * cy) - cy * (ax * ax + ay * ay)) -
                     by * (ax * (cx * cx + cy * cy) - cx * (ax * ax + ay * ay));

        return det > 1e-8;
    }

    private void Triangulate()
    {
        if (points.Count < 3) return;

        // Super triangle
        double minX = points.Min(p => p.X), maxX = points.Max(p => p.X);
        double minY = points.Min(p => p.Y), maxY = points.Max(p => p.Y);
        double dx = maxX - minX + 1, dy = maxY - minY + 1;
        double midX = (minX + maxX) / 2, midY = (minY + maxY) / 2;

        var p1 = new Point(midX - 20 * dx, midY - dy);
        var p2 = new Point(midX + 20 * dx, midY - dy);
        var p3 = new Point(midX, midY + 20 * dy);

        points.Add(p1); points.Add(p2); points.Add(p3);
        triangles.Add(new Triangle(points.Count - 3, points.Count - 2, points.Count - 1));

        for (int i = 0; i < points.Count - 3; i++)
        {
            var bad = new List<int>();
            var edgeMap = new Dictionary<(int, int), int>();

            for (int j = 0; j < triangles.Count; j++)
            {
                var t = triangles[j];
                if (InCircumcircle(points[i], points[t.A], points[t.B], points[t.C]))
                {
                    bad.Add(j);
                    var edges = new[] { (t.A, t.B), (t.B, t.C), (t.C, t.A) };
                    foreach (var e in edges)
                    {
                        var key = e.Item1 < e.Item2 ? e : (e.Item2, e.Item1);
                        edgeMap[key] = edgeMap.GetValueOrDefault(key, 0) + 1;
                    }
                }
            }

            var polygon = new List<(int, int)>();
            foreach (var kv in edgeMap)
                if (kv.Value == 1) polygon.Add(kv.Key);

            // Remove bad triangles
            triangles = triangles.Where((t, idx) => !bad.Contains(idx)).ToList();

            // Add new triangles
            foreach (var (a, b) in polygon)
                triangles.Add(new Triangle(a, b, i));
        }

        // Remove super triangle
        int superStart = points.Count - 3;
        triangles = triangles.Where(t => t.A < superStart && t.B < superStart && t.C < superStart).ToList();
        points.RemoveRange(superStart, 3);
    }

    public List<Triangle> Triangles => triangles;
}

// === TEST ===
class Program
{
    static void Main()
    {
        var pts = new List<Point> {
            new Point(0, 0), new Point(10, 0), new Point(5, 10),
            new Point(3, 4), new Point(8, 6), new Point(2, 8)
        };

        var dt = new DelaunayTriangulation(pts);
        var tris = dt.Triangles;

        Console.WriteLine($"Delaunay Triangulation: {tris.Count} triangles");
        foreach (var t in tris)
            Console.WriteLine($"{t.A} {t.B} {t.C}");
    }
}
```





## Complexity
- **Time:**  
  - Incremental: average O(n log n), worst-case O(n²).  
  - Divide and conquer / sweep line: O(n log n).  
- **Space:** O(n), storing triangles and adjacency.

---

## Key Concepts Recap
- **Circumcircle property:** no point inside any triangle’s circumcircle.  
- **Angle optimality:** avoids skinny triangles.  
- **Duality:** Delaunay ↔ Voronoi.  
- **Robustness:** requires careful handling of floating-point precision.

  
✅ **Conclusion**  
The **Delaunay triangulation** is the geometric dual of the **Voronoi diagram**.  
It ensures well-shaped triangles, supports efficient spatial queries, and underpins many algorithms in graphics, mesh generation, and scientific computing.  
Together with Voronoi diagrams, it forms the backbone of modern computational geometry.

