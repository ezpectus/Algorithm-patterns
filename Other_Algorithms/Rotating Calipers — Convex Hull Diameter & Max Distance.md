# Rotating Calipers — Convex Hull Diameter & Max Distance  
*O(n) after hull — The Legendary Geometric Weapon*

---

## Origin & Motivation

**Rotating Calipers** was introduced by **Michael Shamos** in 1978 as part of his PhD thesis on computational geometry.
It solves a family of **O(n)** problems on **convex polygons**, including:

- **Diameter** (farthest pair of points)
- **Width** (minimum distance between parallel lines)
- **Maximum distance**
- **Smallest enclosing rectangle**, etc.

**Key insight**:  
On a **convex hull**, the **farthest points** are always **antipodal** — they lie on opposite sides of the polygon.
Instead of checking all pairs (n²) pairs → we **rotate two parallel "calipers"** around the hull.

---

## Core Idea — Antipodal Pairs

1. Compute **convex hull** first (O(n log n))
2. Start with two points (e.g., leftmost and rightmost)
3. Maintain **two pointers i, j** on hull
4. At each step:
   - Current distance `hull[i] ↔ hull[j]` is a candidate
   - Rotate the caliper that gives **larger angle** with next edge
5. The **maximum** distance seen = **diameter**

**Guarantee**: every antipodal pair is visited exactly once → **O(n)**

---

## Full Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    long long x, y;
    Point(long long x = 0, long long y = 0) : x(x), y(y) {}
    
    Point operator-(const Point& p) const { return {x - p.x, y - p.y}; }
    long long cross(const Point& p) const { return x * p.y - y * p.x; }
    long long dist2() const { return x * x + y * y; }
    
    bool operator<(const Point& p) const {
        return x < p.x || (x == p.x && y < p.y);
    }
};

// Convex Hull — Graham Scan O(n log n)
vector<Point> convexHull(vector<Point> pts) {
    int n = pts.size();
    if (n <= 1) return pts;
    
    sort(pts.begin(), pts.end());
    
    vector<Point> hull;
    hull.reserve(n);
    
    // Build lower hull
    for (int i = 0; i < n; ++i) {
        while (hull.size() >= 2 && 
               (hull.back() - hull[hull.size()-2]).cross(pts[i] - hull.back()) <= 0)
            hull.pop_back();
        hull.push_back(pts[i]);
    }
    
    size_t lowerSize = hull.size();
    
    // Build upper hull
    for (int i = n-2; i >= 0; --i) {
        while (hull.size() > lowerSize && 
               (hull.back() - hull[hull.size()-2]).cross(pts[i] - hull.back()) <= 0)
            hull.pop_back();
        hull.push_back(pts[i]);
    }
    
    hull.pop_back(); // last point is duplicated
    return hull;
}

// Rotating Calipers — O(n) diameter of convex hull
long long rotatingCalipers(const vector<Point>& hull) {
    int n = hull.size();
    if (n <= 1) return 0;
    if (n == 2) return (hull[0] - hull[1]).dist2();
    
    // Find initial antipodal pair: leftmost and rightmost points
    int i = 0, j = 1;
    for (int k = 0; k < n; ++k) {
        if (hull[k].x <= hull[i].x) i = k;
        if (hull[k].x >= hull[j].x) j = k;
    }
    
    int start_i = i, start_j = j;
    long long maxDist = 0;
    
    do {
        maxDist = max(maxDist, (hull[i] - hull[j]).dist2());
        
        // Next candidates
        int ni = (i + 1) % n;
        int nj = (j + 1) % n;
        
        // Move the pointer that gives the larger (or equal) turn
        long long turn = (hull[ni] - hull[i]).cross(hull[nj] - hull[j]);
        if (turn >= 0)
            j = nj;
        else
            i = ni;
            
    } while (i != start_i || j != start_j);
    
    return maxDist;
}

// === TEST ===
int main() {
    vector<Point> points = {
        {0, 0}, {10, 0}, {10, 10}, {0, 10},
        {5, 5}, {1, 8}, {9, 2}, {3, 9}
    };
    
    cout << "Input points: " << points.size() << "\n";
    
    auto hull = convexHull(points);
    cout << "Convex hull size: " << hull.size() << "\n";
    
    long long diameterSq = rotatingCalipers(hull);
    double diameter = sqrt(diameterSq);
    
    cout << "Diameter squared: " << diameterSq << "\n";
    cout << fixed << setprecision(6);
    cout << "Actual diameter: " << diameter << "\n";
    
    return 0;
}
```


## Full Implementation (C#)
```cpp
using System;
using System.Collections.Generic;
using System.Linq;

