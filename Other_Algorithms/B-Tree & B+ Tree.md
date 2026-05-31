# B-Tree / B+ Tree

## Origin & Motivation

**B-Tree** is a self-balancing search tree introduced by Bayer and McCreight in 1970, designed for storage systems where data lives on disk and random I/O dominates cost. Unlike BSTs that minimize comparisons, B-Trees minimize **block transfers**: each node holds many keys so a single disk read fetches an entire node worth of data. The "B" stands for balanced — not binary.

**The problem it solves:** Binary search trees on disk require O(log₂ n) node reads, each a separate I/O. With n = 10⁹ and a BST node holding one key that is 30 I/Os per lookup. A B-Tree node holds hundreds of keys — the tree height collapses to O(log_t n), meaning 2–4 I/Os for the same dataset.

**B+ Tree** is the dominant database variant: internal nodes store only routing keys, all data lives in leaves, and leaves are linked in a list for O(log_t n + m) range scans.

**The key idea:** Every node (except root) stays between half-full and completely full — `t-1` to `2t-1` keys. Insertions maintain this via **split** (node full); deletions maintain it via **rotate** (borrow from sibling) or **merge** (both siblings at minimum). All leaves sit at the same depth — the tree is perfectly height-balanced at all times.

Complexity: **O(log_t n)** height, **O(t · log_t n)** comparisons per operation, **O(log_t n)** disk I/Os per operation.

---

## Where It Is Used

- Database engines: InnoDB (MySQL), PostgreSQL, SQLite — primary and secondary indexes
- File systems: NTFS, HFS+, Btrfs — directory and extent trees
- Key-value stores: LevelDB, RocksDB — sorted table structure
- Operating system virtual memory managers and page tables
- Competitive programming: order-statistic trees, offline range queries with large fanout

---

## Core Structure

### Node Layout (minimum degree t)

Each internal node holds between `t-1` and `2t-1` keys and between `t` and `2t` children. The root may hold as few as 1 key.

```
Node {
    keys[0..2t-2]       // sorted key array, currently n keys
    children[0..2t-1]   // child pointers (null for leaves)
    n                   // current key count
    is_leaf
}
```

**Invariants:**
```
t-1  ≤ node.n ≤ 2t-1       (except root: 1 ≤ root.n ≤ 2t-1)
keys[0] < keys[1] < ... < keys[n-1]
All keys in children[i] subtree lie in (keys[i-1], keys[i])
All leaves at same depth h = floor(log_t((n+1)/2))
```

### B-Tree vs B+ Tree

| Property | B-Tree | B+ Tree |
|---|---|---|
| Data stored at | Every node | Leaves only |
| Internal nodes | Keys + data pointers | Routing keys only |
| Leaf linking | No | Doubly linked list |
| Range scan | Traverse tree | Walk leaf list |
| Space per internal node | Larger | Smaller → more keys → shallower tree |
| Duplicate keys in tree | No | Internal key copied from leaf on split |

---

## Operations

### Search

```
Search(node, k):
    i = first index s.t. k ≤ node.keys[i]   // binary search within node

    if i < node.n and k == node.keys[i]:
        return (node, i)                      // found
    if node.is_leaf:
        return NOT_FOUND

    disk_read(node.children[i])               // one I/O
    return Search(node.children[i], k)
```

Cost: **O(log_t n)** disk reads, **O(t · log_t n)** comparisons.

### Insert

Pre-emptive splits on the way down ensure the parent always has room — no backtracking.

```
Insert(T, k):
    if root is full (n == 2t-1):
        s = new node
        s.children[0] = T.root
        SplitChild(s, 0)          // split root; tree height grows by 1
        T.root = s
    InsertNonFull(T.root, k)

InsertNonFull(node, k):
    i = index to insert / descend
    if node.is_leaf:
        shift keys right, insert k
    else:
        if child[i] is full: SplitChild(node, i)   // pre-emptive split
        InsertNonFull(correct child, k)

SplitChild(parent, i):
    y = parent.children[i]           // full child: 2t-1 keys
    z = new node (right half of y)
    z.keys = y.keys[t..2t-2]         // upper t-1 keys
    y.keys = y.keys[0..t-2]          // lower t-1 keys stay in y
    promote y.keys[t-1] to parent    // median moves up
    insert z into parent.children[i+1]
```

### Delete

Three cases — all handled with a single top-down pass using pre-emptive fixes:

```
Case 1: k in leaf node
    → Remove k directly.

Case 2: k in internal node x
    2a. Left child ch[i] has ≥ t keys
        → Replace k with predecessor; delete predecessor from ch[i].
    2b. Right child ch[i+1] has ≥ t keys
        → Replace k with successor; delete successor from ch[i+1].
    2c. Both children have exactly t-1 keys
        → Merge ch[i], k, ch[i+1] into ch[i]; delete k from merged node.

Case 3: k not found in internal node x, descend to ch[i]
    If ch[i] has only t-1 keys, fix first:
    3a. Adjacent sibling has ≥ t keys
        → Rotate: move key from sibling up to x, move x's key down to ch[i].
    3b. All adjacent siblings have t-1 keys
        → Merge ch[i] with a sibling and a key from x.
    Then recurse into the (possibly merged) ch[i].
```

### B+ Tree Split Distinction

When a **leaf** splits in a B+ Tree the median key is **copied** up (not moved) — it stays in the right leaf so all data remains accessible via the leaf list:

