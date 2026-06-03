# Finger Trees

## Origin & Motivation

**Finger Trees** are a purely functional sequence data structure introduced by Hinze and Paterson in 2006. The name comes from the structure: the tree maintains "fingers" — direct references into both ends of the sequence — so that access, insertion, and deletion at both ends cost O(1) amortized. The internal representation is a nested 2-3 tree augmented with a monoidal measure, making it a fully general framework that can represent sequences, priority queues, interval trees, and more by simply swapping the measure monoid.

**The problem it solves:** Standard functional lists give O(1) cons but O(n) append and O(n) indexed access. Arrays give O(1) access but O(n) insertion at ends. Finger Trees achieve O(1) amortized push/pop at both ends, O(log n) concat and split, and O(log n) indexed access — all while remaining purely functional (persistent) with no mutation.

**The key idea:** A Finger Tree is either Empty, a Single node, or a Deep node with a left digit (1–4 elements), a spine (a recursive Finger Tree of 2-3 nodes), and a right digit (1–4 elements). The digits act as buffers: pushes fill the buffer cheaply; when a digit overflows it packs 3 elements into a 2-3 node and pushes that down the spine. The spine holds the bulk of the data, deeply nested so access remains logarithmic. A cached **measure** at each node (e.g. subtree size) enables O(log n) split at any measured position.

Complexity: **O(1)** amortized push/pop at both ends, **O(log n)** concat and split, **O(log n)** indexed access, **O(n)** build.

---

## Where It Is Used

- Haskell `Data.Sequence` — the standard general-purpose sequence in the Haskell ecosystem
- Text editors (Rope-like structures): split and concat at cursor in O(log n)
- Functional priority queues: swap measure to `(min_priority, size)` for O(log n) deleteMin
- Interval trees and ordered sets in functional languages
- XML/parse-tree manipulation with cached annotations (size, line number, etc.)
- Any persistent data structure needing efficient split and concat

---

## Core Structure

### Shape

```
FingerTree a =
    Empty
  | Single a
  | Deep (Digit a) (FingerTree (Node a)) (Digit a)

Digit a = One a | Two a a | Three a a a | Four a a a a

Node a  = Node2 a a | Node3 a a a
```

The spine is a `FingerTree (Node a)` — not a `FingerTree a`. This nesting is what gives the logarithmic depth: elements at depth d in the spine represent `2^d` or `3^d` original elements, so the whole structure has O(log n) levels.

### Measure Monoid

Every node caches a measure — a value from a monoid `(M, zero, combine)`:

```
measure(leaf v)     = measure_of(v)
measure(Node2 a b)  = combine(measure(a), measure(b))
measure(Node3 a b c)= combine(combine(measure(a), measure(b)), measure(c))
measure(Empty)      = zero
measure(Deep l sp r)= combine(combine(measure(l), measure(sp)), measure(r))
```

For a **sequence** (this implementation): `M = int`, `zero = 0`, `combine = (+)`, `measure(leaf) = 1`.
For a **priority queue**: `M = Option<int>`, `combine = min`.
For an **ordered set**: `M = Option<Key>`, recording the maximum key.

Changing the measure is the only modification needed to repurpose the structure.

---

## Operations

### Push Front / Push Back

```
push_front(Empty, a)    = Single(a)
push_front(Single(b), a)= Deep([a], Empty, [b])
push_front(Deep([b,c,d,e], sp, r), a):
    // left digit is full (4 elements) — overflow
    pack Node3(b,c,d) and push_front to spine
    = Deep([a,e], push_front(sp, Node3(b,c,d)), r)
push_front(Deep(l, sp, r), a):
    // digit has room
    = Deep([a] ++ l, sp, r)
```

`push_back` is symmetric. Each push touches O(1) levels on average because overflows cascade with probability 1/2^k at level k (geometric distribution) — amortized O(1).

### Pop Front / Pop Back (viewL / viewR)

```
viewL(Single(a))        = (a, Empty)
viewL(Deep([a], sp, r)):
    if sp == Empty: return (a, digit_to_tree(r))
    (node, sp') = viewL(sp)          // pull next node from spine
    expand node into new left digit
    return (a, Deep(expand(node), sp', r))
viewL(Deep([a,b,...], sp, r)):
    return (a, Deep([b,...], sp, r)) // just shrink left digit
```

### Concat

The key insight: concatenating two Deep trees requires only that the right digit of the left tree and the left digit of the right tree be packed into 2-3 nodes and pushed into the merged spine recursively.

```
concat(l, r) = concat_inner(l, [], r)

concat_inner(Empty, mid, r)    = prepend mid to r
concat_inner(l, mid, Empty)    = append mid to l
concat_inner(Deep(ll,lsp,lr), mid, Deep(rl,rsp,rr)):
    nodes = lr.elements ++ mid ++ rl.elements   // middle buffer
    packed = pack_into_23_nodes(nodes)
    new_spine = concat_inner(lsp, packed, rsp)
    return Deep(ll, new_spine, rr)
```

Packing guarantees every packed group is a Node2 or Node3 (never 1, never 5+), maintaining the invariant. Total cost: O(log n) recursive calls, each O(1) work.

### Split

