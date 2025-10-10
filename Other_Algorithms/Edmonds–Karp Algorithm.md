# 🧠 Edmonds–Karp Algorithm — Max Flow via BFS (O(V·E²))

---

## 📜 Origin & Motivation

The **Edmonds–Karp algorithm** is a refinement of the **Ford–Fulkerson method**, introduced by **Jack Edmonds** and **Richard Karp** in 1972.  
It solves the **maximum flow problem** in a directed graph with capacities, using **BFS** to find shortest augmenting paths.

Unlike naive DFS-based approaches, Edmonds–Karp guarantees **polynomial time** and avoids infinite loops on cyclic graphs.

---

## 🧩 Use Cases

- 🔗 Network flow optimization  
- 🧬 Data routing and bandwidth allocation  
- 🏭 Supply chain and logistics modeling  
- 🎮 Competitive programming: clean max-flow engine

---

## 🔁 When to Use Edmonds–Karp vs Alternatives

| Scenario                          | Edmonds–Karp | Ford–Fulkerson | Dinic’s | Push-Relabel |
|----------------------------------|--------------|----------------|--------|--------------|
| Small graphs (V ≤ 1000)          | ✅           | ✅              | ✅      | ✅            |
| Large graphs (V ≥ 10⁴)           | ❌           | ❌              | ✅      | ✅            |
| Weighted edges                   | ✅ (capacities only) | ✅      | ✅      | ✅            |
| Online queries                   | ❌           | ❌              | ❌      | ❌            |
| Deterministic performance        | ✅           | ❌              | ✅      | ✅            |

---

## 🧱 Core Architecture

### 🎯 Triggers

| Condition                        | Action in Code           |
|----------------------------------|--------------------------|
| Residual capacity > 0           | Include edge in BFS      |
| BFS finds path from s to t      | Augment flow             |
| No path found                   | Terminate algorithm      |

---

### 🔧 Algorithm Steps

1. Initialize residual graph with forward and reverse edges  
2. While a path exists from `s` to `t`:
   - Use **BFS** to find shortest augmenting path  
   - Track parent and edge index for backtracking  
   - Compute **minimum residual capacity** along the path  
   - Update residual capacities (forward and reverse)  
   - Add path flow to total flow

---

## 🚀 C++ Implementation

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

struct Edge {
    int to, rev;
    int cap;
};

class EdmondsKarp {
public:
    int V;
    vector<vector<Edge>> adj;

    EdmondsKarp(int V) : V(V), adj(V) {}

    void addEdge(int u, int v, int cap) {
        adj[u].push_back({v, (int)adj[v].size(), cap});
        adj[v].push_back({u, (int)adj[u].size() - 1, 0});
    }

    int bfs(int s, int t, vector<int>& parent, vector<int>& edgeIndex) {
        fill(parent.begin(), parent.end(), -1);
        queue<int> q;
        q.push(s);
        parent[s] = s;

        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (int i = 0; i < adj[u].size(); ++i) {
                Edge& e = adj[u][i];
                if (parent[e.to] == -1 && e.cap > 0) {
                    parent[e.to] = u;
                    edgeIndex[e.to] = i;
                    if (e.to == t) return true;
                    q.push(e.to);
                }
            }
        }
        return false;
    }

    int maxFlow(int s, int t) {
        int flow = 0;
        vector<int> parent(V), edgeIndex(V);

        while (bfs(s, t, parent, edgeIndex)) {
            int pathFlow = INF;
            for (int v = t; v != s; v = parent[v]) {
                int u = parent[v];
                Edge& e = adj[u][edgeIndex[v]];
                pathFlow = min(pathFlow, e.cap);
            }

            for (int v = t; v != s; v = parent[v]) {
                int u = parent[v];
                Edge& e = adj[u][edgeIndex[v]];
                e.cap -= pathFlow;
                adj[v][e.rev].cap += pathFlow;
            }

            flow += pathFlow;
        }

        return flow;
    }
};
```

# 🧠 Edmonds–Karp Algorithm — Max Flow via BFS (O(V·E²))

---

## 📜 Origin & Motivation

The **Edmonds–Karp algorithm** is a refinement of the **Ford–Fulkerson method**, introduced by **Jack Edmonds** and **Richard Karp** in 1972.  
It solves the **maximum flow problem** in a directed graph with capacities, using **BFS** to find shortest augmenting paths.

Unlike naive DFS-based approaches, Edmonds–Karp guarantees **polynomial time** and avoids infinite loops on cyclic graphs.

---

## 🧩 Use Cases

- 🔗 Network flow optimization  
- 🧬 Data routing and bandwidth allocation  
- 🏭 Supply chain and logistics modeling  
- 🎮 Competitive programming: clean max-flow engine

---

## 🔁 When to Use Edmonds–Karp vs Alternatives

| Scenario                          | Edmonds–Karp | Ford–Fulkerson | Dinic’s | Push-Relabel |
|----------------------------------|--------------|----------------|--------|--------------|
| Small graphs (V ≤ 1000)          | ✅           | ✅              | ✅      | ✅            |
| Large graphs (V ≥ 10⁴)           | ❌           | ❌              | ✅      | ✅            |
| Weighted edges                   | ✅ (capacities only) | ✅      | ✅      | ✅            |
| Online queries                   | ❌           | ❌              | ❌      | ❌            |
| Deterministic performance        | ✅           | ❌              | ✅      | ✅            |

---

## 🧱 Core Architecture

### 🎯 Triggers

| Condition                        | Action in Code           |
|----------------------------------|--------------------------|
| Residual capacity > 0           | Include edge in BFS      |
| BFS finds path from s to t      | Augment flow             |
| No path found                   | Terminate algorithm      |

---

### 🔧 Algorithm Steps

1. Initialize residual graph with forward and reverse edges  
2. While a path exists from `s` to `t`:
   - Use **BFS** to find shortest augmenting path  
   - Track parent and edge index for backtracking  
   - Compute **minimum residual capacity** along the path  
   - Update residual capacities (forward and reverse)  
   - Add path flow to total flow

---

## 🚀 C# Implementation

```csharp
public class EdmondsKarp {
    private int V;
    private List<(int to, int rev, int cap)>[] adj;
    private const int INF = int.MaxValue;

