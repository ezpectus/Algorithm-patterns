# Strongly Connected Components — Kosaraju vs Tarjan

## Origin & Motivation

A **Strongly Connected Component (SCC)** of a directed graph is a maximal set of vertices such that every vertex is reachable from every other vertex in the set. Decomposing a directed graph into SCCs — the **condensation** — produces a DAG where each node represents one SCC. This is one of the most fundamental graph decompositions, enabling topological reasoning on graphs with cycles.

Two classic linear-time algorithms exist:

- **Kosaraju's algorithm** (Rao Kosaraju, 1978; independently Mihai Sharir, 1981) — two DFS passes: first on the original graph to get a reverse-finish-time ordering, then on the transposed graph following that ordering.

- **Tarjan's algorithm** (Robert Tarjan, 1972) — single DFS pass with a stack, using discovery times and low-link values to identify SCC roots when the stack unwinds.

Both run in **O(V + E)** time. The choice between them is a matter of implementation preference: Kosaraju is conceptually simpler (two clean DFS passes), Tarjan is more cache-friendly (single pass, one stack).

Complexity: **O(V + E)** time, **O(V + E)** space for both.

---

## Where SCCs Are Used

- Condensation DAG: reduce a directed graph with cycles to a DAG for topological processing
- 2-SAT: each variable and its negation form constraints; SCCs determine satisfiability
- Finding all cycles in a directed graph
- Detecting mutual dependencies (circular imports, deadlock detection)
- Competitive programming: connectivity queries, strongly connected subgraph problems
- Compiler optimization: identifying mutually recursive functions

---

## Kosaraju's Algorithm

### Idea

If we finish DFS on a vertex `v` later than vertex `u` in the original graph, then in the transposed graph `u` is not reachable from `v` in a different SCC. Running DFS on `G^T` in decreasing finish-time order assigns each new DFS tree to exactly one SCC.

### Two-Pass Structure

**Pass 1** — DFS on original graph G, record finish order (stack):
```
visited = {}
stack = []
for each unvisited vertex v:
    dfs1(v)

dfs1(v):
    visited.add(v)
    for each neighbor u of v:
        if u not visited: dfs1(u)
    stack.push(v)    // push after all neighbors processed
```

**Pass 2** — DFS on transposed graph G^T in reverse finish order:
```
comp = array of -1
scc_id = 0
while stack not empty:
    v = stack.pop()
    if comp[v] == -1:
        dfs2(v, scc_id)
        scc_id++

dfs2(v, c):
    comp[v] = c
    for each neighbor u of v in G^T:
        if comp[u] == -1: dfs2(u, c)
```

**Result:** `comp[v]` = SCC index of vertex v. SCCs numbered in **reverse topological order** of the condensation DAG (SCC 0 is a sink in the condensation).

---

## Tarjan's Algorithm

### Idea

Single DFS. Each vertex gets a **discovery time** `disc[v]` and a **low-link** value `low[v]` = smallest discovery time reachable from the subtree of `v` via at most one back/cross edge to an ancestor on the current DFS stack.

When `low[v] == disc[v]`, vertex `v` is the root of an SCC — pop everything from the stack down to `v`.

### Single-Pass Structure

```
timer = 0
stack = []
on_stack = {}
disc = array of -1
low  = array of -1
comp = array of -1
scc_id = 0

for each unvisited vertex v:
    dfs(v)

dfs(v):
    disc[v] = low[v] = timer++
    stack.push(v)
    on_stack.add(v)

    for each neighbor u of v:
        if disc[u] == -1:
            dfs(u)
            low[v] = min(low[v], low[u])
        elif on_stack[u]:
            low[v] = min(low[v], disc[u])

    if low[v] == disc[v]:   // v is SCC root
        while True:
            u = stack.pop()
            on_stack.remove(u)
            comp[u] = scc_id
        scc_id++
```

