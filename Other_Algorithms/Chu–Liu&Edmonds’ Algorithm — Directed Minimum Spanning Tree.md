# Chu–Liu/Edmonds’ Algorithm — Directed Minimum Spanning Tree (Arborescence)

---

## Origin & Motivation
Chu–Liu/Edmonds’ algorithm was introduced independently by Chu & Liu (1965) and Jack Edmonds (1967).  
It generalizes the concept of a minimum spanning tree (MST) from **undirected graphs** to **directed graphs**.

- In directed graphs, the equivalent structure is called a **minimum spanning arborescence** (or optimum branching).  
- An arborescence is a directed rooted tree that spans all vertices, with edges directed away from the root.  
- The algorithm finds such a structure with minimum total edge weight.  

This is vital in:
- Network routing and optimization  
- Compiler design (dominator trees)  
- Dependency resolution  
- Hierarchical clustering in directed settings  

---

## Core Idea
The algorithm operates in several phases:

1. **Select minimum incoming edges**  
   For each vertex (except the root), choose the incoming edge with minimum weight.

2. **Check for cycles**  
   If no cycles are formed, the chosen edges form the minimum spanning arborescence.  
   If cycles exist, contract each cycle into a single super‑node.

3. **Adjust weights and recurse**  
   Recalculate edge weights for the contracted graph and repeat the process until no cycles remain.

4. **Expand contracted cycles**  
   Reconstruct the final arborescence by expanding contracted cycles back into original vertices.

The algorithm ensures that the resulting directed spanning tree has minimum total weight.

---

## Where It’s Used
- **Network design** — optimal routing trees in directed communication networks  
- **Control flow analysis** — dominator trees in compilers  
- **Dependency graphs** — resolving minimum cost hierarchies  
- **Machine learning** — directed clustering and spanning structures  
- **Game AI** — building efficient decision trees with directed dependencies  

---

## Chu–Liu/Edmonds’ Algorithm (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge {
    int u, v;
    int w;
};

int chuLiuEdmonds(int root, int n, vector<Edge>& edges) {
    const int INF = 1e9;
    int res = 0;
    while (true) {
        // Step 1: find minimum incoming edge for each node
        vector<int> in(n, INF), pre(n, -1);
        for (auto& e : edges) {
            if (e.u != e.v && e.w < in[e.v]) {
                in[e.v] = e.w;
                pre[e.v] = e.u;
            }
        }
        in[root] = 0;
        // If some node has no incoming edge, arborescence impossible
        for (int i = 0; i < n; i++) {
            if (in[i] == INF) return -1;
        }
        // Step 2: detect cycles
        int cnt = 0;
        vector<int> id(n, -1), vis(n, -1);
        for (int i = 0; i < n; i++) res += in[i];
        for (int i = 0; i < n; i++) {
            int v = i;
            while (vis[v] != i && id[v] == -1 && v != root) {
                vis[v] = i;
                v = pre[v];
            }
            if (v != root && id[v] == -1) {
                for (int u = pre[v]; u != v; u = pre[u]) id[u] = cnt;
                id[v] = cnt++;
            }
        }
        if (cnt == 0) break; // no cycles
        for (int i = 0; i < n; i++) if (id[i] == -1) id[i] = cnt++;
        // Step 3: contract cycles and adjust edges
        vector<Edge> newEdges;
        for (auto& e : edges) {
            int u = id[e.u], v = id[e.v];
            int w = e.w;
            if (u != v) w -= in[e.v];
            newEdges.push_back({u, v, w});
        }
        n = cnt;
        root = id[root];
        edges = newEdges;
    }
    return res;
}

int main() {
    int n = 4, root = 0;
    vector<Edge> edges = {
        {0,1,1}, {0,2,5}, {1,2,1}, {2,3,1}, {1,3,3}
    };
    int ans = chuLiuEdmonds(root, n, edges);
    cout << "Minimum Arborescence Weight = " << ans << endl;
}
```

## Chu–Liu/Edmonds’ Algorithm (C#)
```cpp
using System;
using System.Collections.Generic;

public class Edge {
    public int U, V, W;
    public Edge(int u, int v, int w) { U = u; V = v; W = w; }
}

