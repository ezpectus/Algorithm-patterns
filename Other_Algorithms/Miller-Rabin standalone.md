# Miller-Rabin Primality Test

## Origin & Motivation

The Miller-Rabin test was developed by Gary Miller in 1976 (deterministic, under GRH) and made probabilistic by Michael Rabin in 1980. It is a **compositeness witness test** based on Fermat's little theorem and properties of square roots of unity modulo primes.

For a prime `p` and any `a` with `gcd(a, p) = 1`: `a^{p-1} ≡ 1 (mod p)` (Fermat). Writing `p-1 = 2^r * d` (d odd), the sequence `a^d, a^{2d}, ..., a^{2^r d}` must end in 1 and the last term before 1 must be -1 (or the first term is 1). If this fails for some `a`, then `n` is **definitely composite** — `a` is a **witness to compositeness**. If it passes for all tested `a`, `n` is **probably prime**.

By choosing `a` from a carefully selected **deterministic witness set**, the test becomes exact (not probabilistic) for all `n` below specific bounds — making it the standard primality test for competitive programming and cryptography.

Complexity: **O(k log^2 n)** per test where k = number of witnesses.

---

## Where It Is Used

- Primality testing in competitive programming (n up to 10^{18})
- RSA key generation (certifying large primes)
- Pollard's Rho (primality check after finding factors)
- Cryptographic protocols requiring prime generation
- Number theory: sieve extensions, smooth number detection
- Deterministic primality for moderate n (< 3.2 × 10^{18})

---

## Mathematical Foundation

### Fermat's Little Theorem

For prime `p` and `gcd(a, p) = 1`: `a^{p-1} ≡ 1 (mod p)`

### Square Roots of Unity

For a prime `p`, the only square roots of 1 modulo `p` are ±1. That is:
```
x^2 ≡ 1 (mod p)  ⟹  x ≡ 1 or x ≡ -1 (mod p)
```

### Miller-Rabin Criterion

Write `n - 1 = 2^r * d` with `d` odd. For a prime `n`, any `a` with `gcd(a, n) = 1` satisfies at least one of:
```
a^d ≡ 1 (mod n)
a^{2^j * d} ≡ -1 (mod n)   for some j ∈ {0, 1, ..., r-1}
```

If neither holds, `a` is a **Miller-Rabin witness** to the compositeness of `n`.

**Strong probable prime (SPRP):** `n` is a strong probable prime to base `a` if it passes the Miller-Rabin test for `a`. Every prime is an SPRP for every valid base. A composite that is SPRP to base `a` is a **strong pseudoprime** to base `a` — these are extremely rare.

---

## Deterministic Witness Sets

For `n` below specific bounds, testing only a fixed set of witnesses gives a **provably correct** (not probabilistic) answer:

| Upper bound | Witnesses needed | Source |
|---|---|---|
| n < 2,047 | {2} | — |
| n < 1,373,653 | {2, 3} | — |
| n < 9,080,191 | {31, 73} | — |
| n < 25,326,001 | {2, 3, 5} | — |
| n < 3,215,031,751 | {2, 3, 5, 7} | — |
| n < 3,317,044,064,679,887,385,961,981 | {2,3,5,7,11,13,17,19,23,29,31,37} | Verified |
| n < 3.2 × 10^{18} | {2,3,5,7,11,13,17,19,23,29,31,37} | Sufficient for 64-bit |

For randomized testing: k random witnesses in [2, n-2] give error probability ≤ 4^{-k}.

---

## Algorithm

```
is_prime(n):
    if n < 2: return false
    if n in small_primes: return true
    if n is even or divisible by small prime: return false

    write n-1 = 2^r * d  (d odd)

    for each witness a:
        x = a^d mod n
        if x == 1 or x == n-1: continue  (passes this witness)

        for j = 1 to r-1:
            x = x^2 mod n
            if x == n-1: break  (passes)
        else:
            return false  (a is a witness to compositeness)

    return true  (probably prime / certainly prime for det. witnesses)
```

