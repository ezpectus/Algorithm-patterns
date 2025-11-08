# Min-Cost Flow — The King of Flow Optimization  
*O(F × n log n) or O(n²m) — Flow with Cost*

---

## Origin & Motivation  
**Min-Cost Flow (MCF)** — extension of **Maximum Flow** where **each edge has a cost** and we want to push **maximum flow at minimum total cost**.

**Goal**:  
> **Push flow from source to sink → minimize total cost**

**Classic problems**:  
- **Transportation** (min cost to ship goods)  
- **Assignment** (min cost bipartite matching)  
- **Network optimization**

---

## Where It’s Used  

| Domain | Use Case |
|-------|----------|
| **Competitive Programming** | Codeforces, AtCoder (MCF tasks) |
| **Logistics & Transport** | Shipping, routing |
| **Bipartite Matching** | Min-cost assignment |
| **Image Segmentation** | Graph cuts |
| **Economics** | Resource allocation |

---

## When to Use Min-Cost Flow  

| Requirement | MCF | Max Flow | Dijkstra |
|------------|-----|----------|----------|
| **Flow + Cost** | Yes | No | No |
| **Negative costs** | Yes (with potentials) | No | No |
| **Max flow needed** | Yes | Yes | No |
| **Only shortest path** | No | No | Yes |
| **Bipartite matching** | Yes | Yes (Hopcroft-Karp) | Yes (KM) |

> **Use MCF when**:  
> - You have **capacities + costs**  
> - You need **max flow at min cost**  
> - You allow **negative costs** (with care)

---

## Core Idea — Step by Step  

### **1. Model as Flow Network**  
```text
(s) → nodes → (t)
cap(u,v), cost(u,v)
```

### 2. Successive Shortest Path (SPFA / Dijkstra + Potentials)
```
while (can push flow):
    find cheapest path from s to t in residual graph
    push min capacity along path
    add cost × flow
```

### 3. Use Potentials to Handle Negative Costs
```
reduced_cost(u,v) = cost(u,v) + h[u] - h[v]
```

### 4. Update Potentials After Each Augmentation
```
h[v] += dist[v]
```

## Full Implementation (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

using ll = long long;
const ll INF = 1e18;

struct MinCostFlow {
    struct Edge {
        int to, rev;
        ll cap, cost;
    };

    int n;
    vector<vector<Edge>> g;
    vector<ll> h, dist;
    vector<int> prevv, preve;

    MinCostFlow(int _n) : n(_n), g(_n), h(_n), dist(_n), prevv(_n), preve(_n) {}

    void add_edge(int from, int to, ll cap, ll cost) {
        g[from].push_back({to, (int)g[to].size(), cap, cost});
        g[to].push_back({from, (int)g[from].size() - 1, 0, -cost});
    }

    ll min_cost_flow(int s, int t, ll f = INF) {
        ll res = 0;
        fill(h.begin(), h.end(), 0LL);

        while (f > 0) {
            // Dijkstra with potentials
            priority_queue<pair<ll,int>, vector<pair<ll,int>>, greater<>> pq;
            fill(dist.begin(), dist.end(), INF);
            dist[s] = 0;
            pq.emplace(0, s);

            while (!pq.empty()) {
                auto [cost, v] = pq.top(); pq.pop();
                if (dist[v] < cost) continue;
                for (int i = 0; i < g[v].size(); ++i) {
                    Edge &e = g[v][i];
                    if (e.cap > 0 && dist[e.to] > dist[v] + e.cost + h[v] - h[e.to]) {
                        dist[e.to] = dist[v] + e.cost + h[v] - h[e.to];
                        prevv[e.to] = v;
                        preve[e.to] = i;
                        pq.emplace(dist[e.to], e.to);
                    }
                }
            }

            if (dist[t] == INF) {
                return -1;  // cannot send more flow
            }

            // Update potentials
            for (int v = 0; v < n; ++v) {
                if (dist[v] < INF) h[v] += dist[v];
            }

            // Find min capacity along path
            ll d = f;
            for (int v = t; v != s; v = prevv[v]) {
                d = min(d, g[prevv[v]][preve[v]].cap);
            }

            f -= d;
            res += d * h[t];  // h[t] = total reduced cost

            // Update residual graph
            for (int v = t; v != s; v = prevv[v]) {
                Edge &e = g[prevv[v]][preve[v]];
                e.cap -= d;
                g[v][e.rev].cap += d;
            }
        }
        return res;
    }
};

// === USAGE EXAMPLE ===
int main() {
    int n = 4, s = 0, t = 3;
    MinCostFlow mcf(n);

    mcf.add_edge(0, 1, 10, 2);
    mcf.add_edge(0, 2, 5,  1);
    mcf.add_edge(1, 3, 10, 3);
    mcf.add_edge(2, 3, 5,  4);

    ll cost = mcf.min_cost_flow(s, t, 15);
    cout << "Min cost: " << cost << endl;  // 75

    return 0;
}
```

## Full Implementation (C#)
```cpp
using System;
using System.Collections.Generic;

public class MinCostFlow
{
    private class Edge
    {
        public int to, rev;
        public long cap, cost;
        public Edge(int to, int rev, long cap, long cost)
        {
            this.to = to; this.rev = rev; this.cap = cap; this.cost = cost;
        }
    }