public class Edmonds {
    public static int ChuLiuEdmonds(int root, int n, List<Edge> edges) {
        const int INF = int.MaxValue / 2;
        int res = 0;
        while (true) {
            int[] in = new int[n];
            int[] pre = new int[n];
            for (int i = 0; i < n; i++) { in[i] = INF; pre[i] = -1; }
            foreach (var e in edges) {
                if (e.U != e.V && e.W < in[e.V]) {
                    in[e.V] = e.W;
                    pre[e.V] = e.U;
                }
            }
            in[root] = 0;
            for (int i = 0; i < n; i++) if (in[i] == INF) return -1;
            res += Sum(in);
            int cnt = 0;
            int[] id = new int[n];
            int[] vis = new int[n];
            for (int i = 0; i < n; i++) { id[i] = -1; vis[i] = -1; }
            for (int i = 0; i < n; i++) {
                int v = i;
                while (vis[v] != i && id[v] == -1 && v != root) {
                    vis[v] = i;
                    v = pre[v];
                }
                if (v != root && id[v] == -1) {
                    for (int u = pre[v]; u != v; u = pre[u]) id[u] = cnt;
                    id[v] = cnt++;
                }
            }
            if (cnt == 0) break;
            for (int i = 0; i < n; i++) if (id[i] == -1) id[i] = cnt++;
            var newEdges = new List<Edge>();
            foreach (var e in edges) {
                int u = id[e.U], v = id[e.V];
                int w = e.W;
                if (u != v) w -= in[e.V];
                newEdges.Add(new Edge(u, v, w));
            }
            n = cnt;
            root = id[root];
            edges = newEdges;
        }
        return res;
    }

    private static int Sum(int[] arr) {
        int s = 0; foreach (var x in arr) s += x; return s;
    }

    public static void Main() {
        int n = 4, root = 0;
        var edges = new List<Edge> {
            new Edge(0,1,1), new Edge(0,2,5),
            new Edge(1,2,1), new Edge(2,3,1),
            new Edge(1,3,3)
        };
        int ans = ChuLiuEdmonds(root, n, edges);
        Console.WriteLine("Minimum Arborescence Weight = " + ans);
    }
}
```



##  Complexity Analysis

### Time Complexity
The algorithm proceeds in iterative phases:

1. **Minimum incoming edge selection**  
   For each vertex (except the root), select the minimum incoming edge.  
   Complexity: O(E).

2. **Cycle detection**  
   Using predecessor tracking and visitation arrays, cycles are identified.  
   Complexity: O(V + E).

3. **Cycle contraction and weight adjustment**  
   Each cycle is contracted into a super‑node, and edge weights are adjusted.  
   Complexity: O(E).

4. **Iterations**  
   In the worst case, up to O(V) contractions may occur.  
   Overall worst‑case complexity: **O(E · V)**.

**Optimizations**  
With advanced data structures (e.g., priority queues, union‑find variants), the complexity can be reduced to **O(E log V)**, making the algorithm more practical for large graphs.

---

### Space Complexity
Memory usage is linear in the size of the graph:

- **O(V)** for arrays:  
  - `in[]` — minimum incoming edge weights  
  - `pre[]` — predecessors  
  - `id[]` — contracted cycle identifiers  
  - `vis[]` — visitation markers  

- **O(E)** for the edge list representation.  

Total: **O(V + E)**.

---

## Key Concepts Recap
- **Arborescence**: a directed spanning tree rooted at a given node, with edges directed outward.  
- **Minimum incoming edge**: each non‑root vertex must select one incoming edge of minimum weight.  
- **Cycle contraction**: cycles are collapsed into super‑nodes to enforce tree structure.  
- **Weight adjustment**: edge weights are reduced to account for chosen incoming edges.  
- **Expansion**: contracted cycles are expanded back to reconstruct the final arborescence.

---

##  Conclusion
Chu–Liu/Edmonds’ algorithm extends the minimum spanning tree concept to directed graphs.  
It systematically constructs a minimum spanning arborescence, which is crucial in:

- Routing and network optimization  
- Compiler analysis (dominator trees)  
- Dependency resolution in directed systems  

Although more complex than undirected MST algorithms such as Kruskal or Prim, Chu–Liu/Edmonds’ remains the **standard solution** for directed minimum spanning tree problems, balancing correctness with efficiency.


---


