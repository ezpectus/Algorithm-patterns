# Gomory–Hu Tree: All-pairs Min-cut in Undirected Graphs

---

##  Origin and Motivation
- **Origin:** Proposed by Ralph Gomory and Tien-Yien Hu in 1961.  
- **Motivation:** Computing min-cut for every pair of vertices via max-flow is too expensive (`O(n^2)` flows).  
- **Key Insight:** In undirected graphs, the family of all s–t min-cuts is structured enough to be represented by a weighted tree.  
  - The minimum edge weight on the path between `s` and `t` in this tree equals the min-cut capacity between them.  

---

##  Where It’s Used

| Domain              | Use Case                                      |
|---------------------|-----------------------------------------------|
| Network reliability | Bottleneck detection, redundancy planning     |
| Graph partitioning  | Cut-based clustering, community detection     |
| VLSI / CAD          | Netlist partitioning under capacity constraints |
| Data center networks| Capacity planning, failure scenarios          |
| Algorithmic research| Cut structures, sparsification benchmarks     |

---

##  When to Use vs Alternatives

| Task                        | Gomory–Hu Tree | Per-pair Max-flow | Global Min-cut (Stoer–Wagner) | Sparsifiers / Approximations |
|-----------------------------|----------------|------------------|-------------------------------|------------------------------|
| All-pairs min-cut (exact)   | ✅ n−1 flows    | ❌ n·(n−1)/2 flows| ❌ Only global                | ⚠️ Often approximate         |
| Interactive s–t queries     | ✅ Instant via path-min edge | ❌ Recompute each time | ❌ Not applicable | ✅ Fast but approximate |
| Undirected graphs           | ✅ Native       | ✅ Works          | ✅ Works                      | ✅ Works                     |
| Directed graphs             | ❌ Not supported| ✅ Works          | ❌ Not supported              | ⚠️ Some variants             |
| Exactness                   | ✅ Exact        | ✅ Exact          | ✅ Exact (global only)        | ❌ Approximate               |

---

##  Core Idea
- **Tree Representation:** Build a weighted tree `T` on the same vertex set.  
  - For any two vertices `s, t`, the min-cut capacity = minimum edge weight on the unique path between `s` and `t` in `T`.  
  - Removing that edge induces the corresponding min-cut partition.  

### Construction Sketch (n−1 max-flows)
1. Initialize parent array `P` with `P[i] = 1` (or any root) for `i = 2..n`.  
2. For each `i` from 2..n:  
   - Compute a max-flow/min-cut between `i` and `P[i]` in the original graph.  
   - Let `S` be the source-side of the min-cut containing `i`.  
   - For every `j > i` with `P[j] = P[i]` and `j ∈ S`, set `P[j] = i` (re-parent into i’s side).  
   - Add an edge `(i, P[i])` in the tree with weight = min-cut value.  
   - If `P[i] ∈ S`, swap roles to ensure the tree edge attaches to the opposite side.  

### Querying
- For any `s, t`, locate the path in `T` and take the minimum edge weight.  
- This value = min-cut capacity between `s` and `t`.  
- The cut itself is obtained by removing that edge and taking connected components.  

---

##  Key Takeaway
The Gomory–Hu tree compresses **all-pairs min-cut information** into a single weighted tree using only `n−1` max-flow computations.  
Once built, **any s–t min-cut query is answered instantly** by finding the minimum edge weight along the path between `s` and `t`.  


## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// Dinic's Max-Flow for undirected graphs via directed residual edges
struct Edge {
    int v, rev;
    long long cap;
    Edge(int v, int rev, long long cap) : v(v), rev(rev), cap(cap) {}
};

struct Dinic {
    int n;
    vector<vector<Edge>> g;
    vector<int> level, it;

    Dinic(int n) : n(n), g(n), level(n), it(n) {}

    // Add undirected capacity: two directed edges with cap and cap
    void addUndirected(int u, int v, long long c) {
        addDirected(u, v, c);
        addDirected(v, u, c);
    }

    void addDirected(int u, int v, long long c) {
        Edge a(v, (int)g[v].size(), c);
        Edge b(u, (int)g[u].size(), 0);
        g[u].push_back(a);
        g[v].push_back(b);
    }

