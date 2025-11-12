# 🧠 Floyd–Warshall — All-Pairs Shortest Path 

## 📜 Origin & Motivation
The Floyd–Warshall algorithm is a dynamic programming method for finding shortest paths between **all pairs of vertices** in a weighted graph.  
It generalizes Dijkstra’s single-source shortest path to a full matrix solution.  

- Origin: Robert Floyd (1962), based on Stephen Warshall’s work (1962).  
- Motivation: Provide a **simple, matrix-based algorithm** that works for dense graphs and supports negative edge weights (but not negative cycles).

---

## 🧩 Where It’s Used
- 🔍 All-pairs shortest paths in dense graphs  
- 📊 Network routing (minimal latency between nodes)  
- 🧬 Bioinformatics (sequence alignment graphs)  
- 📚 Graph theory teaching (classic DP example)  
- 🎮 Game AI (precomputing shortest paths between map locations)  
- 🔁 Transitive closure (Warshall’s variant for reachability)  

---

## 🔁 When to Use vs Alternatives

| Task / Scenario              | Floyd–Warshall | Dijkstra (with PQ) | Bellman–Ford |
|-------------------------------|----------------|--------------------|--------------|
| All-pairs shortest paths      | ✅              | ❌ (needs n runs)  | ❌            |
| Single-source shortest path   | ❌              | ✅                  | ✅            |
| Handles negative weights      | ✅              | ❌                  | ✅            |
| Detects negative cycles       | ✅              | ❌                  | ✅            |
| Sparse graphs efficiency      | ❌              | ✅                  | ❌            |
| Simplicity of implementation  | ✅              | ❌                  | ✅            |

---

## 🧱 Core Idea
Dynamic programming on adjacency matrix:

- Let `D[i][j]` = shortest path from `i` to `j`.  
- For each intermediate vertex `k`, update:  
```D[i][j] = min(D[i][j], D[i][k] + D[k][j])```

- Base case: `D[i][j]` = direct edge weight (∞ if no edge, 0 if i=j).  
- After n iterations, `D` contains all shortest path distances.

---

## 🚀 Implementation (C++)

```cpp
#include <vector>
#include <algorithm>
using namespace std;

const int INF = 1e9;

vector<vector<int>> floydWarshall(vector<vector<int>>& graph) {
    int n = graph.size();
    auto dist = graph;  // Copy graph to dist

    // Initialize
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            if (i == j) dist[i][j] = 0;
            else if (dist[i][j] == 0) dist[i][j] = INF;  // No edge
        }
    }

    // DP: Relax via each intermediate
    for (int k = 0; k < n; k++) {
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                if (dist[i][k] < INF && dist[k][j] < INF) {
                    dist[i][j] = min(dist[i][j], dist[i][k] + dist[k][j]);
                }
            }
        }
    }

    // Optional: Detect negative cycles
    for (int i = 0; i < n; i++) {
        if (dist[i][i] < 0) {
            // Negative cycle exists
        }
    }

    return dist;
}
```
---

## 🚀 Implementation (C#)
```cpp
public static int[,] FloydWarshall(int[,] graph, int n) {
    int[,] dist = (int[,])graph.Clone();

    const int INF = int.MaxValue / 2;  // Avoid overflow

    // Initialize
    for (int i = 0; i < n; i++) {
        for (int j = 0; j < n; j++) {
            if (i == j) dist[i, j] = 0;
            else if (graph[i, j] == 0) dist[i, j] = INF;
        }
    }

    // DP
    for (int k = 0; k < n; k++) {
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                if (dist[i, k] < INF && dist[k, j] < INF) {
                    long sum = (long)dist[i, k] + dist[k, j];
                    if (sum < dist[i, j]) {
                        dist[i, j] = (int)sum;
                    }
                }
            }
        }
    }

    // Optional: Negative cycle check
    for (int i = 0; i < n; i++) {
        if (dist[i, i] < 0) {
            // Negative cycle
        }
    }

    return dist;
}
```


# Floyd–Warshall — Complexity & Architectural Notes

## ⏱️ Complexity Analysis
- **Time:** O(n³) — triple nested loop over vertices  
- **Space:** O(n²) — adjacency matrix of distances  
- **Best suited for:** Dense graphs with up to a few thousand vertices  

---

## ⚠️ Pitfalls
- **Negative cycles:** if `D[i][i] < 0` after execution, a negative cycle exists  
- **Sparse graphs:** inefficient compared to Dijkstra’s algorithm  
- **Initialization:** must set `dist[i][j] = ∞` if no edge, and `dist[i][i] = 0`  
- **Overflow risk:** use large sentinel values (`INT_MAX`) carefully  
- **Path reconstruction:** requires an extra predecessor matrix  

---

## ✅ Conclusion
Floyd–Warshall is a **classic DP algorithm** for all-pairs shortest paths:

- ⚡ Elegant matrix recurrence  
- 📊 Detects negative cycles  
- 🔗 Works with negative weights  
- 🛡️ Deterministic and easy to implement  

👉 **Key takeaway:**  
Floyd–Warshall is the go-to algorithm when you need **all-pairs shortest paths** in dense graphs, with support for negative weights and cycle detection.  
It balances **simplicity, clarity, and mathematical elegance**, making it a cornerstone of graph algorithms.

---




