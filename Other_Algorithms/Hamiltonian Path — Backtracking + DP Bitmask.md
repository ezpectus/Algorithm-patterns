# Hamiltonian Path — Backtracking + DP Bitmask

## Origin & Motivation

A **Hamiltonian path** visits every vertex of a graph exactly once. A **Hamiltonian cycle** (circuit) is a Hamiltonian path that returns to the starting vertex. Unlike Euler paths (which traverse every edge exactly once and are solvable in O(V+E)), Hamiltonian path is **NP-complete** — no polynomial-time algorithm is known.

Two practical approaches:
- **Backtracking with pruning:** Exhaustive DFS with early termination on dead ends. Exponential worst case O(n!) but often fast on sparse graphs or with good pruning.
- **Held-Karp DP (bitmask DP):** Dynamic programming over subsets. O(2^n * n²) time — exponential but tractable for n ≤ 20, far better than O(n!).

The Held-Karp algorithm (1962) was one of the earliest applications of dynamic programming to combinatorial optimization. It is the basis of the exact TSP (Traveling Salesman Problem) algorithm.

Complexity: **O(n! / pruning)** backtracking, **O(2^n * n²)** bitmask DP.

---

## Where It Is Used

- TSP (Traveling Salesman): minimum-weight Hamiltonian cycle
- Route planning with mandatory waypoints
- Competitive programming: exact Hamiltonian path/cycle on small graphs
- Bioinformatics: genome sequencing (Hamiltonian path on overlap graphs)
- Puzzle solving: knight's tour, Gray codes
- Circuit board design: optimal component visit order

---

## Problem Variants

| Variant | Description |
|---|---|
| Hamiltonian path existence | Does any path visiting all vertices exist? |
| Hamiltonian cycle existence | Does any cycle visiting all vertices exist? |
| Count Hamiltonian paths | How many distinct paths exist? |
| Minimum weight Hamiltonian path | Shortest path visiting all vertices (no return) |
| Minimum weight Hamiltonian cycle (TSP) | Shortest cycle visiting all vertices |

---

## Part 1 — Backtracking with Pruning

### Algorithm

DFS from each starting vertex. At each step, try extending the current path by one unvisited vertex. Prune if:
- No unvisited neighbor exists and not all vertices visited (dead end)
- Warnsdorff heuristic: prefer vertices with fewest onward moves (reduces branching)

```
path = [start]
visited = {start}

backtrack():
    if |path| == n: found Hamiltonian path
    
    v = path.last()
    for each neighbor u of v not in visited:
        path.append(u)
        visited.add(u)
        backtrack()
        path.pop()
        visited.remove(u)
```

### Pruning Strategies

- **Connectivity pruning:** After removing current path vertices, check if remaining graph is still connected. If not, prune.
- **Warnsdorff's rule:** Among available next vertices, prefer the one with fewest unvisited neighbors (greedy heuristic — not guaranteed optimal but dramatically reduces search).
- **Dead-end detection:** If any unvisited vertex has 0 available neighbors, prune immediately.
- **Degree-one forcing:** If an unvisited vertex has exactly one available neighbor, that edge is forced — take it immediately.

---

## Part 2 — Bitmask DP (Held-Karp)

### State

`dp[mask][v]` = true/false (existence) or minimum cost (optimization)

- `mask` = bitmask of visited vertices (bit i = 1 means vertex i visited)
- `v` = current vertex (last vertex in the partial path)

`dp[mask][v]` is reachable iff there exists a Hamiltonian path through exactly the vertices in `mask`, ending at `v`.

### Recurrence

```
Base:
  dp[1 << v][v] = true   for each starting vertex v
  (or: dp[1 << 0][0] = true if we fix the start)

Transition:
  dp[mask | (1<<u)][u] = true
    if dp[mask][v] is true
    and (v, u) is an edge
    and bit u is NOT set in mask

Answer:
  Hamiltonian path exists iff dp[(1<<n)-1][v] is true for any v
  Hamiltonian cycle exists iff dp[(1<<n)-1][v] and edge (v, 0) exists
```

### For minimum weight (TSP):

