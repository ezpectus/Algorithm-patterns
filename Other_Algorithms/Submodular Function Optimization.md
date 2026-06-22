# Submodular Function Optimization

## Origin & Motivation

**Submodular functions** are set functions `f: 2^E → ℝ` satisfying a **diminishing returns** property: the marginal value of adding an element to a set can only decrease as the set grows. Formally, for all `A ⊆ B ⊆ E` and `x ∉ B`:

```
f(A ∪ {x}) - f(A)  ≥  f(B ∪ {x}) - f(B)
```

Equivalently: `f(A) + f(B) ≥ f(A ∪ B) + f(A ∩ B)` for all `A, B ⊆ E`.

Submodular functions were studied by Edmonds (1970) in the context of polymatroids, and the algorithmic theory was developed by Cunningham, Schrijver, Lovász, and Fujishige in the 1980s–2000s.

**The problems they solve:**
- **Maximization** — most NP-hard in general, but the greedy algorithm gives a `(1 - 1/e)`-approximation for *monotone* submodular functions under cardinality constraints, and `1/2`-approximation for unconstrained maximization. These are tight: no polynomial algorithm can do better unless P=NP.
- **Minimization** — solvable in polynomial time (Cunningham 1985, Schrijver 2000, Iwata-Fleischer-Fujishige 2001). This is the deepest result — unlike maximization, exact minimization is polynomial.

**Why they matter:** Hundreds of practical objectives in machine learning, combinatorial optimisation, and operations research are submodular: coverage, cuts, mutual information, log-determinants, facility location, feature selection — all are submodular, so their maximization and minimization inherit the corresponding guarantees.

---

## Where It Is Used

- **Maximum coverage** — cover as many items as possible with `k` sets (e.g. facility placement, sensor deployment). The greedy algorithm gives `(1-1/e) ≈ 0.632`-approximation.
- **Max cut** — partition graph nodes to maximise cut edges. The cut function is submodular; local search gives `1/2`-approximation.
- **Feature selection / information gain** — mutual information and entropy functions are submodular; greedy subset selection is provably near-optimal.
- **Document summarization** — coverage and diversity objectives are submodular; greedy gives guaranteed quality bounds.
- **Minimum cut / graph cuts** — cut functions are submodular; minimization finds minimum cuts in polynomial time.
- **Influence maximization** — the expected number of influenced nodes in a social network is monotone submodular; greedy gives `(1-1/e)` approximation.
- **MAP inference in graphical models** — submodular energy functions allow polynomial-time inference via graph cuts (QPBO, graph-cut based MAP).

---

## Core Concepts

### Submodularity and Diminishing Returns

```
Marginal value of x given set S:
    Δ(x | S) = f(S ∪ {x}) - f(S)

Submodularity ⟺ Δ(x | A) ≥ Δ(x | B)  for all A ⊆ B, x ∉ B
(adding x to a smaller context is at least as valuable as adding it
 to a larger context — the hallmark of diminishing returns)
```

### Monotone vs Non-Monotone

```
Monotone submodular:    f(A) ≤ f(B) whenever A ⊆ B
  (adding elements never hurts)
  Examples: coverage, rank functions, probability of union

Non-monotone submodular: neither monotone nor antimonotone
  Examples: graph cuts, mutual information, log-determinant
```

### Common Submodular Functions

```
Coverage:     f(S) = |⋃_{i∈S} C_i|         (monotone, submodular)
  Marginal of x: number of new items covered by x given S

Cut function: f(S) = |{(u,v) ∈ E : u∈S, v∉S}|   (non-monotone, submodular)
  Marginal of x: #(x's neighbours outside S) - #(x's neighbours inside S)

Rank function of a matroid: f(S) = rank(S)        (monotone, submodular)
  Marginal of x: 1 if S+x is independent, 0 otherwise

Log-determinant: f(S) = log det(I + A_S)           (monotone, submodular)
  for a PSD matrix A; measures information content of feature subset S

Weighted coverage: f(S) = sum of weights of covered items   (monotone, submodular)

Entropy: H(X_S) for a set of random variables X_S            (monotone, submodular)
Mutual information: I(X_S; Y) as function of S               (monotone, submodular)
```

### The Polymatroid

