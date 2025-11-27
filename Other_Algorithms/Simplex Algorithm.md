# Simplex Algorithm — Basic Method of Linear Programming

---

## Origin & Motivation
The **Simplex Algorithm** was introduced by **George Dantzig in 1947**.  
It is the foundational method for solving **linear programming (LP)** problems, which involve:

- **Objective function**: maximize or minimize a linear function.  
- **Constraints**: linear equalities or inequalities.  
- **Feasible region**: convex polytope defined by constraints.  

The key insight: the optimal solution of an LP lies at a **vertex** of the feasible region.  
The Simplex method systematically moves from vertex to vertex to find the optimum.

---

## Core Idea
1. **Convert to standard form**  
   - Transform inequalities into equalities using slack variables.  
   - Ensure all variables are non-negative.  

2. **Initial basic feasible solution**  
   - Start at a vertex of the feasible region (often slack variables = RHS).  

3. **Pivot operations**  
   - Choose an entering variable (to improve the objective).  
   - Choose a leaving variable (to maintain feasibility).  
   - Perform Gaussian elimination to update the tableau.  

4. **Iterate**  
   - Move along edges of the polytope to adjacent vertices.  
   - Each move improves or maintains the objective function.  

5. **Termination**  
   - Stop when no entering variable can improve the objective.  
   - The current solution is optimal.  

---

## Where It’s Used
- **Operations research** — resource allocation, scheduling, logistics.  
- **Finance** — portfolio optimization, cost minimization.  
- **Engineering** — production planning, network flows.  
- **Computer science** — optimization in algorithms and machine learning.  

---

## Simplex Algorithm (C++)
```cpp
#include <bits/stdc++.h>
using namespace std;

const double EPS = 1e-9;
const double INF = 1e18;

class Simplex {
private:
    int m, n;                    // constraints, variables
    vector<vector<double>> table;
    vector<int> basis;

    void pivot(int r, int s) {
        double inv = 1.0 / table[r][s];
        for (int j = 0; j <= n; j++) table[r][j] *= inv;
        for (int i = 0; i <= m; i++) {
            if (i == r) continue;
            double factor = table[i][s];
            for (int j = 0; j <= n; j++)
                table[i][j] -= factor * table[r][j];
        }
        basis[r] = s;
    }

    bool phase1() {
        while (true) {
            int s = -1;
            for (int j = 0; j < n; j++) {
                if (table[m][j] < -EPS) { s = j; break; }
            }
            if (s == -1) return true;

            int r = -1;
            double mn = INF;
            for (int i = 0; i < m; i++) {
                if (table[i][s] > EPS) {
                    double ratio = table[i][n] / table[i][s];
                    if (ratio < mn - EPS || (abs(ratio - mn) < EPS && basis[i] < basis[r])) {
                        mn = ratio; r = i;
                    }
                }
            }
            if (r == -1) return false;  // unbounded
            pivot(r, s);
        }
    }

    double phase2() {
        while (true) {
            int s = -1;
            for (int j = 0; j < n; j++) {
                if (table[m][j] > EPS) { s = j; break; }
            }
            if (s == -1) return table[m][n];

            int r = -1;
            double mn = INF;
            for (int i = 0; i < m; i++) {
                if (table[i][s] > EPS) {
                    double ratio = table[i][n] / table[i][s];
                    if (ratio < mn - EPS || (abs(ratio - mn) < EPS && basis[i] < basis[r])) {
                        mn = ratio; r = i;
                    }
                }
            }
            if (r == -1) return INF;  // unbounded
            pivot(r, s);
        }
    }

public:
    double solve(const vector<vector<double>>& A, const vector<double>& b, const vector<double>& c) {
        m = A.size();
        n = c.size();
        table.assign(m + 1, vector<double>(n + m + 1, 0.0));
        basis.resize(m);

        // Build initial tableau
        for (int i = 0; i < m; i++) {
            for (int j = 0; j < n; j++) table[i][j] = A[i][j];
            table[i][n + i] = 1.0;           // slack variables
            table[i][n + m] = b[i];
            basis[i] = n + i;
        }
        for (int j = 0; j < n; j++) table[m][j] = -c[j];

        // Phase 1: feasibility
        if (!phase1()) return -INF;  // infeasible

        // Remove artificial variables and restore objective
        table[m] = vector<double>(n + m + 1, 0.0);
        for (int j = 0; j < n; j++) table[m][j] = -c[j];

        return phase2();
    }
};
```