```
dp[mask][v] = minimum cost to visit exactly the vertices in mask, ending at v

dp[1<<v][v]        = 0          (start at v, cost 0)
dp[mask|(1<<u)][u] = min(dp[mask][v] + cost(v, u))

Answer: min over v of (dp[full_mask][v] + cost(v, start))  // TSP
        min over v of dp[full_mask][v]                       // Hamiltonian path
```

### Path Reconstruction

Store `parent[mask][v]` = the previous vertex in the optimal transition. Backtrack from the final state to recover the full path.

---

## Complexity Analysis

| Method | Time | Space | Practical n |
|---|---|---|---|
| Backtracking (worst) | O(n!) | O(n) | n ≤ 12 |
| Backtracking (with pruning) | Much better in practice | O(n) | n ≤ 20 |
| Bitmask DP | O(2^n * n²) | O(2^n * n) | n ≤ 20 |
| Bitmask DP (count paths) | O(2^n * n²) | O(2^n * n) | n ≤ 20 |

For n=20: 2^20 * 400 ≈ 4×10^8 operations — borderline, fits in ~2s.
For n=25: 2^25 * 625 ≈ 2×10^10 — too slow for DP; backtracking needed.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const ll INF = 1e18;

// ================================================================
// BACKTRACKING — Hamiltonian Path existence and enumeration
// ================================================================
struct HamiltonianBacktrack {
    int n;
    vector<vector<int>> adj; // adjacency list
    vector<bool> visited;
    vector<int>  path;
    int solutions;

    HamiltonianBacktrack(int n) : n(n), adj(n), visited(n,false), solutions(0) {}

    void add_edge(int u, int v) { adj[u].push_back(v); adj[v].push_back(u); }

    bool backtrack(int depth) {
        if (depth == n) {
            solutions++;
            return true; // return false to find ALL solutions
        }

        int v = path.back();

        // Warnsdorff heuristic: sort neighbors by degree (fewest first)
        vector<pair<int,int>> neighbors; // {available_degree, vertex}
        for (int u : adj[v]) {
            if (visited[u]) continue;
            int deg = 0;
            for (int w : adj[u]) if (!visited[w]) deg++;
            neighbors.push_back({deg, u});
        }
        sort(neighbors.begin(), neighbors.end());

        for (auto [deg, u] : neighbors) {
            if (visited[u]) continue;
            path.push_back(u);
            visited[u] = true;

            if (backtrack(depth + 1)) return true; // return false to count all

            path.pop_back();
            visited[u] = false;
        }
        return false;
    }

    // Find first Hamiltonian path (or any, trying all starts)
    vector<int> find_path() {
        for (int start = 0; start < n; start++) {
            fill(visited.begin(), visited.end(), false);
            path.clear();
            path.push_back(start);
            visited[start] = true;
            if (backtrack(1)) return path;
        }
        return {}; // no Hamiltonian path
    }

    // Check Hamiltonian cycle: path must exist from start that ends at a neighbor of start
    bool find_cycle() {
        for (int start = 0; start < n; start++) {
            fill(visited.begin(), visited.end(), false);
            path.clear();
            path.push_back(start);
            visited[start] = true;
            if (backtrack(1)) {
                // Check if last vertex connects back to start
                int last = path.back();
                for (int u : adj[last])
                    if (u == start) return true;
            }
        }
        return false;
    }

    int count_paths() {
        solutions = 0;
        for (int start = 0; start < n; start++) {
            fill(visited.begin(), visited.end(), false);
            path.clear();
            path.push_back(start);
            visited[start] = true;

            // Modified backtrack that doesn't early-return
            function<void(int)> bt = [&](int depth) {
                if (depth == n) { solutions++; return; }
                int v = path.back();
                for (int u : adj[v]) {
                    if (visited[u]) continue;
                    path.push_back(u); visited[u] = true;
                    bt(depth + 1);
                    path.pop_back(); visited[u] = false;
                }
            };
            bt(1);
        }
        return solutions / 2; // each undirected path counted twice (forward/backward)
    }
};

// ================================================================
// BITMASK DP — Hamiltonian path existence, count, minimum cost
// ================================================================
struct HamiltonianDP {
    int n;
    vector<vector<ll>> cost; // cost[u][v] = edge weight (INF if no edge)
    int full_mask;