```
B-Tree internal split:
    median key MOVES to parent — gone from both children

B+ Tree leaf split:
    right.keys[0] is COPIED to parent — stays in right leaf

B+ Tree internal split:
    same as B-Tree — median key moves up (not a leaf, no data loss)
```

### Range Scan (B+ Tree only)

```
RangeScan(lo, hi):
    leaf = descend to first leaf with key ≥ lo    // O(log_t n)

    while leaf != null:
        for each (k, v) in leaf:
            if k > hi: return results
            if k ≥ lo: emit (k, v)
        leaf = leaf.next                           // O(1) per step
```

Cost: **O(log_t n + m)** — optimal sequential I/O.

---

## Height and Complexity

```
Min keys in tree of height h:   n ≥ 2t^h - 1
Therefore:                       h ≤ log_t((n+1)/2)
```

With t = 1000 (4 KB page, 8-byte keys) a tree over 10⁹ elements has height ≤ 3.

| Operation | Comparisons | Disk I/Os |
|---|---|---|
| Search | O(t · log_t n) | O(log_t n) |
| Insert | O(t · log_t n) | O(log_t n) |
| Delete | O(t · log_t n) | O(log_t n) |
| Range scan (m results) | O(t · log_t n + m) | O(log_t n + m/t) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// B-TREE — minimum degree T (each non-root node: T-1..2T-1 keys)
// Uses vector<> nodes for correctness; all CLRS cases implemented.
// ================================================================
template<int T = 3>
struct BTree {
    struct Node {
        vector<int>   keys;
        vector<Node*> ch;
        bool leaf;
        Node(bool lf = true) : leaf(lf) {}
        int n() const { return (int)keys.size(); }
    };
    Node* root;
    BTree() : root(new Node(true)) {}

    // ---- Search ----
    pair<Node*,int> search(Node* x, int k) {
        int i = (int)(lower_bound(x->keys.begin(), x->keys.end(), k) - x->keys.begin());
        if (i < x->n() && x->keys[i] == k) return {x, i};
        if (x->leaf) return {nullptr, -1};
        return search(x->ch[i], k);
    }
    bool contains(int k) { return search(root, k).first != nullptr; }

    // ---- Split child ch[i] of x (ch[i] is full: 2T-1 keys) ----
    void splitChild(Node* x, int i) {
        Node* y = x->ch[i];
        Node* z = new Node(y->leaf);
        z->keys.assign(y->keys.begin() + T, y->keys.end());
        if (!y->leaf) z->ch.assign(y->ch.begin() + T, y->ch.end());
        int med = y->keys[T - 1];
        y->keys.resize(T - 1);
        if (!y->leaf) y->ch.resize(T);
        x->keys.insert(x->keys.begin() + i, med);
        x->ch.insert(x->ch.begin() + i + 1, z);
    }

    // ---- Insert ----
    void insNonFull(Node* x, int k) {
        int i = (int)(lower_bound(x->keys.begin(), x->keys.end(), k) - x->keys.begin());
        if (i < x->n() && x->keys[i] == k) return; // skip duplicate
        if (x->leaf) { x->keys.insert(x->keys.begin() + i, k); return; }
        if (x->ch[i]->n() == 2*T-1) { splitChild(x, i); if (k > x->keys[i]) i++; }
        if (i < x->n() && x->keys[i] == k) return;
        insNonFull(x->ch[i], k);
    }
    void insert(int k) {
        if (root->n() == 2*T-1) {
            Node* s = new Node(false);
            s->ch.push_back(root); root = s;
            splitChild(s, 0);
        }
        insNonFull(root, k);
    }

    // ---- Delete helpers ----
    int predKey(Node* x, int i) { Node* c = x->ch[i];   while (!c->leaf) c = c->ch.back(); return c->keys.back(); }
    int succKey(Node* x, int i) { Node* c = x->ch[i+1]; while (!c->leaf) c = c->ch[0];     return c->keys[0]; }

    void merge(Node* x, int i) {
        Node* y = x->ch[i]; Node* z = x->ch[i+1];
        y->keys.push_back(x->keys[i]);
        y->keys.insert(y->keys.end(), z->keys.begin(), z->keys.end());
        if (!y->leaf) y->ch.insert(y->ch.end(), z->ch.begin(), z->ch.end());
        x->keys.erase(x->keys.begin() + i);
        x->ch.erase(x->ch.begin() + i + 1);
        delete z;
    }

    // Ensure ch[i] has >= T keys before descending (pre-emptive fix)
    void ensureMin(Node* x, int i) {
        if (i > 0 && x->ch[i-1]->n() >= T) {
            // rotate right: borrow from left sibling
            Node* c = x->ch[i]; Node* l = x->ch[i-1];
            c->keys.insert(c->keys.begin(), x->keys[i-1]);
            if (!c->leaf) c->ch.insert(c->ch.begin(), l->ch.back());
            x->keys[i-1] = l->keys.back();
            l->keys.pop_back();
            if (!l->leaf) l->ch.pop_back();
        } else if (i < (int)x->ch.size()-1 && x->ch[i+1]->n() >= T) {
            // rotate left: borrow from right sibling
            Node* c = x->ch[i]; Node* r = x->ch[i+1];
            c->keys.push_back(x->keys[i]);
            if (!c->leaf) c->ch.push_back(r->ch[0]);
            x->keys[i] = r->keys[0];
            r->keys.erase(r->keys.begin());
            if (!r->leaf) r->ch.erase(r->ch.begin());
        } else {
            // merge
            if (i >= (int)x->ch.size()-1) i--;
            merge(x, i);
        }
    }

