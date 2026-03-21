# Approximation Algorithms — LP Rounding and Greedy Proofs

## Origin & Motivation

For NP-hard optimization problems, finding the exact optimum is intractable in the worst case. Approximation algorithms produce solutions guaranteed to be within a **multiplicative factor** of the optimum in polynomial time. This factor — the **approximation ratio** — is the central quality measure.

Two systematic design paradigms dominate the field:

- **LP Relaxation + Rounding** — solve the linear programming relaxation of the integer program (polynomial via simplex / interior point), then round the fractional solution to an integer one while controlling the loss.
- **Greedy with analysis** — design a greedy heuristic and prove its approximation ratio via a charging argument, potential function, or primal-dual analysis.

Both paradigms require a **dual certificate** or **lower bound** on OPT to prove the ratio. The LP relaxation provides such a bound directly: `LP_OPT <= OPT` for minimization. Greedy analyses build the bound by comparing each greedy decision to an averaging or exchange argument.

---

## Approximation Ratio — Definition

For a minimization problem, algorithm A has approximation ratio ρ if for all instances:

```
A(I) <= ρ * OPT(I)
```

For maximization: `A(I) >= (1/ρ) * OPT(I)`, or equivalently the ratio is expressed as `OPT/A <= ρ`.

A **PTAS** (Polynomial Time Approximation Scheme) achieves ratio `(1 + ε)` for any `ε > 0`, with runtime polynomial in n for fixed ε. An **FPTAS** has runtime polynomial in both n and 1/ε.

---

## Where It Is Used

- Vertex Cover, Set Cover (LP rounding)
- Traveling Salesman Problem (Christofides: 3/2)
- Knapsack (FPTAS via DP scaling)
- Scheduling (greedy list scheduling: 2 - 1/m)
- Facility Location (LP + primal-dual)
- Max-Cut (SDP rounding: 0.878)
- Steiner Tree, k-median, k-means

---

## Part 1 — LP Relaxation and Rounding

### General Framework

**Integer Program (IP):**
```
min  c^T x
s.t. Ax >= b
     x in {0,1}^n
```

**LP Relaxation:**
```
min  c^T x
s.t. Ax >= b
     0 <= x <= 1       (relax integrality)
```

LP_OPT ≤ IP_OPT (relaxation is a superset — feasible set is larger for minimization LP).

**Rounding step:** Convert fractional `x*` to integer `x_int` while:
1. Maintaining feasibility (all constraints satisfied).
2. Bounding the cost increase: `c^T x_int <= ρ * c^T x* <= ρ * OPT`.

### Example: Vertex Cover (ratio 2)

**Problem:** Given graph G=(V,E), find minimum subset S ⊆ V such that every edge has at least one endpoint in S.

**LP relaxation:**
```
min  sum_v x_v
s.t. x_u + x_v >= 1    for all edges (u,v)
     0 <= x_v <= 1
```

**Rounding:** Set `x_v = 1` if `x*_v >= 1/2`, else `x_v = 0`.

**Feasibility:** For every edge (u,v), `x*_u + x*_v >= 1` → at least one is `>= 1/2` → it is rounded to 1. ✓

**Ratio:** `sum x_v <= 2 * sum x*_v <= 2 * LP_OPT <= 2 * OPT`. ✓

### Example: Set Cover (ratio ln n)

**Problem:** Universe U of n elements, collection of sets S_1,...,S_m with costs c_i. Find minimum cost subcollection covering all elements.

**LP relaxation:**
```
min  sum_i c_i * y_i
s.t. sum_{i: e in S_i} y_i >= 1    for all e in U
     0 <= y_i <= 1
```

**Randomized rounding (ln n approximation):**

Round each `y*_i` independently with probability `y*_i * ln n`. Each element `e` is uncovered with probability at most:

```
Pr[e uncovered] = prod_{i: e in S_i} (1 - y*_i * ln n)
               <= exp(-sum y*_i * ln n)
               <= exp(-ln n)   (since sum y*_i >= 1)
               = 1/n
```

By union bound, all n elements are covered with probability ≥ 1 - 1 (roughly; repeat O(log n) times to boost). Expected cost = `ln n * LP_OPT <= ln n * OPT`.

### Integrality Gap

The **integrality gap** of an LP relaxation is:

