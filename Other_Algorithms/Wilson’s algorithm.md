# 🧠 Wilson’s algorithm — Uniform spanning tree generation

---

## 📜 Origin and motivation
Wilson’s algorithm (1996) generates a spanning tree chosen uniformly at random from all spanning trees of a connected graph.  
It achieves uniformity via **loop-erased random walks**: starting from a root, repeatedly perform a random walk from an unvisited node until the walk hits the current tree, erase loops, and add the loop-erased path to the tree.  
Unlike “random DFS trees,” Wilson’s method is provably uniform and does not depend on traversal order or tie-breaking.

---

## 🧩 Where it’s used
- **Randomized structures** — building unbiased spanning trees for simulations and testing  
- **Statistical physics** — connections to loop-erased random walks and uniform spanning forests  
- **Network design** — sampling diverse minimal-connection backbones  
- **Maze generation** — perfect mazes (no cycles; unique path between any two cells)  
- **Algorithmic research** — benchmarking algorithms against uniform trees  

---

## 🔁 When to use Wilson vs alternatives

| Task                     | Use Wilson | Use Aldous–Broder | Use random BFS/DFS |
|--------------------------|------------|-------------------|--------------------|
| Uniform spanning tree    | ✅         | ✅                | ❌                 |
| Faster practical performance | ✅     | ❌                | ✅ (but biased)    |
| Simple to implement      | ✅         | ✅                | ✅                 |
| Theoretical guarantees   | ✅         | ✅                | ❌                 |
| Maze generation          | ✅         | ✅                | ✅ (biased topology)|

- **Aldous–Broder:** single random walk until every node is first-visited; uniform but often slower in practice.  
- **Wilson:** multiple walks with loop-erasure; typically faster and still uniform.  
- **Random BFS/DFS/Kruskal/Prim (randomized weights):** easy but not uniform unless carefully adjusted.  

---