A submodular function `f` with `f(∅) = 0` and `f` monotone defines a **polymatroid** — a polytope analogous to the matroid polytope, but with fractional points. The polymatroid `P(f) = {x ≥ 0 : x(S) ≤ f(S) for all S ⊆ E}` plays the same role for submodular functions that the matroid polytope plays for matroids.

---

## Algorithms

### 1. Greedy Maximization — `(1 - 1/e)` Approximation

For **monotone** submodular `f` with `f(∅) = 0`, subject to cardinality constraint `|S| ≤ k`:

```
GreedyMaximize(E, f, k):
    S = ∅
    for iter = 1 to k:
        e* = argmax_{e ∉ S} Δ(e | S)    // element with max marginal gain
        if Δ(e* | S) ≤ 0: break          // no improvement possible
        S = S ∪ {e*}
    return S
```

**Approximation ratio:** `f(S_greedy) ≥ (1 - 1/e) · f(S_opt) ≈ 0.632 · OPT`

**Proof sketch:** Let `S_opt = {o_1,...,o_k}` (optimal). After `i` greedy steps, the remaining gain satisfies `f(S_opt) - f(S_i) ≤ Δ(e* | S_i) · (k - |S_i|) / k`... summing gives the `(1-1/e)` bound by a coupling argument.

**Tightness:** No polynomial algorithm achieves ratio better than `(1-1/e)` for cardinality-constrained monotone submodular maximization unless P=NP (Feige 1998).

**Time:** `O(k · n)` oracle calls.

### 2. Double Greedy — `1/3` Approximation

For **general (non-monotone)** submodular, **unconstrained** maximization:

```
DoubleGreedy(E, f):
    Shuffle E randomly into order e_1, ..., e_n
    X = ∅     (starts empty)
    Y = E     (starts full)
    for i = 1 to n:
        a = Δ(e_i | X)           // gain of adding e_i to X
        b = f(Y) - f(Y \ {e_i})  // loss of removing e_i from Y
        if a ≥ b:
            X = X ∪ {e_i}        // both agree: add e_i
        else:
            Y = Y \ {e_i}        // both agree: discard e_i
    return X   (= Y at termination)
```

**Approximation ratio:** `E[f(S_dg)] ≥ (1/3) · f(S_opt)`

The deterministic variant gives `1/3`; a randomized version achieves `1/3`. A more sophisticated algorithm (RANDOMIZED-DOUBLE-GREEDY by Buchbinder et al. 2012) achieves `1/2`.

**Time:** `O(n)` oracle calls.

### 3. Local Search — `1/2` Approximation

For **general** submodular, **unconstrained** maximization:

```
LocalSearch(E, f):
    S = ∅  (or any initial set)
    repeat:
        for each e ∈ E:
            if f(S △ {e}) > f(S):     // flip e improves objective
                S = S △ {e}
    until no improving flip exists
    return S
```

**Approximation ratio:** Any local optimum satisfies `f(S) ≥ (1/2) · f(S_opt)`.

**Proof:** Let `S*` be the global optimum. Since `S` is locally optimal, no single flip improves `f`. Sum the "no-improvement" conditions over all elements of `S △ S*`; the two halves of the sum each bound `f(S)` in terms of `f(S*)`.

**Time:** `O(n)` per round, `O(n²)` rounds in the worst case.

### 4. Submodular Minimization — Exact Polynomial Time

Submodular minimization is solvable in polynomial time. Three landmark algorithms:

```
Cunningham (1985): O(n^5 · EO) — first strongly polynomial algorithm
Schrijver (2000):  O(n^6 · EO) — simpler, based on minimum norm point in polymatroid
Iwata-Fleischer-Fujishige (2001): O(n^7 · EO) — alternative strongly polynomial

where EO = time for one oracle evaluation.

Modern implementations (e.g. in SFO toolbox, LEMON library) run much
faster in practice using the Minimum Norm Point (MNP) algorithm:
    find the point of minimum Euclidean norm in the base polytope B(f)
    the support of this point gives the minimizing set S.
```

For small `n`, brute force `O(2^n · EO)` is the simplest correct approach.

---

## Complexity Summary

