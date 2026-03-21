# Dynamic Trees (Top Trees) — Extension of Link-Cut Trees

## Origin & Motivation

Top Trees were introduced by Alstrup, Holm, de Lichtenberg, and Thorup in 2003 as a generalization of Link-Cut Trees (Sleator–Tarjan, 1983). While Link-Cut Trees maintain a forest under link/cut operations and answer path queries for **associative, invertible** functions, Top Trees lift the invertibility restriction and additionally support **subtree aggregates** — making them applicable to a strictly broader class of problems.

The key abstraction: edges are grouped into **clusters** hierarchically. Each cluster represents a connected subgraph with at most two **boundary vertices** (endpoints exposed to the rest of the tree). Clusters are combined in a binary tree of merges, where each internal node stores the aggregate of its subtree of clusters. This mirrors the way a segment tree aggregates array intervals, but over dynamic tree paths and subtrees.

Complexity: **O(log n)** amortized per link, cut, and aggregate query.

---

## Relationship to Link-Cut Trees

| Feature | Link-Cut Trees | Top Trees |
|---|---|---|
| Path queries | Yes (invertible functions only) | Yes (any associative function) |
| Subtree queries | No | Yes |
| Path updates | Yes | Yes |
| Subtree updates | Yes (with care) | Yes |
| Implementation complexity | Medium | High |
| Constant factor | Small | Larger |
| Underlying structure | Splay trees on preferred paths | Compressed binary cluster tree |

Link-Cut Trees can be seen as a special case where the cluster tree is maintained implicitly via splay trees and only path aggregates are exposed.

---

## Where It Is Used

- Dynamic tree path aggregates (sum, min, max, GCD) without invertibility requirement
- Subtree aggregate queries on dynamic forests
- Heavy-light decomposition replacement in online settings
- Offline tree problems requiring reroot operations
- Competitive programming: dynamic connectivity on trees with subtree queries
- Theoretical algorithms: dynamic minimum spanning forest, 2-edge-connectivity

---

## Core Concepts

### Cluster

A **cluster** `C` is a connected subgraph of the tree with a designated set of **boundary vertices** (at most two). The cluster exposes an aggregate value `agg(C)` computed over all edges (and optionally vertices) inside it.

Three cluster types arise during the decomposition:

| Type | Description | Boundary vertices |
|---|---|---|
| Base cluster | Single edge | Both endpoints |
| Compress cluster | Two clusters sharing one internal boundary | Two outermost endpoints |
| Rake cluster | A path cluster with a subtree cluster attached at one end | Two endpoints of the path part |

### Top Tree Structure

The Top Tree is a binary tree over clusters. Its root represents the entire tree as one cluster. A **compress** node merges two clusters that share a boundary (path extension). A **rake** node merges a path cluster with a subtree hanging off one of its endpoints.

Maintaining this structure under link/cut reduces to a sequence of O(log n) split and join operations on the cluster tree, each taking O(1) after an O(log n) access path is exposed via splaying or similar.

### Aggregate Propagation

Each node stores:
- `path_agg` — aggregate over the path between its two boundary vertices
- `subtree_agg` — aggregate over all vertices/edges in the cluster including hanging subtrees

On compress: `path_agg = merge(left.path_agg, right.path_agg)`  
On rake: `path_agg = left.path_agg` (subtree attached, path unchanged), `subtree_agg` absorbs right child's aggregate

---

## Complexity Analysis

| Operation | Time |
|---|---|
| Link (add edge) | O(log n) amortized |
| Cut (remove edge) | O(log n) amortized |
| Path query (u → v) | O(log n) amortized |
| Subtree query (rooted at v) | O(log n) amortized |
| Path update | O(log n) amortized |
| Subtree update | O(log n) amortized |
| Space | O(n) |

The amortized bound follows from the same potential argument as splay trees when the cluster tree is maintained with splaying.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// -------------------------------------------------------
// Link-Cut Tree — practical foundation for Top Trees
// Supports: link, cut, path aggregate (sum), find_root, lca
// Extend push_up() to add subtree aggregates for full Top Trees
// -------------------------------------------------------

struct Node {
    int ch[2], par;
    long long val;    // value at this vertex
    long long path;   // path aggregate (sum of val on path to root of splay tree)
    long long sub;    // subtree aggregate (all nodes in virtual subtree)
    bool rev;         // lazy reverse flag
    bool is_root;     // true if this node is root of its splay tree

    Node() : ch{0,0}, par(0), val(0), path(0), sub(0), rev(false), is_root(true) {}
};

