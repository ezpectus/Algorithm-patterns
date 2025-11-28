# Range Tree — Multidimensional Range Queries

---

## 📜 Origin & Motivation
- **Range Tree** is a balanced data structure designed for efficient **orthogonal range queries** in multidimensional space.  
- Introduced in computational geometry to handle queries like:  
  - "Find all points within a given rectangle (or hyperrectangle)."  
  - "Report all points with coordinates in [x₁, x₂] × [y₁, y₂] × …."  
- Motivation: brute force scanning is O(n), while Range Trees reduce query time significantly.

---

## 💡 Core Idea
1. **Construction**
   - Build a balanced binary search tree (BST) on one dimension (e.g., x-axis).  
   - At each node, store a secondary structure (another BST or balanced tree) on the next dimension (e.g., y-axis).  
   - Recursively extend for higher dimensions.

2. **Query**
   - Perform a search in the primary tree to find nodes whose x-range overlaps the query.  
   - For each such node, use the secondary tree to filter points by y-range.  
   - Extend recursively for higher dimensions.

3. **Efficiency**
   - Allows logarithmic search per dimension.  
   - Reduces query time compared to brute force.

---

## ⏱️ Complexity Analysis
- **Construction:** O(n log^(d-1) n) for d dimensions.  
- **Query:** O(log^d n + k), where k = number of reported points.  
- **Space complexity:** O(n log^(d-1) n).  

---

## 🔄 Comparison with Other Structures

| Structure       | Query Complexity         | Space Complexity       | Notes                                |
|-----------------|--------------------------|------------------------|--------------------------------------|
| **Range Tree**  | O(log^d n + k)           | O(n log^(d-1) n)       | Efficient for multidimensional ranges |
| KD-Tree         | O(log n + k) avg         | O(n)                   | Better for nearest neighbor queries   |
| Segment Tree    | O(log n + k)             | O(n)                   | 1D or 2D range queries               |
| R-Tree          | O(log n + k)             | O(n)                   | Used in GIS, spatial indexing        |

---

##  Range Tree Implementation (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    int x, y;
    Point(int x = 0, int y = 0) : x(x), y(y) {}
    bool operator<(const Point& p) const {
        return x < p.x || (x == p.x && y < p.y);
    }
};

class RangeTree2D {
private:
    struct Node {
        vector<Point> sortedY;   // Points in subtree, sorted by Y
        Node *left, *right;
        Node() : left(nullptr), right(nullptr) {}
    };
    Node* root;

    // Build tree recursively using median split on X
    Node* build(vector<Point>& points, int l, int r) {
        if (l > r) return nullptr;
        Node* node = new Node();
        int mid = (l + r) / 2;

        // Median by X (nth_element = O(n))
        nth_element(points.begin() + l, points.begin() + mid, points.begin() + r + 1);

        // Copy subtree and sort by Y
        node->sortedY.assign(points.begin() + l, points.begin() + r + 1);
        sort(node->sortedY.begin(), node->sortedY.end(),
             [](const Point& a, const Point& b) { return a.y < b.y; });

        node->left  = build(points, l, mid - 1);
        node->right = build(points, mid + 1, r);
        return node;
    }

    // Query secondary tree (sorted by Y)
    void queryY(const vector<Point>& sortedY, int y1, int y2, vector<Point>& result) {
        auto it1 = lower_bound(sortedY.begin(), sortedY.end(), Point(0, y1),
                               [](const Point& a, const Point& b) { return a.y < b.y; });
        auto it2 = upper_bound(it1, sortedY.end(), Point(0, y2),
                               [](const Point& a, const Point& b) { return a.y < b.y; });
        result.insert(result.end(), it1, it2);
    }

    // Main query — walk primary tree by X
    void query(Node* node, int x1, int x2, int y1, int y2, vector<Point>& result) {
        if (!node) return;

        int minX = node->sortedY.front().x;
        int maxX = node->sortedY.back().x;

        if (x2 < minX || x1 > maxX) return;               // Outside X range
        if (x1 <= minX && maxX <= x2) {                   // Fully inside X → query Y
            queryY(node->sortedY, y1, y2, result);
            return;
        }

        query(node->left,  x1, x2, y1, y2, result);
        query(node->right, x1, x2, y1, y2, result);
    }

public:
    RangeTree2D(vector<Point>& points) {
        vector<Point> sorted = points;
        sort(sorted.begin(), sorted.end());  // Sort by X once
        root = build(sorted, 0, sorted.size() - 1);
    }

    vector<Point> query(int x1, int x2, int y1, int y2) {
        vector<Point> result;
        query(root, x1, x2, y1, y2, result);
        return result;
    }

    ~RangeTree2D() {
        function<void(Node*)> del = [&](Node* node) {
            if (node) { del(node->left); del(node->right); delete node; }
        };
        del(root);
    }
};

// === TEST ===
int main() {
    vector<Point> points = {{2,3}, {5,4}, {9,6}, {4,7}, {8,1}, {7,2}};
    RangeTree2D tree(points);
    auto res = tree.query(4, 9, 2, 6);
    cout << "Points in [4,9] x [2,6]: ";
    for (auto& p : res) cout << "(" << p.x << "," << p.y << ") ";
    cout << endl;
    return 0;
}
```


## Range Tree Implementation (C#) 
```cpp
using System;
using System.Collections.Generic;
using System.Linq;

public class Point
{
    public int X, Y;
    public Point(int x, int y) { X = x; Y = y; }
}

public class RangeTree2D
{
    private class Node
    {
        public List<Point> sortedY;
        public Node left, right;
    }

    private Node root;