| Problem | Constraint | Algorithm | Ratio | Oracle Calls |
|---|---|---|---|---|
| Monotone max | `\|S\| ≤ k` | Greedy | `1-1/e` | O(kn) |
| Monotone max | Matroid | Greedy (on matroid) | `1-1/e` | O(r²n) |
| General max | Unconstrained | Local search | `1/2` | O(n³) |
| General max | Unconstrained | Double greedy | `1/3 (det) / 1/2 (rand)` | O(n) |
| General max | `\|S\| ≤ k` | Continuous greedy + rounding | `1-1/e` | O(n³/ε) |
| Minimization | Unconstrained | MNP / Schrijver | Exact | O(n⁶·EO) |
| Minimization | Unconstrained | Brute force | Exact | O(2^n·EO) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// SUBMODULAR FUNCTION INTERFACE
// All algorithms operate through: eval(S) and marginal(S, e)
// ================================================================

// ---- Coverage Function: f(S) = |union of sets covered by S| ----
// Monotone submodular.
struct CoverageFunction {
    int n, m; // n elements, m items to cover
    vector<vector<int>> covers; // covers[i] = items covered by element i

    CoverageFunction(int n, int m, vector<vector<int>> covers)
        : n(n), m(m), covers(covers) {}

    double eval(const vector<bool>& S) const {
        vector<bool> covered(m, false);
        for (int i = 0; i < n; i++)
            if (S[i]) for (int item : covers[i]) covered[item] = true;
        return (double)count(covered.begin(), covered.end(), true);
    }

    // Marginal gain of adding element e to set S
    double marginal(const vector<bool>& S, int e) const {
        if (S[e]) return 0.0;
        vector<bool> covered(m, false);
        for (int i = 0; i < n; i++)
            if (S[i]) for (int item : covers[i]) covered[item] = true;
        int gain = 0;
        for (int item : covers[e]) if (!covered[item]) gain++;
        return (double)gain;
    }
};

// ---- Cut Function: f(S) = |edges with exactly one endpoint in S| ----
// Non-monotone submodular.
struct CutFunction {
    int n;
    vector<pair<int,int>> edges;

    CutFunction(int n, vector<pair<int,int>> edges) : n(n), edges(edges) {}

    double eval(const vector<bool>& S) const {
        double cut = 0;
        for (auto [u,v] : edges) if (S[u] != S[v]) cut++;
        return cut;
    }

    double marginal(const vector<bool>& S, int e) const {
        // Flip e and compute change
        double before = eval(S);
        vector<bool> Snew = S; Snew[e] = !Snew[e];
        return eval(Snew) - before;
    }
};

// ================================================================
// GREEDY MAXIMIZATION — (1 - 1/e) approximation
// For MONOTONE submodular f, cardinality constraint |S| ≤ k.
// ================================================================
template<typename F>
vector<bool> greedy_maximize(int n, int k, F& f) {
    vector<bool> S(n, false);
    for (int iter = 0; iter < k; iter++) {
        int best_elem = -1;
        double best_gain = -1e18;
        for (int e = 0; e < n; e++) {
            if (S[e]) continue;
            double gain = f.marginal(S, e);
            if (gain > best_gain) { best_gain = gain; best_elem = e; }
        }
        if (best_elem == -1 || best_gain <= 0) break;
        S[best_elem] = true;
    }
    return S;
}

// ================================================================
// DOUBLE GREEDY — 1/3 approximation (randomized)
// For GENERAL submodular, UNCONSTRAINED maximization.
// Processes elements in random order; maintains X (empty to full)
// and Y (full to empty) simultaneously, choosing each element's
// fate based on comparing marginal gain in X vs marginal loss in Y.
// ================================================================
template<typename F>
vector<bool> double_greedy(int n, F& f, mt19937& rng) {
    vector<int> order(n); iota(order.begin(), order.end(), 0);
    shuffle(order.begin(), order.end(), rng);

    vector<bool> X(n, false); // starts empty
    vector<bool> Y(n, true);  // starts full

    for (int e : order) {
        double a = f.marginal(X, e);    // gain of adding e to X

        // b = f(Y \ {e}) - f(Y) = loss of removing e from Y
        vector<bool> Yme = Y; Yme[e] = false;
        double b = f.eval(Yme) - f.eval(Y);

        if (a >= b) {
            X[e] = true;  // include e: X gains e, Y keeps e
        } else {
            Y[e] = false; // exclude e: X skips e, Y loses e
        }
    }
    return X; // at termination X == Y
}

