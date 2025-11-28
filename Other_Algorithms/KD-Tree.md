# KD-Tree — Nearest Neighbor Search

---

## 📜 Origin & Motivation
- **KD-Tree (k-dimensional tree)** was introduced by **Jon Bentley in 1975**.  
- It is a **binary space-partitioning data structure** used for organizing points in k-dimensional space.  
- Main purpose: efficient **nearest neighbor search**, **range queries**, and **multidimensional indexing**.  

---

## 💡 Core Idea
1. **Construction**
   - Recursively split the dataset along coordinate axes.  
   - At each level, choose an axis (e.g., x, y, z) and partition points by median.  
   - Left subtree: points with smaller coordinate values.  
   - Right subtree: points with larger coordinate values.  

2. **Nearest Neighbor Search**
   - Traverse the tree to find the region containing the query point.  
   - Backtrack and check other regions if they could contain closer points.  
   - Use bounding boxes to prune search space.  

3. **Efficiency**
   - Reduces search complexity compared to brute force.  
   - Especially effective in low to moderate dimensions (k ≤ 20).  

---

## ⏱️ Complexity Analysis
- **Construction:** O(n log n)  
- **Nearest neighbor query:** Average O(log n), worst-case O(n)  
- **Space complexity:** O(n)  

---

## 🔄 Comparison with Other Algorithms

| Algorithm        | Use Case                  | Complexity (Query) | Notes                                |
|------------------|---------------------------|--------------------|--------------------------------------|
| **KD-Tree**      | Nearest neighbor, range   | O(log n) avg       | Efficient in low dimensions          |
| Brute Force      | Nearest neighbor          | O(n)               | Simple but slow for large datasets   |
| Ball Tree        | High-dimensional search   | O(log n) avg       | Better for higher dimensions         |
| R-Tree           | Spatial indexing          | O(log n)           | Used in GIS, rectangles not points   |

---

##  KD-Tree Implementation (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

struct Point {
    vector<double> coords;
    Point(double x, double y) { coords = {x, y}; }
};

struct KDNode {
    Point point;
    KDNode *left, *right;
    KDNode(Point p) : point(p), left(NULL), right(NULL) {}
};

KDNode* build(vector<Point>& points, int depth=0) {
    if (points.empty()) return NULL;
    int axis = depth % points[0].coords.size();
    sort(points.begin(), points.end(), [&](Point a, Point b){
        return a.coords[axis] < b.coords[axis];
    });
    int mid = points.size()/2;
    KDNode* node = new KDNode(points[mid]);
    vector<Point> left(points.begin(), points.begin()+mid);
    vector<Point> right(points.begin()+mid+1, points.end());
    node->left = build(left, depth+1);
    node->right = build(right, depth+1);
    return node;
}

double dist(Point a, Point b) {
    double d=0;
    for(int i=0;i<a.coords.size();i++)
        d += (a.coords[i]-b.coords[i])*(a.coords[i]-b.coords[i]);
    return sqrt(d);
}

void nearest(KDNode* root, Point target, int depth, Point& best, double& bestDist) {
    if (!root) return;
    double d = dist(root->point, target);
    if (d < bestDist) { bestDist = d; best = root->point; }
    int axis = depth % target.coords.size();
    KDNode* next = target.coords[axis] < root->point.coords[axis] ? root->left : root->right;
    KDNode* other = target.coords[axis] < root->point.coords[axis] ? root->right : root->left;
    nearest(next, target, depth+1, best, bestDist);
    if (fabs(target.coords[axis]-root->point.coords[axis]) < bestDist)
        nearest(other, target, depth+1, best, bestDist);
}

int main() {
    vector<Point> points = {Point(2,3), Point(5,4), Point(9,6), Point(4,7), Point(8,1), Point(7,2)};
    KDNode* root = build(points);
    Point target(9,2);
    Point best = points[0];
    double bestDist = dist(best, target);
    nearest(root, target, 0, best, bestDist);
    cout << "Nearest point: (" << best.coords[0] << "," << best.coords[1] << ") Dist=" << bestDist << endl;
}
```


## KD-Tree Implementation (C#)
```cpp
using System;
using System.Collections.Generic;

class Point {
    public double[] Coords;
    public Point(double x, double y) { Coords = new double[]{x,y}; }
}

class KDNode {
    public Point Point;
    public KDNode Left, Right;
    public KDNode(Point p) { Point = p; }
}