public class Point
{
    public long X, Y;
    public Point(long x, long y) { X = x; Y = y; }
}

public class RotatingCalipers
{
    private static long Cross(Point o, Point a, Point b) =>
        (a.X - o.X) * (b.Y - o.Y) - (a.Y - o.Y) * (b.X - o.X);

    private static long Dist2(Point a, Point b) =>
        (a.X - b.X) * (a.X - b.X) + (a.Y - b.Y) * (a.Y - b.Y);

    // Graham Scan Convex Hull
    public static List<Point> ConvexHull(List<Point> points)
    {
        var sorted = points.OrderBy(p => p.X).ThenBy(p => p.Y).ToList();
        var hull = new List<Point>();

        foreach (var p in sorted)
        {
            while (hull.Count >= 2 && Cross(hull[^2], hull[^1], p) <= 0)
                hull.RemoveAt(hull.Count - 1);
            hull.Add(p);
        }

        int lower = hull.Count;
        for (int i = points.Count - 2; i >= 0; i--)
        {
            var p = points[i];
            while (hull.Count > lower && Cross(hull[^2], hull[^1], p) <= 0)
                hull.RemoveAt(hull.Count - 1);
            hull.Add(p);
        }
        hull.RemoveAt(hull.Count - 1);
        return hull;
    }

    // Rotating Calipers — diameter in O(n)
    public static long MaxDistanceSquared(List<Point> hull)
    {
        int n = hull.Count;
        if (n <= 1) return 0;
        if (n == 2) return Dist2(hull[0], hull[1]);

        int i = 0, j = 1;
        for (int k = 0; k < n; k++)
        {
            if (hull[k].X <= hull[i].X) i = k;
            if (hull[k].X >= hull[j].X) j = k;
        }

        int si = i, sj = j;
        long maxDist = 0;

        do
        {
            maxDist = Math.Max(maxDist, Dist2(hull[i], hull[j]));

            int ni = (i + 1) % n;
            int nj = (j + 1) % n;

            if (Cross(hull[i], hull[ni], hull[nj]) >= 0)
                j = nj;
            else
                i = ni;
        } while (i != si || j != sj);

        return maxDist;
    }
}

// === TEST ===
class Program
{
    static void Main()
    {
        var points = new List<Point> {
            new(0, 0), new(10, 0), new(10, 10), new(0, 10),
            new(5, 5), new(1, 8), new(9, 2), new(3, 9)
        };

        var hull = RotatingCalipers.ConvexHull(points);
        long diameterSq = RotatingCalipers.MaxDistanceSquared(hull);

        Console.WriteLine($"Input points: {points.Count}");
        Console.WriteLine($"Hull size: {hull.Count}");
        Console.WriteLine($"Diameter squared: {diameterSq}");
        Console.WriteLine($"Actual diameter: {Math.Sqrt(diameterSq):F6}");
    }
}
```


## Complexity Analysis

| **Metric**         | **Value**           | **Notes**                                      |
|---------------------|---------------------|------------------------------------------------|
| **Convex Hull**     | **O(n log n)**      | Graham Scan / Monotone Chain / Jarvis (worst)  |
| **Rotating Calipers** | **O(n)**          | Exactly one full rotation around the hull      |
| **Total**           | **O(n log n)**      | Standard complexity for geometric diameter    |
| **Space**           | **O(n)**            | Only hull storage required                    |

**With n = 10⁵ → runs in milliseconds**

---

## Insight — Reusable Fichka

> **Rotating Calipers = O(n) magic after convex hull**

### What is Rotating Calipers?

**Rotating Calipers** is a powerful technique that simulates two **parallel supporting lines** ("calipers") rotating around a **convex polygon**.

At each step:
- The lines are **tangent** to the hull
- The **distance** between them (or points of contact) gives a **candidate** for extremal value

**Crucial property**:  
All **antipodal pairs** (vertices where supporting lines touch) are visited exactly once → **O(n)** total.

### Pattern

1. **Compute convex hull first** (mandatory)
2. Initialize two pointers `i`, `j` at antipodal positions (e.g., min/max X)
3. While not full rotation:
   - Update current metric (distance, width, etc.)
   - Decide which pointer to move by comparing **directions** of next edges
   - Advance the pointer that gives **larger rotation angle**

### Applies to

- **Diameter** — farthest pair of points
- **Minimum width** — smallest distance between parallel supporting lines
- **Maximum area triangle** on hull
- **Smallest enclosing rectangle / circle**
- **Largest inscribed triangle**

### Fichka:

> **After you have a convex hull → Rotating Calipers turns any O(n²) brute-force geometric query into O(n)**
**One of the most elegant and powerful techniques in computational geometry.**

---

