# Bellman–Ford with Negative Cycle Detection — Shortest Paths and Negative Cycles

## Origin & Motivation

Bellman–Ford was developed independently by Shimbel (1955), Ford (1956), and Bellman (1958). Unlike Dijkstra, it handles **negative edge weights** and detects **negative cycles** — cycles whose total weight is negative, making shortest paths undefined (arbitrarily small) for any vertex reachable through them.

The algorithm relaxes all edges `n-1` times. After `n-1` iterations, all shortest paths (which contain at most `n-1` edges in a cycle-free graph) are finalized. A further **n-th iteration** that still produces a relaxation identifies a negative cycle: some path can be shortened indefinitely.

Complexity: **O(V * E)** time, **O(V)** space.

---

## Where It Is Used

- Shortest paths in graphs with negative weights (currency exchange, financial arbitrage)
- Negative cycle detection (arbitrage detection, constraint systems)
- Difference constraint systems (scheduling, temporal reasoning)
- Network routing protocols (RIP — Routing Information Protocol)
- Preprocessing step in Johnson's algorithm for all-pairs shortest paths

---

## Comparison with Dijkstra

| Property | Bellman–Ford | Dijkstra |
|---|---|---|
| Negative weights | Yes | No |
| Negative cycle detection | Yes | No |
| Time complexity | O(V * E) | O(E log V) |
| Space complexity | O(V) | O(V + E) |
| Graph type | Any directed | Non-negative weights |
| Implementation complexity | Simple | Moderate |

---

## Core Idea

**Relaxation:** For every edge `(u, v, w)`, if `dist[u] + w < dist[v]`, update `dist[v] = dist[u] + w`.

**Iterations:**
- After `k` iterations, `dist[v]` contains the shortest path using at most `k` edges.
- After `n-1` iterations, all true shortest paths are found (assuming no negative cycles).
- If a **n-th** relaxation succeeds for any edge, a negative cycle exists reachable from the source.

**Negative cycle reconstruction:** To find the actual cycle, track predecessors and follow them backward from any vertex updated in the n-th iteration until a cycle is detected.

```
Initialize: dist[src] = 0, dist[others] = INF, pred[v] = -1

Repeat n-1 times:
    for each edge (u, v, w):
        if dist[u] + w < dist[v]:
            dist[v] = dist[u] + w
            pred[v] = u

// n-th pass: detect negative cycles
for each edge (u, v, w):
    if dist[u] + w < dist[v]:
        // v is on or reachable from a negative cycle
        reconstruct cycle via pred[]
```

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Initialization | O(V) | O(V) |
| Main relaxation loop | O(V * E) | O(V) |
| Negative cycle detection | O(E) | O(V) |
| Cycle reconstruction | O(V) | O(V) |
| Total | O(V * E) | O(V) |

**Early termination:** If no relaxation occurs in a full pass, the algorithm can terminate early — shortest paths are already optimal. Best case: O(E) (already optimal graph, e.g., a DAG).

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct Edge {
    int u, v;
    long long w;
};

const long long INF = 1e18;

struct BellmanFord {
    int n;
    vector<Edge> edges;
    vector<long long> dist;
    vector<int> pred;

    BellmanFord(int n) : n(n), dist(n, INF), pred(n, -1) {}

    void add_edge(int u, int v, long long w) {
        edges.push_back({u, v, w});
    }

    // Run from source src.
    // Returns true if no negative cycle is reachable from src.
    // Returns false if a negative cycle exists; neg_cycle contains one such cycle.
    bool solve(int src, vector<int>& neg_cycle) {
        dist[src] = 0;

        int last_relaxed = -1;

        // n-1 relaxation passes
        for (int iter = 0; iter < n; iter++) {
            last_relaxed = -1;
            for (auto& e : edges) {
                if (dist[e.u] == INF) continue;
                if (dist[e.u] + e.w < dist[e.v]) {
                    dist[e.v] = dist[e.u] + e.w;
                    pred[e.v] = e.u;
                    last_relaxed = e.v;
                }
            }
            // Early termination: no relaxation in this pass
            if (last_relaxed == -1) return true;
        }

        // If we reach here, last_relaxed is on or reachable from a negative cycle.
        // Walk back n steps through pred[] to guarantee we land inside the cycle.
        int x = last_relaxed;
        for (int i = 0; i < n; i++)
            x = pred[x];

        // x is now guaranteed to be inside a negative cycle.
        // Reconstruct the cycle by following pred[] until we revisit x.
        neg_cycle.clear();
        int cur = x;
        do {
            neg_cycle.push_back(cur);
            cur = pred[cur];
        } while (cur != x);
        neg_cycle.push_back(x);
        reverse(neg_cycle.begin(), neg_cycle.end());

        return false;
    }

