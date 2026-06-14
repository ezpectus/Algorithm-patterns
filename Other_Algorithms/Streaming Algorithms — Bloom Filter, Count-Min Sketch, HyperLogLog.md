# Streaming Algorithms — Bloom Filter, Count-Min Sketch, HyperLogLog

## Origin & Motivation

**Streaming algorithms** process a sequence of data in one pass, using memory far smaller than the input size — typically `O(log n)` or `O(1/ε)` instead of `O(n)`. Three foundational structures cover the three classic streaming questions:

**Bloom Filter** (Bloom, 1970) — *"have I seen this item before?"* Answers set-membership queries using `O(n)` bits total (a small constant per item), with no false negatives but a tunable false-positive rate.

**Count-Min Sketch** (Cormode & Muthukrishnan, 2005) — *"how many times have I seen this item?"* Estimates frequency counts using `O((1/ε) log(1/δ))` counters, guaranteeing the estimate is never below the true count and is above it by at most `ε·N` (N = total stream length) with probability `1-δ`.

**HyperLogLog** (Flajolet et al., 2007) — *"how many distinct items have I seen?"* Estimates cardinality using `O(log log n)` bits per bucket and `O(2^b)` buckets, achieving relative error `~1.04/√m` with `m = 2^b` buckets.

**The problem they solve:** Exact answers to these three questions require `O(n)` space (a hash set, a hash map, or storing all distinct items). For streams of billions of events — web traffic, network packets, database query logs — `O(n)` space is infeasible. These structures trade a small, *quantifiable* error probability for sublinear (often constant) space.

---

## Where It Is Used

- Bloom Filter: database systems (avoid disk lookups for non-existent keys — Cassandra, HBase, PostgreSQL), web caches (avoid caching one-hit-wonders), spell checkers, CDN edge caching
- Count-Min Sketch: network traffic analysis (heavy-hitter detection, DDoS detection), database query optimizers (join cardinality estimation), recommendation systems (item popularity)
- HyperLogLog: distinct-visitor counting (Redis `PFCOUNT`, Google BigQuery `APPROX_COUNT_DISTINCT`), database `COUNT(DISTINCT ...)` approximation, log analytics (unique IPs, unique users)
- All three: real-time analytics dashboards, A/B testing infrastructure, anomaly detection pipelines

---

## Core Structure: Bloom Filter

### Structure

```
m bits (initially all 0), k independent hash functions h₁, ..., h_k

Insert(x):
    for i = 1 to k:
        bits[hᵢ(x) mod m] = 1

Query(x):
    for i = 1 to k:
        if bits[hᵢ(x) mod m] == 0:
            return FALSE  // definitely not in set
    return TRUE  // probably in set (may be false positive)
```

### False Positive Rate

After inserting `n` items into `m` bits with `k` hash functions, the probability a bit is still 0 is `(1 - 1/m)^{kn} ≈ e^{-kn/m}`. The false positive rate is:

```
p ≈ (1 - e^{-kn/m})^k

Optimal k (minimizing p for fixed m, n):
    k* = (m/n) · ln(2)

Required m for target false-positive rate p, n items:
    m = -n · ln(p) / (ln 2)²
```

**No false negatives, ever** — if an item was inserted, all `k` of its bits are set, so the query always returns TRUE.

---

## Core Structure: Count-Min Sketch

### Structure

```
d × w table of counters (all 0), d independent hash functions h₁, ..., h_d
each hᵢ maps items to {0, ..., w-1}

Add(x, count=1):
    for i = 1 to d:
        table[i][hᵢ(x) mod w] += count

Estimate(x):
    return min over i of table[i][hᵢ(x) mod w]
```

### Error Bound

Each row independently overestimates the true count by the sum of counts from other items hashing to the same cell — a non-negative quantity, so **the estimate never underestimates**. Taking the minimum across `d` rows reduces the chance that *all* rows have a large collision-induced error:

```
For width w = ⌈e/ε⌉ and depth d = ⌈ln(1/δ)⌉:

    true_count(x) <= estimate(x) <= true_count(x) + ε·N

with probability at least 1-δ, where N = total stream length (sum of all counts).
```

**Why `e/ε` for width?** By Markov's inequality, the expected collision error per row is `N/w`. Setting `w = e/ε` makes `Pr[collision error > ε·N] < 1/e`. Taking the min over `d = ln(1/δ)` independent rows reduces the failure probability to `(1/e)^d = e^{-d} ≤ δ`.