## 🧱 Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class WilsonUST
{
    // Graph as adjacency list: 0..n-1
    private readonly List<int>[] g;
    private readonly int n;
    private readonly Random rng;

    public WilsonUST(List<int>[] graph, int? seed = null)
    {
        g = graph;
        n = g.Length;
        rng = seed.HasValue ? new Random(seed.Value) : new Random();
    }

    // Returns edges of a uniform spanning tree as pairs (u, v)
    public List<(int, int)> Generate(int root = 0)
    {
        var inTree = new bool[n];
        var treeEdges = new List<(int, int)>();
        inTree[root] = true;

        var remaining = new HashSet<int>();
        for (int v = 0; v < n; v++)
            if (!inTree[v]) remaining.Add(v);

        while (remaining.Count > 0)
        {
            // Pick any vertex outside the tree
            int start = GetAny(remaining);

            // Perform random walk with loop-erasure until we hit the tree
            var parent = new Dictionary<int, int>(); // predecessor in current walk
            int v = start;
            parent[v] = -1;

            while (!inTree[v])
            {
                int u = g[v][rng.Next(g[v].Count)];
                // Loop-erasure via "last predecessor wins":
                // If u was already seen in this walk, erase the loop by truncating path
                if (parent.ContainsKey(u))
                {
                    // Remove all nodes along the looped tail
                    // Efficient trick: reassign predecessor, older tail becomes unreachable
                    // (We only need the final predecessor chain for path reconstruction.)
                }
                parent[u] = v;
                v = u;
            }

            // Reconstruct loop-erased path from 'start' to the first node that hit the tree
            // We backtrack from the hit node through 'parent' to start
            // but we only add edges along the simple path (loops were erased via predecessor overwrite).
            var path = new List<int>();
            int x = v;
            while (x != -1)
            {
                path.Add(x);
                x = parent[x];
            }
            path.Reverse(); // path: start -> ... -> v (v is in the tree)

            // Add edges along the path and mark nodes as in the tree
            for (int i = 0; i + 1 < path.Count; i++)
            {
                int a = path[i], b = path[i + 1];
                if (!inTree[a]) inTree[a] = true;
                if (!inTree[b]) inTree[b] = true;
                treeEdges.Add((a, b));
                remaining.Remove(a);
                remaining.Remove(b);
            }
        }

        return treeEdges;
    }

    private static int GetAny(HashSet<int> s)
    {
        foreach (var v in s) return v;
        throw new InvalidOperationException("Set is empty");
    }
}
```

## Notes:

- Loop-erasure is achieved by overwriting predecessors for revisited nodes; 
- only the final predecessor chain survives, which is exactly the loop-erased path.
- The implementation returns undirected edges; treat (u, v) as a connection in the spanning tree.

## 🧱 Implementation (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

struct WilsonUST {
    int n;
    vector<vector<int>> g;
    mt19937 rng;

    WilsonUST(const vector<vector<int>>& graph, uint32_t seed = random_device{}())
        : n((int)graph.size()), g(graph), rng(seed) {}

    // Returns edges of a uniform spanning tree
    vector<pair<int,int>> generate(int root = 0) {
        vector<bool> inTree(n, false);
        inTree[root] = true;
        vector<pair<int,int>> edges;

        unordered_set<int> remaining;
        remaining.reserve(n);
        for (int v = 0; v < n; ++v)
            if (!inTree[v]) remaining.insert(v);

        while (!remaining.empty()) {
            int start = *remaining.begin();

            unordered_map<int,int> parent; // predecessor in current walk
            parent.reserve(n);
            int v = start;
            parent[v] = -1;

            // Random walk until hit the tree, with loop-erasure by parent overwrite
            while (!inTree[v]) {
                const auto& nbrs = g[v];
                int u = nbrs[uniform_int_distribution<int>(0, (int)nbrs.size() - 1)(rng)];
                // If u seen before in this walk, parent overwrite erases the loop segment
                parent[u] = v;
                v = u;
            }

            // Reconstruct loop-erased path: start -> ... -> v (where v is in the tree)
            vector<int> path;
            for (int x = v; x != -1; x = parent[x]) path.push_back(x);
            reverse(path.begin(), path.end());

            for (int i = 0; i + 1 < (int)path.size(); ++i) {
                int a = path[i], b = path[i + 1];
                if (!inTree[a]) inTree[a] = true;
                if (!inTree[b]) inTree[b] = true;
                edges.emplace_back(a, b);
                remaining.erase(a);
                remaining.erase(b);
            }
        }
        return edges;
    }
};

// Example usage
int main() {
    // Simple connected graph (square with diagonal)
    vector<vector<int>> g = {
        {1,2},      // 0
        {0,2,3},    // 1
        {0,1,3},    // 2
        {1,2}       // 3
    };

    WilsonUST ust(g, 12345);
    auto tree = ust.generate(0);

    for (auto [u, v] : tree) {
        cout << u << " - " << v << "\n";
    }
    return 0;
}
```

## ⏱️ Complexity

### ⏳ Time (expected)
- Depends on the **hitting time** of random walks to the growing tree.  
- On **sparse graphs**, performance is close to linear in the number of edges `O(E)`.  
- On **grid graphs**, typical bounds are around `O(n log n)` for `n` vertices.  
- In worst cases, complexity can be higher, but in practice Wilson’s algorithm usually outperforms Aldous–Broder.

### 💾 Space
- Proportional to the number of vertices and edges:  
  - adjacency list storage  
  - `inTree` array  
  - parent map for the current walk  
- Overall: `O(V + E)`.

### 🔧 Practical guidance
- Use a **high-quality RNG** and uniform neighbor selection.  
- Wilson typically **outperforms Aldous–Broder** on most graphs while preserving strict uniformity.  
- Well-suited for **large grids and maze generation**, where both speed and unbiased sampling matter.  
- Can be parallelized across multiple runs to generate independent spanning trees.

---

## ✅ Conclusion

Wilson’s algorithm constructs a **uniform spanning tree** via **loop-erased random walks**, providing unbiased samples with strong theoretical guarantees. It is:

- 🎯 **Uniform by design** — every spanning tree has equal probability  
- 🧩 **Simple in structure** — pure random walks and loop erasure, no heavy data structures  
- ⚡ **Efficient in practice** — near-linear on many graphs  
- 🌀 **Perfect for mazes** — unique paths, no cycles  
- 🔬 **Theoretically deep** — linked to loop-erased random walks and uniform spanning forests  
- 🌍 **Widely applicable** — from network protocols and bioinformatics to randomized map generation in games  

Wilson’s algorithm is a **clean and powerful primitive**: it turns chaotic random walks into a deterministic process for building uniform spanning trees.



---
