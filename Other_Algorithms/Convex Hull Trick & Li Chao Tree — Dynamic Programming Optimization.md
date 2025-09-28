# 🧠 Convex Hull Trick & Li Chao Tree — Dynamic Programming Optimization

---

## 📜 Origin & Motivation

The **Convex Hull Trick (CHT)** and **Li Chao Tree** are advanced techniques for optimizing **dynamic programming (DP)** transitions involving **linear functions**.

- **Convex Hull Trick** was popularized in competitive programming circles around 2010, though its geometric roots trace back to computational geometry.
- **Li Chao Tree** was introduced by **Li Chao** in 2005 as a segment tree variant that supports efficient insertion and querying of lines.

Both are designed to **speed up DP recurrences** of the form:

```
dp[i] = min(dp[j] + m_j * x_i + b_j)
```

Instead of brute-force checking all previous states, they maintain a structure of lines and query the minimum efficiently.

---

## 💡 Core Idea

### 🔹 Convex Hull Trick

- Maintain a set of **lines** \( y = mx + b \) in a **monotonic slope order**
- Use **binary search** or **pointer walk** to find the best line for a given \( x \)
- Works best when:
  - Slopes \( m_j \) are **monotonic**
  - Queries \( x_i \) are **monotonic**

### 🔹 Li Chao Tree

- Segment tree where each node stores a line
- Supports **arbitrary insertion order** of lines
- Queries for minimum value at point \( x \) in **logarithmic time**
- Handles **non-monotonic** slopes and queries

Both structures allow **online insertion** and **fast querying**, turning naive \( O(n^2) \) DP into \( O(n \log n) \) or even \( O(n) \) in special cases.

---

## 🧩 Where It’s Used

- 📈 **DP with linear cost functions**  
- 🧮 **Convex optimization problems**  
- 🧠 **Slope trick transformations**  
- 🗺️ **Path cost minimization**  
- 🧊 **Geometric DP** — envelopes, hulls, and visibility  
- 🏁 **Competitive programming** — tasks like line envelopes, min-cost jumps, etc.

---

## 🔁 When to Use CHT vs Li Chao Tree

| Scenario                          | Use CHT | Use Li Chao Tree |
|-----------------------------------|--------|------------------|
| Monotonic slopes & queries        | ✅     | ❌               |
| Arbitrary slope insertion         | ❌     | ✅               |
| Fast online queries               | ✅     | ✅               |
| Segment tree integration          | ❌     | ✅               |
| Memory-sensitive implementation   | ✅     | ❌               |

---

## 🧱 Convex Hull Trick (C++)

Convex Hull Trick maintains a set of lines sorted by slope and uses binary search to find the minimum value at a given `x`.  
It works best when:
- Slopes are added in **monotonic order**
- Queries are made in **monotonic order**

### 🔧 Core Mechanics

- Each line is represented as `y = m * x + b`
- `intersect()` computes the x-coordinate where two lines cross
- `addLine()` maintains the convex hull by removing obsolete lines
- `query(x)` performs binary search to find the best line for given `x`

### 🧩 C++ Implementation

```cpp
struct Line {
    long long m, b;
    double intersect(const Line& other) const {
        return (double)(other.b - b) / (m - other.m);
    }
};

vector<Line> hull;

void addLine(long long m, long long b) {
    Line newLine = {m, b};
    while (hull.size() >= 2 &&
           newLine.intersect(hull[hull.size()-2]) <= hull.back().intersect(hull[hull.size()-2]))
        hull.pop_back(); // Remove line that is no longer optimal
    hull.push_back(newLine);
}

long long query(long long x) {
    int l = 0, r = hull.size() - 1;
    while (l < r) {
        int mid = (l + r) / 2;
        if (hull[mid].intersect(hull[mid+1]) < x)
            l = mid + 1;
        else
            r = mid;
    }
    return hull[l].m * x + hull[l].b;
}
```

## 🧱 Convex Hull Trick (C#)
- This version mirrors the C++ logic using List<Line> and binary search. 
- It’s ideal for DP transitions like dp[i] = min(dp[j] + m_j * x_i + b_j) when slopes and queries are monotonic.

## 🧩 C# Implementation
```cpp
class Line {
    public long M, B;
    public double Intersect(Line other) {
        return (double)(other.B - B) / (M - other.M);
    }

    public long Eval(long x) => M * x + B;
}

class ConvexHullTrick {
    private List<Line> hull = new();

    public void AddLine(long m, long b) {
        var newLine = new Line { M = m, B = b };
        while (hull.Count >= 2 &&
               newLine.Intersect(hull[^2]) <= hull[^1].Intersect(hull[^2]))
            hull.RemoveAt(hull.Count - 1);
        hull.Add(newLine);
    }

    public long Query(long x) {
        int l = 0, r = hull.Count - 1;
        while (l < r) {
            int mid = (l + r) / 2;
            if (hull[mid].Intersect(hull[mid + 1]) < x)
                l = mid + 1;
            else
                r = mid;
        }
        return hull[l].Eval(x);
    }
}
```

---

## 🧱 Li Chao Tree (C++)
- Li Chao Tree is a segment tree variant that supports arbitrary insertion of lines and queries for minimum value at any point.
  
 It’s ideal when:
 
- Slopes are not monotonic
- Queries are randomized
- You need logarithmic insertion/query without slope assumptions

## 🔧 Core Mechanics

- Each node stores a line
- Insert compares new line with current line at l, mid, and r
- Recursively inserts into left/right subtree depending on dominance
- Query traverses tree to find minimum value at x

##  🧩 C++ Implementation

