# Euler Path / Euler Circuit

## Origin & Motivation

The Euler path and circuit problem originates from Leonhard Euler's 1736 solution to the **Königsberg Bridge Problem** — the first theorem in graph theory. Euler proved that a walk traversing every edge exactly once exists if and only if the graph satisfies specific degree conditions.

- **Euler circuit (Eulerian circuit):** A closed walk that visits every edge exactly once and returns to the starting vertex.
- **Euler path (Eulerian path):** An open walk that visits every edge exactly once, starting and ending at different vertices.

The key insight is that existence depends only on vertex degrees, and **Hierholzer's algorithm** constructs the path/circuit in **O(V + E)** time by greedily following edges and splicing together sub-tours.

Complexity: **O(V + E)** time, **O(V + E)** space.

---

## Where It Is Used

- Route planning: traverse every road/edge exactly once (Chinese Postman Problem)
- DNA sequencing: de Bruijn graphs, genome assembly
- Graph drawing: single-stroke drawing problems
- Competitive programming: edge traversal, sequence reconstruction
- Puzzle solving: "draw this figure without lifting the pen"
- Circuit board inspection: traverse every connection exactly once

---

## Existence Conditions

### Undirected Graph

| Condition | Result |
|---|---|
| All vertices have even degree AND graph is connected (ignoring isolated vertices) | Euler **circuit** exists |
| Exactly 2 vertices have odd degree AND graph is connected | Euler **path** exists (between the two odd-degree vertices) |
| More than 2 odd-degree vertices | Neither exists |

### Directed Graph

| Condition | Result |
|---|---|
| Every vertex: `in_degree == out_degree` AND graph is connected (as undirected) | Euler **circuit** exists |
| Exactly one vertex: `out_degree - in_degree = 1` (start), one vertex: `in_degree - out_degree = 1` (end), all others equal, AND graph is connected | Euler **path** exists |
| Otherwise | Neither exists |

---

## Hierholzer's Algorithm

**Idea:** Start from any valid vertex. Follow edges greedily (removing used edges), forming a cycle. When stuck (no unused edges from current vertex), backtrack and splice in sub-cycles encountered along the way.

**Efficient implementation using a stack:**

```
path = []
stack = [start]

while stack not empty:
    v = stack.top()
    if v has unused edges:
        pick any unused edge (v, u)
        mark edge as used
        stack.push(u)
    else:
        stack.pop()
        path.append(v)

reverse(path) → Euler path/circuit
```

Each edge is pushed and popped exactly once → O(E) total.

**Key:** Use adjacency list with a pointer (iterator) to the next unused edge per vertex, advancing it as edges are used. This avoids re-scanning used edges.

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Degree check (existence) | O(V + E) | O(V) |
| Hierholzer's algorithm | O(V + E) | O(V + E) |
| Handling multigraphs | O(V + E) | O(V + E) |
| Total | O(V + E) | O(V + E) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// EULER PATH / CIRCUIT — Undirected Graph
// ================================================================
struct EulerUndirected {
    int n;
    vector<vector<pair<int,int>>> g; // {neighbor, edge_id}
    vector<bool> used_edge;
    int num_edges;

    EulerUndirected(int n) : n(n), g(n), num_edges(0) {}

    void add_edge(int u, int v) {
        int id = num_edges++;
        g[u].push_back({v, id});
        g[v].push_back({u, id});
        used_edge.push_back(false);
    }

