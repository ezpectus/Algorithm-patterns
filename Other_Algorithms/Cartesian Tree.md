# Cartesian Tree

## Origin & Motivation

**Cartesian Tree** is a binary tree derived from a sequence, satisfying simultaneously two classic properties: the **BST property on indices** (for every node, all left subtree indices are smaller and all right subtree indices are larger) and the **heap property on values** (every node's value is less than or equal to its children's values — min-heap variant). The name comes from interpreting the sequence as a set of 2D points `(index, value)` — the Cartesian Tree is the shape naturally imposed by these two ordering constraints at once.

**The problem it solves:** Given an array `a[0..n-1]`, the Cartesian Tree encodes the full RMQ (Range Minimum Query) structure: the root is the global minimum, the left subtree is the Cartesian Tree of the left subarray, and the right subtree is the Cartesian Tree of the right subarray. This makes `LCA(i, j)` on the Cartesian Tree exactly equal to `RMQ(i, j)` on the array — reducing RMQ to LCA in O(n) time and enabling O(1) queries after an O(n) Euler-tour + sparse-table preprocessing.

**Treap** (Tree + Heap) is the randomised Cartesian Tree: keys are user-supplied (BST property), priorities are random (heap property). The expected height is O(log n), and since the structure matches a Cartesian Tree of the random priorities, all operations inherit O(log n) expected time with a clean split/merge API.

Complexity (Treap): **O(n)** build, **O(log n)** expected insert/erase/kth/split/merge. Complexity (RMQ): **O(n)** build, **O(1)** query.

---

## Where It Is Used

- Range Minimum / Maximum Query preprocessing (LCA on Cartesian Tree)
- Treap as a self-balancing BST with split/merge: order-statistic tree, interval tree, persistent sequences
- Suffix arrays: LCP array's Cartesian Tree gives the "LCP interval tree" for suffix queries
- Competitive programming: implicit Treap for sequence operations (insert, delete, reverse, rotate)
- Expression parsing: operator-precedence parsing produces a Cartesian Tree (values = precedence)
- Histogram problems: the "largest rectangle in histogram" uses a Cartesian Tree implicitly

---

## Core Structure

### Two Properties Simultaneously

```
For every node v with index i and value a[i]:
  BST:  all nodes in left(v)  have index < i
        all nodes in right(v) have index > i
  Heap: a[i] <= a[left(v)]   (min-heap)
        a[i] <= a[right(v)]
```

Given a sequence with distinct values the Cartesian Tree is unique. With duplicate values the standard convention is to break ties by index (leftmost minimum becomes root).

### Connection to RMQ

```
RMQ(l, r) = index of minimum in a[l..r]
           = root of Cartesian Tree of a[l..r]
           = LCA(l, r) on the full Cartesian Tree
```

This equivalence holds because the root of any subtree spanning indices `[l..r]` is the minimum of that range.

---

## Operations

### Build — O(n) Stack Algorithm

Walk left to right maintaining a stack of the current right spine (path from root to rightmost node). For each new element `a[i]`:

```
Build(a[0..n-1]):
    stack = empty   // right-spine stack
    for i = 0 to n-1:
        last = -1
        while stack not empty and a[stack.top()] > a[i]:
            last = stack.pop()   // these become left subtree of i
        
        if last != -1:
            node[i].left = last   // last popped node is left child
        if stack not empty:
            node[stack.top()].right = i  // i becomes right child of new top
        
        stack.push(i)
    root = bottom of stack
```

Each node is pushed and popped exactly once → O(n) total.

### Treap: Split and Merge

Split and Merge are the two fundamental Treap operations — all others reduce to them.