```
max over instances I of: OPT_integer(I) / OPT_LP(I)
```

It is a lower bound on the achievable approximation ratio via LP rounding for this relaxation. Vertex cover LP has integrality gap exactly 2. Set cover LP has integrality gap Θ(log n).

---

## Part 2 — Greedy with Approximation Proofs

### Greedy Vertex Cover (ratio 2)

**Algorithm:** Repeatedly pick any uncovered edge (u,v), add both u and v to the cover, mark all edges incident to u or v as covered.

**Proof:** Let M be the matching formed by picked edges. |M| is a lower bound on OPT (every vertex cover must include at least one endpoint of each matching edge, and matching edges are disjoint). The greedy cover has size `2|M|`. Thus: `greedy = 2|M| <= 2 * OPT`. ✓

### Greedy Set Cover (ratio H_n = 1 + 1/2 + ... + 1/n ≈ ln n)

**Algorithm:** Repeatedly pick the set with the highest **coverage ratio** (uncovered elements per unit cost). Remove covered elements. Repeat.

**Proof (charging):** When the k-th element is covered, the set that covers it has coverage ratio ≥ `(n-k+1)/(OPT_remaining) >= 1/OPT`. The cost charged to this element is at most `OPT / (n - k + 1)`. Total cost:

```
sum_{k=1}^{n} OPT/(n-k+1) = OPT * H_n
```

Thus greedy ≤ H_n * OPT where H_n = 1 + 1/2 + ... + 1/n ≈ ln n. ✓

### Greedy Scheduling — Makespan Minimization (ratio 2)

**Problem:** Assign n jobs with processing times p_j to m machines. Minimize makespan (max load).

**List Scheduling:** Assign each job to the machine with current minimum load.

**Proof:** Let T* be optimal makespan. Let j be the job completing last. Machine i (running j) has load `L_i - p_j <= T*` (all other machines had load ≤ makespan). Also `p_j <= T*`. Thus:

```
makespan = L_i = (L_i - p_j) + p_j <= T* + T* = 2T*
```

Ratio 2. With **LPT (Longest Processing Time first)**: ratio `4/3 - 1/(3m)`.

### Greedy Knapsack (ratio 1/2 via augmentation)

**Problem:** Items with weights w_i and values v_i; knapsack capacity W. Maximize total value.

**Greedy by value density:** Sort by v_i/w_i. Take greedily until next item does not fit. Take `max(greedy solution, single best item)`.

**Proof:** Greedy fractional value = OPT_fractional >= OPT. The rounding loss is at most one item (the item that did not fit). Its value <= OPT. So `max(greedy, best_item) >= OPT/2`. Ratio 1/2.

**FPTAS:** Scale item values: `v'_i = floor(v_i * n/ε / v_max)`. Run exact DP on scaled values in O(n^2/ε). Approximation ratio 1-ε.

---

## Primal-Dual Method

An alternative to rounding: maintain a primal feasible (or infeasible) solution and a **dual** solution, update both simultaneously, and use complementary slackness to relate their costs.

**Schema:**
```
Start: dual y = 0, primal solution empty
While primal infeasible:
    Raise some dual variable y_e until a constraint goes tight
    Add the corresponding primal variable (set / vertex) to solution
    
Proof: primal cost <= ρ * dual cost <= ρ * OPT_LP <= ρ * OPT
```

Used for: Set Cover (ratio H_n), Steiner Tree (ratio 2), Facility Location (ratio 1.5).

---

## Complexity Analysis

| Problem | Naive exact | Approximation | Ratio | Method |
|---|---|---|---|---|
| Vertex Cover | NP-hard | O(V + E) | 2 | LP rounding or greedy matching |
| Set Cover | NP-hard | O(nm) | ln n | Greedy density |
| Knapsack | NP-hard | O(n^2/ε) | 1-ε | FPTAS: DP scaling |
| Makespan (m machines) | NP-hard | O(n log n) | 4/3 | LPT greedy |
| TSP (metric) | NP-hard | O(n^3) | 3/2 | Christofides |
| Max-Cut | NP-hard | O(n^2) | 0.878 | SDP + Goemans-Williamson rounding |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// 1. VERTEX COVER — LP Rounding (ratio 2)
//    Solve LP via greedy half-integrality observation:
//    Optimal LP solution for vertex cover has x*_v in {0, 1/2, 1}.
//    Round all 1/2 values to 1 → ratio 2.
// ================================================================