    bool bfs(int s, int t) {
        fill(level.begin(), level.end(), -1);
        queue<int> q;
        level[s] = 0;
        q.push(s);
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (auto &e : g[u]) {
                if (level[e.v] < 0 && e.cap > 0) {
                    level[e.v] = level[u] + 1;
                    q.push(e.v);
                }
            }
        }
        return level[t] >= 0;
    }

    long long dfs(int u, int t, long long f) {
        if (u == t) return f;
        for (int &i = it[u]; i < (int)g[u].size(); i++) {
            Edge &e = g[u][i];
            if (e.cap > 0 && level[e.v] == level[u] + 1) {
                long long pushed = dfs(e.v, t, min(f, e.cap));
                if (pushed > 0) {
                    e.cap -= pushed;
                    g[e.v][e.rev].cap += pushed;
                    return pushed;
                }
            }
        }
        return 0;
    }

    long long maxflow(int s, int t) {
        long long flow = 0;
        while (bfs(s, t)) {
            fill(it.begin(), it.end(), 0);
            while (true) {
                long long pushed = dfs(s, t, LLONG_MAX);
                if (pushed == 0) break;
                flow += pushed;
            }
        }
        return flow;
    }

    // After maxflow, retrieve source-side of min-cut (reachable in residual graph)
    vector<char> mincutReachableFrom(int s) {
        vector<char> vis(n, false);
        queue<int> q;
        vis[s] = true;
        q.push(s);
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (auto &e : g[u]) {
                if (!vis[e.v] && e.cap > 0) {
                    vis[e.v] = true;
                    q.push(e.v);
                }
            }
        }
        return vis;
    }
};

// Gomory–Hu Tree builder for undirected graphs
struct GomoryHu {
    int n;
    vector<vector<pair<int,long long>>> tree; // adjacency with weights

    GomoryHu(int n) : n(n), tree(n) {}

    // Build from edge list (1-indexed nodes) to 0-indexed internally
    void build(int N, const vector<tuple<int,int,long long>>& edges) {
        auto buildFlowGraph = [&](int excludeU = -1, int excludeV = -1) -> Dinic {
            Dinic din(N);
            for (auto &e : edges) {
                int u, v; long long c;
                tie(u, v, c) = e;
                --u; --v;
                din.addUndirected(u, v, c);
            }
            return din;
        };

        vector<int> parent(N, 0);
        for (int i = 1; i < N; i++) parent[i] = 0; // root = 0
        vector<long long> cutVal(N, 0);

        for (int i = 1; i < N; i++) {
            Dinic din = buildFlowGraph();
            int s = i, t = parent[i];
            long long f = din.maxflow(s, t);
            cutVal[i] = f;

            auto reach = din.mincutReachableFrom(s);

            for (int j = i + 1; j < N; j++) {
                if (parent[j] == parent[i] && reach[j]) {
                    parent[j] = i;
                }
            }
            if (reach[parent[i]]) {
                parent[i] = parent[parent[i]];
            }
        }

        for (int i = 1; i < N; i++) {
            int p = parent[i];
            long long w = cutVal[i];
            tree[i].push_back({p, w});
            tree[p].push_back({i, w});
        }
    }

    // Query min-cut capacity between u and v via path minimum (0-indexed nodes)
    long long minCutOnPath(int u, int v) {
        vector<char> vis(n, false);
        long long ans = LLONG_MAX;
        bool found = false;
        function<bool(int,long long)> dfs = [&](int x, long long curMin) {
            if (x == v) { ans = curMin; found = true; return true; }
            vis[x] = true;
            for (auto &e : tree[x]) {
                int y = e.first; long long w = e.second;
                if (!vis[y]) {
                    if (dfs(y, min(curMin, w))) return true;
                }
            }
            return false;
        };
        dfs(u, LLONG_MAX);
        return found ? ans : -1;
    }
};

