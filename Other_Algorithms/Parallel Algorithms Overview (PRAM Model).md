# Parallel Algorithms Overview (PRAM Model)

## Origin & Motivation

The **PRAM model** (Parallel Random Access Machine) was introduced by Fortune and Wyllie in 1978 as the standard theoretical model for analyzing parallel algorithms. It consists of an unbounded number of processors operating in lockstep, all sharing a single global memory, communicating implicitly through reads and writes to that memory.

**The problem it solves:** Sequential algorithm analysis uses a single cost metric (running time = number of operations). Parallel algorithms need **two** metrics: total operations (**work**) and the length of the longest dependency chain (**span**, also called depth). The PRAM model and the associated **work-span model** make these explicit, allowing precise statements like "this algorithm has O(n) work and O(log n) span" — which translates directly to "O(n/p + log n) time on p processors" via Brent's scheduling theorem.

**The key idea:** By idealizing away communication cost (assuming uniform-cost shared memory access), the PRAM model isolates the fundamental question — how much can an algorithm's dependency chain be shortened by adding processors? — from hardware-specific concerns. Real parallel hardware (GPUs, multicore CPUs) approximates PRAM behavior closely enough that PRAM-optimal algorithms (parallel scan, reduction, list ranking) are the basis of real-world parallel libraries.

---

## Where It Is Used

- GPU programming: CUDA/OpenCL prefix sum, reduction, and sort primitives are direct PRAM algorithm implementations
- Parallel libraries: Intel TBB, C++ parallel STL, .NET TPL implement fork-join reduction and scan
- Database query engines: parallel aggregation (SUM, MAX, GROUP BY) uses parallel reduction
- Compilers: auto-vectorization and auto-parallelization rely on work-span analysis
- Graph algorithms: parallel BFS, connected components, and list ranking are PRAM-derived
- Theoretical CS: NC complexity class (problems solvable in polylog time with polynomial processors) is defined via PRAM

---

## Core Concepts

### PRAM Variants

PRAM models differ in how concurrent memory accesses are handled:

```
EREW (Exclusive Read, Exclusive Write):
    No two processors may read or write the same memory cell simultaneously.
    Weakest model — closest to real shared-memory hardware.

CREW (Concurrent Read, Exclusive Write):
    Multiple processors may read the same cell simultaneously,
    but writes must be exclusive. Models read-only shared data well.

CRCW (Concurrent Read, Concurrent Write):
    Both reads and writes may be concurrent. Requires a conflict-resolution
    rule for simultaneous writes to the same cell:
      - COMMON:    all concurrent writers must write the SAME value
      - ARBITRARY: an arbitrary writer succeeds (adversarial — algorithm
                    must be correct regardless of which one)
      - PRIORITY:  the writer with lowest processor ID succeeds

Strength ordering: EREW ⊆ CREW ⊆ CRCW(common) ⊆ CRCW(arbitrary) ⊆ CRCW(priority)
A CRCW algorithm can be simulated on CREW with an O(log p) slowdown,
and on EREW with an additional O(log p) slowdown (Brent's simulation).
```

### Work-Span Model

```
Work  W(n) = total number of operations across all processors
             (= sequential running time if run on 1 processor)

Span  S(n) = length of the longest chain of sequentially-dependent
             operations (= running time with infinitely many processors)

Brent's Theorem:
    T_p(n) = O(W(n)/p + S(n))
    
    Running time on p processors is bounded by work divided among
    processors, plus the unavoidable span (critical path) length.
```

**Why both matter:** An algorithm with `W=O(n)`, `S=O(log n)` ("work-efficient") achieves near-linear speedup up to `p ≈ n/log n` processors. An algorithm with `W=O(n log n)`, `S=O(log n)` does more total work but may still be useful if `p` is large enough that `W/p` is small.

### Parallel Prefix Sum (Scan)

The scan operation `scan([a₀,a₁,...,a_{n-1}]) = [0, a₀, a₀+a₁, ..., a₀+...+a_{n-2}]` (exclusive) is the canonical example of a problem that looks inherently sequential (each output depends on all previous inputs) but admits an `O(n)` work, `O(log n)` span parallel algorithm — the **Blelloch scan**.