```cpp
struct Line {
    long long m, b;
    long long eval(long long x) const {
        return m * x + b;
    }
};

struct Node {
    Line line;
    Node *left = nullptr, *right = nullptr;
};

void insert(Node*& node, int l, int r, Line newLine) {
    if (!node) {
        node = new Node{newLine};
        return;
    }
    int mid = (l + r) / 2;
    bool leftBetter = newLine.eval(l) < node->line.eval(l);
    bool midBetter = newLine.eval(mid) < node->line.eval(mid);

    if (midBetter) swap(node->line, newLine);
    if (r - l == 1) return;

    if (leftBetter != midBetter)
        insert(node->left, l, mid, newLine);
    else
        insert(node->right, mid, r, newLine);
}

long long query(Node* node, int l, int r, int x) {
    if (!node) return LLONG_MAX;
    int mid = (l + r) / 2;
    long long res = node->line.eval(x);
    if (r - l == 1) return res;
    if (x < mid)
        return min(res, query(node->left, l, mid, x));
    else
        return min(res, query(node->right, mid, r, x));
}
```

## 🧱 Li Chao Tree (C#)
- This version uses recursive insertion and querying over a segment range. 
- It supports arbitrary line insertion and is robust for non-monotonic DP transitions.

## 🧩 C# Implementation
```cpp
class Line {
    public long M, B;
    public long Eval(long x) => M * x + B;
}

class Node {
    public Line Line;
    public Node Left, Right;
}

class LiChaoTree {
    private Node root;
    private readonly int L, R;

    public LiChaoTree(int l, int r) {
        L = l;
        R = r;
    }

    public void Insert(Line newLine) => Insert(ref root, L, R, newLine);

    private void Insert(ref Node node, int l, int r, Line newLine) {
        if (node == null) {
            node = new Node { Line = newLine };
            return;
        }

        int mid = (l + r) / 2;
        bool leftBetter = newLine.Eval(l) < node.Line.Eval(l);
        bool midBetter = newLine.Eval(mid) < node.Line.Eval(mid);

        if (midBetter) (node.Line, newLine) = (newLine, node.Line);
        if (r - l == 1) return;

        if (leftBetter != midBetter)
            Insert(ref node.Left, l, mid, newLine);
        else
            Insert(ref node.Right, mid, r, newLine);
    }

    public long Query(int x) => Query(root, L, R, x);

    private long Query(Node node, int l, int r, int x) {
        if (node == null) return long.MaxValue;
        int mid = (l + r) / 2;
        long res = node.Line.Eval(x);
        if (r - l == 1) return res;
        if (x < mid)
            return Math.Min(res, Query(node.Left, l, mid, x));
        else
            return Math.Min(res, Query(node.Right, mid, r, x));
    }
}
```

## ⏱️ Complexity Analysis

---

### 🔧 Time Complexity

| Operation            | Convex Hull Trick       | Li Chao Tree         |
|----------------------|-------------------------|----------------------|
| Add line             | `O(1)` amortized        | `O(log N)`           |
| Query at point `x`   | `O(log N)`              | `O(log N)`           |
| Total DP             | `O(N)` or `O(N log N)`  | `O(N log N)`         |

- **Convex Hull Trick** achieves `O(1)` amortized insertion when:
  - Slopes are added in **monotonic order**
  - Queries are made in **increasing order**
  - This allows pointer-based traversal or binary search over the hull

- **Li Chao Tree** supports:
  - **Arbitrary slope insertion**
  - **Randomized queries**
  - Each insertion/query takes `O(log N)` due to segment tree traversal

- Both techniques reduce naive `O(N^2)` DP transitions to **near-linear time**, making them essential for performance-critical problems.

---

### 💾 Space Complexity

- **Convex Hull Trick**:  
  - `O(N)` for storing lines in a deque or vector  
  - No auxiliary structures — ideal for memory-sensitive environments  
  - Simple to implement and debug

- **Li Chao Tree**:  
  - `O(N log N)` nodes in worst case  
  - Each insertion may split across multiple segments  
  - Requires dynamic memory allocation and tree maintenance  
  - More robust, but heavier

---

## 🧠 Key Concepts Recap

Let’s reinforce the architectural essence behind both structures:

- **Linear DP transitions** of the form:

```
dp[i] = min(dp[j] + m_j * x_i + b_j)
```

  can be optimized using geometric structures

- **Convex Hull Trick (CHT)**  
  - Best for monotonic slopes and queries  
  - Uses binary search or pointer walk over line set  
  - Lightweight and fast

- **Li Chao Tree**  
  - Handles arbitrary slope insertions  
  - Uses segment tree to maintain line dominance  
  - More flexible, slightly heavier

- **Core trick**:  
  Maintain an **envelope of lines** and query the **minimum value** at each point efficiently

- **Result**:  
  Brute-force `O(N^2)` transitions become `O(N log N)` or even `O(N)` in ideal cases

---

## ✅ Conclusion

Convex Hull Trick and Li Chao Tree are **geometric accelerators** for dynamic programming.  
They transform slow, quadratic transitions into **logarithmic-time queries**, enabling real-time optimization in performance-critical tasks.

Whether you're solving:

- 🧮 **Cost minimization** problems  
- 🧠 **Envelope-based DP** transitions  
- 🏁 **Competitive programming** challenges  
- 🧊 **Geometry-driven** state updates

These tools give you:

- ⚡ **Speed** — from brute-force to near-linear  
- 🧠 **Structure** — maintainable and scalable  
- 🛠️ **Control** — over complexity and precision

They’re not just tricks — they’re **architectural upgrades** to your DP engine.  
Once mastered, they become part of your **optimization toolkit**, ready to deploy when brute-force hits a wall.

---