// Example usage
int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);

    int N = 4;
    vector<tuple<int,int,long long>> edges = {
        {1,2,3}, {2,3,4}, {3,4,5}, {1,4,2}, {2,4,1}
    };

    GomoryHu gh(N);
    gh.build(N, edges);

    // Query min-cut between node 1 and 3 (1-indexed -> adjust to 0-index in API)
    long long ans = gh.minCutOnPath(0, 2);
    cout << ans << "\n"; // prints min-cut capacity between 1 and 3
    return 0;
}
```

## Code Highlights

- **Dinic:** Efficient max-flow algorithm used for each of the `n−1` flow computations in the Gomory–Hu construction.  
- **MinCutReachableFrom:** Extracts the source-side of the min-cut via residual reachability after max-flow.  
- **Parent maintenance:** Core of Gomory–Hu; re-parents nodes that share the min-cut side to maintain tree structure.  
- **Tree queries:** The minimum edge weight along the path between two nodes equals the s–t min-cut capacity.  

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class Edge {
    public int V, Rev;
    public long Cap;
    public Edge(int v, int rev, long cap) { V = v; Rev = rev; Cap = cap; }
}

public class Dinic {
    private int n;
    private List<Edge>[] g;
    private int[] level, it;

    public Dinic(int n) {
        this.n = n;
        g = new List<Edge>[n];
        for (int i = 0; i < n; i++) g[i] = new List<Edge>();
        level = new int[n];
        it = new int[n];
    }

    public void AddUndirected(int u, int v, long c) {
        AddDirected(u, v, c);
        AddDirected(v, u, c);
    }

    public void AddDirected(int u, int v, long c) {
        var a = new Edge(v, g[v].Count, c);
        var b = new Edge(u, g[u].Count, 0);
        g[u].Add(a);
        g[v].Add(b);
    }

    private bool Bfs(int s, int t) {
        for (int i = 0; i < n; i++) level[i] = -1;
        var q = new Queue<int>();
        level[s] = 0;
        q.Enqueue(s);
        while (q.Count > 0) {
            int u = q.Dequeue();
            foreach (var e in g[u]) {
                if (level[e.V] < 0 && e.Cap > 0) {
                    level[e.V] = level[u] + 1;
                    q.Enqueue(e.V);
                }
            }
        }
        return level[t] >= 0;
    }

    private long Dfs(int u, int t, long f) {
        if (u == t) return f;
        for (; it[u] < g[u].Count; it[u]++) {
            var e = g[u][it[u]];
            if (e.Cap > 0 && level[e.V] == level[u] + 1) {
                long pushed = Dfs(e.V, t, Math.Min(f, e.Cap));
                if (pushed > 0) {
                    e.Cap -= pushed;
                    g[e.V][e.Rev].Cap += pushed;
                    return pushed;
                }
            }
        }
        return 0;
    }

    public long Maxflow(int s, int t) {
        long flow = 0;
        while (Bfs(s, t)) {
            for (int i = 0; i < n; i++) it[i] = 0;
            while (true) {
                long pushed = Dfs(s, t, long.MaxValue);
                if (pushed == 0) break;
                flow += pushed;
            }
        }
        return flow;
    }

    public bool[] MinCutReachableFrom(int s) {
        var vis = new bool[n];
        var q = new Queue<int>();
        vis[s] = true;
        q.Enqueue(s);
        while (q.Count > 0) {
            int u = q.Dequeue();
            foreach (var e in g[u]) {
                if (!vis[e.V] && e.Cap > 0) {
                    vis[e.V] = true;
                    q.Enqueue(e.V);
                }
            }
        }
        return vis;
    }
}

public class GomoryHu {
    private int n;
    private List<(int,int,long)> edges; // (u,v,c) 1-indexed input
    public List<(int to,long w)>[] Tree;

    public GomoryHu(int n) {
        this.n = n;
        Tree = new List<(int to,long w)>[n];
        for (int i = 0; i < n; i++) Tree[i] = new List<(int,long)>();
        edges = new List<(int,int,long)>();
    }

    public void AddEdge(int u, int v, long c) {
        edges.Add((u, v, c));
    }

    private Dinic BuildFlowGraph() {
        var din = new Dinic(n);
        foreach (var e in edges) {
            int u = e.Item1 - 1;
            int v = e.Item2 - 1;
            long c = e.Item3;
            din.AddUndirected(u, v, c);
        }
        return din;
    }

    public void Build() {
        int N = n;
        var parent = new int[N];
        for (int i = 1; i < N; i++) parent[i] = 0;
        var cutVal = new long[N];

        for (int i = 1; i < N; i++) {
            var din = BuildFlowGraph();
            int s = i, t = parent[i];
            long f = din.Maxflow(s, t);
            cutVal[i] = f;

            var reach = din.MinCutReachableFrom(s);

            for (int j = i + 1; j < N; j++) {
                if (parent[j] == parent[i] && reach[j]) {
                    parent[j] = i;
                }
            }
            if (reach[parent[i]]) {
                parent[i] = parent[parent[i]];
            }
        }

        for (int i = 1; i < N; i++) {
            int p = parent[i];
            long w = cutVal[i];
            Tree[i].Add((p, w));
            Tree[p].Add((i, w));
        }
    }

    public long MinCutOnPath(int u, int v) { // 0-indexed
        var vis = new bool[n];
        long ans = long.MaxValue;
        bool found = false;
        Func<int,long,bool> dfs = null;
        dfs = (x, curMin) => {
            if (x == v) { ans = curMin; found = true; return true; }
            vis[x] = true;
            foreach (var e in Tree[x]) {
                int y = e.to;
                long w = e.w;
                if (!vis[y]) {
                    if (dfs(y, Math.Min(curMin, w))) return true;
                }
            }
            return false;
        };
        dfs(u, long.MaxValue);
        return found ? ans : -1;
    }
}

// Example usage:
// var gh = new GomoryHu(4);
// gh.AddEdge(1,2,3); gh.AddEdge(2,3,4); gh.AddEdge(3,4,5); gh.AddEdge(1,4,2); gh.AddEdge(2,4,1);
// gh.Build();
// long ans = gh.MinCutOnPath(0,2); // min-cut between 1 and 3
```

