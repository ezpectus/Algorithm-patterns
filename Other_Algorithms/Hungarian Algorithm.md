# Hungarian Algorithm — Optimal Assignment Engine

---

## Origin & Motivation

**Hungarian Algorithm** was introduced in **1955** by **Harold Kuhn**, based on earlier work by **Dénes Kőnig** and **Jenő Egerváry**.  
It solves the **assignment problem**:  
> Given `n` agents and `n` tasks with a cost matrix `C[i][j]`, assign each agent to a **unique task** such that the **total cost is minimized**.

It is **strongly polynomial**, **deterministic**, and **guarantees an optimal solution**.

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Task Scheduling | Machines ↔ Jobs |
| Logistics | Drivers ↔ Routes |
| AI Matching | Agents ↔ Goals |
| Game Theory | Players ↔ Roles |
| Resource Allocation | Workers ↔ Projects |

---

## When to Use Hungarian vs Alternatives

| Scenario | Hungarian | Greedy | Max-Flow | Bipartite Matching |
|--------|-----------|--------|----------|--------------------|
| Cost matrix available | Yes | No | Yes | Yes |
| Optimal assignment required | Yes | No | Yes | Yes |
| Weighted bipartite graph | Yes | No | Yes | Yes |
| Fast approximate result | No | Yes | No | No |
| Unequal agents/tasks | Warning (pad matrix) | Yes | Yes | Yes |

---

## Core Idea

1. **Subtract row minimums** from each row  
2. **Subtract column minimums** from each column  
3. **Cover all zeros** with minimum number of lines  
4. If **number of lines < n**, **adjust matrix** and repeat  
5. Once **all zeros are covered**, find a **zero matching** using DFS or BFS  
6. Construct the **optimal assignment** from matched zeros  

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const int INF = 1e9;

int hungarian(const vector<vector<int>>& cost) {
    int n = cost.size();
    vector<int> u(n + 1), v(n + 1), p(n + 1), way(n + 1);

    for (int i = 1; i <= n; ++i) {
        p[0] = i;
        int j0 = 0;
        vector<int> minv(n + 1, INF);
        vector<char> used(n + 1, false);

        do {
            used[j0] = true;
            int i0 = p[j0], delta = INF, j1 = -1;
            for (int j = 1; j <= n; ++j) {
                if (!used[j]) {
                    int cur = cost[i0 - 1][j - 1] - u[i0] - v[j];
                    if (cur < minv[j]) {
                        minv[j] = cur;
                        way[j] = j0;
                    }
                    if (minv[j] < delta) {
                        delta = minv[j];
                        j1 = j;
                    }
                }
            }

            for (int j = 0; j <= n; ++j) {
                if (used[j]) {
                    u[p[j]] += delta;
                    v[j] -= delta;
                } else {
                    minv[j] -= delta;
                }
            }
            j0 = j1;
        } while (p[j0] != 0);

        do {
            int j1 = way[j0];
            p[j0] = p[j1];
            j0 = j1;
        } while (j0);
    }

    return -v[0]; // total cost
}
```

## Implementation (C#)

```csharp
using System;
using System.Linq;

public class HungarianAlgorithm {
    public static int Solve(int[][] cost) {
        int n = cost.Length;
        int[] u = new int[n + 1], v = new int[n + 1], p = new int[n + 1], way = new int[n + 1];

        for (int i = 1; i <= n; i++) {
            p[0] = i;
            int j0 = 0;
            int[] minv = Enumerable.Repeat(int.MaxValue, n + 1).ToArray();
            bool[] used = new bool[n + 1];

            do {
                used[j0] = true;
                int i0 = p[j0], delta = int.MaxValue, j1 = -1;
                for (int j = 1; j <= n; j++) {
                    if (!used[j]) {
                        int cur = cost[i0 - 1][j - 1] - u[i0] - v[j];
                        if (cur < minv[j]) {
                            minv[j] = cur;
                            way[j] = j0;
                        }
                        if (minv[j] < delta) {
                            delta = minv[j];
                            j1 = j;
                        }
                    }
                }

                for (int j = 0; j <= n; j++) {
                    if (used[j]) {
                        u[p[j]] += delta;
                        v[j] -= delta;
                    } else {
                        minv[j] -= delta;
                    }
                }
                j0 = j1;
            } while (p[j0] != 0);

            do {
                int j1 = way[j0];
                p[j0] = p[j1];
                j0 = j1;
            } while (j0 != 0);
        }

        return -v[0]; // total cost
    }
}
```
## Complexity Analysis

| Phase               | Complexity |
|---------------------|------------|
| **Matrix transform** | O(n²)      |
| **Matching search**  | O(n³)      |
| **Space**            | O(n²)      |

---

## Conclusion

**Hungarian Algorithm** is the **optimal assignment engine**:

* Guarantees **minimum cost matching**  
* Works on **weighted bipartite graphs**  
* **Polynomial-time** and **deterministic**  
* Ideal for **scheduling, logistics, and resource allocation**  

> **Key takeaway**:  
> **When you need perfect matching with minimum cost, Hungarian is the gold standard** — elegant, efficient, and **exact**.

---

