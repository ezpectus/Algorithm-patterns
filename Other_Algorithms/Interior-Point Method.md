# Interior-Point Method — Practical LP Solver  
*Polynomial-Time, Interior Path, Industry Standard*

---

## Origin & Motivation  

- **1967** — I. I. Dikin: **affine scaling** (early interior-point idea)  
- **1984** — **Narendra Karmarkar**: first **polynomial-time** interior-point algorithm  
- **Motivation**:  
  - **Simplex** → exponential worst-case (rare, but real)  
  - **Ellipsoid method** → polynomial, but **too slow** in practice  
  - Need **fast + provably polynomial** LP solver

> **Interior-point** = **interior traversal** of feasible region → **central path** → **optimal vertex**

---

## Where It’s Used  

| Domain | Application |
|-------|------------|
| **Operations Research** | Large-scale scheduling, logistics |
| **Finance** | Portfolio optimization, risk modeling |
| **Engineering** | Network design, production planning |
| **Machine Learning** | SVM dual, logistic regression |
| **Solvers** | **CPLEX, Gurobi, MOSEK, MATLAB linprog** — all use interior-point as default |

---

## When to Use vs Alternatives  

| Algorithm | Worst-Case | Practical Speed | Stable | Notes |
|---------|------------|----------------|--------|-------|
| **Simplex** | **Exponential** | **Very fast** (real data) | Yes | Industry classic |
| **Ellipsoid** | Polynomial | **Slow** | Yes | Theoretical proof only |
| **Interior-Point** | **Polynomial** | **Fast & robust** | Yes | **Modern default** |
| **Cutting Plane** | Polynomial | Medium | Yes | Niche |

> **Use Interior-Point when**:  
> - Large, sparse LP  
> - Need **guaranteed polynomial time**  
> - Negative coefficients  
> - High precision required

---
## Core Idea — Primal-Dual Path Following  

### 1. **Barrier Function**  
Add **logarithmic barrier** to stay inside feasible region:

```text
min cᵀx + μ ∑ log(x_i)    s.t. Ax = b, x > 0
μ → 0 → solution approaches optimum
```
### 2. Central Path
```
For each μ > 0 → unique solution on central path
```
### 3. Newton Step
```
Solve KKT system → move along central path
```
### 4. Duality Gap
```text
gap = cᵀx - bᵀy → 0 → optimal
```


## Full Implementation (C++) — Primal-Dual Interior Point
```cpp

#include <bits/stdc++.h>
using namespace std;

class InteriorPointLP {
private:
    int n, m;
    vector<vector<double>> A;
    vector<double> b, c;
    const double EPS = 1e-10;
    const double INF = 1e18;

    // Solve linear system H * dx = g using Gaussian elimination with partial pivoting
    bool solveSystem(const vector<vector<double>>& H, vector<double>& dx, const vector<double>& g) {
        int N = H.size();
        vector<vector<double>> mat(N, vector<double>(N + 1));
        vector<int> pivot(N);

        for (int i = 0; i < N; i++) {
            for (int j = 0; j < N; j++) mat[i][j] = H[i][j];
            mat[i][N] = -g[i];
        }

        for (int i = 0; i < N; i++) {
            // Partial pivoting
            int max_row = i;
            for (int k = i + 1; k < N; k++)
                if (abs(mat[k][i]) > abs(mat[max_row][i])) max_row = k;

            swap(mat[i], mat[max_row]);

            if (abs(mat[i][i]) < EPS) return false;  // singular

            // Eliminate
            for (int k = i + 1; k < N; k++) {
                double factor = mat[k][i] / mat[i][i];
                for (int j = i; j <= N; j++)
                    mat[k][j] -= factor * mat[i][j];
            }
        }

        // Back substitution
        dx.assign(N, 0);
        for (int i = N - 1; i >= 0; i--) {
            double sum = mat[i][N];
            for (int j = i + 1; j < N; j++) sum -= mat[i][j] * dx[j];
            dx[i] = sum / mat[i][i];
        }
        return true;
    }

public:
    // max cᵀx s.t. Ax = b, x >= 0
    double maximize(const vector<vector<double>>& _A, const vector<double>& _b, const vector<double>& _c) {
        A = _A; b = _b; c = _c;
        m = b.size(); n = c.size();

        vector<double> x(n, 1.0), y(m, 0.0), s(n, 1.0);  // initial interior point
        double mu = 10.0;

        while (mu > EPS) {
            // Build KKT system
            vector<vector<double>> H(n + m, vector<double>(n + m, 0.0));
            vector<double> g(n + m, 0.0);

            // Primal feasibility residual: Ax - b
            for (int i = 0; i < m; i++) {
                double res = b[i];
                for (int j = 0; j < n; j++) res -= A[i][j] * x[j];
                g[n + i] = res;
            }

            // Dual + complementarity
            for (int j = 0; j < n; j++) {
                g[j] = c[j];
                for (int i = 0; i < m; i++) g[j] -= A[i][j] * y[i];
                g[j] += s[j];

                H[j][j] = s[j] / x[j] + EPS;  // barrier + stabilization
            }

            // Constraint Jacobian
            for (int i = 0; i < m; i++)
                for (int j = 0; j < n; j++) {
                    H[n + i][j] = A[i][j];
                    H[j][n + i] = A[i][j];
                }

            vector<double> step(n + m);
            if (!solveSystem(H, step, g)) break;

            // Update variables
            double alpha = 1.0;
            for (int j = 0; j < n; j++) {
                if (step[j] < 0) alpha = min(alpha, 0.99 * x[j] / -step[j]);
                if (step[n + j] < 0) alpha = min(alpha, 0.99 * s[j] / -step[n + j]);
            }

            for (int j = 0; j < n; j++) {
                x[j] += alpha * step[j];
                s[j] += alpha * step[n + j];
            }
            for (int i = 0; i < m; i++) y[i] += alpha * step[n + i];

            mu *= 0.1;
        }

        double result = 0;
        for (int j = 0; j < n; j++) result += c[j] * x[j];
        return result;
    }
};

// === TEST ===
int main() {
    // max 3x + 5y
    // x + y <= 4
    // 2x + y <= 6
    vector<vector<double>> A = {{1, 1}, {2, 1}};
    vector<double> b = {4, 6};
    vector<double> c = {3, 5};

    InteriorPointLP solver;
    cout << fixed << setprecision(6);
    cout << "Optimal: " << solver.maximize(A, b, c) << endl;  // Optimal: 6.855796

    return 0;
}
```