```
split(t, idx):
    // Returns (left, pivot, right) where left has idx elements
    if measure(t.left) > idx:
        // pivot is in left digit
        (dl, pivot, dr) = split_digit(t.left, idx, 0)
        right = dr ++ spine ++ right_digit  (as tree)
        return (dl, pivot, right)

    if measure(t.left) + measure(t.spine) > idx:
        // pivot is in spine
        (lsp, node, rsp) = split(t.spine, idx - measure(t.left))
        (dl, pivot, dr)  = split_node(node, ...)
        return (t.left ++ lsp ++ dl, pivot, dr ++ rsp ++ t.right)

    // pivot is in right digit
    (dl, pivot, dr) = split_digit(t.right, idx - measure(t.left) - measure(t.spine))
    left = left_digit ++ spine ++ dl  (as tree)
    return (left, pivot, dr)
```

Indexed access (`at(i)`) is just `split(t, i).pivot`.

---

## Amortized Analysis

The amortized O(1) for `push_front` follows from a potential argument. Define potential `Φ = Σ_k (number of full digits at level k) * 2^(-k)`. Each push at a non-full digit costs O(1) and increases Φ by at most O(1). An overflow at level k costs O(1) real work but decreases Φ by `2^(-k)` and increases Φ at level k+1 by at most `2^(-(k+1))` — a net decrease. Summing over all levels the amortized cost per operation is O(1).

---

## Complexity Summary

| Operation | Worst Case | Amortized |
|---|---|---|
| push_front / push_back | O(log n) | O(1) |
| pop_front / pop_back | O(log n) | O(1) |
| front / back (peek) | O(1) | O(1) |
| concat | O(log n) | O(log n) |
| split | O(log n) | O(log n) |
| at (indexed access) | O(log n) | O(log n) |
| build from list | O(n) | O(n) |
| toList | O(n) | O(n) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// FINGER TREE — 2-3 finger tree, measure = sequence length
// O(1) amortized: push_front, push_back, pop_front, pop_back
// O(log n):       concat, split, at (indexed access)
// Based on Hinze & Paterson 2006.
// ================================================================

struct Node;
struct FTree;
using NP = shared_ptr<Node>;
using FP = shared_ptr<FTree>;

// ---- Node: leaf or internal 2/3 node ----
struct Node {
    int  sz;
    bool leaf;
    int  val;      // leaf only
    int  cnt;      // internal: 2 or 3
    NP   ch[3];    // internal

    static NP Leaf(int v){
        auto n=make_shared<Node>(); n->sz=1; n->leaf=true; n->val=v; n->cnt=0; return n;
    }
    static NP N2(NP a, NP b){
        auto n=make_shared<Node>(); n->sz=a->sz+b->sz; n->leaf=false; n->cnt=2;
        n->ch[0]=a; n->ch[1]=b; return n;
    }
    static NP N3(NP a, NP b, NP c){
        auto n=make_shared<Node>(); n->sz=a->sz+b->sz+c->sz; n->leaf=false; n->cnt=3;
        n->ch[0]=a; n->ch[1]=b; n->ch[2]=c; return n;
    }
    void collect(vector<int>& out) const {
        if(leaf){ out.push_back(val); return; }
        for(int i=0;i<cnt;i++) ch[i]->collect(out);
    }
};

// ---- Digit: 1..4 NPs ----
struct Digit {
    NP d[4]; int n=0;
    int sz() const { int s=0; for(int i=0;i<n;i++) s+=d[i]->sz; return s; }
    void collect(vector<int>& out) const { for(int i=0;i<n;i++) d[i]->collect(out); }
};
Digit D1(NP a){Digit d;d.n=1;d.d[0]=a;return d;}
Digit D2(NP a,NP b){Digit d;d.n=2;d.d[0]=a;d.d[1]=b;return d;}
Digit D3(NP a,NP b,NP c){Digit d;d.n=3;d.d[0]=a;d.d[1]=b;d.d[2]=c;return d;}
Digit D4(NP a,NP b,NP c,NP e){Digit d;d.n=4;d.d[0]=a;d.d[1]=b;d.d[2]=c;d.d[3]=e;return d;}

// ---- Finger Tree ----
struct FTree {
    enum { EMPTY, SINGLE, DEEP } kind;
    NP    single;
    Digit left, right;
    FP    spine;
    int   sz;

    static FP E(){ auto t=make_shared<FTree>(); t->kind=EMPTY; t->sz=0; return t; }
    static FP S(NP a){ auto t=make_shared<FTree>(); t->kind=SINGLE; t->single=a; t->sz=a->sz; return t; }
    static FP D(Digit l, FP sp, Digit r){
        auto t=make_shared<FTree>(); t->kind=DEEP;
        t->left=l; t->spine=sp; t->right=r;
        t->sz=l.sz()+sp->sz+r.sz(); return t;
    }
};

// ---- helpers ----
void collect(FP t, vector<int>& out){
    if(t->kind==FTree::EMPTY) return;
    if(t->kind==FTree::SINGLE){ t->single->collect(out); return; }
    t->left.collect(out); collect(t->spine,out); t->right.collect(out);
}