const int MAXN = 300005;
Node t[MAXN];

// Pull aggregate from children
// path = sum of val on the preferred path (splay tree)
// sub  = sum of val in the entire subtree represented by this splay node
void push_up(int v) {
    t[v].path = t[t[v].ch[0]].path + t[v].val + t[t[v].ch[1]].path;
    t[v].sub  = t[t[v].ch[0]].sub  + t[v].val + t[t[v].ch[1]].sub;
    // Note: virtual children contributions are accumulated separately
    // For full subtree aggregates, maintain an additional "virt" sum
}

void push_down(int v) {
    if (!t[v].rev) return;
    int l = t[v].ch[0], r = t[v].ch[1];
    swap(t[v].ch[0], t[v].ch[1]);
    if (l) { t[l].rev ^= 1; t[l].is_root = t[v].is_root; }
    if (r) { t[r].rev ^= 1; t[r].is_root = t[v].is_root; }
    t[v].rev = false;
}

bool is_root(int v) { return t[v].is_root; }

void rotate(int v) {
    int p = t[v].par, g = t[p].par;
    int side = (t[p].ch[1] == v);
    int c = t[v].ch[side ^ 1];

    if (!is_root(p)) t[g].ch[t[g].ch[1] == p] = v;
    t[v].ch[side ^ 1] = p;
    t[p].ch[side]     = c;

    if (c) t[c].par = p;
    t[v].par = g;
    t[p].par = v;

    t[p].is_root = false;
    t[v].is_root = (g == 0 || (t[g].ch[0] != v && t[g].ch[1] != v));

    push_up(p);
    push_up(v);
}

// Push lazy flags along the access path
void push_all(int v) {
    if (!is_root(v)) push_all(t[v].par);
    push_down(v);
}

void splay(int v) {
    push_all(v);
    while (!is_root(v)) {
        int p = t[v].par;
        if (!is_root(p)) {
            int g = t[p].par;
            if ((t[g].ch[0] == p) == (t[p].ch[0] == v)) rotate(p);
            else rotate(v);
        }
        rotate(v);
    }
    push_up(v);
}

// Access: makes v the root of its splay tree and connects it
// to the top of the represented tree's preferred path
void access(int v) {
    int last = 0;
    for (int u = v; u; u = t[u].par) {
        splay(u);
        t[u].is_root = false;          // u's right child becomes virtual
        t[u].ch[1] = last;
        push_up(u);
        last = u;
    }
    splay(v);
}

// Make v the root of its represented tree
void make_root(int v) {
    access(v);
    t[v].rev ^= 1;
    push_down(v);
}

// Find root of tree containing v
int find_root(int v) {
    access(v);
    while (t[v].ch[0]) { push_down(v); v = t[v].ch[0]; }
    splay(v);
    return v;
}

// Link v and u (they must be in different trees)
void link(int v, int u) {
    make_root(v);
    if (find_root(u) != v) {
        t[v].par = u;
        // For subtree aggregates: update u's virtual child sum here
    }
}

// Cut edge between v and u
void cut(int v, int u) {
    make_root(v);
    access(u);
    // After access(u), v should be the left child of u in the splay tree
    if (t[u].ch[0] == v && t[v].ch[1] == 0) {
        t[u].ch[0] = 0;
        t[v].par   = 0;
        t[u].is_root = true;
        push_up(u);
    }
}

// Path sum from v to u
long long path_query(int v, int u) {
    make_root(v);
    access(u);
    return t[u].path;
}

// Point update
void update(int v, long long val) {
    splay(v);
    t[v].val = val;
    push_up(v);
}

// -------------------------------------------------------
// Top Tree cluster abstraction (sketch)
// For a full Top Tree, each cluster node stores:
//   path_agg  — aggregate over boundary-to-boundary path
//   full_agg  — aggregate over entire cluster (path + hanging subtrees)
// Compress and rake operations:
// -------------------------------------------------------

struct ClusterAgg {
    long long path_sum = 0;  // sum on path between boundary vertices
    long long full_sum = 0;  // sum including subtree vertices
};

ClusterAgg compress(ClusterAgg L, ClusterAgg R) {
    // Two path clusters sharing one internal boundary: concatenate paths
    return { L.path_sum + R.path_sum, L.full_sum + R.full_sum };
}

ClusterAgg rake(ClusterAgg path_cluster, ClusterAgg subtree_cluster) {
    // Attach subtree_cluster as a hanging subtree at one endpoint of path_cluster
    // Path remains unchanged; full aggregate absorbs the hanging subtree
    return { path_cluster.path_sum, path_cluster.full_sum + subtree_cluster.full_sum };
}