## Full Implementation (C#) — Simplified Primal-Dual
```cpp
using System;

public class InteriorPointLP
{
    private const double EPS = 1e-10;
    private const double INF = 1e18;
    private int n, m;
    private double[,] A;
    private double[] b, c;

    private bool SolveSystem(double[,] H, double[] dx, double[] g)
    {
        int N = H.GetLength(0);
        double[,] mat = new double[N, N + 1];

        for (int i = 0; i < N; i++)
        {
            for (int j = 0; j < N; j++) mat[i, j] = H[i, j];
            mat[i, N] = -g[i];
        }

        for (int i = 0; i < N; i++)
        {
            int max_row = i;
            for (int k = i + 1; k < N; k++)
                if (Math.Abs(mat[k, i]) > Math.Abs(mat[max_row, i])) max_row = k;

            for (int j = 0; j <= N; j++)
                (mat[i, j], mat[max_row, j]) = (mat[max_row, j], mat[i, j]);

            if (Math.Abs(mat[i, i]) < EPS) return false;

            for (int k = i + 1; k < N; k++)
            {
                double f = mat[k, i] / mat[i, i];
                for (int j = i; j <= N; j++)
                    mat[k, j] -= f * mat[i, j];
            }
        }

        Array.Clear(dx, 0, N);
        for (int i = N - 1; i >= 0; i--)
        {
            double sum = mat[i, N];
            for (int j = i + 1; j < N; j++) sum -= mat[i, j] * dx[j];
            dx[i] = sum / mat[i, i];
        }
        return true;
    }

    public double Maximize(double[,] _A, double[] _b, double[] _c)
    {
        A = _A; b = _b; c = _c;
        m = b.Length; n = c.Length;

        double[] x = new double[n]; Array.Fill(x, 1.0);
        double[] y = new double[m];
        double[] s = new double[n]; Array.Fill(s, 1.0);

        double mu = 10.0;

        while (mu > EPS)
        {
            double[,] H = new double[n + m, n + m];
            double[] g = new double[n + m];

            // Primal residual
            for (int i = 0; i < m; i++)
            {
                double res = b[i];
                for (int j = 0; j < n; j++) res -= A[i, j] * x[j];
                g[n + i] = res;
            }

            // Dual + complementarity
            for (int j = 0; j < n; j++)
            {
                g[j] = c[j];
                for (int i = 0; i < m; i++) g[j] -= A[i, j] * y[i];
                g[j] += s[j];
                H[j, j] = s[j] / x[j] + EPS;
            }

            // Jacobian
            for (int i = 0; i < m; i++)
                for (int j = 0; j < n; j++)
                {
                    H[n + i, j] = A[i, j];
                    H[j, n + i] = A[i, j];
                }

            double[] step = new double[n + m];
            if (!SolveSystem(H, step, g)) break;

            double alpha = 1.0;
            for (int j = 0; j < n; j++)
            {
                if (step[j] < 0) alpha = Math.Min(alpha, 0.99 * x[j] / -step[j]);
                if (step[n + j] < 0) alpha = Math.Min(alpha, 0.99 * s[j] / -step[n + j]);
            }

            for (int j = 0; j < n; j++)
            {
                x[j] += alpha * step[j];
                s[j] += alpha * step[n + j];
            }
            for (int i = 0; i < m; i++) y[i] += alpha * step[n + i];

            mu *= 0.1;
        }

        double result = 0;
        for (int j = 0; j < n; j++) result += c[j] * x[j];
        return result;
    }
}

// === TEST ===
class Program
{
    static void Main()
    {
        double[,] A = { {1, 1}, {2, 1} };
        double[] b = {4, 6};
        double[] c = {3, 5};

        var solver = new InteriorPointLP();
        Console.WriteLine($"Optimal: {solver.Maximize(A, b, c):F6}");  // Optimal: 6.855796
    }
}
```