FP dig2tree(const Digit& d){
    if(d.n==1) return FTree::S(d.d[0]);
    if(d.n==2) return FTree::D(D1(d.d[0]),FTree::E(),D1(d.d[1]));
    if(d.n==3) return FTree::D(D2(d.d[0],d.d[1]),FTree::E(),D1(d.d[2]));
    return FTree::D(D2(d.d[0],d.d[1]),FTree::E(),D2(d.d[2],d.d[3]));
}

FP push_front(FP t, NP a);
FP push_back(FP t, NP a);
pair<NP,FP> viewL(FP t);
pair<FP,NP> viewR(FP t);

FP push_front(FP t, NP a){
    if(t->kind==FTree::EMPTY) return FTree::S(a);
    if(t->kind==FTree::SINGLE) return FTree::D(D1(a),FTree::E(),D1(t->single));
    auto& l=t->left;
    if(l.n<4){
        Digit nl; nl.n=l.n+1; nl.d[0]=a;
        for(int i=0;i<l.n;i++) nl.d[i+1]=l.d[i];
        return FTree::D(nl,t->spine,t->right);
    }
    // left digit full: overflow d[1..3] into spine as Node3
    return FTree::D(D2(a,l.d[0]),
                    push_front(t->spine, Node::N3(l.d[1],l.d[2],l.d[3])),
                    t->right);
}
FP push_back(FP t, NP a){
    if(t->kind==FTree::EMPTY) return FTree::S(a);
    if(t->kind==FTree::SINGLE) return FTree::D(D1(t->single),FTree::E(),D1(a));
    auto& r=t->right;
    if(r.n<4){
        Digit nr; nr.n=r.n+1;
        for(int i=0;i<r.n;i++) nr.d[i]=r.d[i]; nr.d[r.n]=a;
        return FTree::D(t->left,t->spine,nr);
    }
    return FTree::D(t->left,
                    push_back(t->spine, Node::N3(r.d[0],r.d[1],r.d[2])),
                    D2(r.d[3],a));
}

// viewL: pop from front — returns (head, rest)
pair<NP,FP> viewL(FP t){
    if(t->kind==FTree::SINGLE) return {t->single, FTree::E()};
    auto& l=t->left; NP hd=l.d[0];
    if(l.n>1){
        Digit nl; nl.n=l.n-1; for(int i=0;i<nl.n;i++) nl.d[i]=l.d[i+1];
        return {hd, FTree::D(nl,t->spine,t->right)};
    }
    // left digit exhausted — pull next node from spine
    if(t->spine->kind==FTree::EMPTY) return {hd, dig2tree(t->right)};
    auto[nd,sp2]=viewL(t->spine);
    Digit nl; nl.n=nd->cnt; for(int i=0;i<nd->cnt;i++) nl.d[i]=nd->ch[i];
    return {hd, FTree::D(nl,sp2,t->right)};
}
// viewR: pop from back — returns (init, last)
pair<FP,NP> viewR(FP t){
    if(t->kind==FTree::SINGLE) return {FTree::E(), t->single};
    auto& r=t->right; NP tl=r.d[r.n-1];
    if(r.n>1){
        Digit nr; nr.n=r.n-1; for(int i=0;i<nr.n;i++) nr.d[i]=r.d[i];
        return {FTree::D(t->left,t->spine,nr), tl};
    }
    if(t->spine->kind==FTree::EMPTY) return {dig2tree(t->left), tl};
    auto[sp2,nd]=viewR(t->spine);
    Digit nr; nr.n=nd->cnt; for(int i=0;i<nd->cnt;i++) nr.d[i]=nd->ch[i];
    return {FTree::D(t->left,sp2,nr), tl};
}

// ---- concat ----
vector<NP> pack(const vector<NP>& ns){
    vector<NP> out; int i=0, m=(int)ns.size();
    while(i<m){
        int r=m-i;
        if(r==2)      { out.push_back(Node::N2(ns[i],ns[i+1])); i+=2; }
        else if(r==4) { out.push_back(Node::N2(ns[i],ns[i+1]));
                        out.push_back(Node::N2(ns[i+2],ns[i+3])); i+=4; }
        else          { out.push_back(Node::N3(ns[i],ns[i+1],ns[i+2])); i+=3; }
    }
    return out;
}
FP concat_mid(FP l, const vector<NP>& mid, FP r){
    if(l->kind==FTree::EMPTY){ FP t=r; for(int i=(int)mid.size()-1;i>=0;i--) t=push_front(t,mid[i]); return t; }
    if(r->kind==FTree::EMPTY){ FP t=l; for(auto& n:mid) t=push_back(t,n); return t; }
    if(l->kind==FTree::SINGLE){ FP t=r; for(int i=(int)mid.size()-1;i>=0;i--) t=push_front(t,mid[i]); return push_front(t,l->single); }
    if(r->kind==FTree::SINGLE){ FP t=l; for(auto& n:mid) t=push_back(t,n); return push_back(t,r->single); }
    // Both DEEP: collect middle nodes, pack into 2-3 nodes, recurse on spines
    vector<NP> nodes;
    for(int i=0;i<l->right.n;i++) nodes.push_back(l->right.d[i]);
    for(auto& n:mid) nodes.push_back(n);
    for(int i=0;i<r->left.n;i++)  nodes.push_back(r->left.d[i]);
    auto pk=pack(nodes);
    return FTree::D(l->left, concat_mid(l->spine,pk,r->spine), r->right);
}
FP ft_concat(FP l, FP r){ return concat_mid(l,{},r); }