**Result:** `comp[v]` = SCC index. SCCs numbered in **topological order** of the condensation DAG (SCC 0 is a source).

---

## Comparison Table

| Property | Kosaraju | Tarjan |
|---|---|---|
| DFS passes | 2 | 1 |
| Extra graph needed | G^T (transposed) | No |
| Data structures | 2 visited arrays + stack | disc[], low[], on_stack[], stack |
| SCC numbering | Reverse topological | Topological |
| Implementation clarity | Simpler (two clean DFS) | Moderate (low-link logic) |
| Cache efficiency | Two passes over edges | One pass (better cache) |
| Recursive stack depth | 2 * O(V) | 1 * O(V) |
| Handles multigraphs | Yes | Yes |
| Time complexity | O(V + E) | O(V + E) |
| Space complexity | O(V + E) | O(V) extra |

---

## Complexity Analysis

| Step | Kosaraju | Tarjan |
|---|---|---|
| DFS pass 1 | O(V + E) | — |
| DFS pass 2 | O(V + E) | — |
| Single DFS | — | O(V + E) |
| Build G^T | O(V + E) | Not needed |
| Total time | O(V + E) | O(V + E) |
| Stack space | O(V) | O(V) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// KOSARAJU'S ALGORITHM
// ================================================================
struct Kosaraju {
    int n;
    vector<vector<int>> g;   // original graph
    vector<vector<int>> rg;  // reversed graph
    vector<bool> visited;
    vector<int>  order;      // finish order from pass 1
    vector<int>  comp;       // comp[v] = SCC id
    int num_sccs;

    Kosaraju(int n) : n(n), g(n), rg(n),
        visited(n, false), comp(n, -1), num_sccs(0) {}

    void add_edge(int u, int v) {
        g[u].push_back(v);
        rg[v].push_back(u);
    }

    // Pass 1: DFS on original graph, record finish order
    void dfs1(int v) {
        visited[v] = true;
        for (int u : g[v])
            if (!visited[u]) dfs1(u);
        order.push_back(v);
    }

    // Pass 2: DFS on reversed graph, assign components
    void dfs2(int v, int c) {
        comp[v] = c;
        for (int u : rg[v])
            if (comp[u] == -1) dfs2(u, c);
    }

    // Returns number of SCCs
    int solve() {
        // Pass 1
        for (int v = 0; v < n; v++)
            if (!visited[v]) dfs1(v);

        // Pass 2 in reverse finish order
        for (int i = n - 1; i >= 0; i--) {
            int v = order[i];
            if (comp[v] == -1) {
                dfs2(v, num_sccs);
                num_sccs++;
            }
        }
        return num_sccs;
        // comp[v]: SCCs numbered in REVERSE topological order of condensation
        // (comp=0 is a sink SCC in the condensation)
    }

    // Build condensation DAG
    // Returns adjacency list of the condensation (scc_count nodes)
    vector<vector<int>> condensation() {
        vector<set<int>> tmp(num_sccs);
        for (int u = 0; u < n; u++)
            for (int v : g[u])
                if (comp[u] != comp[v])
                    tmp[comp[u]].insert(comp[v]);

        vector<vector<int>> dag(num_sccs);
        for (int i = 0; i < num_sccs; i++)
            dag[i].assign(tmp[i].begin(), tmp[i].end());
        return dag;
    }
};

// ================================================================
// TARJAN'S ALGORITHM
// ================================================================
struct Tarjan {
    int n;
    vector<vector<int>> g;
    vector<int> disc, low, comp;
    vector<bool> on_stack;
    stack<int>   stk;
    int timer_, num_sccs;

    Tarjan(int n) : n(n), g(n),
        disc(n, -1), low(n), comp(n, -1),
        on_stack(n, false), timer_(0), num_sccs(0) {}

    void add_edge(int u, int v) { g[u].push_back(v); }