    // dp_exist[mask][v] = can we visit exactly vertices in mask, ending at v?
    vector<vector<bool>> dp_exist;

    // dp_min[mask][v] = min cost path visiting exactly mask, ending at v
    vector<vector<ll>>   dp_min;

    // parent[mask][v] = previous vertex in optimal path
    vector<vector<int>>  parent;

    HamiltonianDP(int n, vector<vector<ll>> cost)
        : n(n), cost(cost), full_mask((1<<n)-1),
          dp_exist(1<<n, vector<bool>(n, false)),
          dp_min(1<<n, vector<ll>(n, INF)),
          parent(1<<n, vector<int>(n, -1)) {}

    // ----------------------------------------------------------------
    // Existence: does any Hamiltonian path exist?
    // ----------------------------------------------------------------
    bool solve_exist() {
        // Base: start from each vertex
        for (int v = 0; v < n; v++)
            dp_exist[1<<v][v] = true;

        for (int mask = 1; mask < (1<<n); mask++) {
            for (int v = 0; v < n; v++) {
                if (!dp_exist[mask][v]) continue;
                if (!((mask >> v) & 1)) continue;

                for (int u = 0; u < n; u++) {
                    if ((mask >> u) & 1) continue;        // u already visited
                    if (cost[v][u] == INF) continue;      // no edge

                    dp_exist[mask | (1<<u)][u] = true;
                }
            }
        }

        // Any vertex at full mask
        for (int v = 0; v < n; v++)
            if (dp_exist[full_mask][v]) return true;
        return false;
    }

    // ----------------------------------------------------------------
    // Minimum cost Hamiltonian path (no fixed start, no return)
    // ----------------------------------------------------------------
    ll solve_min_path() {
        for (int v = 0; v < n; v++)
            dp_min[1<<v][v] = 0;

        for (int mask = 1; mask < (1<<n); mask++) {
            for (int v = 0; v < n; v++) {
                if (dp_min[mask][v] == INF) continue;
                if (!((mask >> v) & 1)) continue;

                for (int u = 0; u < n; u++) {
                    if ((mask >> u) & 1) continue;
                    if (cost[v][u] == INF) continue;

                    ll new_cost = dp_min[mask][v] + cost[v][u];
                    if (new_cost < dp_min[mask|(1<<u)][u]) {
                        dp_min[mask|(1<<u)][u] = new_cost;
                        parent[mask|(1<<u)][u] = v;
                    }
                }
            }
        }

        ll ans = INF;
        for (int v = 0; v < n; v++)
            ans = min(ans, dp_min[full_mask][v]);
        return ans;
    }

    // ----------------------------------------------------------------
    // Minimum cost Hamiltonian CYCLE (TSP), fixed start = 0
    // ----------------------------------------------------------------
    ll solve_tsp() {
        // Start from vertex 0
        dp_min[1][0] = 0;

        for (int mask = 1; mask < (1<<n); mask++) {
            if (!(mask & 1)) continue; // vertex 0 must be in mask
            for (int v = 0; v < n; v++) {
                if (dp_min[mask][v] == INF) continue;
                if (!((mask >> v) & 1)) continue;

                for (int u = 0; u < n; u++) {
                    if ((mask >> u) & 1) continue;
                    if (cost[v][u] == INF) continue;

                    ll nc = dp_min[mask][v] + cost[v][u];
                    if (nc < dp_min[mask|(1<<u)][u]) {
                        dp_min[mask|(1<<u)][u] = nc;
                        parent[mask|(1<<u)][u] = v;
                    }
                }
            }
        }

        ll ans = INF;
        for (int v = 1; v < n; v++) {
            if (dp_min[full_mask][v] == INF) continue;
            if (cost[v][0] == INF) continue;
            ans = min(ans, dp_min[full_mask][v] + cost[v][0]);
        }
        return ans;
    }

