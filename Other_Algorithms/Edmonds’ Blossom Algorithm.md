# Edmonds’ Blossom Algorithm — Maximum Matching in General Graphs

---

## Origin & Motivation

**Edmonds’ Blossom Algorithm**, introduced in **1961**, was the **first polynomial-time solution** for computing a **maximum matching in non-bipartite graphs**.  
Its key innovation: **shrinking odd-length cycles (“blossoms”)** to unblock **augmenting paths** — a breakthrough that extended matching theory **beyond bipartite constraints**.

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Network Design | Pairing under constraints |
| Combinatorial Optimization | Vertex cover, edge coloring |
| Graph Theory | Perfect matchings, factor-critical graphs |
| Game Theory | Stable pairings, strategic matchings |
| Bioinformatics | Molecule pairing, RNA folding |

---

## When to Use Blossom vs Alternatives

| Scenario | Blossom | Hopcroft–Karp | Hungarian | Greedy |
|--------|---------|---------------|-----------|--------|
| General graph (non-bipartite) | Yes | No | No | No |
| Bipartite graph | Yes | Yes | Yes | Yes |
| Perfect matching required | Yes | Yes | Yes | No |
| Weighted matching | No | No | Yes | No |
| Fast approximate matching | No | No | No | Yes |

---

## Core Idea

* Start with an **empty matching**  
* Search for **augmenting paths** — alternating unmatched/matched edges  
* If an **odd-length cycle** blocks the path:  
  * **Shrink** the cycle into a **pseudo-node (blossom)**  
  * Continue searching in the **contracted graph**  
* Once an augmenting path is found:  
  * **Expand blossoms**  
  * **Flip** matched/unmatched edges along the path  
* Repeat until **no augmenting paths remain**  

This guarantees a **maximum cardinality matching** in general graphs.

---

## Implementation (C++)