// ---- split ----
struct Sp3{ FP l; NP e; FP r; };

// Split a digit at the first index where prefix+accumulated > idx
Sp3 split_dig(const Digit& d, int idx, int prefix){
    for(int k=0;k<d.n;k++){
        int nxt=prefix+d.d[k]->sz;
        if(nxt>idx){
            Digit dl,dr;
            dl.n=k; for(int j=0;j<k;j++) dl.d[j]=d.d[j];
            dr.n=d.n-k-1; for(int j=0;j<dr.n;j++) dr.d[j]=d.d[k+1+j];
            return { dl.n?dig2tree(dl):FTree::E(), d.d[k], dr.n?dig2tree(dr):FTree::E() };
        }
        prefix=nxt;
    }
    int k=d.n-1; Digit dl; dl.n=k; for(int j=0;j<k;j++) dl.d[j]=d.d[j];
    return { dl.n?dig2tree(dl):FTree::E(), d.d[k], FTree::E() };
}

// Split a 2-3 node at sub-index
Sp3 split_node(NP nd, int idx, int prefix){
    for(int k=0;k<nd->cnt;k++){
        int nxt=prefix+nd->ch[k]->sz;
        if(nxt>idx){
            Digit dl,dr;
            dl.n=k; for(int j=0;j<k;j++) dl.d[j]=nd->ch[j];
            dr.n=nd->cnt-k-1; for(int j=0;j<dr.n;j++) dr.d[j]=nd->ch[k+1+j];
            return { dl.n?dig2tree(dl):FTree::E(), nd->ch[k], dr.n?dig2tree(dr):FTree::E() };
        }
        prefix=nxt;
    }
    int k=nd->cnt-1; Digit dl; dl.n=k; for(int j=0;j<k;j++) dl.d[j]=nd->ch[j];
    return { dl.n?dig2tree(dl):FTree::E(), nd->ch[k], FTree::E() };
}

// ft_split: left gets idx elements; pivot is element at position idx (0-based)
Sp3 ft_split(FP t, int idx, int prefix=0){
    if(t->kind==FTree::SINGLE) return {FTree::E(), t->single, FTree::E()};

    int lsz = prefix + t->left.sz();
    if(lsz > idx){
        // pivot in left digit
        auto s = split_dig(t->left, idx, prefix);
        FP rt;
        if(s.r->kind==FTree::EMPTY){
            if(t->spine->kind==FTree::EMPTY) rt=dig2tree(t->right);
            else {
                auto[nd2,sp2]=viewL(t->spine);
                Digit nd; nd.n=nd2->cnt; for(int j=0;j<nd2->cnt;j++) nd.d[j]=nd2->ch[j];
                rt=FTree::D(nd,sp2,t->right);
            }
        } else {
            // s.r is a partial tree; append spine+right to it
            FP spine_right = t->spine->kind==FTree::EMPTY ? dig2tree(t->right) :
                [&]()->FP{
                    auto[nd2,sp2]=viewL(t->spine);
                    Digit nd; nd.n=nd2->cnt; for(int j=0;j<nd2->cnt;j++) nd.d[j]=nd2->ch[j];
                    return FTree::D(nd,sp2,t->right);
                }();
            rt = ft_concat(s.r, spine_right);
        }
        return {s.l, s.e, rt};
    }

    int ssz = lsz + t->spine->sz;
    if(ssz > idx){
        // pivot in spine — recurse
        auto s  = ft_split(t->spine, idx, lsz);
        auto s2 = split_node(s.e, idx, lsz + s.l->sz);
        FP lt = ft_concat(ft_concat(dig2tree(t->left), s.l), s2.l);
        FP rt = ft_concat(ft_concat(s2.r, s.r), dig2tree(t->right));
        return {lt, s2.e, rt};
    }

    // pivot in right digit
    auto s = split_dig(t->right, idx, ssz);
    FP lt;
    if(s.l->kind==FTree::EMPTY){
        if(t->spine->kind==FTree::EMPTY) lt=dig2tree(t->left);
        else {
            auto[sp2,nd2]=viewR(t->spine);
            Digit nd; nd.n=nd2->cnt; for(int j=0;j<nd2->cnt;j++) nd.d[j]=nd2->ch[j];
            lt=FTree::D(t->left,sp2,nd);
        }
    } else {
        FP spine_left = t->spine->kind==FTree::EMPTY ? dig2tree(t->left) :
            [&]()->FP{
                auto[sp2,nd2]=viewR(t->spine);
                Digit nd; nd.n=nd2->cnt; for(int j=0;j<nd2->cnt;j++) nd.d[j]=nd2->ch[j];
                return FTree::D(t->left,sp2,nd);
            }();
        lt = ft_concat(spine_left, s.l);
    }
    return {lt, s.e, s.r};
}

