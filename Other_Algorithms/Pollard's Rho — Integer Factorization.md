# Pollard's Rho — Integer Factorization

## Origin & Motivation

Pollard's Rho algorithm was introduced by John Pollard in 1975 as a factorization method that dramatically outperforms trial division for large composites. The name comes from the shape of the sequence it generates — when visualized as a path in a graph, it resembles the Greek letter ρ (rho): a tail leading into a cycle.

The algorithm exploits the **birthday paradox**: in a pseudorandom sequence `x_0, x_1, x_2, ...` modulo `n`, a collision `x_i ≡ x_j (mod p)` for some small prime factor `p` is expected after only **O(p^{1/4})** steps (birthday bound in Z/pZ). Since we don't know `p`, we compute `gcd(|x_i - x_j|, n)` — if it's non-trivial, we've found a factor.

Floyd's cycle detection (or Brent's improvement) finds the collision in O(√p) steps without storing the full sequence.

Combined with Miller-Rabin primality testing, Pollard's Rho gives a complete integer factorization algorithm in **O(n^{1/4} polylog n)** expected time — the fastest known general-purpose factorization for numbers up to ~10^{20}.

Complexity: **O(n^{1/4} log n)** expected per factor, **O(n^{1/4} polylog n)** total.

---

## Where It Is Used

- Integer factorization for cryptanalysis
- Competitive programming: factorize numbers up to 10^{18}
- RSA security analysis (factoring n = p*q)
- Number theory computations requiring prime factorization
- Euler's totient, Möbius function on large numbers
- Finding the smallest prime factor of large composites

---

## Core Algorithm

### Pseudorandom Function

Use `f(x) = (x^2 + c) mod n` for a randomly chosen constant `c ≠ 0, 2`. This function generates a pseudorandom sequence that behaves like a random function modulo any prime factor `p` of `n`.

### Floyd's Cycle Detection

Maintain two pointers: `tortoise` (advances one step) and `hare` (advances two steps). Both start at x_0. When they meet, the cycle length in Z/nZ has been traversed. Meanwhile, compute `gcd(|tortoise - hare|, n)` at each step.

```
x = y = random_start
c = random_constant
d = 1

while d == 1:
    x = f(x)          // tortoise: one step
    y = f(f(y))       // hare: two steps
    d = gcd(|x - y|, n)

if d == n: retry with different c
else: d is a non-trivial factor of n
```

### Brent's Improvement

Brent's variant is ~36% faster in practice. Instead of Floyd's tortoise-and-hare, it saves checkpoints and uses product accumulation:

```
y = random_start, c = random_constant
m = 128 (batch size)
g = 1, q = 1, r = 1

while g == 1:
    x = y
    repeat r times: y = f(y)
    k = 0
    while k < r and g == 1:
        ys = y
        repeat min(m, r-k) times:
            y = f(y)
            q = q * |x - y| mod n
        g = gcd(q, n)
        k += m
    r *= 2

if g == n:
    // Backtrack from ys
    while g == 1:
        ys = f(ys)
        g = gcd(|x - ys|, n)

if g == n: retry
else: g is a factor
```

Batching `gcd` computations (multiply `m` differences before taking gcd) reduces the number of costly gcd calls from O(√p) to O(√p / m).

---

## Complete Factorization

Combine Pollard's Rho with Miller-Rabin to recursively factor all prime factors:

```
factorize(n):
    if n == 1: return []
    if is_prime(n): return [n]
    
    p = pollard_rho(n)    // finds one factor (may be composite)
    return factorize(p) + factorize(n/p)
```

---

## Complexity Analysis

| Operation | Expected Time | Notes |
|---|---|---|
| Find factor p of n | O(p^{1/4} log n) | p = smallest prime factor |
| Complete factorization | O(n^{1/4} log^2 n) | repeated application |
| Miller-Rabin primality | O(k log^2 n) | k rounds |
| Trial division (baseline) | O(n^{1/2}) | much slower |

For n = 10^{18} with smallest factor p ≈ 10^{9}: O(p^{1/4}) ≈ O(178) steps — extremely fast.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll  = long long;
using ull = unsigned long long;
using lll = __int128;