class KDTree {
    public KDNode Build(List<Point> points, int depth=0) {
        if (points.Count == 0) return null;
        int axis = depth % points[0].Coords.Length;
        points.Sort((a,b)=>a.Coords[axis].CompareTo(b.Coords[axis]));
        int mid = points.Count/2;
        KDNode node = new KDNode(points[mid]);
        node.Left = Build(points.GetRange(0, mid), depth+1);
        node.Right = Build(points.GetRange(mid+1, points.Count-mid-1), depth+1);
        return node;
    }

    public double Dist(Point a, Point b) {
        double d=0;
        for(int i=0;i<a.Coords.Length;i++)
            d += (a.Coords[i]-b.Coords[i])*(a.Coords[i]-b.Coords[i]);
        return Math.Sqrt(d);
    }

    public void Nearest(KDNode root, Point target, int depth, ref Point best, ref double bestDist) {
        if (root == null) return;
        double d = Dist(root.Point, target);
        if (d < bestDist) { bestDist = d; best = root.Point; }
        int axis = depth % target.Coords.Length;
        KDNode next = target.Coords[axis] < root.Point.Coords[axis] ? root.Left : root.Right;
        KDNode other = target.Coords[axis] < root.Point.Coords[axis] ? root.Right : root.Left;
        Nearest(next, target, depth+1, ref best, ref bestDist);
        if (Math.Abs(target.Coords[axis]-root.Point.Coords[axis]) < bestDist)
            Nearest(other, target, depth+1, ref best, ref bestDist);
    }
}

class Program {
    static void Main() {
        List<Point> points = new List<Point>{
            new Point(2,3), new Point(5,4), new Point(9,6),
            new Point(4,7), new Point(8,1), new Point(7,2)
        };
        KDTree tree = new KDTree();
        KDNode root = tree.Build(points);
        Point target = new Point(9,2);
        Point best = points[0];
        double bestDist = tree.Dist(best, target);
        tree.Nearest(root, target, 0, ref best, ref bestDist);
        Console.WriteLine($"Nearest point: ({best.Coords[0]},{best.Coords[1]}) Dist={bestDist}");
    }
}
```


##  Conclusion

The **KD-Tree** is a powerful and elegant data structure for **nearest neighbor search** in multidimensional spaces.  
Its design leverages recursive space partitioning, which allows queries to be answered much faster than brute force methods, especially in **low-dimensional datasets**.

### Strengths
- **Efficiency:** Average query time O(log n), compared to O(n) for brute force.  
- **Practicality:** Works well for dimensions up to ~20, beyond which performance degrades.  
- **Applications:**  
  - **Computer graphics:** collision detection, ray tracing.  
  - **Robotics:** path planning, obstacle avoidance.  
  - **Machine learning:** k-Nearest Neighbors (kNN) classification/regression.  
  - **Spatial databases:** indexing and querying geographic data.

### Limitations
- **High-dimensional curse:** As dimensionality increases, KD-Tree performance approaches brute force.  
- **Dynamic updates:** Insertions/deletions can unbalance the tree, requiring rebuilding for efficiency.  
- **Memory overhead:** Requires storing splitting planes and bounding boxes.

### Alternatives for High Dimensions
- **Ball Trees:** Better suited for higher dimensions, partitioning data into hyperspheres.  
- **Approximate Nearest Neighbor (ANN):** Sacrifices exactness for speed, widely used in large-scale ML (e.g., FAISS, Annoy).  
- **R-Trees:** Efficient for spatial indexing of rectangles and polygons, common in GIS.

---

### Comparative Summary

| Method                | Best Use Case                  | Query Complexity | Notes                                    |
|-----------------------|--------------------------------|-----------------|------------------------------------------|
| **KD-Tree**           | Low-dimensional nearest neighbor | O(log n) avg    | Fast, simple, widely used                |
| **Brute Force**       | Very small datasets             | O(n)            | Simple but slow for large n              |
| **Ball Tree**         | Higher dimensions               | O(log n) avg    | Uses hyperspheres, better scaling        |
| **ANN Methods**       | Very large datasets             | ~O(log n)       | Approximate results, huge speed gains    |
| **R-Tree**            | Spatial/geographic indexing     | O(log n)        | Works with rectangles/polygons           |

---

### Final Thought
The KD-Tree remains a **cornerstone of computational geometry**:  
- It balances **simplicity, efficiency, and versatility**.  
- Ideal for problems in **2D–20D spaces**, where exact nearest neighbor queries are required.  
- For higher dimensions or massive datasets, hybrid or approximate methods are preferred.  

In practice, KD-Tree is often the **first choice** for nearest neighbor search, forming the backbone of many algorithms in **graphics, robotics, and machine learning**.


---
