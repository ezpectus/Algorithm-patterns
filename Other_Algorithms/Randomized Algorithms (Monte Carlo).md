# Randomized Algorithms (Monte Carlo) — Probabilistic with Bounded Error

## Origin & Motivation

Monte Carlo algorithms are randomized algorithms that always run in **bounded polynomial time** but may return an **incorrect answer with bounded probability**. They contrast with Las Vegas algorithms (always correct, random runtime) and deterministic algorithms (always correct, deterministic runtime).

The name comes from the Monte Carlo casino — randomness is the core computational resource. The key design principle: if the probability of error is bounded by `δ`, then repeating the algorithm `k` times and taking a majority vote or AND/OR of results reduces error to `δ^k` exponentially, while only multiplying runtime by `k`.

Monte Carlo techniques underpin some of the most efficient known algorithms for primality testing, polynomial identity testing, approximate counting, hashing, and streaming computation — many of which have no known efficient deterministic counterparts.

**One-sided vs two-sided error:**
- **One-sided:** If the answer is YES, the algorithm always returns YES. If the answer is NO, it returns YES with probability ≤ δ (false positive). Or symmetrically: never false negatives. These are stronger.
- **Two-sided:** The algorithm may err in either direction with probability ≤ δ.

---

## Where It Is Used

- Primality testing (Miller-Rabin: one-sided error)
- Polynomial identity testing (Schwartz-Zippel lemma)
- Approximate counting (MCMC, streaming)
- Hashing and fingerprinting (Rabin-Karp, randomized hashing)
- Random sampling (reservoir sampling, randomized quicksort)
- Graph algorithms (minimum cut, random spanning trees)
- Numerical integration (Monte Carlo integration)
- Approximate nearest neighbors, LSH

---

## Core Concepts

### Error Amplification

If a Monte Carlo algorithm has error probability `δ < 1/2`, run it `k` times independently:

```
Two-sided error: majority vote → error <= 2 * (2δ(1-δ))^(k/2)  (Chernoff)
One-sided error: AND of k runs → error <= δ^k
```

For `δ = 1/3`, running `k = 60` times reduces error to `(1/3)^60 ≈ 10^{-29}`.

### Markov and Chebyshev Inequalities

**Markov:** For non-negative X with mean μ: `Pr[X >= t] <= μ/t`

**Chebyshev:** For X with mean μ, variance σ²: `Pr[|X - μ| >= t] <= σ²/t²`

**Chernoff bound:** For sum of independent {0,1} RVs with mean μ:
```
Pr[X >= (1+δ)μ] <= exp(-μδ²/3)     for 0 < δ <= 1
Pr[X <= (1-δ)μ] <= exp(-μδ²/2)
```

### Schwartz-Zippel Lemma

For a nonzero multivariate polynomial `P(x_1,...,x_k)` of total degree `d` over a field F, and `r_1,...,r_k` chosen uniformly at random from a finite set S ⊆ F:

```
Pr[P(r_1,...,r_k) = 0] <= d / |S|
```

This gives a Monte Carlo test for polynomial identity: evaluate at a random point; if zero, output "identical" (may be wrong with prob ≤ d/|S|); if nonzero, output "not identical" (always correct).

---

## Key Algorithms

| Algorithm | Problem | Error type | Error prob | Time |
|---|---|---|---|---|
| Miller-Rabin | Primality | One-sided (false prime) | ≤ 4^{-k} | O(k log²n) |
| Freivalds | Matrix multiplication verify | Two-sided | ≤ 2^{-k} | O(kn²) |
| Rabin-Karp | String matching | One-sided (false match) | ≤ n/|F| | O(n+m) |
| Schwartz-Zippel | Polynomial identity | Two-sided | ≤ d/|S| | O(poly) |
| Min-Cut (Karger) | Global minimum cut | Two-sided | ≤ 1 - 1/n² | O(n²) |
| Reservoir sampling | Uniform sample stream | None (exact) | 0 | O(n) |
| Monte Carlo integration | Numerical integral | Two-sided | O(1/sqrt(N)) | O(N) |

---

## Complexity Classes

| Class | Definition |
|---|---|
| BPP | Problems solvable by poly-time two-sided Monte Carlo with error ≤ 1/3 |
| RP | Problems solvable by poly-time one-sided Monte Carlo (no false negatives) |
| co-RP | Complement of RP (no false positives) |
| ZPP | RP ∩ co-RP = Las Vegas algorithms |
| PP | Like BPP but error can be just under 1/2 (not amplifiable) |