```
Up-sweep (reduce phase):
    for d = 0 to log₂(n)-1:
        for i = 0 to n-1 step 2^(d+1)  (in parallel):
            a[i + 2^(d+1) - 1] += a[i + 2^d - 1]
    // After up-sweep, a[n-1] holds the total sum;
    // a forms an implicit binary tree of partial sums.

Down-sweep (distribute phase):
    a[n-1] = 0  // identity element, for EXCLUSIVE scan
    for d = log₂(n)-1 down to 0:
        for i = 0 to n-1 step 2^(d+1)  (in parallel):
            t = a[i + 2^d - 1]
            a[i + 2^d - 1] = a[i + 2^(d+1) - 1]
            a[i + 2^(d+1) - 1] += t
```

Each phase does `n/2 + n/4 + ... + 1 = n-1` total operations across `log₂ n` rounds → `O(n)` work, `O(log n)` span.

### List Ranking via Pointer Jumping

Given a linked list (represented as `next[i]`), compute the distance from each node to the end of the list. Sequentially this is `O(n)` but inherently a chain of dependencies (span `O(n)`). **Pointer jumping** restructures it to span `O(log n)`:

```
for all i (in parallel):
    dist[i] = (next[i] == -1) ? 0 : 1

repeat log₂(n) times:
    for all i (in parallel):
        if next[i] != -1 and next[next[i]] != -1:
            dist[i] += dist[next[i]]
            next[i]  = next[next[i]]   // "jump" — skip ahead

// dist[i] now holds the distance from i to the tail
```

Each round doubles the "jump distance" each pointer can see — after `log₂ n` rounds every node has accumulated the full distance. Work: `O(n)` per round × `O(log n)` rounds = `O(n log n)`. Span: `O(log n)`.

---

## CRCW Example: O(1) Maximum-Finding

The CRCW (common) model permits an extremely fast — but processor-hungry — maximum-finding algorithm:

```
Allocate n² processors, indexed by pairs (i,j).
flag[i] = 1 for all i  (initially everyone is a candidate)

In ONE parallel step:
    Processor (i,j): if a[i] < a[j], write 0 to flag[i]
    
    (Multiple processors may write 0 to the same flag[i] —
     this is conflict-free under CRCW-common since they all
     write the SAME value.)

The unique i with flag[i] == 1 (assuming distinct values) is argmax.
```

This runs in **O(1) time** with **O(n²) processors** — illustrating that CRCW's power comes at the cost of processor count. The work-span model would describe this as `W=O(n²)`, `S=O(1)`.

---

## Complexity Summary

| Algorithm | Work | Span | PRAM Variant |
|---|---|---|---|
| Parallel reduction (sum/max/etc) | O(n) | O(log n) | EREW |
| Parallel prefix sum (Blelloch scan) | O(n) | O(log n) | EREW |
| List ranking (pointer jumping) | O(n log n) | O(log n) | EREW |
| Maximum finding (CRCW) | O(n²) | O(1) | CRCW-common |
| Maximum finding (work-efficient) | O(n) | O(log n) | EREW |
| Merge two sorted arrays | O(n) | O(log n) | EREW |
| Parallel merge sort | O(n log n) | O(log² n) | EREW |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
#include <thread>
using namespace std;
using ll = long long;

// ================================================================
// PARALLEL FOR — splits [0,n) into chunks across num_threads threads.
// Building block for the algorithms below: each "for all i in parallel"
// step in the pseudocode maps to one parallel_for call.
// ================================================================
void parallel_for(int n, function<void(int)> body, int num_threads=4) {
    if (n <= 0) return;
    num_threads = min(num_threads, max(1, n));
    vector<thread> threads;
    int chunk = (n + num_threads - 1) / num_threads;
    for (int t = 0; t < num_threads; t++) {
        int lo = t*chunk, hi = min(n, lo+chunk);
        if (lo >= hi) continue;
        threads.emplace_back([&body, lo, hi]() {
            for (int i = lo; i < hi; i++) body(i);
        });
    }
    for (auto& th : threads) th.join();
}

