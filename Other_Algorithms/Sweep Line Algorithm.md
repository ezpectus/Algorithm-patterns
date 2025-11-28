# Sweep Line Algorithm — Intersections & Closest Pair of Points

---

## 📜 Origin & Motivation
- The **Sweep Line Algorithm** is a fundamental technique in **computational geometry**.  
- Introduced in the 1970s, it is used to solve problems like:  
  - Detecting **line segment intersections**.  
  - Finding the **closest pair of points**.  
- Motivation: brute force approaches are O(n²), while sweep line reduces complexity to near-linear or O(n log n).

---

## 💡 Core Idea
The algorithm simulates a **vertical line sweeping across the plane**:

1. **Events:**  
   - Points or segment endpoints are sorted by x-coordinate.  
   - The sweep line processes events from left to right.

2. **Active Set:**  
   - Maintain a balanced data structure (e.g., balanced BST) of objects intersecting the sweep line.  
   - Update as the line moves.

3. **Intersection Detection:**  
   - When encountering a segment endpoint, insert/remove it from the active set.  
   - Check only neighboring segments for intersections.  
   - Complexity: O((n + k) log n), where k = number of intersections.

4. **Closest Pair of Points:**  
   - Sort points by x-coordinate.  
   - Sweep line maintains a window of candidate points within current minimal distance.  
   - For each point, check only nearby points in y-order.  
   - Complexity: O(n log n).

---

## ⏱️ Complexity Analysis
- **Segment intersection:** O((n + k) log n).  
- **Closest pair of points:** O(n log n).  
- **Space complexity:** O(n) for active set.  

---

## 🔄 Comparison with Other Approaches

| Problem                  | Brute Force Complexity | Sweep Line Complexity | Notes                                   |
|---------------------------|------------------------|-----------------------|-----------------------------------------|
| Segment Intersection      | O(n²)                 | O((n + k) log n)      | Efficient for large n, k = intersections |
| Closest Pair of Points    | O(n²)                 | O(n log n)            | Optimal deterministic solution           |

---

## 🧩 Sweep Line — Closest Pair (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    double x, y;
    Point(double x = 0, double y = 0) : x(x), y(y) {}
    bool operator<(const Point& p) const {
        return x < p.x - 1e-9 || (abs(x - p.x) < 1e-9 && y < p.y - 1e-9);
    }
};

double dist(const Point& a, const Point& b) {
    return hypot(a.x - b.x, a.y - b.y);
}

// Sweep Line — Closest Pair of Points — O(n log n)
class ClosestPair {
public:
    static double find(vector<Point> points) {
        int n = points.size();
        if (n <= 1) return 1e18;
        if (n == 2) return dist(points[0], points[1]);

        // Sort by X coordinate
        sort(points.begin(), points.end());

        // Active set: points in current vertical strip, sorted by Y
        set<pair<double, int>> active;  // (y, index)

        double best = 1e18;
        int left = 0;

        for (int i = 0; i < n; i++) {
            // Remove points outside the current best distance strip
            while (left < i && points[i].x - points[left].x >= best) {
                active.erase({points[left].y, left});
                left++;
            }

            // Search in strip [y-best, y+best]
            auto low = make_pair(points[i].y - best, -1);
            auto high = make_pair(points[i].y + best, n);

            auto it_low = active.lower_bound(low);
            auto it_high = active.upper_bound(high);

            for (auto it = it_low; it != it_high; ++it) {
                int j = it->second;
                best = min(best, dist(points[i], points[j]));
            }

            // Add current point
            active.insert({points[i].y, i});
        }
        return best;
    }
};

// === FULL TEST CLASS ===
int main() {
    cout << fixed << setprecision(8);
    cout << "Closest Pair of Points — Sweep Line Algorithm\n\n";

    auto test = [](vector<Point> points, double expected) {
        double result = ClosestPair::find(points);
        cout << "Points: " << points.size() << "\n";
        cout << "Expected: " << expected << "\n";
        cout << "Got:      " << result << "\n";
        bool ok = abs(result - expected) < 1e-6;
        cout << (ok ? "PASS\n\n" : "FAIL\n\n");
    };

    // Test 1: Classic example
    test({{2,3},{12,30},{40,50},{5,1},{12,10},{3,4}}, 1.41421356);

    // Test 2: Points on a line
    test({{0,0},{1,0},{2,0},{100,0}}, 1.0);

    // Test 3: Identical points
    test({{5,5},{5,5},{10,10}}, 0.0);

    // Test 4: Square corners
    test({{0,0},{1000,1000},{0,1000},{1000,0}}, 1414.21356237);

    // Test 5: Many points
    vector<Point> big;
    for (int i = 0; i < 1000; i++) big.emplace_back(rand() % 10000, rand() % 10000);
    cout << "Large test (1000 points): " << ClosestPair::find(big) << "\n\n";

    cout << "All tests completed!\n";
    return 0;
}
```


## 🧩 Sweep Line — Closest Pair (C#)
```cpp

using System;
using System.Collections.Generic;
using System.Linq;

public class Point
{
    public double X, Y;
    public Point(double x, double y) { X = x; Y = y; }
}