// ================================================================
// MILLER-RABIN PRIMALITY TEST
// Deterministic for n < 3.2 * 10^18 with these witnesses.
// ================================================================
ll mulmod(ll a, ll b, ll m) {
    return (lll)a * b % m;
}

ll powmod(ll a, ll b, ll m) {
    ll res = 1; a %= m;
    for (; b; b >>= 1, a = mulmod(a, a, m))
        if (b & 1) res = mulmod(res, a, m);
    return res;
}

bool miller_rabin(ll n, ll a) {
    if (n % a == 0) return n == a;
    ll d = n - 1; int r = 0;
    while (d % 2 == 0) { d /= 2; r++; }
    ll x = powmod(a, d, n);
    if (x == 1 || x == n - 1) return true;
    for (int i = 0; i < r - 1; i++) {
        x = mulmod(x, x, n);
        if (x == n - 1) return true;
    }
    return false;
}

bool is_prime(ll n) {
    if (n < 2) return false;
    for (ll a : {2LL,3LL,5LL,7LL,11LL,13LL,17LL,19LL,23LL,29LL,31LL,37LL})
        if (n == a) return true;
        else if (!miller_rabin(n, a)) return false;
    return true;
}

// ================================================================
// POLLARD'S RHO — Brent's variant
// Returns a non-trivial factor of n, or n if n is prime.
// ================================================================
mt19937_64 rng(chrono::steady_clock::now().time_since_epoch().count());

ll pollard_rho(ll n) {
    if (n % 2 == 0) return 2;
    if (is_prime(n)) return n;

    while (true) {
        // Random starting values
        ll x = rng() % (n - 2) + 2;
        ll y = x;
        ll c = rng() % (n - 1) + 1;
        ll d = 1;

        // Brent's variant with product accumulation
        ll q = 1;
        ll ys, xs;
        int m = 128; // batch size

        do {
            xs = x;
            for (int i = 0; i < (int)(d = 1, m); i++) {
                y = (mulmod(y, y, n) + c) % n;
            }
            // Not quite right; use proper Brent below
        } while (false); // placeholder

        // Proper Brent's variant:
        x = rng() % (n - 2) + 2;
        c = rng() % (n - 1) + 1;
        y = x;
        d = 1;
        ll r = 1;
        q = 1;

        while (d == 1) {
            xs = x; // save position
            for (ll i = 0; i < r; i++)
                x = (mulmod(x, x, n) + c) % n;

            ll k = 0;
            while (k < r && d == 1) {
                ys = x;
                for (ll i = 0; i < min((ll)m, r - k); i++) {
                    x = (mulmod(x, x, n) + c) % n;
                    q = mulmod(q, abs(xs - x), n); // accumulate product
                }
                d = __gcd(q, n);
                k += m;
            }
            r *= 2;
        }

        if (d == n) {
            // Backtrack from ys
            d = 1;
            while (d == 1) {
                ys = (mulmod(ys, ys, n) + c) % n;
                d = __gcd(abs(xs - ys), n);
            }
        }

        if (d != n) return d;
        // else: retry with different c, x
    }
}

// ================================================================
// COMPLETE FACTORIZATION using Pollard's Rho + Miller-Rabin
// Returns sorted vector of prime factors (with repetition).
// ================================================================
void factorize(ll n, vector<ll>& factors) {
    if (n == 1) return;
    if (is_prime(n)) {
        factors.push_back(n);
        return;
    }
    // Find a non-trivial factor
    ll d = n;
    while (d == n) d = pollard_rho(n);
    factorize(d,     factors);
    factorize(n / d, factors);
}

vector<ll> factorize(ll n) {
    vector<ll> factors;
    factorize(n, factors);
    sort(factors.begin(), factors.end());
    return factors;
}

// ================================================================
// GET PRIME FACTORIZATION AS MAP {prime: exponent}
// ================================================================
map<ll,int> prime_factorization(ll n) {
    auto factors = factorize(n);
    map<ll,int> result;
    for (ll p : factors) result[p]++;
    return result;
}

