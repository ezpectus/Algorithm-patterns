# Online Algorithms — Competitive Analysis, Ski Rental, Paging

## Origin & Motivation

**Online algorithms** make irrevocable decisions on a sequence of inputs revealed one at a time, without knowledge of future inputs. **Competitive analysis** (Sleator & Tarjan, 1985) is the framework for evaluating them: compare an online algorithm's cost to the cost of the *optimal offline algorithm* `OPT` — one that sees the entire input sequence in advance.

**Definition:** An online algorithm `ALG` is **c-competitive** if for every input sequence `σ`:

```
ALG(σ) <= c * OPT(σ) + a       (a = additive constant, often 0)
```

`c` is the **competitive ratio** — the worst-case multiplicative penalty for not knowing the future. Lower `c` means the online algorithm is closer to clairvoyant.

**The problem it solves:** Many real systems must commit to decisions in real time — caching, scheduling, resource allocation — where the future is unknown. Competitive analysis provides *worst-case guarantees independent of the input distribution*, unlike average-case analysis which requires assumptions about the input.

Two canonical problems illustrate the framework:

**Ski Rental** — the simplest possible online decision problem (rent vs buy), with a clean **2x** deterministic bound and **e/(e-1) ≈ 1.58x** randomized bound.

**Paging (Cache Replacement)** — the canonical online algorithm, with LRU/FIFO achieving the optimal deterministic bound `k` (cache size) and the randomized Marking algorithm achieving `O(log k)`.

---

## Where It Is Used

- Operating systems: page replacement (LRU/FIFO/Clock are paging algorithms)
- CPU caches: cache eviction policies (LRU variants in L1/L2/L3)
- CDN and web caching: which content to evict when cache is full
- Cloud resource provisioning: ski-rental framing for "buy reserved instance vs pay-as-you-go"
- Equipment leasing decisions: literal ski-rental (buy equipment vs rent per use)
- Online scheduling: job scheduling without knowledge of future job arrivals
- Network routing: buffer management, TCP congestion control analysis

---

## Core Concepts: Competitive Analysis

### The Adversary Model

Competitive analysis assumes an **adversary** that knows the online algorithm's strategy (deterministic) or its probability distribution (randomized) and chooses the worst-case input sequence to maximize `ALG(σ)/OPT(σ)`.

```
Deterministic competitive ratio:
    c = sup over σ of  ALG(σ) / OPT(σ)

Randomized competitive ratio (against oblivious adversary):
    c = sup over σ of  E[ALG(σ)] / OPT(σ)

    "Oblivious" = adversary fixes σ before seeing ALG's random choices.
```

Randomization helps because a deterministic algorithm's exact behavior can always be anticipated and exploited by the adversary; a randomized algorithm forces the adversary to commit to `σ` without knowing which specific run will occur.

---

## Ski Rental Problem

### Problem Statement

```
Buying skis costs B. Renting costs 1 per day.
You don't know how many days n you will ski (revealed one day at a time).
Each morning you decide: rent today, or buy (if not already bought)?
Minimize total cost.

OPT(n) = min(n, B)   [knows n in advance: rent if n<B, buy if n>=B]
```

### Deterministic Algorithm: Buy on Day B

```
Strategy: rent days 1..B-1; if still skiing on day B, buy.

If n < B:  cost = n           (never reach buy day)         ratio = 1
If n >= B: cost = (B-1) + B = 2B-1                           ratio = (2B-1)/B = 2 - 1/B
```

**This is optimal among deterministic algorithms** — any strategy that buys on day `B' != B` has worst-case ratio `>= 2 - 1/B'` for the appropriate adversary choice, and `B'=B` minimizes this.

### Randomized Algorithm: e/(e-1)-Competitive

Pick a random "buy day" `T ∈ {1,...,B}` from the distribution:

```
p(T=i) = (1 - 1/B)^(B-i) / Z,   Z = sum_{i=1}^{B} (1-1/B)^(B-i)
```

This distribution concentrates probability on later days (closer to `B`) but with exponentially decreasing weight for earlier buy-days — a discretized exponential distribution. As `B → ∞`, the competitive ratio of this strategy approaches:

```
c -> e / (e-1) ≈ 1.58198
```

This is the **optimal randomized competitive ratio** for ski rental against an oblivious adversary — no randomized algorithm can do better.

---

## Paging (Cache Replacement) Problem

### Problem Statement