    public EdmondsKarp(int vertexCount) {
        V = vertexCount;
        adj = new List<(int, int, int)>[V];
        for (int i = 0; i < V; i++) adj[i] = new List<(int, int, int)>();
    }

    public void AddEdge(int u, int v, int cap) {
        adj[u].Add((v, adj[v].Count, cap));
        adj[v].Add((u, adj[u].Count - 1, 0)); // reverse edge
    }

    private bool BFS(int s, int t, int[] parent, int[] edgeIndex) {
        Array.Fill(parent, -1);
        Queue<int> q = new();
        q.Enqueue(s);
        parent[s] = s;

        while (q.Count > 0) {
            int u = q.Dequeue();
            for (int i = 0; i < adj[u].Count; i++) {
                var (v, _, cap) = adj[u][i];
                if (parent[v] == -1 && cap > 0) {
                    parent[v] = u;
                    edgeIndex[v] = i;
                    if (v == t) return true;
                    q.Enqueue(v);
                }
            }
        }
        return false;
    }

    public int MaxFlow(int s, int t) {
        int flow = 0;
        int[] parent = new int[V];
        int[] edgeIndex = new int[V];

        while (BFS(s, t, parent, edgeIndex)) {
            int pathFlow = INF;
            for (int v = t; v != s; v = parent[v]) {
                int u = parent[v];
                var (to, rev, cap) = adj[u][edgeIndex[v]];
                pathFlow = Math.Min(pathFlow, cap);
            }

            for (int v = t; v != s; v = parent[v]) {
                int u = parent[v];
                var (to, rev, cap) = adj[u][edgeIndex[v]];
                adj[u][edgeIndex[v]] = (to, rev, cap - pathFlow);
                var (backTo, backRev, backCap) = adj[v][rev];
                adj[v][rev] = (backTo, backRev, backCap + pathFlow);
            }

            flow += pathFlow;
        }

        return flow;
    }
}
```

⏱️ Complexity Analysis  
BFS per phase: O(E) — each search explores all residual edges  
Augmenting path count: O(V·E) — bounded by number of phases and edge saturation  
Total time complexity: O(V·E²) — worst-case for dense graphs with small capacities  
Space complexity: O(V + E) — adjacency list, residual graph, and tracking arrays

⚠️ Pitfalls  
🔁 Residual graph must maintain reverse edges for correct flow updates  
🧹 Parent and edge index arrays must be reset before each BFS phase  
⚠️ Forgetting to update both forward and reverse edges leads to incorrect flow  
🧩 Graphs with cycles or multiple parallel edges require careful edge indexing  
🧠 BFS must track shortest paths — DFS-based variants may loop or degrade performance

✅ Conclusion  
Edmonds–Karp is a deterministic BFS-based engine for computing max flow:

🔁 Guarantees polynomial time even on cyclic graphs  
📊 Uses shortest-path layering to avoid redundant augmentations  
🧠 Ideal for medium-sized networks where correctness and clarity matter  
🏆 A foundational algorithm in flow theory, often used as a benchmark

👉 Key takeaway: Edmonds–Karp transforms the Ford–Fulkerson method into a predictable, layer-driven flow engine — clean, scalable, and reliable for unweighted capacity networks.


---