```cpp
const int N = 500;
vector<int> adj[N];
int match[N], base[N], p[N];
bool used[N], blossom[N];
int n;

int lca(int a, int b) {
    bool visited[N] = {};
    while (true) {
        a = base[a];
        visited[a] = true;
        if (match[a] == -1) break;
        a = p[match[a]];
    }
    while (true) {
        b = base[b];
        if (visited[b]) return b;
        if (match[b] == -1) break;
        b = p[match[b]];
    }
    return -1;
}

void markPath(int v, int b, int child) {
    while (base[v] != b) {
        blossom[base[v]] = blossom[base[match[v]]] = true;
        p[v] = child;
        child = match[v];
        v = p[match[v]];
    }
}

int findPath(int root) {
    fill(used, used + n, false);
    fill(p, p + n, -1);
    iota(base, base + n, 0);
    queue<int> q;
    q.push(root);
    used[root] = true;

    while (!q.empty()) {
        int v = q.front(); q.pop();
        for (int u : adj[v]) {
            if (base[v] == base[u] || match[v] == u) continue;
            if (u == root || (match[u] != -1 && p[match[u]] != -1)) {
                int curbase = lca(v, u);
                fill(blossom, blossom + n, false);
                markPath(v, curbase, u);
                markPath(u, curbase, v);
                for (int i = 0; i < n; ++i)
                    if (blossom[base[i]]) {
                        base[i] = curbase;
                        if (!used[i]) {
                            used[i] = true;
                            q.push(i);
                        }
                    }
            } else if (p[u] == -1) {
                p[u] = v;
                if (match[u] == -1) return u;
                used[match[u]] = true;
                q.push(match[u]);
            }
        }
    }
    return -1;
}

int edmonds() {
    fill(match, match + n, -1);
    int res = 0;
    for (int v = 0; v < n; ++v)
        if (match[v] == -1) {
            int u = findPath(v);
            if (u != -1) {
                res++;
                int cur = u, prev;
                while (cur != -1) {
                    prev = p[cur];
                    int next = match[prev];
                    match[cur] = prev;
                    match[prev] = cur;
                    cur = next;
                }
            }
        }
    return res;
}
```

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class EdmondsBlossom {
    const int N = 500;
    List<int>[] adj = new List<int>[N];
    int[] match = new int[N], baseNode = new int[N], parent = new int[N];
    bool[] used = new bool[N], blossom = new bool[N];
    int n;

    public EdmondsBlossom(int vertices) {
        n = vertices;
        for (int i = 0; i < n; i++) {
            adj[i] = new List<int>();
            match[i] = -1;
        }
    }

    public void AddEdge(int u, int v) {
        adj[u].Add(v);
        adj[v].Add(u);
    }

    int LCA(int a, int b) {
        bool[] visited = new bool[n];
        while (true) {
            a = baseNode[a];
            visited[a] = true;
            if (match[a] == -1) break;
            a = parent[match[a]];
        }
        while (true) {
            b = baseNode[b];
            if (visited[b]) return b;
            if (match[b] == -1) break;
            b = parent[match[b]];
        }
        return -1;
    }

    void MarkPath(int v, int b, int child) {
        while (baseNode[v] != b) {
            blossom[baseNode[v]] = blossom[baseNode[match[v]]] = true;
            parent[v] = child;
            child = match[v];
            v = parent[match[v]];
        }
    }

    int FindPath(int root) {
        Array.Fill(used, false);
        Array.Fill(parent, -1);
        for (int i = 0; i < n; i++) baseNode[i] = i;

        Queue<int> q = new Queue<int>();
        q.Enqueue(root);
        used[root] = true;

        while (q.Count > 0) {
            int v = q.Dequeue();
            foreach (int u in adj[v]) {
                if (baseNode[v] == baseNode[u] || match[v] == u) continue;
                if (u == root || (match[u] != -1 && parent[match[u]] != -1)) {
                    int curbase = LCA(v, u);
                    Array.Fill(blossom, false);
                    MarkPath(v, curbase, u);
                    MarkPath(u, curbase, v);
                    for (int i = 0; i < n; i++)
                        if (blossom[baseNode[i]]) {
                            baseNode[i] = curbase;
                            if (!used[i]) {
                                used[i] = true;
                                q.Enqueue(i);
                            }
                        }
                } else if (parent[u] == -1) {
                    parent[u] = v;
                    if (match[u] == -1) return u;
                    used[match[u]] = true;
                    q.Enqueue(match[u]);
                }
            }
        }
        return -1;
    }

    public int MaximumMatching() {
        Array.Fill(match, -1);
        int res = 0;
        for (int v = 0; v < n; v++)
            if (match[v] == -1) {
                int u = FindPath(v);
                if (u != -1) {
                    res++;
                    int cur = u, prev;
                    while (cur != -1) {
                        prev = parent[cur];
                        int next = match[prev];
                        match[cur] = prev;
                        match[prev] = cur;
                        cur = next;
                    }
                }
            }
        return res;
    }
}
```

## Complexity Analysis

| Phase               | Complexity         |
|---------------------|--------------------|
| **Augmenting path** | O(n²)              |
| **Total matching**  | O(n³)              |
| **Space**           | O(n + m)           |

> **Improved versions** (Micali–Vazirani) achieve **O(√n · m)**  
> **Blossom shrinking** is the bottleneck in naive implementations

---

## Pitfalls

* **Complex to implement correctly** — many edge cases in blossom handling  
* Requires **careful tracking** of base nodes and parent pointers  
* **Not suitable for weighted matching** — use Hungarian or Edmonds’ weighted version  
* **C# lacks native graph libraries** — manual porting required  

---

## Conclusion

Edmonds’ Blossom Algorithm is the **global matching engine for general graphs**:

* Handles **odd cycles** via **blossom shrinking**  
* Guarantees **maximum cardinality matching**  
* Enables **polynomial-time solutions** to non-bipartite matching  
* Forms the **foundation** for advanced graph algorithms  

> **Key takeaway**:  
> **Blossom is the go-to algorithm when bipartite assumptions break** — it’s the **cherry on top for general matching problems**.

---