```
Cache holds k pages. A sequence of page requests arrives one at a time.
If the requested page is in the cache: HIT (no cost).
If not: FAULT (cost 1) — must evict some page from the cache (if full)
                          and load the requested page.

OPT(σ) = minimum total faults, knowing the entire sequence in advance.
```

### Belady's Algorithm (Offline OPT)

```
On a fault with a full cache, evict the page whose NEXT occurrence
in the remaining sequence is furthest in the future
(or that never occurs again — evict it first).

This is PROVABLY OPTIMAL (Belady, 1966) — minimizes total faults.
```

### LRU and FIFO: k-Competitive (Deterministic)

```
LRU (Least Recently Used): on a fault, evict the page that was
    accessed longest ago.

FIFO (First In First Out): on a fault, evict the page that has
    been in the cache longest (regardless of access pattern).

Both achieve competitive ratio k (cache size) — and this is OPTIMAL:
no deterministic algorithm can do better than k-competitive.
```

**Why k is a lower bound for ANY deterministic algorithm:** the adversary always requests the one page NOT currently in the algorithm's cache (it knows the algorithm's state). Over `k+1` distinct pages cycling, the algorithm faults on every request while `OPT` (with foresight) faults only once per `k+1`-cycle in the limit — giving ratio approaching `k`.

### Randomized Marking Algorithm: O(log k)-Competitive

```
Maintain a "marked" bit per cached page. Initially all unmarked.

On request for page p:
    if p in cache:
        mark(p)
    else (fault):
        if all cached pages are marked:
            unmark all pages          // START NEW PHASE
        evict a uniformly random UNMARKED page
        bring p into cache, mark(p)
```

**Competitive ratio: `2 * H_k`** where `H_k = 1 + 1/2 + ... + 1/k ≈ ln(k)` is the k-th harmonic number. This is exponentially better than the deterministic `k` bound for large `k` (`O(log k)` vs `O(k)`).

**Why marking helps:** a "phase" ends when `k` distinct pages have been requested since the phase began. Within a phase, marked pages are "recently useful" — the algorithm never evicts them. By choosing the eviction target uniformly among unmarked pages, the adversary cannot predict which page will be evicted, spreading the "damage" of any single bad choice across `O(log k)` instead of being concentrated.

---

## Complexity Summary

| Algorithm | Problem | Competitive Ratio | Type |
|---|---|---|---|
| Buy-on-day-B | Ski Rental | 2 - 1/B → 2 | Deterministic, optimal |
| Exponential distribution | Ski Rental | e/(e-1) ≈ 1.582 | Randomized, optimal |
| LRU | Paging | k | Deterministic, optimal |
| FIFO | Paging | k | Deterministic, optimal |
| Belady's algorithm | Paging | 1 (= OPT by definition) | Offline optimal |
| Marking algorithm | Paging | 2·H_k ≈ 2 ln k | Randomized |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// ================================================================
// SKI RENTAL — Deterministic: rent B-1 days, buy on day B
// Worst-case ratio = 2 - 1/B (optimal among deterministic algorithms)
// ================================================================
int ski_deterministic_cost(int B, int n) {
    if (n < B) return n;      // never reach buy day — matches OPT exactly
    return (B-1) + B;          // = 2B-1
}
int ski_opt(int B, int n) { return min(n, B); }

// ================================================================
// SKI RENTAL — Randomized: buy on a random day T drawn from
//   p(T=i) ∝ (1-1/B)^(B-i),  i=1..B
// Competitive ratio -> e/(e-1) as B -> infinity (optimal randomized).
// ================================================================
vector<double> ski_randomized_distribution(int B) {
    vector<double> p(B+1, 0.0);
    double norm = 0;
    for (int i=1;i<=B;i++) { p[i]=pow(1.0-1.0/B, B-i); norm+=p[i]; }
    for (int i=1;i<=B;i++) p[i]/=norm;
    return p;
}

// Expected cost if the adversary chooses n ski days
double ski_randomized_expected_cost(const vector<double>& p, int B, int n) {
    double E=0;
    for (int T=1;T<=B;T++) {
        double cost = (T<=n) ? (T-1+B) : n; // buy on day T if T<=n, else rent all n days
        E += p[T]*cost;
    }
    return E;
}

// Worst-case (over n) competitive ratio for a given B
double ski_randomized_ratio(int B) {
    auto p = ski_randomized_distribution(B);
    double max_ratio=0;
    for (int n=1;n<=3*B;n++) {
        double E=ski_randomized_expected_cost(p,B,n);
        max_ratio=max(max_ratio, E/ski_opt(B,n));
    }
    return max_ratio;
}