    // ---- Delete (CLRS 3 cases) ----
    void del(Node* x, int k) {
        int i = (int)(lower_bound(x->keys.begin(), x->keys.end(), k) - x->keys.begin());
        if (i < x->n() && x->keys[i] == k) {
            if (x->leaf) {
                // Case 1: key in leaf
                x->keys.erase(x->keys.begin() + i);
            } else if (x->ch[i]->n() >= T) {
                // Case 2a: replace with predecessor
                int p = predKey(x, i); x->keys[i] = p; del(x->ch[i], p);
            } else if (x->ch[i+1]->n() >= T) {
                // Case 2b: replace with successor
                int s = succKey(x, i); x->keys[i] = s; del(x->ch[i+1], s);
            } else {
                // Case 2c: merge then delete
                merge(x, i); del(x->ch[i], k);
            }
        } else {
            if (x->leaf) return; // not found
            // Case 3: ensure child has >= T keys before descending
            bool isLast = (i == x->n());
            if (x->ch[i]->n() < T) ensureMin(x, i);
            if (isLast && i > x->n()) del(x->ch[i-1], k);
            else                      del(x->ch[i],   k);
        }
    }
    void remove(int k) {
        del(root, k);
        if (root->n() == 0 && !root->leaf) { Node* o = root; root = root->ch[0]; delete o; }
    }

    // ---- In-order traversal ----
    void traverse(Node* x, vector<int>& out) {
        for (int i = 0; i < x->n(); i++) {
            if (!x->leaf) traverse(x->ch[i], out);
            out.push_back(x->keys[i]);
        }
        if (!x->leaf) traverse(x->ch.back(), out);
    }
    vector<int> toSorted() { vector<int> v; traverse(root, v); return v; }
};

// ================================================================
// B+ TREE — minimum degree T, all data in leaves, linked leaf list
// Separator invariant: internal keys[i] = min(ch[i+1])
// refreshSeps() keeps all separators correct after structural changes
// ================================================================
template<int T = 3>
struct BPlusTree {
    static const int MAX = 2*T - 1;

    struct Node {
        int   keys[2*T];
        int   vals[2*T];   // leaf only
        Node* ch[2*T+1];   // internal only
        Node* next;        // leaf linked list
        int   n;
        bool  leaf;
        Node(bool lf) : n(0), leaf(lf), next(nullptr) { fill(ch, ch+2*T+1, nullptr); }
    };
    Node* root;
    BPlusTree() : root(new Node(true)) {}

    // ---- Search ----
    int* search(int k) {
        Node* c = root;
        while (!c->leaf) { int i = ub(c, k); c = c->ch[i]; }
        int i = lb(c, k);
        if (i < c->n && c->keys[i] == k) return &c->vals[i];
        return nullptr;
    }

    // ---- Insert ----
    void insert(int k, int v = 0) {
        auto [nc, pk] = insRec(root, k, v);
        if (!nc) return;
        Node* nr = new Node(false);
        nr->keys[0] = pk; nr->ch[0] = root; nr->ch[1] = nc; nr->n = 1;
        root = nr;
    }

    pair<Node*,int> splitLeaf(Node* L) {
        int m = (L->n+1)/2;
        Node* R = new Node(true); R->n = L->n - m;
        for (int i = 0; i < R->n; i++) { R->keys[i] = L->keys[m+i]; R->vals[i] = L->vals[m+i]; }
        L->n = m; R->next = L->next; L->next = R;
        return {R, R->keys[0]};  // copy-up: smallest key of right leaf
    }
    pair<Node*,int> splitInternal(Node* x) {
        int m = x->n/2; int pk = x->keys[m];
        Node* R = new Node(false); R->n = x->n - m - 1;
        for (int i = 0; i < R->n;  i++) R->keys[i] = x->keys[m+1+i];
        for (int i = 0; i <= R->n; i++) R->ch[i]   = x->ch[m+1+i];
        x->n = m; return {R, pk};
    }
    pair<Node*,int> insRec(Node* x, int k, int v) {
        if (x->leaf) {
            int i = lb(x, k);
            if (i < x->n && x->keys[i] == k) { x->vals[i] = v; return {nullptr, 0}; }
            for (int j = x->n; j > i; j--) { x->keys[j] = x->keys[j-1]; x->vals[j] = x->vals[j-1]; }
            x->keys[i] = k; x->vals[i] = v; x->n++;
            if (x->n > MAX) return splitLeaf(x);
            return {nullptr, 0};
        }
        int i = ub(x, k);
        auto [nc, pk] = insRec(x->ch[i], k, v);
        if (!nc) return {nullptr, 0};
        for (int j = x->n; j > i; j--) { x->keys[j] = x->keys[j-1]; x->ch[j+1] = x->ch[j]; }
        x->keys[i] = pk; x->ch[i+1] = nc; x->n++;
        if (x->n > MAX) return splitInternal(x);
        return {nullptr, 0};
    }

    // ---- Delete ----
    void remove(int k) {
        delRec(root, k);
        if (root->n == 0 && !root->leaf) { Node* o = root; root = root->ch[0]; delete o; }
    }

    int getMin(Node* x) { while (!x->leaf) x = x->ch[0]; return x->keys[0]; }