---

## Core Structure: HyperLogLog

### Structure

HyperLogLog is based on the observation that for a random 64-bit hash, the probability of seeing `k` leading zero bits is `2^{-k}` — so the *maximum* number of leading zeros observed across `n` hashes is a noisy estimator of `log₂(n)`. Using many independent estimators (buckets) and averaging reduces the noise.

```
b bits → m = 2^b buckets, each storing a small integer (max ~64-b, fits in 6 bits)

Add(x):
    h = hash(x)                     // 64-bit hash
    j = h & (m-1)                   // low b bits select the bucket
    w = h >> b                      // remaining (64-b) bits
    rho = leading_zeros(w) + 1      // position of first 1-bit (1-indexed)
    M[j] = max(M[j], rho)           // keep the maximum seen per bucket

Estimate():
    Z = sum over j of 2^{-M[j]}
    E = alpha_m * m^2 / Z           // harmonic-mean-based estimator

    alpha_m = 0.7213 / (1 + 1.079/m)   // bias correction constant

    if E <= 2.5*m:                  // small-range correction
        V = count of buckets with M[j] == 0
        if V > 0: E = m * ln(m/V)   // linear counting estimator
```

### Why Harmonic Mean?

Each bucket's `M[j]` is roughly `log₂` of the number of distinct items hashed to that bucket. Averaging `2^{M[j]}` directly is dominated by outliers (buckets that got an unlucky long run of zeros). The harmonic mean — averaging `2^{-M[j]}` and inverting — is robust to such outliers, giving the `1.04/√m` relative error bound.

---

## Complexity Summary

| Structure | Space | Insert | Query | Error Guarantee |
|---|---|---|---|---|
| Bloom Filter | O(n) bits, ~1.44·log₂(1/p) bits/item | O(k) | O(k) | No false negatives; false positive rate p (tunable) |
| Count-Min Sketch | O((1/ε)·log(1/δ)) counters | O(d) | O(d) | Never underestimates; overestimate ≤ ε·N w.p. 1-δ |
| HyperLogLog | O(2^b · log log n) bits | O(1) | O(2^b) for estimate | Relative error ≈ 1.04/√m, m=2^b |

For typical parameters: Bloom filter at 1% FP rate uses ~9.6 bits/item; Count-Min at ε=0.01, δ=0.01 uses 5×272=1360 counters (≈5.4KB at 32 bits); HyperLogLog at b=14 uses 16384 buckets × 6 bits ≈ 12KB, estimating cardinalities up to billions with ~0.8% error.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ull = unsigned long long;

// ================================================================
// HASH FUNCTION FAMILY — splitmix64-style mixing
// hash_fn(x, seed) gives independent-looking hashes for different seeds
// ================================================================
ull hash_fn(ull x, int seed) {
    x += (ull)seed * 0x9E3779B97F4A7C15ULL;
    x ^= x >> 30; x *= 0xbf58476d1ce4e5b9ULL;
    x ^= x >> 27; x *= 0x94d049bb133111ebULL;
    x ^= x >> 31;
    return x;
}

// ================================================================
// BLOOM FILTER
// Set membership with no false negatives, tunable false positive rate.
// ================================================================
struct BloomFilter {
    int m, k;            // m bits, k hash functions
    vector<bool> bits;

    BloomFilter(int m, int k) : m(m), k(k), bits(m, false) {}

    void insert(ull x) {
        for (int i = 0; i < k; i++) bits[hash_fn(x,i) % m] = true;
    }
    bool contains(ull x) {
        for (int i = 0; i < k; i++)
            if (!bits[hash_fn(x,i) % m]) return false;
        return true;
    }

    // Optimal number of hash functions for n items, m bits
    static int optimal_k(int m, int n) {
        return max(1, (int)round((double)m/n * log(2)));
    }
    // Bits needed for n items at target false-positive rate p
    static int bits_for(int n, double p) {
        return (int)ceil(-(double)n * log(p) / (log(2)*log(2)));
    }
};

// ================================================================
// COUNT-MIN SKETCH
// Frequency estimation: never underestimates, bounded overestimate.
// ================================================================
struct CountMinSketch {
    int d, w;             // d rows (hash functions), w columns (width)
    vector<vector<int>> table;

    CountMinSketch(int d, int w) : d(d), w(w), table(d, vector<int>(w,0)) {}

