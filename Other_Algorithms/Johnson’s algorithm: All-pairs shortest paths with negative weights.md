# Johnson’s Algorithm: All-pairs Shortest Paths with Negative Weights

---

##  Origin and Motivation
- **Origin:** Johnson’s algorithm combines **Bellman–Ford** potentials with **Dijkstra** to efficiently handle negative edges in sparse graphs.  
- **Motivation:**  
  - Floyd–Warshall is simple but runs in `O(n^3)` time.  
  - Dijkstra is fast but fails with negative edges.  
  - Johnson reweights edges to nonnegative values and runs Dijkstra `n` times, achieving near-linear APSP on sparse graphs.  
- **Key Insight:**  
  - Add a super-source and run Bellman–Ford to compute vertex potentials `h[v]`.  
  - Reweight edges: `w'(u,v) = w(u,v) + h[u] − h[v]`.  
  - All `w'` are nonnegative, preserving shortest-path order.  

---

##  Where It’s Used

| Domain               | Use Case                                               |
|----------------------|--------------------------------------------------------|
| Routing and networks | APSP on large sparse graphs with possible negative edges |
| Optimization on graphs | Distance-based DP and cost propagation                 |
| Systems analysis     | Latency and dependency graphs with penalties           |
| Competitive programming | APSP with negative edges but no negative cycles        |

---

##  When to Use vs Alternatives

| Task                          | Johnson | Floyd–Warshall | n×Dijkstra | Bellman–Ford APSP |
|-------------------------------|---------|----------------|------------|-------------------|
| APSP with negative edges, sparse | ✅ Yes | Works but `O(n^3)` | ❌ Needs nonnegative | ❌ Too slow (`O(n·m·n)`) |
| APSP with dense graphs        | ⚠️ Sometimes | ✅ Best simple choice | ❌ No | ❌ No |
| Negative cycle detection      | ✅ Yes (via Bellman–Ford) | ✅ Yes | ❌ No | ✅ Yes |
| Weighted nonnegative edges    | ✅ Yes, but n×Dijkstra may suffice | ⚠️ Overkill | ✅ Yes | ❌ Not needed |

---

##  Core Idea
- **Potentials via Bellman–Ford:**  
  - Add super-source `s*` connected to all vertices with zero-weight edges.  
  - Run Bellman–Ford to compute `h[v] = dist(s*, v)`.  
  - If a negative cycle is detected, APSP is undefined.  

- **Reweighting:**  
  - Define new weights:  
    

```
    w'(u,v) = w(u,v) + h[u] - h[v]
   ```

  
  - All `w'` are nonnegative.  
  - Preserves shortest paths:  
    

```
dist'(u,v) = dist(u,v) + h[u] - h[v]
 ```
- **Per-source Dijkstra:**  
  - For each source `u`, run Dijkstra on `w'` to get `dist'(u,v)`.  
  - Recover original distances:  
    

    ```
    dist(u,v) = dist'(u,v) - h[u] + h[v]
    ```

  

---

##  Key Takeaway
Johnson’s algorithm bridges the gap between **Dijkstra’s speed** and **Floyd–Warshall’s generality**, 
enabling efficient **all-pairs shortest paths** on sparse graphs with negative edges but no negative cycles.  



## Implementation (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge { int u, v; long long w; };

const long long INF = (1LL<<60);

bool bellmanFord(int n, const vector<Edge>& edges, vector<long long>& h) {
    h.assign(n, 0); // super-source with 0 edges to all nodes
    for (int i = 0; i < n - 1; i++) {
        bool any = false;
        for (auto &e : edges) {
            if (h[e.u] + e.w < h[e.v]) {
                h[e.v] = h[e.u] + e.w;
                any = true;
            }
        }
        if (!any) break;
    }
    // Negative cycle check
    for (auto &e : edges) {
        if (h[e.u] + e.w < h[e.v]) return false;
    }
    return true;
}