    // Refresh all separator keys in x from actual subtree minima.
    // separators keys[i] = min(ch[i+1]) must always hold for correct descent.
    void refreshSeps(Node* x) {
        if (!x->leaf)
            for (int i = 0; i < x->n; i++) x->keys[i] = getMin(x->ch[i+1]);
    }

    bool delRec(Node* x, int k) {
        if (x->leaf) {
            int i = lb(x, k);
            if (i >= x->n || x->keys[i] != k) return false;
            for (int j = i; j < x->n-1; j++) { x->keys[j] = x->keys[j+1]; x->vals[j] = x->vals[j+1]; }
            x->n--;
            return x->n < T-1;
        }
        int i = ub(x, k);
        bool uf = delRec(x->ch[i], k);
        refreshSeps(x); // keep separators correct after any structural change below
        if (!uf) return false;
        return fixUnderflow(x, i);
    }

    bool fixUnderflow(Node* x, int i) {
        // Borrow from left sibling
        if (i > 0 && x->ch[i-1]->n >= T) {
            Node* c = x->ch[i]; Node* l = x->ch[i-1];
            if (c->leaf) {
                for (int j = c->n; j > 0; j--) { c->keys[j] = c->keys[j-1]; c->vals[j] = c->vals[j-1]; }
                c->keys[0] = l->keys[l->n-1]; c->vals[0] = l->vals[l->n-1];
                c->n++; l->n--;
            } else {
                for (int j = c->n;   j > 0; j--) c->keys[j] = c->keys[j-1];
                for (int j = c->n+1; j > 0; j--) c->ch[j]   = c->ch[j-1];
                c->keys[0] = x->keys[i-1]; c->ch[0] = l->ch[l->n];
                c->n++; l->n--;
            }
            refreshSeps(x); return false;
        }
        // Borrow from right sibling
        if (i < x->n && x->ch[i+1]->n >= T) {
            Node* c = x->ch[i]; Node* r = x->ch[i+1];
            if (c->leaf) {
                c->keys[c->n] = r->keys[0]; c->vals[c->n] = r->vals[0]; c->n++;
                for (int j = 0; j < r->n-1; j++) { r->keys[j] = r->keys[j+1]; r->vals[j] = r->vals[j+1]; }
                r->n--;
            } else {
                c->keys[c->n] = x->keys[i]; c->ch[c->n+1] = r->ch[0]; c->n++;
                for (int j = 0; j < r->n-1; j++) r->keys[j] = r->keys[j+1];
                for (int j = 0; j < r->n;   j++) r->ch[j]   = r->ch[j+1];
                r->n--;
            }
            refreshSeps(x); return false;
        }
        // Merge
        if (i > 0) i--;
        mergeChildren(x, i);
        refreshSeps(x);
        return x->n < T-1;
    }

    void mergeChildren(Node* x, int i) {
        Node* L = x->ch[i]; Node* R = x->ch[i+1];
        if (L->leaf) {
            for (int j = 0; j < R->n; j++) { L->keys[L->n+j] = R->keys[j]; L->vals[L->n+j] = R->vals[j]; }
            L->n += R->n; L->next = R->next; delete R;
        } else {
            L->keys[L->n] = x->keys[i];
            for (int j = 0; j < R->n;  j++) L->keys[L->n+1+j] = R->keys[j];
            for (int j = 0; j <= R->n; j++) L->ch[L->n+1+j]   = R->ch[j];
            L->n += R->n + 1; delete R;
        }
        for (int j = i;   j < x->n-1; j++) x->keys[j] = x->keys[j+1];
        for (int j = i+1; j < x->n;   j++) x->ch[j]   = x->ch[j+1];
        x->n--;
    }

    // ---- Range scan: O(log_t n + m) ----
    vector<pair<int,int>> rangeScan(int lo, int hi) {
        vector<pair<int,int>> res;
        Node* c = root;
        while (!c->leaf) { int i = ub(c, lo-1); c = c->ch[i]; }
        while (c) {
            for (int i = 0; i < c->n; i++) {
                if (c->keys[i] > hi) return res;
                if (c->keys[i] >= lo) res.push_back({c->keys[i], c->vals[i]});
            }
            c = c->next;
        }
        return res;
    }

    void collectLeaves(vector<int>& out) {
        Node* c = root; while (!c->leaf) c = c->ch[0];
        while (c) { for (int i = 0; i < c->n; i++) out.push_back(c->keys[i]); c = c->next; }
    }

    static int lb(Node* x, int k) {
        int lo=0,hi=x->n; while(lo<hi){int m=(lo+hi)/2;if(x->keys[m]<k)lo=m+1;else hi=m;} return lo;
    }
    static int ub(Node* x, int k) {
        int lo=0,hi=x->n; while(lo<hi){int m=(lo+hi)/2;if(x->keys[m]<=k)lo=m+1;else hi=m;} return lo;
    }
};