// ================================================================
// LOCAL SEARCH — 1/2 approximation
// For GENERAL submodular, UNCONSTRAINED maximization.
// Repeatedly flips the element giving the greatest improvement.
// ================================================================
template<typename F>
vector<bool> local_search(int n, F& f, int max_iters = 10000) {
    vector<bool> S(n, false);
    bool improved = true;
    int iter = 0;
    while (improved && iter < max_iters) {
        improved = false; iter++;
        double cur = f.eval(S);
        double best_delta = 0; int best_e = -1;
        for (int e = 0; e < n; e++) {
            vector<bool> Snew = S; Snew[e] = !Snew[e];
            double delta = f.eval(Snew) - cur;
            if (delta > best_delta) { best_delta = delta; best_e = e; }
        }
        if (best_e != -1) { S[best_e] = !S[best_e]; improved = true; }
    }
    return S;
}

// ================================================================
// BRUTE FORCE MAXIMIZATION / MINIMIZATION — exact O(2^n)
// For small n (≤ 20), use as ground truth.
// ================================================================
template<typename F>
pair<vector<bool>, double> brute_max(int n, F& f, int k = -1) {
    double best = -1e18; vector<bool> best_S(n, false);
    for (int mask = 0; mask < (1<<n); mask++) {
        if (k >= 0 && __builtin_popcount(mask) > k) continue;
        vector<bool> S(n, false);
        for (int i = 0; i < n; i++) if (mask>>i&1) S[i] = true;
        double v = f.eval(S);
        if (v > best) { best = v; best_S = S; }
    }
    return {best_S, best};
}

template<typename F>
pair<vector<bool>, double> brute_min(int n, F& f) {
    double best = 1e18; vector<bool> best_S(n, false);
    for (int mask = 0; mask < (1<<n); mask++) {
        vector<bool> S(n, false);
        for (int i = 0; i < n; i++) if (mask>>i&1) S[i] = true;
        double v = f.eval(S);
        if (v < best) { best = v; best_S = S; }
    }
    return {best_S, best};
}