vector<long long> dijkstra(int n, const vector<vector<pair<int,long long>>>& adj, int s) {
    vector<long long> dist(n, INF);
    dist[s] = 0;
    priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<pair<long long,int>>> pq;
    pq.push({0, s});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d != dist[u]) continue;
        for (auto &e : adj[u]) {
            int v = e.first; long long w = e.second;
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}

// Johnson's Algorithm for APSP
// n: number of vertices, edges: list of (u,v,w) with 0..n-1 indexing
// Returns matrix dist[u][v] or empty on negative cycle
vector<vector<long long>> johnson(int n, const vector<Edge>& edges) {
    // 1) Compute potentials h via Bellman–Ford on original edges plus super-source (implicit)
    vector<long long> h;
    if (!bellmanFord(n, edges, h)) {
        return {}; // negative cycle detected
    }

    // 2) Build reweighted adjacency
    vector<vector<pair<int,long long>>> adj(n);
    adj.assign(n, {});
    for (auto &e : edges) {
        long long rw = e.w + h[e.u] - h[e.v];
        // rw must be >= 0; guard against tiny negatives due to precision (integers here)
        adj[e.u].push_back({e.v, rw});
    }

    // 3) Run Dijkstra from each node on reweighted graph
    vector<vector<long long>> dist(n, vector<long long>(n, INF));
    for (int u = 0; u < n; u++) {
        auto d = dijkstra(n, adj, u);
        for (int v = 0; v < n; v++) {
            if (d[v] < INF) {
                dist[u][v] = d[v] - h[u] + h[v];
            }
        }
    }
    return dist;
}

int main() {
    int n = 5;
    vector<Edge> edges = {
        {0,1,3}, {0,2,8}, {0,4,-4},
        {1,3,1}, {1,4,7},
        {2,1,4},
        {3,2,-5}, {3,0,2},
        {4,3,6}
    };

    auto dist = johnson(n, edges);
    if (dist.empty()) {
        cout << "Negative cycle detected\n";
    } else {
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                if (dist[i][j] >= INF/2) cout << "INF ";
                else cout << dist[i][j] << " ";
            }
            cout << "\n";
        }
    }
    return 0;
}
```



## Implementation (C#)
```csharp
using System;
using System.Collections.Generic;

public struct Edge {
    public int U, V;
    public long W;
    public Edge(int u, int v, long w) { U = u; V = v; W = w; }
}

public class Johnson {
    const long INF = (1L << 60);

    private static bool BellmanFord(int n, List<Edge> edges, out long[] h) {
        h = new long[n];
        // h initialized to 0 simulates super-source edges of weight 0 to every node
        for (int i = 0; i < n - 1; i++) {
            bool any = false;
            foreach (var e in edges) {
                if (h[e.U] + e.W < h[e.V]) {
                    h[e.V] = h[e.U] + e.W;
                    any = true;
                }
            }
            if (!any) break;
        }
        foreach (var e in edges) {
            if (h[e.U] + e.W < h[e.V]) return false;
        }
        return true;
    }

    private static long[] Dijkstra(int n, List<(int to, long w)>[] adj, int s) {
        var dist = new long[n];
        for (int i = 0; i < n; i++) dist[i] = INF;
        dist[s] = 0;
        var pq = new PriorityQueue<(int v, long d), long>();
        pq.Enqueue((s, 0), 0);
        while (pq.Count > 0) {
            var cur = pq.Dequeue();
            int u = cur.v; long d = cur.d;
            if (d != dist[u]) continue;
            foreach (var e in adj[u]) {
                int v = e.to; long w = e.w;
                if (dist[u] + w < dist[v]) {
                    dist[v] = dist[u] + w;
                    pq.Enqueue((v, dist[v]), dist[v]);
                }
            }
        }
        return dist;
    }

