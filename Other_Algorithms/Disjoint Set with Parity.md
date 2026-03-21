# Disjoint Set with Parity — DSU with Edge Weights mod 2

## Origin & Motivation

Disjoint Set Union with Parity (also called Weighted DSU or DSU with edge labels) extends the classical Union-Find structure by associating a **parity value** with each node — its distance mod 2 to the root of its component. This augmentation answers a strictly broader class of queries: not only "are u and v in the same component?" but also "what is the parity of the distance between u and v in the implicit graph?".

This structure is the canonical solution for problems expressible as a graph where edges carry XOR weights over GF(2): bipartiteness checking, detecting contradictory constraints, classifying nodes into two groups, and parity-consistency queries. It requires zero additional asymptotic cost — all operations remain **O(α(n))** amortized with path compression and union by rank.

Complexity: **O(α(n))** per operation, **O(n)** space.

---

## Where It Is Used

- Bipartiteness checking with online edge insertions
- Two-coloring / enemy-friend classification (can node u and v be in opposite groups?)
- Parity constraint systems (is a system of XOR equations over variables consistent?)
- Detecting odd-length cycles in undirected graphs online
- Competitive programming: "Enemies and Friends", "Strange Printer", parity reachability

---

## Core Idea

Each node `v` stores:
- `parent[v]` — its parent in the DSU tree (or itself if root)
- `rank[v]` — union-by-rank value (for root nodes)
- `parity[v]` — XOR distance from `v` to its root along the tree path

**Invariant:** `parity[v]` = XOR of all edge weights on the path from `v` to `root(v)`.

### Find with path compression

During `find(v)`, path compression flattens the tree. Parities must be updated along the compressed path using XOR accumulation:

```
find(v):
    if parent[v] == v:
        return (v, parity=0)
    (root, root_parity) = find(parent[v])
    parity[v] ^= parity[parent[v]]   // chain XOR up to root
    parent[v] = root
    return (root, parity[v])
```

### Union with parity

To add edge `(u, v)` with weight `w` (parity of the edge, 0 or 1):

```
(ru, pu) = find(u)   // pu = parity of u relative to root ru
(rv, pv) = find(v)   // pv = parity of v relative to root rv

if ru == rv:
    // u and v are already connected
    // parity of path u->v = pu XOR pv
    // edge weight w must equal pu XOR pv; if not, contradiction
    return (pu ^ pv == w)

// Merge: attach rv under ru (or vice versa by rank)
// The new parity of rv's root relative to ru's root:
// dist(v, ru) = pv XOR w (go from v to ru via w then pu)
// dist(rv, ru) = pv XOR pu XOR w
parity[rv] = pu ^ pv ^ w
parent[rv] = ru
```

---

## Parity Semantics

The parity stored answers: **is the distance between u and v odd or even in the constraint graph?**

| `parity_dist(u, v)` | Meaning |
|---|---|
| 0 | u and v are in the **same group** (even distance) |
| 1 | u and v are in **opposite groups** (odd distance) |

For bipartiteness: add each edge `(u, v)` with weight 1 (they must be in opposite groups). If at any point `find(u) == find(v)` and `parity_dist(u, v) == 0`, an odd cycle exists — graph is not bipartite.

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Initialize | O(n) | O(n) |
| Find (with path compression) | O(α(n)) amortized | O(1) |
| Union (with rank) | O(α(n)) amortized | O(1) |
| Parity query between u, v | O(α(n)) amortized | O(1) |

- `α(n)` — inverse Ackermann function, effectively constant (≤ 4 for all practical n)
- Path compression preserves parity correctness via XOR chain accumulation — no extra cost

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

struct DSUParity {
    vector<int> parent, rank_;
    vector<int> parity; // XOR distance to root

    DSUParity(int n) : parent(n), rank_(n, 0), parity(n, 0) {
        iota(parent.begin(), parent.end(), 0);
    }