---

## Complexity Analysis

| Operation | Cost | Notes |
|---|---|---|
| Write n-1 = 2^r * d | O(log n) | bit operations |
| Compute a^d mod n | O(log n) multiplications | fast exponentiation |
| Each modular multiply | O(log^2 n) bit operations | or O(1) with 128-bit |
| One witness test | O(r * log n * log^2 n) = O(log^3 n) | r ≤ 63 for 64-bit |
| k witnesses total | O(k log^3 n) | k = 12 for deterministic |

For 64-bit n: ~12 witnesses × ~64 squarings × O(1) mulmod = ~768 operations. Extremely fast.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll  = long long;
using ull = unsigned long long;

// ================================================================
// MODULAR ARITHMETIC — overflow-safe for 64-bit n
// ================================================================

// Multiply a*b mod m without overflow (using __int128)
ll mulmod(ll a, ll b, ll m) {
    return (__int128)a * b % m;
}

// Fast modular exponentiation: a^b mod m
ll powmod(ll a, ll b, ll m) {
    ll res = 1;
    a %= m;
    if (a < 0) a += m;
    while (b > 0) {
        if (b & 1) res = mulmod(res, a, m);
        a = mulmod(a, a, m);
        b >>= 1;
    }
    return res;
}

// ================================================================
// MILLER-RABIN — Single witness test
// Returns true if n passes the test for witness a
// (n is "probably prime" or "strong pseudoprime to base a").
// Returns false if a is a witness to n's compositeness.
// ================================================================
bool miller_rabin_witness(ll n, ll a) {
    // Handle small cases
    if (n % a == 0) return n == a;

    // Write n-1 = 2^r * d, d odd
    ll d = n - 1;
    int r = 0;
    while (d % 2 == 0) { d /= 2; r++; }

    // Compute x = a^d mod n
    ll x = powmod(a, d, n);

    // Check Miller-Rabin conditions
    if (x == 1 || x == n - 1) return true;

    for (int j = 0; j < r - 1; j++) {
        x = mulmod(x, x, n);
        if (x == n - 1) return true;
    }

    return false; // a is a witness to compositeness
}

// ================================================================
// MILLER-RABIN — Deterministic for n < 3.2 * 10^18
// Uses the minimal sufficient witness set for 64-bit integers.
// ================================================================
bool is_prime(ll n) {
    if (n < 2) return false;
    if (n == 2) return true;
    if (n % 2 == 0) return false;

    // Small prime fast path
    for (ll p : {3LL, 5LL, 7LL, 11LL, 13LL, 17LL, 19LL, 23LL, 29LL, 31LL, 37LL}) {
        if (n == p) return true;
        if (n % p == 0) return false;
    }

    // Deterministic witness set for n < 3,215,031,751 (32-bit)
    if (n < 3215031751LL) {
        for (ll a : {2LL, 3LL, 5LL, 7LL})
            if (!miller_rabin_witness(n, a)) return false;
        return true;
    }

    // Deterministic for n < 3.2 * 10^18 (64-bit)
    for (ll a : {2LL,3LL,5LL,7LL,11LL,13LL,17LL,19LL,23LL,29LL,31LL,37LL})
        if (!miller_rabin_witness(n, a)) return false;
    return true;
}

// ================================================================
// MILLER-RABIN — Randomized version (for arbitrary n, bounded error)
// k rounds: error probability <= 4^{-k}
// ================================================================
bool is_prime_random(ll n, int k = 20) {
    if (n < 2) return false;
    if (n == 2 || n == 3) return true;
    if (n % 2 == 0) return false;

    // Small factors
    for (ll p : {3LL,5LL,7LL,11LL,13LL}) {
        if (n == p) return true;
        if (n % p == 0) return false;
    }

    mt19937_64 rng(chrono::steady_clock::now().time_since_epoch().count());
    uniform_int_distribution<ll> dist(2, n - 2);

    for (int i = 0; i < k; i++) {
        ll a = dist(rng);
        if (!miller_rabin_witness(n, a)) return false;
    }
    return true; // prime with probability >= 1 - 4^{-k}
}

