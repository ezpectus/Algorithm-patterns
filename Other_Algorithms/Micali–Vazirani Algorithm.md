# 🔷 Micali–Vazirani Algorithm — General Graph Matching in O(√E)

---

## 📜 Origin & Motivation

The **Micali–Vazirani algorithm**, introduced in 1980 by **Silvio Micali** and **Vijay Vazirani**, remains the fastest known algorithm for computing **maximum cardinality matching in general graphs**.  
It improves upon Edmonds’ blossom algorithm by achieving a time complexity of **O(√E · V)**, where *E* is the number of edges and *V* is the number of vertices.

> Core idea: use layered BFS and DFS phases to find shortest augmenting paths in general (non-bipartite) graphs, while carefully managing blossoms and alternating paths.

Unlike bipartite matching (Hopcroft–Karp), general graphs require handling **odd-length cycles** (blossoms), which complicates path discovery.  
Micali–Vazirani introduces **petals, buds, and layers** to manage this complexity efficiently.

---

## 🧱 Architecture

| Component         | Role                                                       |
|------------------|------------------------------------------------------------|
| `Layered BFS`     | Builds alternating path layers from unmatched vertices     |
| `DFS Phase`       | Searches for augmenting paths within layered structure     |
| `Blossom Handling`| Contracts odd cycles to preserve alternating path validity |
| `Petal/Bud`       | Internal structures to manage contracted blossoms          |
| `Matching[]`      | Stores current matching state                              |
| `Visited[]`       | Tracks exploration during DFS                              |

---

## 🔧 Execution Phases

1. **Initialization**  
   - Start with empty matching  
   - Identify free vertices

2. **Layered BFS Construction**  
   - Build alternating layers from free vertices  
   - Track even/odd levels to distinguish path types

3. **DFS Augmentation**  
   - Search for shortest augmenting paths  
   - Use petals and buds to navigate blossoms

4. **Augment Matching**  
   - Flip edges along augmenting path  
   - Update matching array

5. **Repeat Until No Augmenting Path Exists**  
   - Each phase finds all shortest augmenting paths  
   - Guarantees O(√E) phases

---

## 🚀 C++ Code Skeleton (Simplified)

```cpp
const int MAXN = 1000;
vector<int> graph[MAXN];
int match[MAXN], base[MAXN], parent[MAXN];
bool used[MAXN], blossom[MAXN];
queue<int> q;

int lca(int a, int b) {
    bool visited[MAXN] = {};
    while (true) {
        a = base[a];
        visited[a] = true;
        if (match[a] == -1) break;
        a = parent[match[a]];
    }
    while (true) {
        b = base[b];
        if (visited[b]) return b;
        if (match[b] == -1) break;
        b = parent[match[b]];
    }
    return -1;
}

void markPath(int v, int b, int child) {
    while (base[v] != b) {
        blossom[base[v]] = blossom[base[match[v]]] = true;
        parent[v] = child;
        child = match[v];
        v = parent[match[v]];
    }
}

bool bfs(int root, int n) {
    fill(used, used + n, false);
    fill(parent, parent + n, -1);
    for (int i = 0; i < n; ++i) base[i] = i;
    q = queue<int>();
    q.push(root);
    used[root] = true;

    while (!q.empty()) {
        int v = q.front(); q.pop();
        for (int u : graph[v]) {
            if (base[v] == base[u] || match[v] == u) continue;
            if (u == root || (match[u] != -1 && parent[match[u]] != -1)) {
                int curBase = lca(v, u);
                fill(blossom, blossom + n, false);
                markPath(v, curBase, u);
                markPath(u, curBase, v);
                for (int i = 0; i < n; ++i)
                    if (blossom[base[i]]) {
                        base[i] = curBase;
                        if (!used[i]) {
                            used[i] = true;
                            q.push(i);
                        }
                    }
            } else if (parent[u] == -1) {
                parent[u] = v;
                if (match[u] == -1) {
                    int cur = u;
                    while (cur != -1) {
                        int pv = parent[cur], pp = match[pv];
                        match[cur] = pv;
                        match[pv] = cur;
                        cur = pp;
                    }
                    return true;
                }
                used[match[u]] = true;
                q.push(match[u]);
            }
        }
    }
    return false;
}

int maxMatching(int n) {
    fill(match, match + n, -1);
    int res = 0;
    for (int i = 0; i < n; ++i)
        if (match[i] == -1 && bfs(i, n))
            ++res;
    return res;
}
```