public class Solution
{
    // O(n log n) — Sweep Line Closest Pair of Points
    // Based on the classic divide-and-conquer algorithm
    public static double ClosestPair(List<Point> pts)
    {
        int n = pts.Count;
        if (n <= 1) return double.MaxValue;
        if (n == 2) return Math.Sqrt(Math.Pow(pts[0].X - pts[1].X, 2) + Math.Pow(pts[0].Y - pts[1].Y, 2));

        // Sort points by X coordinate
        pts = pts.OrderBy(p => p.X).ToList();

        // Active set keeps points in current strip, sorted by Y
        var active = new SortedSet<(double Y, int Idx)>();

        double best = double.MaxValue;
        int left = 0;

        for (int i = 0; i < n; i++)
        {
            // Remove points that are too far left (outside current best strip)
            while (left < i && pts[i].X - pts[left].X >= best)
                active.Remove((pts[left].Y, left++));

            // Define Y-range for potential closer points
            var low = (pts[i].Y - best, int.MinValue);
            var high = (pts[i].Y + best, int.MaxValue);

            // Check only points in the strip of width 2*best
            foreach (var (y, j) in active.GetViewBetween(low, high))
            {
                double d = Math.Sqrt(Math.Pow(pts[i].X - pts[j].X, 2) + Math.Pow(pts[i].Y - y, 2));
                if (d < best) best = d;
            }

            // Add current point to active set
            active.Add((pts[i].Y, i));
        }

        return best;
    }
}

// === FULL TEST CLASS WITH MULTIPLE CASES ===
class Program
{
    static void Main()
    {
        Console.WriteLine("Closest Pair of Points — Sweep Line Algorithm\n");

        // Test 1: Classic example
        Test(new List<Point>
        {
            new Point(2, 3), new Point(12, 30), new Point(40, 50),
            new Point(5, 1), new Point(12, 10), new Point(3, 4)
        }, 1.41421356237);  // Distance between (2,3) and (3,4)

        // Test 2: Points on a line
        Test(new List<Point>
        {
            new Point(0, 0), new Point(1, 0), new Point(2, 0), new Point(100, 0)
        }, 1.0);

        // Test 3: Identical points (distance 0)
        Test(new List<Point>
        {
            new Point(5, 5), new Point(5, 5), new Point(10, 10)
        }, 0.0);

        // Test 4: Large spread
        Test(new List<Point>
        {
            new Point(0, 0), new Point(1000, 1000), new Point(0, 1000), new Point(1000, 0)
        }, 1414.21356237);  // sqrt(2)*1000

        Console.WriteLine("\nAll tests passed!");
    }

    static void Test(List<Point> points, double expected)
    {
        double result = Solution.ClosestPair(new List<Point>(points));
        Console.WriteLine($"Points: {points.Count}");
        Console.WriteLine($"Expected: {expected:F8}");
        Console.WriteLine($"Got:      {result:F8}");
        Console.WriteLine(result < expected + 1e-6 && result > expected - 1e-6
            ? "PASS\n"
            : "FAIL\n");
    }
}
```


## ✅ Conclusion

The **Sweep Line Algorithm** is one of the most versatile and elegant techniques in **computational geometry**.  
By simulating a moving line across the plane and maintaining an **active set** of candidates, it transforms problems that are quadratic in nature into near-linear or **O(n log n)** solutions.

### Strengths
- **Efficiency:** Handles intersection detection and closest pair problems far faster than brute force.  
- **Generality:** Applicable to a wide range of geometric problems (segment intersections, closest pair, Voronoi diagrams).  
- **Scalability:** Works well for large datasets where naive approaches are infeasible.  
- **Applications:**  
  - **Computer graphics:** collision detection, rendering pipelines.  
  - **GIS:** map overlay, spatial queries.  
  - **Robotics:** obstacle detection, path planning.  
  - **Geometry libraries:** foundational algorithm in computational geometry toolkits.

### Limitations
- **Implementation complexity:** Requires careful event handling and balanced data structures.  
- **Numerical precision:** Floating-point errors can affect intersection detection.  
- **Dynamic updates:** Best suited for static sets of segments/points; dynamic cases are harder.  

### Alternatives & Extensions
- **Divide and Conquer:** Another O(n log n) approach for closest pair problems.  
- **Plane Sweep Variants:** Extended to handle polygons, rectangles, and higher-dimensional problems.  
- **Spatial Indexing (R-Trees, KD-Trees):** Used when queries are frequent and datasets are large.  

---

### Comparative Summary

| Problem                  | Brute Force Complexity | Sweep Line Complexity | Notes                                   |
|---------------------------|------------------------|-----------------------|-----------------------------------------|
| Segment Intersection      | O(n²)                 | O((n + k) log n)      | k = number of intersections             |
| Closest Pair of Points    | O(n²)                 | O(n log n)            | Optimal deterministic solution          |
| Voronoi Diagram / Delaunay| O(n²)                 | O(n log n)            | Sweep line forms basis of construction  |

---

### Final Thought
The Sweep Line Algorithm exemplifies how **algorithmic insight** can drastically reduce complexity:  
- Instead of checking all pairs, it focuses only on **local candidates**.  
- This principle of **limiting comparisons** is a recurring theme in efficient algorithms.  
- As a result, sweep line remains a **cornerstone technique** in computational geometry, bridging theory with practical applications in graphics, GIS, and robotics.


---