```
Split(v, key):
    // Returns (L, R): L has keys < key, R has keys >= key
    if v == null: return (null, null)
    if v.key < key:
        (L, R) = Split(v.right, key)
        v.right = L
        return (v, R)
    else:
        (L, R) = Split(v.left, key)
        v.left = R
        return (L, v)

Merge(L, R):
    // L keys < R keys; heap property maintained by priority
    if L == null: return R
    if R == null: return L
    if L.pri > R.pri:
        L.right = Merge(L.right, R)
        return L
    else:
        R.left = Merge(L, R.left)
        return R
```

Both run in O(height) = O(log n) expected.

### Treap: Insert, Erase, Kth

```
Insert(key):
    (L, R) = Split(root, key)
    node = new_node(key, random_priority)
    root = Merge(Merge(L, node), R)

Erase(key):
    // remove exactly one occurrence of key
    (L, MR) = Split(root, key)
    (M, R)  = Split(MR, key+1)
    M = Merge(M.left, M.right)  // remove root of M
    root = Merge(Merge(L, M), R)

Kth(v, k):   // 1-indexed k-th smallest
    ls = size(v.left)
    if k <= ls:     return Kth(v.left,  k)
    if k == ls+1:   return v.key
    return Kth(v.right, k - ls - 1)
```

### RMQ via LCA on Cartesian Tree

After building the Cartesian Tree in O(n):

```
Euler Tour DFS → sequence of 2n-1 node visits
first[v] = first occurrence of v in Euler tour
depth[v] = depth of v in tree

RMQ(l, r):
    // LCA of nodes l and r
    pl = first[l], pr = first[r]
    if pl > pr: swap(pl, pr)
    lca = argmin_depth(euler[pl..pr])  // sparse table query O(1)
    return lca   // index of min in a[l..r]
```

Preprocessing: O(n) Euler tour + O(n log n) sparse table. Query: O(1).

---

## Complexity Summary

| Operation | Cartesian Build | Treap | RMQ Build | RMQ Query |
|---|---|---|---|---|
| Time | O(n) | O(log n) expected | O(n log n) | O(1) |
| Space | O(n) | O(n) | O(n log n) | — |

For Treap all bounds are expected (with high probability O(log n) since random priorities give balanced height).

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// CARTESIAN TREE
// BST on indices, min-heap on values
// O(n) build using right-spine stack
// ================================================================
struct CTNode {
    int key, val;
    int left = -1, right = -1, parent = -1;
};

vector<CTNode> build_cartesian(const vector<int>& a) {
    int n = (int)a.size();
    vector<CTNode> T(n);
    for (int i = 0; i < n; i++) { T[i].key = i; T[i].val = a[i]; }
    stack<int> st;
    for (int i = 0; i < n; i++) {
        int last = -1;
        while (!st.empty() && T[st.top()].val > T[i].val) {
            last = st.top(); st.pop();
        }
        if (last != -1) { T[i].left = last; T[last].parent = i; }
        if (!st.empty()) { T[st.top()].right = i; T[i].parent = st.top(); }
        st.push(i);
    }
    return T;
}

int cartesian_root(const vector<CTNode>& T) {
    for (int i = 0; i < (int)T.size(); i++)
        if (T[i].parent == -1) return i;
    return -1;
}

// ================================================================
// TREAP — randomised Cartesian tree
// Split / Merge / Insert / Erase / Kth / Count_less
// ================================================================
mt19937 rng(42);

struct Treap {
    struct Node {
        int key, pri, sz;
        int L = -1, R = -1;
    };
    vector<Node> pool;
    int root = -1;

    int new_node(int key) {
        pool.push_back({key, (int)rng(), 1});
        return (int)pool.size()-1;
    }
    int sz(int v) { return v==-1?0:pool[v].sz; }
    void pull(int v) { if(v!=-1) pool[v].sz=1+sz(pool[v].L)+sz(pool[v].R); }

    // split: left tree keys < key, right tree keys >= key
    pair<int,int> split(int v, int key) {
        if (v==-1) return {-1,-1};
        if (pool[v].key < key) {
            auto[l,r]=split(pool[v].R, key);
            pool[v].R=l; pull(v); return {v,r};
        } else {
            auto[l,r]=split(pool[v].L, key);
            pool[v].L=r; pull(v); return {l,v};
        }
    }