    void dfs(int v) {
        disc[v] = low[v] = timer_++;
        stk.push(v);
        on_stack[v] = true;

        for (int u : g[v]) {
            if (disc[u] == -1) {
                dfs(u);
                low[v] = min(low[v], low[u]);
            } else if (on_stack[u]) {
                low[v] = min(low[v], disc[u]);
            }
        }

        // v is SCC root: pop stack until v
        if (low[v] == disc[v]) {
            while (true) {
                int u = stk.top(); stk.pop();
                on_stack[u] = false;
                comp[u] = num_sccs;
                if (u == v) break;
            }
            num_sccs++;
        }
    }

    int solve() {
        for (int v = 0; v < n; v++)
            if (disc[v] == -1) dfs(v);
        return num_sccs;
        // comp[v]: SCCs numbered in TOPOLOGICAL order of condensation
        // (comp=0 is a source SCC)
    }

    // Build condensation DAG
    vector<vector<int>> condensation() {
        vector<set<int>> tmp(num_sccs);
        for (int u = 0; u < n; u++)
            for (int v : g[u])
                if (comp[u] != comp[v])
                    tmp[comp[u]].insert(comp[v]);

        vector<vector<int>> dag(num_sccs);
        for (int i = 0; i < num_sccs; i++)
            dag[i].assign(tmp[i].begin(), tmp[i].end());
        return dag;
    }
};

// ================================================================
// ITERATIVE TARJAN — avoids recursion stack overflow for large graphs
// ================================================================
struct TarjanIterative {
    int n;
    vector<vector<int>> g;
    vector<int> disc, low, comp;
    vector<bool> on_stack;
    stack<int>   stk;
    int timer_, num_sccs;

    TarjanIterative(int n) : n(n), g(n),
        disc(n,-1), low(n,0), comp(n,-1),
        on_stack(n,false), timer_(0), num_sccs(0) {}

    void add_edge(int u, int v) { g[u].push_back(v); }