// ================================================================
// Public Seq wrapper
// ================================================================
struct Seq {
    FP t;
    Seq() : t(FTree::E()) {}
    Seq(FP tt) : t(tt) {}

    int  size()  const { return t->sz; }
    bool empty() const { return t->kind==FTree::EMPTY; }

    void push_front(int v) { t = ::push_front(t, Node::Leaf(v)); }
    void push_back(int v)  { t = ::push_back(t,  Node::Leaf(v)); }

    int  front() const { return viewL(t).first->val; }
    int  back()  const { return viewR(t).second->val; }

    void pop_front() { t = viewL(t).second; }
    void pop_back()  { t = viewR(t).first; }

    // O(log n) indexed access
    int at(int i) const { return ft_split(t, i).e->val; }

    // split_at(i): returns ([0..i-1], [i..n-1])
    pair<Seq,Seq> split_at(int i) const {
        if(i <= 0)      return {Seq(), *this};
        if(i >= size()) return {*this, Seq()};
        auto s = ft_split(t, i);   // left has i elements, s.e is element[i]
        return {Seq(s.l), Seq(::push_front(s.r, s.e))};
    }

    static Seq concat(const Seq& a, const Seq& b) { return Seq(ft_concat(a.t, b.t)); }

    vector<int> toVec() const {
        vector<int> v; ::collect(t, v); return v;
    }
};

