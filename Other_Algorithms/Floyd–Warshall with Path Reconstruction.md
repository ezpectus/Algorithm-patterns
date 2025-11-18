# Floyd–Warshall with Path Reconstruction  
**Dense graph APSP with traceability**

---

##  Origin and Motivation
- **Origin:** Floyd–Warshall is a dynamic programming algorithm for all-pairs shortest paths (APSP).  
- **Date of creation:** The algorithm was independently published in the **1960s** by **Robert Floyd (1962)** and **Stephen Warshall (1962)**.  
- **Motivation:** It works on dense graphs and handles negative edges (but not negative cycles).  
- **Extension:** With path reconstruction, we not only compute shortest distances but also recover the actual shortest paths.  

---

##  Where It’s Used

| Domain               | Use Case                                    |
|----------------------|---------------------------------------------|
| Dense networks       | APSP in fully connected or dense graphs     |
| Systems analysis     | Dependency graphs with penalties            |
| Education            | Teaching APSP and dynamic programming       |
| Competitive programming | APSP with path recovery for traceability |

---

##  When to Use vs Alternatives

| Task                          | Floyd–Warshall | Johnson | Dijkstra APSP |
|-------------------------------|----------------|---------|---------------|
| Dense graphs                  | ✅ Best choice | ⚠️ Overkill | ❌ Too slow |
| Sparse graphs with negatives  | ❌ Too slow    | ✅ Yes   | ❌ Needs nonnegative |
| Path reconstruction needed    | ✅ Easy        | ⚠️ Extra work | ⚠️ Requires parent arrays |
| Negative cycle detection      | ✅ Built-in    | ✅ Yes   | ❌ No |

---

##  Core Idea
- Initialize distance matrix `dist[i][j]` with edge weights (or INF if no edge).  
- Initialize `next[i][j]` with `j` if edge `(i,j)` exists.  
- Triple loop over all vertices `k, i, j`:  
  - If `dist[i][k] + dist[k][j] < dist[i][j]`, update `dist[i][j]` and set `next[i][j] = next[i][k]`.  
- After completion, `dist[i][j]` holds shortest distance, and `next[i][j]` allows path reconstruction.  
- Path reconstruction: repeatedly follow `next[i][j]` until destination is reached.  

---

##  Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const long long INF = 1e15;

void floydWarshall(int n, vector<vector<long long>>& dist, vector<vector<int>>& next) {
    for (int k = 0; k < n; k++) {
        for (int i = 0; i < n; i++) {
            for (int j = 0; j < n; j++) {
                if (dist[i][k] < INF && dist[k][j] < INF && dist[i][k] + dist[k][j] < dist[i][j]) {
                    dist[i][j] = dist[i][k] + dist[k][j];
                    next[i][j] = next[i][k];
                }
            }
        }
    }
}

vector<int> getPath(int u, int v, const vector<vector<int>>& next) {
    if (next[u][v] == -1) return {};
    vector<int> path = {u};
    while (u != v) {
        u = next[u][v];
        path.push_back(u);
    }
    return path;
}

int main() {
    int n = 4;
    vector<vector<long long>> dist(n, vector<long long>(n, INF));
    vector<vector<int>> next(n, vector<int>(n, -1));

    // Example edges
    dist[0][1] = 5; next[0][1] = 1;
    dist[0][3] = 10; next[0][3] = 3;
    dist[1][2] = 3; next[1][2] = 2;
    dist[2][3] = 1; next[2][3] = 3;

    for (int i = 0; i < n; i++) dist[i][i] = 0, next[i][i] = i;

    floydWarshall(n, dist, next);

    cout << "Shortest distance 0->3: " << dist[0][3] << "\n";
    auto path = getPath(0, 3, next);
    cout << "Path: ";
    for (int x : path) cout << x << " ";
    cout << "\n";
}
```

## Implementation (C#)
```csharp
using System;
using System.Collections.Generic;

class FloydWarshall {
    const long INF = (long)1e15;

    public static void Run(int n, long[,] dist, int[,] next) {
        for (int k = 0; k < n; k++) {
            for (int i = 0; i < n; i++) {
                for (int j = 0; j < n; j++) {
                    if (dist[i,k] < INF && dist[k,j] < INF && dist[i,k] + dist[k,j] < dist[i,j]) {
                        dist[i,j] = dist[i,k] + dist[k,j];
                        next[i,j] = next[i,k];
                    }
                }
            }
        }
    }

    public static List<int> GetPath(int u, int v, int[,] next) {
        if (next[u,v] == -1) return new List<int>();
        var path = new List<int> { u };
        while (u != v) {
            u = next[u,v];
            path.Add(u);
        }
        return path;
    }

    static void Main() {
        int n = 4;
        long[,] dist = new long[n,n];
        int[,] next = new int[n,n];
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++) {
                dist[i,j] = INF;
                next[i,j] = -1;
            }

        dist[0,1] = 5; next[0,1] = 1;
        dist[0,3] = 10; next[0,3] = 3;
        dist[1,2] = 3; next[1,2] = 2;
        dist[2,3] = 1; next[2,3] = 3;

        for (int i = 0; i < n; i++) { dist[i,i] = 0; next[i,i] = i; }

        Run(n, dist, next);

        Console.WriteLine("Shortest distance 0->3: " + dist[0,3]);
        var path = GetPath(0, 3, next);
        Console.Write("Path: ");
        foreach (var x in path) Console.Write(x + " ");
    }
}
```


#  Complexity Analysis — Floyd–Warshall with Path Reconstruction

---

## Time Complexity
- **O(n^3)** — cubic time due to the triple nested loop over all vertices `(i, j, k)`.  
- Best suited for **dense graphs** where `m ≈ n^2`.  
- For sparse graphs, Johnson’s algorithm or repeated Dijkstra runs are more efficient.  

## Space Complexity
- **O(n^2)** for storing the distance matrix.  
- **O(n^2)** for the `next[i][j]` matrix used in path reconstruction.  
- Total: quadratic space, which is manageable for moderate `n` but heavy for very large graphs.  

---

#  Impact of Design Choices

| Choice              | Effect                                                                 |
|---------------------|------------------------------------------------------------------------|
| Dense vs sparse     | Works best on dense graphs; sparse graphs better with Johnson or repeated Dijkstra. |
| Path reconstruction | Requires `next[i][j]` matrix; adds memory but enables traceability of shortest paths. |
| Negative cycles     | Detectable if `dist[i][i] < 0` after completion.                       |
| Initialization      | Proper setup of `dist` and `next` matrices is critical for correctness. |
| Edge weights type   | Use 64-bit integers (`long long` in C++ / `long` in C#) to avoid overflow. |

---

#  Pitfalls

- **Forgetting to initialize `next[i][j]`:** Path reconstruction fails if not set correctly at the start.  
- **Sparse graphs:** Algorithm becomes inefficient compared to Johnson or Dijkstra.  
- **Negative cycles:** If not checked, results may be invalid.  
- **Overflow risks:** Large edge weights can exceed 32-bit integer limits.  
- **Disconnected graphs:** Distances remain `INF`; must handle output carefully.  

---

#  Conclusion

- **What it gives:** Exact APSP on dense graphs with path reconstruction.  
- **Why it matters:** Provides not only shortest distances but also the actual paths, useful for traceability, debugging, and applications requiring route details.  
- **Key takeaway:** Floyd–Warshall with path reconstruction is the **go-to APSP method** for dense graphs when both distances and paths must be recovered.  




---





