# 🧠 Dijkstra’s Algorithm — Shortest Path Engine

## 📜 Origin & Motivation
Dijkstra’s algorithm, introduced by Edsger W. Dijkstra in 1956, is the classic solution for **single‑source shortest paths** in graphs with **non‑negative edge weights**.  
It uses a greedy strategy with a **priority queue** to always expand the closest unvisited node.  

Complexity with adjacency list + binary heap: **O(E log V)**.  
This makes it efficient for sparse graphs.

---

## 🧩 Where It’s Used
- 🚦 Routing (GPS, network packets)  
- 📊 Graph analytics (shortest paths in weighted networks)  
- 🎮 Game AI (pathfinding on weighted maps)  
- 🏗️ Infrastructure (transportation, logistics, scheduling)  

---

## 🔁 When to Use Dijkstra vs Alternatives

| Scenario                        | Use Dijkstra | Use BFS | Use Bellman–Ford | Use Floyd–Warshall |
|---------------------------------|--------------|---------|------------------|--------------------|
| Non‑negative weights            | ✅           | ❌      | ✅               | ✅                 |
| Unweighted graph                | ❌           | ✅      | ❌               | ❌                 |
| Negative weights                | ❌           | ❌      | ✅               | ❌                 |
| All‑pairs shortest paths        | ❌           | ❌      | ❌               | ✅                 |
| Sparse graph, single source     | ✅           | ❌      | ❌               | ❌                 |

---

## 🧱 Core Idea
1. Initialize distances: `dist[source] = 0`, others = ∞.  
2. Use a **priority queue (min‑heap)** keyed by distance.  
3. Extract the closest node, relax all outgoing edges.  
4. If a shorter path is found, update distance and push into the queue.  
5. Repeat until all nodes are processed.  

---

## 🚀 Implementation (C++)

```cpp
struct Edge { int to; long long w; };
using P = pair<long long,int>;

vector<long long> dijkstra(int n, int src, vector<vector<Edge>>& adj) {
    const long long INF = 1e18;
    vector<long long> dist(n, INF);
    priority_queue<P, vector<P>, greater<P>> pq;

    dist[src] = 0;
    pq.push({0, src});

    while (!pq.empty()) {
        auto [d, v] = pq.top(); pq.pop();
        if (d != dist[v]) continue;
        for (auto& e : adj[v]) {
            if (dist[e.to] > d + e.w) {
                dist[e.to] = d + e.w;
                pq.push({dist[e.to], e.to});
            }
        }
    }
    return dist;
}
```

## 🚀 Implementation (C#)
```cpp
using System;
using System.Collections.Generic;

public class Edge {
    public int To;
    public long W;
    public Edge(int to, long w) { To = to; W = w; }
}

public class Dijkstra {
    public static long[] Run(int n, int src, List<Edge>[] adj) {
        const long INF = (long)1e18;
        var dist = new long[n];
        Array.Fill(dist, INF);

        var pq = new PriorityQueue<int, long>();
        dist[src] = 0;
        pq.Enqueue(src, 0);

        while (pq.Count > 0) {
            pq.TryDequeue(out int v, out long d);
            if (d != dist[v]) continue;

            foreach (var e in adj[v]) {
                if (dist[e.To] > d + e.W) {
                    dist[e.To] = d + e.W;
                    pq.Enqueue(e.To, dist[e.To]);
                }
            }
        }
        return dist;
    }
}
```

## ⏱️ Complexity Analysis (with deeper details)

- **Preprocessing:** O(V)  
  - Initialize the distance array and data structures.  
  - This is a linear step and does not affect the asymptotic bound.  

- **Main loop:** O(E log V)  
  - Each edge can be relaxed at most once.  
  - Each distance update → insertion into the priority queue.  
  - Extracting the minimum distance from the heap costs O(log V).  
  - Total: O(E log V).  

- **Space:** O(V + E)  
  - Store adjacency list (O(V + E)).  
  - Additionally: distance array O(V), priority queue O(V).  

---

## ⚡ Why It Works Best on Sparse Graphs

- In **sparse graphs** (E ≈ V) → O(E log V) ≈ O(V log V).  
  Very efficient, especially compared to O(V²) with adjacency matrix implementations.  

- In **dense graphs** (E ≈ V²) → O(E log V) ≈ O(V² log V).  
  Here, Dijkstra is less competitive; adjacency‑matrix implementations with O(V²) can be preferable since the extra log V factor becomes costly.  

---

## 🔍 Impact of Data Structures

- **Binary heap (priority_queue):** standard choice → O(E log V).  
- **Fibonacci heap:** theoretical O(E + V log V), but slower in practice due to large constants.  
- **Pairing heap / radix heap:** in some scenarios (e.g., bounded edge weights) can give practical speedups.  

---

## ⚠️ Pitfalls 

- **Negative weights:** Algorithm strictly requires non‑negative edge weights.  
- **Priority queue duplicates:** The same node may appear multiple times in the queue.  
  - Fix: check `if (d != dist[v]) continue;` when extracting.  
- **Graph representation:**  
  - Adjacency list → O(E log V).  
  - Adjacency matrix → O(V²), simpler for dense graphs.  
- **Disconnected graph:** Nodes unreachable from the source remain with dist = ∞.  

---

## ✅ Conclusion

Dijkstra’s Algorithm is a **Shortest Path Engine**:

- ⚡ On sparse graphs it runs almost like O(V log V).  
- 📊 Optimal for problems with non‑negative weights.  
- 🔗 Integrates naturally with priority queues.  
- 🛡️ Provides deterministic guarantees.  

👉 **Key takeaway:**  
Dijkstra shines on **sparse graphs with non‑negative weights**, where it runs nearly linear in the number of edges.  
The choice of data structure (binary heap, radix heap) and graph representation (adjacency list) directly impacts practical performance.


---

