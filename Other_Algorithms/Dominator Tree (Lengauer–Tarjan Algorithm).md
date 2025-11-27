# Dominator Tree (Lengauer–Tarjan Algorithm) — Control Flow Graphs and Compilers

---

## Origin & Motivation
The **Lengauer–Tarjan algorithm** was introduced in 1979 by Thomas Lengauer and Robert Tarjan.  
It is a fundamental algorithm in compiler theory and graph analysis, designed to efficiently compute **dominator trees** in directed graphs.

- In a **control flow graph (CFG)**, a node *u* dominates node *v* if every path from the entry node to *v* passes through *u*.  
- The **dominator tree** organizes these relationships hierarchically.  
- This structure is crucial in compiler optimizations, program analysis, and understanding control dependencies.

---

## Core Idea
The algorithm computes dominators using a combination of **DFS numbering**, **union–find with path compression**, and **link–eval techniques**:

1. **DFS traversal**  
   Assigns each node a DFS number and builds a spanning tree rooted at the entry node.

2. **Semi-dominators**  
   For each node, compute the smallest DFS ancestor that dominates it via paths outside the DFS tree.

3. **Link–Eval structure**  
   Uses union–find with path compression to efficiently evaluate minimum semi-dominators.

4. **Immediate dominators**  
   From semi-dominators, derive immediate dominators (idom) for each node.

5. **Tree construction**  
   Build the dominator tree by linking each node to its immediate dominator.

---

## Where It’s Used
- **Compiler design** — building dominator trees for control flow analysis, SSA form construction, loop detection.  
- **Program optimization** — dead code elimination, register allocation, restructuring.  
- **Static analysis** — understanding dependencies and control structures.  
- **Security analysis** — identifying critical control points in execution paths.  
- **Graph theory research** — efficient computation of dominance relations in directed graphs.

---

## Lengauer–Tarjan Algorithm (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

struct LT {
    int n, root;
    vector<vector<int>> adj, radj, bucket;
    vector<int> semi, idom, parent, vertex, label, ancestor;
    vector<int> dom;

    LT(int n, int root) : n(n), root(root) {
        adj.assign(n, {});
        radj.assign(n, {});
        bucket.assign(n, {});
        semi.resize(n);
        idom.resize(n);
        parent.resize(n);
        vertex.resize(n);
        label.resize(n);
        ancestor.resize(n);
        dom.resize(n);
    }

    void addEdge(int u, int v) { adj[u].push_back(v); }

    void dfs(int v, int& time) {
        time++;
        semi[v] = time;
        vertex[time] = v;
        label[v] = v;
        ancestor[v] = -1;
        for (int w : adj[v]) {
            if (!semi[w]) {
                parent[w] = v;
                dfs(w, time);
            }
            radj[w].push_back(v);
        }
    }

    int eval(int v) {
        if (ancestor[v] == -1) return label[v];
        compress(v);
        return semi[label[v]] < semi[label[ancestor[v]]] ? label[v] : label[ancestor[v]];
    }

    void compress(int v) {
        if (ancestor[ancestor[v]] != -1) {
            compress(ancestor[v]);
            if (semi[label[ancestor[v]]] < semi[label[v]])
                label[v] = label[ancestor[v]];
            ancestor[v] = ancestor[ancestor[v]];
        }
    }

    void link(int v, int w) { ancestor[w] = v; }

    void run() {
        int time = 0;
        dfs(root, time);
        for (int i = time; i >= 2; i--) {
            int w = vertex[i];
            for (int v : radj[w]) {
                int u = eval(v);
                semi[w] = min(semi[w], semi[u]);
            }
            bucket[vertex[semi[w]]].push_back(w);
            link(parent[w], w);
            for (int v : bucket[parent[w]]) {
                int u = eval(v);
                idom[v] = (semi[u] < semi[v]) ? u : parent[w];
            }
            bucket[parent[w]].clear();
        }
        for (int i = 2; i <= time; i++) {
            int w = vertex[i];
            if (idom[w] != vertex[semi[w]])
                idom[w] = idom[idom[w]];
            dom[w] = idom[w];
        }
        dom[root] = -1; // root has no dominator
    }
};
```

## Lengauer–Tarjan Algorithm (C#)
```cpp
using System;
using System.Collections.Generic;

public class LengauerTarjan {
    private int n, root;
    private List<int>[] adj, radj, bucket;
    private int[] semi, idom, parent, vertex, label, ancestor, dom;