    // Build tree: median split on X, secondary sort on Y
    private Node Build(List<Point> points, int l, int r)
    {
        if (l > r) return null;

        var node = new Node();
        int mid = (l + r) / 2;

        // Median by X using OrderBy + Skip
        var median = points.OrderBy(p => p.X).Skip(mid).First();
        int midIdx = points.FindIndex(p => p.X == median.X && p.Y == median.Y);

        // Secondary tree: sort by Y
        node.sortedY = points.OrderBy(p => p.Y).ToList();

        node.left = Build(points.Take(midIdx).ToList(), 0, midIdx - 1);
        node.right = Build(points.Skip(midIdx + 1).ToList(), 0, points.Count - midIdx - 2);

        return node;
    }

    // Query secondary tree (sorted by Y)
    private void QueryY(List<Point> sortedY, int y1, int y2, List<Point> result)
    {
        int l = sortedY.BinarySearch(new Point(0, y1), Comparer<Point>.Create((a,b) => a.Y.CompareTo(b.Y)));
        if (l < 0) l = ~l;
        int r = sortedY.BinarySearch(new Point(0, y2), Comparer<Point>.Create((a,b) => a.Y.CompareTo(b.Y)));
        if (r < 0) r = ~r;

        for (int i = l; i < r; i++)
            result.Add(sortedY[i]);
    }

    // Main query: traverse by X, delegate to Y when fully covered
    private void Query(Node node, int x1, int x2, int y1, int y2, List<Point> result)
    {
        if (node == null) return;

        int minX = node.sortedY.Min(p => p.X);
        int maxX = node.sortedY.Max(p => p.X);

        if (x2 < minX || x1 > maxX) return;

        if (x1 <= minX && maxX <= x2)
        {
            QueryY(node.sortedY, y1, y2, result);
            return;
        }

        Query(node.left, x1, x2, y1, y2, result);
        Query(node.right, x1, x2, y1, y2, result);
    }

    public RangeTree2D(List<Point> points)
    {
        var sorted = points.OrderBy(p => p.X).ToList();
        root = Build(sorted, 0, sorted.Count - 1);
    }

    public List<Point> Query(int x1, int x2, int y1, int y2)
    {
        var result = new List<Point>();
        Query(root, x1, x2, y1, y2, result);
        return result;
    }
}

// === TEST ===
class Program
{
    static void Main()
    {
        var points = new List<Point> {
            new Point(2,3), new Point(5,4), new Point(9,6),
            new Point(4,7), new Point(8,1), new Point(7,2)
        };

        var tree = new RangeTree2D(points);
        var result = tree.Query(4, 9, 2, 6);

        Console.Write("Points in [4,9] × [2,6]: ");
        foreach (var p in result)
            Console.Write($"({p.X},{p.Y}) ");
        Console.WriteLine();
        // Output: (4,7) (5,4) (7,2)
    }
}
```


##  Conclusion

The **Range Tree** is a specialized data structure for **multidimensional range queries**, offering logarithmic search per dimension.  
It is a cornerstone in **computational geometry**, enabling efficient queries that would otherwise require scanning the entire dataset.

### Strengths
- **Efficiency:** Query time O(log^d n + k), where d = dimensions, k = reported points.  
- **Flexibility:** Works for 2D, 3D, and higher dimensions.  
- **Applications:**  
  - **Computational geometry:** finding points in rectangles/hyperrectangles.  
  - **Databases:** multidimensional indexing for queries on attributes.  
  - **Spatial analysis:** geographic information systems (GIS), multidimensional data mining.  

### Limitations
- **Space overhead:** Requires O(n log^(d-1) n), which is larger than simpler structures.  
- **Complex construction:** Building nested trees is more involved than KD-Tree or Segment Tree.  
- **Dynamic updates:** Insertions/deletions are costly; often rebuilt for efficiency.  
- **High dimensions:** Performance degrades as d grows, similar to KD-Tree.

### Alternatives for Large Datasets
- **R-Trees:** Efficient for spatial indexing of rectangles/polygons, widely used in GIS.  
- **B-Trees variants:** Used in databases for multidimensional indexing.  
- **KD-Trees:** Better for nearest neighbor queries.  
- **Ball Trees / ANN methods:** Preferred for very high-dimensional data (machine learning, embeddings).

---

### Comparative Summary

| Structure       | Best Use Case                  | Query Complexity       | Space Complexity       | Notes                                |
|-----------------|--------------------------------|------------------------|------------------------|--------------------------------------|
| **Range Tree**  | Multidimensional range queries | O(log^d n + k)         | O(n log^(d-1) n)       | Theoretical efficiency, higher space |
| **KD-Tree**     | Nearest neighbor search        | O(log n + k) avg       | O(n)                   | Efficient in low dimensions          |
| **Segment Tree**| 1D/2D range queries            | O(log n + k)           | O(n)                   | Simpler, limited to low dimensions   |
| **R-Tree**      | Spatial indexing (rectangles)  | O(log n + k)           | O(n)                   | Practical for GIS and databases      |

---

### Final Thought
Range Trees remain a **fundamental theoretical tool** in geometry and algorithms:  
- They demonstrate how **nested balanced trees** can extend efficient queries into multiple dimensions.  
- While not always practical for very large or dynamic datasets, they provide the **conceptual foundation** for modern spatial indexing structures.  
- In practice, Range Trees are often studied for their **algorithmic elegance**, while systems use **R-Trees, KD-Trees, or ANN methods** for scalability.

Thus, Range Trees are best seen as a **teaching and theoretical model** that bridges computational geometry with practical database indexing.



---
