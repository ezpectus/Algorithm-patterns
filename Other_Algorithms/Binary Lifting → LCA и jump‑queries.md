# 🔷 Binary Lifting — Tree Jumping & LCA Engine

---

## 📜 Origin & Motivation

Binary Lifting emerged in the early 2000s within the competitive programming community, notably on platforms like TopCoder, Codeforces, and the Polish Olympiad in Informatics. 
It was designed to solve tree-based ancestor queries and LCA (Lowest Common Ancestor) problems more efficiently than naive traversal or Euler Tour + RMQ approaches.

Before Binary Lifting:
- Finding the k-th ancestor of a node required O(k) time.
- LCA queries relied on segment trees or heavy preprocessing.

Binary Lifting introduced a jump table that leverages the binary representation of integers. It decomposes jumps into powers of two, allowing:
- O(log n) LCA queries
- O(log k) ancestor jumps
- O(n · log n) preprocessing

It became a standard in tree algorithms due to its simplicity, speed, and adaptability.

---

## 🧱 Architecture

| Component         | Role                                                  |
|------------------|-------------------------------------------------------|
| `up[node][j]`     | Stores the 2^j-th ancestor of `node`                 |
| `depth[node]`     | Stores depth of each node for LCA alignment          |
| `logN`            | Maximum jump exponent (⌈log₂(n)⌉)                    |
| DFS traversal     | Initializes depth and immediate parent               |
| Binary table fill | Builds jump table using dynamic programming          |

---

## 🔧 Execution Phases

1. **DFS Initialization**  
   - Set `up[node][0] = parent[node]`  
   - Set `depth[node] = depth[parent] + 1`

2. **Jump Table Construction**  
   - For each `j > 0`:  
     `up[node][j] = up[up[node][j-1]][j-1]`

3. **LCA Query**  
   - Equalize depths using binary jumps  
   - Jump both nodes upward until common ancestor found

4. **K-th Ancestor Query**  
   - Decompose `k` into binary  
   - For each set bit `j`, jump to `up[node][j]`

---

## 🧠 Trigger Map

| Trigger Condition         | Action                                 |
|---------------------------|----------------------------------------|
| `depth[u] < depth[v]`     | Swap nodes to ensure `u` is deeper     |
| `depth[u] != depth[v]`    | Jump `u` upward to match depth         |
| `up[u][j] != up[v][j]`    | Jump both nodes upward simultaneously  |
| `u == v`                  | Return LCA                             |
| `k & (1 << j)`            | Jump to 2^j-th ancestor                |

---

## 🚀 C++ Code Skeleton

```cpp
const int MAXN = 1e5;
const int LOG = 20;
vector<int> tree[MAXN];
int up[MAXN][LOG], depth[MAXN];

void dfs(int node, int parent) {
    up[node][0] = parent;
    depth[node] = depth[parent] + 1;
    for (int j = 1; j < LOG; ++j)
        up[node][j] = up[up[node][j - 1]][j - 1];
    for (int child : tree[node])
        if (child != parent)
            dfs(child, node);
}

int lift(int node, int k) {
    for (int j = 0; j < LOG; ++j)
        if (k & (1 << j))
            node = up[node][j];
    return node;
}

int lca(int u, int v) {
    if (depth[u] < depth[v]) swap(u, v);
    u = lift(u, depth[u] - depth[v]);
    if (u == v) return u;
    for (int j = LOG - 1; j >= 0; --j)
        if (up[u][j] != up[v][j])
            u = up[u][j], v = up[v][j];
    return up[u][0];
}

```


## 🚀 C# Code Skeleton
```csharp
const int MAXN = 100000;
const int LOG = 20;
List<int>[] tree = new List<int>[MAXN];
int[,] up = new int[MAXN, LOG];
int[] depth = new int[MAXN];

void DFS(int node, int parent) {
    up[node, 0] = parent;
    depth[node] = depth[parent] + 1;
    for (int j = 1; j < LOG; j++)
        up[node, j] = up[up[node, j - 1], j - 1];
    foreach (int child in tree[node])
        if (child != parent)
            DFS(child, node);
}

int Lift(int node, int k) {
    for (int j = 0; j < LOG; j++)
        if ((k & (1 << j)) != 0)
            node = up[node, j];
    return node;
}

int LCA(int u, int v) {
    if (depth[u] < depth[v]) (u, v) = (v, u);
    u = Lift(u, depth[u] - depth[v]);
    if (u == v) return u;
    for (int j = LOG - 1; j >= 0; j--)
        if (up[u, j] != up[v, j]) {
            u = up[u, j];
            v = up[v, j];
        }
    return up[u, 0];
}
```

## ⏱️ Complexity — Binary Lifting

Binary Lifting achieves logarithmic query times by precomputing jump tables for each node in a tree. 
The preprocessing phase builds a table `up[node][j]` that stores the 2^j-th ancestor of each node. 
This allows both LCA and k-th ancestor queries to be answered in O(log n) time.

| Operation              | Time Complexity      | Description |
|------------------------|----------------------|-------------|
| Preprocessing          | O(n · log n)         | DFS traversal + jump table fill for each node and each level |
| LCA Query              | O(log n)             | Depth equalization + binary jumps upward |
| K-th Ancestor Query    | O(log k)             | Binary decomposition of k into powers of two |
| Space                  | O(n · log n)         | Jump table stores log n ancestors per node |

---

## ⚠️ Pitfalls — Binary Lifting

Despite its elegance, Binary Lifting has several architectural constraints that must be respected to avoid incorrect behavior or inefficiencies.

- **Static Tree Requirement**  
  The tree must remain unchanged after preprocessing. No insertions, deletions, or reparenting are allowed. For dynamic trees, use Heavy-Light Decomposition or Link-Cut Trees.

- **Correct Depth Initialization**  
  Depth must be assigned during DFS traversal. Incorrect depth values will break LCA alignment and ancestor jumps.

- **Bottom-Up Table Construction**  
  The jump table must be filled in increasing order of `j`. Each entry depends on the previous level:  
  `up[node][j] = up[up[node][j-1]][j-1]`

- **No Lazy Filling**  
  You cannot fill entries on demand during queries. All entries must be precomputed to guarantee O(log n) performance.

---

## ✅ Use Cases — Binary Lifting

Binary Lifting is applicable in a wide range of tree-based problems, especially where fast ancestor access or path analysis is required.

- **LCA Queries in Static Trees**  
  Efficient resolution of lowest common ancestor between any two nodes.

- **K-th Ancestor Queries**  
  Jumping upward by k steps using binary decomposition.

- **Distance Between Nodes**  
  Compute distance as `depth[u] + depth[v] - 2 · depth[lca(u, v)]`

- **Path-Based Aggregations**  
  Combine values along the path (min, max, sum) using auxiliary structures like segment trees or binary lifting with metadata.

- **Tree-Based DP Optimizations**  
  Use jump tables to propagate or query DP values efficiently across tree levels.

---

## 🧩 Summary — Binary Lifting

Binary Lifting is a foundational tree algorithm that enables fast ancestor and LCA queries using precomputed jump tables.  
It replaces naive traversal with logarithmic jumps, leveraging binary decomposition and layered propagation.  
Widely used in competitive programming, graph theory, and system design, it offers a clean and scalable solution for static tree analysis.

It’s not just a trick — it’s a **jump engine** for tree traversal, built on binary logic and architectural clarity.



---
