# Link/Cut Trees

## Origin & Motivation

**Link/Cut Trees** were introduced by Daniel Sleator and Robert Tarjan in 1983 to maintain a dynamic forest of rooted trees under structural changes while supporting path queries and updates in **O(log n)** amortized time per operation.

**The problem they solve:** Static trees support path queries via Heavy-Light Decomposition in O(log² n), but the tree structure cannot change. When edges are inserted and deleted online — as in dynamic graph algorithms, network flow, or LCT-based MST maintenance — a purely static structure fails. Link/Cut Trees handle all of: `link(u,v)`, `cut(u,v)`, `path_query(u,v)`, `path_update(u,v)`, `makeRoot(u)`, `findRoot(u)`, and `connected(u,v)` — all in O(log n) amortized.

**The key idea:** Partition the edges of each represented tree into **preferred paths** — chains of edges currently "in use". Each preferred path is stored as a splay tree keyed by depth. The collection of splay trees for one represented tree forms the **auxiliary forest**. When `access(v)` is called it restructures the preferred paths to expose the root-to-v path, splaying v to the top. This restructuring amortizes to O(log n) per operation via the access lemma.

Complexity: **O(log n)** amortized per operation, **O(n)** space.

---

## Where It Is Used

- Dynamic connectivity in forests (online link/cut with path queries)
- Network flow: maintaining blocking-flow trees (Sleator-Tarjan's original application)
- Dynamic MST: offline/online minimum spanning forest under edge insertions/deletions
- Competitive programming: path sum/max/min on trees that change topology
- LCT-based solutions for problems equivalent to dynamic tree path aggregation
- Euler tour alternatives when subtree queries are not needed

---

## Core Structure

### Preferred Path Decomposition

Each node has at most one **preferred child** — the child on its currently preferred path. Preferred paths are contiguous root-to-leaf segments of the represented tree. Each preferred path is stored as a splay tree where the in-order traversal gives nodes in order of increasing depth.

```
Represented tree:         Preferred paths (example):
      1                     Splay tree A: [1 - 2 - 4]
     / \                    Splay tree B: [3]
    2   3                   Splay tree C: [5]
   / \
  4   5
```

### Node Structure

```
Node {
    ch[2]   // left and right children in the splay tree
    p       // parent pointer (either splay parent or path-parent pointer)
    val     // value stored at this node
    sum     // aggregate (sum/min/max) of the splay subtree
    rev     // lazy reversal flag (for makeRoot)
}

isRoot(x): x is the root of its splay tree
         ⟺ x->p == null  OR  x->p->ch[0] != x AND x->p->ch[1] != x
```

The `p` pointer serves two roles:
- If `x` is not the splay root: `p` is the splay tree parent (standard BST parent).
- If `x` is the splay root: `p` is the **path-parent pointer** to the node just above this preferred path in the represented tree (a "virtual" edge, not counted as a splay edge).

---

## Operations

### access(x)

Exposes the path from the represented-tree root to `x`. After `access(x)`, `x` is the root of its splay tree and `x->ch[1] == nullptr` (no preferred child below).

```
access(x):
    last = null
    y = x
    while y != null:
        splay(y)
        y->ch[1] = last      // detach old preferred child, attach last
        y->pull()            // recompute aggregate
        last = y
        y = y->p             // follow path-parent pointer upward
    splay(x)                 // bring x to splay root
    return last              // last non-null = LCA information
```

### makeRoot(x)

Reroots the represented tree at `x`. Implemented by reversing the path from current root to `x`:

```
makeRoot(x):
    access(x)      // expose root-to-x path
    x->rev ^= 1    // flip the path (reverse in-order = reverse depth order)
    x->push()      // propagate immediately
```

The `rev` flag, when propagated, swaps `ch[0]` and `ch[1]` of a splay node — effectively reversing the depth order of the path, making `x` the new root.

### link(u, v)

Connect node `u` as a child of `v`. Precondition: `u` and `v` are in different represented trees.

```
link(u, v):
    makeRoot(u)          // make u the root of its tree
    u->p = v             // attach u under v via path-parent pointer
```

### cut(u, v)

Remove the edge between `u` and `v`. Precondition: edge `(u,v)` exists.

```
cut(u, v):
    makeRoot(u)          // make u the root
    access(v)            // expose path u → ... → v
    // now u == v->ch[0] (u is the left child of v in splay tree)
    v->ch[0] = null
    u->p = null
    v->pull()
```

### findRoot(x)

Find the root of `x`'s represented tree.

```
findRoot(x):
    access(x)            // expose path to current root
    while x->ch[0]:      // go to leftmost (shallowest) node
        x->push()
        x = x->ch[0]
    splay(x)             // bring root to top for amortization
    return x
```

### pathSum / pathQuery(u, v)

Aggregate value along the path from `u` to `v`:

```
pathQuery(u, v):
    makeRoot(u)    // u becomes root
    access(v)      // expose u → v path
    return v->sum  // aggregate is stored at splay root v
```

---

## Amortized Analysis

The **access lemma** proves O(log n) amortized per `access` call. Define the **rank** of a node as `r(x) = log₂(size of splay subtree of x)`. The potential function is `Φ = Σ r(x)` over all nodes.

Each `access` traverses a sequence of preferred path switches. The number of switches is proportional to the number of "heavy-light" boundary crossings in the represented tree, which by a telescoping argument is O(log n) amortized. Each splay operation within `access` costs O(actual work) − O(rank change), and the total rank change across one `access` is O(log n).

Result: all operations (`link`, `cut`, `access`, `makeRoot`, `findRoot`, `pathQuery`) run in **O(log n)** amortized.

---

## Complexity Summary

| Operation | Amortized Time | Notes |
|---|---|---|
| access(x) | O(log n) | core primitive |
| makeRoot(x) | O(log n) | uses access + rev flag |
| link(u, v) | O(log n) | makeRoot + parent assignment |
| cut(u, v) | O(log n) | makeRoot + access + detach |
| findRoot(x) | O(log n) | access + leftmost walk |
| connected(u, v) | O(log n) | findRoot(u) == findRoot(v) |
| pathQuery(u, v) | O(log n) | makeRoot + access + read sum |
| pathUpdate(u, v, delta) | O(log n) | makeRoot + access + lazy tag |
| Space | O(n) | one splay node per tree node |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// LINK/CUT TREE — Splay-based dynamic tree
// Aggregate: path sum. Easily change to min/max/xor.
// ================================================================
struct Node {
    Node* ch[2];
    Node* p;
    int   val, sum;
    bool  rev;

    Node(int v = 0) : p(nullptr), val(v), sum(v), rev(false) {
        ch[0] = ch[1] = nullptr;
    }

    // Is this node the root of its splay tree?
    // (i.e. p is null or p does not claim this as a splay child)
    bool isRoot() {
        return !p || (p->ch[0] != this && p->ch[1] != this);
    }

    // Push lazy reversal flag down to children
    void push() {
        if (rev) {
            swap(ch[0], ch[1]);
            if (ch[0]) ch[0]->rev ^= 1;
            if (ch[1]) ch[1]->rev ^= 1;
            rev = false;
        }
    }

    // Recompute aggregate from children
    void pull() {
        sum = val;
        if (ch[0]) sum += ch[0]->sum;
        if (ch[1]) sum += ch[1]->sum;
    }
};

// Push from root of splay tree down to x (recursive, used before splay)
void pushAll(Node* x) {
    if (!x->isRoot()) pushAll(x->p);
    x->push();
}

// Standard splay rotation
void rotate(Node* x) {
    Node* y = x->p;
    Node* z = y->p;
    int k = (y->ch[1] == x);
    if (!y->isRoot()) z->ch[z->ch[1] == y] = x;
    x->p = z;
    y->ch[k] = x->ch[k^1];
    if (x->ch[k^1]) x->ch[k^1]->p = y;
    x->ch[k^1] = y;
    y->p = x;
    y->pull(); x->pull();
}

// Splay x to the root of its splay tree
void splay(Node* x) {
    pushAll(x); // push lazy flags from root down to x first
    while (!x->isRoot()) {
        Node* y = x->p;
        if (!y->isRoot()) {
            Node* z = y->p;
            // Zig-zig: same direction → rotate parent first
            // Zig-zag: different direction → rotate x twice
            if ((z->ch[1] == y) == (y->ch[1] == x)) rotate(y);
            else rotate(x);
        }
        rotate(x);
    }
    x->pull();
}

// ---- Core LCT primitives ----

// Expose path from root to x; return the last node before x on the access path
Node* access(Node* x) {
    Node* last = nullptr;
    for (Node* y = x; y; y = y->p) {
        splay(y);
        y->ch[1] = last; // detach preferred child below y, attach last
        y->pull();
        last = y;
    }
    splay(x);
    return last;
}

// Reroot the represented tree at x
void makeRoot(Node* x) {
    access(x);
    x->rev ^= 1; // flip path order → x becomes shallowest (root)
    x->push();
}

// Find the root of x's represented tree — O(log n) amortized
Node* findRoot(Node* x) {
    access(x);
    while (x->ch[0]) { x->push(); x = x->ch[0]; }
    splay(x); // important for amortized bound
    return x;
}

// Are u and v in the same tree?
bool connected(Node* u, Node* v) {
    return findRoot(u) == findRoot(v);
}

// Link u and v (must be in different trees)
void link(Node* u, Node* v) {
    makeRoot(u);
    if (findRoot(v) != u) // guard: prevent cycle
        u->p = v;
}

// Cut the edge between u and v (edge must exist)
void cut(Node* u, Node* v) {
    makeRoot(u);
    access(v);
    // After makeRoot(u) and access(v): u is v->ch[0] and u->ch[1] == null
    if (v->ch[0] == u && !u->ch[1]) {
        v->ch[0] = nullptr;
        u->p = nullptr;
        v->pull();
    }
}

// Path sum from u to v
int pathSum(Node* u, Node* v) {
    makeRoot(u);
    access(v);
    return v->sum;
}

// Update value of node x to val
void updateVal(Node* x, int val) {
    splay(x);
    x->val = val;
    x->pull();
}

// ================================================================
// Usage + stress test
// ================================================================
int main() {
    // ---- Demo: tree 1-2-3-4, 1-5, 2-6, 3-7 ----
    {
        printf("=== Link/Cut Tree Demo ===\n");
        const int N = 7;
        vector<Node*> nd(N+1);
        for (int i=1;i<=N;i++) nd[i] = new Node(i);

        link(nd[1],nd[2]); link(nd[2],nd[3]); link(nd[3],nd[4]);
        link(nd[1],nd[5]); link(nd[2],nd[6]); link(nd[3],nd[7]);

        // Path 4→3→2→1→5: sum = 4+3+2+1+5 = 15
        printf("pathSum(4,5) = %d  (expect 15)\n", pathSum(nd[4],nd[5]));
        // Path 6→2→3→7: sum = 6+2+3+7 = 18
        printf("pathSum(6,7) = %d  (expect 18)\n", pathSum(nd[6],nd[7]));
        printf("connected(1,4) = %d  (expect 1)\n", connected(nd[1],nd[4]));

        cut(nd[2],nd[3]);
        printf("\nAfter cut(2,3):\n");
        printf("connected(1,4) = %d  (expect 0)\n", connected(nd[1],nd[4]));
        printf("connected(1,2) = %d  (expect 1)\n", connected(nd[1],nd[2]));
        // Path 1→2→6: sum = 1+2+6 = 9
        printf("pathSum(1,6) = %d  (expect 9)\n", pathSum(nd[1],nd[6]));

        link(nd[2],nd[4]); // reconnect 2-4
        printf("\nAfter link(2,4): connected(1,4) = %d  (expect 1)\n",
               connected(nd[1],nd[4]));

        updateVal(nd[3], 30); // change value of node 3
        printf("After update(3,30): pathSum(1,7) = %d  (expect 1+2+30+7=40)\n",
               pathSum(nd[1],nd[7]));
    }

    // ---- Stress test: LCT vs DSU for connectivity ----
    {
        printf("\n=== Stress Test 3000 ops ===\n");
        srand(42);
        const int SN = 200;
        vector<Node*> nodes(SN+1);
        for (int i=1;i<=SN;i++) nodes[i] = new Node(i);

        // DSU for ground-truth connectivity
        vector<int> par(SN+1); iota(par.begin(),par.end(),0);
        function<int(int)> find=[&](int x)->int{
            return par[x]==x?x:par[x]=find(par[x]);};

        set<pair<int,int>> edges;
        int errors = 0;

        for (int op=0; op<3000; op++) {
            int type = rand()%3;
            if (type==0) { // link
                int u=rand()%SN+1, v=rand()%SN+1;
                if (u==v) continue;
                if (find(u)!=find(v)) {
                    link(nodes[u],nodes[v]);
                    par[find(u)]=find(v);
                    edges.insert({min(u,v),max(u,v)});
                }
            } else if (type==1) { // cut
                if (edges.empty()) continue;
                auto it=edges.begin();
                advance(it, rand()%edges.size());
                int u=it->first, v=it->second;
                cut(nodes[u],nodes[v]);
                edges.erase(it);
                // Rebuild DSU from scratch
                iota(par.begin(),par.end(),0);
                map<int,vector<int>> adj;
                for (auto& [a,b]:edges){adj[a].push_back(b);adj[b].push_back(a);}
                for (int i=1;i<=SN;i++){
                    queue<int>q; if(find(i)!=i) continue;
                    q.push(i);
                    while(!q.empty()){
                        int x=q.front();q.pop();
                        for (int nb:adj[x])
                            if(find(nb)!=find(x)){par[find(nb)]=find(x);q.push(nb);}
                    }
                }
            } else { // connectivity query
                int u=rand()%SN+1, v=rand()%SN+1;
                bool lct_ans = connected(nodes[u],nodes[v]);
                bool dsu_ans = (find(u)==find(v));
                if (lct_ans!=dsu_ans) errors++;
            }
        }
        printf("Result: %s\n",errors==0?"OK":("FAIL "+to_string(errors)).c_str());
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
// LINK/CUT TREE — C#
// ================================================================
public class LCTNode
{
    public LCTNode[] Ch = new LCTNode[2];
    public LCTNode P;
    public int Val, Sum;
    public bool Rev;

    public LCTNode(int val) { Val = val; Sum = val; }

    public bool IsRoot() =>
        P == null || (P.Ch[0] != this && P.Ch[1] != this);

    public void Push()
    {
        if (!Rev) return;
        (Ch[0], Ch[1]) = (Ch[1], Ch[0]);
        if (Ch[0] != null) Ch[0].Rev ^= true;
        if (Ch[1] != null) Ch[1].Rev ^= true;
        Rev = false;
    }

    public void Pull()
    {
        Sum = Val;
        if (Ch[0] != null) Sum += Ch[0].Sum;
        if (Ch[1] != null) Sum += Ch[1].Sum;
    }
}

public class LinkCutTree
{
    static void PushAll(LCTNode x)
    {
        if (!x.IsRoot()) PushAll(x.P);
        x.Push();
    }

    static void Rotate(LCTNode x)
    {
        var y = x.P; var z = y.P;
        int k = y.Ch[1] == x ? 1 : 0;
        if (!y.IsRoot()) z.Ch[z.Ch[1] == y ? 1 : 0] = x;
        x.P = z;
        y.Ch[k] = x.Ch[k^1];
        if (x.Ch[k^1] != null) x.Ch[k^1].P = y;
        x.Ch[k^1] = y; y.P = x;
        y.Pull(); x.Pull();
    }

    static void Splay(LCTNode x)
    {
        PushAll(x);
        while (!x.IsRoot())
        {
            var y = x.P;
            if (!y.IsRoot())
            {
                var z = y.P;
                bool same = (z.Ch[1]==y) == (y.Ch[1]==x);
                if (same) Rotate(y); else Rotate(x);
            }
            Rotate(x);
        }
        x.Pull();
    }

    public static LCTNode Access(LCTNode x)
    {
        LCTNode last = null;
        for (var y = x; y != null; y = y.P)
        {
            Splay(y);
            y.Ch[1] = last;
            y.Pull();
            last = y;
        }
        Splay(x);
        return last;
    }

    public static void MakeRoot(LCTNode x)
    {
        Access(x);
        x.Rev ^= true;
        x.Push();
    }

    public static LCTNode FindRoot(LCTNode x)
    {
        Access(x);
        while (x.Ch[0] != null) { x.Push(); x = x.Ch[0]; }
        Splay(x);
        return x;
    }

    public static bool Connected(LCTNode u, LCTNode v) =>
        FindRoot(u) == FindRoot(v);

    public static void Link(LCTNode u, LCTNode v)
    {
        MakeRoot(u);
        if (FindRoot(v) != u) u.P = v;
    }

    public static void Cut(LCTNode u, LCTNode v)
    {
        MakeRoot(u);
        Access(v);
        if (v.Ch[0] == u && u.Ch[1] == null)
        {
            v.Ch[0] = null;
            u.P = null;
            v.Pull();
        }
    }

    public static int PathSum(LCTNode u, LCTNode v)
    {
        MakeRoot(u);
        Access(v);
        return v.Sum;
    }

    public static void UpdateVal(LCTNode x, int val)
    {
        Splay(x);
        x.Val = val;
        x.Pull();
    }

    public static void Main()
    {
        int N = 7;
        var nd = new LCTNode[N+1];
        for (int i=1;i<=N;i++) nd[i] = new LCTNode(i);

        Link(nd[1],nd[2]); Link(nd[2],nd[3]); Link(nd[3],nd[4]);
        Link(nd[1],nd[5]); Link(nd[2],nd[6]); Link(nd[3],nd[7]);

        Console.WriteLine($"pathSum(4,5) = {PathSum(nd[4],nd[5])}  (expect 15)");
        Console.WriteLine($"pathSum(6,7) = {PathSum(nd[6],nd[7])}  (expect 18)");
        Console.WriteLine($"connected(1,4) = {Connected(nd[1],nd[4])}  (expect True)");

        Cut(nd[2],nd[3]);
        Console.WriteLine($"\nAfter cut(2,3):");
        Console.WriteLine($"connected(1,4) = {Connected(nd[1],nd[4])}  (expect False)");
        Console.WriteLine($"connected(1,2) = {Connected(nd[1],nd[2])}  (expect True)");
        Console.WriteLine($"pathSum(1,6) = {PathSum(nd[1],nd[6])}  (expect 9)");
    }
}
```

---

## The `rev` Flag — How `makeRoot` Works

`makeRoot(x)` must reverse the depth order of the root-to-x path, making `x` the new shallowest node (root). This is implemented by a lazy **reversal flag**:

```
Before makeRoot(x):   depth order in splay tree = [root, ..., x]
                      in-order left-to-right = shallow to deep

After rev ^= 1:       logical order = [x, ..., root]
                      x is now at depth 0 (leftmost = shallowest)

When push() propagates rev:
    swap(ch[0], ch[1])   ← reverses in-order traversal
    propagate to children

This works because the splay tree stores the path in depth order,
and reversing in-order = reversing depth order = flipping root.
```

---

## Pitfalls

- **`pushAll` before splay** — before splaying `x`, all lazy `rev` flags from the splay tree root down to `x` must be pushed. Without this, rotations may swap children at the wrong nodes, corrupting the tree structure permanently. `pushAll` recursively pushes from the root down.
- **`isRoot` checks parent's children, not just parent** — `isRoot` returns true if `x->p` does not have `x` as a splay child (`p->ch[0] != x && p->ch[1] != x`). This is the case for path-parent pointers (virtual edges). Checking only `p == null` misses the case where `p` is a path-parent, not a splay parent.
- **`findRoot` must splay after finding the root** — after walking to the leftmost node in `access(x)`'s splay tree, `splay(root)` must be called. Without it, the amortized O(log n) bound breaks — the leftmost walk can be O(n) in the worst case if the tree is not rebalanced.
- **`cut` requires `makeRoot` first** — the standard cut implementation works only when `u` is the splay-tree root of a singleton (no `ch[1]`). `makeRoot(u)` followed by `access(v)` arranges exactly this: `u` becomes `v->ch[0]` with no right child.
- **`link` must guard against cycles** — if `u` and `v` are already connected, `link` would create a cycle in the represented tree. Always check `findRoot(v) != u` after `makeRoot(u)` before setting `u->p = v`.
- **Aggregate must be updated after every structural change** — `pull()` must be called whenever `ch[0]` or `ch[1]` changes. Missing a `pull()` after `rotate`, `access`, or `cut` leaves stale sums and makes path queries return wrong results.
- **`access` loop uses path-parent pointer, not splay parent** — the `access` loop walks `y = y->p` where `p` is the path-parent pointer (a virtual edge). After `splay(y)`, `y` is the splay root, so `y->p` is exactly the path-parent if it exists. Using the splay-tree parent would end the loop too early.

---

## Conclusion

Link/Cut Trees are the **canonical data structure for dynamic tree path queries**:

- Each represented tree is decomposed into preferred paths stored as splay trees. The `access` operation restructures these paths in O(log n) amortized via the access lemma.
- `makeRoot` enables arbitrary-root path queries by reversing the depth order of a path using a lazy `rev` flag — the same mechanism as reversals in implicit treaps.
- All six primitives — `link`, `cut`, `access`, `makeRoot`, `findRoot`, `pathQuery` — reduce to sequences of `access` calls and splay operations, all amortized O(log n).
- The dual role of the `p` pointer (splay parent vs path-parent) is the central implementation detail: `isRoot()` distinguishes the two cases by checking whether `p` claims `x` as a splay child.

**Key takeaway:** Link/Cut Trees work by maintaining a preferred-path decomposition of the forest. `access(x)` restructures the decomposition to expose the root-to-x path, paying O(log n) amortized for each path boundary crossed. Everything else — link, cut, makeRoot, findRoot — is one or two `access` calls plus O(1) pointer manipulation.