    void add(ull x, int count = 1) {
        for (int i = 0; i < d; i++) table[i][hash_fn(x,i) % w] += count;
    }
    int estimate(ull x) {
        int result = INT_MAX;
        for (int i = 0; i < d; i++) result = min(result, table[i][hash_fn(x,i)%w]);
        return result;
    }

    // Width for additive error epsilon*N
    static int width_for_epsilon(double epsilon) { return (int)ceil(M_E/epsilon); }
    // Depth for confidence 1-delta
    static int depth_for_delta(double delta)     { return (int)ceil(log(1.0/delta)); }
};

// ================================================================
// HYPERLOGLOG
// Cardinality (distinct count) estimation. m=2^b buckets.
// Standard error ≈ 1.04/sqrt(m).
// ================================================================
struct HyperLogLog {
    int b, m;
    vector<int> M;        // M[j] = max leading-zero-run+1 seen in bucket j
    double alpha;

    HyperLogLog(int b) : b(b), m(1<<b), M(m,0) {
        if      (m==16) alpha=0.673;
        else if (m==32) alpha=0.697;
        else if (m==64) alpha=0.709;
        else            alpha=0.7213/(1.0+1.079/m);
    }

    void add(ull x) {
        ull h = hash_fn(x, 0);
        int  j = (int)(h & (ull)(m-1)); // low b bits → bucket index
        ull  w = h >> b;                // remaining (64-b) bits
        int rho = (w==0) ? (64-b+1) : (__builtin_clzll(w) - b + 1);
        M[j] = max(M[j], rho);
    }

    double estimate() const {
        double sum=0;
        for (int j=0;j<m;j++) sum += pow(2.0, -M[j]);
        double E = alpha * m * m / sum;
        if (E <= 2.5*m) { // small-range correction (linear counting)
            int V=0; for (int j=0;j<m;j++) if (M[j]==0) V++;
            if (V>0) E = m * log((double)m/V);
        }
        return E;
    }