    // Check existence and return start vertex.
    // Returns -1 if neither Euler path nor circuit exists.
    // Sets is_circuit = true if circuit exists, false if path.
    int check(bool& is_circuit) {
        // Check connectivity (ignoring isolated vertices)
        int start = -1;
        for (int v = 0; v < n; v++)
            if (!g[v].empty()) { start = v; break; }
        if (start == -1) { is_circuit = true; return 0; } // empty graph

        // BFS/DFS connectivity check on non-isolated vertices
        vector<bool> visited(n, false);
        queue<int> q;
        q.push(start); visited[start] = true;
        int reachable = 1;
        while (!q.empty()) {
            int v = q.front(); q.pop();
            for (auto [u, eid] : g[v]) {
                if (!visited[u]) { visited[u] = true; reachable++; q.push(u); }
            }
        }
        // Count non-isolated vertices
        int non_isolated = 0;
        for (int v = 0; v < n; v++) if (!g[v].empty()) non_isolated++;
        if (reachable != non_isolated) return -1; // disconnected

        // Count odd-degree vertices
        vector<int> odd_verts;
        for (int v = 0; v < n; v++)
            if (g[v].size() % 2 == 1) odd_verts.push_back(v);

        if (odd_verts.size() == 0) {
            is_circuit = true;
            return start; // any vertex works as start
        } else if (odd_verts.size() == 2) {
            is_circuit = false;
            return odd_verts[0]; // start from one odd-degree vertex
        } else {
            return -1; // no Euler path or circuit
        }
    }

    // Hierholzer's algorithm — O(V + E)
    vector<int> solve() {
        bool is_circuit;
        int start = check(is_circuit);
        if (start == -1) return {}; // doesn't exist

        // Edge iterator per vertex
        vector<int> it(n, 0); // it[v] = index of next unused edge in g[v]

        vector<int> path;
        stack<int> stk;
        stk.push(start);

        while (!stk.empty()) {
            int v = stk.top();
            // Advance iterator past used edges
            while (it[v] < (int)g[v].size() &&
                   used_edge[g[v][it[v]].second])
                it[v]++;

            if (it[v] == (int)g[v].size()) {
                // No unused edges: add to path
                stk.pop();
                path.push_back(v);
            } else {
                // Follow next unused edge
                auto [u, eid] = g[v][it[v]];
                used_edge[eid] = true;
                it[v]++;
                stk.push(u);
            }
        }

        reverse(path.begin(), path.end());

        // Validate: path should have E+1 vertices
        if ((int)path.size() != num_edges + 1) return {}; // not all edges used
        return path;
    }
};

// ================================================================
// EULER PATH / CIRCUIT — Directed Graph
// ================================================================
struct EulerDirected {
    int n;
    vector<vector<pair<int,int>>> g; // {neighbor, edge_id}
    vector<int> in_deg, out_deg;
    int num_edges;

    EulerDirected(int n)
        : n(n), g(n), in_deg(n,0), out_deg(n,0), num_edges(0) {}

    void add_edge(int u, int v) {
        int id = num_edges++;
        g[u].push_back({v, id});
        out_deg[u]++;
        in_deg[v]++;
    }

    // Returns start vertex or -1 if no Euler path/circuit exists.
    int check(bool& is_circuit) {
        // Connectivity check (treat as undirected)
        int start = -1;
        for (int v = 0; v < n; v++)
            if (out_deg[v] + in_deg[v] > 0) { start = v; break; }
        if (start == -1) { is_circuit = true; return 0; }

        vector<bool> visited(n, false);
        // Build undirected adjacency for connectivity
        vector<vector<int>> ug(n);
        for (int u = 0; u < n; u++)
            for (auto [v, eid] : g[u]) {
                ug[u].push_back(v); ug[v].push_back(u);
            }
        queue<int> q;
        q.push(start); visited[start] = true;
        int reachable = 1;
        while (!q.empty()) {
            int v = q.front(); q.pop();
            for (int u : ug[v])
                if (!visited[u]) { visited[u] = true; reachable++; q.push(u); }
        }
        int non_isolated = 0;
        for (int v = 0; v < n; v++)
            if (out_deg[v] + in_deg[v] > 0) non_isolated++;
        if (reachable != non_isolated) return -1;

        // Check degree conditions
        int start_cand = -1, end_cand = -1;
        bool valid = true;
        for (int v = 0; v < n; v++) {
            int diff = out_deg[v] - in_deg[v];
            if (diff == 0) continue;
            else if (diff ==  1) { if (start_cand != -1) { valid=false; break; } start_cand = v; }
            else if (diff == -1) { if (end_cand   != -1) { valid=false; break; } end_cand   = v; }
            else { valid = false; break; }
        }
        if (!valid) return -1;

        if (start_cand == -1 && end_cand == -1) {
            is_circuit = true;
            return start; // circuit
        } else if (start_cand != -1 && end_cand != -1) {
            is_circuit = false;
            return start_cand; // path from start_cand to end_cand
        }
        return -1;
    }

