# 🧠 Heavy-Light Decomposition (HLD) — Tree Path Engine

## 📜 Origin & Motivation
Many problems on trees require **path queries** (sum, min, max, xor, etc.) or **subtree updates**.  
Naively, traversing a path can take O(n).  
**Heavy-Light Decomposition** breaks the tree into disjoint chains so that any path can be covered by O(log n) chains.  
Combined with a **Segment Tree / Fenwick Tree**, this enables fast queries and updates.

---

## 🧩 Where It’s Used
- 📊 Path queries (sum, min, max, xor)  
- 🌲 Subtree updates and queries  
- ⚡ LCA computation (alternative to binary lifting)  
- 🎮 Dynamic tree problems in competitive programming  
- 🔗 Hybrid with Segment Tree → range queries on paths  

---

## 🔁 When to Use HLD vs Alternatives

| Task / Scenario              | Use HLD | Use Binary Lifting | Use Euler Tour + Segment Tree |
|------------------------------|---------|--------------------|-------------------------------|
| Path queries (sum/min/max)   | ✅      | ❌                 | ❌                            |
| LCA queries only             | ❌      | ✅                 | ✅                            |
| Subtree queries              | ✅      | ❌                 | ✅                            |
| Dynamic updates on paths     | ✅      | ❌                 | ❌                            |
| Static queries only          | ❌      | ✅                 | ✅                            |

---

## 🧱 Core Idea
1. **Heavy edge:** For each node, choose the child with the largest subtree as the *heavy child*.  
2. **Light edge:** All other edges are *light*.  
3. **Chains:** Heavy edges form chains; light edges break chains.  
4. **Decomposition property:**  
   - Any root‑to‑leaf path crosses at most O(log n) light edges.  
   - Thus, any path can be decomposed into O(log n) chains.  
5. **Segment Tree overlay:** Flatten chains into an array and build a segment tree for fast range queries.

---

## 🚀 Implementation Sketch (C++)

```cpp
struct HLD {
    int n;
    vector<vector<int>> adj;
    vector<int> parent, depth, heavy, head, pos, sz;
    int curPos;

    HLD(int n) : n(n), adj(n), parent(n), depth(n), heavy(n, -1),
                 head(n), pos(n), sz(n), curPos(0) {}

    void addEdge(int u, int v) {
        adj[u].push_back(v);
        adj[v].push_back(u);
    }

    int dfs(int v, int p) {
        parent[v] = p;
        sz[v] = 1;
        int maxSub = 0;
        for (int u : adj[v]) if (u != p) {
            depth[u] = depth[v] + 1;
            int sub = dfs(u, v);
            sz[v] += sub;
            if (sub > maxSub) {
                maxSub = sub;
                heavy[v] = u;
            }
        }
        return sz[v];
    }

    void decompose(int v, int h) {
        head[v] = h;
        pos[v] = curPos++;
        if (heavy[v] != -1)
            decompose(heavy[v], h);
        for (int u : adj[v]) if (u != parent[v] && u != heavy[v])
            decompose(u, u);
    }

    void build(int root = 0) {
        dfs(root, -1);
        decompose(root, root);
    }

    // Example: query path u-v using segment tree
    int query(int u, int v, SegmentTree& st) {
        int res = 0;
        while (head[u] != head[v]) {
            if (depth[head[u]] < depth[head[v]]) swap(u, v);
            res += st.query(pos[head[u]], pos[u]);
            u = parent[head[u]];
        }
        if (depth[u] > depth[v]) swap(u, v);
        res += st.query(pos[u], pos[v]);
        return res;
    }
};
```
- Time: preprocessing O(n), each path query/update touches O(log n) chains; each segment tree op costs O(log n) → O(log² n) per path op.
- Space: O(n) arrays for HLD plus O(n) for segment tree.
- Edge weights: store the weight on the child’s position pos[child] if queries are edge-based.
- Variants: switch segment tree aggregation to min/max/xor; replace lazy add with assignment if needed.


## 🚀 Implementation Sketch (C#)