## Simplex Algorithm (C#)
```cpp
using System;

public class Simplex
{
    private const double EPS = 1e-9;
    private const double INF = 1e18;

    private int m, n;
    private double[,] table;
    private int[] basis;

    private void Pivot(int r, int s)
    {
        double inv = 1.0 / table[r, s];
        for (int j = 0; j <= n + m; j++) table[r, j] *= inv;
        for (int i = 0; i <= m; i++)
        {
            if (i == r) continue;
            double factor = table[i, s];
            for (int j = 0; j <= n + m; j++)
                table[i, j] -= factor * table[r, j];
        }
        basis[r] = s;
    }

    private bool Phase1()
    {
        while (true)
        {
            int s = -1;
            for (int j = 0; j < n; j++)
                if (table[m, j] < -EPS) { s = j; break; }
            if (s == -1) return true;

            int r = -1;
            double mn = INF;
            for (int i = 0; i < m; i++)
            {
                if (table[i, s] > EPS)
                {
                    double ratio = table[i, n + m] / table[i, s];
                    if (ratio < mn - EPS || (Math.Abs(ratio - mn) < EPS && basis[i] < basis[r]))
                    {
                        mn = ratio; r = i;
                    }
                }
            }
            if (r == -1) return false;
            Pivot(r, s);
        }
    }

    private double Phase2()
    {
        while (true)
        {
            int s = -1;
            for (int j = 0; j < n; j++)
                if (table[m, j] > EPS) { s = j; break; }
            if (s == -1) return table[m, n + m];

            int r = -1;
            double mn = INF;
            for (int i = 0; i < m; i++)
            {
                if (table[i, s] > EPS)
                {
                    double ratio = table[i, n + m] / table[i, s];
                    if (ratio < mn - EPS || (Math.Abs(ratio - mn) < EPS && basis[i] < basis[r]))
                    {
                        mn = ratio; r = i;
                    }
                }
            }
            if (r == -1) return INF;
            Pivot(r, s);
        }
    }

    public double Solve(double[,] A, double[] b, double[] c)
    {
        m = b.Length;
        n = c.Length;
        table = new double[m + 1, n + m + 1];
        basis = new int[m];

        // Build tableau
        for (int i = 0; i < m; i++)
        {
            for (int j = 0; j < n; j++) table[i, j] = A[i, j];
            table[i, n + i] = 1.0;
            table[i, n + m] = b[i];
            basis[i] = n + i;
        }
        for (int j = 0; j < n; j++) table[m, j] = -c[j];

        if (!Phase1()) return -INF;
        Array.Clear(table[m], 0, n + m + 1);
        for (int j = 0; j < n; j++) table[m, j] = -c[j];

        return Phase2();
    }
}
```


##  Complexity Analysis

### Time Complexity
Theoretical and practical performance differ:

- **Worst-case:**  
  The Simplex algorithm can take exponential time in carefully constructed pathological cases.  
  This is due to the possibility of cycling through many vertices of the feasible polytope before reaching the optimum.

- **Average-case / Practical performance:**  
  In practice, the algorithm is extremely efficient.  
  Most real-world LP problems are solved in polynomial-like time, often with only a few pivot steps relative to the problem size.

- **Modern variants:**  
  Interior-point methods guarantee polynomial time complexity (O(n³L), where L is input size).  
  However, Simplex often outperforms them in practice due to fewer iterations and better cache locality.

**Summary:**  
- Worst-case: exponential.  
- Practical: near-polynomial, very fast.  
- Optimized variants: polynomial guarantees.

---

### Space Complexity
- **Tableau storage:** O(m·n), where m = number of constraints, n = number of variables.  
- **Auxiliary arrays:** O(m + n) for basis tracking, pivot selection, and slack variables.  
- **Total:** Linear in problem size, dominated by tableau storage.

---

## Key Concepts Recap
- **Feasible region:** The convex polytope defined by linear constraints.  
- **Vertex optimality:** Optimal solutions lie at vertices of the feasible region.  
- **Pivoting:** Moving along edges of the polytope to adjacent vertices via basis exchange.  
- **Slack variables:** Introduced to convert inequalities into equalities, enabling tableau formulation.  
- **Termination condition:** When no entering variable can improve the objective, the current solution is optimal.  
- **Degeneracy:** Multiple optimal solutions may exist; Simplex can detect and handle ties.  
- **Cycling prevention:** Techniques like Bland’s rule ensure termination even in degenerate cases.

---

---

## ✅ Conclusion
The **Simplex Algorithm** remains the cornerstone of linear programming:

- It leverages geometric intuition (polytope vertices) with algebraic operations (pivoting).  
- Despite exponential worst-case complexity, it is highly efficient in practice.  
- It is widely used in operations research, finance, engineering, and computer science.  
- Modern LP solvers combine Simplex with interior-point methods, choosing adaptively based on problem structure.  

The algorithm exemplifies how mathematical theory and practical heuristics can coexist:  
**theoretically complex, but practically indispensable.**


---