Containments: ZPP ⊆ RP ⊆ BPP, and most researchers believe BPP = P (derandomization conjecture).

---

## Complexity Analysis

| Algorithm | Time | Error after k rounds |
|---|---|---|
| Miller-Rabin (k rounds) | O(k log²n) | ≤ 4^{-k} |
| Freivalds (k rounds) | O(kn²) | ≤ 2^{-k} |
| Rabin-Karp | O(n + m) | ≤ m/p (p = prime modulus) |
| Karger min-cut (n² rounds) | O(n⁴) | O(1/n²) → nearly certain |
| Monte Carlo integration | O(N) samples | O(1/sqrt(N)) std dev |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll  = long long;
using ull = unsigned long long;
using u128 = __uint128_t;

// ================================================================
// 1. MILLER-RABIN PRIMALITY TEST
//    One-sided error: composite always detected (no false negatives),
//    prime declared composite with prob <= 4^(-k).
//    With deterministic witness sets, exact for n < 3.2*10^18.
// ================================================================

ll mulmod(ll a, ll b, ll m) {
    return (__int128)a * b % m;
}

ll powmod(ll a, ll b, ll m) {
    ll res = 1;
    a %= m;
    while (b > 0) {
        if (b & 1) res = mulmod(res, a, m);
        a = mulmod(a, a, m);
        b >>= 1;
    }
    return res;
}

bool miller_rabin_witness(ll n, ll a) {
    if (n % a == 0) return n == a;
    ll d = n - 1;
    int r = 0;
    while (d % 2 == 0) { d /= 2; r++; }

    ll x = powmod(a, d, n);
    if (x == 1 || x == n - 1) return true;
    for (int i = 0; i < r - 1; i++) {
        x = mulmod(x, x, n);
        if (x == n - 1) return true;
    }
    return false;
}

// Deterministic for n < 3,215,031,751 with {2,3,5,7}
// Deterministic for n < 3.2*10^18 with the full set below
bool is_prime(ll n) {
    if (n < 2)  return false;
    if (n == 2 || n == 3 || n == 5 || n == 7) return true;
    if (n % 2 == 0 || n % 3 == 0) return false;
    // Deterministic witness set for all n < 3.2 * 10^18
    for (ll a : {2LL, 3LL, 5LL, 7LL, 11LL, 13LL, 17LL, 19LL, 23LL, 29LL, 31LL, 37LL})
        if (!miller_rabin_witness(n, a)) return false;
    return true;
}

// Randomized version: k rounds, error <= 4^(-k)
bool miller_rabin_random(ll n, int k = 20) {
    if (n < 2) return false;
    if (n == 2 || n == 3) return true;
    if (n % 2 == 0) return false;

    mt19937_64 rng(chrono::steady_clock::now().time_since_epoch().count());
    uniform_int_distribution<ll> dist(2, n - 2);

    for (int i = 0; i < k; i++)
        if (!miller_rabin_witness(n, dist(rng))) return false;
    return true;
}

// ================================================================
// 2. FREIVALDS' ALGORITHM — Matrix Multiplication Verification
//    Given A, B, C (n x n matrices), verify A*B == C.
//    Two-sided error <= 1/2 per round; k rounds gives <= 2^{-k}.
//    Time: O(k * n^2) vs O(n^3) for naive multiplication.
// ================================================================
bool freivalds(
    const vector<vector<ll>>& A,
    const vector<vector<ll>>& B,
    const vector<vector<ll>>& C,
    int k = 20)
{
    int n = A.size();
    mt19937 rng(42);
    uniform_int_distribution<int> coin(0, 1);

    for (int iter = 0; iter < k; iter++) {
        // Random {0,1} vector r
        vector<ll> r(n);
        for (int i = 0; i < n; i++) r[i] = coin(rng);

        // Compute Br
        vector<ll> Br(n, 0);
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                Br[i] += B[i][j] * r[j];

        // Compute A(Br)
        vector<ll> ABr(n, 0);
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                ABr[i] += A[i][j] * Br[j];

        // Compute Cr
        vector<ll> Cr(n, 0);
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                Cr[i] += C[i][j] * r[j];

        // Check ABr == Cr
        if (ABr != Cr) return false; // definitely not equal
    }
    return true; // equal with probability >= 1 - 2^{-k}
}

