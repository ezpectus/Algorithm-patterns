# Ellipsoid Method — Theoretical Optimization Technique

---

## Origin & Motivation
The **Ellipsoid Method** was first introduced by **Naum Shor (1970s)** for convex optimization.  
Later, **Arkadi Nemirovski and David Yudin (1975)** refined it for convex minimization.  
In **1979**, **Leonid Khachiyan** applied it to **linear programming**, proving that LP can be solved in **polynomial time**.  

This was a breakthrough: while the **Simplex algorithm** was efficient in practice, its worst-case complexity was exponential.  
The Ellipsoid Method provided the first **theoretical guarantee** of polynomial solvability.

---

## Core Idea
The method iteratively shrinks ellipsoids that enclose the feasible region:

1. **Initialization**  
   Start with a large ellipsoid that contains the feasible region.

2. **Iteration**  
   - Check whether the center of the ellipsoid is feasible.  
   - If not, use a separating hyperplane (from violated constraints) to cut the ellipsoid.  
   - Replace the ellipsoid with a smaller one that still contains the feasible region.

3. **Convergence**  
   Each iteration reduces the volume of the ellipsoid.  
   After polynomially many steps, the ellipsoid becomes small enough to approximate the optimal solution.

---

## Where It’s Used
- **Theoretical computer science** — proving polynomial-time solvability of linear programming.  
- **Convex optimization theory** — basis for more advanced algorithms.  
- **Combinatorial optimization** — used in proofs of complexity results.  
- Rarely used in practice, since interior-point methods and Simplex are faster.

---