    // ----------------------------------------------------------------
    // Count Hamiltonian paths (no fixed start)
    // ----------------------------------------------------------------
    ll count_paths() {
        vector<vector<ll>> dp(1<<n, vector<ll>(n, 0));
        for (int v = 0; v < n; v++) dp[1<<v][v] = 1;

        for (int mask = 1; mask < (1<<n); mask++) {
            for (int v = 0; v < n; v++) {
                if (!dp[mask][v] || !((mask>>v)&1)) continue;
                for (int u = 0; u < n; u++) {
                    if ((mask>>u)&1) continue;
                    if (cost[v][u] == INF) continue;
                    dp[mask|(1<<u)][u] += dp[mask][v];
                }
            }
        }

        ll total = 0;
        for (int v = 0; v < n; v++) total += dp[full_mask][v];
        return total; // directed paths; divide by 2 for undirected
    }

    // Reconstruct minimum path from parent table
    vector<int> reconstruct_path() {
        // Find best end vertex
        ll best = INF; int end_v = -1;
        for (int v = 0; v < n; v++)
            if (dp_min[full_mask][v] < best) { best = dp_min[full_mask][v]; end_v = v; }
        if (end_v == -1) return {};

        vector<int> path;
        int mask = full_mask, v = end_v;
        while (v != -1) {
            path.push_back(v);
            int prev = parent[mask][v];
            mask ^= (1<<v);
            v = prev;
        }
        reverse(path.begin(), path.end());
        return path;
    }