// ================================================================
// EULER'S TOTIENT using factorization
// phi(n) = n * product (1 - 1/p) for each prime p | n
// ================================================================
ll euler_totient(ll n) {
    auto pf = prime_factorization(n);
    ll result = n;
    for (auto& [p, e] : pf) {
        result = result / p * (p - 1);
    }
    return result;
}

// ================================================================
// NUMBER OF DIVISORS using factorization
// d(n) = product (e_i + 1) for n = p1^e1 * p2^e2 * ...
// ================================================================
ll num_divisors(ll n) {
    auto pf = prime_factorization(n);
    ll result = 1;
    for (auto& [p, e] : pf) result *= (e + 1);
    return result;
}

// ================================================================
// SUM OF DIVISORS using factorization
// sigma(n) = product (p^{e+1} - 1) / (p - 1)
// ================================================================
ll sum_divisors(ll n) {
    auto pf = prime_factorization(n);
    ll result = 1;
    for (auto& [p, e] : pf) {
        ll term = 1, pk = 1;
        for (int i = 0; i <= e; i++) { term += pk; pk *= p; }
        result *= term - 1; // (p^{e+1}-1)/(p-1) = 1+p+...+p^e
    }
    return result;
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Basic factorization
    for (ll n : {12LL, 100LL, 360LL, 999983LL,
                 1000000007LL, 123456789012345LL,
                 (ll)998244353 * 999999937LL}) {
        auto f = factorize(n);
        printf("%lld = ", n);
        for (int i = 0; i < (int)f.size(); i++) {
            printf("%lld", f[i]);
            if (i+1 < (int)f.size()) printf(" * ");
        }
        printf("\n");
    }

    // Verify
    printf("\nVerification:\n");
    ll n = (ll)998244353 * 999999937LL;
    auto pf = prime_factorization(n);
    ll product = 1;
    for (auto& [p, e] : pf) {
        for (int i = 0; i < e; i++) product *= p;
    }
    printf("  %lld: product check = %s\n", n, product == n ? "OK" : "FAIL");

    // Number theory functions
    printf("\nEuler totient:\n");
    for (ll n2 : {6LL, 10LL, 12LL, 100LL, 1000000007LL}) {
        printf("  phi(%lld) = %lld\n", n2, euler_totient(n2));
    }

    printf("\nNumber of divisors:\n");
    for (ll n2 : {1LL, 6LL, 12LL, 36LL, 100LL}) {
        printf("  d(%lld) = %lld\n", n2, num_divisors(n2));
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Numerics;

public static class PollardRho {

    private static readonly Random Rng = new(
        (int)DateTimeOffset.UtcNow.ToUnixTimeMilliseconds());

    // ================================================================
    // UTILITIES
    // ================================================================
    private static long MulMod(long a, long b, long m) =>
        (long)((BigInteger)a * b % m);

    private static long PowMod(long a, long b, long m) {
        long res = 1; a %= m;
        for (; b > 0; b >>= 1, a = MulMod(a, a, m))
            if ((b & 1) == 1) res = MulMod(res, a, m);
        return res;
    }

    private static long GCD(long a, long b) =>
        b == 0 ? a : GCD(b, a % b);

    // ================================================================
    // MILLER-RABIN — deterministic for n < 3.2 * 10^18
    // ================================================================
    private static bool MillerRabin(long n, long a) {
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

    public static bool IsPrime(long n) {
        if (n < 2) return false;
        foreach (long a in new long[]{2,3,5,7,11,13,17,19,23,29,31,37}) {
            if (n == a) return true;
            if (n % a == 0) return false;
        }
        foreach (long a in new long[]{2,3,5,7,11,13,17,19,23,29,31,37})
            if (!MillerRabin(n, a)) return false;
        return true;
    }

    // ================================================================
    // POLLARD'S RHO — Brent's variant
    // Returns a non-trivial factor of n, or n if prime.
    // ================================================================
    public static long FindFactor(long n) {
        if (n % 2 == 0) return 2;
        if (IsPrime(n))  return n;

        while (true) {
            long x = (long)(Rng.NextInt64() % (n - 2)) + 2;
            long c = (long)(Rng.NextInt64() % (n - 1)) + 1;
            long y = x, d = 1, q = 1;
            long xs = x, ys = y;
            long r = 1;
            const int m = 128;

            while (d == 1) {
                xs = x;
                for (long i = 0; i < r; i++)
                    x = (MulMod(x, x, n) + c) % n;

                long k = 0;
                while (k < r && d == 1) {
                    ys = x;
                    for (long i = 0; i < Math.Min(m, r - k); i++) {
                        x  = (MulMod(x, x, n) + c) % n;
                        q  = MulMod(q, Math.Abs(xs - x), n);
                    }
                    d = GCD(q, n);
                    k += m;
                }
                r *= 2;
            }

            if (d == n) {
                d = 1;
                while (d == 1) {
                    ys = (MulMod(ys, ys, n) + c) % n;
                    d  = GCD(Math.Abs(xs - ys), n);
                }
            }

            if (d != n) return d;
        }
    }

    // ================================================================
    // COMPLETE FACTORIZATION
    // ================================================================
    private static void Factorize(long n, List<long> factors) {
        if (n == 1) return;
        if (IsPrime(n)) { factors.Add(n); return; }
        long d = n;
        while (d == n) d = FindFactor(n);
        Factorize(d,     factors);
        Factorize(n / d, factors);
    }

    public static List<long> Factorize(long n) {
        var f = new List<long>();
        Factorize(n, f);
        f.Sort();
        return f;
    }

    public static SortedDictionary<long,int> PrimeFactorization(long n) {
        var result = new SortedDictionary<long, int>();
        foreach (long p in Factorize(n)) {
            result.TryGetValue(p, out int cnt);
            result[p] = cnt + 1;
        }
        return result;
    }

    // ================================================================
    // NUMBER THEORY FUNCTIONS
    // ================================================================
    public static long EulerTotient(long n) {
        long result = n;
        foreach (var (p, _) in PrimeFactorization(n))
            result = result / p * (p - 1);
        return result;
    }

    public static long NumDivisors(long n) {
        long result = 1;
        foreach (var (_, e) in PrimeFactorization(n))
            result *= (e + 1);
        return result;
    }

    // ================================================================
    // Usage
    // ================================================================
    public static void Main() {
        long[] tests = {
            12, 100, 360, 999983,
            1000000007L,
            123456789012345L,
            998244353L * 999999937L
        };

        Console.WriteLine("Factorizations:");
        foreach (long n in tests) {
            var f = Factorize(n);
            Console.WriteLine($"  {n} = {string.Join(" * ", f)}");
        }

        Console.WriteLine("\nEuler's totient:");
        foreach (long n in new long[]{6,10,12,100,1000000007L})
            Console.WriteLine($"  phi({n}) = {EulerTotient(n)}");

        Console.WriteLine("\nNumber of divisors:");
        foreach (long n in new long[]{1,6,12,36,100})
            Console.WriteLine($"  d({n}) = {NumDivisors(n)}");
    }
}
```

---

## Algorithm Visualization — The ρ Shape

```
Sequence modulo n:    x_0 → x_1 → x_2 → x_3 → ... → x_i → ...
                                                ↗           ↘
Sequence modulo p:         tail                  cycle (length λ)
(p = small factor of n)   (length μ)             x_{μ+λ} = x_μ

The path in Z/pZ eventually enters a cycle:
  x_0, x_1, ..., x_{μ-1},  x_μ, x_{μ+1}, ..., x_{μ+λ-1}, x_μ, ...
                             ↑___________________________|
                             cycle = the "ρ" loop

Floyd's detection: tortoise at position k, hare at position 2k
  When hare catches tortoise: k ≡ 0 (mod λ), so k ≥ μ
  At this point: gcd(x_k - x_{2k}, n) may reveal factor p
```

---

## Batch GCD Optimization

Instead of computing `gcd(|x - y|, n)` at every step (expensive), batch `m` steps:

```
Accumulate: q = q * |x_i - y_i| mod n   for i = 1..m
Then: gcd(q, n)

If gcd != 1: backtrack to find exact step
If gcd == n: q had a multiple of n → reduce batch size or backtrack from ys
```

Typical batch size `m = 128` reduces gcd calls by 128x. Each gcd costs O(log n) — this optimization makes Brent's variant ~128x faster than step-by-step Floyd.

---

## Pitfalls

- **`abs(x - y)` with signed overflow** — `x` and `y` are in `[0, n-1]`, so `x - y` is in `[-(n-1), n-1]`. For `n` near `2^63`, `x - y` overflows `int64`. Use `unsigned long long` or compute `x >= y ? x - y : y - x`. In C# use `Math.Abs(x - y)` — but this can also overflow for negative `long`. Cast to `ulong` first.
- **c = 0 or c = n-2 must be avoided** — `f(x) = x^2 mod n` (c=0) generates only 0 and 1. `f(x) = x^2 - 2 mod n` can produce degenerate cycles. These constants should be excluded in the random selection.
- **`d == n` requires backtracking** — when `gcd(product, n) == n`, the product contains a multiple of `n` (not just p), so `p` is hidden. Backtrack from the last saved `ys` with step-by-step gcd until a proper factor is found. Forgetting this case causes the algorithm to loop forever.
- **Recursion on `factorize(d)` and `factorize(n/d)` — both may be composite** — Pollard's Rho finds *a* factor `d`, which may itself be composite. Both `d` and `n/d` must be recursively factorized. Returning `d` as a final factor without primality checking gives wrong factorizations for highly composite numbers.
- **Miller-Rabin witness set must be deterministic for large n** — for n < 3.2×10^18, the 12-witness set {2,3,5,7,11,13,17,19,23,29,31,37} is provably sufficient. Using fewer witnesses (e.g., just {2,3,5}) may misclassify some composites as prime, causing Pollard's Rho to return `n` instead of a factor.
- **Infinite loop when n = p^k** — if `n` is a perfect power `p^k`, all `gcd(|x-y|, n)` computations may return `n` until a very specific `c` is tried. Add trial division for small primes (2, 3, 5, ..., 1000) before invoking Pollard's Rho to handle these cases quickly.
- **`mulmod` overflow for n near 2^62** — `a * b mod n` overflows `int64` when `a, b ≈ n ≈ 2^62`. Always use `__int128` (C++) or `BigInteger` (C#) for the multiply-then-mod step. Using plain `long long` multiplication silently wraps around and produces wrong residues.

---

## Complexity Summary

| Method | Time complexity | Practical limit |
|---|---|---|
| Trial division | O(n^{1/2}) | n ≤ 10^{12} |
| Pollard's Rho | O(n^{1/4} log n) | n ≤ 10^{20} |
| Quadratic Sieve | O(exp(√(log n log log n))) | n ≤ 10^{50} |
| GNFS | O(exp((64/9)^{1/3} (log n)^{1/3} (log log n)^{2/3})) | n ≤ 10^{300} |

For competitive programming (n ≤ 10^{18}): Pollard's Rho with Miller-Rabin handles all cases in milliseconds.

---

## Conclusion

Pollard's Rho is the **standard factorization algorithm for numbers up to 10^{20}**:

- Exploits the birthday paradox in Z/pZ to find a factor in O(p^{1/4}) steps, where p is the smallest prime factor.
- Brent's variant with batched GCD computation reduces constant factors by ~128x over the basic Floyd's cycle detection.
- Combined with Miller-Rabin primality testing, it gives complete prime factorization in O(n^{1/4} polylog n) expected time.
- For n ≤ 10^{18}, a typical factorization completes in under a millisecond.

**Key takeaway:**  
Pollard's Rho is the first choice for factoring large numbers in competitive programming. The implementation has three components: Miller-Rabin for primality, `mulmod` via `__int128` for overflow-safe arithmetic, and Brent's cycle detection with batched GCD. The only non-trivial case is `gcd == n` (backtrack from saved position). Everything else is a straightforward random walk with cycle detection.