    int merge(int l, int r) {
        if (l==-1) return r; if (r==-1) return l;
        if (pool[l].pri>pool[r].pri) { pool[l].R=merge(pool[l].R,r); pull(l); return l; }
        else                          { pool[r].L=merge(l,pool[r].L); pull(r); return r; }
    }

    void insert(int key) {
        auto[l,r]=split(root, key);
        root=merge(merge(l, new_node(key)), r);
    }

    // erase exactly one occurrence of key (only call if key is present)
    void erase(int key) {
        auto[l,mr]=split(root, key);
        auto[m,r] =split(mr, key+1);
        if (m!=-1) m=merge(pool[m].L, pool[m].R); // remove root of m
        root=merge(merge(l,m),r);
    }

    bool contains(int key) {
        auto[l,mr]=split(root, key);
        auto[m,r] =split(mr, key+1);
        bool found=(m!=-1);
        root=merge(merge(l,m),r);
        return found;
    }

    // k-th smallest, 1-indexed
    int kth(int v, int k) {
        int ls=sz(pool[v].L);
        if (k<=ls)   return kth(pool[v].L, k);
        if (k==ls+1) return pool[v].key;
        return kth(pool[v].R, k-ls-1);
    }
    int kth(int k) { return kth(root, k); }

    // number of elements strictly less than key
    int count_less(int key) {
        auto[l,r]=split(root, key);
        int cnt=sz(l);
        root=merge(l,r);
        return cnt;
    }

    int size() { return sz(root); }

    void collect(int v, vector<int>& out) {
        if (v==-1) return;
        collect(pool[v].L, out);
        out.push_back(pool[v].key);
        collect(pool[v].R, out);
    }
    vector<int> toSorted() { vector<int> v; collect(root,v); return v; }
};

// ================================================================
// RMQ via Cartesian Tree LCA — O(n) build, O(1) query
// LCA(i,j) on the Cartesian Tree = index of minimum in a[i..j]
// Euler tour length = 2n-1; sparse table over it gives O(1) LCA.
// ================================================================
struct CartesianRMQ {
    int n;
    vector<int> val, euler, first, dep;
    vector<vector<int>> sparse;

    void build(const vector<int>& a) {
        n=(int)a.size(); val=a;
        auto T=build_cartesian(a);
        int root=cartesian_root(T);

        // Euler tour: each node appears when first visited and after each child
        first.resize(n); dep.resize(n,0);
        function<void(int,int)> dfs=[&](int v,int d){
            first[v]=(int)euler.size();
            euler.push_back(v); dep[v]=d;
            if (T[v].left !=-1) { dfs(T[v].left, d+1);  euler.push_back(v); }
            if (T[v].right!=-1) { dfs(T[v].right,d+1);  euler.push_back(v); }
        };
        dfs(root,0);

        // Sparse table over Euler tour by depth (stores index into euler[])
        int m=(int)euler.size();
        int LOG=1; while((1<<LOG)<=m) LOG++;
        sparse.assign(LOG, vector<int>(m));
        for (int i=0;i<m;i++) sparse[0][i]=i;
        for (int j=1;j<LOG;j++)
            for (int i=0;i+(1<<j)<=m;i++){
                int a=sparse[j-1][i], b=sparse[j-1][i+(1<<(j-1))];
                sparse[j][i]=dep[euler[a]]<=dep[euler[b]]?a:b;
            }
    }

    // Returns index in a[] of minimum in a[l..r]
    int query_idx(int l, int r) {
        int pl=first[l], pr=first[r];
        if (pl>pr) swap(pl,pr);
        int k=__lg(pr-pl+1);
        int a=sparse[k][pl], b=sparse[k][pr-(1<<k)+1];
        return dep[euler[a]]<=dep[euler[b]]?euler[a]:euler[b];
    }