## Ellipsoid Method (C++ Implementation)
```cpp
#include <bits/stdc++.h>
using namespace std;

// Ellipsoid feasibility for A x <= b. Returns {feasible, x}.
struct EllipsoidResult {
    bool feasible;
    vector<double> x;
};

static const double EPS = 1e-9;

// Find a violated constraint index; returns -1 if none.
int findViolation(const vector<vector<double>>& A, const vector<double>& b, const vector<double>& x) {
    int m = (int)A.size();
    for (int i = 0; i < m; ++i) {
        double val = inner_product(A[i].begin(), A[i].end(), x.begin(), 0.0);
        if (val > b[i] + 1e-12) return i;
    }
    return -1;
}

// Add an extra constraint: c^T x >= t (for LP by binary search).
// Return index of violation if c^T x < t - tol; otherwise -1.
int objectiveViolation(const vector<double>& c, double t, const vector<double>& x) {
    double val = inner_product(c.begin(), c.end(), x.begin(), 0.0);
    if (val < t - 1e-12) return 1; // indicator of objective cut
    return -1;
}

// Ellipsoid feasibility solver.
// If c, t are provided (c.size() == n), enforce c^T x >= t as an extra constraint.
EllipsoidResult ellipsoidFeasibility(
    const vector<vector<double>>& A, const vector<double>& b,
    int maxIter,
    vector<double> x0, vector<vector<double>> Q0,
    const vector<double>& c = {}, double t = -numeric_limits<double>::infinity()
) {
    int n = (int)x0.size();
    vector<double> x = x0;
    vector<vector<double>> Q = Q0;

    for (int iter = 0; iter < maxIter; ++iter) {
        // Check feasibility at center
        int vi = findViolation(A, b, x);
        int objCut = -1;
        if (!c.empty()) objCut = objectiveViolation(c, t, x);
        if (vi == -1 && objCut == -1) {
            return {true, x};
        }

        // Select separating hyperplane
        vector<double> g(n, 0.0);
        if (objCut != -1) {
            // c^T x >= t is violated; use -c^T z <= -t => a = -c, b' = -t
            for (int i = 0; i < n; ++i) g[i] = -c[i];
        } else {
            g = A[vi]; // a_i^T z <= b_i
        }

        // Compute u = Q g / sqrt(g^T Q g)
        vector<double> Qg(n, 0.0);
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                Qg[i] += Q[i][j] * g[j];

        double denom = 0.0;
        for (int i = 0; i < n; ++i) denom += g[i] * Qg[i];
        if (denom <= EPS) break; // numerical failure; abort

        double s = sqrt(denom);
        vector<double> u(n);
        for (int i = 0; i < n; ++i) u[i] = Qg[i] / s;

        // Update center: x' = x - (1/(n+1)) * u
        for (int i = 0; i < n; ++i) x[i] -= u[i] / (n + 1);

        // Update shape: Q' = (n^2 / (n^2 - 1)) * (Q - (2/(n+1)) * u u^T Q)
        vector<vector<double>> u_outer_Q(n, vector<double>(n, 0.0));
        // compute u * (u^T Q) => columns are (u[k] * (u^T Q)[j])
        vector<double> uTQ(n, 0.0);
        for (int j = 0; j < n; ++j) {
            for (int k = 0; k < n; ++k) uTQ[j] += u[k] * Q[k][j];
        }
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                u_outer_Q[i][j] = u[i] * uTQ[j];

        double alpha = (double)(n * n) / (double)(n * n - 1);
        double beta = 2.0 / (n + 1);

        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                Q[i][j] = alpha * (Q[i][j] - beta * u_outer_Q[i][j]);

        // Optional: stopping criterion based on ellipsoid size
        double traceQ = 0.0;
        for (int i = 0; i < n; ++i) traceQ += Q[i][i];
        if (traceQ < 1e-12) break;
    }
    return {false, {}};
}

// Helper: build initial ellipsoid covering a box [-R, R]^n
pair<vector<double>, vector<vector<double>>> initialEllipsoid(int n, double R) {
    vector<double> x0(n, 0.0); // center at origin
    vector<vector<double>> Q(n, vector<double>(n, 0.0));
    for (int i = 0; i < n; ++i) Q[i][i] = R * R; // diagonal matrix
    return {x0, Q};
}

int main() {
    // Example: A x <= b feasibility
    // x1 + x2 <= 5
    // 2x1 + x2 <= 6
    vector<vector<double>> A = {{1, 1}, {2, 1}};
    vector<double> b = {5, 6};

    int n = 2;
    auto [x0, Q0] = initialEllipsoid(n, 10.0);
    EllipsoidResult res = ellipsoidFeasibility(A, b, 2000, x0, Q0);

    if (res.feasible) {
        cout << "Feasible point found: ";
        for (double xi : res.x) cout << xi << " ";
        cout << "\n";
    } else {
        cout << "No feasible point found (or numerical failure).\n";
    }

    // LP via binary search: maximize c^T x, c = (3, 2)
    vector<double> c = {3, 2};
    double lo = -10.0, hi = 20.0; // bounds for objective
    vector<double> bestX;
    for (int it = 0; it < 40; ++it) {
        double mid = 0.5 * (lo + hi);
        auto [x0b, Q0b] = initialEllipsoid(n, 10.0);
        EllipsoidResult r2 = ellipsoidFeasibility(A, b, 2000, x0b, Q0b, c, mid);
        if (r2.feasible) { lo = mid; bestX = r2.x; }
        else { hi = mid; }
    }
    cout << "Approx max c^T x ≈ " << lo << "\n";
    if (!bestX.empty()) {
        cout << "Point: "; for (double xi : bestX) cout << xi << " "; cout << "\n";
    }
    return 0;
}
```
## Ellipsoid Method (C#)
```cpp
using System;
using System.Collections.Generic;

public class EllipsoidResult {
    public bool Feasible;
    public double[] X;
}

public class Ellipsoid {
    const double EPS = 1e-9;

    static int FindViolation(double[][] A, double[] b, double[] x) {
        int m = A.Length;
        for (int i = 0; i < m; i++) {
            double val = 0.0;
            for (int j = 0; j < x.Length; j++) val += A[i][j] * x[j];
            if (val > b[i] + 1e-12) return i;
        }
        return -1;
    }

    static int ObjectiveViolation(double[] c, double t, double[] x) {
        if (c == null) return -1;
        double val = 0.0;
        for (int i = 0; i < x.Length; i++) val += c[i] * x[i];
        if (val < t - 1e-12) return 1;
        return -1;
    }

    public static EllipsoidResult Feasibility(
        double[][] A, double[] b, int maxIter,
        double[] x0, double[][] Q0,
        double[] c = null, double t = double.NegativeInfinity
    ) {
        int n = x0.Length;
        double[] x = (double[])x0.Clone();
        double[][] Q = new double[n][];
        for (int i = 0; i < n; i++) {
            Q[i] = (double[])Q0[i].Clone();
        }

        for (int iter = 0; iter < maxIter; iter++) {
            int vi = FindViolation(A, b, x);
            int objCut = ObjectiveViolation(c, t, x);
            if (vi == -1 && objCut == -1) {
                return new EllipsoidResult { Feasible = true, X = x };
            }

            double[] g = new double[n];
            if (objCut != -1) {
                for (int i = 0; i < n; i++) g[i] = -c[i];
            } else {
                for (int i = 0; i < n; i++) g[i] = A[vi][i];
            }

            double[] Qg = new double[n];
            for (int i = 0; i < n; i++)
                for (int j = 0; j < n; j++)
                    Qg[i] += Q[i][j] * g[j];

            double denom = 0.0;
            for (int i = 0; i < n; i++) denom += g[i] * Qg[i];
            if (denom <= EPS) break;

            double s = Math.Sqrt(denom);
            double[] u = new double[n];
            for (int i = 0; i < n; i++) u[i] = Qg[i] / s;

            for (int i = 0; i < n; i++) x[i] -= u[i] / (n + 1);

            double[] uTQ = new double[n];
            for (int j = 0; j < n; j++)
                for (int k = 0; k < n; k++)
                    uTQ[j] += u[k] * Q[k][j];

            double alpha = (double)(n * n) / (double)(n * n - 1);
            double beta = 2.0 / (n + 1);

            for (int i = 0; i < n; i++) {
                for (int j = 0; j < n; j++) {
                    double uOuterQ = u[i] * uTQ[j];
                    Q[i][j] = alpha * (Q[i][j] - beta * uOuterQ);
                }
            }

            double traceQ = 0.0;
            for (int i = 0; i < n; i++) traceQ += Q[i][i];
            if (traceQ < 1e-12) break;
        }
        return new EllipsoidResult { Feasible = false, X = null };
    }

    public static (double[] x0, double[][] Q0) InitialEllipsoid(int n, double R) {
        double[] x0 = new double[n];
        double[][] Q0 = new double[n][];
        for (int i = 0; i < n; i++) {
            x0[i] = 0.0;
            Q0[i] = new double[n];
            Q0[i][i] = R * R;
        }
        return (x0, Q0);
    }

    public static void Main() {
        // A x <= b feasibility
        double[][] A = {
            new double[] {1, 1},
            new double[] {2, 1}
        };
        double[] b = {5, 6};

        var init = InitialEllipsoid(2, 10.0);
        var res = Feasibility(A, b, 2000, init.x0, init.Q0);
        if (res.Feasible) {
            Console.Write("Feasible point: ");
            foreach (var xi in res.X) Console.Write(xi + " ");
            Console.WriteLine();
        } else {
            Console.WriteLine("No feasible point found (or numerical failure).");
        }

        // LP via binary search: maximize c^T x
        double[] c = {3, 2};
        double lo = -10.0, hi = 20.0;
        double[] bestX = null;
        for (int it = 0; it < 40; it++) {
            double mid = 0.5 * (lo + hi);
            var init2 = InitialEllipsoid(2, 10.0);
            var r2 = Feasibility(A, b, 2000, init2.x0, init2.Q0, c, mid);
            if (r2.Feasible) { lo = mid; bestX = r2.X; }
            else { hi = mid; }
        }
        Console.WriteLine("Approx max c^T x ≈ " + lo);
        if (bestX != null) {
            Console.Write("Point: ");
            foreach (var xi in bestX) Console.Write(xi + " ");
            Console.WriteLine();
        }
    }
}
```


