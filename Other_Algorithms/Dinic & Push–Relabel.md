# 🧠 Dinic & Push–Relabel — Max Flow Engines for Network Optimization

## 📜 Origin & Motivation

**Maximum Flow** algorithms solve problems where you want to push as much flow as possible from a **source** to a **sink** through a network with **capacity constraints**.

- **Dinic’s Algorithm** was introduced by **E.A. Dinic in 1970**. It builds layered level graphs using BFS and finds blocking flows via DFS. It’s especially efficient on **sparse graphs** and **unit-capacity networks**.
- **Push–Relabel Algorithm** was developed by **Andrew V. Goldberg and Robert E. Tarjan in 1986**. It uses **local operations** (pushes and relabels) and is highly effective on **dense graphs**, with strong potential for **parallelization**.

These algorithms are central to:

- 🌐 Network routing  
- 🧮 Bipartite matching  
- 🧠 Image segmentation  
- 🔁 Circulation and connectivity problems  
- 🏆 Competitive programming and industrial optimization

---

## 🧩 Where They’re Used

- 🌐 Network flow optimization (bandwidth, traffic, logistics)  
- 🧮 Bipartite matching and assignment problems  
- 🧠 Image segmentation and graph cuts  
- 🏗️ Project scheduling and resource allocation  
- 🏆 Competitive programming: classic flow problems, min-cut, matching

---

## 🔁 When to Use Dinic vs Push–Relabel

| Scenario / Graph Type              | Use Dinic        | Use Push–Relabel |
|-----------------------------------|------------------|------------------|
| Sparse graphs (few edges)         | ✅               | ❌               |
| Dense graphs (many edges)         | ❌               | ✅               |
| Small graphs                      | ✅               | ✅               |
| Bipartite matching                | ✅               | ❌               |
| Circulation / cost flow variants  | ❌               | ✅ (with tweaks) |
| Parallelization potential         | ❌               | ✅               |

---

## 🧱 Core Ideas

### 🔹 Dinic’s Algorithm

- Builds a **level graph** using BFS from the source  
- Uses **DFS** to find blocking flows within the level graph  
- Repeats until no more augmenting paths exist  
- Efficient for **unit capacity**, **bipartite graphs**, and **sparse networks**  
- Guarantees **polynomial time** for bounded capacities

### 🔹 Push–Relabel Algorithm

- Maintains **preflows** (flow may exceed capacity at intermediate nodes)  
- Pushes **excess flow locally** between adjacent nodes  
- Relabels nodes to increase their height and enable further pushes  
- Uses **height labels** and **excess tracking**  
- Can be **parallelized** and optimized with heuristics (e.g., highest-label selection, gap relabeling)

---


## 🚀 Dinic’s Implementation (C++)
```cpp
struct Edge {
    int to, rev;
    int cap;
};

class Dinic {
    int n;
    vector<vector<Edge>> adj;
    vector<int> level, ptr;

public:
    Dinic(int n) : n(n), adj(n), level(n), ptr(n) {}

    void addEdge(int u, int v, int cap) {
        adj[u].push_back({v, (int)adj[v].size(), cap});
        adj[v].push_back({u, (int)adj[u].size() - 1, 0});
    }

    bool bfs(int s, int t) {
        fill(level.begin(), level.end(), -1);
        queue<int> q;
        level[s] = 0;
        q.push(s);
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (auto &e : adj[u]) {
                if (e.cap && level[e.to] == -1) {
                    level[e.to] = level[u] + 1;
                    q.push(e.to);
                }
            }
        }
        return level[t] != -1;
    }

    int dfs(int u, int t, int flow) {
        if (u == t || flow == 0) return flow;
        for (int &i = ptr[u]; i < adj[u].size(); ++i) {
            Edge &e = adj[u][i];
            if (level[e.to] == level[u] + 1 && e.cap) {
                int pushed = dfs(e.to, t, min(flow, e.cap));
                if (pushed) {
                    e.cap -= pushed;
                    adj[e.to][e.rev].cap += pushed;
                    return pushed;
                }
            }
        }
        return 0;
    }

    int maxFlow(int s, int t) {
        int flow = 0;
        while (bfs(s, t)) {
            fill(ptr.begin(), ptr.end(), 0);
            while (int pushed = dfs(s, t, INT_MAX))
                flow += pushed;
        }
        return flow;
    }
};
```


## 🚀 Push–Relabel Implementation (C#)
```csharp
public class PushRelabel {
    private readonly int n;
    private readonly List<(int to, int rev, int cap)>[] adj;
    private readonly int[] height, excess;

    public PushRelabel(int size) {
        n = size;
        adj = new List<(int, int, int)>[n];
        for (int i = 0; i < n; i++) adj[i] = new List<(int, int, int)>();
        height = new int[n];
        excess = new int[n];
    }

    public void AddEdge(int u, int v, int cap) {
        adj[u].Add((v, adj[v].Count, cap));
        adj[v].Add((u, adj[u].Count - 1, 0));
    }

    private void Push(int u, int i) {
        var (v, rev, cap) = adj[u][i];
        int flow = Math.Min(excess[u], cap);
        if (flow > 0 && height[u] == height[v] + 1) {
            adj[u][i] = (v, rev, cap - flow);
            var back = adj[v][rev];
            adj[v][rev] = (back.to, back.rev, back.cap + flow);
            excess[u] -= flow;
            excess[v] += flow;
        }
    }

    private void Relabel(int u) {
        int minHeight = int.MaxValue;
        foreach (var (v, _, cap) in adj[u])
            if (cap > 0)
                minHeight = Math.Min(minHeight, height[v]);
        if (minHeight < int.MaxValue)
            height[u] = minHeight + 1;
    }

    public int MaxFlow(int s, int t) {
        height[s] = n;
        foreach (var (v, _, cap) in adj[s]) {
            if (cap > 0) {
                adj[s][adj[s].IndexOf((v, _, cap))] = (v, _, 0);
                var back = adj[v][adj[v].FindIndex(e => e.to == s)];
                adj[v][adj[v].IndexOf(back)] = (back.to, back.rev, back.cap + cap);
                excess[v] += cap;
            }
        }

        Queue<int> active = new Queue<int>();
        for (int i = 0; i < n; i++)
            if (i != s && i != t && excess[i] > 0)
                active.Enqueue(i);

        while (active.Count > 0) {
            int u = active.Dequeue();
            bool pushed = false;
            for (int i = 0; i < adj[u].Count; i++) {
                int oldExcess = excess[u];
                Push(u, i);
                if (excess[u] < oldExcess) {
                    pushed = true;
                    if (excess[adj[u][i].to] > 0 && adj[u][i].to != s && adj[u][i].to != t)
                        active.Enqueue(adj[u][i].to);
                }
            }
            if (!pushed) Relabel(u);
            if (excess[u] > 0) active.Enqueue(u);
        }

        return excess[t];
    }
}
```