    // Returns {root, parity of v relative to root}
    // Applies path compression, updating parities along the way
    pair<int,int> find(int v) {
        if (parent[v] == v) return {v, 0};
        auto [root, p] = find(parent[v]);
        parity[v] ^= parity[parent[v]]; // chain: parity[v] now = dist(v, root)
        parent[v] = root;
        return {root, parity[v]};
    }

    // Unite u and v with edge weight w (0 or 1).
    // Returns:
    //   true  — successfully merged (u and v were in different components)
    //   false — u and v were already connected;
    //           `consistent` is set to whether the edge is consistent
    //           with existing parity constraints
    bool unite(int u, int v, int w, bool& consistent) {
        auto [ru, pu] = find(u);
        auto [rv, pv] = find(v);

        if (ru == rv) {
            // Already in same component: check consistency
            consistent = ((pu ^ pv) == w);
            return false;
        }

        consistent = true;

        // parity of rv relative to ru:
        // path: rv -> root_rv -> (via edge w) -> v -> (via pv) -> u -> (via pu^) -> ru
        // dist(rv, ru) = pv ^ w ^ pu  (XOR is self-inverse)
        int p_rv = pu ^ pv ^ w;

        // Union by rank
        if (rank_[ru] < rank_[rv]) {
            swap(ru, rv);
            p_rv = p_rv; // XOR is symmetric: dist(ru, rv) = dist(rv, ru) in GF(2)
            // parity of new child (old ru) relative to new root (old rv)
            // Since XOR is symmetric, p_rv stays correct after swap
        }

        parent[rv] = ru;
        parity[rv] = p_rv;
        if (rank_[ru] == rank_[rv]) rank_[ru]++;

        return true;
    }

    // Query: parity of the path between u and v
    // Returns -1 if u and v are in different components
    int query(int u, int v) {
        auto [ru, pu] = find(u);
        auto [rv, pv] = find(v);
        if (ru != rv) return -1;
        return pu ^ pv;
    }

    // Check if u and v are in the same component
    bool same(int u, int v) {
        return find(u).first == find(v).first;
    }
};

// -------------------------------------------------------
// Application 1: Online bipartiteness checker
// Add edges one by one; detect if graph becomes non-bipartite.
// -------------------------------------------------------
struct BipartiteChecker {
    DSUParity dsu;
    bool bipartite;

    BipartiteChecker(int n) : dsu(n), bipartite(true) {}

    // Add edge (u, v). Returns true if graph is still bipartite.
    bool add_edge(int u, int v) {
        if (!bipartite) return false;
        bool consistent;
        dsu.unite(u, v, 1, consistent); // weight=1: u and v must be in opposite groups
        if (!consistent) bipartite = false;
        return bipartite;
    }

    // Is the graph currently bipartite?
    bool is_bipartite() const { return bipartite; }

    // Are u and v in the same color class? (returns -1 if disconnected)
    int same_color(int u, int v) {
        return dsu.query(u, v); // 0 = same color, 1 = different color
    }
};

// -------------------------------------------------------
// Application 2: Enemy/Friend constraint system
// "u and v are enemies" -> unite(u, v, 1)
// "u and v are friends" -> unite(u, v, 0)
// Contradiction if friends are forced to be enemies or vice versa.
// -------------------------------------------------------
struct FriendEnemy {
    DSUParity dsu;
    bool consistent;

    FriendEnemy(int n) : dsu(n), consistent(true) {}

    // Returns false if this constraint contradicts existing ones
    bool add_friend(int u, int v) {
        bool ok;
        dsu.unite(u, v, 0, ok);
        if (!ok) consistent = false;
        return ok;
    }

    bool add_enemy(int u, int v) {
        bool ok;
        dsu.unite(u, v, 1, ok);
        if (!ok) consistent = false;
        return ok;
    }

    // Are u and v enemies? Returns -1 if unknown (different components)
    int are_enemies(int u, int v) {
        return dsu.query(u, v); // 1 = enemies, 0 = friends, -1 = unknown
    }