// ================================================================
// PAGING — LRU (Least Recently Used)
// k-competitive deterministic algorithm.
// ================================================================
int lru_faults(const vector<int>& seq, int k) {
    list<int> cache; // front = most recently used
    unordered_map<int,list<int>::iterator> pos;
    int faults=0;
    for (int p : seq) {
        if (pos.count(p)) {
            cache.erase(pos[p]);
            cache.push_front(p);
            pos[p]=cache.begin();
        } else {
            faults++;
            if ((int)cache.size()==k) {
                int evict=cache.back();
                cache.pop_back();
                pos.erase(evict);
            }
            cache.push_front(p);
            pos[p]=cache.begin();
        }
    }
    return faults;
}

// ================================================================
// PAGING — FIFO (First In First Out)
// Also k-competitive deterministic.
// ================================================================
int fifo_faults(const vector<int>& seq, int k) {
    queue<int> q;
    unordered_set<int> in_cache;
    int faults=0;
    for (int p : seq) {
        if (!in_cache.count(p)) {
            faults++;
            if ((int)q.size()==k) {
                int evict=q.front(); q.pop();
                in_cache.erase(evict);
            }
            q.push(p);
            in_cache.insert(p);
        }
    }
    return faults;
}

// ================================================================
// PAGING — BELADY'S OPT (offline optimal)
// Evict the cached page whose next use is furthest in the future
// (or never again). Requires the full sequence; O(n log n) via
// pointer-advance over precomputed occurrence lists.
// ================================================================
int belady_faults(const vector<int>& seq, int k) {
    int n=(int)seq.size();
    unordered_map<int,vector<int>> occ;
    for (int i=0;i<n;i++) occ[seq[i]].push_back(i);
    unordered_map<int,int> ptr; // value -> index into occ[value]

    set<int> cache;
    int faults=0;
    for (int i=0;i<n;i++) {
        int p=seq[i];
        if (!cache.count(p)) {
            faults++;
            if ((int)cache.size()==k) {
                int evict=-1; long long farthest=-1;
                for (int c:cache) {
                    auto& ov=occ[c]; int pc=ptr[c];
                    long long nextpos=(pc<(int)ov.size())?ov[pc]:LLONG_MAX;
                    if (nextpos>farthest) { farthest=nextpos; evict=c; }
                }
                cache.erase(evict);
            }
            cache.insert(p);
        }
        ptr[p]++; // advance past current occurrence
    }
    return faults;
}

