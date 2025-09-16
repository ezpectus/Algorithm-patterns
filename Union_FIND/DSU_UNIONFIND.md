# 🧩 Union-Find (Disjoint Set Union) — Algorithm Template

## 📘 Introduction

**Union-Find**, also known as **Disjoint Set Union (DSU)**, is a data structure designed to manage a collection of elements split into **disjoint (non-overlapping) subsets**.

It supports two fundamental operations:

- `Find(x)` — identifies the representative (leader) of the subset containing element `x`
- `Union(x, y)` — merges the subsets containing elements `x` and `y`

This structure is ideal for problems involving **connectivity**, **grouping**, or **equivalence relations**.

---

## 🧠 Core Idea

> “Each element belongs to a component, and we can merge components by linking their leaders.”

Internally, DSU represents each component as a **tree**, where each node points to its parent.  
To optimize performance, DSU uses **path compression** — flattening the tree during `Find(x)` so future queries are faster.

---

## 🛠️ When to Use DSU

Apply DSU when your problem involves:

- Tracking **connected components** in a dynamic structure
- Merging **groups** based on relationships or events
- Maintaining **equivalence classes** or **group identities**
- Efficiently answering **connectivity queries**

---

## 🔍 Common Scenarios

DSU shines in problems like:

- **Graph connectivity**  
  _e.g., Number of Provinces, Connected Components_

- **Cycle detection** in undirected graphs  
  _e.g., Detect Cycle in Graph_

- **Minimum Spanning Tree** construction  
  _e.g., Kruskal’s Algorithm_

- **Grouping entities**  
  _e.g., Accounts Merge, Friend Circles_

- **Grid-based union**  
  _e.g., Union version of Number of Islands_


## 🧱 DSU Template — C++
```cpp
vector<int> parent;

void init(int n) {
    parent.resize(n);
    for (int i = 0; i < n; ++i)
        parent[i] = i;
}

int find(int x) {
    if (parent[x] != x)
        parent[x] = find(parent[x]); // Path compression
    return parent[x];
}

void unite(int x, int y) {
    int px = find(x);
    int py = find(y);
    if (px != py)
        parent[px] = py;
}
```

## 🧱 DSU Template — C#
```csharp
int[] parent;

void Init(int n) {
    parent = new int[n];
    for (int i = 0; i < n; i++)
        parent[i] = i;
}

int Find(int x) {
    if (parent[x] != x)
        parent[x] = Find(parent[x]); // Path compression
    return parent[x];
}

void Unite(int x, int y) {
    int px = Find(x);
    int py = Find(y);
    if (px != py)
        parent[px] = py;
}
```

## 📊 DSU with Size or Rank

Adding `size[]` or `rank[]` helps optimize unions by attaching smaller trees under larger ones.  
This prevents deep trees and improves performance, especially in repeated merges.

---

### 🧱 C++ Example with Size

```cpp
vector<int> parent, size;

void init(int n) {
    parent.resize(n);
    size.assign(n, 1);
    for (int i = 0; i < n; ++i)
        parent[i] = i;
}

int find(int x) {
    if (parent[x] != x)
        parent[x] = find(parent[x]); // Path compression
    return parent[x];
}

void unite(int x, int y) {
    int px = find(x), py = find(y);
    if (px == py) return;
    if (size[px] < size[py]) swap(px, py);
    parent[py] = px;
    size[px] += size[py];
}
```
## 🧩 Common DSU Problems

DSU is a versatile tool used across a wide range of algorithmic problems. Below are the most common categories where DSU shines:

| Problem Type           | Example Problems                                | Description                                                                 |
|------------------------|--------------------------------------------------|-----------------------------------------------------------------------------|
| Connected Components   | Number of Provinces, Graph Clustering           | Track which nodes belong to the same group or region                       |
| Cycle Detection        | Detect Cycle in Undirected Graph                | Check if adding an edge creates a cycle by testing if two nodes are already connected |
| Kruskal’s MST          | Minimum Spanning Tree                           | Use DSU to decide whether to include an edge without forming a cycle       |
| Grouping               | Accounts Merge, Friend Circles                  | Merge entities based on shared attributes or relationships                 |
| Grid Connectivity      | Union version of Number of Islands              | Treat each cell as a node and merge adjacent land cells into components    |

---

## ⏱️ Complexity

### ⏳ Time Complexity

- **`find(x)` and `union(x, y)`** operations run in **amortized O(α(n))** time  
- Here, **α(n)** is the inverse Ackermann function — which grows extremely slowly  
- For all practical purposes, this is **near-constant time**

### 🧮 Space Complexity

- **O(n)** for the `parent[]` array  
- Optional: **O(n)** for `size[]` or `rank[]` if using optimized unions

---

## ❌ Common Mistakes

Avoid these pitfalls when implementing DSU:

- **Forgetting path compression**  
  → Leads to deep trees and slow `find()` operations

- **Union without checking roots**  
  → May incorrectly merge already-connected components

- **Using DSU when DFS/BFS is more appropriate**  
  → DSU is not ideal for shortest path or traversal problems

- **Not initializing `parent[i] = i`**  
  → Results in undefined behavior and broken structure

- **Confusing indices vs. values**  
  → Especially dangerous when working with non-zero-based or custom IDs

---

## 🧠 Memorization Tips

Here’s how to internalize DSU as a reusable pattern:

- DSU = `parent[]` + `find()` + `unite()`  
- Always apply **path compression** in `find()`  
- Use `size[]` or `rank[]` to optimize `unite()`  
- Use `map<int, int>` if IDs are **non-sequential or string-based**  
- Think in terms of **components**, not individual nodes

---

## 🧗 DSU Training Progression

Master DSU step-by-step:

- ✅ Basic implementation (C++ / C#)
- ✅ Component merging
- ✅ Counting connected groups
- ✅ DSU with `size[]` or `rank[]`
- ✅ DSU with string or custom IDs (`map`-based)
- ✅ DSU in grid-based problems (e.g., island merging)
- ✅ DSU with rollback (for persistent or historical state tracking)

---

## 🧠 Conclusion

DSU is a **modular, scalable tool** for managing connectivity and equivalence.  
Once internalized, it becomes your go-to strategy for:

- 🔗 Efficient **component tracking**
- 🔄 Dynamic **merging**
- 📈 Graph-based **optimization**

> **Mastering DSU** means mastering the art of **structural control** — knowing:
> - When to **merge**
> - How to **compress**
> - What to **track**

It’s not just an algorithm — it’s a mindset for building systems that scale.



---