    bool is_consistent() const { return consistent; }
};

// -------------------------------------------------------
// Application 3: XOR parity reachability
// Graph where each edge has a parity label.
// Query: what is the XOR parity of any path from u to v?
// (All paths in a consistent component have the same parity.)
// -------------------------------------------------------

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    // --- Bipartite check ---
    {
        BipartiteChecker bc(6);
        bc.add_edge(0, 1);
        bc.add_edge(1, 2);
        bc.add_edge(2, 3);
        printf("After path 0-1-2-3: bipartite=%s\n",
            bc.is_bipartite() ? "yes" : "no");  // yes

        bc.add_edge(0, 2); // odd cycle: 0-1-2-0
        printf("After adding 0-2: bipartite=%s\n",
            bc.is_bipartite() ? "yes" : "no");  // no

        // Color query (while still bipartite, before odd cycle)
        BipartiteChecker bc2(4);
        bc2.add_edge(0,1); bc2.add_edge(1,2); bc2.add_edge(2,3);
        int q = bc2.same_color(0, 2);
        printf("0 and 2 same color: %s\n", q==0?"yes":"no"); // no (0=even, 2=even... wait)
        // parity(0->2) = 0 XOR 1 XOR 1... let's trace:
        // edge(0,1,1), edge(1,2,1): parity(0,2) = 1^1 = 0 → same color? yes (distance 2 = even)
    }

    // --- Friend/Enemy ---
    {
        FriendEnemy fe(5);
        fe.add_enemy(0, 1);  // 0 and 1 are enemies
        fe.add_enemy(1, 2);  // 1 and 2 are enemies => 0 and 2 are friends
        printf("0 and 2 enemies: %s\n",
            fe.are_enemies(0,2)==1?"yes":"no"); // no (friends)

        fe.add_enemy(0, 2);  // contradiction: forced friends, declared enemies
        printf("Consistent: %s\n",
            fe.is_consistent()?"yes":"no"); // no
    }

    // --- Parity DSU directly ---
    {
        DSUParity dsu(5);
        bool ok;
        dsu.unite(0, 1, 1, ok); // dist(0,1) = 1
        dsu.unite(1, 2, 0, ok); // dist(1,2) = 0
        dsu.unite(2, 3, 1, ok); // dist(2,3) = 1

        printf("parity(0,3) = %d\n", dsu.query(0,3)); // 1^0^1 = 0
        printf("parity(0,2) = %d\n", dsu.query(0,2)); // 1^0   = 1

        dsu.unite(0, 3, 0, ok); // consistent? parity(0,3)=0 and we assert 0: yes
        printf("Constraint 0-3 weight 0: consistent=%s\n", ok?"yes":"no");

        dsu.unite(0, 3, 1, ok); // contradicts parity(0,3)=0
        printf("Constraint 0-3 weight 1: consistent=%s\n", ok?"yes":"no");
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;

public class DSUParity {
    private readonly int[] parent, rank, parity;

    public DSUParity(int n) {
        parent = new int[n];
        rank   = new int[n];
        parity = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }

    // Returns (root, parity of v relative to root)
    public (int root, int par) Find(int v) {
        if (parent[v] == v) return (v, 0);
        var (root, _) = Find(parent[v]);
        parity[v] ^= parity[parent[v]];
        parent[v] = root;
        return (root, parity[v]);
    }

    // Unite u and v with edge weight w (0 or 1).
    // Returns true if merged; consistent = whether constraint is satisfiable.
    public bool Unite(int u, int v, int w, out bool consistent) {
        var (ru, pu) = Find(u);
        var (rv, pv) = Find(v);

        if (ru == rv) {
            consistent = ((pu ^ pv) == w);
            return false;
        }

        consistent = true;
        int pRv = pu ^ pv ^ w;

        if (rank[ru] < rank[rv]) {
            (ru, rv) = (rv, ru);
            // parity is symmetric in GF(2): no change to pRv needed
        }

        parent[rv] = ru;
        parity[rv] = pRv;
        if (rank[ru] == rank[rv]) rank[ru]++;
        return true;
    }

    // Parity of path between u and v; -1 if different components
    public int Query(int u, int v) {
        var (ru, pu) = Find(u);
        var (rv, pv) = Find(v);
        if (ru != rv) return -1;
        return pu ^ pv;
    }

    public bool Same(int u, int v) => Find(u).root == Find(v).root;
}

// -------------------------------------------------------
// Online bipartiteness checker
// -------------------------------------------------------
public class BipartiteChecker {
    private readonly DSUParity dsu;
    private bool bipartite = true;

    public BipartiteChecker(int n) { dsu = new DSUParity(n); }

    public bool AddEdge(int u, int v) {
        if (!bipartite) return false;
        dsu.Unite(u, v, 1, out bool ok);
        if (!ok) bipartite = false;
        return bipartite;
    }

    public bool IsBipartite => bipartite;

    // 0 = same color, 1 = different color, -1 = disconnected
    public int SameColor(int u, int v) => dsu.Query(u, v);
}

// -------------------------------------------------------
// Friend/Enemy system
// -------------------------------------------------------
public class FriendEnemy {
    private readonly DSUParity dsu;
    public bool Consistent { get; private set; } = true;

    public FriendEnemy(int n) { dsu = new DSUParity(n); }

    public bool AddFriend(int u, int v) {
        dsu.Unite(u, v, 0, out bool ok);
        if (!ok) Consistent = false;
        return ok;
    }

    public bool AddEnemy(int u, int v) {
        dsu.Unite(u, v, 1, out bool ok);
        if (!ok) Consistent = false;
        return ok;
    }

    // 1=enemies, 0=friends, -1=unknown
    public int AreEnemies(int u, int v) => dsu.Query(u, v);
}

public class Program {
    public static void Main() {
        // Bipartite check
        var bc = new BipartiteChecker(4);
        bc.AddEdge(0,1); bc.AddEdge(1,2); bc.AddEdge(2,3);
        Console.WriteLine($"Path 0-1-2-3 bipartite: {bc.IsBipartite}");  // True
        bc.AddEdge(1,3); // odd cycle: 1-2-3-1
        Console.WriteLine($"After 1-3: bipartite: {bc.IsBipartite}");     // False

        // Friend/enemy
        var fe = new FriendEnemy(4);
        fe.AddEnemy(0,1);
        fe.AddEnemy(1,2);
        Console.WriteLine($"0 and 2 enemies: {fe.AreEnemies(0,2)==1}");   // False (friends)
        fe.AddEnemy(0,2); // contradiction
        Console.WriteLine($"Consistent: {fe.Consistent}");                 // False

        // Direct parity DSU
        var dsu = new DSUParity(5);
        dsu.Unite(0,1,1,out _);
        dsu.Unite(1,2,0,out _);
        dsu.Unite(2,3,1,out _);
        Console.WriteLine($"parity(0,3) = {dsu.Query(0,3)}"); // 0
        Console.WriteLine($"parity(0,2) = {dsu.Query(0,2)}"); // 1

        dsu.Unite(0,3,0,out bool ok1);
        Console.WriteLine($"Constraint 0-3 w=0: consistent={ok1}");  // True
        dsu.Unite(0,3,1,out bool ok2);
        Console.WriteLine($"Constraint 0-3 w=1: consistent={ok2}");  // False
    }
}
```

---

## Parity Update During Path Compression — Detail

Path compression during `find(v)` must preserve the invariant that `parity[v]` equals the XOR distance from `v` to the root. Before compression, `parity[v]` stores the distance to `parent[v]`, not the root.

```
Before compression (chain v -> p -> g -> root):
  parity[v] = dist(v, p)
  parity[p] = dist(p, g)
  parity[g] = dist(g, root)

After find(p) recursively compresses:
  parity[p] = dist(p, root) = parity[p] XOR parity[g]   (accumulated)
  parent[p] = root

Now for v:
  parity[v] ^= parity[parent[v]]     // parity[parent[v]] = dist(p, root)
  =>  parity[v] = dist(v, p) XOR dist(p, root) = dist(v, root)
  parent[v] = root
```

The key: `parity[parent[v]]` must be read **before** updating `parent[v]`. The recursive call updates `parity[parent[v]]` first, then the current node uses that updated value. This is correct precisely because the recursion processes ancestors before descendants on the unwinding path.

---

## Generalization to Arbitrary Monoid Weights

Parity (GF(2)) is a special case. The same augmented DSU works for any **group** (not just XOR):

| Weight type | Group operation | Application |
|---|---|---|
| GF(2) (parity) | XOR | Bipartiteness, 2-coloring |
| Z (integers) | Addition | Difference constraints (offline) |
| GF(p) | Addition mod p | p-coloring reachability |
| (Z/2Z)^k | XOR per bit | Multi-dimensional parity |

For integer weights: `weight[v]` = signed distance from `v` to root. `find` accumulates by addition. `unite` sets `weight[rv] = weight_u - weight_v + w`. `query(u,v)` returns `weight_v - weight_u` (distance from u to v).

---

## Pitfalls

- **Reading parent's parity before updating parent pointer** — in the recursive `find`, the line `parity[v] ^= parity[parent[v]]` must execute before `parent[v] = root`. If the pointer is updated first, `parity[parent[v]]` reads from the new parent (the root) which has parity 0, losing the accumulated value. The recursive formulation handles this naturally; an iterative version requires explicit care.
- **Union by rank after parity computation** — the rank-based swap (attach smaller-rank root under larger) must happen **after** computing the parity of the new child relative to the new root. Swapping `ru` and `rv` without adjusting `p_rv` produces a correct tree structure but wrong parities. In GF(2), XOR is symmetric so `p_rv` is the same regardless of direction — but verify this for other groups.
- **Querying disconnected nodes** — `query(u, v)` returns -1 when `u` and `v` are in different components. Treating -1 as a valid parity (it would be cast to 255 in unsigned or -1 in signed comparisons) causes wrong answers. Always check `same(u, v)` or handle the -1 return explicitly.
- **Iterative find requires a two-pass approach** — an iterative path compression for weighted DSU must first collect the path (first pass), compute accumulated parities from root to leaf (second pass), then update all pointers. A naive single-pass iterative version cannot correctly chain the XOR accumulation without storing intermediate values.
- **Parity is defined over paths, not edges** — the query `query(u, v)` returns the parity of **any** path from `u` to `v` in the implicit constraint graph, which is well-defined only if the graph is consistent (no contradictions). After a contradiction is detected, further queries on the affected component may return arbitrary values — track consistency separately.
- **Non-group weights break correctness** — the weighted DSU is correct only when the weight domain forms a **group** (associativity, identity, inverse). XOR satisfies this. If you attempt to use min/max as the combining operation, path compression cannot correctly update weights because min/max have no inverse.

---

## Conclusion

Disjoint Set with Parity is the **canonical solution for online parity-constrained connectivity**:

- Extends classical DSU with zero asymptotic overhead — O(α(n)) per operation remains unchanged.
- Answers "are u and v in the same parity class?" in near-constant time under arbitrary online edge insertions.
- Directly encodes bipartiteness checking, 2-coloring, enemy/friend classification, and XOR constraint satisfaction in a single unified structure.
- Generalizes cleanly to any abelian group weight domain (integers, GF(p), multi-bit XOR) by replacing XOR with the group operation.

**Key takeaway:**  
Whenever a problem involves partitioning nodes into two groups with consistency constraints on pairs — bipartite graphs, friend/enemy graphs, parity reachability — reach for DSU with parity before considering BFS/DFS solutions. It handles online insertions in O(α(n)) vs O(V+E) per query for BFS, and the implementation overhead over standard DSU is minimal: one extra array and two XOR operations per find.