    // Returns dist[u][v] matrix or null if negative cycle
    public static long[,] Run(int n, List<Edge> edges) {
        if (!BellmanFord(n, edges, out var h)) return null;

        var adj = new List<(int to, long w)>[n];
        for (int i = 0; i < n; i++) adj[i] = new List<(int, long)>();
        foreach (var e in edges) {
            long rw = e.W + h[e.U] - h[e.V];
            adj[e.U].Add((e.V, rw));
        }

        var dist = new long[n, n];
        for (int i = 0; i < n; i++) {
            var d = Dijkstra(n, adj, i);
            for (int j = 0; j < n; j++) {
                dist[i, j] = d[j] >= INF / 2 ? INF : d[j] - h[i] + h[j];
            }
        }
        return dist;
    }
}

class Program {
    static void Main() {
        int n = 5;
        var edges = new List<Edge> {
            new Edge(0,1,3), new Edge(0,2,8), new Edge(0,4,-4),
            new Edge(1,3,1), new Edge(1,4,7),
            new Edge(2,1,4),
            new Edge(3,2,-5), new Edge(3,0,2),
            new Edge(4,3,6)
        };

        var dist = Johnson.Run(n, edges);
        if (dist == null) {
            Console.WriteLine("Negative cycle detected");
        } else {
            for (int i = 0; i < n; i++) {
                for (int j = 0; j < n; j++) {
                    if (dist[i, j] >= (1L << 59)) Console.Write("INF ");
                    else Console.Write(dist[i, j] + " ");
                }
                Console.WriteLine();
            }
        }
    }
}
```

#  Complexity Analysis — Johnson’s Algorithm

---

## Build Phase
- **Potentials via Bellman–Ford:** `O(n·m)`  
  - Runs once to compute vertex potentials and detect negative cycles.  

## Per-Source Dijkstra
- **On reweighted graph:** `O(n·(m log n))` with binary heap.  
- **Typical bound:** `O(n·(m + n log n))`.  

## Total Complexity
- `O(n·m + n·m log n)` → simplifies to `O(n·m log n)` on sparse graphs.  
- **Comparison:** Beats `O(n^3)` of Floyd–Warshall when `m ≪ n^2`.  

## Space Complexity
- `O(n + m)` for graph storage.  
- `O(n^2)` for APSP distance matrix.  

---

#  Impact of Design Choices

| Choice                | Effect                                                                 |
|-----------------------|------------------------------------------------------------------------|
| Graph density         | Sparse graphs benefit most; for dense graphs, Floyd–Warshall may be simpler. |
| Priority queue        | Binary heap is sufficient; pairing/Fibonacci heaps improve theory but rarely practice. |
| Edge weights type     | Use 64-bit integers to avoid overflow during reweighting and summations. |
| Negative cycles       | Mandatory check via Bellman–Ford before reweighting; APSP undefined if present. |
| Directed vs undirected| Johnson is typically applied to directed graphs; undirected works as directed with symmetric edges. |

---

#  Pitfalls

- **Skipping negative cycle check:** Leads to invalid potentials and wrong results.  
- **Wrong reweighting formula:** Must be `w'(u,v) = w(u,v) + h[u] − h[v]`; any deviation breaks correctness.  
- **Integer overflow:** Reweighting can increase values; always use 64-bit.  
- **Disconnected graphs:** Distances may remain `INF`; handle output and initialization properly.  
- **Priority queue mismatch:** Running Dijkstra directly on negative edges without reweighting yields incorrect results.  

---

#  Conclusion

- **What it gives:** Near-linear APSP for sparse graphs with negative edges but no negative cycles, by combining Bellman–Ford potentials with Dijkstra.  
- **Why it matters:** Bridges the gap between Dijkstra’s speed and Floyd–Warshall’s generality, delivering practical performance on large sparse graphs.  
- **Key takeaway:**  
  - Compute potentials with Bellman–Ford.  
  - Reweight edges to nonnegative.  
  - Run Dijkstra from each node.  
  - Convert back with potentials to get exact APSP.  



---