## ⏱️ Complexity Analysis

Understanding the performance of **Dinic** and **Push–Relabel** requires analyzing both worst-case behavior and practical efficiency across graph types. Each algorithm has strengths depending on graph density, capacity distribution, and structural constraints.

| Algorithm        | Time Complexity                                                                 | Space Complexity | Best Use Case                            |
|------------------|----------------------------------------------------------------------------------|------------------|------------------------------------------|
| **Dinic**        | - Worst-case: **O(V²·E)**  
|                  | - Optimized: **O(E·√V)** for unit capacities  
|                  | - Degenerate layered networks: **O(V³)**                                         | O(V + E)         | Sparse graphs, bipartite matching        |
| **Push–Relabel** | - Worst-case: **O(V³)**  
|                  | - With heuristics (highest-label, gap relabeling): **O(V²·√E)**  
|                  | - Parallelizable variants: **near-linear in practice**                           | O(V + E)         | Dense graphs, circulation, parallel flow |

---

### 🔍 Dinic’s Performance Breakdown

- Builds a **level graph** using BFS from the source  
- Uses DFS to find **blocking flows** within the level graph  
- Repeats until no more augmenting paths exist  
- Performs best on **unit-capacity graphs**, **bipartite networks**, and **grid-like structures**  
- In sparse graphs, level graphs are narrow → DFS remains efficient  
- In dense graphs, level graphs become wide → DFS may explore redundant paths

---

### 🔍 Push–Relabel Performance Breakdown

- Maintains **preflows** and pushes excess locally between adjacent nodes  
- Relabels nodes to increase height and enable further pushes  
- Heuristics like **highest-label selection** and **gap relabeling** drastically improve performance  
- Particularly effective on **dense graphs** where Dinic’s layered approach becomes costly  
- Can be **parallelized**, making it suitable for multi-threaded or GPU-based environments  
- Supports **circulation**, **cost flow**, and **generalized flow** variants with extensions

---

### 🧠 Architectural Insight

- **Dinic** is structurally layered and globally coordinated — ideal for problems with clear path structure  
- **Push–Relabel** is locally reactive and state-driven — ideal for problems with high connectivity and excess flow propagation  
- Both rely on **residual graphs**, **reverse edges**, and **capacity tracking**, but differ in traversal and update philosophy



---

## ⚠️ Pitfalls

Even though both algorithms are robust, they require careful implementation to avoid subtle bugs and performance traps.

- 🧠 **Dinic’s DFS must strictly follow level graph structure**  
  Skipping levels or revisiting nodes outside the current layer breaks correctness and may cause infinite loops or incorrect flow.

- 🔁 **Push–Relabel requires precise relabeling logic**  
  If relabeling is too aggressive or not triggered when needed, the algorithm may stall or loop indefinitely.

- 🧹 **Residual graph maintenance is critical**  
  Both algorithms rely on residual capacities and reverse edges. Forgetting to update reverse flows or misindexing edges leads to invalid flow states.

- ⚠️ **Integer overflow risks**  
  Flow values can grow large, especially in networks with high capacities. Always use `long long` (C++) or `BigInteger` (C#) when needed.

- 🧩 **Edge direction and reverse indexing must be symmetric**  
  Each edge must have a corresponding reverse edge with correct indexing. Mistakes here silently corrupt flow propagation.

- 🧠 **Push–Relabel may underperform on sparse graphs**  
  Without heuristics, it may perform unnecessary relabels and pushes, making Dinic a better choice in such cases.

---

## ✅ Conclusion

**Dinic and Push–Relabel** are two foundational engines for solving **maximum flow** problems — each with distinct architectural strengths.

- 🔄 **Dinic** builds layered BFS structures and uses DFS to find blocking flows.  
  It’s elegant, intuitive, and highly efficient on sparse graphs and bipartite networks.

- 💧 **Push–Relabel** operates locally, pushing excess and relabeling nodes to maintain flow feasibility.  
  It’s robust, parallelizable, and excels on dense graphs and circulation problems.

- 🧠 Both algorithms maintain **residual graphs**, support **reverse edges**, and guarantee **maximum flow correctness** when implemented properly.

- 🏆 In competitive programming and real-world optimization, choosing the right engine depends on graph density, capacity distribution, and parallelization needs.

👉 **Key takeaway:**  
Maximum flow is not just about finding paths — it’s about building scalable, architecture-aware engines for global network reasoning.  
**Dinic** is your go-to for sparse, structured graphs.  
**Push–Relabel** is your powerhouse for dense, dynamic, and parallel workloads.  
Together, they form the backbone of modern flow-based problem solving.