// ================================================================
// PARALLEL REDUCTION — Work O(n), Span O(log n)
// Fork-join tree: split range, recurse on each half (one half in
// a new thread), combine results. depth bounds total thread count
// (2^max_depth threads spawned in the worst case).
// ================================================================
template<typename T, typename Op>
T parallel_reduce(const vector<T>& a, int lo, int hi, Op op,
                  int depth = 0, int max_depth = 4) {
    int n = hi - lo;
    if (n == 1) return a[lo];
    if (depth >= max_depth || n < 256) {
        T res = a[lo];
        for (int i = lo+1; i < hi; i++) res = op(res, a[i]);
        return res;
    }
    int mid = lo + n/2;
    T left, right;
    thread t([&]() { left = parallel_reduce(a, lo, mid, op, depth+1, max_depth); });
    right = parallel_reduce(a, mid, hi, op, depth+1, max_depth);
    t.join();
    return op(left, right);
}

// ================================================================
// PARALLEL PREFIX SUM — Blelloch work-efficient scan
// Work O(n), Span O(log n). n must be a power of 2 (pad with 0s).
// In-place EXCLUSIVE scan: a[i] becomes sum(original[0..i-1]).
// Returns the total sum.
// ================================================================
ll parallel_scan(vector<ll>& a, int num_threads = 4) {
    int n = (int)a.size();
    int levels = __lg(n); // n = 2^levels

    // Up-sweep: build a binary tree of partial sums in place.
    // After this loop, a[n-1] holds the total sum of all elements.
    for (int d = 0; d < levels; d++) {
        int stride = 1 << (d+1), half = 1 << d;
        int count = n / stride;
        parallel_for(count, [&](int i) {
            int idx = i*stride + stride - 1;
            a[idx] += a[idx - half];
        }, num_threads);
    }

    ll total = a[n-1];
    a[n-1] = 0; // identity element for exclusive scan

    // Down-sweep: propagate prefix sums from the root back to the leaves.
    for (int d = levels-1; d >= 0; d--) {
        int stride = 1 << (d+1), half = 1 << d;
        int count = n / stride;
        parallel_for(count, [&](int i) {
            int idx = i*stride + stride - 1;
            ll t = a[idx-half];
            a[idx-half] = a[idx];
            a[idx]     += t;
        }, num_threads);
    }
    return total;
}

// ================================================================
// LIST RANKING via POINTER JUMPING — Work O(n log n), Span O(log n)
// Input:  next[i] = index of successor node, -1 for the tail.
// Output: dist[i] = number of hops from i to the tail.
// Double buffering avoids read/write races within each round.
// ================================================================
vector<ll> list_ranking(vector<int> nxt, int num_threads = 4) {
    int n = (int)nxt.size();
    vector<ll> dist(n);
    for (int i = 0; i < n; i++) dist[i] = (nxt[i] == -1) ? 0 : 1;

    vector<int> nxt_buf = nxt;
    vector<ll>  dist_buf = dist;

    bool changed = true;
    while (changed) {
        changed = false;
        parallel_for(n, [&](int i) {
            int ni = nxt[i];
            if (ni != -1 && nxt[ni] != -1) {
                dist_buf[i] = dist[i] + dist[ni]; // combine distances
                nxt_buf[i]  = nxt[ni];            // jump ahead
            } else {
                dist_buf[i] = dist[i];
                nxt_buf[i]  = nxt[i];
            }
        }, num_threads);

        for (int i = 0; i < n; i++) if (nxt_buf[i] != nxt[i]) changed = true;
        nxt = nxt_buf; dist = dist_buf;
    }
    return dist;
}

