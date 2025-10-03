# 🧠 Centroid Decomposition — Divide & Conquer Engine for Tree Queries

## 📜 Origin & Motivation

**Centroid Decomposition** is a tree-cutting technique that emerged in the early 2000s from algorithmic graph theory.  
While it’s not attributed to a single inventor, it gained popularity in **competitive programming between 2008–2012**, especially on platforms like **TopCoder**, **Codeforces**, and **SPOJ**.

> Core idea: recursively remove centroids to split the tree into balanced components and process global queries efficiently.

It’s ideal for problems where naive DFS is too slow, and queries span **multiple paths**, **all node pairs**, or require **distance-based aggregation**.

---

## 🧩 Where It’s Used

- 📈 Global path queries (e.g., count paths with sum ≤ k)  
- 🧮 Counting node pairs with constraints (distance, value, parity)  
- 🧠 Subtree isolation to avoid recomputation  
- 🏹 Offline tree queries with heavy preprocessing  
- 🏆 Competitive programming: tree problems with global constraints

---

## 🔁 When to Use Centroid Decomposition vs Alternatives

| Task / Scenario                         | Centroid Decomposition | DFS/BFS | Heavy-Light Decomposition |
|----------------------------------------|-------------------------|---------|---------------------------|
| Global path queries (all pairs)        | ✅                      | ❌      | ❌                        |
| Subtree queries                         | ✅                      | ✅      | ✅                        |
| Online LCA / dynamic updates            | ❌                      | ✅      | ✅                        |
| Distance-based counting                 | ✅                      | ❌      | ❌                        |
| Tree diameter / simple traversal        | ❌                      | ✅      | ❌                        |

---

## 🧱 Core Idea

- A **centroid** is a node whose removal splits the tree into components of size ≤ n/2  
- Recursively remove centroids to build a **centroid tree**  
- At each centroid, process all paths passing through it  
- Use **frequency maps**, **depth arrays**, or **value counters** to answer queries  
- Guarantees **O(log n)** decomposition depth due to balanced splitting

---

## 🚀 Implementation (C++)

```cpp
const int MAXN = 100005;
vector<int> tree[MAXN];
bool removed[MAXN];
int subtreeSize[MAXN];

void computeSubtreeSize(int u, int p) {
    subtreeSize[u] = 1;
    for (int v : tree[u]) {
        if (v != p && !removed[v]) {
            computeSubtreeSize(v, u);
            subtreeSize[u] += subtreeSize[v];
        }
    }
}

int findCentroid(int u, int p, int totalSize) {
    for (int v : tree[u]) {
        if (v != p && !removed[v] && subtreeSize[v] > totalSize / 2)
            return findCentroid(v, u, totalSize);
    }
    return u;
}

void decompose(int u) {
    computeSubtreeSize(u, -1);
    int c = findCentroid(u, -1, subtreeSize[u]);
    removed[c] = true;

    // Process paths through centroid c here

    for (int v : tree[c]) {
        if (!removed[v])
            decompose(v);
    }
}

```


## 🚀 Implementation (C#)
```csharp
public class CentroidDecomposition {
    private List<int>[] tree;
    private bool[] removed;
    private int[] subtreeSize;
    private int n;

    public CentroidDecomposition(int size) {
        n = size;
        tree = new List<int>[n];
        removed = new bool[n];
        subtreeSize = new int[n];
        for (int i = 0; i < n; i++) tree[i] = new List<int>();
    }

    public void AddEdge(int u, int v) {
        tree[u].Add(v);
        tree[v].Add(u);
    }

    private void ComputeSubtreeSize(int u, int p) {
        subtreeSize[u] = 1;
        foreach (int v in tree[u]) {
            if (v != p && !removed[v]) {
                ComputeSubtreeSize(v, u);
                subtreeSize[u] += subtreeSize[v];
            }
        }
    }

    private int FindCentroid(int u, int p, int totalSize) {
        foreach (int v in tree[u]) {
            if (v != p && !removed[v] && subtreeSize[v] > totalSize / 2)
                return FindCentroid(v, u, totalSize);
        }
        return u;
    }

    public void Decompose(int u) {
        ComputeSubtreeSize(u, -1);
        int c = FindCentroid(u, -1, subtreeSize[u]);
        removed[c] = true;

        // Process paths through centroid c here

        foreach (int v in tree[c]) {
            if (!removed[v])
                Decompose(v);
        }
    }
}
```
## ⏱️ Complexity Analysis

- **Decomposition depth:** O(log n) — each centroid removal splits the tree into balanced parts  
- **Per-level processing:** O(n) — each level may touch all nodes once  
- **Total time complexity:** O(n log n) — for most global path-based problems  
- **Space complexity:** O(n) — for tree structure, subtree sizes, and auxiliary data

---

## ⚠️ Pitfalls

- 🔁 **Subtree sizes must be recomputed** before each decomposition step  
- 🧹 **Global state (e.g., frequency maps, depth counters)** must be cleared per centroid to avoid contamination  
- ⚠️ **Double-counting** is easy to introduce when aggregating across centroids — careful separation of contributions is essential  
- 🧩 **Edge cases** like single-node trees or disconnected graphs require special handling  
- 🧠 **Query logic must be designed to respect decomposition boundaries** — naive aggregation may break correctness

---

## ✅ Conclusion

Centroid Decomposition is a **divide-and-conquer engine for tree queries**:

- 🔪 Efficiently splits trees into balanced components  
- 📊 Enables fast counting and aggregation across paths and subtrees  
- 🧠 Ideal for problems where naive DFS is too slow or redundant  
- 🏆 A powerful tool in the algorithmic arsenal for advanced tree problems

👉 **Key takeaway:** Centroid Decomposition transforms complex tree problems into manageable recursive components — elegant, scalable, and indispensable for global path queries.



---
