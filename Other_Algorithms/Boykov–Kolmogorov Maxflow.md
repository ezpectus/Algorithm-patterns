# 🔷 Boykov–Kolmogorov Maxflow — Graph-Cut Engine for Image Segmentation

## 📜 Origin & Motivation

The Boykov–Kolmogorov algorithm was introduced in the early 2000s by Yuri Boykov and Vladimir Kolmogorov to solve energy minimization problems in computer vision.  
Unlike classical maxflow algorithms, it was designed for grid-like graphs typical in image segmentation, stereo vision, and texture synthesis.

Before Boykov–Kolmogorov:
- Ford–Fulkerson and Push–Relabel were too slow for vision tasks.
- Image segmentation required specialized graph-cut methods.

Boykov–Kolmogorov introduced a bidirectional search tree strategy:
- Grow trees from source and sink  
- Find augmenting paths  
- Push flow and update residual graph

It became a standard in vision libraries like OpenCV and PyMaxflow.

---

## 🧱 Architecture

| Component        | Role                                                  |
|------------------|-------------------------------------------------------|
| Graph            | Represents pixels and their connections              |
| Source/Sink      | Represent foreground/background terminals            |
| Edge capacity    | Encodes likelihood and smoothness constraints        |
| Residual graph   | Tracks remaining capacity after each flow push       |
| Search trees     | Grown from source and sink to find augmenting paths  |

---

## 🔧 Execution Phases

### Graph Construction
- Each pixel → node  
- Neighboring pixels → edges with pairwise costs  
- Source/sink → unary likelihoods

### Tree Growth
- Grow search trees from source and sink  
- Stop when they meet

### Augmentation
- Push flow along discovered path  
- Update residual capacities

### Termination
- Repeat until no augmenting path exists  
- Extract cut → segmentation result

---

## 🚀 C++ Code Skeleton

```cpp
#include <vector>
#include <queue>
using namespace std;

struct Edge {
    int to, rev;
    int cap;
};

class MaxFlow {
    int n;
    vector<vector<Edge>> graph;
    vector<bool> visited;

public:
    MaxFlow(int size) : n(size), graph(size), visited(size) {}

    void addEdge(int u, int v, int cap) {
        graph[u].push_back({v, (int)graph[v].size(), cap});
        graph[v].push_back({u, (int)graph[u].size() - 1, 0});
    }

    bool dfs(int u, int t) {
        if (u == t) return true;
        visited[u] = true;
        for (Edge &e : graph[u]) {
            if (!visited[e.to] && e.cap > 0) {
                if (dfs(e.to, t)) {
                    e.cap -= 1;
                    graph[e.to][e.rev].cap += 1;
                    return true;
                }
            }
        }
        return false;
    }

    int maxFlow(int s, int t) {
        int flow = 0;
        while (true) {
            fill(visited.begin(), visited.end(), false);
            if (!dfs(s, t)) break;
            flow += 1;
        }
        return flow;
    }
};

```

## 🚀 C# Code Skeleton
```csharp
using System;
using System.Collections.Generic;

class Edge {
    public int To, Rev, Cap;
    public Edge(int to, int rev, int cap) {
        To = to; Rev = rev; Cap = cap;
    }
}

class MaxFlow {
    int n;
    List<Edge>[] graph;
    bool[] visited;

    public MaxFlow(int size) {
        n = size;
        graph = new List<Edge>[n];
        visited = new bool[n];
        for (int i = 0; i < n; i++) graph[i] = new List<Edge>();
    }

    public void AddEdge(int u, int v, int cap) {
        graph[u].Add(new Edge(v, graph[v].Count, cap));
        graph[v].Add(new Edge(u, graph[u].Count - 1, 0));
    }

    bool DFS(int u, int t) {
        if (u == t) return true;
        visited[u] = true;
        foreach (Edge e in graph[u]) {
            if (!visited[e.To] && e.Cap > 0) {
                if (DFS(e.To, t)) {
                    e.Cap -= 1;
                    graph[e.To][e.Rev].Cap += 1;
                    return true;
                }
            }
        }
        return false;
    }

    public int MaxFlow(int s, int t) {
        int flow = 0;
        while (true) {
            Array.Fill(visited, false);
            if (!DFS(s, t)) break;
            flow += 1;
        }
        return flow;
    }
}
```
## ⏱️ Complexity Analysis

| Operation            | Time Complexity        | Description                                      |
|----------------------|------------------------|--------------------------------------------------|
| Graph construction   | O(n + m)               | n pixels, m edges between neighboring pixels     |
| Maxflow computation  | Empirical near-linear  | Fast on grid graphs due to localized augmenting paths |
| Space                | O(n + m)               | Stores residual graph, search trees, and metadata |

### Notes:
- **Empirical near-linear** means that although worst-case complexity is not linear, in practice (especially on image grids), the algorithm performs close to linear time.
- **Grid graphs** benefit from spatial locality — augmenting paths are short and predictable.
- **Memory usage** scales with number of pixels and edges, which is manageable for typical image sizes.

---

## ⚠️ Pitfalls

- 🧩 **Edge Capacity Design**  
  Capacities must encode both **unary likelihoods** (foreground/background) and **pairwise smoothness** (neighbor similarity).  
  Poor design leads to inaccurate cuts and noisy segmentation.

- ⚠️ **Grid Topology Sensitivity**  
  Algorithm is optimized for **regular 4-connected or 8-connected grids**.  
  Irregular graphs or sparse connectivity degrade performance and cut quality.

- 🔁 **Static Graph Assumption**  
  The graph is built once and used for the entire segmentation.  
  **Dynamic updates** (e.g., changing pixel weights) require full recomputation — no incremental updates supported.

- 🧠 **Binary Label Limitation**  
  Native algorithm supports **binary segmentation**.  
  Multi-label segmentation requires iterative or layered graph constructions.

---

## ✅ Use Cases

- 🖼️ **Image Segmentation**  
  Foreground/background separation using min-cut on pixel graph.

- 🧠 **Stereo Vision**  
  Depth estimation by modeling disparity as flow between pixel correspondences.

- 🎨 **Texture Synthesis**  
  Seamless blending of image patches via energy minimization.

- 🧬 **Medical Imaging**  
  Tumor boundary detection, organ segmentation, and anatomical region isolation.

- 📸 **Interactive Tools**  
  Used in GrabCut, LazySnapping, and other user-guided segmentation systems.

---

## 🧩 Summary

**Boykov–Kolmogorov Maxflow** is a **vision-optimized graph-cut engine** designed for structured image data:

- 🔍 Efficient on grid-like graphs with spatial coherence  
- 🧠 Converts pixel-level data into flow-based optimization problems  
- ⚙️ Balances unary likelihoods and pairwise smoothness for clean segmentation  
- 🏆 Powers real-time tools in OpenCV, PyMaxflow, and academic research

👉 **Key takeaway:**  
It’s not just a maxflow algorithm — it’s a **segmentation engine** built for vision, speed, and architectural clarity.  
Perfect for tasks where **structure, locality, and precision** matter.



---