    int solve() {
        // Explicit DFS stack: (vertex, iterator_index)
        vector<int> call_stack, edge_idx(n, 0);

        for (int start = 0; start < n; start++) {
            if (disc[start] != -1) continue;
            call_stack.push_back(start);
            disc[start] = low[start] = timer_++;
            stk.push(start);
            on_stack[start] = true;

            while (!call_stack.empty()) {
                int v = call_stack.back();

                if (edge_idx[v] < (int)g[v].size()) {
                    int u = g[v][edge_idx[v]++];
                    if (disc[u] == -1) {
                        call_stack.push_back(u);
                        disc[u] = low[u] = timer_++;
                        stk.push(u);
                        on_stack[u] = true;
                    } else if (on_stack[u]) {
                        low[v] = min(low[v], disc[u]);
                    }
                } else {
                    // All neighbors processed: unwind
                    call_stack.pop_back();
                    if (!call_stack.empty()) {
                        int parent = call_stack.back();
                        low[parent] = min(low[parent], low[v]);
                    }
                    if (low[v] == disc[v]) {
                        while (true) {
                            int u = stk.top(); stk.pop();
                            on_stack[u] = false;
                            comp[u] = num_sccs;
                            if (u == v) break;
                        }
                        num_sccs++;
                    }
                }
            }
        }
        return num_sccs;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // Example graph with 3 SCCs:
    // 0 -> 1 -> 2 -> 0 (cycle: SCC {0,1,2})
    // 2 -> 3 -> 4 -> 3 (cycle: SCC {3,4})
    // 2 -> 5            (SCC {5})

    printf("=== Kosaraju ===\n");
    {
        Kosaraju kos(6);
        kos.add_edge(0,1); kos.add_edge(1,2); kos.add_edge(2,0);
        kos.add_edge(2,3); kos.add_edge(3,4); kos.add_edge(4,3);
        kos.add_edge(2,5);

        int cnt = kos.solve();
        printf("SCCs: %d\n", cnt);
        printf("comp[v]: ");
        for (int v = 0; v < 6; v++) printf("%d ", kos.comp[v]);
        printf("\n");
        // Note: Kosaraju numbers in REVERSE topological order
        // SCC containing {5} is a sink -> lowest comp id

        auto dag = kos.condensation();
        printf("Condensation edges:\n");
        for (int i = 0; i < cnt; i++)
            for (int j : dag[i])
                printf("  SCC%d -> SCC%d\n", i, j);
    }

    printf("\n=== Tarjan ===\n");
    {
        Tarjan tar(6);
        tar.add_edge(0,1); tar.add_edge(1,2); tar.add_edge(2,0);
        tar.add_edge(2,3); tar.add_edge(3,4); tar.add_edge(4,3);
        tar.add_edge(2,5);

        int cnt = tar.solve();
        printf("SCCs: %d\n", cnt);
        printf("comp[v]: ");
        for (int v = 0; v < 6; v++) printf("%d ", tar.comp[v]);
        printf("\n");
        // Tarjan numbers in TOPOLOGICAL order
        // SCC {0,1,2} is a source -> lowest comp id

        auto dag = tar.condensation();
        printf("Condensation edges:\n");
        for (int i = 0; i < cnt; i++)
            for (int j : dag[i])
                printf("  SCC%d -> SCC%d\n", i, j);
    }

    printf("\n=== Iterative Tarjan (large graph safe) ===\n");
    {
        TarjanIterative ti(6);
        ti.add_edge(0,1); ti.add_edge(1,2); ti.add_edge(2,0);
        ti.add_edge(2,3); ti.add_edge(3,4); ti.add_edge(4,3);
        ti.add_edge(2,5);
        int cnt = ti.solve();
        printf("SCCs: %d\n", cnt);
        printf("comp[v]: ");
        for (int v = 0; v < 6; v++) printf("%d ", ti.comp[v]);
        printf("\n");
    }

    // 2-SAT connection: each SCC determines satisfiability
    printf("\n=== 2-SAT via SCC ===\n");
    // x0 OR x1: (not x0 -> x1) AND (not x1 -> x0)
    // Literal encoding: pos(i)=2i, neg(i)=2i+1
    {
        int vars = 2;
        Tarjan sat(2 * vars);
        // x0 OR x1: clause encoding
        // not x0 -> x1: edge (1 -> 2)
        // not x1 -> x0: edge (3 -> 0)
        sat.add_edge(1, 2); // neg(x0) -> pos(x1)
        sat.add_edge(3, 0); // neg(x1) -> pos(x0)
        sat.solve();
        printf("2-SAT satisfiable: ");
        bool ok = true;
        for (int i = 0; i < vars; i++) {
            if (sat.comp[2*i] == sat.comp[2*i+1]) { ok = false; break; }
        }
        printf("%s\n", ok ? "YES" : "NO");
        if (ok) {
            printf("Assignment: ");
            for (int i = 0; i < vars; i++)
                // var i is true if pos(i) has HIGHER comp id in Tarjan
                printf("x%d=%d ", i, sat.comp[2*i] > sat.comp[2*i+1] ? 1 : 0);
            printf("\n");
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

// ================================================================
// KOSARAJU'S ALGORITHM
// ================================================================
public class Kosaraju {
    private readonly int n;
    private readonly List<int>[] g, rg;
    private readonly bool[] visited;
    private readonly List<int> order = new();
    public readonly int[] Comp;
    public int NumSccs { get; private set; }

    public Kosaraju(int n) {
        this.n = n;
        g = new List<int>[n]; rg = new List<int>[n];
        for (int i = 0; i < n; i++) { g[i] = new(); rg[i] = new(); }
        visited = new bool[n];
        Comp = new int[n];
        Array.Fill(Comp, -1);
    }

    public void AddEdge(int u, int v) { g[u].Add(v); rg[v].Add(u); }

    private void Dfs1(int v) {
        visited[v] = true;
        foreach (int u in g[v]) if (!visited[u]) Dfs1(u);
        order.Add(v);
    }

    private void Dfs2(int v, int c) {
        Comp[v] = c;
        foreach (int u in rg[v]) if (Comp[u] == -1) Dfs2(u, c);
    }

    public int Solve() {
        for (int v = 0; v < n; v++) if (!visited[v]) Dfs1(v);
        for (int i = n - 1; i >= 0; i--) {
            int v = order[i];
            if (Comp[v] == -1) { Dfs2(v, NumSccs); NumSccs++; }
        }
        return NumSccs;
    }

    public List<int>[] Condensation() {
        var tmp = new HashSet<int>[NumSccs];
        for (int i = 0; i < NumSccs; i++) tmp[i] = new();
        for (int u = 0; u < n; u++)
            foreach (int v in g[u])
                if (Comp[u] != Comp[v]) tmp[Comp[u]].Add(Comp[v]);
        var dag = new List<int>[NumSccs];
        for (int i = 0; i < NumSccs; i++)
            dag[i] = new List<int>(tmp[i]);
        return dag;
    }
}

// ================================================================
// TARJAN'S ALGORITHM
// ================================================================
public class Tarjan {
    private readonly int n;
    private readonly List<int>[] g;
    private readonly int[] disc, low;
    public  readonly int[] Comp;
    private readonly bool[] onStack;
    private readonly Stack<int> stk = new();
    private int timer_, scc_;
    public int NumSccs => scc_;

    public Tarjan(int n) {
        this.n = n;
        g = new List<int>[n];
        for (int i = 0; i < n; i++) g[i] = new();
        disc = new int[n]; Array.Fill(disc, -1);
        low  = new int[n];
        Comp = new int[n]; Array.Fill(Comp, -1);
        onStack = new bool[n];
    }

    public void AddEdge(int u, int v) => g[u].Add(v);

    private void Dfs(int v) {
        disc[v] = low[v] = timer_++;
        stk.Push(v); onStack[v] = true;

        foreach (int u in g[v]) {
            if (disc[u] == -1) { Dfs(u); low[v] = Math.Min(low[v], low[u]); }
            else if (onStack[u]) low[v] = Math.Min(low[v], disc[u]);
        }

        if (low[v] == disc[v]) {
            while (true) {
                int u = stk.Pop(); onStack[u] = false; Comp[u] = scc_;
                if (u == v) break;
            }
            scc_++;
        }
    }

    public int Solve() {
        for (int v = 0; v < n; v++) if (disc[v] == -1) Dfs(v);
        return scc_;
    }

    public List<int>[] Condensation() {
        var tmp = new HashSet<int>[scc_];
        for (int i = 0; i < scc_; i++) tmp[i] = new();
        for (int u = 0; u < n; u++)
            foreach (int v in g[u])
                if (Comp[u] != Comp[v]) tmp[Comp[u]].Add(Comp[v]);
        var dag = new List<int>[scc_];
        for (int i = 0; i < scc_; i++) dag[i] = new List<int>(tmp[i]);
        return dag;
    }
}

// ================================================================
// ITERATIVE TARJAN — safe for deep graphs
// ================================================================
public class TarjanIterative {
    private readonly int n;
    private readonly List<int>[] g;
    private readonly int[] disc, low, edgeIdx;
    public  readonly int[] Comp;
    private readonly bool[] onStack;
    private readonly Stack<int> stk = new();
    private int timer_, scc_;
    public int NumSccs => scc_;

    public TarjanIterative(int n) {
        this.n = n;
        g = new List<int>[n];
        for (int i = 0; i < n; i++) g[i] = new();
        disc    = new int[n]; Array.Fill(disc, -1);
        low     = new int[n];
        edgeIdx = new int[n];
        Comp    = new int[n]; Array.Fill(Comp, -1);
        onStack = new bool[n];
    }

    public void AddEdge(int u, int v) => g[u].Add(v);

    public int Solve() {
        var callStack = new Stack<int>();
        for (int start = 0; start < n; start++) {
            if (disc[start] != -1) continue;
            callStack.Push(start);
            disc[start] = low[start] = timer_++;
            stk.Push(start); onStack[start] = true;

            while (callStack.Count > 0) {
                int v = callStack.Peek();
                if (edgeIdx[v] < g[v].Count) {
                    int u = g[v][edgeIdx[v]++];
                    if (disc[u] == -1) {
                        callStack.Push(u);
                        disc[u] = low[u] = timer_++;
                        stk.Push(u); onStack[u] = true;
                    } else if (onStack[u]) {
                        low[v] = Math.Min(low[v], disc[u]);
                    }
                } else {
                    callStack.Pop();
                    if (callStack.Count > 0) {
                        int parent = callStack.Peek();
                        low[parent] = Math.Min(low[parent], low[v]);
                    }
                    if (low[v] == disc[v]) {
                        while (true) {
                            int u = stk.Pop(); onStack[u] = false; Comp[u] = scc_;
                            if (u == v) break;
                        }
                        scc_++;
                    }
                }
            }
        }
        return scc_;
    }
}

public class Program {
    public static void Main() {
        // Graph: {0,1,2} cycle, {3,4} cycle, {5} singleton
        void Test(string name, int[] comp, int cnt, List<int>[] dag) {
            Console.WriteLine($"=== {name} ===");
            Console.WriteLine($"SCCs: {cnt}");
            Console.WriteLine($"comp: {string.Join(" ", comp)}");
            Console.WriteLine("DAG edges:");
            for (int i = 0; i < cnt; i++)
                foreach (int j in dag[i])
                    Console.WriteLine($"  SCC{i} -> SCC{j}");
        }

        var k = new Kosaraju(6);
        k.AddEdge(0,1); k.AddEdge(1,2); k.AddEdge(2,0);
        k.AddEdge(2,3); k.AddEdge(3,4); k.AddEdge(4,3); k.AddEdge(2,5);
        int kn = k.Solve();
        Test("Kosaraju", k.Comp, kn, k.Condensation());

        Console.WriteLine();
        var t = new Tarjan(6);
        t.AddEdge(0,1); t.AddEdge(1,2); t.AddEdge(2,0);
        t.AddEdge(2,3); t.AddEdge(3,4); t.AddEdge(4,3); t.AddEdge(2,5);
        int tn = t.Solve();
        Test("Tarjan", t.Comp, tn, t.Condensation());

        Console.WriteLine();
        var ti = new TarjanIterative(6);
        ti.AddEdge(0,1); ti.AddEdge(1,2); ti.AddEdge(2,0);
        ti.AddEdge(2,3); ti.AddEdge(3,4); ti.AddEdge(4,3); ti.AddEdge(2,5);
        Console.WriteLine($"=== Iterative Tarjan ===");
        Console.WriteLine($"SCCs: {ti.Solve()}");
        Console.WriteLine($"comp: {string.Join(" ", ti.Comp)}");
    }
}
```

---

## Condensation DAG

After computing SCCs, the **condensation** is the DAG where each node represents one SCC and each edge `(C_i, C_j)` exists when some edge in the original graph goes from a vertex in `C_i` to a vertex in `C_j`.

Properties:
- Always a DAG (no cycles — each cycle is inside an SCC)
- Topological order of the condensation = SCC ordering (Tarjan: direct, Kosaraju: reversed)
- Source SCCs in the condensation = SCCs reachable from all others
- Sink SCCs = SCCs from which nothing else is reachable

Use cases:
- Find all vertices reachable from a given SCC
- Longest path in the original graph via DAG DP on condensation
- 2-SAT: check if any variable and its negation are in the same SCC

---

## SCC Numbering Convention

| Algorithm | Comp=0 is... | Topological order |
|---|---|---|
| Tarjan | Source (first finished SCC) | comp[u] < comp[v] if u before v |
| Kosaraju | Sink (last in finish order) | comp[u] > comp[v] if u before v |

For 2-SAT: variable `x_i` is true iff `comp[pos(i)] > comp[neg(i)]` in Tarjan, or `comp[pos(i)] < comp[neg(i)]` in Kosaraju (since Kosaraju reverses topological order). **Always check the convention of your specific implementation before applying 2-SAT.**

---

## Pitfalls

- **Kosaraju: DFS pass 2 must use transposed graph** — a common mistake is running the second DFS on the original graph. The second pass must use `rg` (reversed edges). Using `g` again finds DFS trees on the original graph, which do not correspond to SCCs.
- **Tarjan: low-link uses disc[u], not low[u] for non-tree edges** — when processing edge `(v, u)` where u is already on the stack (back/cross edge), update `low[v] = min(low[v], disc[u])`, not `low[u]`. Using `low[u]` for cross edges is wrong and causes incorrect SCC identification in graphs that are not DAGs. The correct rule: use `low[u]` only for tree edges (when u was just discovered from v).
- **on_stack vs visited** — Tarjan requires `on_stack[u]` (currently in DFS stack), not just `visited[u]` (ever visited). Cross edges to vertices already popped from the stack must not update `low[v]` — only edges to vertices still on the stack are relevant. Confusing these two conditions merges distinct SCCs.
- **Recursive Tarjan overflows for large graphs** — with V = 10^5 vertices in a linear chain, the recursion depth reaches 10^5, overflowing the default stack. Always use the iterative variant for competitive programming. The recursive version is correct but only safe for small graphs (V ≤ a few thousand).
- **SCC numbering direction** — Tarjan numbers SCCs in topological order (comp=0 is first/source), while Kosaraju numbers in reverse topological order (comp=0 is last/sink). For 2-SAT, the assignment rule depends on which direction "later in topological order" corresponds to. Mixing up the direction gives wrong variable assignments that still pass trivial tests.
- **Condensation: must deduplicate edges** — the condensation may have multiple original edges between the same pair of SCCs. Use a `set` or mark visited pairs to produce a simple DAG. A multi-edge condensation is still correct for most purposes but wastes space and causes duplicate processing.
- **Self-loops do not affect SCC structure** — a self-loop `(v, v)` means v is in a cycle of length 1 with itself. It does not merge v with any other vertex. In Tarjan, `on_stack[v]` is true when the self-loop is processed and `disc[v]` is the current value — `low[v]` correctly stays as `disc[v]`, and v forms its own SCC.

---

## Conclusion

Both Kosaraju and Tarjan are **O(V + E) SCC algorithms** with identical asymptotic complexity but different implementation structure:

- **Kosaraju** is easier to understand and implement correctly: two clean DFS passes with a clear conceptual separation. It requires building the transposed graph but the code is straightforward. Prefer it for readability.
- **Tarjan** requires only one DFS pass and no transposed graph. It is faster in practice due to better cache behavior and is the standard choice in competitive programming for its compactness.
- The **iterative Tarjan** variant is essential for production use on large graphs — recursive Tarjan overflows the call stack on graphs with long chains.
- Both produce the condensation DAG — the key structure for downstream algorithms (2-SAT, longest path, reachability in directed graphs with cycles).

**Key takeaway:**  
For competitive programming, use Tarjan (iterative variant for safety). For understanding and teaching, use Kosaraju. For 2-SAT specifically, use Tarjan and apply `x_i = true iff comp[2i] > comp[2i+1]`. The only subtle correctness point in Tarjan is the low-link update rule: `low[u]` for tree edges, `disc[u]` for back edges to stack vertices, and nothing for edges to already-popped vertices.
