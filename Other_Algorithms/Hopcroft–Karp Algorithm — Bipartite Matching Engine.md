# 🧠 Hopcroft–Karp Algorithm — Bipartite Matching Engine

## 📜 Origin & Motivation

The **Hopcroft–Karp algorithm** was introduced in **1973** by **John Hopcroft** and **Richard Karp**, and independently by **Alexander Karzanov**.  
It was designed to solve the **maximum-cardinality matching problem** in bipartite graphs more efficiently than previous approaches like Ford–Fulkerson and the Hungarian algorithm.

When solving **maximum bipartite matching**, naive DFS-based methods run in  
**O(n · m)** — too slow for large graphs.  
Hopcroft–Karp is a **batch-based offline engine** that improves performance by alternating between:

- **BFS phase**: builds a layered graph of shortest augmenting paths  
- **DFS phase**: finds multiple disjoint augmenting paths in parallel  

This leads to a total complexity of **O(√n · m)** — optimal for unweighted bipartite graphs.


---

## 🧩 Use Cases

- 🔗 Maximum matching in bipartite graphs  
- 🧬 Assigning tasks to workers  
- 🏫 Student-to-project allocation  
- 🎮 Competitive programming: fast matching without weights

---

## 🔁 When to Use Hopcroft–Karp vs Alternatives

| Scenario                          | Hopcroft–Karp | DFS Matching | Hungarian |
|----------------------------------|----------------|--------------|-----------|
| Maximum matching (unweighted)    | ✅             | ❌ (slow)     | ❌        |
| Weighted matching                | ❌             | ❌           | ✅        |
| Small graphs (n ≤ 100)           | ✅             | ✅           | ✅        |
| Large bipartite graphs (n ≥ 10⁴) | ✅             | ❌           | ❌        |
| Online queries                   | ❌             | ✅           | ❌        |

---

## 🧱 Core Architecture

### 🎯 Triggers

| Condition                        | Action in Code           |
|----------------------------------|--------------------------|
| Node is unmatched in U          | Start BFS from it        |
| BFS finds layered path to V     | Proceed to DFS phase     |
| DFS finds augmenting path       | Update matching pairs    |
| No path found                   | Terminate algorithm      |

---

### 🔧 Algorithm Steps

1. Initialize matching arrays for both sides: `pairU`, `pairV`  
2. BFS phase:  
   - Build level graph from unmatched nodes in U  
   - Track distances to guide DFS  
3. DFS phase:  
   - Find augmenting paths using level info  
   - Update matches for each successful path  
4. Repeat until no more augmenting paths exist

---

## 🚀 C++ Implementation

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

class HopcroftKarp {
public:
    int U, V;
    vector<vector<int>> adj;
    vector<int> pairU, pairV, dist;

    HopcroftKarp(int uSize, int vSize) : U(uSize), V(vSize) {
        adj.resize(U);
        pairU.assign(U, -1);
        pairV.assign(V, -1);
        dist.resize(U);
    }

    void addEdge(int u, int v) {
        adj[u].push_back(v);
    }

    bool bfs() {
        queue<int> q;
        for (int u = 0; u < U; ++u) {
            dist[u] = (pairU[u] == -1) ? 0 : INF;
            if (pairU[u] == -1) q.push(u);
        }

        bool found = false;
        while (!q.empty()) {
            int u = q.front(); q.pop();
            for (int v : adj[u]) {
                int pu = pairV[v];
                if (pu == -1) {
                    found = true;
                } else if (dist[pu] == INF) {
                    dist[pu] = dist[u] + 1;
                    q.push(pu);
                }
            }
        }
        return found;
    }

    bool dfs(int u) {
        for (int v : adj[u]) {
            int pu = pairV[v];
            if (pu == -1 || (dist[pu] == dist[u] + 1 && dfs(pu))) {
                pairU[u] = v;
                pairV[v] = u;
                return true;
            }
        }
        dist[u] = INF;
        return false;
    }