// Exact LP rounding for vertex cover: use maximal matching instead
// (equivalent to LP rounding, ratio 2, no LP solver needed)
vector<int> vertex_cover_2approx(int n, vector<pair<int,int>>& edges) {
    vector<bool> covered(n, false);
    vector<int> cover;
    for (auto [u, v] : edges) {
        if (!covered[u] && !covered[v]) {
            covered[u] = covered[v] = true;
            cover.push_back(u);
            cover.push_back(v);
        }
    }
    return cover;
}

// ================================================================
// 2. SET COVER — Greedy (ratio H_n ≈ ln n)
// ================================================================
struct SetCover {
    int n; // universe size
    vector<vector<int>> sets;
    vector<double> cost;

    SetCover(int n) : n(n) {}

    void add_set(vector<int> elements, double c) {
        sets.push_back(elements);
        cost.push_back(c);
    }

    // Returns indices of selected sets and total cost
    pair<vector<int>, double> solve() {
        vector<bool> covered(n, false);
        int uncovered = n;
        vector<int> chosen;
        double total_cost = 0.0;

        while (uncovered > 0) {
            double best_ratio = 1e18;
            int best_set = -1;

            for (int i = 0; i < (int)sets.size(); i++) {
                int new_cover = 0;
                for (int e : sets[i])
                    if (!covered[e]) new_cover++;
                if (new_cover == 0) continue;
                double ratio = cost[i] / new_cover;
                if (ratio < best_ratio) {
                    best_ratio = ratio;
                    best_set = i;
                }
            }

            if (best_set == -1) break; // universe not fully coverable

            chosen.push_back(best_set);
            total_cost += cost[best_set];
            for (int e : sets[best_set]) {
                if (!covered[e]) { covered[e] = true; uncovered--; }
            }
        }

        return {chosen, total_cost};
    }
};

// ================================================================
// 3. KNAPSACK — FPTAS (ratio 1-ε via DP scaling)
// ================================================================
struct KnapsackFPTAS {
    vector<int> w, v;
    int W, n;

    KnapsackFPTAS(vector<int> weights, vector<int> values, int capacity)
        : w(weights), v(values), W(capacity), n(weights.size()) {}

    // Returns approximate total value and selected items
    pair<int, vector<int>> solve(double eps) {
        int v_max = *max_element(v.begin(), v.end());
        // Scaling factor K = eps * v_max / n
        double K = eps * v_max / n;
        if (K < 1) K = 1;

        // Scale values
        vector<int> v_scaled(n);
        for (int i = 0; i < n; i++)
            v_scaled[i] = (int)(v[i] / K);

        int V_sum = 0;
        for (int x : v_scaled) V_sum += x;

        // DP: dp[j] = min weight to achieve scaled value exactly j
        vector<int> dp(V_sum + 1, INT_MAX);
        vector<vector<int>> choice(n, vector<int>(V_sum + 1, 0));
        dp[0] = 0;

        for (int i = 0; i < n; i++) {
            for (int j = V_sum; j >= v_scaled[i]; j--) {
                if (dp[j - v_scaled[i]] != INT_MAX &&
                    dp[j - v_scaled[i]] + w[i] < dp[j]) {
                    dp[j] = dp[j - v_scaled[i]] + w[i];
                    choice[i][j] = 1;
                }
            }
        }

        // Find max achievable scaled value within capacity W
        int best_j = 0;
        for (int j = V_sum; j >= 0; j--) {
            if (dp[j] <= W) { best_j = j; break; }
        }

        // Reconstruct (simplified: return total value)
        int total_value = 0;
        int remaining = best_j;
        vector<int> items;
        for (int i = n-1; i >= 0; i--) {
            if (remaining >= v_scaled[i] && choice[i][remaining]) {
                items.push_back(i);
                total_value += v[i];
                remaining -= v_scaled[i];
            }
        }

        return {total_value, items};
    }
};

// ================================================================
// 4. MAKESPAN MINIMIZATION — LPT Greedy (ratio 4/3 - 1/(3m))
// ================================================================
struct LPTScheduling {
    int m; // number of machines

    LPTScheduling(int m) : m(m) {}