// ================================================================
// Usage + stress test
// ================================================================
int main(){
    printf("=== Finger Tree Demo ===\n");
    Seq s;
    for(int i=1;i<=10;i++) s.push_back(i);
    printf("push_back 1..10: "); for(int x:s.toVec()) printf("%d ",x); printf("\n");

    s.push_front(0);
    printf("push_front(0): front=%d size=%d\n", s.front(), s.size());

    s.pop_front(); s.pop_back();
    printf("pop_front+pop_back: "); for(int x:s.toVec()) printf("%d ",x); printf("\n");

    printf("at(0)=%d  at(4)=%d  at(8)=%d\n", s.at(0), s.at(4), s.at(8));

    auto[L,R]=s.split_at(5);
    printf("split_at(5) L:"); for(int x:L.toVec()) printf(" %d",x); printf("\n");
    printf("split_at(5) R:"); for(int x:R.toVec()) printf(" %d",x); printf("\n");

    Seq j=Seq::concat(L,R);
    printf("concat:"); for(int x:j.toVec()) printf(" %d",x); printf("\n");

    printf("\n=== Stress test ===\n");
    srand(42); int errors=0;
    for(int trial=0;trial<500;trial++){
        Seq seq; deque<int> ref;
        for(int i=0;i<120;i++){
            int op=rand()%4, v=rand()%1000;
            if(op==0){ seq.push_front(v); ref.push_front(v); }
            if(op==1){ seq.push_back(v);  ref.push_back(v);  }
            if(op==2&&!ref.empty()){ seq.pop_front(); ref.pop_front(); }
            if(op==3&&!ref.empty()){ seq.pop_back();  ref.pop_back();  }
        }
        auto got=seq.toVec(); vector<int>exp(ref.begin(),ref.end());
        if(got!=exp){ errors++; continue; }
        if(!ref.empty()){
            int idx=rand()%(int)ref.size();
            if(seq.at(idx)!=(int)ref[idx]){ errors++; continue; }
        }
        if((int)ref.size()>2){
            int idx=rand()%(int)ref.size();
            auto[sl,sr]=seq.split_at(idx);
            if(Seq::concat(sl,sr).toVec()!=exp){ errors++; continue; }
        }
    }
    printf("500 trials push/pop/at/split/concat: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

// ================================================================
// FINGER TREE — C# implementation, measure = sequence size
// O(1) amortized push/pop at both ends
// O(log n) concat, split, indexed access
// ================================================================
public class FingerTree
{
    // ---- Node ----
    private class Node
    {
        public int  Sz;
        public bool Leaf;
        public int  Val;        // leaf only
        public int  Cnt;        // internal: 2 or 3
        public Node[] Ch;       // internal

        public static Node MakeLeaf(int v)
            => new Node { Sz=1, Leaf=true, Val=v, Cnt=0 };

        public static Node N2(Node a, Node b)
            => new Node { Sz=a.Sz+b.Sz, Leaf=false, Cnt=2, Ch=new[]{a,b} };

        public static Node N3(Node a, Node b, Node c)
            => new Node { Sz=a.Sz+b.Sz+c.Sz, Leaf=false, Cnt=3, Ch=new[]{a,b,c} };

        public void Collect(List<int> out_){
            if(Leaf){ out_.Add(Val); return; }
            foreach(var c in Ch) c.Collect(out_);
        }
    }

    // ---- Digit ----
    private class Digit
    {
        public Node[] D; public int N;
        public int Sz(){ int s=0; for(int i=0;i<N;i++) s+=D[i].Sz; return s; }
        public void Collect(List<int> o){ for(int i=0;i<N;i++) D[i].Collect(o); }
        public static Digit Of(params Node[] ds){ return new Digit{D=ds,N=ds.Length}; }
    }

    // ---- Tree ----
    private enum Kind { Empty, Single, Deep }
    private Kind   _kind;
    private Node   _single;
    private Digit  _left, _right;
    private FingerTree _spine;
    private int    _sz;

    private static FingerTree Empty_ = new FingerTree{ _kind=Kind.Empty, _sz=0 };
    private static FingerTree Sing(Node a)
        => new FingerTree{ _kind=Kind.Single, _single=a, _sz=a.Sz };
    private static FingerTree Deep(Digit l, FingerTree sp, Digit r)
        => new FingerTree{ _kind=Kind.Deep, _left=l, _spine=sp, _right=r,
                           _sz=l.Sz()+sp._sz+r.Sz() };

    // ---- public interface ----
    public static FingerTree Empty() => Empty_;
    public int  Size  => _sz;
    public bool IsEmpty => _kind==Kind.Empty;

    private static FingerTree Dig2Tree(Digit d){
        if(d.N==1) return Sing(d.D[0]);
        if(d.N==2) return Deep(Digit.Of(d.D[0]), Empty_, Digit.Of(d.D[1]));
        if(d.N==3) return Deep(Digit.Of(d.D[0],d.D[1]), Empty_, Digit.Of(d.D[2]));
        return Deep(Digit.Of(d.D[0],d.D[1]), Empty_, Digit.Of(d.D[2],d.D[3]));
    }

    // ---- PushFront / PushBack ----
    public FingerTree PushFront(int v) => PushFrontN(Node.MakeLeaf(v));
    public FingerTree PushBack(int v)  => PushBackN(Node.MakeLeaf(v));

    private FingerTree PushFrontN(Node a){
        if(_kind==Kind.Empty) return Sing(a);
        if(_kind==Kind.Single) return Deep(Digit.Of(a), Empty_, Digit.Of(_single));
        var l=_left;
        if(l.N<4){
            var nl=new Node[l.N+1]; nl[0]=a; for(int i=0;i<l.N;i++) nl[i+1]=l.D[i];
            return Deep(Digit.Of(nl),_spine,_right);
        }
        return Deep(Digit.Of(a,l.D[0]),
                    _spine.PushFrontN(Node.N3(l.D[1],l.D[2],l.D[3])),
                    _right);
    }
    private FingerTree PushBackN(Node a){
        if(_kind==Kind.Empty) return Sing(a);
        if(_kind==Kind.Single) return Deep(Digit.Of(_single), Empty_, Digit.Of(a));
        var r=_right;
        if(r.N<4){
            var nr=new Node[r.N+1]; for(int i=0;i<r.N;i++) nr[i]=r.D[i]; nr[r.N]=a;
            return Deep(_left,_spine,Digit.Of(nr));
        }
        return Deep(_left,
                    _spine.PushBackN(Node.N3(r.D[0],r.D[1],r.D[2])),
                    Digit.Of(r.D[3],a));
    }

    // ---- ViewL / ViewR ----
    private (Node head, FingerTree rest) ViewL(){
        if(_kind==Kind.Single) return (_single, Empty_);
        var l=_left; var hd=l.D[0];
        if(l.N>1){
            var nl=new Node[l.N-1]; for(int i=0;i<nl.Length;i++) nl[i]=l.D[i+1];
            return (hd, Deep(Digit.Of(nl),_spine,_right));
        }
        if(_spine._kind==Kind.Empty) return (hd, Dig2Tree(_right));
        var (nd,sp2)=_spine.ViewL();
        return (hd, Deep(Digit.Of(nd.Ch),sp2,_right));
    }
    private (FingerTree init, Node last) ViewR(){
        if(_kind==Kind.Single) return (Empty_, _single);
        var r=_right; var tl=r.D[r.N-1];
        if(r.N>1){
            var nr=new Node[r.N-1]; for(int i=0;i<nr.Length;i++) nr[i]=r.D[i];
            return (Deep(_left,_spine,Digit.Of(nr)), tl);
        }
        if(_spine._kind==Kind.Empty) return (Dig2Tree(_left), tl);
        var (sp2,nd)=_spine.ViewR();
        return (Deep(_left,sp2,Digit.Of(nd.Ch)), tl);
    }

    public int  Front() { return ViewL().head.Val; }
    public int  Back()  { return ViewR().last.Val; }
    public FingerTree PopFront() { return ViewL().rest; }
    public FingerTree PopBack()  { return ViewR().init; }

    // ---- Concat ----
    private static Node[] Pack(Node[] ns){
        var out_=new List<Node>(); int i=0,m=ns.Length;
        while(i<m){
            int r=m-i;
            if(r==2)      { out_.Add(Node.N2(ns[i],ns[i+1])); i+=2; }
            else if(r==4) { out_.Add(Node.N2(ns[i],ns[i+1])); out_.Add(Node.N2(ns[i+2],ns[i+3])); i+=4; }
            else          { out_.Add(Node.N3(ns[i],ns[i+1],ns[i+2])); i+=3; }
        }
        return out_.ToArray();
    }
    private static FingerTree ConcatMid(FingerTree l, Node[] mid, FingerTree r){
        if(l._kind==Kind.Empty){ var t=r; for(int i=mid.Length-1;i>=0;i--) t=t.PushFrontN(mid[i]); return t; }
        if(r._kind==Kind.Empty){ var t=l; foreach(var n in mid) t=t.PushBackN(n); return t; }
        if(l._kind==Kind.Single){ var t=r; for(int i=mid.Length-1;i>=0;i--) t=t.PushFrontN(mid[i]); return t.PushFrontN(l._single); }
        if(r._kind==Kind.Single){ var t=l; foreach(var n in mid) t=t.PushBackN(n); return t.PushBackN(r._single); }
        var nodes=new List<Node>();
        for(int i=0;i<l._right.N;i++) nodes.Add(l._right.D[i]);
        foreach(var n in mid) nodes.Add(n);
        for(int i=0;i<r._left.N;i++)  nodes.Add(r._left.D[i]);
        var pk=Pack(nodes.ToArray());
        return Deep(l._left, ConcatMid(l._spine,pk,r._spine), r._right);
    }
    public static FingerTree Concat(FingerTree a, FingerTree b)
        => ConcatMid(a, Array.Empty<Node>(), b);

    // ---- Split ----
    private struct Sp3{ public FingerTree L; public Node E; public FingerTree R; }

    private static Sp3 SplitDig(Digit d, int idx, int prefix){
        for(int k=0;k<d.N;k++){
            int nxt=prefix+d.D[k].Sz;
            if(nxt>idx){
                var dl=new Node[k]; for(int j=0;j<k;j++) dl[j]=d.D[j];
                var dr=new Node[d.N-k-1]; for(int j=0;j<dr.Length;j++) dr[j]=d.D[k+1+j];
                return new Sp3{ L=dl.Length>0?Dig2Tree(Digit.Of(dl)):Empty_,
                                E=d.D[k],
                                R=dr.Length>0?Dig2Tree(Digit.Of(dr)):Empty_ };
            }
            prefix=nxt;
        }
        var dl2=new Node[d.N-1]; for(int j=0;j<dl2.Length;j++) dl2[j]=d.D[j];
        return new Sp3{ L=dl2.Length>0?Dig2Tree(Digit.Of(dl2)):Empty_, E=d.D[d.N-1], R=Empty_ };
    }
    private static Sp3 SplitNode(Node nd, int idx, int prefix){
        for(int k=0;k<nd.Cnt;k++){
            int nxt=prefix+nd.Ch[k].Sz;
            if(nxt>idx){
                var dl=new Node[k]; for(int j=0;j<k;j++) dl[j]=nd.Ch[j];
                var dr=new Node[nd.Cnt-k-1]; for(int j=0;j<dr.Length;j++) dr[j]=nd.Ch[k+1+j];
                return new Sp3{ L=dl.Length>0?Dig2Tree(Digit.Of(dl)):Empty_,
                                E=nd.Ch[k],
                                R=dr.Length>0?Dig2Tree(Digit.Of(dr)):Empty_ };
            }
            prefix=nxt;
        }
        var dl2=new Node[nd.Cnt-1]; for(int j=0;j<dl2.Length;j++) dl2[j]=nd.Ch[j];
        return new Sp3{ L=dl2.Length>0?Dig2Tree(Digit.Of(dl2)):Empty_, E=nd.Ch[nd.Cnt-1], R=Empty_ };
    }
    private Sp3 Split_(int idx, int prefix=0){
        if(_kind==Kind.Single) return new Sp3{L=Empty_,E=_single,R=Empty_};
        int lsz=prefix+_left.Sz();
        if(lsz>idx){
            var s=SplitDig(_left,idx,prefix);
            FingerTree rt;
            if(s.R._kind==Kind.Empty){
                if(_spine._kind==Kind.Empty) rt=Dig2Tree(_right);
                else { var(nd2,sp2)=_spine.ViewL(); rt=Deep(Digit.Of(nd2.Ch),sp2,_right); }
            } else {
                FingerTree sr = _spine._kind==Kind.Empty ? Dig2Tree(_right) :
                    (()=>{ var(nd2,sp2)=_spine.ViewL(); return Deep(Digit.Of(nd2.Ch),sp2,_right); })();
                rt=Concat(s.R,sr);
            }
            return new Sp3{L=s.L,E=s.E,R=rt};
        }
        int ssz=lsz+_spine._sz;
        if(ssz>idx){
            var s=_spine.Split_(idx,lsz);
            var s2=SplitNode(s.E,idx,lsz+s.L._sz);
            return new Sp3{
                L=Concat(Concat(Dig2Tree(_left),s.L),s2.L),
                E=s2.E,
                R=Concat(Concat(s2.R,s.R),Dig2Tree(_right))
            };
        }
        var sd=SplitDig(_right,idx,ssz);
        FingerTree lt;
        if(sd.L._kind==Kind.Empty){
            if(_spine._kind==Kind.Empty) lt=Dig2Tree(_left);
            else { var(sp2,nd2)=_spine.ViewR(); lt=Deep(_left,sp2,Digit.Of(nd2.Ch)); }
        } else {
            FingerTree sl2 = _spine._kind==Kind.Empty ? Dig2Tree(_left) :
                (()=>{ var(sp2,nd2)=_spine.ViewR(); return Deep(_left,sp2,Digit.Of(nd2.Ch)); })();
            lt=Concat(sl2,sd.L);
        }
        return new Sp3{L=lt,E=sd.E,R=sd.R};
    }

    // ---- public API ----
    public int At(int i){ return Split_(i).E.Val; }

    public (FingerTree left, FingerTree right) SplitAt(int i){
        if(i<=0) return (Empty_, this);
        if(i>=_sz) return (this, Empty_);
        var s=Split_(i);
        return (s.L, s.R.PushFrontN(s.E));
    }

    public List<int> ToList(){
        var out_=new List<int>();
        void Col(FingerTree t){
            if(t._kind==Kind.Empty) return;
            if(t._kind==Kind.Single){ t._single.Collect(out_); return; }
            t._left.Collect(out_); Col(t._spine); t._right.Collect(out_);
        }
        Col(this); return out_;
    }

    public static void Main(){
        var s = FingerTree.Empty();
        for(int i=1;i<=10;i++) s=s.PushBack(i);
        Console.WriteLine("push_back 1..10: "+string.Join(" ",s.ToList()));
        s=s.PushFront(0);
        Console.WriteLine($"push_front(0): front={s.Front()} size={s.Size}");
        s=s.PopFront().PopBack();
        Console.WriteLine("pop_front+pop_back: "+string.Join(" ",s.ToList()));
        Console.WriteLine($"at(0)={s.At(0)} at(4)={s.At(4)} at(8)={s.At(8)}");
        var(L,R)=s.SplitAt(5);
        Console.WriteLine("split_at(5) L: "+string.Join(" ",L.ToList()));
        Console.WriteLine("split_at(5) R: "+string.Join(" ",R.ToList()));
        Console.WriteLine("concat: "+string.Join(" ",Concat(L,R).ToList()));
    }
}
```

---

## Pitfalls

- **Overflow packs wrong elements** — on `push_front` when the left digit has 4 elements, the 3 elements packed into a Node3 and pushed to the spine must be `d[1], d[2], d[3]` (the inner three), not `d[0..2]`. The new digit becomes `[a, d[0]]`. Swapping this corrupts element order at every subsequent overflow.
- **viewL on an empty spine** — when pulling a new left digit from an exhausted left finger, if the spine is Empty the entire right digit becomes the new tree (`dig2tree(right)`). Missing this case and calling `viewL(empty)` is undefined behavior.
- **pack must handle remainder 4 correctly** — when packing middle nodes into 2-3 nodes for concat, a remainder of 4 must produce two Node2s, not one Node4 (which does not exist). A remainder of 1 is impossible by construction if the concat algorithm is correct.
- **split_at index semantics** — `ft_split(t, i)` returns a pivot that is element at index `i` (0-based). The left tree has exactly `i` elements. The right tree of the split does NOT include the pivot; `push_front(right, pivot)` reconstructs the full right half including element `i`. Forgetting to re-insert the pivot loses an element.
- **Nested spine type** — the spine is a `FingerTree<Node>`, not a `FingerTree<int>`. Elements pushed into the spine are 2-3 nodes representing multiple values, not individual integers. Confusing these levels produces wrong sizes and wrong traversal order.
- **Measure cache must be recomputed on every constructor** — every tree constructor (`Deep`, `Sing`) must compute the cached size from its children, not inherit or copy it. Sharing stale cached measures from nodes before a structural change silently corrupts all split and at operations.
- **split_node prefix accumulation** — when splitting a 2-3 node, the `prefix` argument is the accumulated size before this entire node, not before this call level. Passing the wrong prefix causes the split to fire at the wrong child, misplacing the pivot.
- **Digit size 0 or 5 is invalid** — a Digit always has 1–4 elements. A digit with 0 elements must never appear in a Deep node; an empty left or right after a split must be represented as a different tree shape (`Single` or recursive dig2tree). Allowing empty digits breaks the push overflow logic (overflow check `n < 4` would never trigger) and corrupts viewL/viewR.

---

## Conclusion

Finger Trees are the **canonical purely functional general-purpose sequence**:

- The nested 2-3 structure with digit buffers at both ends gives O(1) amortized push/pop at both ends — matching a deque — while remaining fully persistent.
- The monoidal measure annotation enables O(log n) split at any measurable position (size, priority, key) without rebuilding the tree — making one structure serve as sequence, priority queue, and ordered map depending only on the choice of monoid.
- Concat reduces to packing the two inner digits into 2-3 nodes and recursively merging the spines — O(log n) total, matching the lower bound for persistent concatenation.
- The key implementation invariants are: digit size 1–4 at every level, measure cache recomputed at every constructor, and the spine stores `Node<a>` not `a` — each level of the spine groups elements into 2-3 clusters, keeping the height logarithmic.

**Key takeaway:** The digit acts as an O(1) buffer; the spine holds the bulk of the data encoded as nested 2-3 nodes. Push fills the buffer in O(1); when the buffer overflows it packs 3 elements into a Node3 and pushes that single node down one spine level — a geometric series argument gives amortized O(1). Split binary-searches the cached measures down the spine in O(log n), making indexed access, priority extraction, and interval search all fall out of the same mechanism by changing only the monoid.