// ================================================================
// VERIFY SUBMODULARITY — check f(A)+f(B) >= f(A∪B)+f(A∩B)
// ================================================================
template<typename F>
bool verify_submodular(int n, F& f, int trials = 500, int seed = 42) {
    mt19937 rng(seed);
    for (int t = 0; t < trials; t++) {
        vector<bool> A(n,false), B(n,false);
        for (int i = 0; i < n; i++) { A[i]=rng()%2; B[i]=rng()%2; }
        vector<bool> AuB(n), AiB(n);
        for (int i = 0; i < n; i++) { AuB[i]=A[i]||B[i]; AiB[i]=A[i]&&B[i]; }
        double lhs = f.eval(A) + f.eval(B);
        double rhs = f.eval(AuB) + f.eval(AiB);
        if (lhs < rhs - 1e-9) return false;
    }
    return true;
}

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- Maximum Coverage: greedy vs brute force ----
    {
        printf("=== Maximum Coverage (Monotone, Greedy) ===\n");
        CoverageFunction cf(5, 8,
            {{0,1,2}, {1,2,3}, {3,4,5}, {5,6,7}, {0,4,6}});
        printf("Is submodular: %s\n", verify_submodular(5,cf)?"YES":"NO");

        for (int k = 1; k <= 4; k++) {
            auto Sg = greedy_maximize(5, k, cf);
            double vg = cf.eval(Sg);
            auto [Sb, vb] = brute_max(5, cf, k);
            printf("k=%d  greedy=%.0f  exact=%.0f  ratio=%.3f  (bound=%.3f)\n",
                   k, vg, vb, vb>0?vg/vb:1.0, 1.0-1.0/M_E);
        }
    }

    // ---- Max Cut: double greedy + local search ----
    {
        printf("\n=== Max Cut (Non-Monotone, Double Greedy + Local Search) ===\n");
        int n = 6;
        vector<pair<int,int>> edges;
        for (int i=0;i<n;i++) for (int j=i+1;j<n;j++) edges.push_back({i,j});
        CutFunction cf(n, edges);
        printf("Is submodular: %s\n", verify_submodular(n,cf)?"YES":"NO");

        auto [Sb,vb] = brute_max(n, cf);
        printf("Brute force max cut = %.0f\n", vb);

        int trials = 500;
        double sum_dg=0, sum_ls=0;
        for (int t=0; t<trials; t++) {
            mt19937 rng(t);
            sum_dg += cf.eval(double_greedy(n, cf, rng));
            sum_ls += cf.eval(local_search(n, cf));
        }
        printf("Double greedy avg = %.2f  ratio = %.3f  (bound >= 1/3 = %.3f)\n",
               sum_dg/trials, sum_dg/trials/vb, 1.0/3.0);
        printf("Local search avg  = %.2f  ratio = %.3f  (bound >= 1/2 = %.3f)\n",
               sum_ls/trials, sum_ls/trials/vb, 0.5);
    }

    // ---- Submodular minimization: cut function ----
    {
        printf("\n=== Submodular Minimization (Cut Function) ===\n");
        int n = 5;
        vector<pair<int,int>> edges = {{0,1},{1,2},{2,3},{3,4},{0,2},{1,3}};
        CutFunction cf(n, edges);
        auto [Sm, vm] = brute_min(n, cf);
        printf("Min cut = %.0f (achieved at S=empty or S=full for cut function)\n", vm);
        vector<bool> empty(n,false), full(n,true);
        printf("f(empty)=%.0f  f(full)=%.0f\n", cf.eval(empty), cf.eval(full));
        // Find non-trivial minimum cut
        double non_trivial = 1e18; int best_mask=-1;
        for (int mask=1; mask<(1<<n)-1; mask++) {
            vector<bool> S(n,false);
            for(int i=0;i<n;i++) if(mask>>i&1) S[i]=true;
            if (cf.eval(S)<non_trivial){ non_trivial=cf.eval(S); best_mask=mask; }
        }
        printf("Non-trivial min cut = %.0f at mask=%d\n", non_trivial, best_mask);
    }

    // ---- Stress: greedy approximation guarantee ----
    {
        printf("\n=== Stress: Greedy (1-1/e) guarantee, 500 coverage instances ===\n");
        srand(42); int violations = 0;
        double bound = 1.0 - 1.0/M_E;
        for (int trial=0; trial<500; trial++) {
            int n=4+rand()%4, m=6+rand()%6;
            int k=1+rand()%3;
            vector<vector<int>> covers(n);
            for (int i=0;i<n;i++){
                int nc=1+rand()%4;
                for (int j=0;j<nc;j++) covers[i].push_back(rand()%m);
            }
            CoverageFunction cf(n,m,covers);
            double vg = cf.eval(greedy_maximize(n,k,cf));
            auto [Sb,vb] = brute_max(n,cf,k);
            if (vb > 1e-9 && vg/vb < bound - 0.02) violations++;
        }
        printf("Violations (with 0.02 slack): %d (expect ~0)\n", violations);
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ================================================================
// SUBMODULAR FUNCTION INTERFACE
// ================================================================
public interface ISubmodular
{
    double Eval(bool[] S);
    double Marginal(bool[] S, int e); // gain of adding/flipping e
}

// ================================================================
// COVERAGE FUNCTION — f(S) = |union of sets covered by S|
// ================================================================
public class CoverageFunction : ISubmodular
{
    private readonly int _n, _m;
    private readonly int[][] _covers;

    public CoverageFunction(int n, int m, int[][] covers)
    { _n=n; _m=m; _covers=covers; }

    public double Eval(bool[] S)
    {
        var covered = new bool[_m];
        for (int i=0;i<_n;i++)
            if (S[i]) foreach (int item in _covers[i]) covered[item]=true;
        return covered.Count(c=>c);
    }

    public double Marginal(bool[] S, int e)
    {
        if (S[e]) return 0;
        var covered = new bool[_m];
        for (int i=0;i<_n;i++)
            if (S[i]) foreach (int item in _covers[i]) covered[item]=true;
        return _covers[e].Count(item => !covered[item]);
    }
}

// ================================================================
// CUT FUNCTION — f(S) = edges crossing S / E\S
// ================================================================
public class CutFunction : ISubmodular
{
    private readonly (int u, int v)[] _edges;
    private readonly int _n;

    public CutFunction(int n, (int u, int v)[] edges) { _n=n; _edges=edges; }

    public double Eval(bool[] S) => _edges.Count(e => S[e.u] != S[e.v]);

    public double Marginal(bool[] S, int e)
    {
        var Snew = (bool[])S.Clone(); Snew[e] = !Snew[e];
        return Eval(Snew) - Eval(S);
    }
}

// ================================================================
// GREEDY MAXIMIZATION — (1-1/e) for monotone submodular, |S|<=k
// ================================================================
public static class SubmodularOpt
{
    public static bool[] GreedyMaximize(int n, int k, ISubmodular f)
    {
        var S = new bool[n];
        for (int iter=0; iter<k; iter++)
        {
            int bestElem=-1; double bestGain=double.NegativeInfinity;
            for (int e=0;e<n;e++) {
                if (S[e]) continue;
                double g = f.Marginal(S, e);
                if (g > bestGain) { bestGain=g; bestElem=e; }
            }
            if (bestElem==-1 || bestGain<=0) break;
            S[bestElem]=true;
        }
        return S;
    }

    // ================================================================
    // DOUBLE GREEDY — 1/3 approximation, general submodular
    // ================================================================
    public static bool[] DoubleGreedy(int n, ISubmodular f, Random rng)
    {
        var order = Enumerable.Range(0,n).OrderBy(_=>rng.Next()).ToArray();
        var X = new bool[n];        // empty
        var Y = Enumerable.Repeat(true, n).ToArray(); // full

        foreach (int e in order)
        {
            double a = f.Marginal(X, e);
            var Yme = (bool[])Y.Clone(); Yme[e]=false;
            double b = f.Eval(Yme) - f.Eval(Y);
            if (a >= b) X[e]=true;
            else        Y[e]=false;
        }
        return X;
    }

    // ================================================================
    // LOCAL SEARCH — 1/2 approximation, general submodular
    // ================================================================
    public static bool[] LocalSearch(int n, ISubmodular f, int maxIters=10000)
    {
        var S = new bool[n];
        bool improved=true; int iter=0;
        while (improved && iter<maxIters)
        {
            improved=false; iter++;
            double cur=f.Eval(S);
            double bestDelta=0; int bestE=-1;
            for (int e=0;e<n;e++) {
                var Snew=(bool[])S.Clone(); Snew[e]=!Snew[e];
                double d=f.Eval(Snew)-cur;
                if (d>bestDelta) { bestDelta=d; bestE=e; }
            }
            if (bestE!=-1) { S[bestE]=!S[bestE]; improved=true; }
        }
        return S;
    }

    // ================================================================
    // BRUTE FORCE MAX (for n <= 20, with optional cardinality bound k)
    // ================================================================
    public static (bool[] S, double val) BruteMax(int n, ISubmodular f, int k=-1)
    {
        double best=double.NegativeInfinity; bool[] bestS=new bool[n];
        for (int mask=0; mask<(1<<n); mask++)
        {
            if (k>=0 && CountBits(mask)>k) continue;
            var S=new bool[n];
            for(int i=0;i<n;i++) if((mask>>i&1)>0) S[i]=true;
            double v=f.Eval(S); if(v>best){best=v; bestS=S;}
        }
        return (bestS, best);
    }

    private static int CountBits(int x){ int c=0; while(x>0){c+=x&1;x>>=1;} return c; }

    public static void Main()
    {
        // Coverage
        var cf = new CoverageFunction(5, 8, new[]{ new[]{0,1,2}, new[]{1,2,3},
            new[]{3,4,5}, new[]{5,6,7}, new[]{0,4,6} });
        for (int k=1; k<=4; k++) {
            var Sg = GreedyMaximize(5, k, cf);
            var (Sb,vb) = BruteMax(5, cf, k);
            double vg = cf.Eval(Sg);
            Console.WriteLine($"k={k}: greedy={vg} exact={vb} ratio={vg/vb:F3}");
        }

        // Max cut
        var edges = new (int,int)[]{ (0,1),(0,2),(0,3),(1,2),(1,3),(2,3) };
        var cut = new CutFunction(4, edges);
        var rng = new Random(42);
        var Sdg = DoubleGreedy(4, cut, rng);
        var Sls = LocalSearch(4, cut);
        var (_,vopt) = BruteMax(4, cut);
        Console.WriteLine($"MaxCut: opt={vopt} dg={cut.Eval(Sdg):F0} ls={cut.Eval(Sls):F0}");
    }
}
```

---

## The Diminishing Returns Guarantee — Why Greedy Works

The `(1-1/e)` bound for greedy on monotone submodular functions is one of the most elegant results in approximation algorithms. The argument:

```
Let S* = {o_1,...,o_k} be the optimal set.  Let S_i be the greedy set after i steps.

By submodularity (each o_j can only help less given S_i than given ∅):
    f(S*) - f(S_i) ≤ Σ_j Δ(o_j | S_i)   [submodularity: union bound on marginals]
                   ≤ k · max_e Δ(e | S_i)  [there are k elements in S*]
                   = k · Δ(e_{i+1} | S_i)  [greedy chose the best e_{i+1}]

So:  f(S*) - f(S_i) ≤ k · (f(S_{i+1}) - f(S_i))

Let  δ_i = f(S*) - f(S_i).  Then:  δ_{i+1} ≤ δ_i · (1 - 1/k)

By induction:  δ_k ≤ δ_0 · (1-1/k)^k ≤ f(S*) · (1/e)

Therefore:  f(S_k) = f(S*) - δ_k ≥ f(S*) · (1 - 1/e)
```

The bound `(1-1/k)^k ≤ 1/e` is just the definition of `e` in the limit.

---

## Pitfalls

- **Greedy only gives `(1-1/e)` for MONOTONE submodular** — for non-monotone functions, greedy has no approximation guarantee. Use double greedy or local search for non-monotone functions.
- **Marginal gain computation must not modify `S`** — `Δ(e | S) = f(S ∪ {e}) - f(S)` requires two oracle calls. A common bug is modifying the set `S` in place during the loop and computing marginals against the modified set. Always copy `S` before evaluation, or implement `marginal` using the current `S` without side effects.
- **Local search can be slow** — each round of local search costs `O(n)` oracle calls and there may be `O(n·f_max)` rounds. For large `n`, use a lazy evaluation (cache marginal gains and update only when the set changes) to speed up individual rounds.
- **Double greedy requires a fresh random shuffle each call** — calling `double_greedy` multiple times with the same order gives the same result. The `1/3` guarantee holds in expectation over the random shuffle; for a single deterministic run the ratio can be worse.
- **Verifying submodularity is probabilistic** — the `verify_submodular` function checks `f(A)+f(B) ≥ f(A∪B)+f(A∩B)` for random pairs. A function that fails for only one pair out of exponentially many will pass with high probability. Exact verification requires checking all `O(2^{2n})` pairs — infeasible for large `n`. In practice, test on carefully chosen adversarial pairs (e.g. `A ⊆ B` with a single extra element).
- **Coverage function marginal must check items NOT already covered** — `Δ(e | S)` counts items covered by `e` that are NOT covered by any element of `S`. Computing this requires building the full coverage of `S` first (O(n·m) per marginal computation). Caching the current coverage bitvector across greedy iterations reduces total cost to `O(k·n·m)` instead of `O(k·n²·m)`.
- **Submodular minimization is polynomial but NOT practical via brute force** — the polynomial-time algorithms (MNP, Schrijver) have complexity `O(n^6)` or higher, making them impractical for `n > 100`. For practical submodular minimization on graphs (min cut, graph cuts for MAP inference), use specialised algorithms (max-flow for s-t cuts, graph-cut libraries for energy minimization).

---

## Conclusion

Submodular Function Optimization is the **combinatorial analogue of convex optimization** — submodular minimization is polynomial (like convex minimization) while submodular maximization is NP-hard but admits strong approximations (like maximizing a concave function over a discrete domain):

- **Greedy maximization** gives the optimal `(1-1/e)`-approximation for monotone submodular under cardinality constraints in `O(kn)` oracle calls — simply choose the element with the highest marginal gain at each step.
- **Local search** gives a `1/2`-approximation for unconstrained general submodular maximization — any local optimum is within a factor of 2 of the global optimum.
- **Double greedy** gives a `1/3` deterministic (or `1/2` randomized) approximation for unconstrained maximization in `O(n)` oracle calls — the most efficient algorithm for this setting.
- **Exact minimization** is polynomial via the Minimum Norm Point algorithm on the base polytope, though the polynomial degree is high; graph-cut based methods solve special cases in near-linear time.

**Key takeaway:** a function is submodular if and only if it has the diminishing returns property. Recognising this structure in your objective immediately unlocks a toolkit of approximation algorithms with provable guarantees — greedy for monotone maximization, local search or double greedy for general maximization, and flow-based or MNP algorithms for exact minimization.