    vector<int> solve() {
        bool is_circuit;
        int start = check(is_circuit);
        if (start == -1) return {};

        vector<int> it(n, 0);
        vector<int> path;
        stack<int> stk;
        stk.push(start);

        while (!stk.empty()) {
            int v = stk.top();
            if (it[v] == (int)g[v].size()) {
                stk.pop();
                path.push_back(v);
            } else {
                auto [u, eid] = g[v][it[v]++];
                stk.push(u);
            }
        }

        reverse(path.begin(), path.end());
        if ((int)path.size() != num_edges + 1) return {};
        return path;
    }
};

// ================================================================
// APPLICATION: Sequence reconstruction from k-mers (de Bruijn)
// Given set of k-mers, reconstruct sequence via Euler path on
// de Bruijn graph: nodes = (k-1)-mers, edges = k-mers
// ================================================================
string reconstruct_sequence(const vector<string>& kmers) {
    if (kmers.empty()) return "";
    int k = kmers[0].size();

    // Map (k-1)-mers to node indices
    map<string, int> node_id;
    int cnt = 0;
    auto get_id = [&](const string& s) -> int {
        auto it = node_id.find(s);
        if (it == node_id.end()) { node_id[s] = cnt++; return cnt-1; }
        return it->second;
    };

    for (const string& km : kmers) {
        get_id(km.substr(0, k-1));
        get_id(km.substr(1, k-1));
    }

    EulerDirected ed(cnt);
    for (const string& km : kmers) {
        int u = get_id(km.substr(0, k-1));
        int v = get_id(km.substr(1, k-1));
        ed.add_edge(u, v);
    }

    auto path = ed.solve();
    if (path.empty()) return ""; // no valid reconstruction

    // Reconstruct sequence from path
    // Build reverse map: id -> string
    vector<string> id_to_str(cnt);
    for (auto& [s, id] : node_id) id_to_str[id] = s;

    string result = id_to_str[path[0]];
    for (int i = 1; i < (int)path.size(); i++)
        result += id_to_str[path[i]].back();
    return result;
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Undirected: Königsberg-style graph
    {
        printf("=== Undirected Euler Circuit ===\n");
        // Simple: 0-1-2-3-0 + 0-2 + 1-3 (all even degree)
        EulerUndirected eu(4);
        eu.add_edge(0,1); eu.add_edge(1,2); eu.add_edge(2,3);
        eu.add_edge(3,0); eu.add_edge(0,2); eu.add_edge(1,3);
        auto path = eu.solve();
        if (!path.empty()) {
            printf("Circuit: ");
            for (int v : path) printf("%d ", v);
            printf("\n");
        } else {
            printf("No Euler circuit\n");
        }
    }

    {
        printf("\n=== Undirected Euler Path ===\n");
        // Path: 0-1-2-3-1 (vertices 0 and 3 have odd degree)
        EulerUndirected eu(4);
        eu.add_edge(0,1); eu.add_edge(1,2);
        eu.add_edge(2,3); eu.add_edge(3,1);
        auto path = eu.solve();
        if (!path.empty()) {
            printf("Path: ");
            for (int v : path) printf("%d ", v);
            printf("\n");
        }
    }

    {
        printf("\n=== No Euler Path (Königsberg) ===\n");
        // All 4 vertices have odd degree
        EulerUndirected eu(4);
        eu.add_edge(0,1); eu.add_edge(0,2); eu.add_edge(0,3);
        eu.add_edge(1,2); eu.add_edge(2,3);
        auto path = eu.solve();
        printf("%s\n", path.empty() ? "No Euler path/circuit" : "Found path");
    }

    // Directed: circuit
    {
        printf("\n=== Directed Euler Circuit ===\n");
        EulerDirected ed(3);
        ed.add_edge(0,1); ed.add_edge(1,2); ed.add_edge(2,0);
        ed.add_edge(0,2); ed.add_edge(2,1); ed.add_edge(1,0);
        auto path = ed.solve();
        if (!path.empty()) {
            printf("Circuit: ");
            for (int v : path) printf("%d ", v);
            printf("\n");
        }
    }

    // Directed: path
    {
        printf("\n=== Directed Euler Path ===\n");
        // 0->1->2->3, 1->3->1 (out[0]=1,in[0]=0; out[3]=1,in[3]=2...)
        // Let's use simple: 0->1->2->0->3->1
        EulerDirected ed(4);
        ed.add_edge(0,1); ed.add_edge(1,2); ed.add_edge(2,0);
        ed.add_edge(0,3); ed.add_edge(3,1);
        // Check: out/in: 0:2/1, 1:1/2, 2:1/1, 3:1/1
        // out[0]-in[0]=1 (start), in[1]-out[1]=1 (end) -> path from 0 to 1
        auto path = ed.solve();
        if (!path.empty()) {
            printf("Path: ");
            for (int v : path) printf("%d ", v);
            printf("\n");
        }
    }

    // de Bruijn sequence reconstruction
    {
        printf("\n=== de Bruijn sequence reconstruction ===\n");
        vector<string> kmers = {"ATCG", "TCGA", "CGAT", "GATC"};
        string seq = reconstruct_sequence(kmers);
        printf("Reconstructed: %s\n", seq.c_str()); // ATCGATC or similar
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
// EULER UNDIRECTED
// ================================================================
public class EulerUndirected {
    private readonly int n;
    private readonly List<(int v, int id)>[] g;
    private readonly List<bool> usedEdge = new();
    private int numEdges;

    public EulerUndirected(int n) {
        this.n = n;
        g = new List<(int,int)>[n];
        for (int i = 0; i < n; i++) g[i] = new();
    }

    public void AddEdge(int u, int v) {
        int id = numEdges++;
        g[u].Add((v, id)); g[v].Add((u, id));
        usedEdge.Add(false);
    }

    private int Check(out bool isCircuit) {
        isCircuit = false;
        int start = -1;
        for (int v = 0; v < n; v++) if (g[v].Count > 0) { start = v; break; }
        if (start == -1) { isCircuit = true; return 0; }

        var visited = new bool[n];
        var q = new Queue<int>();
        q.Enqueue(start); visited[start] = true; int reach = 1;
        while (q.Count > 0) {
            int v = q.Dequeue();
            foreach (var (u, _) in g[v])
                if (!visited[u]) { visited[u] = true; reach++; q.Enqueue(u); }
        }
        int nonIso = 0;
        for (int v = 0; v < n; v++) if (g[v].Count > 0) nonIso++;
        if (reach != nonIso) return -1;

        var odd = new List<int>();
        for (int v = 0; v < n; v++) if (g[v].Count % 2 == 1) odd.Add(v);

        if      (odd.Count == 0) { isCircuit = true;  return start; }
        else if (odd.Count == 2) { isCircuit = false; return odd[0]; }
        return -1;
    }

    public List<int> Solve() {
        int start = Check(out bool isCircuit);
        if (start == -1) return new();

        var it   = new int[n];
        var path = new List<int>();
        var stk  = new Stack<int>();
        stk.Push(start);

        while (stk.Count > 0) {
            int v = stk.Peek();
            while (it[v] < g[v].Count && usedEdge[g[v][it[v]].id]) it[v]++;
            if (it[v] == g[v].Count) { stk.Pop(); path.Add(v); }
            else {
                var (u, eid) = g[v][it[v]++];
                usedEdge[eid] = true;
                stk.Push(u);
            }
        }

        path.Reverse();
        return path.Count == numEdges + 1 ? path : new();
    }
}

// ================================================================
// EULER DIRECTED
// ================================================================
public class EulerDirected {
    private readonly int n;
    private readonly List<(int v, int id)>[] g;
    private readonly int[] inDeg, outDeg;
    private int numEdges;

    public EulerDirected(int n) {
        this.n = n;
        g      = new List<(int,int)>[n];
        inDeg  = new int[n];
        outDeg = new int[n];
        for (int i = 0; i < n; i++) g[i] = new();
    }

    public void AddEdge(int u, int v) {
        int id = numEdges++;
        g[u].Add((v, id)); outDeg[u]++; inDeg[v]++;
    }

    private int Check(out bool isCircuit) {
        isCircuit = false;
        int start = -1;
        for (int v = 0; v < n; v++)
            if (outDeg[v] + inDeg[v] > 0) { start = v; break; }
        if (start == -1) { isCircuit = true; return 0; }

        // Connectivity as undirected
        var ug = new List<int>[n];
        for (int i = 0; i < n; i++) ug[i] = new();
        for (int u = 0; u < n; u++)
            foreach (var (v, _) in g[u]) { ug[u].Add(v); ug[v].Add(u); }

        var vis = new bool[n];
        var q   = new Queue<int>();
        q.Enqueue(start); vis[start] = true; int reach = 1;
        while (q.Count > 0) {
            int v = q.Dequeue();
            foreach (int u in ug[v])
                if (!vis[u]) { vis[u] = true; reach++; q.Enqueue(u); }
        }
        int nonIso = 0;
        for (int v = 0; v < n; v++) if (outDeg[v]+inDeg[v] > 0) nonIso++;
        if (reach != nonIso) return -1;

        int sc = -1, ec = -1; bool valid = true;
        for (int v = 0; v < n; v++) {
            int d = outDeg[v] - inDeg[v];
            if      (d ==  1) { if (sc != -1) { valid=false; break; } sc = v; }
            else if (d == -1) { if (ec != -1) { valid=false; break; } ec = v; }
            else if (d !=  0) { valid = false; break; }
        }
        if (!valid) return -1;

        if (sc == -1 && ec == -1) { isCircuit = true;  return start; }
        if (sc != -1 && ec != -1) { isCircuit = false; return sc; }
        return -1;
    }

    public List<int> Solve() {
        int start = Check(out bool isCircuit);
        if (start == -1) return new();

        var it   = new int[n];
        var path = new List<int>();
        var stk  = new Stack<int>();
        stk.Push(start);

        while (stk.Count > 0) {
            int v = stk.Peek();
            if (it[v] == g[v].Count) { stk.Pop(); path.Add(v); }
            else { var (u, _) = g[v][it[v]++]; stk.Push(u); }
        }

        path.Reverse();
        return path.Count == numEdges + 1 ? path : new();
    }
}

public class Program {
    public static void Main() {
        // Undirected circuit
        var eu = new EulerUndirected(4);
        eu.AddEdge(0,1); eu.AddEdge(1,2); eu.AddEdge(2,3);
        eu.AddEdge(3,0); eu.AddEdge(0,2); eu.AddEdge(1,3);
        var p1 = eu.Solve();
        Console.WriteLine($"Undirected circuit: {string.Join("->", p1)}");

        // Directed circuit
        var ed = new EulerDirected(3);
        ed.AddEdge(0,1); ed.AddEdge(1,2); ed.AddEdge(2,0);
        ed.AddEdge(0,2); ed.AddEdge(2,1); ed.AddEdge(1,0);
        var p2 = ed.Solve();
        Console.WriteLine($"Directed circuit: {string.Join("->", p2)}");

        // Directed path
        var ep = new EulerDirected(4);
        ep.AddEdge(0,1); ep.AddEdge(1,2); ep.AddEdge(2,0);
        ep.AddEdge(0,3); ep.AddEdge(3,1);
        var p3 = ep.Solve();
        Console.WriteLine($"Directed path: {string.Join("->", p3)}");
    }
}
```

---

## Key Observations

### Why Hierholzer Works

When we pop a vertex with no unused edges and add it to the path, we are effectively inserting a complete subcircuit. The stack maintains the invariant that:
- The current stack represents a partial path from start to `stk.top()`
- Every vertex popped to `path` has been fully explored

Since every edge is used exactly once and each vertex is popped exactly once, the total work is O(V + E).

### Edge Iterator Trick

Maintaining an iterator `it[v]` pointing to the next unused edge avoids re-scanning already-used edges. Without this, each vertex scan could revisit O(degree) used edges, giving O(E * max_degree) worst case instead of O(E).

---

## Pitfalls

- **Directed graph: track in-degree and out-degree separately** — the existence condition for directed Euler path is `out[start] - in[start] = 1` and `in[end] - out[end] = 1`. Using just degree (in+out) as in the undirected case gives wrong conditions for directed graphs.
- **Undirected multigraph: mark edges by ID, not vertex** — with multiple edges between the same pair, marking a neighbor vertex as "visited" instead of the specific edge causes all parallel edges to be skipped after the first. Always use edge IDs and `used_edge[eid]`.
- **Connectivity check on the correct subgraph** — isolated vertices (degree 0) should be ignored in the connectivity check. A graph with isolated vertices is Eulerian if the non-isolated part is connected and satisfies degree conditions. Including isolated vertices in the connectivity count falsely reports disconnected graphs.
- **Directed connectivity check uses undirected version** — for directed Euler path/circuit existence, connectivity is checked on the **underlying undirected graph** (ignoring edge directions). Using reachability in the directed sense (ignoring reverse edges) gives wrong results — a strongly connected directed graph satisfies strong connectivity, but we need weak connectivity here.
- **Remaining stack after DFS** — in the undirected case, if the start vertex has even degree (circuit), the stack should be empty after the path is fully built. For a path (two odd-degree vertices), same. If the stack is not empty, some edges were unreachable — the graph is disconnected and was incorrectly passed to the algorithm.
- **Validation: path length must be E+1** — always verify `path.size() == num_edges + 1`. If not all edges were traversed (due to connectivity issues not caught by the check), the path is incomplete. This catches subtle bugs where the connectivity check passes but the graph has bridges that trap the DFS.

---

## Complexity Summary

| Graph type | Existence check | Construction | Total |
|---|---|---|---|
| Undirected | O(V + E) | O(V + E) | O(V + E) |
| Directed | O(V + E) | O(V + E) | O(V + E) |
| de Bruijn (k-mer) | O(k * E) | O(V + E) | O(k * E) |

---

## Conclusion

Euler paths and circuits are **one of the few NP-hard-looking problems that are actually polynomial** — existence is checkable in O(V+E) via degree conditions, and construction via Hierholzer's algorithm is O(V+E) with the edge-iterator optimization.

- The key algorithmic idea is the **stack-based Hierholzer**: greedily follow edges, and when stuck, splice the dead-end vertex into the final path — naturally assembling subcircuits in correct order.
- Directed and undirected versions differ only in the existence conditions (degree balance vs. even degree) and connectivity check (weak vs. undirected).
- The de Bruijn graph application makes Euler paths the foundation of **DNA sequence assembly** — one of the most impactful algorithmic applications in bioinformatics.

**Key takeaway:**  
Implement Hierholzer with an edge iterator per vertex (`it[v]` advancing past used edges). This single optimization makes the algorithm O(V+E) instead of O(E * max_degree). The existence check is always done before construction — save time by returning early if degree conditions fail, before building the adjacency structure or running DFS.