// ================================================================
// 3. RABIN-KARP STRING MATCHING — Rolling hash with random prime
//    One-sided: false match prob <= n/p per occurrence.
//    Expected O(n + m) time.
// ================================================================
vector<int> rabin_karp(const string& text, const string& pattern) {
    int n = text.size(), m = pattern.size();
    if (m > n) return {};

    // Random large prime modulus to reduce collision probability
    // Use two hashes for extra safety
    const ll MOD1 = 1e9 + 7, BASE1 = 131;
    const ll MOD2 = 1e9 + 9, BASE2 = 137;

    auto rollhash = [&](const string& s, ll base, ll mod) -> vector<ll> {
        int len = s.size();
        vector<ll> h(len + 1, 0), pw(len + 1, 1);
        for (int i = 0; i < len; i++) {
            h[i+1] = (h[i] * base + s[i]) % mod;
            pw[i+1] = pw[i] * base % mod;
        }
        return h; // prefix hashes; use pw for range queries
    };

    // Compute all prefix hashes
    auto h1 = rollhash(text, BASE1, MOD1);
    auto h2 = rollhash(text, BASE2, MOD2);
    auto ph = rollhash(pattern, BASE1, MOD1);
    auto ph2= rollhash(pattern, BASE2, MOD2);

    // Precompute powers
    ll pw1 = 1, pw2 = 1;
    for (int i = 0; i < m; i++) { pw1 = pw1 * BASE1 % MOD1; pw2 = pw2 * BASE2 % MOD2; }

    ll pat_hash1 = ph[m], pat_hash2 = ph2[m];

    vector<int> matches;
    for (int i = 0; i + m <= n; i++) {
        ll win1 = (h1[i+m] - h1[i] * pw1 % MOD1 + MOD1) % MOD1;
        ll win2 = (h2[i+m] - h2[i] * pw2 % MOD2 + MOD2) % MOD2;
        if (win1 == pat_hash1 && win2 == pat_hash2) {
            // Verify to eliminate false positives (optional, makes it Las Vegas)
            if (text.substr(i, m) == pattern) matches.push_back(i);
        }
    }
    return matches;
}

// ================================================================
// 4. KARGER'S MINIMUM CUT ALGORITHM
//    Randomly contract edges until 2 vertices remain.
//    Single run: Pr[correct] >= 2/n(n-1).
//    Run n^2 * ln(n) times: Pr[failure] <= 1/n.
// ================================================================
struct KargerMinCut {
    struct Edge { int u, v; };

    int n;
    vector<Edge> edges;
    mt19937 rng;

    KargerMinCut(int n) : n(n), rng(42) {}
    void add_edge(int u, int v) { edges.push_back({u, v}); }

    // Single contraction run: returns cut size
    int contract() {
        // DSU for contraction
        vector<int> parent(n), sz(n, 1);
        iota(parent.begin(), parent.end(), 0);

        function<int(int)> find = [&](int x) {
            return parent[x] == x ? x : parent[x] = find(parent[x]);
        };

        auto unite = [&](int a, int b) {
            a = find(a); b = find(b);
            if (a == b) return;
            if (sz[a] < sz[b]) swap(a, b);
            parent[b] = a; sz[a] += sz[b];
        };

        int components = n;
        vector<Edge> E = edges; // copy edges

        while (components > 2) {
            // Pick random edge
            int idx = uniform_int_distribution<int>(0, E.size()-1)(rng);
            int u = find(E[idx].u), v = find(E[idx].v);
            if (u != v) { unite(u, v); components--; }
            // Remove self-loops lazily: keep edge, skip if same component
        }

        // Count crossing edges
        int cut = 0;
        for (auto& e : E)
            if (find(e.u) != find(e.v)) cut++;
        return cut;
    }

    // Run multiple times, return minimum found
    int min_cut(int iterations = -1) {
        if (iterations < 0) iterations = (int)(n * (ll)n * log(n + 1)) + 1;
        iterations = min(iterations, 200); // cap for demo
        int best = INT_MAX;
        for (int i = 0; i < iterations; i++)
            best = min(best, contract());
        return best;
    }
};