    private int n;
    private List<Edge>[] g;
    private long[] h, dist;
    private int[] prevv, preve;
    private const long INF = 1L << 60;

    public MinCostFlow(int n)
    {
        this.n = n;
        g = new List<Edge>[n];
        for (int i = 0; i < n; i++) g[i] = new List<Edge>();
        h = new long[n]; dist = new long[n];
        prevv = new int[n]; preve = new int[n];
    }

    public void AddEdge(int from, int to, long cap, long cost)
    {
        g[from].Add(new Edge(to, g[to].Count, cap, cost));
        g[to].Add(new Edge(from, g[from].Count - 1, 0, -cost));
    }

    public long MinCostFlow(int s, int t, long f = INF)
    {
        long res = 0;
        Array.Fill(h, 0L);

        while (f > 0)
        {
            var pq = new PriorityQueue<(long, int)>();
            Array.Fill(dist, INF);
            dist[s] = 0;
            pq.Enqueue((0, s));

            while (pq.Count > 0)
            {
                var (cost, v) = pq.Dequeue();
                if (dist[v] < cost) continue;
                for (int i = 0; i < g[v].Count; i++)
                {
                    var e = g[v][i];
                    if (e.cap > 0 && dist[e.to] > dist[v] + e.cost + h[v] - h[e.to])
                    {
                        dist[e.to] = dist[v] + e.cost + h[v] - h[e.to];
                        prevv[e.to] = v;
                        preve[e.to] = i;
                        pq.Enqueue((dist[e.to], e.to));
                    }
                }
            }

            if (dist[t] == INF) return -1;

            for (int v = 0; v < n; v++)
                if (dist[v] < INF) h[v] += dist[v];

            long d = f;
            for (int v = t; v != s; v = prevv[v])
                d = Math.Min(d, g[prevv[v]][preve[v]].cap);

            f -= d;
            res += d * h[t];

            for (int v = t; v != s; v = prevv[v])
            {
                var e = g[prevv[v]][preve[v]];
                e.cap -= d;
                g[v][e.rev].cap += d;
            }
        }
        return res;
    }
}

// Priority Queue Helper
public class PriorityQueue<T>
{
    private List<T> heap = new List<T>();
    private Comparison<T> comparison;
    public int Count => heap.Count;

    public PriorityQueue(Comparison<T> comparison = null)
    {
        this.comparison = comparison ?? Comparer<T>.Default.Compare;
    }

    public void Enqueue(T item)
    {
        heap.Add(item);
        int i = heap.Count - 1;
        while (i > 0)
        {
            int p = (i - 1) / 2;
            if (comparison(heap[p], item) <= 0) break;
            heap[i] = heap[p];
            i = p;
        }
        heap[i] = item;
    }

    public T Dequeue()
    {
        T ret = heap[0];
        T x = heap[heap.Count - 1];
        heap.RemoveAt(heap.Count - 1);
        int i = 0;
        while (i * 2 + 1 < heap.Count)
        {
            int a = i * 2 + 1, b = i * 2 + 2;
            int next = b < heap.Count && comparison(heap[b], heap[a]) < 0 ? b : a;
            if (comparison(heap[next], x) >= 0) break;
            heap[i] = heap[next];
            i = next;
        }
        if (heap.Count > 0) heap[i] = x;
        return ret;
    }
}

// === USAGE ===
class Program
{
    static void Main()
    {
        var mcf = new MinCostFlow(4);
        mcf.AddEdge(0, 1, 10, 2);
        mcf.AddEdge(0, 2, 5, 1);
        mcf.AddEdge(1, 3, 10, 3);
        mcf.AddEdge(2, 3, 5, 4);

        long cost = mcf.MinCostFlow(0, 3, 15);
        Console.WriteLine($"Min cost: {cost}");  // 75
    }
}
```

## Complexity Analysis  

| Metric | **Value** | **Notes** |
|--------|-----------|---------|
| **Time** | **O(F × n log n)** | **Dijkstra + potentials** — fast in practice |
| **Time (SPFA)** | **O(F × m)** | Slower, can degrade on dense graphs |
| **Space** | **O(n + m)** | Graph + auxiliary arrays |
| **Negative Costs** | Yes | **Safe with potentials** |

> **F = total flow pushed**  
> **Fast in practice** with **Dijkstra + potentials**  
> **No negative cycles allowed**

---

## Pitfalls & Fixes  

| **Issue** | **Fix / Mitigation** |
|----------|-----------------------|
| **Negative cycles** | **Ensure no negative cycle** or detect with Bellman-Ford |
| **SPFA too slow** | **Use Dijkstra + potentials** (recommended) |
| **Wrong flow limit** | Set `f = INF` or **known max flow** |
| **Integer overflow** | Use `long long` (C++) / `long` (C#) |
| **Forgot reverse edges** | **Always add reverse edge** with `cap=0`, `cost=-cost` |

---

## Conclusion  
**Min-Cost Flow — the ultimate flow optimizer**:

- **Max flow at minimum cost**  
- **Handles negative edge costs**  
- **Solves assignment, transportation, routing**  
- **Competitive programming staple**

**Key takeaway:**  
> **When flow has a price — use Min-Cost Flow.**  
> **Potentials. Dijkstra. No negative cycles. Profit.**


---