// In a full Top Tree implementation:
//   - split(v, u) exposes the path from v to u as the root cluster
//   - join() rebuilds the cluster tree after modifications
//   - Each cluster recomputes agg bottom-up using compress/rake
// This yields subtree queries not achievable with plain Link-Cut Trees.

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    int n = 6;
    for (int i = 1; i <= n; i++) {
        t[i].val  = i;
        t[i].path = i;
        t[i].sub  = i;
        t[i].is_root = true;
    }

    // Build a path: 1 - 2 - 3 - 4 - 5 - 6
    link(1, 2); link(2, 3); link(3, 4); link(4, 5); link(5, 6);

    printf("Path sum 1->4: %lld\n", path_query(1, 4)); // 1+2+3+4 = 10

    // Cut edge between 3 and 4
    cut(3, 4);
    printf("Path sum 1->3: %lld\n", path_query(1, 3)); // 1+2+3 = 6

    // Relink
    link(3, 4);
    update(2, 10);
    printf("Path sum 1->4 after update: %lld\n", path_query(1, 4)); // 1+10+3+4 = 18

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;

// -------------------------------------------------------
// Link-Cut Tree in C# — foundation for Top Trees
// Supports: link, cut, path sum, make_root, find_root
// -------------------------------------------------------
public class LinkCutTree {
    private const int MAXN = 300005;

    private readonly int[]  ch   = new int[MAXN * 2];   // ch[v*2], ch[v*2+1]
    private readonly int[]  par  = new int[MAXN];
    private readonly long[] val  = new long[MAXN];
    private readonly long[] path = new long[MAXN];       // path aggregate
    private readonly bool[] rev  = new bool[MAXN];
    private readonly bool[] isRoot = new bool[MAXN];

    public LinkCutTree(int n) {
        for (int i = 1; i <= n; i++) isRoot[i] = true;
    }

    private int L(int v) => ch[v * 2];
    private int R(int v) => ch[v * 2 + 1];
    private void SetL(int v, int c) { ch[v * 2]     = c; }
    private void SetR(int v, int c) { ch[v * 2 + 1] = c; }

    private void PushUp(int v) {
        path[v] = path[L(v)] + val[v] + path[R(v)];
    }

    private void PushDown(int v) {
        if (!rev[v]) return;
        int l = L(v), r = R(v);
        (ch[v * 2], ch[v * 2 + 1]) = (r, l);   // swap children
        if (l != 0) rev[l] ^= true;
        if (r != 0) rev[r] ^= true;
        rev[v] = false;
    }

    private void PushAll(int v) {
        if (!isRoot[v]) PushAll(par[v]);
        PushDown(v);
    }

    private void Rotate(int v) {
        int p = par[v], g = par[p];
        bool side = R(p) == v;
        int c = side ? L(v) : R(v);

        if (!isRoot[p]) {
            if (L(g) == p) SetL(g, v);
            else           SetR(g, v);
        }
        if (side) { SetL(v, p); SetR(p, c); }
        else      { SetR(v, p); SetL(p, c); }

        if (c != 0) par[c] = p;
        par[v] = g;
        par[p] = v;

        isRoot[v] = isRoot[p];
        isRoot[p] = false;

        PushUp(p);
        PushUp(v);
    }

    private void Splay(int v) {
        PushAll(v);
        while (!isRoot[v]) {
            int p = par[v];
            if (!isRoot[p]) {
                int g = par[p];
                bool zigzig = (L(g) == p) == (L(p) == v);
                Rotate(zigzig ? p : v);
            }
            Rotate(v);
        }
        PushUp(v);
    }

    private void Access(int v) {
        int last = 0;
        for (int u = v; u != 0; u = par[u]) {
            Splay(u);
            isRoot[u] = false;
            SetR(u, last);
            PushUp(u);
            last = u;
        }
        Splay(v);
    }

    public void MakeRoot(int v) {
        Access(v);
        rev[v] ^= true;
        PushDown(v);
    }

    public int FindRoot(int v) {
        Access(v);
        while (L(v) != 0) { PushDown(v); v = L(v); }
        Splay(v);
        return v;
    }

    public void Link(int v, int u) {
        MakeRoot(v);
        if (FindRoot(u) != v)
            par[v] = u;
    }

    public void Cut(int v, int u) {
        MakeRoot(v);
        Access(u);
        if (L(u) == v && R(v) == 0) {
            SetL(u, 0);
            par[v] = 0;
            isRoot[u] = true;
            PushUp(u);
        }
    }