    vector<int> reconstruct_tsp() {
        // Find best last vertex for TSP
        ll best = INF; int end_v = -1;
        for (int v = 1; v < n; v++) {
            if (dp_min[full_mask][v] == INF || cost[v][0] == INF) continue;
            if (dp_min[full_mask][v] + cost[v][0] < best) {
                best = dp_min[full_mask][v] + cost[v][0];
                end_v = v;
            }
        }
        if (end_v == -1) return {};

        vector<int> path;
        int mask = full_mask, v = end_v;
        while (v != -1) {
            path.push_back(v);
            int prev = parent[mask][v];
            mask ^= (1<<v);
            v = prev;
        }
        reverse(path.begin(), path.end());
        path.push_back(0); // close cycle
        return path;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // --- Backtracking ---
    {
        printf("=== Backtracking ===\n");
        HamiltonianBacktrack hb(5);
        hb.add_edge(0,1); hb.add_edge(1,2); hb.add_edge(2,3);
        hb.add_edge(3,4); hb.add_edge(4,0); hb.add_edge(0,2);

        auto path = hb.find_path();
        printf("Hamiltonian path: ");
        if (path.empty()) printf("not found\n");
        else { for (int v : path) printf("%d ", v); printf("\n"); }

        printf("Has Hamiltonian cycle: %s\n", hb.find_cycle() ? "yes" : "no");
    }

    // --- Bitmask DP ---
    {
        printf("\n=== Bitmask DP — Existence ===\n");
        int n = 4;
        auto inf = INF;
        // Complete graph K4: all edges exist with weight 1
        vector<vector<ll>> cost(n, vector<ll>(n, 1));
        for (int i = 0; i < n; i++) cost[i][i] = INF;

        HamiltonianDP dp(n, cost);
        printf("Hamiltonian path exists: %s\n", dp.solve_exist() ? "yes" : "no");
    }

    {
        printf("\n=== Bitmask DP — Min cost path + TSP ===\n");
        int n = 4;
        // Asymmetric costs
        vector<vector<ll>> cost = {
            {INF, 10,  15, 20},
            {10, INF,  35, 25},
            {15,  35, INF, 30},
            {20,  25,  30, INF}
        };

        HamiltonianDP dp(n, cost);
        ll tsp_cost = dp.solve_tsp();
        printf("TSP min cost: %lld\n", tsp_cost); // 10+25+30+15=80

        auto tour = dp.reconstruct_tsp();
        printf("TSP tour: ");
        for (int v : tour) printf("%d ", v);
        printf("\n");
    }

    {
        printf("\n=== Bitmask DP — Count paths ===\n");
        int n = 4;
        vector<vector<ll>> cost(n, vector<ll>(n, 1));
        for (int i = 0; i < n; i++) cost[i][i] = INF;

        HamiltonianDP dp(n, cost);
        ll cnt = dp.count_paths();
        printf("Directed Hamiltonian paths in K4: %lld\n", cnt);   // 4*3*2*1 = 24
        printf("Undirected Hamiltonian paths in K4: %lld\n", cnt/2); // 12
    }

    {
        printf("\n=== Min Hamiltonian Path (no cycle) ===\n");
        int n = 4;
        vector<vector<ll>> cost = {
            {INF,  1, INF,  4},
            {  1, INF,  2, INF},
            {INF,  2, INF,  3},
            {  4, INF,  3, INF}
        };

        HamiltonianDP dp(n, cost);
        ll path_cost = dp.solve_min_path();
        printf("Min Hamiltonian path cost: %lld\n", path_cost); // 1+2+3=6

        auto path = dp.reconstruct_path();
        printf("Path: ");
        for (int v : path) printf("%d ", v);
        printf("\n"); // 0->1->2->3
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

// ================================================================
// BACKTRACKING
// ================================================================
public class HamiltonianBacktrack {
    private readonly int n;
    private readonly List<int>[] adj;
    private readonly bool[] visited;
    private readonly List<int> path = new();

    public HamiltonianBacktrack(int n) {
        this.n = n;
        adj     = new List<int>[n];
        visited = new bool[n];
        for (int i = 0; i < n; i++) adj[i] = new();
    }

    public void AddEdge(int u, int v) { adj[u].Add(v); adj[v].Add(u); }

    private bool Backtrack(int depth) {
        if (depth == n) return true;
        int v = path[^1];
        foreach (int u in adj[v]) {
            if (visited[u]) continue;
            path.Add(u); visited[u] = true;
            if (Backtrack(depth + 1)) return true;
            path.RemoveAt(path.Count - 1); visited[u] = false;
        }
        return false;
    }

    public List<int>? FindPath() {
        for (int s = 0; s < n; s++) {
            Array.Fill(visited, false);
            path.Clear(); path.Add(s); visited[s] = true;
            if (Backtrack(1)) return new List<int>(path);
        }
        return null;
    }

    public bool HasCycle() {
        for (int s = 0; s < n; s++) {
            Array.Fill(visited, false);
            path.Clear(); path.Add(s); visited[s] = true;
            if (Backtrack(1)) {
                int last = path[^1];
                if (adj[last].Contains(s)) return true;
            }
        }
        return false;
    }
}

// ================================================================
// BITMASK DP
// ================================================================
public class HamiltonianDP {
    private readonly int n, full;
    private readonly long[,] cost;
    private readonly long[,] dp;
    private readonly int[,]  par;
    private const long INF = long.MaxValue / 2;

    public HamiltonianDP(int n, long[,] cost) {
        this.n = n; full = (1 << n) - 1;
        this.cost = cost;
        dp  = new long[1<<n, n]; for (int i=0;i<(1<<n);i++) for(int j=0;j<n;j++) dp[i,j]=INF;
        par = new int[1<<n, n];  for (int i=0;i<(1<<n);i++) for(int j=0;j<n;j++) par[i,j]=-1;
    }

    public long SolveMinPath() {
        for (int v = 0; v < n; v++) dp[1<<v, v] = 0;

        for (int mask = 1; mask < (1<<n); mask++)
            for (int v = 0; v < n; v++) {
                if (dp[mask,v] == INF || ((mask>>v)&1) == 0) continue;
                for (int u = 0; u < n; u++) {
                    if (((mask>>u)&1) != 0 || cost[v,u] == INF) continue;
                    long nc = dp[mask,v] + cost[v,u];
                    if (nc < dp[mask|(1<<u), u]) {
                        dp[mask|(1<<u), u] = nc;
                        par[mask|(1<<u), u] = v;
                    }
                }
            }

        long ans = INF;
        for (int v = 0; v < n; v++) ans = Math.Min(ans, dp[full, v]);
        return ans;
    }

    public long SolveTSP() {
        dp[1, 0] = 0;
        for (int mask = 1; mask < (1<<n); mask++) {
            if ((mask & 1) == 0) continue;
            for (int v = 0; v < n; v++) {
                if (dp[mask,v] == INF || ((mask>>v)&1)==0) continue;
                for (int u = 0; u < n; u++) {
                    if (((mask>>u)&1)!=0 || cost[v,u]==INF) continue;
                    long nc = dp[mask,v]+cost[v,u];
                    if (nc < dp[mask|(1<<u),u]) {
                        dp[mask|(1<<u),u]=nc; par[mask|(1<<u),u]=v;
                    }
                }
            }
        }
        long ans = INF;
        for (int v = 1; v < n; v++)
            if (dp[full,v]!=INF && cost[v,0]!=INF)
                ans = Math.Min(ans, dp[full,v]+cost[v,0]);
        return ans;
    }

    public long CountPaths() {
        var cnt = new long[1<<n, n];
        for (int v=0;v<n;v++) cnt[1<<v,v]=1;
        for (int mask=1;mask<(1<<n);mask++)
            for (int v=0;v<n;v++) {
                if (cnt[mask,v]==0||((mask>>v)&1)==0) continue;
                for (int u=0;u<n;u++) {
                    if(((mask>>u)&1)!=0||cost[v,u]==INF) continue;
                    cnt[mask|(1<<u),u]+=cnt[mask,v];
                }
            }
        long total=0;
        for(int v=0;v<n;v++) total+=cnt[full,v];
        return total;
    }

    public List<int> ReconstructTSP() {
        long best=INF; int ev=-1;
        for(int v=1;v<n;v++)
            if(dp[full,v]!=INF&&cost[v,0]!=INF&&dp[full,v]+cost[v,0]<best)
                { best=dp[full,v]+cost[v,0]; ev=v; }
        if(ev<0) return new();
        var path=new List<int>();
        int mask=full,v2=ev;
        while(v2>=0) { path.Add(v2); int p=par[mask,v2]; mask^=(1<<v2); v2=p; }
        path.Reverse(); path.Add(0);
        return path;
    }
}

public class Program {
    public static void Main() {
        // Backtracking
        var hb = new HamiltonianBacktrack(5);
        hb.AddEdge(0,1); hb.AddEdge(1,2); hb.AddEdge(2,3);
        hb.AddEdge(3,4); hb.AddEdge(4,0); hb.AddEdge(0,2);
        var p = hb.FindPath();
        Console.WriteLine($"Path: {(p==null?"none":string.Join("->",p))}");
        Console.WriteLine($"Cycle: {hb.HasCycle()}");

        // Bitmask DP
        const long I = long.MaxValue/2;
        long[,] c = { {I,10,15,20},{10,I,35,25},{15,35,I,30},{20,25,30,I} };
        var dp = new HamiltonianDP(4, c);
        Console.WriteLine($"TSP: {dp.SolveTSP()}");
        Console.WriteLine($"Tour: {string.Join("->", dp.ReconstructTSP())}");

        // Count
        long[,] c2 = new long[4,4];
        for(int i=0;i<4;i++) for(int j=0;j<4;j++) c2[i,j]= i==j?I:1;
        var dp2 = new HamiltonianDP(4, c2);
        Console.WriteLine($"Directed Ham. paths in K4: {dp2.CountPaths()}");
    }
}
```

---

## Bitmask DP — State Space Visualization

```
n=3, vertices {0,1,2}, full_mask = 111₂ = 7

Masks and reachable states (complete graph):
  001 (only 0): dp[001][0] = 0       ← base
  010 (only 1): dp[010][1] = 0       ← base
  100 (only 2): dp[100][2] = 0       ← base

  011 (0,1 visited): dp[011][1] = cost(0→1)   [came from 0]
                     dp[011][0] = cost(1→0)   [came from 1]
  101 (0,2 visited): dp[101][2] = cost(0→2)
                     dp[101][0] = cost(2→0)
  110 (1,2 visited): dp[110][2] = cost(1→2)
                     dp[110][1] = cost(2→1)

  111 (all): dp[111][2] = min(dp[011][0]+cost(0→2), dp[011][1]+cost(1→2))
             dp[111][1] = min(dp[101][0]+cost(0→1), dp[101][2]+cost(2→1))
             dp[111][0] = min(dp[110][1]+cost(1→0), dp[110][2]+cost(2→0))

Answer (min path, no return): min(dp[111][0], dp[111][1], dp[111][2])
Answer (TSP, return to 0):    min(dp[111][1]+cost(1→0), dp[111][2]+cost(2→0))
```

---

## Pitfalls

- **Bitmask DP: vertex 0 must be in all masks for TSP** — TSP with fixed start vertex 0 initializes `dp[1][0] = 0` and skips all masks where bit 0 is not set. Forgetting this filter processes invalid states (paths not containing vertex 0) and inflates the state space by 2× without benefit.
- **Backtracking: avoid revisiting start for cycle check** — when checking for a Hamiltonian cycle, after finding a full path verify that the last vertex has an edge back to the start. Do not mark the start as "unvisited" mid-backtrack and re-enter it as a cycle closer — this incorrectly allows revisiting the start vertex as an intermediate node.
- **Count paths: directed vs undirected** — the bitmask DP counts directed paths (A→B and B→A are different). For undirected Hamiltonian paths, divide by 2. For cycles, divide by 2n (each cycle is counted n times for n starting points, each in 2 directions). Be explicit about what "count" means in the problem statement.
- **INF addition overflow** — when computing `dp[mask][v] + cost[v][u]`, if `dp[mask][v] = INF` (unreachable) and `cost[v][u]` is also large, the sum overflows `int64`. Always check `dp[mask][v] != INF` before adding. Use `INF = 1e18` (half of LLONG_MAX) so that `INF + any_cost` does not overflow signed 64-bit.
- **Memory for bitmask DP** — `dp[2^n][n]` with `n=20` uses `2^20 * 20 * 8 bytes = 160 MB`. This is at the limit of typical memory constraints. For existence-only queries (boolean DP), use bitsets: store `dp[mask]` as an n-bit integer, pack n states into one word. This reduces memory 64×.
- **Backtracking: Warnsdorff only helps on structured graphs** — the Warnsdorff heuristic (prefer vertices with fewest onward neighbors) is highly effective for grid-like graphs (knight's tour) but may not help on dense or random graphs. For competitive programming on general graphs, simple DFS without Warnsdorff is often faster to implement and equally fast in practice.
- **Reconstruction needs parent array sized 2^n × n** — same memory as the DP table itself. For tight memory limits, omit the parent array and reconstruct by re-running the DP: at each step, find which previous state led to the current optimal value. This doubles the reconstruction time but halves the memory for the parent array.

---

## Complexity Summary

| Method | Time | Space | Practical limit |
|---|---|---|---|
| Backtracking (no pruning) | O(n!) | O(n) | n ≤ 12 |
| Backtracking (Warnsdorff) | Much less in practice | O(n) | n ≤ 20 on sparse |
| Bitmask DP (existence) | O(2^n * n²) | O(2^n * n) | n ≤ 20 |
| Bitmask DP (min cost/TSP) | O(2^n * n²) | O(2^n * n) | n ≤ 20 |
| Bitmask DP (count) | O(2^n * n²) | O(2^n * n) | n ≤ 20 |

---

## Conclusion

Hamiltonian path is **NP-complete** — the most famous example of a problem where no polynomial algorithm is known and none is expected:

- **Backtracking** is practical for small n (≤ 15 with pruning) and sparse graphs where dead-ends are found quickly. The Warnsdorff heuristic dramatically reduces search on grid-like structures.
- **Bitmask DP (Held-Karp)** is the standard exact algorithm for n ≤ 20, reducing O(n!) to O(2^n * n²) — a factor of n!/2^n/n² improvement, which is enormous for n=20 (3.6×10^18 / 4×10^8 ≈ 9×10^9 speedup).
- TSP (minimum Hamiltonian cycle) is the most important application — Held-Karp remains the best known exact algorithm after 60 years.

**Key takeaway:**  
For n ≤ 20, always use bitmask DP. For n > 20, use backtracking with aggressive pruning (connectivity check, degree-one forcing, Warnsdorff). The bitmask DP is a straightforward extension of the general bitmask DP framework: state = (visited_set, current_vertex), transition = try adding each unvisited neighbor. The only difference from standard bitmask DP is the memory pressure at n=20 — plan for 160 MB and use `int` instead of `long` when costs fit.