// ================================================================
// Usage / verification
// ================================================================
int main() {
    // --- BTree demo ---
    {
        BTree<3> bt;
        for (int x : {10,20,5,6,12,30,7,17,3,1,8,15,25,35}) bt.insert(x);
        printf("BTree sorted: ");
        for (int x : bt.toSorted()) printf("%d ", x); printf("\n");
        bt.remove(6); bt.remove(20); bt.remove(10);
        printf("After del 6,20,10: ");
        for (int x : bt.toSorted()) printf("%d ", x); printf("\n");
        printf("contains(12)=%d  contains(20)=%d\n", bt.contains(12), bt.contains(20));
    }
    // --- BPlusTree demo ---
    {
        BPlusTree<3> bp;
        for (int x : {10,20,5,6,12,30,7,17,3,1,8,15,25,35}) bp.insert(x, x*10);
        auto r = bp.rangeScan(6, 20);
        printf("B+ Range [6,20]: "); for (auto [k,v] : r) printf("%d ", k); printf("\n");
        printf("search(12)=%d\n", *bp.search(12));
        bp.remove(12);
        printf("search(12) after remove=%s\n", bp.search(12) ? "found" : "null");
    }
    // --- Stress tests ---
    {
        srand(42); int errors = 0;
        for (int trial = 0; trial < 1000; trial++) {
            BTree<3> bt; set<int> ref;
            for (int i = 0; i < 80; i++) { int x=rand()%40; bt.insert(x); ref.insert(x); }
            auto chk = [&]{ auto g=bt.toSorted(); vector<int>e(ref.begin(),ref.end()); return g==e; };
            if (!chk()) { errors++; continue; }
            vector<int> keys(ref.begin(), ref.end());
            shuffle(keys.begin(), keys.end(), default_random_engine(trial));
            for (int k : keys) { bt.remove(k); ref.erase(k); if (!chk()){errors++;break;} }
        }
        printf("BTree  1000 trials: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }
    {
        srand(99); int errors = 0;
        for (int trial = 0; trial < 1000; trial++) {
            BPlusTree<3> bp; map<int,int> ref;
            for (int i = 0; i < 80; i++) { int x=rand()%40; bp.insert(x,x*10); ref[x]=x*10; }
            auto chk = [&]{
                vector<int> got,exp; bp.collectLeaves(got);
                for (auto&[k,v]:ref) exp.push_back(k); return got==exp;
            };
            vector<int> keys; for (auto&[k,v]:ref) keys.push_back(k);
            shuffle(keys.begin(), keys.end(), default_random_engine(trial));
            for (int k : keys) { bp.remove(k); ref.erase(k); if (!chk()){errors++;break;} }
        }
        printf("BPTree 1000 trials: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
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
// B-TREE — C#, minimum degree Deg
// ================================================================
public class BTree<TKey> where TKey : IComparable<TKey>
{
    private const int Deg = 3;

    private class Node
    {
        public List<TKey> Keys     = new();
        public List<Node> Children = new();
        public bool       Leaf;
        public int        N => Keys.Count;
        public Node(bool leaf) { Leaf = leaf; }
    }

    private Node _root = new Node(true);

    // ---- Search ----
    public bool Contains(TKey k) => Search(_root, k) != null;

    private Node? Search(Node x, TKey k)
    {
        int i = LowerBound(x.Keys, k);
        if (i < x.N && x.Keys[i].CompareTo(k) == 0) return x;
        if (x.Leaf) return null;
        return Search(x.Children[i], k);
    }

    // ---- Insert ----
    public void Insert(TKey k)
    {
        if (_root.N == 2*Deg-1)
        {
            var s = new Node(false);
            s.Children.Add(_root); _root = s;
            SplitChild(s, 0);
        }
        InsNonFull(_root, k);
    }

    private void SplitChild(Node x, int i)
    {
        var y = x.Children[i];
        var z = new Node(y.Leaf);
        z.Keys.AddRange(y.Keys.Skip(Deg));
        if (!y.Leaf) z.Children.AddRange(y.Children.Skip(Deg));
        TKey med = y.Keys[Deg-1];
        y.Keys.RemoveRange(Deg-1, y.Keys.Count-(Deg-1));
        if (!y.Leaf) y.Children.RemoveRange(Deg, y.Children.Count-Deg);
        x.Keys.Insert(i, med);
        x.Children.Insert(i+1, z);
    }

    private void InsNonFull(Node x, TKey k)
    {
        int i = LowerBound(x.Keys, k);
        if (i < x.N && x.Keys[i].CompareTo(k) == 0) return; // duplicate
        if (x.Leaf) { x.Keys.Insert(i, k); return; }
        if (x.Children[i].N == 2*Deg-1)
        {
            SplitChild(x, i);
            if (k.CompareTo(x.Keys[i]) > 0) i++;
            if (i < x.N && x.Keys[i].CompareTo(k) == 0) return;
        }
        InsNonFull(x.Children[i], k);
    }

    // ---- Delete ----
    public void Remove(TKey k)
    {
        Del(_root, k);
        if (_root.N == 0 && !_root.Leaf) _root = _root.Children[0];
    }

    private TKey PredKey(Node x, int i) { var c=x.Children[i]; while(!c.Leaf)c=c.Children[^1]; return c.Keys[^1]; }
    private TKey SuccKey(Node x, int i) { var c=x.Children[i+1]; while(!c.Leaf)c=c.Children[0]; return c.Keys[0]; }

    private void Merge(Node x, int i)
    {
        var y=x.Children[i]; var z=x.Children[i+1];
        y.Keys.Add(x.Keys[i]);
        y.Keys.AddRange(z.Keys);
        if (!y.Leaf) y.Children.AddRange(z.Children);
        x.Keys.RemoveAt(i);
        x.Children.RemoveAt(i+1);
    }

    private void EnsureMin(Node x, int i)
    {
        if (i > 0 && x.Children[i-1].N >= Deg)
        {
            var c=x.Children[i]; var l=x.Children[i-1];
            c.Keys.Insert(0, x.Keys[i-1]);
            if (!c.Leaf) c.Children.Insert(0, l.Children[^1]);
            x.Keys[i-1] = l.Keys[^1];
            l.Keys.RemoveAt(l.N-1);
            if (!l.Leaf) l.Children.RemoveAt(l.Children.Count-1);
        }
        else if (i < x.N && x.Children[i+1].N >= Deg)
        {
            var c=x.Children[i]; var r=x.Children[i+1];
            c.Keys.Add(x.Keys[i]);
            if (!c.Leaf) c.Children.Add(r.Children[0]);
            x.Keys[i] = r.Keys[0];
            r.Keys.RemoveAt(0);
            if (!r.Leaf) r.Children.RemoveAt(0);
        }
        else
        {
            if (i >= x.Children.Count-1) i--;
            Merge(x, i);
        }
    }

    private void Del(Node x, TKey k)
    {
        int i = LowerBound(x.Keys, k);
        if (i < x.N && x.Keys[i].CompareTo(k) == 0)
        {
            if (x.Leaf) { x.Keys.RemoveAt(i); return; }
            if (x.Children[i].N >= Deg)
                { TKey p=PredKey(x,i); x.Keys[i]=p; Del(x.Children[i],p); }
            else if (x.Children[i+1].N >= Deg)
                { TKey s=SuccKey(x,i); x.Keys[i]=s; Del(x.Children[i+1],s); }
            else
                { Merge(x,i); Del(x.Children[i],k); }
        }
        else
        {
            if (x.Leaf) return;
            bool isLast = (i == x.N);
            if (x.Children[i].N < Deg) EnsureMin(x, i);
            if (isLast && i > x.N) Del(x.Children[i-1], k);
            else Del(x.Children[i], k);
        }
    }

    // ---- Traversal ----
    public List<TKey> ToSorted() { var out_ = new List<TKey>(); Traverse(_root, out_); return out_; }

    private void Traverse(Node x, List<TKey> out_)
    {
        for (int i = 0; i < x.N; i++)
        {
            if (!x.Leaf) Traverse(x.Children[i], out_);
            out_.Add(x.Keys[i]);
        }
        if (!x.Leaf) Traverse(x.Children[^1], out_);
    }

    private static int LowerBound(List<TKey> list, TKey k)
    {
        int lo=0, hi=list.Count;
        while (lo<hi) { int mid=(lo+hi)/2; if (list[mid].CompareTo(k)<0) lo=mid+1; else hi=mid; }
        return lo;
    }
}

// ================================================================
// B+ TREE — C#, minimum degree Deg, linked leaves
// ================================================================
public class BPlusTree
{
    private const int Deg = 3;
    private const int Max = 2*Deg-1;

    private class Node
    {
        public int[]  Keys = new int[2*Deg+1];
        public int[]  Vals = new int[2*Deg+1];
        public Node[] Ch   = new Node[2*Deg+2];
        public Node?  Next;
        public int    N;
        public bool   Leaf;
        public Node(bool leaf) { Leaf = leaf; }
    }

    private Node _root = new Node(true);

    // ---- Search ----
    public int? Search(int k)
    {
        var c = _root;
        while (!c.Leaf) { int i = UpperBound(c, k); c = c.Ch[i]; }
        int idx = LowerBound(c, k);
        if (idx < c.N && c.Keys[idx] == k) return c.Vals[idx];
        return null;
    }

    // ---- Insert ----
    public void Insert(int k, int v = 0)
    {
        var (nc, pk) = InsRec(_root, k, v);
        if (nc == null) return;
        var nr = new Node(false);
        nr.Keys[0]=pk; nr.Ch[0]=_root; nr.Ch[1]=nc; nr.N=1;
        _root = nr;
    }

    private (Node? nc, int pk) InsRec(Node x, int k, int v)
    {
        if (x.Leaf)
        {
            int i = LowerBound(x, k);
            if (i < x.N && x.Keys[i] == k) { x.Vals[i]=v; return (null,0); }
            for (int j=x.N;j>i;j--) { x.Keys[j]=x.Keys[j-1]; x.Vals[j]=x.Vals[j-1]; }
            x.Keys[i]=k; x.Vals[i]=v; x.N++;
            return x.N > Max ? SplitLeaf(x) : (null, 0);
        }
        int ci = UpperBound(x, k);
        var (nc, pk) = InsRec(x.Ch[ci], k, v);
        if (nc == null) return (null, 0);
        for (int j=x.N;j>ci;j--) { x.Keys[j]=x.Keys[j-1]; x.Ch[j+1]=x.Ch[j]; }
        x.Keys[ci]=pk; x.Ch[ci+1]=nc; x.N++;
        return x.N > Max ? SplitInternal(x) : (null, 0);
    }

    private (Node, int) SplitLeaf(Node L)
    {
        int m=(L.N+1)/2; var R=new Node(true); R.N=L.N-m;
        for (int i=0;i<R.N;i++) { R.Keys[i]=L.Keys[m+i]; R.Vals[i]=L.Vals[m+i]; }
        L.N=m; R.Next=L.Next; L.Next=R;
        return (R, R.Keys[0]);
    }
    private (Node, int) SplitInternal(Node x)
    {
        int m=x.N/2; int pk=x.Keys[m]; var R=new Node(false); R.N=x.N-m-1;
        for (int i=0;i<R.N;i++)  R.Keys[i]=x.Keys[m+1+i];
        for (int i=0;i<=R.N;i++) R.Ch[i]  =x.Ch[m+1+i];
        x.N=m; return (R, pk);
    }

    // ---- Delete ----
    public void Remove(int k)
    {
        DelRec(_root, k);
        if (_root.N==0 && !_root.Leaf) _root = _root.Ch[0];
    }

    private int GetMin(Node x) { while (!x.Leaf) x=x.Ch[0]; return x.Keys[0]; }

    private void RefreshSeps(Node x)
    {
        if (!x.Leaf)
            for (int i=0; i<x.N; i++) x.Keys[i] = GetMin(x.Ch[i+1]);
    }

    private bool DelRec(Node x, int k)
    {
        if (x.Leaf)
        {
            int i = LowerBound(x, k);
            if (i>=x.N || x.Keys[i]!=k) return false;
            for (int j=i;j<x.N-1;j++) { x.Keys[j]=x.Keys[j+1]; x.Vals[j]=x.Vals[j+1]; }
            x.N--;
            return x.N < Deg-1;
        }
        int ci = UpperBound(x, k);
        bool uf = DelRec(x.Ch[ci], k);
        RefreshSeps(x);
        if (!uf) return false;
        return FixUnderflow(x, ci);
    }

    private bool FixUnderflow(Node x, int i)
    {
        if (i>0 && x.Ch[i-1].N>=Deg)
        {
            var c=x.Ch[i]; var l=x.Ch[i-1];
            if (c.Leaf)
            {
                for (int j=c.N;j>0;j--) { c.Keys[j]=c.Keys[j-1]; c.Vals[j]=c.Vals[j-1]; }
                c.Keys[0]=l.Keys[l.N-1]; c.Vals[0]=l.Vals[l.N-1]; c.N++; l.N--;
            }
            else
            {
                for (int j=c.N;  j>0;j--) c.Keys[j]=c.Keys[j-1];
                for (int j=c.N+1;j>0;j--) c.Ch[j] =c.Ch[j-1];
                c.Keys[0]=x.Keys[i-1]; c.Ch[0]=l.Ch[l.N]; c.N++; l.N--;
            }
            RefreshSeps(x); return false;
        }
        if (i<x.N && x.Ch[i+1].N>=Deg)
        {
            var c=x.Ch[i]; var r=x.Ch[i+1];
            if (c.Leaf)
            {
                c.Keys[c.N]=r.Keys[0]; c.Vals[c.N]=r.Vals[0]; c.N++;
                for (int j=0;j<r.N-1;j++) { r.Keys[j]=r.Keys[j+1]; r.Vals[j]=r.Vals[j+1]; }
                r.N--;
            }
            else
            {
                c.Keys[c.N]=x.Keys[i]; c.Ch[c.N+1]=r.Ch[0]; c.N++;
                for (int j=0;j<r.N-1;j++) r.Keys[j]=r.Keys[j+1];
                for (int j=0;j<r.N;  j++) r.Ch[j]  =r.Ch[j+1];
                r.N--;
            }
            RefreshSeps(x); return false;
        }
        if (i>0) i--;
        MergeChildren(x, i);
        RefreshSeps(x);
        return x.N < Deg-1;
    }

    private void MergeChildren(Node x, int i)
    {
        var L=x.Ch[i]; var R=x.Ch[i+1];
        if (L.Leaf)
        {
            for (int j=0;j<R.N;j++) { L.Keys[L.N+j]=R.Keys[j]; L.Vals[L.N+j]=R.Vals[j]; }
            L.N+=R.N; L.Next=R.Next;
        }
        else
        {
            L.Keys[L.N]=x.Keys[i];
            for (int j=0;j<R.N;  j++) L.Keys[L.N+1+j]=R.Keys[j];
            for (int j=0;j<=R.N; j++) L.Ch[L.N+1+j] =R.Ch[j];
            L.N+=R.N+1;
        }
        for (int j=i;  j<x.N-1;j++) x.Keys[j]=x.Keys[j+1];
        for (int j=i+1;j<x.N;  j++) x.Ch[j]  =x.Ch[j+1];
        x.N--;
    }

    // ---- Range scan ----
    public List<(int key, int val)> RangeScan(int lo, int hi)
    {
        var res = new List<(int,int)>();
        var c = _root;
        while (!c.Leaf) { int i=UpperBound(c,lo-1); c=c.Ch[i]; }
        while (c != null)
        {
            for (int i=0;i<c.N;i++)
            {
                if (c.Keys[i] > hi) return res;
                if (c.Keys[i] >= lo) res.Add((c.Keys[i], c.Vals[i]));
            }
            c = c.Next!;
        }
        return res;
    }

    private static int LowerBound(Node x, int k)
    { int lo=0,hi=x.N; while(lo<hi){int m=(lo+hi)/2;if(x.Keys[m]<k)lo=m+1;else hi=m;} return lo; }
    private static int UpperBound(Node x, int k)
    { int lo=0,hi=x.N; while(lo<hi){int m=(lo+hi)/2;if(x.Keys[m]<=k)lo=m+1;else hi=m;} return lo; }

    public static void Main()
    {
        // BTree demo
        var bt = new BTree<int>();
        foreach (int x in new[]{10,20,5,6,12,30,7,17,3,1,8,15,25,35})
            bt.Insert(x);
        Console.WriteLine("BTree: " + string.Join(" ", bt.ToSorted()));
        bt.Remove(6); bt.Remove(20); bt.Remove(10);
        Console.WriteLine("After del 6,20,10: " + string.Join(" ", bt.ToSorted()));
        Console.WriteLine("Contains 12: " + bt.Contains(12) + "  Contains 20: " + bt.Contains(20));

        // B+Tree demo
        var bp = new BPlusTree();
        foreach (int x in new[]{10,20,5,6,12,30,7,17,3,1,8,15,25,35})
            bp.Insert(x, x*10);
        var range = bp.RangeScan(6, 20);
        Console.WriteLine("B+ Range [6,20]: " + string.Join(" ", range.Select(r => r.key)));
        Console.WriteLine("Search 12: " + bp.Search(12));
        bp.Remove(12);
        Console.WriteLine("Search 12 after remove: " + bp.Search(12));
    }
}
```

---

## refreshSeps — Why It Is Required

The B+ Tree separator invariant is `keys[i] = min(ch[i+1])`. Standard partial update (`keys[i-1] = getMin(ch[i])`) breaks when deletion cascades through `ch[0]` — that path has no separator index in the current node, so the parent's separator for the entire subtree goes stale.

`refreshSeps` recomputes all separators in a node from actual subtree minima after every structural change (delete, borrow, merge). The cost is O(t) per level — absorbed into the existing O(t · log_t n) per operation — and eliminates the entire class of stale-separator bugs:

```
Before refreshSeps, stale separator example:
    Int[12  17  21  26]          ← keys[0]=12 should be 17 (actual min of ch[1])
      Leaf[3 6 8]                ← ch[0]
      Leaf[17 19]                ← ch[1]: actual min = 17, separator wrong

upper_bound([12,17,21,26], 12) = 2   ← goes to ch[2] = Leaf[21 22 23]
key 12 not found there → deletion silently no-ops → 12 remains in ch[1]

After refreshSeps on every structural change:
    keys[0] = getMin(ch[1]) = 17   ← always correct
upper_bound([17,21,26], 12) = 0     ← correctly goes to ch[0]
```

---

## Pitfalls

- **B-Tree vs B+ Tree split semantics** — in a B-Tree the median key moves to the parent and is gone from both children. In a B+ Tree the smallest key of the right leaf is copied to the parent and stays in the leaf. Mixing these corrupts the leaf linked list.
- **Stale separators in B+ Tree** — the partial update `if(i>0) keys[i-1]=getMin(ch[i])` misses the case where deletion descends through `ch[0]` (i=0), leaving the parent's separator for this node unchanged. Use `refreshSeps` after every borrow, merge, or recursive delete.
- **Pre-emptive fix direction on delete** — `ensureMin` must check the left sibling first (borrow right rotation), then right sibling (borrow left rotation), then merge. Checking right before left is not wrong but produces a different valid tree; consistency matters for testing.
- **Merge reduces parent size — shrink root** — after a merge the parent loses one key. If the root reaches `n=0` and has one child, the tree shrinks in height: replace root with its sole child. Missing this leaves an empty root permanently.
- **Post-merge index shift** — after `ensureMin` merges `ch[i]` with a sibling, the child that absorbed the merge is at a shifted index. The `isLast` flag check (`if isLast && i > x.n`) handles this correctly; removing it causes wrong-child descent.
- **Duplicate insert** — standard B-Tree stores unique keys. After a split in `insNonFull`, the promoted median may equal `k`; guard with a second equality check before recursing, or the key will be inserted twice at different levels.
- **B+ Tree leaf link on split** — after splitting leaf L into L and R: `R.next = L.next; L.next = R`. Missing the first assignment breaks the tail of the linked list; missing the second breaks the chain entirely, making range scan return incomplete results.
- **C# `BinarySearch` returns bitwise complement on miss** — `List<T>.BinarySearch` returns `~insertionPoint` when the key is absent. The predecessor index is `~result - 1`, not `~result`. Use explicit `LowerBound` / `UpperBound` helpers to avoid this.

---

## Complexity Summary

| Structure | Search | Insert | Delete | Range (m results) | Space |
|---|---|---|---|---|---|
| BST (balanced) | O(log n) | O(log n) | O(log n) | O(log n + m) | O(n) |
| B-Tree (degree t) | O(t log_t n) | O(t log_t n) | O(t log_t n) | O(t log_t n + m) | O(n) |
| B+ Tree (degree t) | O(t log_t n) | O(t log_t n) | O(t log_t n) | O(log_t n + m) | O(n) |

With t = 100–1000: `log_t(10⁹) ≤ 3` — effectively constant I/O depth.

---

## Conclusion

B-Trees and B+ Trees are the **canonical data structures for ordered data on disk**:

- Every node holds `t-1` to `2t-1` keys, keeping height at O(log_t n) — with large t this is 2–4 levels for billion-record datasets.
- Insertions use pre-emptive splits on the way down; deletions use pre-emptive borrow/merge on the way down. Both are single-pass top-down — no backtracking.
- B+ Trees route all data to leaves and link them for O(log_t n + m) range scans — this is why every RDBMS and file system uses B+ Trees rather than plain B-Trees.
- The separator invariant in B+ Tree (`keys[i] = min(ch[i+1])`) must be maintained after every structural change at every level. Partial incremental updates miss the i=0 case; a full `refreshSeps` pass per node is the reliable solution.

**Key takeaway:** the split invariant is the clearest distinction between the two variants. B-Tree moves the median up (the key leaves both children); B+ Tree copies the right leaf's minimum up (the key stays in the leaf, keeping all data accessible via the linked list). Everything else — insert, delete, rotate, merge — follows the same logic in both structures.