    // Returns assignment[job] = machine (0-indexed) and makespan
    pair<vector<int>, long long> solve(vector<int>& jobs) {
        int n = jobs.size();
        vector<int> idx(n);
        iota(idx.begin(), idx.end(), 0);
        // Sort jobs by decreasing processing time (LPT)
        sort(idx.begin(), idx.end(), [&](int a, int b){
            return jobs[a] > jobs[b];
        });

        vector<long long> load(m, 0);
        vector<int> assign(n);
        // Min-heap: {load, machine_id}
        priority_queue<pair<long long,int>,
                       vector<pair<long long,int>>,
                       greater<>> pq;
        for (int i = 0; i < m; i++) pq.push({0, i});

        for (int ji : idx) {
            auto [l, machine] = pq.top(); pq.pop();
            assign[ji] = machine;
            load[machine] += jobs[ji];
            pq.push({load[machine], machine});
        }

        long long makespan = *max_element(load.begin(), load.end());
        return {assign, makespan};
    }
};

// ================================================================
// 5. MAX-CUT — Greedy local search (ratio 1/2)
//    For each vertex: assign to partition that maximizes cut edges.
//    Guaranteed >= OPT/2 because OPT <= total edges.
// ================================================================
struct MaxCutGreedy {
    int n;
    vector<vector<pair<int,int>>> adj; // {neighbor, weight}

    MaxCutGreedy(int n) : n(n), adj(n) {}
    void add_edge(int u, int v, int w=1) {
        adj[u].push_back({v,w}); adj[v].push_back({u,w});
    }

    // Returns partition (0 or 1) for each vertex and cut value
    pair<vector<int>, long long> solve() {
        vector<int> side(n, 0);
        long long cut = 0;

        // Greedy: process vertices, assign to maximize cut
        for (int v = 0; v < n; v++) {
            long long gain0 = 0, gain1 = 0;
            for (auto [u, w] : adj[v]) {
                if (u >= v) continue; // only look at already-assigned
                if (side[u] == 0) gain1 += w;
                else              gain0 += w;
            }
            side[v] = (gain1 > gain0) ? 1 : 0;
        }

        // Compute cut
        for (int v = 0; v < n; v++)
            for (auto [u, w] : adj[v])
                if (side[v] != side[u]) cut += w;
        cut /= 2; // each edge counted twice

        // Local search: flip each vertex if it improves cut
        bool improved = true;
        while (improved) {
            improved = false;
            for (int v = 0; v < n; v++) {
                long long delta = 0;
                for (auto [u, w] : adj[v])
                    delta += (side[v] == side[u] ? +w : -w);
                if (delta > 0) { side[v] ^= 1; cut += delta; improved = true; }
            }
        }

        return {side, cut};
    }
};