```cpp
public class HLD
{
    private readonly int n;
    private readonly List<int>[] adj;
    private readonly int[] parent, depth, heavy, head, pos, sz;
    private int curPos;

    public HLD(int n)
    {
        this.n = n;
        adj = new List<int>[n];
        for (int i = 0; i < n; i++) adj[i] = new List<int>();

        parent = new int[n];
        depth  = new int[n];
        heavy  = new int[n];
        head   = new int[n];
        pos    = new int[n];
        sz     = new int[n];

        for (int i = 0; i < n; i++) heavy[i] = -1;
        curPos = 0;
    }

    public void AddEdge(int u, int v)
    {
        adj[u].Add(v);
        adj[v].Add(u);
    }

    private int Dfs(int v, int p)
    {
        parent[v] = p;
        sz[v] = 1;
        int maxSub = 0;

        foreach (var u in adj[v])
        {
            if (u == p) continue;
            depth[u] = depth[v] + 1;
            int sub = Dfs(u, v);
            sz[v] += sub;
            if (sub > maxSub)
            {
                maxSub = sub;
                heavy[v] = u;
            }
        }
        return sz[v];
    }

    private void Decompose(int v, int h)
    {
        head[v] = h;
        pos[v] = curPos++;

        if (heavy[v] != -1)
            Decompose(heavy[v], h);

        foreach (var u in adj[v])
        {
            if (u != parent[v] && u != heavy[v])
                Decompose(u, u);
        }
    }

    public void Build(int root = 0)
    {
        Dfs(root, -1);
        Decompose(root, root);
    }

    // Example: query path u-v using segment tree
    public int Query(int u, int v, SegmentTree st)
    {
        int res = 0;
        while (head[u] != head[v])
        {
            if (depth[head[u]] < depth[head[v]])
            {
                int tmp = u; u = v; v = tmp;
            }
            res += st.Query(pos[head[u]], pos[u]);
            u = parent[head[u]];
        }
        if (depth[u] > depth[v])
        {
            int tmp = u; u = v; v = tmp;
        }
        res += st.Query(pos[u], pos[v]);
        return res;
    }
}
```

- Labels: node values stored at positions; for edge weights, place the edge’s value at the child’s position.
- Updates: both path updates and subtree updates supported via the same segment tree overlay.
- Queries: path sum and subtree sum are shown; swap aggregation for min/max/xor as needed.


---


## ⏱️ Complexity Analysis

- **Preprocessing:** O(n)  
  - DFS to compute subtree sizes, heavy child, parent, depth.  
  - Decomposition into chains.  

- **Query/Update:** O(log² n) with segment tree  
  - O(log n) chains × O(log n) per segment tree query/update.  

- **Space:** O(n)  
  - **HLD arrays:** `parent`, `depth`, `heavy`, `head`, `pos`, `size/base`.  
  - **Segment tree:** values + lazy arrays.  

---

## ⚠️ Pitfalls

- **Indexing:** Flattening nodes into array positions must be consistent; check inclusive ranges in segment queries.  
- **Heavy child selection:** Always pick the child with the largest subtree. Wrong selection breaks the O(log n) chain bound.  
- **Segment tree integration:** Keep query/update interfaces aligned with flattened positions and path decomposition order.  
- **Edge vs node values:** Decide storage model early. For edge queries, place edge weight at the deeper endpoint’s position.  
- **Root choice:** Root affects depth and chain heads; choose a stable root for predictable paths.  
- **Subtree coverage:** Ensure `pos[u]..pos[u]+size[u]-1` is contiguous (it is with this decomposition) before using subtree ops.  

---

## ✅ Conclusion

Heavy-Light Decomposition is a **Tree Path Engine**:

- ⚡ **Breaks chains:** Any path splits into O(log n) chains.  
- 📊 **Fast operations:** Enables efficient path queries and subtree updates.  
- 🔗 **Hybrid:** Works seamlessly with segment trees or Fenwick trees.  
- 🛡️ **Deterministic:** Guarantees predictable worst-case bounds.  

👉 **Key takeaway:** HLD turns tree problems into array problems, unlocking O(log² n) solutions for rich path and subtree operations.





---