##  Complexity Analysis

### Time Complexity
The Ellipsoid Method runs in **polynomial time** relative to the input size, which was its groundbreaking contribution:

- Each iteration involves updating the ellipsoid parameters (center and shape matrix).  
- The number of iterations is bounded polynomially in the number of variables and the bit-length of the input coefficients.  
- Specifically: **O(n² · L)**, where:  
  - *n* = number of variables  
  - *L* = input size in bits  

This was the **first polynomial-time algorithm for linear programming (LP)**, proving that LP belongs to the class **P**.

---

### Space Complexity
- Requires storage of ellipsoid parameters:  
  - **Center vector** (dimension n)  
  - **Shape matrix Q** (n × n, positive definite)  
- Memory usage: **O(n²)** for the matrix representation.  
- Overall: linear in problem dimension, quadratic in storage.

---

##  Comparison with Other Algorithms

| Algorithm       | Worst-case Complexity | Practical Performance | Notes                          |
|-----------------|-----------------------|-----------------------|--------------------------------|
| **Simplex**     | Exponential           | Very fast in practice | Basis of most LP solvers       |
| **Ellipsoid**   | Polynomial (O(n²L))   | Slow in practice      | First polynomial-time proof    |
| **Interior-point** | Polynomial (O(n³L)) | Fast and robust       | Preferred in modern solvers    |

**Interpretation:**  
- Simplex dominates in practice despite exponential worst-case.  
- Ellipsoid is slower but historically important for theory.  
- Interior-point methods balance theory and practice, becoming the standard in modern optimization.

---

##  Key Concepts Recap
- **Ellipsoid:** A generalized ellipse in n dimensions, used to approximate feasible regions.  
- **Separating hyperplane:** Derived from violated constraints, used to cut the ellipsoid.  
- **Volume reduction:** Each iteration shrinks the ellipsoid, ensuring convergence.  
- **Polynomial guarantee:** First algorithm to prove LP solvability in polynomial time.  
- **Practical inefficiency:** Despite theoretical importance, slower than Simplex or interior-point methods.

---

##  Example Walkthrough (Conceptual)
Suppose we want to solve:  
```
maximize cᵀx
subject to A x ≤ b
```

1. **Initialization:**  
   Start with a large ellipsoid covering all feasible solutions.

2. **Iteration:**  
   - Check if the center x is feasible.  
   - If not, find a violated constraint aᵢᵀx > bᵢ.  
   - Construct a separating hyperplane and update the ellipsoid.

3. **Convergence:**  
   - Each update reduces ellipsoid volume.  
   - After polynomially many steps, the ellipsoid approximates the feasible region tightly.  
   - The center provides an approximate solution.

---

## Conclusion
The **Ellipsoid Method** is a landmark in optimization theory:

- It proved that **linear programming belongs to P**, resolving a major open question in computational complexity.  
- While not competitive in practice, it laid the foundation for **modern convex optimization** and **interior-point methods**.  
- Its importance is primarily **theoretical**, showing that optimization problems can be solved efficiently in principle, even if better algorithms are used in practice.  

**Legacy:**  
- Simplex: practical dominance.  
- Ellipsoid: theoretical breakthrough.  
- Interior-point: modern balance of theory and practice.
  
---




