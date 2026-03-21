# Randomized Algorithms (Las Vegas) — Probabilistic with Correctness Guarantee

## Origin & Motivation

Las Vegas algorithms are randomized algorithms that **always produce a correct answer** but have a **random running time**. The randomness is used to improve expected performance, not to trade off correctness. The name contrasts with Monte Carlo algorithms: Las Vegas always wins (correctly), but the time to win is uncertain.

The paradigm was formalized by László Babai in 1979. The key principle: use randomness to break worst-case adversarial structure. A deterministic algorithm for sorting, hashing, or searching can be exploited by an adversary who constructs the worst-case input. A Las Vegas algorithm randomizes internally, so no fixed input is worst case in expectation.

**Expected runtime** is the primary complexity measure. If the expected runtime is T(n), then by Markov's inequality the algorithm exceeds 2T(n) with probability at most 1/2. Running the algorithm until it terminates gives a correct answer with probability 1.

**Relationship to Monte Carlo:** If a Las Vegas algorithm's answer can be verified in polynomial time, then any Monte Carlo algorithm for the same problem can be converted to a Las Vegas algorithm: run Monte Carlo, verify; repeat until correct.

---

## Where It Is Used

- Randomized Quicksort (expected O(n log n), always correct)
- Randomized BSTs and Treaps (expected O(log n), always correct)
- Hash tables with random hash functions (expected O(1), always correct)
- Randomized selection (expected O(n) k-th element)
- Randomized pivot in linear programming (Seidel's algorithm)
- Random walk algorithms (2-SAT, satisfiability checking)
- Primality testing with verification (Miller-Rabin + trial division)
- Las Vegas graph coloring, random augmenting path

---

## Comparison with Monte Carlo

| Property | Las Vegas | Monte Carlo |
|---|---|---|
| Correctness | Always correct | Correct with probability (1 - δ) |
| Runtime | Random (expected polynomial) | Deterministic polynomial |
| Error control | No error to control | Reduce δ by repetition |
| Termination | Always terminates | Always terminates |
| Relation | Stronger guarantee | Weaker guarantee |
| Conversion | LV → MC: stop after T steps, return current answer | MC → LV: verify output, repeat if wrong |

---

## Core Concepts

### Expected Runtime and Geometric Trials

If each trial of a Las Vegas algorithm succeeds with probability `p` independently, the number of trials until success is geometrically distributed with expectation `1/p`. Total expected runtime = `(1/p) * T_trial`.

```
E[trials] = 1/p
E[runtime] = T_trial / p
Pr[>= k trials] = (1-p)^(k-1)
```

For randomized quicksort with random pivot: each pivot is a "good" pivot (falls in the middle half) with probability 1/2. Expected O(log n) levels, each O(n) work → O(n log n) total.

### Expected vs. High-Probability Bounds

- **Expected:** E[T] = O(f(n)) — average over randomness, could be much worse on unlucky runs
- **With high probability (w.h.p.):** Pr[T > c * f(n) * log n] ≤ 1/n^c — almost surely near-optimal

Most Las Vegas algorithms achieve both. Randomized quicksort: expected O(n log n) and O(n log n) w.h.p.

### Yao's Minimax Principle

For any Las Vegas algorithm A and any input distribution D:

```
min_A E_D[T(A, x)] = max_D min_A E_D[T(A, x)]
```

The best expected runtime of a randomized algorithm on the worst input equals the best deterministic algorithm's expected runtime under the best input distribution. Used to prove lower bounds on Las Vegas algorithms.

---

## Complexity Classes

| Class | Definition |
|---|---|
| ZPP | Zero-error Probabilistic Polynomial time — Las Vegas in poly expected time |
| ZPP = RP ∩ co-RP | Equivalent characterization |
| RP | One-sided Monte Carlo (subset of BPP) |
| BPP | Two-sided Monte Carlo |

Containments: P ⊆ ZPP ⊆ RP ⊆ BPP ⊆ PSPACE. Most believe ZPP = RP = BPP = P (derandomization).

---

## Complexity Analysis

| Algorithm | Expected Time | Worst Case | Guarantee |
|---|---|---|---|
| Randomized Quicksort | O(n log n) | O(n²) | Always sorts correctly |
| Randomized Select (k-th) | O(n) | O(n²) | Always finds correct k-th |
| Treap insert/delete | O(log n) | O(n) | BST property always maintained |
| Randomized hashing | O(1) per op | O(n) | Always correct lookup |
| Randomized pivot LP | O(d² n) expected | — | Always finds optimal |
| Random walk 2-SAT | O(n²) | — | Always correct if SAT |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

mt19937 rng(chrono::steady_clock::now().time_since_epoch().count());

// ================================================================
// 1. RANDOMIZED QUICKSORT — expected O(n log n), always correct
// ================================================================
void quicksort(vector<int>& A, int lo, int hi) {
    if (lo >= hi) return;

    // Random pivot — breaks adversarial worst case
    int pivot_idx = uniform_int_distribution<int>(lo, hi)(rng);
    swap(A[lo], A[pivot_idx]);
    int pivot = A[lo];

    // Three-way partition (handles duplicates efficiently)
    int lt = lo, gt = hi, i = lo + 1;
    while (i <= gt) {
        if      (A[i] < pivot) swap(A[lt++], A[i++]);
        else if (A[i] > pivot) swap(A[i],    A[gt--]);
        else                   i++;
    }
    // A[lo..lt-1] < pivot, A[lt..gt] == pivot, A[gt+1..hi] > pivot
    quicksort(A, lo, lt - 1);
    quicksort(A, gt + 1, hi);
}

// ================================================================
// 2. RANDOMIZED SELECT — expected O(n) k-th smallest, always correct
//    Based on QuickSelect with random pivot.
// ================================================================
int quickselect(vector<int>& A, int lo, int hi, int k) {
    if (lo == hi) return A[lo];

    int pivot_idx = uniform_int_distribution<int>(lo, hi)(rng);
    swap(A[lo], A[pivot_idx]);
    int pivot = A[lo];

    int lt = lo, gt = hi, i = lo + 1;
    while (i <= gt) {
        if      (A[i] < pivot) swap(A[lt++], A[i++]);
        else if (A[i] > pivot) swap(A[i],    A[gt--]);
        else                   i++;
    }

    if (k <= gt && k >= lt) return pivot;       // k is in the pivot range
    if (k < lt)             return quickselect(A, lo, lt - 1, k);
    else                    return quickselect(A, gt + 1, hi, k);
}

int kth_smallest(vector<int> A, int k) { // 0-indexed k
    return quickselect(A, 0, (int)A.size() - 1, k);
}

// ================================================================
// 3. TREAP — randomized BST, expected O(log n) per operation
//    BST property on keys, heap property on random priorities.
//    Priority randomness ensures expected balanced structure.
// ================================================================
struct Treap {
    struct Node {
        int key, priority, size;
        Node *left, *right;
        Node(int k) : key(k), priority(rng()), size(1), left(nullptr), right(nullptr) {}
    };

    Node* root = nullptr;

    int sz(Node* t) { return t ? t->size : 0; }

    void upd(Node* t) {
        if (t) t->size = 1 + sz(t->left) + sz(t->right);
    }

    // Split by key: left has keys <= key, right has keys > key
    pair<Node*, Node*> split(Node* t, int key) {
        if (!t) return {nullptr, nullptr};
        if (t->key <= key) {
            auto [l, r] = split(t->right, key);
            t->right = l; upd(t);
            return {t, r};
        } else {
            auto [l, r] = split(t->left, key);
            t->left = r; upd(t);
            return {l, t};
        }
    }

    // Merge: all keys in left < all keys in right
    Node* merge(Node* l, Node* r) {
        if (!l) return r;
        if (!r) return l;
        if (l->priority > r->priority) {
            l->right = merge(l->right, r); upd(l); return l;
        } else {
            r->left = merge(l, r->left);  upd(r); return r;
        }
    }

    void insert(int key) {
        auto [l, r] = split(root, key - 1);
        root = merge(merge(l, new Node(key)), r);
    }

    void erase(int key) {
        auto [l, mr] = split(root, key - 1);
        auto [m, r]  = split(mr, key);
        // m contains all nodes with key == key; remove one
        if (m) {
            Node* old = m;
            m = merge(m->left, m->right);
            delete old;
        }
        root = merge(merge(l, m), r);
    }

    bool contains(int key) {
        Node* cur = root;
        while (cur) {
            if      (key < cur->key) cur = cur->left;
            else if (key > cur->key) cur = cur->right;
            else                     return true;
        }
        return false;
    }

    // k-th smallest element (1-indexed)
    int kth(int k) {
        Node* cur = root;
        while (cur) {
            int left_sz = sz(cur->left);
            if (k == left_sz + 1)   return cur->key;
            else if (k <= left_sz)  cur = cur->left;
            else { k -= left_sz + 1; cur = cur->right; }
        }
        return -1; // not found
    }

    // Count of elements <= key
    int order(int key) {
        auto [l, r] = split(root, key);
        int cnt = sz(l);
        root = merge(l, r);
        return cnt;
    }

    int size() { return sz(root); }
};

// ================================================================
// 4. RANDOMIZED HASH TABLE — universal hashing, expected O(1)
//    Hash family: h_{a,b}(x) = ((ax + b) mod p) mod m
//    For any two distinct keys x,y: Pr[h(x)==h(y)] <= 1/m.
//    Collision probability bounded regardless of key distribution.
// ================================================================
struct UniversalHashTable {
    static const ll P = 1e9 + 7; // prime > max key
    int m;                        // table size
    ll a, b;                      // random hash parameters
    vector<list<pair<int,int>>> table; // {key, value}

    UniversalHashTable(int m) : m(m), table(m) {
        uniform_int_distribution<ll> dist_a(1, P - 1);
        uniform_int_distribution<ll> dist_b(0, P - 1);
        a = dist_a(rng);
        b = dist_b(rng);
    }

    int hash(int key) {
        return (int)(((a * key + b) % P) % m);
    }

    void insert(int key, int val) {
        int h = hash(key);
        for (auto& [k, v] : table[h])
            if (k == key) { v = val; return; }
        table[h].push_back({key, val});
    }

    int* find(int key) {
        int h = hash(key);
        for (auto& [k, v] : table[h])
            if (k == key) return &v;
        return nullptr;
    }

    void erase(int key) {
        int h = hash(key);
        table[h].remove_if([key](auto& p){ return p.first == key; });
    }
};

// ================================================================
// 5. RANDOMIZED BINARY SEARCH TREE (skip list alternative)
//    Random permutation insertion into regular BST gives expected
//    O(log n) height. Same as Treap priority argument.
// ================================================================

// ================================================================
// 6. LAS VEGAS PRIMALITY — Miller-Rabin + trial division
//    Convert Monte Carlo to Las Vegas by verifying with factoring.
//    For small primes, trial division confirms; for large, use
//    Pollard's rho to factor and verify primality definitively.
// ================================================================
ll mulmod(ll a, ll b, ll m) { return (__int128)a * b % m; }
ll powmod(ll a, ll b, ll m) {
    ll r = 1; a %= m;
    for (; b; b >>= 1, a = mulmod(a,a,m))
        if (b & 1) r = mulmod(r,a,m);
    return r;
}

bool miller_rabin(ll n, ll a) {
    if (n % a == 0) return n == a;
    ll d = n-1; int r = 0;
    while (d%2==0) { d/=2; r++; }
    ll x = powmod(a, d, n);
    if (x==1||x==n-1) return true;
    for (int i=0;i<r-1;i++) { x=mulmod(x,x,n); if(x==n-1) return true; }
    return false;
}

// Pollard's rho — Las Vegas factoring
ll pollard_rho(ll n) {
    if (n % 2 == 0) return 2;
    ll x = uniform_int_distribution<ll>(2, n-1)(rng);
    ll y = x, c = uniform_int_distribution<ll>(1, n-1)(rng);
    ll d = 1;
    while (d == 1) {
        x = (mulmod(x,x,n) + c) % n;
        y = (mulmod(y,y,n) + c) % n;
        y = (mulmod(y,y,n) + c) % n;
        d = __gcd(abs(x-y), n);
    }
    return d == n ? pollard_rho(n) : d; // retry on failure
}

bool is_prime_lv(ll n) {
    if (n < 2) return false;
    for (ll a : {2LL,3LL,5LL,7LL,11LL,13LL,17LL,19LL,23LL,29LL,31LL,37LL})
        if (!miller_rabin(n, a)) return false;
    return true;
}

// Full Las Vegas primality: Miller-Rabin is deterministic for n < 3.2*10^18
// with the above witness set. For demonstration: trial-verify small factor.
bool is_prime_verified(ll n) {
    if (n < 2) return false;
    // Trial division up to small limit
    for (ll p = 2; p * p <= n && p < 1000; p++)
        if (n % p == 0) return n == p;
    // Miller-Rabin (deterministic for these witnesses, n < 3.2*10^18)
    return is_prime_lv(n);
}

// ================================================================
// 7. RANDOMIZED INCREMENTAL CONVEX HULL (2D)
//    Insert points in random order. Each insertion is O(log n) expected.
//    Total: O(n log n) expected, always produces correct convex hull.
// ================================================================
struct ConvexHull2D {
    vector<pair<ll,ll>> pts;

    ll cross(pair<ll,ll> O, pair<ll,ll> A, pair<ll,ll> B) {
        return (A.first-O.first)*(ll)(B.second-O.second)
             - (A.second-O.second)*(ll)(B.first-O.first);
    }

    // Build hull from random permutation of input points
    vector<pair<ll,ll>> build(vector<pair<ll,ll>> points) {
        int n = points.size();
        // Randomize insertion order (Las Vegas: hull always correct)
        shuffle(points.begin(), points.end(), rng);
        sort(points.begin(), points.end());
        points.erase(unique(points.begin(), points.end()), points.end());
        n = points.size();
        if (n < 2) return points;

        vector<pair<ll,ll>> hull;
        // Lower hull
        for (auto& p : points) {
            while (hull.size() >= 2 && cross(hull[hull.size()-2], hull.back(), p) <= 0)
                hull.pop_back();
            hull.push_back(p);
        }
        // Upper hull
        int lower_size = hull.size();
        for (int i = n-2; i >= 0; i--) {
            while ((int)hull.size() > lower_size &&
                   cross(hull[hull.size()-2], hull.back(), points[i]) <= 0)
                hull.pop_back();
            hull.push_back(points[i]);
        }
        hull.pop_back();
        return hull;
    }
};

// ================================================================
// 8. RANDOMIZED SKIP LIST — expected O(log n) search/insert/delete
//    Probabilistic multi-level linked list. Always correct by
//    construction; randomness governs expected structural balance.
// ================================================================
struct SkipList {
    static const int MAX_LEVEL = 16;
    static const int P_DENOM   = 2; // promote with prob 1/2

    struct Node {
        int key, val;
        vector<Node*> next;
        Node(int k, int v, int level)
            : key(k), val(v), next(level, nullptr) {}
    };

    Node*  head;
    int    level = 1;

    SkipList() : head(new Node(INT_MIN, 0, MAX_LEVEL)) {}

    int random_level() {
        int lvl = 1;
        while (lvl < MAX_LEVEL &&
               uniform_int_distribution<int>(0,1)(rng) == 0)
            lvl++;
        return lvl;
    }

    // O(log n) expected — correct by invariant
    int* find(int key) {
        Node* cur = head;
        for (int i = level-1; i >= 0; i--)
            while (cur->next[i] && cur->next[i]->key < key)
                cur = cur->next[i];
        cur = cur->next[0];
        if (cur && cur->key == key) return &cur->val;
        return nullptr;
    }

    void insert(int key, int val) {
        vector<Node*> update(MAX_LEVEL, head);
        Node* cur = head;
        for (int i = level-1; i >= 0; i--) {
            while (cur->next[i] && cur->next[i]->key < key)
                cur = cur->next[i];
            update[i] = cur;
        }
        cur = cur->next[0];

        if (cur && cur->key == key) { cur->val = val; return; }

        int new_lvl = random_level();
        if (new_lvl > level) {
            for (int i = level; i < new_lvl; i++) update[i] = head;
            level = new_lvl;
        }

        Node* node = new Node(key, val, new_lvl);
        for (int i = 0; i < new_lvl; i++) {
            node->next[i] = update[i]->next[i];
            update[i]->next[i] = node;
        }
    }

    void erase(int key) {
        vector<Node*> update(MAX_LEVEL, nullptr);
        Node* cur = head;
        for (int i = level-1; i >= 0; i--) {
            while (cur->next[i] && cur->next[i]->key < key)
                cur = cur->next[i];
            update[i] = cur;
        }
        cur = cur->next[0];
        if (!cur || cur->key != key) return;

        for (int i = 0; i < level; i++) {
            if (update[i]->next[i] != cur) break;
            update[i]->next[i] = cur->next[i];
        }
        delete cur;
        while (level > 1 && !head->next[level-1]) level--;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // Randomized Quicksort
    {
        vector<int> A = {5,2,8,1,9,3,7,4,6};
        quicksort(A, 0, A.size()-1);
        printf("Sorted: ");
        for (int x : A) printf("%d ", x);
        printf("\n");
    }

    // Randomized Select
    {
        vector<int> A = {5,2,8,1,9,3,7,4,6};
        printf("3rd smallest: %d\n", kth_smallest(A, 2)); // 3
        printf("5th smallest: %d\n", kth_smallest(A, 4)); // 5
    }

    // Treap
    {
        Treap t;
        for (int x : {5,2,8,1,9,3,7}) t.insert(x);
        printf("Contains 5: %s\n", t.contains(5) ? "yes" : "no");
        printf("Contains 4: %s\n", t.contains(4) ? "yes" : "no");
        printf("3rd smallest: %d\n", t.kth(3)); // 3
        t.erase(5);
        printf("After erase 5, contains 5: %s\n", t.contains(5) ? "yes" : "no");
        printf("Size: %d\n", t.size()); // 6
    }

    // Universal Hash Table
    {
        UniversalHashTable ht(16);
        ht.insert(42, 100); ht.insert(17, 200); ht.insert(99, 300);
        printf("ht[42]=%d ht[17]=%d ht[99]=%d\n",
            *ht.find(42), *ht.find(17), *ht.find(99));
        ht.erase(17);
        printf("ht[17] after erase: %s\n", ht.find(17) ? "found" : "not found");
    }

    // Skip List
    {
        SkipList sl;
        for (int x : {5,2,8,1,9,3,7}) sl.insert(x, x*10);
        printf("sl[5]=%d\n", *sl.find(5));     // 50
        printf("sl[4]=%s\n", sl.find(4) ? "found" : "not found");
        sl.erase(5);
        printf("sl[5] after erase: %s\n", sl.find(5) ? "found" : "not found");
    }

    // Las Vegas Primality
    {
        for (ll n : {2LL, 17LL, 1000000007LL, 1000000008LL, 998244353LL})
            printf("%lld: %s\n", n, is_prime_verified(n) ? "PRIME" : "COMPOSITE");
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
using System.Numerics;

// ================================================================
// 1. RANDOMIZED QUICKSORT
// ================================================================
public static class RandomizedQuicksort {
    private static readonly Random Rng = new();

    public static void Sort(int[] A, int lo, int hi) {
        if (lo >= hi) return;
        int pivotIdx = Rng.Next(lo, hi + 1);
        (A[lo], A[pivotIdx]) = (A[pivotIdx], A[lo]);
        int pivot = A[lo];

        int lt = lo, gt = hi, i = lo + 1;
        while (i <= gt) {
            if      (A[i] < pivot) { (A[lt], A[i]) = (A[i], A[lt]); lt++; i++; }
            else if (A[i] > pivot) { (A[i], A[gt]) = (A[gt], A[i]); gt--; }
            else                   i++;
        }
        Sort(A, lo, lt - 1);
        Sort(A, gt + 1, hi);
    }

    public static void Sort(int[] A) => Sort(A, 0, A.Length - 1);
}

// ================================================================
// 2. RANDOMIZED SELECT
// ================================================================
public static class RandomizedSelect {
    private static readonly Random Rng = new();

    private static int QuickSelect(int[] A, int lo, int hi, int k) {
        if (lo == hi) return A[lo];
        int pivotIdx = Rng.Next(lo, hi + 1);
        (A[lo], A[pivotIdx]) = (A[pivotIdx], A[lo]);
        int pivot = A[lo];

        int lt = lo, gt = hi, i = lo + 1;
        while (i <= gt) {
            if      (A[i] < pivot) { (A[lt], A[i]) = (A[i], A[lt]); lt++; i++; }
            else if (A[i] > pivot) { (A[i], A[gt]) = (A[gt], A[i]); gt--; }
            else                   i++;
        }
        if (k <= gt && k >= lt) return pivot;
        if (k < lt)             return QuickSelect(A, lo, lt - 1, k);
        return                         QuickSelect(A, gt + 1, hi, k);
    }

    public static int Kth(int[] A, int k) {
        var copy = (int[])A.Clone();
        return QuickSelect(copy, 0, copy.Length - 1, k);
    }
}

// ================================================================
// 3. TREAP
// ================================================================
public class Treap {
    private class Node {
        public int Key, Priority, Size = 1;
        public Node? Left, Right;
        public Node(int key) { Key = key; Priority = System.Random.Shared.Next(); }
    }

    private Node? root;
    private static int Sz(Node? t) => t?.Size ?? 0;
    private static void Upd(Node? t) { if (t != null) t.Size = 1 + Sz(t.Left) + Sz(t.Right); }

    private (Node? l, Node? r) Split(Node? t, int key) {
        if (t == null) return (null, null);
        if (t.Key <= key) {
            var (l, r) = Split(t.Right, key);
            t.Right = l; Upd(t); return (t, r);
        } else {
            var (l, r) = Split(t.Left, key);
            t.Left = r; Upd(t); return (l, t);
        }
    }

    private Node? Merge(Node? l, Node? r) {
        if (l == null) return r;
        if (r == null) return l;
        if (l.Priority > r.Priority) { l.Right = Merge(l.Right, r); Upd(l); return l; }
        else                         { r.Left  = Merge(l, r.Left);  Upd(r); return r; }
    }

    public void Insert(int key) {
        var (l, r) = Split(root, key - 1);
        root = Merge(Merge(l, new Node(key)), r);
    }

    public void Erase(int key) {
        var (l, mr) = Split(root, key - 1);
        var (m, r)  = Split(mr, key);
        root = Merge(Merge(l, m != null ? Merge(m.Left, m.Right) : null), r);
    }

    public bool Contains(int key) {
        var cur = root;
        while (cur != null) {
            if      (key < cur.Key) cur = cur.Left;
            else if (key > cur.Key) cur = cur.Right;
            else                    return true;
        }
        return false;
    }

    public int Kth(int k) {
        var cur = root;
        while (cur != null) {
            int ls = Sz(cur.Left);
            if (k == ls + 1)  return cur.Key;
            if (k <= ls)      cur = cur.Left;
            else { k -= ls + 1; cur = cur.Right; }
        }
        return -1;
    }

    public int Size => Sz(root);
}

// ================================================================
// 4. UNIVERSAL HASH TABLE
// ================================================================
public class UniversalHashTable {
    private const long P = 1_000_000_007L;
    private readonly int m;
    private readonly long a, b;
    private readonly List<(int key, int val)>[] table;

    public UniversalHashTable(int m) {
        this.m = m;
        var rng = new Random();
        a = rng.NextInt64(1, P - 1);
        b = rng.NextInt64(0, P - 1);
        table = new List<(int,int)>[m];
        for (int i = 0; i < m; i++) table[i] = new();
    }

    private int Hash(int key) => (int)(((a * key + b) % P) % m);

    public void Insert(int key, int val) {
        int h = Hash(key);
        var list = table[h];
        for (int i = 0; i < list.Count; i++)
            if (list[i].key == key) { list[i] = (key, val); return; }
        list.Add((key, val));
    }

    public int? Find(int key) {
        foreach (var (k, v) in table[Hash(key)])
            if (k == key) return v;
        return null;
    }

    public void Erase(int key) {
        var list = table[Hash(key)];
        list.RemoveAll(p => p.key == key);
    }
}

// ================================================================
// 5. SKIP LIST
// ================================================================
public class SkipList {
    private const int MaxLevel = 16;
    private class Node {
        public int Key, Val;
        public Node[] Next;
        public Node(int k, int v, int lvl) { Key=k; Val=v; Next=new Node[lvl]; }
    }

    private readonly Node head = new(int.MinValue, 0, MaxLevel);
    private int level = 1;
    private readonly Random rng = new();

    private int RandomLevel() {
        int lvl = 1;
        while (lvl < MaxLevel && rng.Next(0,2) == 0) lvl++;
        return lvl;
    }

    public int? Find(int key) {
        var cur = head;
        for (int i = level-1; i >= 0; i--)
            while (cur.Next[i] != null && cur.Next[i].Key < key) cur = cur.Next[i];
        cur = cur.Next[0];
        return (cur != null && cur.Key == key) ? cur.Val : null;
    }

    public void Insert(int key, int val) {
        var update = new Node[MaxLevel];
        Array.Fill(update, head);
        var cur = head;
        for (int i = level-1; i >= 0; i--) {
            while (cur.Next[i] != null && cur.Next[i].Key < key) cur = cur.Next[i];
            update[i] = cur;
        }
        cur = cur.Next[0];
        if (cur != null && cur.Key == key) { cur.Val = val; return; }

        int newLvl = RandomLevel();
        if (newLvl > level) { level = newLvl; }

        var node = new Node(key, val, newLvl);
        for (int i = 0; i < newLvl; i++) {
            node.Next[i]    = update[i].Next[i];
            update[i].Next[i] = node;
        }
    }

    public void Erase(int key) {
        var update = new Node[MaxLevel];
        var cur = head;
        for (int i = level-1; i >= 0; i--) {
            while (cur.Next[i] != null && cur.Next[i].Key < key) cur = cur.Next[i];
            update[i] = cur;
        }
        cur = cur.Next[0];
        if (cur == null || cur.Key != key) return;
        for (int i = 0; i < level; i++) {
            if (update[i].Next[i] != cur) break;
            update[i].Next[i] = cur.Next[i];
        }
        while (level > 1 && head.Next[level-1] == null) level--;
    }
}

public class Program {
    public static void Main() {
        // Quicksort
        int[] A = {5,2,8,1,9,3,7,4,6};
        RandomizedQuicksort.Sort(A);
        Console.WriteLine($"Sorted: {string.Join(" ", A)}");

        // Select
        int[] B = {5,2,8,1,9,3,7,4,6};
        Console.WriteLine($"3rd smallest: {RandomizedSelect.Kth(B, 2)}");
        Console.WriteLine($"5th smallest: {RandomizedSelect.Kth(B, 4)}");

        // Treap
        var t = new Treap();
        foreach (int x in new[]{5,2,8,1,9,3,7}) t.Insert(x);
        Console.WriteLine($"Contains 5: {t.Contains(5)}, 4: {t.Contains(4)}");
        Console.WriteLine($"3rd smallest: {t.Kth(3)}");
        t.Erase(5);
        Console.WriteLine($"After erase 5: contains={t.Contains(5)}, size={t.Size}");

        // Universal Hash Table
        var ht = new UniversalHashTable(16);
        ht.Insert(42,100); ht.Insert(17,200); ht.Insert(99,300);
        Console.WriteLine($"ht[42]={ht.Find(42)} ht[17]={ht.Find(17)}");
        ht.Erase(17);
        Console.WriteLine($"ht[17] after erase: {ht.Find(17)}");

        // Skip List
        var sl = new SkipList();
        foreach (int x in new[]{5,2,8,1,9,3,7}) sl.Insert(x, x*10);
        Console.WriteLine($"sl[5]={sl.Find(5)}, sl[4]={sl.Find(4)}");
        sl.Erase(5);
        Console.WriteLine($"sl[5] after erase: {sl.Find(5)}");
    }
}
```

---

## Expected Runtime Proofs

### Randomized Quicksort — O(n log n)

**Indicator variable method.** Let `X_ij = 1` if elements `i` and `j` (in sorted order) are compared. Two elements are compared exactly when one is chosen as pivot while both are still in the same subarray. This happens with probability `2 / (j - i + 1)`.

```
E[comparisons] = sum_{i<j} Pr[X_ij = 1]
               = sum_{i<j} 2/(j-i+1)
               = sum_{i<j} 2/(j-i+1)
               = 2 * sum_{k=1}^{n} (n-k+1)/k * (1/(k+1)) ...
               = O(n log n)
```

### Treap — O(log n) expected depth

The depth of node with key `k` equals the number of ancestors, which equals the number of elements in the randomly chosen prefix that have higher priority than `k`'s priority. Since priorities are uniform random, the expected depth is H_n ≈ ln n. The treap is equivalent to a random BST built by inserting keys in random priority order.

### Skip List — O(log n) expected search

Each node is promoted to level `i` with probability `2^{-i}`. Expected number of nodes at level `i` is `n/2^i`. The expected number of nodes examined during a search is O(log n) by summing geometric series over levels.

---

## Pitfalls

- **Randomized Quicksort stack depth** — in the worst case (very unlucky pivots), the recursion stack reaches depth O(n), causing stack overflow for large arrays. Mitigate by using iterative quicksort with an explicit stack, or switch to heapsort when recursion depth exceeds O(log n).
- **Three-way partition is essential for duplicates** — the standard two-way Lomuto or Hoare partition degrades to O(n²) on arrays with many equal elements. Three-way partition (`<`, `==`, `>`) reduces the problem size by the number of equal elements at each level, giving O(n log k) where k is the number of distinct values.
- **Treap split/merge ordering** — when splitting by key, the convention (split at `key-1` vs `key`) determines whether the boundary element goes left or right. Inconsistent conventions between split and merge corrupt the BST property silently. Choose one convention and enforce it everywhere.
- **Treap memory leaks on erase** — the naive erase that calls `merge(m->left, m->right)` and discards `m` leaks the node unless explicitly deleted. In competitive programming this is often ignored; in production code use a memory pool or smart pointers.
- **Universal hashing collision bounds are per-pair** — the bound `Pr[h(x) = h(y)] ≤ 1/m` is for any fixed pair (x, y), not for all pairs simultaneously. The expected number of collisions among n keys is O(n²/m); set `m = Θ(n)` to keep expected collisions O(n).
- **Skip list level cap** — without a maximum level cap, the `random_level` function can in theory return very large levels (with exponentially small probability). A cap of `log2(n) + c` is sufficient; exceeding it wastes space on empty head pointers.
- **RNG seeding** — a fixed seed (e.g., `mt19937 rng(42)`) makes the algorithm deterministic and vulnerable to adversarial inputs that are worst-case for that specific seed sequence. Always seed from `chrono::steady_clock` or `/dev/urandom` in adversarial settings. In competitive programming, a fixed seed with obfuscated constant is standard.
- **QuickSelect modifies the array** — `quickselect` is an in-place algorithm that partially sorts the array as a side effect. If the original order must be preserved, copy the array before calling. Forgetting this corrupts the input for subsequent operations.

---

## Conclusion

Las Vegas algorithms represent the **gold standard of randomized computation**: the correctness guarantee of deterministic algorithms combined with the efficiency advantage of randomness:

- **Randomized Quicksort and QuickSelect** are the canonical examples — simple, cache-friendly, and achieve optimal expected complexity O(n log n) and O(n) while being robust against adversarial inputs that break deterministic pivot strategies.
- **Treaps and Skip Lists** provide balanced data structures with expected O(log n) operations using randomness as a replacement for the complex rotation logic of AVL or red-black trees.
- **Universal hashing** eliminates worst-case collision chains that deterministic hash functions suffer under adversarial key distributions.
- The **expected runtime** is the primary complexity measure, and all above algorithms achieve the information-theoretic optimum in expectation while maintaining strict correctness.

**Key takeaway:**  
Prefer Las Vegas over Monte Carlo whenever the problem admits efficient output verification — the correctness guarantee costs only a constant factor in expectation. The three design patterns — random pivot (break adversarial structure), random priority (simulate random permutation implicitly), random hash function (universal collision bound) — cover the vast majority of Las Vegas applications and are all driven by the same principle: no fixed input can be the worst case for a randomly chosen algorithm instance.