// ================================================================
// 5. MONTE CARLO INTEGRATION — estimate integral of f on [a,b]^d
//    Std deviation of estimate = sigma(f) / sqrt(N).
//    Convergence rate O(1/sqrt(N)) regardless of dimension.
// ================================================================
double monte_carlo_integrate(
    function<double(vector<double>&)> f,
    vector<double> lo,
    vector<double> hi,
    int N = 1000000)
{
    int d = lo.size();
    double volume = 1.0;
    for (int i = 0; i < d; i++) volume *= hi[i] - lo[i];

    mt19937_64 rng(12345);
    vector<uniform_real_distribution<double>> dist(d);
    for (int i = 0; i < d; i++)
        dist[i] = uniform_real_distribution<double>(lo[i], hi[i]);

    double sum = 0.0, sum2 = 0.0;
    vector<double> point(d);
    for (int i = 0; i < N; i++) {
        for (int j = 0; j < d; j++) point[j] = dist[j](rng);
        double val = f(point);
        sum += val; sum2 += val * val;
    }

    double mean = sum / N;
    double var  = sum2 / N - mean * mean;
    double stderr_ = sqrt(var / N);

    printf("MC integral = %.6f ± %.6f (95%% CI: ±%.6f)\n",
        mean * volume, stderr_ * volume, 1.96 * stderr_ * volume);
    return mean * volume;
}

// ================================================================
// 6. RESERVOIR SAMPLING — uniform sample of size k from stream
//    Each element in final sample with equal prob k/n. Exact.
// ================================================================
vector<int> reservoir_sample(vector<int>& stream, int k) {
    int n = stream.size();
    vector<int> reservoir(stream.begin(), stream.begin() + min(k, n));
    mt19937 rng(42);

    for (int i = k; i < n; i++) {
        int j = uniform_int_distribution<int>(0, i)(rng);
        if (j < k) reservoir[j] = stream[i];
    }
    return reservoir;
}