    public LengauerTarjan(int n, int root) {
        this.n = n; this.root = root;
        adj = new List<int>[n];
        radj = new List<int>[n];
        bucket = new List<int>[n];
        for (int i = 0; i < n; i++) {
            adj[i] = new List<int>();
            radj[i] = new List<int>();
            bucket[i] = new List<int>();
        }
        semi = new int[n];
        idom = new int[n];
        parent = new int[n];
        vertex = new int[n];
        label = new int[n];
        ancestor = new int[n];
        dom = new int[n];
    }

    public void AddEdge(int u, int v) { adj[u].Add(v); }

    private void Dfs(int v, ref int time) {
        time++;
        semi[v] = time;
        vertex[time] = v;
        label[v] = v;
        ancestor[v] = -1;
        foreach (var w in adj[v]) {
            if (semi[w] == 0) {
                parent[w] = v;
                Dfs(w, ref time);
            }
            radj[w].Add(v);
        }
    }

    private int Eval(int v) {
        if (ancestor[v] == -1) return label[v];
        Compress(v);
        return semi[label[v]] < semi[label[ancestor[v]]] ? label[v] : label[ancestor[v]];
    }

    private void Compress(int v) {
        if (ancestor[ancestor[v]] != -1) {
            Compress(ancestor[v]);
            if (semi[label[ancestor[v]]] < semi[label[v]])
                label[v] = label[ancestor[v]];
            ancestor[v] = ancestor[ancestor[v]];
        }
    }

    private void Link(int v, int w) { ancestor[w] = v; }

    public void Run() {
        int time = 0;
        Dfs(root, ref time);
        for (int i = time; i >= 2; i--) {
            int w = vertex[i];
            foreach (var v in radj[w]) {
                int u = Eval(v);
                semi[w] = Math.Min(semi[w], semi[u]);
            }
            bucket[vertex[semi[w]]].Add(w);
            Link(parent[w], w);
            foreach (var v in bucket[parent[w]]) {
                int u = Eval(v);
                idom[v] = (semi[u] < semi[v]) ? u : parent[w];
            }
            bucket[parent[w]].Clear();
        }
        for (int i = 2; i <= time; i++) {
            int w = vertex[i];
            if (idom[w] != vertex[semi[w]])
                idom[w] = idom[idom[w]];
            dom[w] = idom[w];
        }
        dom[root] = -1;
    }
}
```



## ⏱️ Complexity Analysis

### Time Complexity
The algorithm achieves near-linear performance by combining DFS traversal with efficient union–find operations:

- **DFS traversal:** O(V + E)  
  Each vertex and edge is visited once to assign DFS numbers and build the spanning tree.

- **Semi-dominator computation:** O(E)  
  For each edge, semi-dominators are updated using link–eval structures.

- **Union–find with path compression:** amortized O(α(V)) per operation  
  Ensures efficient evaluation of semi-dominators, where α(V) is the inverse Ackermann function.

- **Overall complexity:** **O(V + E)**  
  Near-linear time, optimal for large control flow graphs.

---

### Space Complexity
Memory usage is linear in graph size:

- **O(V)** for arrays:  
  - `semi[]` — semi-dominators  
  - `idom[]` — immediate dominators  
  - `parent[]` — DFS tree parents  
  - `vertex[]` — DFS numbering  
  - `label[]` — union–find labels  
  - `ancestor[]` — union–find ancestors  
  - `dom[]` — final dominator tree

- **O(E)** for adjacency lists.  

- **Total:** **O(V + E)**.

---

## Key Concepts Recap
- **Dominator:** Node *u* dominates *v* if every path from the entry node to *v* passes through *u*.  
- **Immediate dominator (idom):** The closest dominator of a node, forming the parent in the dominator tree.  
- **Dominator tree:** A hierarchical structure representing dominance relations in the graph.  
- **Semi-dominator:** An intermediate value used to compute immediate dominators efficiently.  
- **Union–find with path compression:** A data structure that accelerates evaluation of semi-dominators.

---

##  Conclusion
The **Lengauer–Tarjan algorithm** is the standard method for computing dominator trees in directed graphs.  
It is central to compiler construction, enabling optimizations such as:

- Static Single Assignment (SSA) form  
- Loop detection and restructuring  
- Dead code elimination  
- Control flow analysis  

With near-linear complexity, it scales to large control flow graphs, making it indispensable in modern program analysis and optimization.


---