## 🚀 C# Code Skeleton (Simplified)
```cpp
class MicaliVazirani {
    List<int>[] graph;
    int[] match, baseNode, parent;
    bool[] used, blossom;
    Queue<int> q;

    public MicaliVazirani(int n) {
        graph = new List<int>[n];
        for (int i = 0; i < n; i++) graph[i] = new List<int>();
        match = new int[n];
        baseNode = new int[n];
        parent = new int[n];
        used = new bool[n];
        blossom = new bool[n];
        q = new Queue<int>();
    }

    int LCA(int a, int b) {
        bool[] visited = new bool[graph.Length];
        while (true) {
            a = baseNode[a];
            visited[a] = true;
            if (match[a] == -1) break;
            a = parent[match[a]];
        }
        while (true) {
            b = baseNode[b];
            if (visited[b]) return b;
            if (match[b] == -1) break;
            b = parent[match[b]];
        }
        return -1;
    }

    void MarkPath(int v, int b, int child) {
        while (baseNode[v] != b) {
            blossom[baseNode[v]] = blossom[baseNode[match[v]]] = true;
            parent[v] = child;
            child = match[v];
            v = parent[match[v]];
        }
    }

    bool BFS(int root) {
        Array.Fill(used, false);
        Array.Fill(parent, -1);
        for (int i = 0; i < graph.Length; i++) baseNode[i] = i;
        q.Clear();
        q.Enqueue(root);
        used[root] = true;

        while (q.Count > 0) {
            int v = q.Dequeue();
            foreach (int u in graph[v]) {
                if (baseNode[v] == baseNode[u] || match[v] == u) continue;
                if (u == root || (match[u] != -1 && parent[match[u]] != -1)) {
                    int curBase = LCA(v, u);
                    Array.Fill(blossom, false);
                    MarkPath(v, curBase, u);
                    MarkPath(u, curBase, v);
                    for (int i = 0; i < graph.Length; i++) {
                        if (blossom[baseNode[i]]) {
                            baseNode[i] = curBase;
                            if (!used[i]) {
                                used[i] = true;
                                q.Enqueue(i);
                            }
                        }
                    }
                } else if (parent[u] == -1) {
                    parent[u] = v;
                    if (match[u] == -1) {
                        int cur = u;
                        while (cur != -1) {
                            int pv = parent[cur], pp = match[pv];
                            match[cur] = pv;
                            match[pv] = cur;
                            cur = pp;
                        }
                        return true;
                    }
                    used[match[u]] = true;
                    q.Enqueue(match[u]);
                }
            }
        }
        return false;
    }

    public int MaxMatching() {
        Array.Fill(match, -1);
        int res = 0;
        for (int i = 0; i < graph.Length; i++)
            if (match[i] == -1 && BFS(i))
                res++;
        return res;
    }
}
```

## ⏱️ Complexity Analysis

| Operation              | Time Complexity      | Description |
|------------------------|----------------------|-------------|
| BFS Layer Construction | O(E) per phase       | Builds alternating layers from unmatched vertices using adjacency traversal |
| DFS Augmentation       | O(V) per path        | Searches for shortest augmenting paths within layered structure |
| Total Matching         | O(√E · V)            | Proven bound for general graphs with efficient blossom handling |
| Space                  | O(V + E)             | Stores graph, matching state, blossom metadata, and search layers |

### Notes:
- The algorithm runs in **O(√E · V)** total time, which is optimal for general graphs.
- Each phase finds **all shortest augmenting paths**, ensuring minimal recomputation.
- Space usage is linear in the number of vertices and edges, with additional arrays for blossom contraction and path tracking.

---

## ⚠️ Pitfalls

- 🧩 **Blossom Handling Is Complex**  
  - Contracting odd-length cycles (blossoms) requires precise tracking of base nodes, buds, and petals.  
  - Incorrect contraction leads to invalid alternating paths and broken augmentations.  
  - Must maintain blossom integrity across multiple phases.

- ⚠️ **Layer Management Must Be Precise**  
  - BFS layers must alternate between matched and unmatched edges.  
  - Misalignment causes DFS to follow invalid paths or miss augmenting opportunities.  
  - Even/odd layer distinction is critical for correctness.

- 🔁 **Not Suitable for Dynamic Graphs**  
  - The algorithm assumes a **static graph** throughout execution.  
  - Any edge insertion, deletion, or weight change requires **full recomputation**.  
  - No incremental updates or dynamic matching support — use dynamic matching algorithms (e.g. dynamic blossom trees) for evolving graphs.

- 🧠 **Implementation Is Nontrivial**  
  - Requires careful orchestration of multiple arrays: `base[]`, `match[]`, `parent[]`, `blossom[]`, `used[]`  
  - Debugging blossom contraction and path restoration is notoriously difficult  
  - Best implemented with extensive logging and visual tracing

- ⚙️ **Limited Parallelism**  
  - Due to tight dependency between BFS and DFS phases, parallelization is hard  
  - Each phase builds on the previous — no independent augmentations

---

📌 This algorithm is powerful but **architecturally delicate** — ideal for static, dense graphs where maximum matching is critical.  
Use with care, and always validate blossom logic before deployment.

---