// ================================================================
// PAGING — RANDOMIZED MARKING ALGORITHM
// O(log k)-competitive (<= 2*H_k, H_k = k-th harmonic number).
// ================================================================
int marking_faults(const vector<int>& seq, int k, mt19937& rng) {
    set<int> cache, marked;
    int faults=0;
    for (int p : seq) {
        if (cache.count(p)) {
            marked.insert(p);
        } else {
            faults++;
            if ((int)cache.size()==k) {
                if (marked.size()==cache.size()) marked.clear(); // new phase
                vector<int> unmarked;
                for (int c:cache) if (!marked.count(c)) unmarked.push_back(c);
                int idx=rng()%unmarked.size();
                cache.erase(unmarked[idx]);
            }
            cache.insert(p);
            marked.insert(p);
        }
    }
    return faults;
}

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- Ski Rental: deterministic ----
    {
        printf("=== Ski Rental: Deterministic (buy on day B) ===\n");
        for (int B : {5,10,20,100}) {
            double max_ratio=0;
            for (int n=1;n<=3*B;n++)
                max_ratio=max(max_ratio,(double)ski_deterministic_cost(B,n)/ski_opt(B,n));
            printf("B=%4d  max_ratio=%.4f  (theory 2-1/B=%.4f)\n",B,max_ratio,2.0-1.0/B);
        }
    }

    // ---- Ski Rental: randomized ----
    {
        printf("\n=== Ski Rental: Randomized ===\n");
        for (int B : {5,10,50,200,1000}) {
            double ratio=ski_randomized_ratio(B);
            printf("B=%5d  max_ratio=%.4f  (theory -> e/(e-1)=%.4f)\n",
                   B,ratio,M_E/(M_E-1));
        }
    }

    // ---- Paging: LRU vs OPT on (k+1)-cycle (adversarial) ----
    {
        printf("\n=== Paging: LRU vs OPT on (k+1)-cycle (adversarial) ===\n");
        for (int k : {2,3,4,5}) {
            vector<int> seq;
            for (int r=0;r<20;r++) for (int p=1;p<=k+1;p++) seq.push_back(p);
            int lru=lru_faults(seq,k), opt=belady_faults(seq,k);
            printf("k=%d  LRU=%3d  OPT=%3d  ratio=%.3f  (theory -> k=%d)\n",
                   k,lru,opt,(double)lru/opt,k);
        }
    }

    // ---- Paging: random sequences ----
    {
        printf("\n=== Paging: random sequences (k=4, universe=10) ===\n");
        srand(42);
        int k=4, universe=10, len=200;
        vector<int> seq(len);
        for (auto& x:seq) x=1+rand()%universe;
        int lru=lru_faults(seq,k), fifo=fifo_faults(seq,k), opt=belady_faults(seq,k);
        printf("LRU=%d  FIFO=%d  OPT=%d\n",lru,fifo,opt);
        printf("LRU/OPT=%.3f  FIFO/OPT=%.3f  (bound: <= k=%d)\n",
               (double)lru/opt,(double)fifo/opt,k);
    }

    // ---- Marking algorithm: empirical competitive ratio ----
    {
        printf("\n=== Paging: Marking algorithm (randomized) ===\n");
        srand(99);
        for (int k : {2,4,8}) {
            int universe=k+1, len=300;
            vector<int> seq(len);
            for (auto& x:seq) x=rand()%universe;
            int opt=belady_faults(seq,k);

            int trials=200; double total_ratio=0;
            for (int t=0;t<trials;t++) {
                mt19937 rng(t);
                total_ratio += (double)marking_faults(seq,k,rng)/opt;
            }
            double Hk=0; for(int i=1;i<=k;i++) Hk+=1.0/i;
            printf("k=%d  OPT=%d  avg(MARK)/OPT=%.3f  (bound 2*H_k=%.3f)\n",
                   k,opt,total_ratio/trials,2*Hk);
        }
    }

    // ---- Stress: LRU/FIFO <= k*OPT bound ----
    {
        printf("\n=== Stress: LRU/FIFO <= k*OPT bound, 300 trials ===\n");
        srand(123); int errors=0;
        for (int t=0;t<300;t++) {
            int k=1+rand()%6;
            int universe=k+1+rand()%8;
            int len=10+rand()%100;
            vector<int> seq(len);
            for (auto& x:seq) x=rand()%universe;
            int lru=lru_faults(seq,k), fifo=fifo_faults(seq,k), opt=belady_faults(seq,k);
            if (lru>k*opt || fifo>k*opt) errors++;
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
using System.Linq;

public static class OnlineAlgorithms
{
    // ================================================================
    // SKI RENTAL — Deterministic
    // ================================================================
    public static int SkiDeterministicCost(int B, int n) => (n < B) ? n : (B-1)+B;
    public static int SkiOpt(int B, int n) => Math.Min(n, B);

    // ================================================================
    // SKI RENTAL — Randomized: p(T=i) ∝ (1-1/B)^(B-i)
    // ================================================================
    public static double[] SkiRandomizedDistribution(int B)
    {
        var p = new double[B+1];
        double norm=0;
        for (int i=1;i<=B;i++) { p[i]=Math.Pow(1.0-1.0/B, B-i); norm+=p[i]; }
        for (int i=1;i<=B;i++) p[i]/=norm;
        return p;
    }

    public static double SkiRandomizedExpectedCost(double[] p, int B, int n)
    {
        double E=0;
        for (int T=1;T<=B;T++)
        {
            double cost = (T<=n) ? (T-1+B) : n;
            E += p[T]*cost;
        }
        return E;
    }

    public static double SkiRandomizedRatio(int B)
    {
        var p=SkiRandomizedDistribution(B);
        double maxRatio=0;
        for (int n=1;n<=3*B;n++)
            maxRatio=Math.Max(maxRatio, SkiRandomizedExpectedCost(p,B,n)/SkiOpt(B,n));
        return maxRatio;
    }

    // ================================================================
    // PAGING — LRU
    // ================================================================
    public static int LruFaults(int[] seq, int k)
    {
        var cache = new LinkedList<int>();
        var pos = new Dictionary<int, LinkedListNode<int>>();
        int faults=0;
        foreach (int p in seq)
        {
            if (pos.TryGetValue(p, out var node))
            {
                cache.Remove(node);
                cache.AddFirst(p);
                pos[p]=cache.First;
            }
            else
            {
                faults++;
                if (cache.Count==k)
                {
                    int evict=cache.Last.Value;
                    cache.RemoveLast();
                    pos.Remove(evict);
                }
                cache.AddFirst(p);
                pos[p]=cache.First;
            }
        }
        return faults;
    }

    // ================================================================
    // PAGING — FIFO
    // ================================================================
    public static int FifoFaults(int[] seq, int k)
    {
        var q = new Queue<int>();
        var inCache = new HashSet<int>();
        int faults=0;
        foreach (int p in seq)
        {
            if (!inCache.Contains(p))
            {
                faults++;
                if (q.Count==k)
                {
                    int evict=q.Dequeue();
                    inCache.Remove(evict);
                }
                q.Enqueue(p);
                inCache.Add(p);
            }
        }
        return faults;
    }

    // ================================================================
    // PAGING — BELADY'S OPT
    // ================================================================
    public static int BeladyFaults(int[] seq, int k)
    {
        int n=seq.Length;
        var occ=new Dictionary<int,List<int>>();
        for(int i=0;i<n;i++){
            if(!occ.ContainsKey(seq[i])) occ[seq[i]]=new List<int>();
            occ[seq[i]].Add(i);
        }
        var ptr=new Dictionary<int,int>();
        var cache=new HashSet<int>();
        int faults=0;
        for(int i=0;i<n;i++){
            int p=seq[i];
            if(!cache.Contains(p)){
                faults++;
                if(cache.Count==k){
                    int evict=-1; long farthest=-1;
                    foreach(int c in cache){
                        int pc=ptr.GetValueOrDefault(c,0);
                        long nextpos = (pc<occ[c].Count) ? occ[c][pc] : long.MaxValue;
                        if(nextpos>farthest){farthest=nextpos;evict=c;}
                    }
                    cache.Remove(evict);
                }
                cache.Add(p);
            }
            ptr[p]=ptr.GetValueOrDefault(p,0)+1;
        }
        return faults;
    }

    // ================================================================
    // PAGING — RANDOMIZED MARKING ALGORITHM
    // ================================================================
    public static int MarkingFaults(int[] seq, int k, Random rng)
    {
        var cache=new HashSet<int>();
        var marked=new HashSet<int>();
        int faults=0;
        foreach(int p in seq){
            if(cache.Contains(p)){
                marked.Add(p);
            } else {
                faults++;
                if(cache.Count==k){
                    if(marked.Count==cache.Count) marked.Clear();
                    var unmarked=cache.Where(c=>!marked.Contains(c)).ToList();
                    int idx=rng.Next(unmarked.Count);
                    cache.Remove(unmarked[idx]);
                }
                cache.Add(p);
                marked.Add(p);
            }
        }
        return faults;
    }

    public static void Main()
    {
        // Ski rental
        Console.WriteLine("=== Ski Rental ===");
        foreach (int B in new[]{10,100}) {
            double maxRatio=0;
            for(int n=1;n<=3*B;n++) maxRatio=Math.Max(maxRatio,(double)SkiDeterministicCost(B,n)/SkiOpt(B,n));
            Console.WriteLine($"Deterministic B={B}: ratio={maxRatio:F4} (theory {2.0-1.0/B:F4})");
            Console.WriteLine($"Randomized    B={B}: ratio={SkiRandomizedRatio(B):F4} (theory e/(e-1)={Math.E/(Math.E-1):F4})");
        }

        // Paging
        Console.WriteLine("\n=== Paging ===");
        var rng=new Random(42);
        int k=4, universe=10, len=200;
        var seq=Enumerable.Range(0,len).Select(_=>1+rng.Next(universe)).ToArray();
        int lru=LruFaults(seq,k), fifo=FifoFaults(seq,k), opt=BeladyFaults(seq,k);
        Console.WriteLine($"LRU={lru} FIFO={fifo} OPT={opt}");
        Console.WriteLine($"LRU/OPT={(double)lru/opt:F3} FIFO/OPT={(double)fifo/opt:F3} (bound <= k={k})");
    }
}
```

---

## Why Marking Achieves 2·H_k — Proof Sketch

Partition the request sequence into **phases**: a phase ends exactly when `k` distinct pages have been requested since the phase began (equivalently, when the marking algorithm would face a fault with all cache pages marked).

**OPT's cost per phase ≥ 1.** Within any phase, `k` distinct pages are requested. Since the cache holds only `k` pages, at least one of these `k` distinct pages was not in OPT's cache at the start of the phase (pigeonhole on the *previous* phase's distinct pages) — so OPT faults at least once per phase. Summing: `OPT(σ) ≥ (number of phases) - 1`.

**Marking's expected cost per phase ≤ H_k.** Within a phase, consider the `j`-th fault on a *new* page (one not requested earlier in this phase). At this point, at most `j-1` pages are marked (from earlier faults this phase) and `k-(j-1)` are unmarked. The page evicted is chosen uniformly from the unmarked set. The probability that this eviction "hurts" — i.e., evicts a page that will be requested later in the phase — is at most `(\text{remaining distinct pages to come}) / (k-j+1) \le 1`, and summing the probabilities of "being unlucky" across all `k` positions in a phase gives the harmonic sum `1 + 1/2 + ... + 1/k = H_k`.

Combining: `E[MARK(σ)] ≤ H_k · (\text{number of phases}) ≤ H_k · (OPT(σ) + 1) \cdot 2` (the factor 2 accounts for boundary effects between phases), giving the `2·H_k` bound.

---

## Pitfalls

- **Ski rental "buy on day B" vs "buy on day B-1"** — off-by-one errors are common. The correct strategy rents on days `1..B-1` and buys on day `B` (total spend if `n>=B` is `(B-1)·1 + B = 2B-1`). Buying "on day B-1" (one day earlier) gives ratio `(2B-2)/B`, which for some `n` can exceed `2-1/B` — always verify against the worst case `n=B` specifically.
- **Randomized ski rental: the distribution must be normalized** — `p[i] ∝ (1-1/B)^{B-i}` must be divided by its sum `Z` to form a valid probability distribution. Skipping normalization gives `E[cost]` values that don't correspond to any real probability distribution, and the computed "ratio" is meaningless.
- **Belady's algorithm: pointer must advance even on hits** — the `ptr[p]++` step at the end of each iteration must execute for BOTH hits and faults. If it's only done on faults, `ptr[p]` becomes stale for pages that were hit, causing `belady_faults` to make wrong eviction choices (evicting a page that's about to be needed, because its "next occurrence" pointer points to an already-passed position).
- **LRU/FIFO `k`-competitive bound includes an additive constant** — the precise statement is `LRU(σ) ≤ k · OPT(σ)`, valid when both start with empty caches (the first `k` faults are "free" misses for both). On sequences shorter than `k`, both `LRU` and `OPT` equal the number of distinct pages, so the ratio is trivially 1 — don't expect to observe the ratio `k` on short sequences; it requires sequences much longer than `k` with adversarial structure.
- **Marking algorithm: "all marked" check must happen BEFORE eviction, on a fault** — the new-phase trigger (`marked.size() == cache.size()`) is checked only when a fault occurs and the cache is full. If checked on every request (including hits), phases would be measured incorrectly and the `2·H_k` bound would not apply.
- **Marking algorithm randomness must be FRESH per eviction, not per phase** — each eviction within a phase independently draws a uniform random unmarked page. Using a single random permutation decided at phase-start (rather than independent draws) changes the algorithm's guarantees and is a common implementation bug.
- **Oblivious vs adaptive adversary** — the `2·H_k` bound for Marking holds against an **oblivious** adversary (fixes the sequence before seeing the algorithm's coin flips). Against an **adaptive** adversary (can react to the algorithm's random choices in real time), the achievable competitive ratio is provably `Ω(k)` — no better than deterministic. Always state which adversary model applies.

---

## Conclusion

Competitive analysis provides **worst-case guarantees for decision-making under uncertainty**, formalized via the ratio `ALG(σ)/OPT(σ)`:

- Ski rental demonstrates the simplest possible online/offline gap: a clean **2x** deterministic penalty (buy on day B), reduced to **e/(e-1) ≈ 1.58x** with randomization (exponential buy-day distribution) — and both bounds are *provably optimal*.
- Paging demonstrates the same pattern at larger scale: LRU/FIFO achieve the optimal deterministic bound `k` (cache size), while the randomized Marking algorithm achieves `O(log k)` — an exponential improvement, again provably optimal against oblivious adversaries.
- Both problems share the structural insight that **randomization defeats an adversary that can anticipate deterministic choices** — the adversary must commit to a worst-case input without knowing which random outcomes will occur, fundamentally limiting how badly it can hurt the algorithm in expectation.

**Key takeaway:** when designing an online algorithm, first find the best deterministic competitive ratio (often achievable by a simple threshold/LRU-style rule), then ask whether randomization can improve it — the technique is almost always to replace a fixed threshold with a carefully-chosen probability distribution (exponential for ski rental, uniform-over-unmarked for paging) that spreads the adversary's potential damage across many possible outcomes.
