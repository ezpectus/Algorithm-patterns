# Kuhn–Munkres (Hungarian) Algorithm  
**Assignment problem (min-cost bipartite matching)**

---

##  Origin & Motivation
- **Origin:** Developed by Harold Kuhn in 1955, refined by James Munkres in 1957.  
- **Motivation:** Solve the **assignment problem** — optimally matching `n` workers to `n` jobs with minimum total cost.  
- **Key Insight:** Transform the cost matrix using row/column reductions and alternating paths until an optimal matching is found.  

---

##  Where It’s Used

| Domain               | Use Case                                    |
|----------------------|---------------------------------------------|
| Operations Research  | Workforce scheduling, resource allocation   |
| Economics            | Optimal assignment of tasks or contracts    |
| Computer Vision      | Object tracking (matching detections across frames) |
| AI / ML              | Matching problems in clustering, graph alignment |
| Logistics            | Assigning vehicles to routes, minimizing cost |

---

##  When to Use vs Alternatives

| Task                          | Hungarian Algorithm | Max-flow / Min-cost flow | Greedy Matching | Auction Algorithm |
|-------------------------------|---------------------|--------------------------|-----------------|------------------|
| Exact min-cost assignment     | ✅ Yes              | ✅ Yes                   | ❌ Not optimal  | ✅ Yes            |
| Bipartite matching (balanced) | ✅ Yes              | ✅ Yes                   | ✅ Works        | ✅ Works          |
| Large sparse graphs           | ⚠️ Less efficient   | ✅ Better                | ✅ Fast         | ✅ Good           |
| Approximate solution needed   | ❌ No               | ❌ No                    | ✅ Yes          | ✅ Yes            |

---

##  Core Idea
- Represent the problem as an **n×n cost matrix**.  
- Perform **row and column reductions** to simplify costs.  
- Construct a bipartite graph of zero-cost edges.  
- Find a maximum matching using alternating paths.  
- If matching is incomplete, adjust the matrix by shifting uncovered elements until a perfect matching is found.  
- Result: Minimum-cost assignment.

---

##  Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// Hungarian Algorithm for assignment problem
struct Hungarian {
    int n;
    vector<vector<int>> cost;
    vector<int> u, v, p, way;

    Hungarian(int n, vector<vector<int>> cost) : n(n), cost(cost) {
        u.assign(n+1, 0);
        v.assign(n+1, 0);
        p.assign(n+1, 0);
        way.assign(n+1, 0);
    }

    int solve() {
        for (int i = 1; i <= n; i++) {
            p[0] = i;
            int j0 = 0;
            vector<int> minv(n+1, INT_MAX);
            vector<char> used(n+1, false);
            do {
                used[j0] = true;
                int i0 = p[j0], delta = INT_MAX, j1 = 0;
                for (int j = 1; j <= n; j++) {
                    if (!used[j]) {
                        int cur = cost[i0-1][j-1] - u[i0] - v[j];
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
            } while (j0);
        }
        return -v[0]; // minimum cost
    }
};

int main() {
    vector<vector<int>> cost = {
        {9, 2, 7, 8},
        {6, 4, 3, 7},
        {5, 8, 1, 8},
        {7, 6, 9, 4}
    };
    Hungarian hungarian(4, cost);
    cout << "Minimum cost: " << hungarian.solve() << "\n";
}
```


##  Implementation (C#)
```cpp
using System;
using System.Collections.Generic;

public class Hungarian {
    private int n;
    private int[,] cost;
    private int[] u, v, p, way;

    public Hungarian(int[,] cost) {
        this.n = cost.GetLength(0);
        this.cost = cost;
        u = new int[n+1];
        v = new int[n+1];
        p = new int[n+1];
        way = new int[n+1];
    }

    public int Solve() {
        for (int i = 1; i <= n; i++) {
            p[0] = i;
            int j0 = 0;
            int[] minv = new int[n+1];
            bool[] used = new bool[n+1];
            for (int j = 0; j <= n; j++) minv[j] = int.MaxValue;

            do {
                used[j0] = true;
                int i0 = p[j0], delta = int.MaxValue, j1 = 0;
                for (int j = 1; j <= n; j++) {
                    if (!used[j]) {
                        int cur = cost[i0-1, j-1] - u[i0] - v[j];
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
        return -v[0];
    }
}

class Program {
    static void Main() {
        int[,] cost = {
            {9, 2, 7, 8},
            {6, 4, 3, 7},
            {5, 8, 1, 8},
            {7, 6, 9, 4}
        };
        var hungarian = new Hungarian(cost);
        Console.WriteLine("Minimum cost: " + hungarian.Solve());
    }
}
```

##  Complexity Analysis

- **Time Complexity:**  
  - `O(n^3)` — cubic time complexity.  
  - Efficient for moderate `n` (up to ~500).  
  - For very large graphs, alternative approaches (e.g., min-cost flow) may be more suitable.  

- **Space Complexity:**  
  - `O(n^2)` for storing the cost matrix.  
  - Additional `O(n)` for auxiliary arrays (`u`, `v`, `p`, `way`).  

---

##  Impact of Design Choices

| Choice                          | Effect                                                                 |
|---------------------------------|------------------------------------------------------------------------|
| Cost matrix representation      | Simple to implement, but requires `O(n^2)` memory.                     |
| Algorithm variant (Kuhn vs Munkres) | Both achieve `O(n^3)` complexity; differences are mainly in implementation details. |
| Scaling to large `n`            | For very large instances, min-cost flow algorithms may be more practical. |
| Integer vs floating costs       | Hungarian works naturally with integers; floating-point requires careful precision handling. |

---

##  Pitfalls

- **Square matrix requirement:**  
  - The algorithm requires a square cost matrix.  
  - If the problem is unbalanced (e.g., more workers than jobs), pad with dummy rows/columns.  

- **Integer overflow:**  
  - Large cost values may cause overflow.  
  - Use 64-bit integers (`long long` in C++ / `long` in C#).  

- **Implementation details:**  
  - Indexing errors and update logic are common pitfalls.  
  - Careful handling of arrays (`u`, `v`, `p`, `way`) is critical.  

- **Scalability:**  
  - Not efficient for very large graphs.  
  - For `n > 1000`, min-cost flow or auction algorithms may be preferable.  

---

##  Conclusion

- **What it gives:**  
  - An exact solution to the assignment problem in polynomial time.  
  - Guarantees optimal matching between two sets in bipartite graphs.  

- **Why it matters:**  
  - Fundamental in optimization, scheduling, and resource allocation.  
  - Widely used in operations research, logistics, and computer vision.  

- **Key takeaway:**  
  - The Hungarian Algorithm is the **go-to method** for solving min-cost bipartite matching in `O(n^3)` time.  


---
