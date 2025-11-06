# Stoer–Wagner Algorithm — Global Min-Cut in Undirected Graphs

---

## Origin & Motivation

**Stoer–Wagner Algorithm**, introduced in **1995** by **Mechthild Stoer** and **Frank Wagner**, solves the **global minimum cut** problem in **undirected weighted graphs**.  
Unlike **max-flow-based approaches** (e.g. Karger’s randomized algorithm), Stoer–Wagner is:

* **Deterministic**  
* **Exact**  
* Works on **general graphs** with **non-negative weights**

It finds the **smallest set of edges** whose removal disconnects the graph.

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Network Reliability | Minimum failure point |
| Graph Clustering | Partitioning into weakly connected components |
| Circuit Design | Cut-based optimization |
| Bioinformatics | Protein interaction networks |
| Infrastructure Resilience | Weakest link detection |

---

## When to Use Stoer–Wagner vs Alternatives

| Scenario | Stoer–Wagner | Karger | Max-Flow | DFS |
|--------|--------------|--------|----------|-----|
| Undirected weighted graph | Yes | Yes | Yes | No |
| Exact min-cut required | Yes | No | Yes | No |
| Deterministic result | Yes | No | Yes | Yes |
| Large sparse graph | No | Yes | Yes | Yes |
| Fast approximate cut | No | Yes | No | Yes |

---

## Core Idea

1. **Repeatedly merge vertices** using **maximum adjacency** (most tightly connected pairs)  
2. **Track the last added vertex** and its **total connection weight**  
3. The **minimum weight** of the **last phase** is the **global min-cut**  
4. **Shrinking** preserves cut values — no need to track partitions explicitly  

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

int stoerWagner(vector<vector<int>>& graph) {
    int n = graph.size();
    vector<int> vertices(n);
    iota(vertices.begin(), vertices.end(), 0);
    int minCut = INT_MAX;

    while (vertices.size() > 1) {
        int sz = vertices.size();
        vector<int> weights(sz, 0);
        vector<bool> added(sz, false);
        int prev = -1;

        for (int i = 0; i < sz; ++i) {
            int sel = -1;
            for (int j = 0; j < sz; ++j)
                if (!added[j] && (sel == -1 || weights[j] > weights[sel]))
                    sel = j;

            if (i == sz - 1) {
                minCut = min(minCut, weights[sel]);
                if (prev != -1) {
                    for (int j = 0; j < sz; ++j) {
                        graph[vertices[prev]][vertices[j]] += graph[vertices[sel]][vertices[j]];
                        graph[vertices[j]][vertices[prev]] += graph[vertices[j]][vertices[sel]];
                    }
                    vertices.erase(vertices.begin() + sel);
                }
                break;
            }

            added[sel] = true;
            prev = sel;
            for (int j = 0; j < sz; ++j)
                if (!added[j])
                    weights[j] += graph[vertices[sel]][vertices[j]];
        }
    }

    return minCut;
}
```

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class StoerWagner {
    public static int Run(int[][] graph) {
        int n = graph.Length;
        var vertices = Enumerable.Range(0, n).ToList();
        int minCut = int.MaxValue;

        while (vertices.Count > 1) {
            int sz = vertices.Count;
            var weights = new int[sz];
            var added = new bool[sz];
            int prev = -1;

            for (int i = 0; i < sz; i++) {
                int sel = -1;
                for (int j = 0; j < sz; j++)
                    if (!added[j] && (sel == -1 || weights[j] > weights[sel]))
                        sel = j;

                if (i == sz - 1) {
                    minCut = Math.Min(minCut, weights[sel]);
                    if (prev != -1) {
                        for (int j = 0; j < sz; j++) {
                            graph[vertices[prev]][vertices[j]] += graph[vertices[sel]][vertices[j]];
                            graph[vertices[j]][vertices[prev]] += graph[vertices[j]][vertices[sel]];
                        }
                        vertices.RemoveAt(sel);
                    }
                    break;
                }

                added[sel] = true;
                prev = sel;
                for (int j = 0; j < sz; j++)
                    if (!added[j])
                        weights[j] += graph[vertices[sel]][vertices[j]];
            }
        }

        return minCut;
    }
}
```


## Complexity Analysis

| Metric        | Value     |
|---------------|-----------|
| **Time**      | O(n³)     |
| **Space**     | O(n²)     |
| **Deterministic** | Yes   |
| **Exact**     | Yes       |

---

## Pitfalls

* Only works for **undirected graphs** with **non-negative weights**  
* **Not suitable for dynamic graphs** — must rebuild from scratch  
* **Slower than randomized methods** on large sparse graphs  
* Requires **careful vertex merging logic** to preserve cut values  

---

## Conclusion

Stoer–Wagner is the **global min-cut engine for undirected graphs**:

* **Deterministic and exact**  
* Handles **arbitrary undirected weighted graphs**  
* Ideal for **network reliability** and **graph partitioning**  
* **Shrinks graph** while preserving cut values  

> **Key takeaway**:  
> **Use Stoer–Wagner when you need a guaranteed minimum cut in a general undirected graph** — no randomness, no flow networks, just **pure structural insight**.

---