    int query_min(int l, int r) { return val[query_idx(l,r)]; }
};

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- Cartesian Tree demo ----
    {
        printf("=== Cartesian Tree Build ===\n");
        vector<int> a={3,1,4,1,5,9,2,6,5};
        auto T=build_cartesian(a);
        int root=cartesian_root(T);
        printf("Array: 3 1 4 1 5 9 2 6 5\n");
        printf("Root: idx=%d val=%d\n", root, a[root]);
        function<void(int)> inorder=[&](int v){
            if(v==-1)return;
            inorder(T[v].left);
            printf("  [idx=%d val=%d]\n",T[v].key,T[v].val);
            inorder(T[v].right);
        };
        inorder(root);
    }

    // ---- Treap demo ----
    {
        printf("\n=== Treap ===\n");
        Treap tr;
        for (int x:{5,3,8,1,4,7,9,2,6}) tr.insert(x);
        printf("Sorted: "); for(int x:tr.toSorted()) printf("%d ",x); printf("\n");
        printf("size=%d  kth(1)=%d  kth(5)=%d  kth(9)=%d\n",
               tr.size(),tr.kth(1),tr.kth(5),tr.kth(9));
        printf("count_less(5)=%d\n",tr.count_less(5));
        tr.erase(5); tr.erase(3);
        printf("After erase 5,3: "); for(int x:tr.toSorted()) printf("%d ",x); printf("\n");
    }

    // ---- RMQ demo ----
    {
        printf("\n=== Cartesian Tree RMQ ===\n");
        vector<int> a={3,1,4,1,5,9,2,6,5,3};
        CartesianRMQ rmq; rmq.build(a);
        printf("Array: 3 1 4 1 5 9 2 6 5 3\n");
        printf("RMQ[0,9]=%d  RMQ[2,7]=%d  RMQ[5,9]=%d\n",
               rmq.query_min(0,9),rmq.query_min(2,7),rmq.query_min(5,9));
    }

    // ---- Treap stress ----
    {
        srand(123); int errors=0;
        for (int trial=0;trial<500;trial++){
            Treap tr; multiset<int> ref;
            for (int i=0;i<100;i++){
                int op=rand()%3,v=rand()%50;
                if(op==0){tr.insert(v);ref.insert(v);}
                if(op==1&&ref.count(v)){tr.erase(v);ref.erase(ref.find(v));}
                if(op==2&&!ref.empty()){
                    int k=rand()%(int)ref.size()+1;
                    auto it=ref.begin();advance(it,k-1);
                    if(tr.kth(k)!=*it) errors++;
                }
            }
            auto got=tr.toSorted();vector<int>exp(ref.begin(),ref.end());
            if(got!=exp) errors++;
        }
        printf("\nTreap stress 500 trials: %s\n",errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- RMQ stress ----
    {
        srand(42); int errors=0;
        for(int trial=0;trial<300;trial++){
            int n=50+rand()%50; vector<int> a(n);
            for(int& x:a) x=rand()%100;
            CartesianRMQ rmq; rmq.build(a);
            for(int q=0;q<50;q++){
                int l=rand()%n,r=rand()%n; if(l>r)swap(l,r);
                int exp=*min_element(a.begin()+l,a.begin()+r+1);
                if(rmq.query_min(l,r)!=exp) errors++;
            }
        }
        printf("RMQ   stress 300x50:  %s\n",errors==0?"OK":("FAIL "+to_string(errors)).c_str());
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
// CARTESIAN TREE — C# implementation
// ================================================================
public static class CartesianTree
{
    public struct CTNode
    {
        public int Key, Val, Left, Right, Parent;
    }

    // O(n) stack build — min-heap on values, BST on indices
    public static CTNode[] Build(int[] a)
    {
        int n = a.Length;
        var T = new CTNode[n];
        for (int i = 0; i < n; i++)
            T[i] = new CTNode { Key=i, Val=a[i], Left=-1, Right=-1, Parent=-1 };

        var st = new Stack<int>();
        for (int i = 0; i < n; i++)
        {
            int last = -1;
            while (st.Count > 0 && T[st.Peek()].Val > T[i].Val)
            {
                last = st.Pop();
            }
            if (last != -1) { T[i].Left = last; T[last].Parent = i; }
            if (st.Count > 0) { T[st.Peek()].Right = i; T[i].Parent = st.Peek(); }
            st.Push(i);
        }
        return T;
    }

    public static int FindRoot(CTNode[] T)
    {
        for (int i = 0; i < T.Length; i++)
            if (T[i].Parent == -1) return i;
        return -1;
    }
}

// ================================================================
// TREAP — C#
// ================================================================
public class Treap
{
    private struct Node
    {
        public int Key, Pri, Sz, L, R;
    }

    private List<Node> _pool = new();
    private int _root = -1;
    private Random _rng = new(42);

    private int NewNode(int key)
    {
        _pool.Add(new Node { Key=key, Pri=_rng.Next(), Sz=1, L=-1, R=-1 });
        return _pool.Count - 1;
    }

    private int Sz(int v) => v == -1 ? 0 : _pool[v].Sz;

    private void Pull(int v)
    {
        if (v == -1) return;
        var n = _pool[v]; n.Sz = 1 + Sz(n.L) + Sz(n.R); _pool[v] = n;
    }

    // split: left has keys < key, right has keys >= key
    private (int l, int r) Split(int v, int key)
    {
        if (v == -1) return (-1, -1);
        var nd = _pool[v];
        if (nd.Key < key)
        {
            var (l, r) = Split(nd.R, key);
            nd.R = l; _pool[v] = nd; Pull(v);
            return (v, r);
        }
        else
        {
            var (l, r) = Split(nd.L, key);
            nd.L = r; _pool[v] = nd; Pull(v);
            return (l, v);
        }
    }

    private int Merge(int l, int r)
    {
        if (l == -1) return r;
        if (r == -1) return l;
        if (_pool[l].Pri > _pool[r].Pri)
        {
            var nd = _pool[l]; nd.R = Merge(nd.R, r); _pool[l] = nd; Pull(l); return l;
        }
        else
        {
            var nd = _pool[r]; nd.L = Merge(l, nd.L); _pool[r] = nd; Pull(r); return r;
        }
    }

    public void Insert(int key)
    {
        var (l, r) = Split(_root, key);
        _root = Merge(Merge(l, NewNode(key)), r);
    }

    // Erase one occurrence of key (only call if key is present)
    public void Erase(int key)
    {
        var (l, mr) = Split(_root, key);
        var (m, r)  = Split(mr, key + 1);
        if (m != -1) m = Merge(_pool[m].L, _pool[m].R);
        _root = Merge(Merge(l, m), r);
    }

    public bool Contains(int key)
    {
        var (l, mr) = Split(_root, key);
        var (m, r)  = Split(mr, key + 1);
        bool found  = m != -1;
        _root = Merge(Merge(l, m), r);
        return found;
    }

    // k-th smallest, 1-indexed
    public int Kth(int k) => KthImpl(_root, k);
    private int KthImpl(int v, int k)
    {
        int ls = Sz(_pool[v].L);
        if (k <= ls)     return KthImpl(_pool[v].L, k);
        if (k == ls + 1) return _pool[v].Key;
        return KthImpl(_pool[v].R, k - ls - 1);
    }

    // number of elements strictly less than key
    public int CountLess(int key)
    {
        var (l, r) = Split(_root, key);
        int cnt = Sz(l);
        _root = Merge(l, r);
        return cnt;
    }

    public int Size => Sz(_root);

    public List<int> ToSorted()
    {
        var out_ = new List<int>();
        void Collect(int v){ if(v==-1)return; Collect(_pool[v].L); out_.Add(_pool[v].Key); Collect(_pool[v].R); }
        Collect(_root);
        return out_;
    }
}

// ================================================================
// RMQ via Cartesian Tree LCA — C#
// ================================================================
public class CartesianRMQ
{
    private int[] _val, _euler, _first, _dep;
    private int[][] _sparse;
    private int _n;

    public void Build(int[] a)
    {
        _n = a.Length; _val = a;
        var T = CartesianTree.Build(a);
        int root = CartesianTree.FindRoot(T);

        var euler = new List<int>();
        _first = new int[_n]; _dep = new int[_n];

        void Dfs(int v, int d)
        {
            _first[v] = euler.Count; euler.Add(v); _dep[v] = d;
            if (T[v].Left  != -1) { Dfs(T[v].Left,  d+1); euler.Add(v); }
            if (T[v].Right != -1) { Dfs(T[v].Right, d+1); euler.Add(v); }
        }
        Dfs(root, 0);
        _euler = euler.ToArray();

        int m = _euler.Length;
        int LOG = 1; while ((1<<LOG) <= m) LOG++;
        _sparse = new int[LOG][];
        for (int j = 0; j < LOG; j++) _sparse[j] = new int[m];
        for (int i = 0; i < m; i++) _sparse[0][i] = i;
        for (int j = 1; j < LOG; j++)
            for (int i = 0; i+(1<<j) <= m; i++)
            {
                int aa=_sparse[j-1][i], bb=_sparse[j-1][i+(1<<(j-1))];
                _sparse[j][i] = _dep[_euler[aa]] <= _dep[_euler[bb]] ? aa : bb;
            }
    }

    public int QueryIdx(int l, int r)
    {
        int pl=_first[l], pr=_first[r];
        if (pl>pr){ int t=pl; pl=pr; pr=t; }
        int k=0; while((1<<(k+1))<=pr-pl+1) k++;
        int a=_sparse[k][pl], b=_sparse[k][pr-(1<<k)+1];
        return _dep[_euler[a]] <= _dep[_euler[b]] ? _euler[a] : _euler[b];
    }

    public int QueryMin(int l, int r) => _val[QueryIdx(l, r)];

    public static void Main()
    {
        // Cartesian Tree demo
        int[] a = {3,1,4,1,5,9,2,6,5};
        var T = CartesianTree.Build(a);
        int root = CartesianTree.FindRoot(T);
        Console.WriteLine($"Root: idx={root} val={a[root]}");

        // Treap demo
        var tr = new Treap();
        foreach (int x in new[]{5,3,8,1,4,7,9,2,6}) tr.Insert(x);
        Console.WriteLine("Sorted: "+string.Join(" ",tr.ToSorted()));
        Console.WriteLine($"kth(1)={tr.Kth(1)} kth(5)={tr.Kth(5)} kth(9)={tr.Kth(9)}");
        Console.WriteLine($"count_less(5)={tr.CountLess(5)}");
        tr.Erase(5); tr.Erase(3);
        Console.WriteLine("After erase 5,3: "+string.Join(" ",tr.ToSorted()));

        // RMQ demo
        var rmq = new CartesianRMQ();
        rmq.Build(new[]{3,1,4,1,5,9,2,6,5,3});
        Console.WriteLine($"RMQ[0,9]={rmq.QueryMin(0,9)} RMQ[2,7]={rmq.QueryMin(2,7)} RMQ[5,9]={rmq.QueryMin(5,9)}");
    }
}
```

---

## O(n) Build — Why the Stack Works

The algorithm maintains the **right spine** of the tree built so far — the path from the root to the rightmost node. This is the only part of the tree that can be affected by inserting the next element, since all new elements arrive from the right.

```
Before inserting a[i]:
    right spine: r_0, r_1, ..., r_k   (values non-decreasing, indices increasing)

a[i] must become the right child of the last spine node with val ≤ a[i].
All spine nodes with val > a[i] get bumped off: the last one becomes the left child
of a[i] (preserving BST order since its index < i).

Each node enters the stack once and exits at most once → O(n) total.
```

---

## Pitfalls

- **Treap erase when key absent** — `erase(key)` splits out a subtree `M` of nodes equal to `key` and removes its root. If the key is absent `M` is empty and `merge(M.left, M.right)` on `-1` produces `-1` — harmless, but the caller must not assume a deletion occurred. Always guard with `contains` or a multiset count before calling `erase`.
- **Treap split semantics** — the split `(L, R)` by `key` puts keys `< key` in `L` and keys `>= key` in `R`. To isolate exactly the nodes equal to `key`, two splits are needed: `split(key)` then `split(key+1)`. Using a single split and taking the root of `R` deletes the wrong node when there are duplicates with smaller priority.
- **Size field must be updated bottom-up** — `pull(v)` must be called after every structural change to `v`'s children (in both `split` and `merge`). Forgetting `pull` after a recursive call leaves `sz` stale, making `kth` and `count_less` return wrong results.
- **RMQ Euler tour length is 2n-1** — the Euler tour visits each internal node once per child (plus the first visit), for a total of `2n-1` entries, not `n`. Allocating only `n` entries for the tour causes out-of-bounds writes.
- **Sparse table query with wrong range** — the two entries compared are `sparse[k][pl]` and `sparse[k][pr-(1<<k)+1]`, where `k = floor(log2(pr-pl+1))`. Using `pr-(1<<k)` (off by one) or `pr` directly produces wrong LCA results for certain range lengths.
- **Duplicate values break uniqueness** — when `a[i] == a[j]` for `i < j`, the standard min-heap property allows either as the root. The stack build resolves this by keeping the leftmost minimum at the root (the right spine discards nodes with `>` not `>=`), giving a well-defined tree. Changing `>` to `>=` in the stack condition inverts this and produces a different valid tree; be consistent.
- **Cartesian Tree ≠ BST on values** — the heap property on values does not mean values increase left-to-right. The left subtree of node `v` contains indices `< v.key` with arbitrary values (all ≥ `v.val`), and similarly for the right. Attempting binary search on values in the Cartesian Tree gives wrong results; only BST search on indices is valid.

---

## Complexity Summary

| Structure | Build | Insert | Erase | Kth | RMQ Query | Space |
|---|---|---|---|---|---|---|
| Cartesian Tree (static) | O(n) | — | — | — | O(1) after O(n log n) prep | O(n log n) |
| Treap | O(n log n) | O(log n) | O(log n) | O(log n) | — | O(n) |
| Treap + split/merge | O(n log n) | O(log n) | O(log n) | O(log n) | O(log n) split | O(n) |

All Treap bounds are expected with high probability.

---

## Conclusion

Cartesian Tree is the **canonical bridge between arrays and binary trees**:

- Statically, the Cartesian Tree of an array encodes the full RMQ structure: `RMQ(l, r) = LCA(l, r)` on the tree. Combined with an Euler tour and sparse table this gives O(n) preprocessing and O(1) queries — the asymptotically optimal RMQ solution.
- Dynamically, the Treap (randomised Cartesian Tree) provides a self-balancing BST with a clean algebraic split/merge API, expected O(log n) per operation, and easy extensions to order-statistics, lazy propagation, and persistence.
- The O(n) stack build is the key algorithmic insight: the right spine is the only part of the tree that changes when a new element arrives from the right, so a single left-to-right scan suffices.

**Key takeaway:** split and merge are the two atoms of the Treap — insert, erase, kth, and even sequence operations (rotate, reverse with lazy flip) all reduce to at most two splits and one merge. The random priority guarantees O(log n) expected height with the same probability as a random BST, without any rebalancing logic.