    public long PathQuery(int v, int u) {
        MakeRoot(v);
        Access(u);
        return path[u];
    }

    public void Update(int v, long value) {
        Splay(v);
        val[v] = value;
        PushUp(v);
    }

    // -------------------------------------------------------
    // Top Tree cluster aggregation (functional sketch)
    // -------------------------------------------------------
    public record ClusterAgg(long PathSum, long FullSum) {
        public static ClusterAgg Zero => new(0, 0);
    }

    public static ClusterAgg Compress(ClusterAgg L, ClusterAgg R) =>
        new(L.PathSum + R.PathSum, L.FullSum + R.FullSum);

    public static ClusterAgg Rake(ClusterAgg pathCluster, ClusterAgg subtreeCluster) =>
        new(pathCluster.PathSum, pathCluster.FullSum + subtreeCluster.FullSum);

    // -------------------------------------------------------
    // Usage
    // -------------------------------------------------------
    public static void Main() {
        int n = 6;
        var lct = new LinkCutTree(n);
        for (int i = 1; i <= n; i++) lct.val[i] = i;

        lct.Link(1, 2); lct.Link(2, 3);
        lct.Link(3, 4); lct.Link(4, 5); lct.Link(5, 6);

        Console.WriteLine($"Path sum 1->4: {lct.PathQuery(1, 4)}");   // 10

        lct.Cut(3, 4);
        Console.WriteLine($"Path sum 1->3: {lct.PathQuery(1, 3)}");   // 6

        lct.Link(3, 4);
        lct.Update(2, 10);
        Console.WriteLine($"Path sum 1->4 after update: {lct.PathQuery(1, 4)}"); // 18
    }
}
```

---

## Pitfalls

- **is_root semantics** — in a Link-Cut Tree, `is_root` means "this node is the root of its splay tree", not the root of the represented tree. Mixing these two concepts corrupts access paths. Every rotation must correctly propagate `is_root` to the new splay root.
- **push_down before any child access** — lazy flags (rev, lazy updates) must be pushed from root to leaf before splaying. Calling `push_all` recursively before `splay` handles this, but forgetting it causes incorrect aggregates that appear correct on small tests.
- **cut preconditions** — `cut(v, u)` is only valid if edge (v, u) exists. After `make_root(v)` and `access(u)`, v must be the immediate left child of u with no right child of v. Failing to verify this before cutting corrupts the forest silently.
- **Non-invertible functions and Link-Cut Trees** — a plain Link-Cut Tree's path aggregate relies on the fact that `agg(path) = agg(left_half) OP agg(right_half)`. For non-invertible operations (e.g., max, GCD), this still works for path queries but **not** for going from aggregate of whole to aggregate of subpath. Top Trees expose the cluster structure explicitly to avoid this.
- **Subtree aggregates require virtual child tracking** — a Link-Cut Tree node's splay tree only covers its preferred path. Subtrees hanging off via virtual edges are not reflected in `path`. To add subtree aggregates, maintain an extra `virt_sum` per node, updated when edges switch between preferred and virtual. Forgetting this gives wrong subtree results while path results remain correct.
- **Rake vs compress confusion** — in Top Trees, raking attaches a subtree to a path endpoint (the path does not extend); compressing extends the path through a shared boundary. Swapping these in the merge logic produces structurally valid trees with wrong aggregates that are extremely hard to debug.
- **Amortized complexity only** — individual operations can cost O(n) in the worst case (a single splay can touch O(n) nodes). The O(log n) guarantee is amortized over a sequence of operations. Do not use in contexts requiring strict per-operation bounds without modification.

---

## Conclusion

Dynamic Trees and Top Trees represent the **highest level of dynamic tree data structures**:

- Top Trees generalize Link-Cut Trees by supporting **any associative aggregate** over both paths and subtrees without requiring invertibility.
- Both achieve **O(log n) amortized** per link, cut, and query via splay-based restructuring.
- Link-Cut Trees are the practical choice for path queries; Top Trees are necessary when subtree aggregates or non-invertible path functions are required.
- The cluster abstraction (base / compress / rake) maps cleanly to a functional interface where only `merge_compress` and `merge_rake` need to change per problem.

**Key takeaway:**  
Use Link-Cut Trees when path aggregates are invertible and subtree queries are not needed — they are simpler and faster in practice. Graduate to Top Trees when the problem demands subtree aggregates or non-invertible path functions; the algorithmic cost is the same O(log n) but the implementation is significantly more involved.