// ================================================================
// COUNT PRIMES in [lo, hi] using Miller-Rabin
// ================================================================
int count_primes_range(ll lo, ll hi) {
    int cnt = 0;
    for (ll n = lo; n <= hi; n++)
        if (is_prime(n)) cnt++;
    return cnt;
}

// ================================================================
// NEXT PRIME >= n
// ================================================================
ll next_prime(ll n) {
    if (n <= 2) return 2;
    ll p = n % 2 == 0 ? n + 1 : n;
    while (!is_prime(p)) p += 2;
    return p;
}

// ================================================================
// PREVIOUS PRIME <= n (>= 2)
// ================================================================
ll prev_prime(ll n) {
    if (n < 2) return -1;
    if (n == 2) return 2;
    ll p = n % 2 == 0 ? n - 1 : n;
    while (p >= 2 && !is_prime(p)) p -= 2;
    return p >= 2 ? p : -1;
}

// ================================================================
// GENERATE PRIMES up to n using sieve + Miller-Rabin for large range
// For small n: use Sieve of Eratosthenes directly
// ================================================================
vector<ll> sieve(ll n) {
    vector<bool> is_p(n + 1, true);
    is_p[0] = is_p[1] = false;
    for (ll i = 2; i * i <= n; i++)
        if (is_p[i])
            for (ll j = i*i; j <= n; j += i)
                is_p[j] = false;
    vector<ll> primes;
    for (ll i = 2; i <= n; i++)
        if (is_p[i]) primes.push_back(i);
    return primes;
}