// ================================================================
// 6. PRIMAL-DUAL SET COVER SKETCH
//    For each uncovered element, raise its dual variable y_e
//    until some set S_i becomes tight: sum_{e in S_i} y_e = c_i
//    Add that set to the cover.
//    Cost: sum c_i <= H_k * OPT where k = max set size
// ================================================================
pair<vector<int>, double> primal_dual_set_cover(
    int n,
    vector<vector<int>>& sets,
    vector<double>& cost)
{
    vector<double> y(n, 0.0);  // dual variables (one per element)
    vector<bool> covered(n, false);
    vector<bool> set_chosen(sets.size(), false);
    vector<int> chosen;
    double total = 0.0;
    int uncovered = n;

    while (uncovered > 0) {
        // Find uncovered element e
        int e = -1;
        for (int i = 0; i < n; i++) if (!covered[i]) { e = i; break; }
        if (e == -1) break;

        // Raise y_e until some set containing e goes tight
        double delta = 1e18;
        int tight_set = -1;
        for (int i = 0; i < (int)sets.size(); i++) {
            if (set_chosen[i]) continue;
            bool has_e = false;
            double slack = cost[i];
            for (int elem : sets[i]) {
                if (elem == e) has_e = true;
                slack -= y[elem];
            }
            if (has_e && slack < delta) { delta = slack; tight_set = i; }
        }

        y[e] += delta;

        // Add tight set (and possibly other sets that became tight)
        for (int i = 0; i < (int)sets.size(); i++) {
            if (set_chosen[i]) continue;
            double slack = cost[i];
            for (int elem : sets[i]) slack -= y[elem];
            if (fabs(slack) < 1e-9) {
                set_chosen[i] = true;
                chosen.push_back(i);
                total += cost[i];
                for (int elem : sets[i])
                    if (!covered[elem]) { covered[elem] = true; uncovered--; }
            }
        }
    }

    return {chosen, total};
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Vertex Cover
    {
        int n = 5;
        vector<pair<int,int>> edges = {{0,1},{1,2},{2,3},{3,4},{4,0}};
        auto cover = vertex_cover_2approx(n, edges);
        printf("Vertex cover (size %d): ", (int)cover.size());
        for (int v : cover) printf("%d ", v);
        printf("\n");
    }

    // Set Cover
    {
        SetCover sc(6);
        sc.add_set({0,1,2}, 3.0);
        sc.add_set({1,3,4}, 2.0);
        sc.add_set({2,4,5}, 2.5);
        sc.add_set({0,3,5}, 2.0);
        auto [chosen, cost] = sc.solve();
        printf("Set cover cost=%.1f, sets: ", cost);
        for (int i : chosen) printf("%d ", i);
        printf("\n");
    }

    // Knapsack FPTAS
    {
        vector<int> w = {2, 3, 4, 5};
        vector<int> v = {3, 4, 5, 6};
        KnapsackFPTAS kfp(w, v, 8);
        auto [val, items] = kfp.solve(0.3); // ε = 0.3
        printf("Knapsack FPTAS (ε=0.3) value = %d\n", val);
    }

    // LPT Scheduling
    {
        vector<int> jobs = {8, 7, 6, 5, 4, 3, 2, 1};
        LPTScheduling lpt(3);
        auto [assign, makespan] = lpt.solve(jobs);
        printf("LPT makespan = %lld\n", makespan);
    }

    // Max-Cut
    {
        MaxCutGreedy mc(4);
        mc.add_edge(0,1); mc.add_edge(1,2); mc.add_edge(2,3); mc.add_edge(0,3);
        mc.add_edge(0,2);
        auto [side, cut] = mc.solve();
        printf("Max cut = %lld\n", cut);
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
// 1. VERTEX COVER — Maximal Matching (ratio 2)
// ================================================================
public static class VertexCover {
    public static List<int> Solve(int n, List<(int u, int v)> edges) {
        var covered = new bool[n];
        var cover   = new List<int>();
        foreach (var (u, v) in edges) {
            if (!covered[u] && !covered[v]) {
                covered[u] = covered[v] = true;
                cover.Add(u); cover.Add(v);
            }
        }
        return cover;
    }
}

// ================================================================
// 2. SET COVER — Greedy density (ratio H_n)
// ================================================================
public static class SetCover {
    public static (List<int> chosen, double cost) Solve(
        int n, List<List<int>> sets, List<double> costs)
    {
        var covered   = new bool[n];
        int uncovered = n;
        var chosen    = new List<int>();
        double total  = 0.0;

        while (uncovered > 0) {
            double bestRatio = double.MaxValue;
            int bestSet = -1;
            for (int i = 0; i < sets.Count; i++) {
                int newCover = sets[i].Count(e => !covered[e]);
                if (newCover == 0) continue;
                double ratio = costs[i] / newCover;
                if (ratio < bestRatio) { bestRatio = ratio; bestSet = i; }
            }
            if (bestSet == -1) break;

            chosen.Add(bestSet);
            total += costs[bestSet];
            foreach (int e in sets[bestSet])
                if (!covered[e]) { covered[e] = true; uncovered--; }
        }
        return (chosen, total);
    }
}

// ================================================================
// 3. KNAPSACK FPTAS
// ================================================================
public static class KnapsackFPTAS {
    public static (int value, List<int> items) Solve(
        int[] w, int[] v, int W, double eps)
    {
        int n = w.Length;
        int vMax = v.Max();
        double K = Math.Max(1.0, eps * vMax / n);
        var vS = v.Select(x => (int)(x / K)).ToArray();
        int vSum = vS.Sum();

        var dp = new int[vSum + 1];
        Array.Fill(dp, int.MaxValue); dp[0] = 0;
        var taken = new bool[n, vSum + 1];

        for (int i = 0; i < n; i++)
            for (int j = vSum; j >= vS[i]; j--)
                if (dp[j - vS[i]] != int.MaxValue &&
                    dp[j - vS[i]] + w[i] < dp[j]) {
                    dp[j] = dp[j - vS[i]] + w[i];
                    taken[i, j] = true;
                }

        int bestJ = Enumerable.Range(0, vSum + 1)
            .Where(j => dp[j] <= W).DefaultIfEmpty(0).Max();

        var items = new List<int>();
        int total = 0, rem = bestJ;
        for (int i = n-1; i >= 0; i--)
            if (rem >= vS[i] && taken[i, rem]) {
                items.Add(i); total += v[i]; rem -= vS[i];
            }

        return (total, items);
    }
}

// ================================================================
// 4. LPT SCHEDULING
// ================================================================
public static class LPTScheduling {
    public static (int[] assign, long makespan) Solve(int[] jobs, int m) {
        int n = jobs.Length;
        var idx = Enumerable.Range(0, n)
            .OrderByDescending(i => jobs[i]).ToArray();

        var load   = new long[m];
        var assign = new int[n];
        var pq     = new SortedSet<(long load, int machine)>(
            Comparer<(long, int)>.Create((a, b) =>
                a.load != b.load ? a.load.CompareTo(b.load)
                                 : a.machine.CompareTo(b.machine)));
        for (int i = 0; i < m; i++) pq.Add((0L, i));

        foreach (int ji in idx) {
            var (l, machine) = pq.Min;
            pq.Remove(pq.Min);
            assign[ji] = machine;
            load[machine] += jobs[ji];
            pq.Add((load[machine], machine));
        }

        return (assign, load.Max());
    }
}

// ================================================================
// 5. MAX-CUT — Greedy + local search (ratio 1/2)
// ================================================================
public class MaxCut {
    private readonly int n;
    private readonly List<(int v, int w)>[] adj;

    public MaxCut(int n) {
        this.n = n;
        adj = Enumerable.Range(0, n)
            .Select(_ => new List<(int,int)>()).ToArray();
    }

    public void AddEdge(int u, int v, int w = 1) {
        adj[u].Add((v, w)); adj[v].Add((u, w));
    }

    public (int[] side, long cut) Solve() {
        var side = new int[n];
        for (int v = 0; v < n; v++) {
            long g0 = 0, g1 = 0;
            foreach (var (u, w) in adj[v]) {
                if (u >= v) continue;
                if (side[u] == 0) g1 += w; else g0 += w;
            }
            side[v] = g1 > g0 ? 1 : 0;
        }

        // Local search
        bool improved = true;
        long cut = ComputeCut(side);
        while (improved) {
            improved = false;
            for (int v = 0; v < n; v++) {
                long delta = 0;
                foreach (var (u, w) in adj[v])
                    delta += side[v] == side[u] ? +w : -w;
                if (delta > 0) { side[v] ^= 1; cut += delta; improved = true; }
            }
        }
        return (side, cut);
    }

    private long ComputeCut(int[] side) {
        long cut = 0;
        for (int v = 0; v < n; v++)
            foreach (var (u, w) in adj[v])
                if (side[v] != side[u]) cut += w;
        return cut / 2;
    }
}

public class Program {
    public static void Main() {
        // Vertex Cover
        var edges = new List<(int,int)> {(0,1),(1,2),(2,3),(3,4),(4,0)};
        var vc = VertexCover.Solve(5, edges);
        Console.WriteLine($"Vertex cover size={vc.Count}: {string.Join(" ", vc)}");

        // Set Cover
        var sets  = new List<List<int>> {
            new(){0,1,2}, new(){1,3,4}, new(){2,4,5}, new(){0,3,5}
        };
        var costs = new List<double> {3.0, 2.0, 2.5, 2.0};
        var (sc, scCost) = SetCover.Solve(6, sets, costs);
        Console.WriteLine($"Set cover cost={scCost:F1} sets={string.Join(" ", sc)}");

        // Knapsack FPTAS
        var (kVal, kItems) = KnapsackFPTAS.Solve(
            new[]{2,3,4,5}, new[]{3,4,5,6}, 8, 0.3);
        Console.WriteLine($"Knapsack FPTAS value={kVal}");

        // LPT Scheduling
        var (assign, makespan) = LPTScheduling.Solve(new[]{8,7,6,5,4,3,2,1}, 3);
        Console.WriteLine($"LPT makespan={makespan}");

        // Max-Cut
        var mc = new MaxCut(4);
        mc.AddEdge(0,1); mc.AddEdge(1,2); mc.AddEdge(2,3);
        mc.AddEdge(0,3); mc.AddEdge(0,2);
        var (_, cut) = mc.Solve();
        Console.WriteLine($"Max cut={cut}");
    }
}
```

---

## Inapproximability Lower Bounds

| Problem | Best ratio | Hardness lower bound | Assumption |
|---|---|---|---|
| Vertex Cover | 2 | 1.36 (UGC: 2-ε) | P≠NP / UGC |
| Set Cover | ln n | (1-ε) ln n | P≠NP |
| Knapsack | 1-ε (FPTAS) | No lower bound (PTAS exists) | — |
| TSP (metric) | 3/2 | 220/219 | P≠NP |
| TSP (general) | No constant ratio | No poly-time ratio | P≠NP |
| Max-Cut | 0.878 (GW) | 16/17 (UGC: 0.878+ε) | UGC |
| Clique | n^(1-ε) | n^(1-ε) | P≠NP |

UGC = Unique Games Conjecture (Khot, 2002).

---

## Pitfalls

- **LP relaxation is not always a valid lower bound** — `LP_OPT ≤ OPT` holds for minimization only when the integer optimum is feasible for the LP (which it always is by relaxation). For maximization, `LP_OPT >= OPT`. Confusing the direction of the bound gives a wrong ratio proof.
- **Rounding must preserve feasibility** — it is not enough to show the rounded cost is small; every constraint must remain satisfied after rounding. For vertex cover, the threshold-1/2 rounding satisfies every edge constraint. A lower threshold (e.g., 1/3) would violate constraints and produce an infeasible cover.
- **Greedy set cover ratio is H_{max_set_size}, not H_n** — the standard proof gives ratio `H_k` where k is the size of the largest set, not the universe size n. When all sets have size ≤ k << n, the bound is H_k ≈ ln k, not ln n.
- **FPTAS scaling requires careful floor** — `v'_i = floor(v_i / K)`. Using `round()` instead of `floor()` may violate the approximation guarantee. The floor ensures scaled values are underestimates, so the DP solution on scaled values corresponds to a feasible solution on original values.
- **Local search for Max-Cut may cycle** — if two adjacent vertices simultaneously improve by flipping, alternating flips loop forever. Process vertices one at a time and rescan only after an actual improvement. Tie-breaking deterministically (e.g., by index) ensures termination.
- **Primal-dual requires tight sets to be immediately included** — when a dual variable raise makes multiple sets tight simultaneously, all of them must be added. Skipping any tight set violates the complementary slackness condition used in the proof.
- **Christofides requires a metric** — the 3/2 ratio for TSP applies only when edge weights satisfy the triangle inequality. On general graphs, TSP has no constant approximation ratio unless P = NP. Applying Christofides to a non-metric instance silently produces a wrong ratio guarantee.

---

## Conclusion

Approximation algorithms transform computationally intractable optimization into tractable problems with rigorous quality guarantees:

- **LP rounding** provides a systematic framework: relax, solve the LP (polynomial), round carefully, prove feasibility and cost bound. The integrality gap of the LP is the fundamental obstacle and the approximation ratio's floor.
- **Greedy with proofs** often achieves the same ratio more efficiently — vertex cover and set cover admit O(n + m) greedy solutions matching their LP-rounding counterparts. The proof technique (charging, matching lower bound, exchange argument) is the core skill.
- **Primal-dual** unifies both: it simultaneously constructs the greedy solution and its LP dual certificate, giving a clean proof that works even without solving the LP explicitly.
- **FPTAS** (knapsack, bin packing) represents the best achievable: approximation ratio `(1+ε)` for any ε at cost O(poly(n, 1/ε)).

**Key takeaway:**  
The ratio proof always requires a lower bound on OPT. For LP rounding, LP_OPT provides it. For greedy, construct an explicit combinatorial lower bound (matching, dual variables, averaging argument). If you cannot find a lower bound, you cannot prove a ratio — and finding the right lower bound is the creative core of approximation algorithm design.