    // Detect ALL vertices with undefined shortest paths
    // (on or reachable from any negative cycle).
    // After solve() returned false, mark all such vertices with dist = -INF.
    void mark_negative_reachable() {
        // Extra pass: propagate -INF through all edges reachable from negative cycle
        // First: identify cycle vertices (dist was updated in n-th pass)
        vector<bool> in_neg(n, false);

        // Re-run one more pass to find all vertices updated after n-1 iterations
        for (auto& e : edges) {
            if (dist[e.u] == INF) continue;
            if (dist[e.u] + e.w < dist[e.v]) {
                dist[e.v] = -INF;
                in_neg[e.v] = true;
            }
        }

        // BFS/propagate -INF forward
        bool changed = true;
        while (changed) {
            changed = false;
            for (auto& e : edges) {
                if (dist[e.u] == -INF && dist[e.v] != -INF) {
                    dist[e.v] = -INF;
                    changed = true;
                }
            }
        }
    }
};

// -------------------------------------------------------
// Difference Constraints solver using Bellman–Ford
// System: x_j - x_i <= w  =>  edge (i -> j, w)
// Add super-source s connected to all vertices with weight 0.
// SAT iff no negative cycle.
// -------------------------------------------------------
bool solve_difference_constraints(
    int n,
    const vector<tuple<int,int,long long>>& constraints,
    vector<long long>& solution)
{
    // Vertices 0..n-1, super-source = n
    BellmanFord bf(n + 1);
    int src = n;
    for (int i = 0; i < n; i++)
        bf.add_edge(src, i, 0);
    for (auto& [i, j, w] : constraints)
        bf.add_edge(i, j, w);

    vector<int> cycle;
    if (!bf.solve(src, cycle)) return false;  // UNSAT

    solution.assign(n, 0);
    for (int i = 0; i < n; i++)
        solution[i] = bf.dist[i];
    return true;
}

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    // Graph with negative cycle: 0->1->2->1 (cycle weight -1)
    {
        BellmanFord bf(4);
        bf.add_edge(0, 1, 1);
        bf.add_edge(1, 2, 2);
        bf.add_edge(2, 1, -4);  // negative cycle: 1->2->1, weight = 2+(-4) = -2
        bf.add_edge(1, 3, 5);

        vector<int> cycle;
        bool ok = bf.solve(0, cycle);
        if (!ok) {
            printf("Negative cycle found: ");
            for (int v : cycle) printf("%d ", v);
            printf("\n");
        } else {
            printf("No negative cycle. Distances from 0:\n");
            for (int i = 0; i < 4; i++)
                printf("  dist[%d] = %lld\n", i, bf.dist[i]);
        }
    }

    // Graph without negative cycle
    {
        BellmanFord bf(4);
        bf.add_edge(0, 1, -1);
        bf.add_edge(1, 2, 3);
        bf.add_edge(0, 2, 4);
        bf.add_edge(2, 3, -2);

        vector<int> cycle;
        bool ok = bf.solve(0, cycle);
        if (ok) {
            printf("No negative cycle. Distances from 0:\n");
            for (int i = 0; i < 4; i++) {
                if (bf.dist[i] == INF) printf("  dist[%d] = INF\n", i);
                else printf("  dist[%d] = %lld\n", i, bf.dist[i]);
            }
        }
    }

    // Difference constraints: x1 - x0 <= 3, x2 - x1 <= -1, x0 - x2 <= 2
    {
        vector<tuple<int,int,long long>> constraints = {
            {0, 1, 3},   // x1 - x0 <= 3  =>  edge 0->1, w=3
            {1, 2, -1},  // x2 - x1 <= -1 =>  edge 1->2, w=-1
            {2, 0, 2}    // x0 - x2 <= 2  =>  edge 2->0, w=2
        };
        vector<long long> sol;
        if (solve_difference_constraints(3, constraints, sol)) {
            printf("Difference constraints SAT: x0=%lld, x1=%lld, x2=%lld\n",
                sol[0], sol[1], sol[2]);
        } else {
            printf("Difference constraints UNSAT (negative cycle)\n");
        }
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public struct Edge {
    public int U, V;
    public long W;
    public Edge(int u, int v, long w) { U=u; V=v; W=w; }
}

public class BellmanFord {
    private readonly int n;
    private readonly List<Edge> edges = new();
    public  readonly long[] Dist;
    public  readonly int[]  Pred;
    private const long INF = long.MaxValue / 2;

    public BellmanFord(int n) {
        this.n = n;
        Dist   = new long[n];
        Pred   = new int[n];
        Array.Fill(Dist, INF);
        Array.Fill(Pred, -1);
    }

    public void AddEdge(int u, int v, long w) => edges.Add(new Edge(u, v, w));

    // Returns true = no negative cycle reachable from src.
    // On false, negCycle contains the reconstructed cycle vertices.
    public bool Solve(int src, out List<int> negCycle) {
        negCycle = new();
        Dist[src] = 0;
        int lastRelaxed = -1;

        for (int iter = 0; iter < n; iter++) {
            lastRelaxed = -1;
            foreach (var e in edges) {
                if (Dist[e.U] == INF) continue;
                if (Dist[e.U] + e.W < Dist[e.V]) {
                    Dist[e.V] = Dist[e.U] + e.W;
                    Pred[e.V] = e.U;
                    lastRelaxed = e.V;
                }
            }
            if (lastRelaxed == -1) return true;
        }

        // Walk back n steps to land inside the cycle
        int x = lastRelaxed;
        for (int i = 0; i < n; i++) x = Pred[x];

        // Trace the cycle
        int cur = x;
        do {
            negCycle.Add(cur);
            cur = Pred[cur];
        } while (cur != x);
        negCycle.Add(x);
        negCycle.Reverse();

        return false;
    }

    // Mark all vertices on or reachable from a negative cycle as -INF
    public void MarkNegativeReachable() {
        bool changed = true;
        // One extra pass to seed -INF
        foreach (var e in edges) {
            if (Dist[e.U] != INF && Dist[e.U] + e.W < Dist[e.V])
                Dist[e.V] = long.MinValue;
        }
        // Propagate
        while (changed) {
            changed = false;
            foreach (var e in edges) {
                if (Dist[e.U] == long.MinValue && Dist[e.V] != long.MinValue) {
                    Dist[e.V] = long.MinValue;
                    changed = true;
                }
            }
        }
    }
}

// -------------------------------------------------------
// Difference Constraints
// -------------------------------------------------------
public class DifferenceConstraints {
    public static bool Solve(
        int n,
        List<(int i, int j, long w)> constraints,
        out long[] solution)
    {
        // Super-source = n, edges to all vertices with weight 0
        var bf = new BellmanFord(n + 1);
        for (int i = 0; i < n; i++) bf.AddEdge(n, i, 0);
        foreach (var (i, j, w) in constraints) bf.AddEdge(i, j, w);

        bool ok = bf.Solve(n, out _);
        solution = ok ? bf.Dist[..n] : Array.Empty<long>();
        return ok;
    }
}

public class Program {
    public static void Main() {
        // Graph with negative cycle
        {
            var bf = new BellmanFord(4);
            bf.AddEdge(0, 1, 1);
            bf.AddEdge(1, 2, 2);
            bf.AddEdge(2, 1, -4);  // negative cycle 1->2->1
            bf.AddEdge(1, 3, 5);

            bool ok = bf.Solve(0, out var cycle);
            if (!ok) {
                Console.Write("Negative cycle: ");
                Console.WriteLine(string.Join(" -> ", cycle));
            }
        }

        // Graph without negative cycle
        {
            var bf = new BellmanFord(4);
            bf.AddEdge(0, 1, -1);
            bf.AddEdge(1, 2,  3);
            bf.AddEdge(0, 2,  4);
            bf.AddEdge(2, 3, -2);

            bool ok = bf.Solve(0, out _);
            if (ok) {
                Console.WriteLine("No negative cycle. Distances:");
                for (int i = 0; i < 4; i++)
                    Console.WriteLine($"  dist[{i}] = {bf.Dist[i]}");
            }
        }

        // Difference constraints
        {
            var constraints = new List<(int, int, long)> {
                (0, 1,  3),
                (1, 2, -1),
                (2, 0,  2)
            };
            bool sat = DifferenceConstraints.Solve(3, constraints, out var sol);
            if (sat) Console.WriteLine($"Constraints SAT: x0={sol[0]}, x1={sol[1]}, x2={sol[2]}");
            else     Console.WriteLine("Constraints UNSAT");
        }
    }
}
```

---

## Negative Cycle Reconstruction — Detail

The naive approach of following `pred[]` from the last relaxed vertex may walk into an unreachable region before hitting the cycle. The correct procedure:

```
1. Take any vertex updated in the n-th pass: call it x.
2. Follow pred[x] exactly n times.
   After n steps, x is GUARANTEED to be inside a cycle
   (pigeonhole: n steps over n vertices must revisit one).
3. From this x, follow pred[] collecting vertices until x repeats.
4. The collected vertices form the negative cycle.
```

This is O(V) and always terminates correctly.

---

## Pitfalls

- **INF overflow on relaxation** — `dist[u] + w` overflows if `dist[u] = INF` and `w` is positive. Always guard with `if (dist[u] == INF) continue` before attempting relaxation. Use `long` / `int64` and set INF to `1e18` rather than `LLONG_MAX`.
- **n-th pass landmine** — in the n-th pass, the vertex updated may be reachable from the negative cycle but not on it. The "walk back n times through pred" step is essential to land inside the cycle itself. Skipping it gives a wrong starting point for reconstruction.
- **Disconnected graphs** — vertices unreachable from `src` keep `dist = INF`. A negative cycle among unreachable vertices is not detected when running from a single source. To detect all negative cycles in the graph, add a super-source connected to all vertices with weight 0 and run from it.
- **Predecessor graph cycles** — `pred[]` forms a functional graph (each node has at most one predecessor). The cycle-finding loop `do { ... } while (cur != x)` always terminates because after step 2 above, `x` is guaranteed inside a cycle in the predecessor graph.
- **Difference constraints — edge direction** — constraint `x_j - x_i <= w` maps to edge `i -> j` with weight `w`, not `j -> i`. Reversing the direction produces wrong results silently; the system will report SAT with an incorrect solution.
- **Early termination must reset last_relaxed** — reset `last_relaxed = -1` at the start of every pass, not once before the loop. Otherwise a relaxation from a previous pass is mistakenly retained, and early termination never fires.
- **Marking all negative-reachable vertices** — after detection, some problems require knowing which vertices have `dist = -INF` (undefined). A single extra propagation pass (BFS/repeated relaxation) is needed; the n-th iteration alone only identifies vertices directly on a negative edge in the cycle, not all downstream vertices.

---

## Conclusion

Bellman–Ford with negative cycle detection is the **definitive single-source shortest path algorithm for general directed graphs**:

- Handles negative edge weights that make Dijkstra inapplicable.
- Detects and reconstructs negative cycles in O(V) extra steps beyond the main loop.
- Directly solves difference constraint systems — a fundamental tool in scheduling and temporal reasoning.
- Serves as the inner engine of Johnson's algorithm, which uses Bellman–Ford once to reweight edges before running Dijkstra V times for all-pairs shortest paths.

**Key takeaway:**  
Prefer Dijkstra when all edge weights are non-negative — it is O(E log V) vs O(V * E). Use Bellman–Ford when negative weights are present or when negative cycle detection / difference constraints are required. The early termination optimization brings best-case complexity down to O(E) on graphs where shortest paths stabilize quickly.