// ================================================================
// PRIME GAP — find largest prime gap up to n
// ================================================================
pair<ll,ll> largest_prime_gap(ll n) {
    ll max_gap = 0, gap_start = 2;
    ll prev = 2;
    for (ll p = 3; p <= n; p += 2) {
        if (is_prime(p)) {
            if (p - prev > max_gap) {
                max_gap  = p - prev;
                gap_start = prev;
            }
            prev = p;
        }
    }
    return {gap_start, max_gap};
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Basic primality testing
    printf("=== Primality Tests ===\n");
    for (ll n : {0LL, 1LL, 2LL, 3LL, 4LL, 17LL, 100LL,
                 999983LL, 1000000007LL, 1000000008LL,
                 998244353LL, 999999999999999989LL}) {
        printf("  is_prime(%lld) = %s\n", n, is_prime(n) ? "true" : "false");
    }

    // Large Mersenne-style primes
    printf("\n=== Large Primes ===\n");
    ll candidates[] = {
        (1LL << 31) - 1,              // 2^31-1 = 2147483647 (Mersenne prime)
        (1LL << 61) - 1,              // 2^61-1 (Mersenne prime)
        999999999999999877LL,
        1000000000000000003LL
    };
    for (ll n : candidates)
        printf("  %lld: %s\n", n, is_prime(n) ? "PRIME" : "COMPOSITE");

    // Pseudoprimes: Carmichael numbers (pass Fermat but are composite)
    printf("\n=== Carmichael Numbers (composite but Fermat fools) ===\n");
    for (ll n : {561LL, 1105LL, 1729LL, 2465LL, 8911LL}) {
        printf("  %lld: Miller-Rabin = %s  (Carmichael = composite)\n",
               n, is_prime(n) ? "prime?" : "composite OK");
    }

    // Next / prev prime
    printf("\n=== Next / Prev Prime ===\n");
    for (ll n : {100LL, 1000LL, 1000000LL, 1000000000LL}) {
        printf("  next_prime(%lld) = %lld\n", n, next_prime(n));
        printf("  prev_prime(%lld) = %lld\n", n, prev_prime(n));
    }

    // Count primes in range
    printf("\n=== Prime Counting ===\n");
    printf("  pi(100)  = %d\n", count_primes_range(2, 100));  // 25
    printf("  pi(1000) = %d\n", count_primes_range(2, 1000)); // 168

    // Prime gaps
    printf("\n=== Largest Prime Gaps ===\n");
    for (ll n : {100LL, 1000LL, 10000LL}) {
        auto [start, gap] = largest_prime_gap(n);
        printf("  Largest gap up to %lld: gap=%lld starting after %lld\n",
               n, gap, start);
    }

    // Randomized test (for illustration)
    printf("\n=== Randomized Miller-Rabin (k=30) ===\n");
    for (ll n : {1000000007LL, 1000000009LL, 1000000011LL})
        printf("  %lld: %s\n", n,
               is_prime_random(n, 30) ? "probably prime" : "composite");

    // Witness analysis: show which witnesses expose a composite
    printf("\n=== Witness Analysis for n=561 (Carmichael) ===\n");
    ll n = 561;
    for (ll a : {2LL, 3LL, 5LL, 7LL, 11LL, 13LL}) {
        bool passes = miller_rabin_witness(n, a);
        printf("  witness a=%lld: %s\n", a,
               passes ? "passes (pseudoprime to this base)" : "EXPOSES as composite");
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

public static class MillerRabin {

    // ================================================================
    // MODULAR ARITHMETIC
    // ================================================================
    public static long MulMod(long a, long b, long m) =>
        (long)((BigInteger)a * b % m);

    public static long PowMod(long a, long b, long m) {
        long res = 1;
        a %= m;
        if (a < 0) a += m;
        while (b > 0) {
            if ((b & 1) == 1) res = MulMod(res, a, m);
            a = MulMod(a, a, m);
            b >>= 1;
        }
        return res;
    }

    // ================================================================
    // SINGLE WITNESS TEST
    // Returns true if n passes Miller-Rabin for witness a.
    // Returns false if a is a witness to compositeness of n.
    // ================================================================
    public static bool WitnessTest(long n, long a) {
        if (n % a == 0) return n == a;

        long d = n - 1;
        int r = 0;
        while (d % 2 == 0) { d /= 2; r++; }

        long x = PowMod(a, d, n);
        if (x == 1 || x == n - 1) return true;

        for (int j = 0; j < r - 1; j++) {
            x = MulMod(x, x, n);
            if (x == n - 1) return true;
        }
        return false;
    }

    // ================================================================
    // DETERMINISTIC MILLER-RABIN — correct for all n < 3.2 × 10^18
    // ================================================================
    private static readonly long[] Witnesses32 = {2, 3, 5, 7};
    private static readonly long[] Witnesses64 = {2,3,5,7,11,13,17,19,23,29,31,37};

    public static bool IsPrime(long n) {
        if (n < 2) return false;
        if (n == 2) return true;
        if (n % 2 == 0) return false;

        // Small primes fast path
        foreach (long p in new long[]{3,5,7,11,13,17,19,23,29,31,37}) {
            if (n == p) return true;
            if (n % p == 0) return false;
        }

        var witnesses = n < 3_215_031_751L ? Witnesses32 : Witnesses64;
        foreach (long a in witnesses)
            if (!WitnessTest(n, a)) return false;
        return true;
    }

    // ================================================================
    // RANDOMIZED MILLER-RABIN — k rounds, error <= 4^{-k}
    // ================================================================
    public static bool IsPrimeRandom(long n, int k = 20) {
        if (n < 2) return false;
        if (n == 2 || n == 3) return true;
        if (n % 2 == 0) return false;

        foreach (long p in new long[]{3,5,7,11,13}) {
            if (n == p) return true;
            if (n % p == 0) return false;
        }

        var rng = new Random();
        for (int i = 0; i < k; i++) {
            long a = (long)(rng.NextDouble() * (n - 3)) + 2;
            if (!WitnessTest(n, a)) return false;
        }
        return true;
    }

    // ================================================================
    // UTILITIES
    // ================================================================
    public static long NextPrime(long n) {
        if (n <= 2) return 2;
        long p = n % 2 == 0 ? n + 1 : n;
        while (!IsPrime(p)) p += 2;
        return p;
    }

    public static long PrevPrime(long n) {
        if (n < 2) return -1;
        if (n == 2) return 2;
        long p = n % 2 == 0 ? n - 1 : n;
        while (p >= 2 && !IsPrime(p)) p -= 2;
        return p >= 2 ? p : -1;
    }

    public static int CountPrimes(long lo, long hi) {
        int cnt = 0;
        for (long n = lo; n <= hi; n++)
            if (IsPrime(n)) cnt++;
        return cnt;
    }

    // ================================================================
    // Usage
    // ================================================================
    public static void Main() {
        Console.WriteLine("=== Primality Tests ===");
        foreach (long n in new long[]{0,1,2,3,4,17,100,999983,
                                       1000000007L,1000000008L,
                                       998244353L,999999999999999989L})
            Console.WriteLine($"  is_prime({n}) = {IsPrime(n)}");

        Console.WriteLine("\n=== Large Primes ===");
        foreach (long n in new long[]{
                (1L<<31)-1, 999999999999999877L, 1000000000000000003L})
            Console.WriteLine($"  {n}: {(IsPrime(n)?"PRIME":"COMPOSITE")}");

        Console.WriteLine("\n=== Carmichael Numbers ===");
        foreach (long n in new long[]{561,1105,1729,2465,8911})
            Console.WriteLine($"  {n}: {(IsPrime(n)?"prime?":"composite OK")}");

        Console.WriteLine("\n=== Next / Prev Prime ===");
        foreach (long n in new long[]{100,1000,1_000_000}) {
            Console.WriteLine($"  next({n}) = {NextPrime(n)}, prev({n}) = {PrevPrime(n)}");
        }

        Console.WriteLine($"\npi(100)  = {CountPrimes(2,100)}");   // 25
        Console.WriteLine($"pi(1000) = {CountPrimes(2,1000)}");   // 168

        Console.WriteLine("\n=== Witness Analysis for n=561 ===");
        long comp = 561;
        foreach (long a in new long[]{2,3,5,7,11,13})
            Console.WriteLine($"  a={a}: {(WitnessTest(comp,a)?"passes":"EXPOSES composite")}");
    }
}
```

---

## Why Carmichael Numbers Don't Fool Miller-Rabin

**Fermat test** fails for Carmichael numbers: 561 = 3 × 11 × 17 satisfies `a^{560} ≡ 1 (mod 561)` for all `gcd(a, 561) = 1`. So Fermat's test always returns "probably prime" for 561.

**Miller-Rabin** catches them: 560 = 2^4 × 35. Testing a=2:

```
x = 2^35 mod 561 = 263
x^2 mod 561 = 166    (≠ 560)
x^2 mod 561 = 67     (≠ 560)
x^2 mod 561 = 1      (≠ 560 and previous ≠ 1)
→ COMPOSITE ✓
```

The sequence `263 → 166 → 67 → 1` reveals that 1 was "reached from" 67 (not ±1), violating the square-root-of-unity condition.

---

## Error Probability Analysis

For a **composite** n and a **random** witness a ∈ [2, n-2]:

```
Pr[a does NOT expose n as composite] ≤ 1/4
```

This bound is tight: achieved by Carmichael numbers with specific structure.

After k independent random witnesses:
```
Pr[all k witnesses fail to expose composite] ≤ (1/4)^k = 4^{-k}

k=10:  error ≤ 10^{-6}
k=20:  error ≤ 10^{-12}
k=40:  error ≤ 10^{-24}
```

For the **deterministic** 12-witness set: error = 0 for all n < 3.2 × 10^{18}.

---

## Pitfalls

- **mulmod overflow** — for `n` near `2^62`, computing `a^2 mod n` requires `a * a` which can reach `2^{124}`. Always use `__int128` (C++) or `BigInteger` (C#). Using `long long` multiplication silently overflows, giving wrong squaring results that make composites look prime.
- **n-1 = 1 when n = 2** — the decomposition `n-1 = 2^r * d` gives `r=1, d=1` for n=3 and `r=0`... wait: n=2: n-1=1=2^0*1, r=0. The loop `for j in 0..r-2` doesn't execute. Then check `x == n-1 = 1`: if `a^1 mod 2 = 0 ≠ 1`. Handle n=2 as a special case before the main algorithm.
- **Witness a must be in [2, n-2]** — testing a=1 always returns "passes" (1^anything = 1 ≡ 1). Testing a=n-1 also always passes (since `(n-1)^d ≡ (-1)^d ≡ ±1 mod n` for odd d). Always restrict a to [2, n-2]. The deterministic witness set handles this implicitly since all witnesses are small.
- **n % a == 0 shortcut** — if n is divisible by the witness a, then n is prime iff n == a. This shortcut avoids the modular exponentiation for small witnesses like 2, 3, 5 and correctly handles n=2, n=3, n=5, etc.
- **Deterministic set is only valid for specific bounds** — the 4-witness set {2,3,5,7} is sufficient only for n < 3,215,031,751 (~3.2 × 10^9). Using it for larger n can misclassify composites. Use the 12-witness set for n < 3.2 × 10^{18}. For n beyond this bound, use a full probabilistic test or a different primality certificate.
- **d must be odd after division** — the decomposition extracts all factors of 2 from n-1. A common bug is stopping at `d = (n-1) / 2` (one division) rather than dividing until d is odd. This gives wrong `r` and `d`, producing incorrect witness tests.
- **Strong pseudoprimes exist for every single base** — for any fixed witness a, infinitely many composites pass the test. Only the combination of multiple witnesses eliminates all composites below the proven bound. Never rely on a single witness for security-sensitive applications.

---

## Complexity Summary

| Variant | Witnesses | Time per test | Valid range | Error |
|---|---|---|---|---|
| Fermat test | k | O(k log^2 n) | All n | Fails on Carmichael |
| Miller-Rabin random | k | O(k log^2 n) | All n | ≤ 4^{-k} |
| Miller-Rabin det. (32-bit) | 4 | O(log^2 n) | n < 3.2×10^9 | Zero |
| Miller-Rabin det. (64-bit) | 12 | O(log^2 n) | n < 3.2×10^{18} | Zero |
| AKS | — | O(log^6 n) | All n | Zero |

Miller-Rabin deterministic (12 witnesses) is the **practical standard** — 10^6x faster than AKS with identical correctness for 64-bit integers.

---

## Conclusion

Miller-Rabin is the **standard primality test for competitive programming and cryptography**:

- Deterministic and exact for all 64-bit integers using the 12-witness set — no probability of error.
- O(log^2 n) per test with `__int128` multiply — handles 10^6 primality tests per second.
- Correctly distinguishes all Carmichael numbers (which fool Fermat's test) via the square-root-of-unity condition.
- The randomized variant with k=20 rounds gives error probability < 10^{-12} for any n, suitable for cryptographic prime generation.

**Key takeaway:**  
For competitive programming, always use the deterministic 12-witness variant — it's exact, faster than the randomized variant (no RNG overhead), and the overhead vs. 4 witnesses is negligible. The three-line core — decompose n-1, compute a^d mod n, check the sequence for 1 or -1 — is the complete algorithm. The only implementation detail that matters is `mulmod` via `__int128` to prevent overflow.