##  Code Highlights
- **AddUndirected:** Undirected capacities are modeled via symmetric directed edges for simplicity in residual graph handling.  
- **Build:** Performs `n−1` max-flow runs; re-parents nodes based on residual reachability to construct the Gomory–Hu tree.  
- **MinCutOnPath:** A single DFS computes the minimum edge weight along the unique path, yielding the s–t min-cut.  

---

##  Complexity Analysis
- **Build time:** `n−1` max-flow computations.  
  - With Dinic on general capacities: roughly `O((n−1) · E · sqrt(V))` to `O((n−1) · E · V^2)` depending on graph density.  
  - In practice, Dinic is efficient on sparse graphs.  
  - For unit capacities or special cases, faster bounds apply.  
- **Tree size:** `O(n)` edges.  
- **Query time (s–t min-cut):**  
  - `O(n)` with naive DFS.  
  - `O(log n)` after preprocessing with LCA/RMQ for path-minimum queries.  
- **Space:**  
  - `O(n + E)` for the input graph.  
  - `O(n)` for the tree.  
  - `O(E)` for residual networks per flow run.  

---

##  Impact of Design Choices

| Choice                          | Effect                                                                 |
|---------------------------------|------------------------------------------------------------------------|
| Max-flow backend (Dinic vs Push–Relabel) | Throughput vs simplicity; push–relabel often faster on dense graphs |
| Undirected modeling             | Symmetric directed edges keep residual logic simple                    |
| Parent update rule              | Critical for correctness of GH decomposition                           |
| Path-min queries                | LCA/RMQ preprocessing yields `O(log n)` s–t min-cut queries            |
| Edge capacities type            | Use 64-bit to avoid overflow on summed flows                           |

---

##  Pitfalls
- **Directed graphs:** Classical Gomory–Hu is for undirected graphs; applying it naively to directed graphs is incorrect.  
- **Residual side extraction:** Must use reachability in residual graph after max-flow to determine the cut partition; mixing preflow state breaks correctness.  
- **Re-parenting mistakes:** Incorrectly updating `parent[]` for nodes sharing the cut side leads to an invalid tree.  
- **Integer overflow:** Always use 64-bit integers for capacities and flow accumulation.  
- **Performance traps:** Rebuilding the flow graph from scratch for each run is simpler but may be slow; careful reuse can improve speed but complicates code.  

---

##  Conclusion
- **What it gives:** A compact, exact representation of all-pairs min-cuts in an undirected capacitated graph using only `n−1` max-flow runs.  
- **Why it matters:** Transforms repeated min-cut queries into fast path-min queries on a tree, enabling interactive analysis and planning.  
- **Key takeaway:** Build the Gomory–Hu tree once; then any s–t min-cut equals the minimum edge weight along the path between `s` and `t` in that tree.  


---