// ================================================================
// 7. RANDOMIZED POLYNOMIAL IDENTITY TESTING (Schwartz-Zippel)
//    Given two poly expressions, evaluate at random point.
//    Error <= d / |F| where d = total degree.
// ================================================================
// Example: test if (x+y)^2 == x^2 + 2xy + y^2 over Z_p
bool polynomial_identity_test(
    function<ll(ll,ll,ll)> poly1,  // evaluates P(x,y) mod p
    function<ll(ll,ll,ll)> poly2,  // evaluates Q(x,y) mod p
    ll p,                          // prime modulus
    int rounds = 30)
{
    mt19937_64 rng(99);
    uniform_int_distribution<ll> dist(1, p - 1);
    for (int i = 0; i < rounds; i++) {
        ll x = dist(rng), y = dist(rng);
        if (poly1(x, y, p) != poly2(x, y, p)) return false; // definitely not equal
    }
    return true; // equal with prob >= 1 - (d/p)^rounds
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Miller-Rabin
    printf("=== Miller-Rabin ===\n");
    for (ll n : {2LL, 17LL, 100LL, 999983LL, 1000000007LL, 1000000009LL * 1000000007LL})
        printf("%lld: %s\n", n, is_prime(n) ? "PRIME" : "COMPOSITE");

    // Freivalds
    printf("\n=== Freivalds ===\n");
    int n = 3;
    vector<vector<ll>> A = {{1,2,3},{4,5,6},{7,8,9}};
    vector<vector<ll>> B = {{9,8,7},{6,5,4},{3,2,1}};
    // C = A*B (correct)
    vector<vector<ll>> C = {{30,24,18},{84,69,54},{138,114,90}};
    // C_wrong
    vector<vector<ll>> Cw = C; Cw[0][0] = 99;
    printf("A*B == C: %s\n",  freivalds(A,B,C,20)  ? "YES" : "NO");
    printf("A*B == Cw: %s\n", freivalds(A,B,Cw,20) ? "YES" : "NO");

    // Rabin-Karp
    printf("\n=== Rabin-Karp ===\n");
    auto matches = rabin_karp("abracadabra", "abra");
    printf("Pattern 'abra' at positions: ");
    for (int p : matches) printf("%d ", p);
    printf("\n");

    // Karger
    printf("\n=== Karger Min-Cut ===\n");
    KargerMinCut kg(4);
    kg.add_edge(0,1); kg.add_edge(0,2); kg.add_edge(1,2);
    kg.add_edge(1,3); kg.add_edge(2,3);
    printf("Min cut = %d\n", kg.min_cut(100)); // expected 2

    // Monte Carlo integration: integral of x^2 on [0,1] = 1/3
    printf("\n=== Monte Carlo Integration ===\n");
    monte_carlo_integrate(
        [](vector<double>& p) { return p[0]*p[0]; },
        {0.0}, {1.0}, 1000000);

    // Reservoir sampling
    printf("\n=== Reservoir Sampling ===\n");
    vector<int> stream(100);
    iota(stream.begin(), stream.end(), 0);
    auto sample = reservoir_sample(stream, 5);
    printf("Sample of 5 from [0..99]: ");
    for (int x : sample) printf("%d ", x);
    printf("\n");

    // Polynomial identity test: (x+y)^2 == x^2+2xy+y^2 mod p
    printf("\n=== Schwartz-Zippel ===\n");
    ll prime = 1e9 + 7;
    auto lhs = [](ll x, ll y, ll p) { return (x+y)%p*(x+y)%p%p; };
    auto rhs = [](ll x, ll y, ll p) { return (x%p*x%p + 2*x%p*y%p + y%p*y%p) % p; };
    auto rhs_wrong = [](ll x, ll y, ll p) { return (x%p*x%p + y%p*y%p) % p; };
    printf("(x+y)^2 == x^2+2xy+y^2: %s\n", polynomial_identity_test(lhs,rhs,prime) ? "YES":"NO");
    printf("(x+y)^2 == x^2+y^2:     %s\n", polynomial_identity_test(lhs,rhs_wrong,prime) ? "YES":"NO");

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
// 1. MILLER-RABIN PRIMALITY TEST
// ================================================================
public static class MillerRabin {
    private static long MulMod(long a, long b, long m) =>
        (long)((BigInteger)a * b % m);

    private static long PowMod(long a, long b, long m) {
        long res = 1; a %= m;
        while (b > 0) {
            if ((b & 1) == 1) res = MulMod(res, a, m);
            a = MulMod(a, a, m);
            b >>= 1;
        }
        return res;
    }

    private static bool IsWitness(long n, long a) {
        if (n % a == 0) return n == a;
        long d = n - 1; int r = 0;
        while (d % 2 == 0) { d /= 2; r++; }
        long x = PowMod(a, d, n);
        if (x == 1 || x == n - 1) return true;
        for (int i = 0; i < r - 1; i++) {
            x = MulMod(x, x, n);
            if (x == n - 1) return true;
        }
        return false;
    }

    // Deterministic for n < 3.2 * 10^18
    public static bool IsPrime(long n) {
        if (n < 2) return false;
        foreach (long a in new long[]{2,3,5,7,11,13,17,19,23,29,31,37}) {
            if (n == a) return true;
            if (n % a == 0) return false;
        }
        foreach (long a in new long[]{2,3,5,7,11,13,17,19,23,29,31,37})
            if (!IsWitness(n, a)) return false;
        return true;
    }

    // Randomized: k rounds, error <= 4^{-k}
    public static bool IsPrimeRandom(long n, int k = 20) {
        if (n < 2) return false;
        if (n == 2 || n == 3) return true;
        if (n % 2 == 0) return false;
        var rng = new Random();
        for (int i = 0; i < k; i++) {
            long a = (long)(rng.NextDouble() * (n - 3)) + 2;
            if (!IsWitness(n, a)) return false;
        }
        return true;
    }
}

// ================================================================
// 2. FREIVALDS' ALGORITHM
// ================================================================
public static class Freivalds {
    public static bool Verify(long[,] A, long[,] B, long[,] C, int k = 20) {
        int n = A.GetLength(0);
        var rng = new Random(42);
        for (int iter = 0; iter < k; iter++) {
            var r  = Enumerable.Range(0, n).Select(_ => (long)rng.Next(0, 2)).ToArray();
            var Br = new long[n];
            for (int i = 0; i < n; i++)
                for (int j = 0; j < n; j++)
                    Br[i] += B[i, j] * r[j];
            var ABr = new long[n];
            for (int i = 0; i < n; i++)
                for (int j = 0; j < n; j++)
                    ABr[i] += A[i, j] * Br[j];
            var Cr = new long[n];
            for (int i = 0; i < n; i++)
                for (int j = 0; j < n; j++)
                    Cr[i] += C[i, j] * r[j];
            if (!ABr.SequenceEqual(Cr)) return false;
        }
        return true;
    }
}

// ================================================================
// 3. RABIN-KARP STRING MATCHING
// ================================================================
public static class RabinKarp {
    public static List<int> Search(string text, string pattern) {
        int n = text.Length, m = pattern.Length;
        var matches = new List<int>();
        if (m > n) return matches;

        const long MOD1 = 1_000_000_007L, BASE1 = 131;
        const long MOD2 = 1_000_000_009L, BASE2 = 137;

        long[] h1 = new long[n+1], h2 = new long[n+1];
        long pw1 = 1, pw2 = 1;
        for (int i = 0; i < n; i++) {
            h1[i+1] = (h1[i] * BASE1 + text[i])  % MOD1;
            h2[i+1] = (h2[i] * BASE2 + text[i])  % MOD2;
        }
        long ph1 = 0, ph2 = 0;
        for (int i = 0; i < m; i++) {
            ph1 = (ph1 * BASE1 + pattern[i]) % MOD1;
            ph2 = (ph2 * BASE2 + pattern[i]) % MOD2;
            pw1 = pw1 * BASE1 % MOD1;
            pw2 = pw2 * BASE2 % MOD2;
        }
        for (int i = 0; i + m <= n; i++) {
            long w1 = (h1[i+m] - h1[i] * pw1 % MOD1 + MOD1) % MOD1;
            long w2 = (h2[i+m] - h2[i] * pw2 % MOD2 + MOD2) % MOD2;
            if (w1 == ph1 && w2 == ph2)
                if (text.Substring(i, m) == pattern)
                    matches.Add(i);
        }
        return matches;
    }
}

// ================================================================
// 4. MONTE CARLO INTEGRATION
// ================================================================
public static class MCIntegration {
    public static double Integrate(
        Func<double[], double> f,
        double[] lo, double[] hi,
        int N = 1_000_000)
    {
        int d = lo.Length;
        double volume = 1.0;
        for (int i = 0; i < d; i++) volume *= hi[i] - lo[i];

        var rng  = new Random(12345);
        var pt   = new double[d];
        double sum = 0, sum2 = 0;
        for (int i = 0; i < N; i++) {
            for (int j = 0; j < d; j++)
                pt[j] = lo[j] + rng.NextDouble() * (hi[j] - lo[j]);
            double val = f(pt);
            sum += val; sum2 += val * val;
        }
        double mean   = sum / N;
        double var_   = sum2 / N - mean * mean;
        double stderr = Math.Sqrt(var_ / N);
        Console.WriteLine($"MC integral = {mean*volume:F6} ± {1.96*stderr*volume:F6} (95% CI)");
        return mean * volume;
    }
}

// ================================================================
// 5. RESERVOIR SAMPLING
// ================================================================
public static class ReservoirSampling {
    public static List<T> Sample<T>(IEnumerable<T> stream, int k) {
        var reservoir = new List<T>();
        var rng = new Random(42);
        int i = 0;
        foreach (var item in stream) {
            if (i < k) reservoir.Add(item);
            else {
                int j = rng.Next(0, i + 1);
                if (j < k) reservoir[j] = item;
            }
            i++;
        }
        return reservoir;
    }
}

public class Program {
    public static void Main() {
        // Miller-Rabin
        Console.WriteLine("=== Miller-Rabin ===");
        foreach (long n in new long[]{2,17,100,999983,1000000007})
            Console.WriteLine($"{n}: {(MillerRabin.IsPrime(n) ? "PRIME":"COMPOSITE")}");

        // Freivalds
        Console.WriteLine("\n=== Freivalds ===");
        long[,] A = {{1,2},{3,4}}, B = {{5,6},{7,8}};
        long[,] C = {{19,22},{43,50}};   // correct: A*B
        long[,] Cw = {{19,22},{43,99}};  // wrong
        Console.WriteLine($"A*B == C:  {Freivalds.Verify(A,B,C)}");
        Console.WriteLine($"A*B == Cw: {Freivalds.Verify(A,B,Cw)}");

        // Rabin-Karp
        Console.WriteLine("\n=== Rabin-Karp ===");
        var m = RabinKarp.Search("abracadabra", "abra");
        Console.WriteLine($"'abra' at: {string.Join(", ", m)}");

        // Monte Carlo Integration: x^2 on [0,1] = 1/3
        Console.WriteLine("\n=== Monte Carlo Integration ===");
        MCIntegration.Integrate(p => p[0]*p[0], new[]{0.0}, new[]{1.0});

        // Reservoir Sampling
        Console.WriteLine("\n=== Reservoir Sampling ===");
        var sample = ReservoirSampling.Sample(Enumerable.Range(0,100), 5);
        Console.WriteLine($"Sample: {string.Join(" ", sample)}");
    }
}
```

---

## Error Amplification in Practice

```
One-sided error (e.g. Miller-Rabin):
  Single round:  error <= 1/4
  20 rounds:     error <= (1/4)^20 = 9.1 * 10^{-13}
  40 rounds:     error <= (1/4)^40 = 8.3 * 10^{-25}

Two-sided error (e.g. Freivalds):
  Single round:  error <= 1/2
  20 rounds:     error <= (1/2)^20 = 9.5 * 10^{-7}
  60 rounds:     error <= (1/2)^60 = 8.7 * 10^{-19}

Monte Carlo integration (not amplifiable by repetition in same way):
  N = 10^4 samples:  std dev ~ 1/100
  N = 10^6 samples:  std dev ~ 1/1000
  N = 10^8 samples:  std dev ~ 1/10000
  (each 100x increase in samples gives 10x std dev reduction)
```

---

## Pitfalls

- **Using non-cryptographic RNG for security-sensitive applications** — `mt19937` is fast but predictable if the seed is known. For cryptographic uses (primality in key generation, etc.), use a CSPRNG (`/dev/urandom`, `std::random_device`, `RandomNumberGenerator` in C#). Predictable randomness in Miller-Rabin used for RSA key generation is a real vulnerability.
- **Miller-Rabin with fixed witnesses is not randomized** — the deterministic witness sets (e.g., {2,3,5,...,37}) make the test exact for n < 3.2×10^18 with no probability of error. Calling this "Monte Carlo" is a misnomer for the deterministic variant. The randomized variant uses random witnesses and has error ≤ 4^{-k}.
- **Freivalds is over integers, not modular** — the standard analysis assumes exact arithmetic. With large matrices, intermediate products overflow `long`. Either use modular arithmetic throughout (with a random prime modulus) or use Python/BigInteger. Overflow produces wrong zero-sum results that falsely confirm equality.
- **Rabin-Karp with one hash has collision probability n*m/p** — with a single 64-bit hash (mod ~10^18), collision probability per position is ~1/10^18, acceptably small. With a 32-bit hash (mod ~10^9), probability is ~n/10^9 which is significant for n = 10^6. Always use double hashing or verify matches explicitly.
- **Karger's algorithm probability per run is only 2/n(n-1)** — a single run has very low success probability. The required number of runs for constant failure probability is Θ(n² log n), giving total time O(n^4 log n) for the basic version. The optimized version (Karger-Stein) runs in O(n² log^3 n).
- **Monte Carlo integration variance, not bias** — the MC estimate is unbiased (expected value equals the true integral), but has variance σ²/N. For functions with high variance (spiky integrands), N must be very large. Variance reduction techniques (importance sampling, control variates, stratified sampling) can reduce the required N by orders of magnitude.
- **Reservoir sampling requires exactly one pass** — the algorithm is correct only if each element is seen exactly once and the replacement step uses the correct index `j = random(0, i)` inclusive. Using `random(0, i-1)` biases toward earlier elements; using `random(1, i)` biases toward later ones.
- **Schwartz-Zippel requires a field, not just a ring** — the lemma holds over any field. Evaluation mod a composite number does not work because zero divisors exist, invalidating the degree argument. Always use a prime modulus.

---

## Conclusion

Monte Carlo algorithms represent a fundamental trade-off: **bounded time in exchange for bounded error probability**. Their power comes from three principles:

- **Error amplification** — repeating k times reduces error exponentially in k with only linear runtime cost.
- **Algebraic structure** — Schwartz-Zippel, fingerprinting, and hashing exploit the fact that random evaluation distinguishes polynomials and strings with high probability over large fields.
- **Statistical estimation** — Monte Carlo integration and counting achieve dimension-independent O(1/sqrt(N)) convergence unavailable to deterministic grid-based methods.

**Key takeaway:**  
A Monte Carlo algorithm is practical when: (1) the error can be amplified to negligibly small by repetition, (2) the runtime per run is much cheaper than the deterministic alternative, and (3) a one-sided error variant exists so that correct answers are always confirmed. Miller-Rabin and Freivalds are canonical examples where all three conditions hold, making Monte Carlo the method of choice in practice despite the theoretical possibility of error.