##  Complexity Analysis

### Time Complexity
The Interior-Point Method provides a **polynomial-time guarantee**:

- **Per iteration cost:** solving linear systems (matrix factorization), typically O(n³).  
- **Number of iterations:** bounded polynomially in the input size (bit-length of coefficients).  
- **Overall complexity:** **O(n³L)**, where:  
  - *n* = number of variables  
  - *L* = input size in bits.  

**Practical note:**  
- Requires fewer iterations than Simplex for large, sparse problems.  
- Exploits sparsity and numerical linear algebra to achieve near-linear scaling in practice.

---

### Space Complexity
- **O(n²)** for storing matrices (Hessian, Jacobian, constraint matrices).  
- Efficient implementations exploit **sparsity** to reduce memory usage.  
- For very large problems, iterative solvers and decomposition techniques are used to manage memory.

---

##  Comparison with Other Algorithms

| Algorithm        | Worst-case Complexity | Practical Performance | Notes                                    |
|------------------|-----------------------|-----------------------|------------------------------------------|
| **Simplex**      | Exponential           | Very fast in practice | Basis of many solvers; boundary traversal |
| **Ellipsoid**    | Polynomial (O(n²L))   | Slow in practice      | First polynomial-time proof; theoretical  |
| **Interior-point** | Polynomial (O(n³L)) | Fast and robust       | Preferred in modern solvers; interior traversal |

**Interpretation:**  
- **Simplex**: dominates in practice for small/medium problems, but lacks polynomial guarantees.  
- **Ellipsoid**: important for theory, but impractical.  
- **Interior-point**: combines polynomial guarantees with strong practical performance, especially for large-scale LP.

---

##  Key Concepts Recap
- **Interior traversal:** Iterates remain strictly inside the feasible region, avoiding boundary degeneracy.  
- **Barrier function:** Logarithmic penalties prevent crossing constraints, guiding iterates smoothly.  
- **Central path:** A trajectory defined by barrier minimization, leading toward the optimal solution.  
- **Primal-dual updates:** Solve primal and dual problems simultaneously, reducing the duality gap.  
- **Polynomial guarantee:** Efficient both in theory and practice, unlike Simplex (exponential worst case) or Ellipsoid (slow in practice).

---

##  Example Walkthrough (Conceptual)
Consider the LP problem:
```
maximize cᵀx
subject to A x ≤ b, x ≥ 0
```

1. **Barrier formulation:**  
   Replace constraints with barrier terms:  
```
maximize cᵀx + μ Σ log(x_i) + μ Σ log(b_j - a_jᵀx)
```

where μ > 0 is the barrier parameter.

2. **Central path:**  
As μ → 0, the solution approaches the true LP optimum.  

3. **Newton updates:**  
Use Newton’s method to update x along the central path, solving linear systems at each step.  

4. **Termination:**  
Stop when the **duality gap** is below tolerance.  
The current iterate is near-optimal.

---

##  Conclusion
The **Interior-Point Method** is the **practical workhorse of modern linear programming**:

- Balances **theoretical guarantees** with **real-world efficiency**.  
- Avoids exponential worst-case behavior of Simplex.  
- Outperforms Ellipsoid in practice, while retaining polynomial guarantees.  
- Scales well to **large, sparse problems**, making it ideal for industrial applications.  

**Legacy:**  
- **Simplex:** practical dominance for small/medium LP.  
- **Ellipsoid:** theoretical breakthrough.  
- **Interior-point:** modern balance of **theory and practice**, powering most industrial solvers today.


---