    // Merge with another HLL of the same size — estimates |A ∪ B|
    void merge(const HyperLogLog& other) {
        for (int j=0;j<m;j++) M[j]=max(M[j],other.M[j]);
    }
};

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- Bloom Filter ----
    {
        printf("=== Bloom Filter ===\n");
        int n=1000; double target_fp=0.01;
        int m=BloomFilter::bits_for(n,target_fp);
        int k=BloomFilter::optimal_k(m,n);
        printf("n=%d target_fp=%.2f -> m=%d bits, k=%d hashes\n",n,target_fp,m,k);

        BloomFilter bf(m,k);
        srand(42);
        set<ull> inserted;
        for (int i=0;i<n;i++){ull x=rand();inserted.insert(x);bf.insert(x);}

        int false_neg=0;
        for (ull x:inserted) if (!bf.contains(x)) false_neg++;
        printf("False negatives: %d (expect 0)\n",false_neg);

        int trials=20000, false_pos=0;
        for (int i=0;i<trials;i++){
            ull x=(ull)rand()<<32|rand();
            if (inserted.count(x)) continue;
            if (bf.contains(x)) false_pos++;
        }
        printf("Measured FP rate: %.4f  (target %.4f)\n",(double)false_pos/trials,target_fp);
    }

    // ---- Count-Min Sketch ----
    {
        printf("\n=== Count-Min Sketch ===\n");
        double epsilon=0.01, delta=0.01;
        int w=CountMinSketch::width_for_epsilon(epsilon);
        int d=CountMinSketch::depth_for_delta(delta);
        printf("epsilon=%.2f delta=%.2f -> width=%d depth=%d\n",epsilon,delta,w,d);

        CountMinSketch cms(d,w);
        map<ull,int> truth;
        srand(99); int N=0;
        for (int i=0;i<50000;i++){
            ull item; int r=rand()%100;
            if      (r<50) item=1;          // heavy hitter ~50%
            else if (r<70) item=2;          // ~20%
            else           item=3+rand()%50;// long tail
            cms.add(item); truth[item]++; N++;
        }

        printf("Total stream length N=%d\n",N);
        int under=0; double max_err=0;
        for (auto& [item,cnt]:truth){
            int est=cms.estimate(item);
            if (est<cnt) under++;
            max_err=max(max_err,(double)(est-cnt)/N);
        }
        printf("Underestimates: %d (expect 0)\n",under);
        printf("Max relative overestimate: %.4f  (bound epsilon=%.2f)\n",max_err,epsilon);
        printf("Heavy hitter (item=1): true=%d est=%d\n",truth[1],cms.estimate(1));
    }

    // ---- HyperLogLog ----
    {
        printf("\n=== HyperLogLog ===\n");
        for (int b : {8, 10, 12, 14}) {
            HyperLogLog hll(b);
            int n=100000;
            srand(7);
            set<ull> distinct;
            for (int i=0;i<n;i++){
                ull x=(ull)rand()<<32 | rand();
                hll.add(x); distinct.insert(x);
            }
            double est=hll.estimate();
            double err=fabs(est-distinct.size())/distinct.size();
            double std_err=1.04/sqrt(hll.m);
            printf("b=%2d m=%5d  true=%6zu  est=%9.0f  rel_err=%.4f  (std_err~%.4f)\n",
                   b,hll.m,distinct.size(),est,err,std_err);
        }
    }

    // ---- HyperLogLog: merge (union) ----
    {
        printf("\n=== HyperLogLog Merge ===\n");
        HyperLogLog h1(12), h2(12);
        set<ull> s1,s2,uni;
        srand(123);
        for (int i=0;i<30000;i++){ull x=(ull)rand()<<20|rand()%(1<<20); h1.add(x); s1.insert(x); uni.insert(x);}
        for (int i=0;i<30000;i++){ull x=((ull)rand()<<20|rand()%(1<<20))+1000000000ULL; h2.add(x); s2.insert(x); uni.insert(x);}
        h1.merge(h2);
        double est=h1.estimate();
        printf("|A|=%zu |B|=%zu |A union B| true=%zu est=%.0f  rel_err=%.4f\n",
               s1.size(),s2.size(),uni.size(),est,fabs(est-uni.size())/uni.size());
    }

    // ---- Stress: Count-Min never underestimates ----
    {
        printf("\n=== Stress: Count-Min Sketch, 200 streams ===\n");
        srand(55); int errors=0;
        for (int t=0;t<200;t++){
            int d=4, w=50+rand()%50;
            CountMinSketch cms(d,w);
            map<int,int> truth;
            int items=20+rand()%50;
            for (int i=0;i<500;i++){
                int x=rand()%items;
                cms.add(x); truth[x]++;
            }
            for (auto& [x,c]:truth) if (cms.estimate(x)<c) errors++;
        }
        printf("Result: %s\n",errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Stress: Bloom filter no false negatives ----
    {
        printf("\n=== Stress: Bloom Filter no false negatives, 100 configs ===\n");
        srand(77); int errors=0;
        for (int t=0;t<100;t++){
            int n=10+rand()%200;
            int m=BloomFilter::bits_for(n,0.05);
            int k=BloomFilter::optimal_k(m,n);
            BloomFilter bf(m,k);
            vector<ull> items;
            for (int i=0;i<n;i++){ull x=rand();items.push_back(x);bf.insert(x);}
            for (ull x:items) if (!bf.contains(x)) errors++;
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

public static class StreamingAlgorithms
{
    // ================================================================
    // HASH FUNCTION FAMILY — splitmix64-style mixing
    // ================================================================
    public static ulong HashFn(ulong x, int seed)
    {
        x += (ulong)seed * 0x9E3779B97F4A7C15UL;
        x ^= x >> 30; x *= 0xbf58476d1ce4e5b9UL;
        x ^= x >> 27; x *= 0x94d049bb133111ebUL;
        x ^= x >> 31;
        return x;
    }

    // ================================================================
    // BLOOM FILTER
    // ================================================================
    public class BloomFilter
    {
        private readonly int _m, _k;
        private readonly bool[] _bits;

        public BloomFilter(int m, int k) { _m=m; _k=k; _bits=new bool[m]; }

        public void Insert(ulong x)
        {
            for (int i=0;i<_k;i++) _bits[HashFn(x,i) % (ulong)_m] = true;
        }
        public bool Contains(ulong x)
        {
            for (int i=0;i<_k;i++)
                if (!_bits[HashFn(x,i) % (ulong)_m]) return false;
            return true;
        }
        public static int OptimalK(int m,int n) => Math.Max(1,(int)Math.Round((double)m/n*Math.Log(2)));
        public static int BitsFor(int n,double p) => (int)Math.Ceiling(-(double)n*Math.Log(p)/(Math.Log(2)*Math.Log(2)));
    }

    // ================================================================
    // COUNT-MIN SKETCH
    // ================================================================
    public class CountMinSketch
    {
        private readonly int _d,_w;
        private readonly int[][] _table;

        public CountMinSketch(int d,int w)
        {
            _d=d; _w=w;
            _table=new int[d][];
            for(int i=0;i<d;i++) _table[i]=new int[w];
        }

        public void Add(ulong x,int count=1)
        {
            for(int i=0;i<_d;i++) _table[i][HashFn(x,(ulong)i==0?0:i) % (ulong)_w]+=count;
        }
        public int Estimate(ulong x)
        {
            int result=int.MaxValue;
            for(int i=0;i<_d;i++) result=Math.Min(result,_table[i][HashFn(x,i)%(ulong)_w]);
            return result;
        }
        public static int WidthForEpsilon(double eps)=>(int)Math.Ceiling(Math.E/eps);
        public static int DepthForDelta(double delta)=>(int)Math.Ceiling(Math.Log(1.0/delta));
    }

    // ================================================================
    // HYPERLOGLOG
    // ================================================================
    public class HyperLogLog
    {
        private readonly int _b,_m;
        private readonly int[] _M;
        private readonly double _alpha;

        public int M_size => _m;

        public HyperLogLog(int b)
        {
            _b=b; _m=1<<b; _M=new int[_m];
            _alpha = _m switch {
                16 => 0.673,
                32 => 0.697,
                64 => 0.709,
                _  => 0.7213/(1.0+1.079/_m)
            };
        }

        public void Add(ulong x)
        {
            ulong h=HashFn(x,0);
            int j=(int)(h & (ulong)(_m-1));
            ulong w=h >> _b;
            int rho = (w==0) ? (64-_b+1) : (LeadingZeros(w) - _b + 1);
            _M[j]=Math.Max(_M[j],rho);
        }

        private static int LeadingZeros(ulong x)
        {
            if (x==0) return 64;
            int n=0; ulong mask=1UL<<63;
            while((x & mask)==0){n++;mask>>=1;}
            return n;
        }

        public double Estimate()
        {
            double sum=0;
            for(int j=0;j<_m;j++) sum+=Math.Pow(2.0,-_M[j]);
            double E=_alpha*_m*_m/sum;
            if (E<=2.5*_m){
                int V=0; for(int j=0;j<_m;j++) if(_M[j]==0) V++;
                if(V>0) E=_m*Math.Log((double)_m/V);
            }
            return E;
        }

        public void Merge(HyperLogLog other)
        {
            for(int j=0;j<_m;j++) _M[j]=Math.Max(_M[j],other._M[j]);
        }
    }

    public static void Main()
    {
        var rng=new Random(42);

        // Bloom filter
        int n=1000; double targetFp=0.01;
        int m=BloomFilter.BitsFor(n,targetFp);
        int k=BloomFilter.OptimalK(m,n);
        var bf=new BloomFilter(m,k);
        var inserted=new HashSet<ulong>();
        for(int i=0;i<n;i++){ulong x=(ulong)rng.Next();inserted.Add(x);bf.Insert(x);}
        int falseNeg=inserted.Count(x=>!bf.Contains(x));
        Console.WriteLine($"Bloom: m={m} k={k} false_negatives={falseNeg} (expect 0)");

        // Count-Min Sketch
        int w=CountMinSketch.WidthForEpsilon(0.01), d=CountMinSketch.DepthForDelta(0.01);
        var cms=new CountMinSketch(d,w);
        var truth=new Dictionary<ulong,int>();
        for(int i=0;i<50000;i++){
            ulong item = rng.Next(100)<50 ? 1UL : (ulong)(3+rng.Next(50));
            cms.Add(item);
            truth[item]=truth.GetValueOrDefault(item)+1;
        }
        Console.WriteLine($"CMS width={w} depth={d}, heavy hitter true={truth[1]} est={cms.Estimate(1)}");

        // HyperLogLog
        var hll=new HyperLogLog(12);
        var distinct=new HashSet<ulong>();
        for(int i=0;i<100000;i++){
            ulong x=((ulong)(uint)rng.Next()<<32)|(uint)rng.Next();
            hll.Add(x); distinct.Add(x);
        }
        Console.WriteLine($"HLL true={distinct.Count} est={hll.Estimate():F0}");
    }
}
```

---

## Choosing Parameters — Worked Example

```
Scenario: web server logging unique visitor IPs, expecting ~10 million
events per day, want <1% error on distinct-IP count.

HyperLogLog: relative error = 1.04/sqrt(m)
   Target error 1% → sqrt(m) = 104 → m = 10816 → round up to m=2^14=16384 (b=14)
   Memory: 16384 buckets × 6 bits = 98304 bits ≈ 12 KB
   (vs. storing 10M distinct IPs as 4-byte integers ≈ 40 MB — over 3000x smaller)

Bloom Filter: "has this IP been seen today?" with 1% false positive rate
   n=10,000,000, p=0.01
   m = -n*ln(p)/(ln2)^2 ≈ 10,000,000 * 4.605 / 0.4805 ≈ 95,850,000 bits ≈ 12 MB
   k = (m/n)*ln(2) ≈ 6.6 → use k=7

Count-Min Sketch: "how many requests from this IP today?" with ε=0.1%, δ=1%
   w = ceil(e/0.001) = 2719
   d = ceil(ln(100)) = 5
   Memory: 5 × 2719 × 4 bytes ≈ 54 KB
   Guarantee: estimate ≤ true_count + 0.001 * 10,000,000 = true_count + 10,000
```

---

## Pitfalls

- **Hash function independence is assumed but rarely verified** — all three structures' theoretical guarantees assume the `k`/`d` hash functions behave like independent random functions. Using related hash functions (e.g. `h_i(x) = h(x) + i` with small `i`) can cause correlated collisions, degrading accuracy below the theoretical bound. Use well-mixed hash families (splitmix, MurmurHash, xxHash).
- **Bloom filter cannot delete items** — the basic Bloom filter has no remove operation; clearing a bit might cause false negatives for other items sharing that bit. Use a **Counting Bloom Filter** (counters instead of bits) if deletion is needed, at the cost of more memory.
- **Count-Min Sketch with negative counts (Count-Median)** — for streams with deletions (negative counts), the minimum-based estimator can become incorrect (estimates can go negative or be inconsistent). Use the **Count-Median** variant (median instead of min across rows) for streams with both increments and decrements.
- **HyperLogLog `rho` calculation for `w=0`** — when the high `(64-b)` bits of the hash are all zero, `__builtin_clzll(0)` is undefined behavior in C++. The reference implementation explicitly checks `w==0` and uses the maximum possible `rho = 64-b+1`. Forgetting this check causes undefined behavior on a (rare but nonzero-probability) input.
- **HyperLogLog merge requires identical bucket counts `m`** — merging two HLL structures with different `b` (different `m`) is meaningless; `M[j]` indices don't correspond to the same hash ranges. Always use the same `b` across all HLL instances that will be merged.
- **Small-range correction threshold (`2.5m`)** — for cardinalities much smaller than `m`, the harmonic-mean estimator has high relative error because most buckets remain at `M[j]=0`. The linear-counting correction (`E = m·ln(m/V)`) is essential for accurate small-cardinality estimates; omitting it causes large errors for `n << m`.
- **Bloom filter `m` must be chosen for the EXPECTED final size, not current size** — unlike Count-Min and HyperLogLog (which are fixed-size and degrade gracefully), a Bloom filter's false-positive rate increases as more items are inserted beyond the planned `n`. If the stream size is unknown in advance, use a **Scalable Bloom Filter** (a sequence of filters with increasing size) or overprovision `m`.

---

## Conclusion

Bloom Filter, Count-Min Sketch, and HyperLogLog form a **complementary toolkit for sublinear-space stream processing**, each targeting one of the three fundamental streaming questions:

- Bloom Filter answers "is x in the set?" with zero false negatives and a tunable false-positive rate, using `O(n)` bits with a small constant factor (~10 bits/item at 1% FP rate).
- Count-Min Sketch answers "what is the frequency of x?" with a one-sided error guarantee (never underestimates), using `O((1/ε)log(1/δ))` counters independent of the number of distinct items.
- HyperLogLog answers "how many distinct items?" using `O(2^b)` small counters, achieving `1.04/√m` relative error — `12 KB` of memory can estimate cardinalities into the billions with under 1% error.

All three rely on the same core trick: replace exact storage with a small number of independent hash-indexed counters/bits, and use the structure of random hashing (collision probability, leading-zero distribution) to bound the resulting error probabilistically.

**Key takeaway:** these structures don't approximate "a little bit wrong sometimes" — they provide *mathematically proven* error bounds (`p`, `ε·N`, `1.04/√m`) that can be tuned by adjusting memory. Choosing parameters is a direct memory-vs-accuracy dial, computed from the formulas above before the stream even starts.