    int maxMatching() {
        int match = 0;
        while (bfs()) {
            for (int u = 0; u < U; ++u) {
                if (pairU[u] == -1 && dfs(u)) {
                    match++;
                }
            }
        }
        return match;
    }
};

```
---

## 🔄 Language Comparison

The following two implementations are equivalent in logic and structure:

- **C++ version** uses STL containers and procedural style  
- **C# version** uses generic collections and object-oriented encapsulation

Both maintain:
- `pairU`, `pairV` — current matchings  
- `dist` — BFS level tracking  
- `adj` — adjacency list for the bipartite graph

Each version performs:
- BFS to build layered paths  
- DFS to find and apply augmenting paths  
- Iterative matching until no augmenting paths remain


## 🚀 C# Implementation
```cpp
public class HopcroftKarp {
    private int U, V;
    private List<int>[] adj;
    private int[] pairU, pairV, dist;
    private const int INF = int.MaxValue;

    public HopcroftKarp(int leftSize, int rightSize) {
        U = leftSize;
        V = rightSize;
        adj = new List<int>[U];
        for (int i = 0; i < U; i++) adj[i] = new List<int>();
        pairU = new int[U];
        pairV = new int[V];
        dist = new int[U];
        Array.Fill(pairU, -1);
        Array.Fill(pairV, -1);
    }

    public void AddEdge(int u, int v) {
        adj[u].Add(v);
    }

    private bool BFS() {
        Queue<int> q = new();
        for (int u = 0; u < U; u++) {
            if (pairU[u] == -1) {
                dist[u] = 0;
                q.Enqueue(u);
            } else {
                dist[u] = INF;
            }
        }

        bool found = false;
        while (q.Count > 0) {
            int u = q.Dequeue();
            foreach (int v in adj[u]) {
                int pu = pairV[v];
                if (pu == -1) {
                    found = true;
                } else if (dist[pu] == INF) {
                    dist[pu] = dist[u] + 1;
                    q.Enqueue(pu);
                }
            }
        }
        return found;
    }

    private bool DFS(int u) {
        foreach (int v in adj[u]) {
            int pu = pairV[v];
            if (pu == -1 || (dist[pu] == dist[u] + 1 && DFS(pu))) {
                pairU[u] = v;
                pairV[v] = u;
                return true;
            }
        }
        dist[u] = INF;
        return false;
    }

    public int MaxMatching() {
        int matching = 0;
        while (BFS()) {
            for (int u = 0; u < U; u++) {
                if (pairU[u] == -1 && DFS(u)) {
                    matching++;
                }
            }
        }
        return matching;
    }
}
```

---

## ⏱️ Complexity Analysis

**Time Complexity:**  
- **O(√n · m)**  
  - Each BFS builds a layered graph of shortest augmenting paths  
  - Each DFS finds multiple disjoint paths in parallel  
  - Total number of phases is bounded by O(√n), each phase may process multiple augmenting paths

**Space Complexity:**  
- **O(n + m)**  
  - Matching arrays: `pairU`, `pairV`  
  - Adjacency list for the graph  
  - Distance array used during BFS

---

## ⚠️ Pitfalls

- ❌ Only works for **unweighted bipartite graphs**  
- ❌ Cannot handle **online queries** — all edges must be known in advance  
- ⚠️ Must carefully **reset `dist` and `pair` arrays** between phases  
- ⚠️ Graph must be bipartite and edges must connect nodes from U to V only

---

## ✅ Conclusion

Hopcroft–Karp is a **batch-based matching engine** for bipartite graphs:

- ⚡ Fast and scalable for large inputs  
- 🧱 Simple once BFS/DFS phases are understood  
- 🔗 Ideal for assignment, pairing, and resource allocation problems

👉 **Key takeaway:**  
If you need **maximum matching in a bipartite graph**, and weights don’t matter — Hopcroft–Karp is your go-to engine.  
It’s fast, deterministic, and easy to integrate into system-level architectures.


---