// ================================================================
// CRCW MAX-FINDING — illustrative O(1)-time, O(n^2)-work algorithm
// Processor (i,j) writes 0 to flag[i] if a[i] < a[j]. Concurrent
// writes of the SAME value 0 are conflict-free under CRCW-common.
// Simulated sequentially here (real PRAM: one parallel step).
// ================================================================
int crcw_max_index(const vector<int>& a) {
    int n = (int)a.size();
    vector<int> flag(n, 1);
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            if (a[i] < a[j]) flag[i] = 0;
    for (int i = 0; i < n; i++) if (flag[i]) return i;
    return -1;
}

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- Parallel Reduction ----
    {
        printf("=== Parallel Reduction (sum) ===\n");
        vector<int> a = {3,1,4,1,5,9,2,6,5,3,5,8,9,7,9,3};
        int sum = parallel_reduce(a, 0, (int)a.size(), [](int x,int y){return x+y;});
        printf("sum=%d (expect %d)\n", sum, accumulate(a.begin(),a.end(),0));

        srand(42); int errors=0;
        for (int t=0; t<300; t++) {
            int n=1+rand()%2000;
            vector<ll> v(n);
            for (auto& x:v) x=rand()%1000;
            ll got_sum=parallel_reduce(v,0,n,[](ll x,ll y){return x+y;});
            ll exp_sum=accumulate(v.begin(),v.end(),0LL);
            ll got_max=parallel_reduce(v,0,n,[](ll x,ll y){return max(x,y);});
            ll exp_max=*max_element(v.begin(),v.end());
            if (got_sum!=exp_sum || got_max!=exp_max) errors++;
        }
        printf("Stress 300 trials (sum+max): %s\n",
               errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Parallel Prefix Sum ----
    {
        printf("\n=== Parallel Prefix Sum (Blelloch) ===\n");
        vector<ll> a = {3,1,7,0,4,1,6,3}; // n=8=2^3
        ll total = parallel_scan(a);
        printf("exclusive scan: ");
        for (ll x:a) printf("%lld ",x);
        printf(" total=%lld\n", total);
        printf("expected:       0 3 4 11 11 15 16 22  total=25\n");

        srand(99); int errors=0;
        for (int t=0; t<500; t++) {
            int logn=1+rand()%9; // n=2..512
            int n=1<<logn;
            vector<ll> orig(n);
            for (auto& x:orig) x=rand()%100;
            vector<ll> b=orig;
            ll total=parallel_scan(b);

            ll running=0;
            for (int i=0;i<n;i++) { if (b[i]!=running) errors++; running+=orig[i]; }
            if (total != accumulate(orig.begin(),orig.end(),0LL)) errors++;
        }
        printf("Stress 500 trials: %s\n",
               errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- List Ranking ----
    {
        printf("\n=== List Ranking (pointer jumping) ===\n");
        vector<int> nxt = {1,2,3,4,-1}; // chain 0->1->2->3->4
        auto dist = list_ranking(nxt);
        printf("dist: "); for (ll d:dist) printf("%lld ",d);
        printf(" (expect 4 3 2 1 0)\n");

        srand(7); int errors=0;
        for (int t=0; t<500; t++) {
            int n=1+rand()%200;
            vector<int> perm(n); for(int i=0;i<n;i++) perm[i]=i;
            shuffle(perm.begin(),perm.end(),mt19937(t));
            vector<int> nxt(n,-1);
            for (int i=0;i+1<n;i++) nxt[perm[i]]=perm[i+1];

            auto dist=list_ranking(nxt);
            vector<ll> expected(n);
            for (int i=0;i<n;i++) expected[perm[i]]=n-1-i;
            if (dist!=expected) errors++;
        }
        printf("Stress 500 trials: %s\n",
               errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- CRCW Max-Finding ----
    {
        printf("\n=== CRCW Max-Finding (O(1) time, O(n^2) work) ===\n");
        vector<int> a = {3,1,4,1,5,9,2,6};
        int idx = crcw_max_index(a);
        printf("argmax=%d value=%d (expect value 9)\n", idx, a[idx]);

        srand(13); int errors=0;
        for (int t=0; t<300; t++) {
            int n=1+rand()%50;
            vector<int> v(n);
            for (auto& x:v) x=rand()%100;
            int got=crcw_max_index(v);
            int exp=(int)(max_element(v.begin(),v.end())-v.begin());
            if (v[got]!=v[exp]) errors++;
        }
        printf("Stress 300 trials: %s\n",
               errors==0?"OK":("FAIL "+to_string(errors)).c_str());
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
using System.Threading.Tasks;

public static class ParallelAlgorithms
{
    // ================================================================
    // PARALLEL FOR — wraps System.Threading.Tasks.Parallel.For
    // ================================================================
    public static void ParallelFor(int n, Action<int> body, int numThreads=4)
    {
        var options = new ParallelOptions { MaxDegreeOfParallelism = numThreads };
        Parallel.For(0, n, options, body);
    }

    // ================================================================
    // PARALLEL REDUCTION — Work O(n), Span O(log n)
    // Fork-join via Task.Run; sequential cutoff bounds task count.
    // ================================================================
    public static T ParallelReduce<T>(T[] a, int lo, int hi, Func<T,T,T> op,
                                       int depth=0, int maxDepth=4)
    {
        int n = hi - lo;
        if (n == 1) return a[lo];
        if (depth >= maxDepth || n < 256)
        {
            T res = a[lo];
            for (int i=lo+1;i<hi;i++) res = op(res, a[i]);
            return res;
        }
        int mid = lo + n/2;
        var task = Task.Run(() => ParallelReduce(a, lo, mid, op, depth+1, maxDepth));
        T right = ParallelReduce(a, mid, hi, op, depth+1, maxDepth);
        T left = task.Result;
        return op(left, right);
    }

    // ================================================================
    // PARALLEL PREFIX SUM — Blelloch scan
    // Work O(n), Span O(log n). n must be a power of 2.
    // In-place EXCLUSIVE scan. Returns total sum.
    // ================================================================
    public static long ParallelScan(long[] a, int numThreads=4)
    {
        int n = a.Length;
        int levels = (int)Math.Log2(n);

        // Up-sweep
        for (int d=0; d<levels; d++)
        {
            int stride=1<<(d+1), half=1<<d;
            int count=n/stride;
            ParallelFor(count, i => {
                int idx=i*stride+stride-1;
                a[idx]+=a[idx-half];
            }, numThreads);
        }

        long total=a[n-1];
        a[n-1]=0;

        // Down-sweep
        for (int d=levels-1; d>=0; d--)
        {
            int stride=1<<(d+1), half=1<<d;
            int count=n/stride;
            ParallelFor(count, i => {
                int idx=i*stride+stride-1;
                long t=a[idx-half];
                a[idx-half]=a[idx];
                a[idx]+=t;
            }, numThreads);
        }
        return total;
    }

    // ================================================================
    // LIST RANKING via POINTER JUMPING — Work O(n log n), Span O(log n)
    // ================================================================
    public static long[] ListRanking(int[] nextArr, int numThreads=4)
    {
        int n = nextArr.Length;
        var nxt = (int[])nextArr.Clone();
        var dist = new long[n];
        for (int i=0;i<n;i++) dist[i] = (nxt[i]==-1) ? 0 : 1;

        var nxtBuf = (int[])nxt.Clone();
        var distBuf = (long[])dist.Clone();

        bool changed = true;
        while (changed)
        {
            changed = false;
            ParallelFor(n, i => {
                int ni = nxt[i];
                if (ni != -1 && nxt[ni] != -1) {
                    distBuf[i] = dist[i] + dist[ni];
                    nxtBuf[i]  = nxt[ni];
                } else {
                    distBuf[i] = dist[i];
                    nxtBuf[i]  = nxt[i];
                }
            }, numThreads);

            for (int i=0;i<n;i++) if (nxtBuf[i]!=nxt[i]) changed=true;
            Array.Copy(nxtBuf, nxt, n);
            Array.Copy(distBuf, dist, n);
        }
        return dist;
    }

    // ================================================================
    // CRCW MAX-FINDING — illustrative O(1)-time, O(n^2)-work
    // ================================================================
    public static int CrcwMaxIndex(int[] a)
    {
        int n = a.Length;
        var flag = new int[n];
        Array.Fill(flag, 1);
        for (int i=0;i<n;i++)
            for (int j=0;j<n;j++)
                if (a[i]<a[j]) flag[i]=0;
        for (int i=0;i<n;i++) if (flag[i]==1) return i;
        return -1;
    }

    public static void Main()
    {
        // Reduction
        var a = new int[]{3,1,4,1,5,9,2,6,5,3,5,8,9,7,9,3};
        int sum = ParallelReduce(a, 0, a.Length, (x,y)=>x+y);
        Console.WriteLine($"sum={sum} (expect {a.Sum()})");

        // Prefix sum
        var b = new long[]{3,1,7,0,4,1,6,3};
        long total = ParallelScan(b);
        Console.WriteLine($"scan=[{string.Join(",",b)}] total={total}");
        Console.WriteLine("expected=[0,3,4,11,11,15,16,22] total=25");

        // List ranking
        var nxt = new int[]{1,2,3,4,-1};
        var dist = ListRanking(nxt);
        Console.WriteLine($"dist=[{string.Join(",",dist)}] (expect 4,3,2,1,0)");

        // CRCW max
        var c = new int[]{3,1,4,1,5,9,2,6};
        int idx = CrcwMaxIndex(c);
        Console.WriteLine($"argmax={idx} value={c[idx]} (expect value 9)");
    }
}
```

---

## Why Scan Looks Sequential But Isn't

The exclusive scan `out[i] = a[0] + a[1] + ... + a[i-1]` appears to require `out[i-1]` before computing `out[i]` — an `O(n)` dependency chain. The Blelloch algorithm breaks this by reformulating scan as two tree traversals:

```
Up-sweep builds a balanced binary tree where each internal node
holds the sum of its subtree's leaves — exactly a parallel
REDUCTION, which is O(log n) span.

Down-sweep traverses the SAME tree top-down, at each node passing
"the sum of everything to my left" to its left child unchanged,
and "my left child's value plus what I received" to its right child.
This is also O(log n) span — each level depends only on the level above.

Total: two O(log n)-span tree traversals = O(log n) span overall,
despite the output having an apparent O(n) sequential dependency.
```

This pattern — reformulate an apparently-sequential recurrence as operations on a balanced tree — is the core technique behind most O(log n)-span parallel algorithms (parallel sorting networks, parallel parsing, segment trees evaluated in parallel).

---

## Pitfalls

- **Span is not "time with 2 processors"** — span `S(n)` is the time with *infinitely many* processors (the longest dependency chain). With `p` finite processors, Brent's theorem gives `T_p = O(W/p + S)`. For `p < W/S`, the `W/p` term dominates; for `p > W/S`, the `S` term dominates and adding processors stops helping.
- **Blelloch scan requires n to be a power of 2** — the up-sweep/down-sweep index arithmetic (`stride = 2^(d+1)`) assumes exact powers of 2. For arbitrary `n`, pad the array with the identity element (0 for sum) up to the next power of 2.
- **List ranking termination: check `next[i]` changes, not `dist[i]`** — the while loop must terminate based on whether any pointer still has a "next-next" to jump to (`next[i] != next_buf[i]`), not based on `dist` values (which can legitimately stay equal across rounds for nodes already at the tail).
- **CRCW "common" requires identical concurrent writes** — the max-finding algorithm's correctness depends on all concurrent writers to `flag[i]` writing the same value (0). If the algorithm instead had writers write *different* values to the same cell, CRCW-common would be undefined — you'd need CRCW-arbitrary or CRCW-priority, which are strictly more powerful (and harder to simulate on real hardware).
- **Double buffering is mandatory for pointer jumping** — updating `next[i]` and `dist[i]` in place during a round causes some processors to read already-updated values from other processors in the same round, breaking the "all processors see round k's values" PRAM semantics. Always read from the previous round's buffer and write to a new buffer.
- **Real hardware ≠ idealized PRAM** — PRAM assumes uniform-cost memory access for all processors. Real multicore CPUs have cache hierarchies (NUMA effects, false sharing) and GPUs have warp-level SIMD constraints. A PRAM-optimal algorithm (optimal `W` and `S`) is a *necessary* but not *sufficient* condition for good real-world performance — memory access patterns matter enormously in practice.
- **Thread creation overhead dominates for small n** — the `parallel_reduce` sequential cutoff (`n < 256`) exists because spawning a thread costs far more than summing 256 integers. Always profile and tune the cutoff; for very small inputs, sequential execution is faster than any parallel scheme.

---

## Conclusion

The PRAM model and its work-span analysis provide the **theoretical foundation for all practical parallel algorithm design**:

- Work `W(n)` and span `S(n)` together determine achievable speedup via Brent's theorem `T_p = O(W/p + S)` — an algorithm must minimize both to scale well.
- Parallel prefix sum (Blelloch scan) is the canonical example of restructuring an apparently sequential recurrence into two `O(log n)`-span tree traversals, achieving `O(n)` work — "work-efficient" parallelism.
- List ranking via pointer jumping shows the same pattern applied to linked structures: each round doubles the effective "reach" of each pointer, converting an `O(n)`-span sequential traversal into `O(log n)` span at the cost of `O(n log n)` total work.
- PRAM variants (EREW/CREW/CRCW) form a strict hierarchy of memory-access power; CRCW algorithms can achieve `O(1)` span but at the cost of polynomial processor counts, illustrating the work-span tradeoff in its starkest form.

**Key takeaway:** real parallel speedup comes from reducing *span*, not just adding processors — and the standard technique for reducing span is to reformulate sequential dependency chains as balanced-tree computations, where each level of the tree can be processed in parallel and the tree has only `O(log n)` levels.
