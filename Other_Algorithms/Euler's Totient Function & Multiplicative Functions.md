# Euler's Totient Function & Multiplicative Functions

## Origin & Motivation

Euler's totient function `φ(n)` was introduced by **Leonhard Euler** in 1763 in his work on
modular arithmetic and generalizations of Fermat's Little Theorem. The function counts how many
integers from `1` to `n` are **coprime** to `n` (i.e., share no common factor with `n` other
than 1).

Formally:

```
φ(n) = |{ k : 1 ≤ k ≤ n, gcd(k, n) = 1 }|
```

**Why does it matter?**  
Euler's theorem states that for any `a` coprime to `n`:

```
a^φ(n) ≡ 1 (mod n)
```

This is a cornerstone of number theory and the direct foundation of RSA encryption. Without
`φ(n)`, RSA key generation — choosing `e` and computing `d = e⁻¹ mod φ(n)` — is impossible.

Beyond totient, the broader family of **multiplicative functions** (functions `f` where
`gcd(m,n)=1 ⟹ f(mn) = f(m)·f(n)`) forms a rich algebraic structure. The totient is just
one member of this family, alongside the divisor count `d(n)`, divisor sum `σ(n)`, Möbius
function `μ(n)`, Liouville function `λ(n)`, and many others. All of them can be computed
efficiently using the **Sieve of Eratosthenes** framework and the **Dirichlet product**.

---

## Definitions

### Euler's Totient φ(n)

```
φ(1) = 1
φ(n) = n · ∏ (1 - 1/p)   for all distinct prime factors p of n
           p|n
```

**Example:**  
`φ(12) = 12 · (1 - 1/2) · (1 - 1/3) = 12 · 1/2 · 2/3 = 4`  
The 4 integers coprime to 12 in [1,12] are: `{1, 5, 7, 11}`.

### Multiplicative Function (definition)

A function `f : ℕ → ℂ` is **multiplicative** if:
- `f(1) = 1`
- `f(mn) = f(m)·f(n)` whenever `gcd(m, n) = 1`

It is **completely multiplicative** if `f(mn) = f(m)·f(n)` for **all** `m, n` (no coprimality
requirement).

The set of multiplicative functions is closed under **Dirichlet convolution** — a key fact that
makes sieve-based computation possible.

---

## Key Multiplicative Functions

| Function | Symbol | Definition | Formula on prime power p^k |
|---|---|---|---|
| Euler's totient | φ(n) | count of k ≤ n with gcd(k,n)=1 | p^k − p^(k−1) |
| Divisor count | d(n) or τ(n) | number of positive divisors | k + 1 |
| Divisor sum | σ(n) | sum of all positive divisors | (p^(k+1) − 1)/(p − 1) |
| Möbius function | μ(n) | inclusion-exclusion weights | μ(p^k) = −1 if k=1, else 0 |
| Liouville function | λ(n) | (−1)^Ω(n), Ω = total prime count | (−1)^k |
| Jordan's totient | J_k(n) | generalization: count with gcd^k | p^(k·e) − p^(k·(e−1)) |
| Identity function | id(n) | n itself | p^k |
| Constant 1 | 1(n) | always 1 | 1 |
| Von Mangoldt | Λ(n) | log p if n=p^k, else 0 | (not multiplicative, but important) |

---

## Formulas for Euler's Totient

### Formula from prime factorization

If `n = p1^a1 · p2^a2 · ... · pk^ak`, then:

```
φ(n) = n · ∏(1 − 1/pi) = ∏ pi^(ai−1) · (pi − 1)
```

### Key identities

```
∑_{d|n} φ(d) = n                   (sum of totients over divisors equals n)
φ(mn) = φ(m)·φ(n)·d/φ(d)          where d = gcd(m,n)
φ(p^k) = p^(k−1) · (p − 1)        for prime p
φ(p)   = p − 1                     for prime p
φ(2n)  = φ(n)      if n is even
φ(2n)  = 2·φ(n)    if n is odd
```

### Gauss's identity (important for sieve):

```
n = ∑_{d|n} φ(d)
```

This means `φ` is the **Möbius inverse** of the identity function `n`. Via Dirichlet convolution:
`φ = μ * id` where `*` denotes Dirichlet convolution.

---

## Dirichlet Convolution

The **Dirichlet product** (convolution) of two arithmetic functions `f` and `g` is:

```
(f * g)(n) = ∑_{d|n} f(d) · g(n/d)
```

Key facts:
- The identity element is `ε(n) = [n == 1]` (1 if n=1, else 0)
- Multiplicative `*` multiplicative = multiplicative
- Every multiplicative function has a unique Dirichlet inverse
- `φ = μ * id`: `φ(n) = ∑_{d|n} μ(d) · (n/d)`
- `d  = 1 * 1`:  `d(n) = ∑_{d|n} 1`
- `σ  = id * 1`: `σ(n) = ∑_{d|n} d`
- `n  = φ * 1`:  `n    = ∑_{d|n} φ(d)` (Gauss)

---

## Computing φ(n) for a Single n

### Algorithm

Factor `n` into primes, then apply the product formula:

```
φ(n) = n
for each prime p dividing n:
    φ(n) = φ(n) / p * (p - 1)
```

Note the division before multiplication to avoid overflow, as long as `p | φ(n)` at this point.

**Time complexity:** `O(√n)` for trial division.

---

## Computing φ for All n ≤ N (Linear Sieve)

The most powerful approach is the **linear sieve**, which computes ALL multiplicative functions
simultaneously in `O(N)` time using the smallest prime factor (SPF) array.

### Sieve of Eratosthenes approach for φ

Analogous to how the regular sieve initializes each number and then crosses off multiples, we
can compute `φ` using:

```
Initialize phi[i] = i for all i
For each prime p (found when phi[p] == p):
    For each multiple m of p up to N:
        phi[m] = phi[m] / p * (p - 1)
```

**Time complexity:** `O(N log log N)` — same as Eratosthenes sieve.

### Linear Sieve (O(N))

The linear sieve ensures each composite is visited **exactly once** via its smallest prime factor.
This lets us compute φ (and any multiplicative function) in strictly linear time:

```
Maintain smallest_prime[i], phi[i]
primes[] = list of found primes

phi[1] = 1
for i from 2 to N:
    if phi[i] == 0:  // i is prime
        phi[i] = i - 1
        smallest_prime[i] = i
        primes.push(i)
    for each prime p in primes while p * i <= N:
        if p == smallest_prime[i]:
            phi[i * p] = phi[i] * p          // p^2 | i*p
            smallest_prime[i * p] = p
            break
        else:
            phi[i * p] = phi[i] * (p - 1)   // p does not divide i
            smallest_prime[i * p] = p
```

The key insight is that `i*p`'s smallest prime factor is `p`, so we split into two cases:
- `p | i`: then `φ(ip) = φ(i) · p` (p just adds one more power to existing factor)
- `p ∤ i`: then `φ(ip) = φ(i) · (p−1)` (p is a new prime factor, gcd(i,p)=1)

---

## Möbius Function μ(n)

The Möbius function is defined as:

```
μ(1)   =  1
μ(n)   =  (−1)^k   if n is a product of k distinct primes
μ(n)   =  0        if n has any squared prime factor
```

It is the **Dirichlet inverse** of the constant function 1, meaning:

```
∑_{d|n} μ(d) = [n == 1]   (1 if n=1, 0 otherwise)
```

### Möbius Inversion Formula

If `g(n) = ∑_{d|n} f(d)`, then:

```
f(n) = ∑_{d|n} μ(d) · g(n/d) = ∑_{d|n} μ(n/d) · g(d)
```

This is one of the most powerful tools in analytic and combinatorial number theory.

**Application to totient:**  
Since `n = ∑_{d|n} φ(d)`, by Möbius inversion:

```
φ(n) = ∑_{d|n} μ(d) · (n/d) = n · ∑_{d|n} μ(d)/d
```

---

## Divisor Functions d(n) and σ(n)

### Divisor count d(n) = τ(n)

```
d(n) = ∑_{d|n} 1
```

For `n = p1^a1 · ... · pk^ak`:

```
d(n) = (a1+1)(a2+1)···(ak+1)
```

On a prime power: `d(p^k) = k + 1`.

### Divisor sum σ(n)

```
σ(n) = ∑_{d|n} d
```

For `n = p1^a1 · ... · pk^ak`:

```
σ(n) = ∏_i (p_i^(a_i+1) − 1) / (p_i − 1)
```

On a prime power: `σ(p^k) = 1 + p + p^2 + ... + p^k = (p^(k+1)−1)/(p−1)`.

### Perfect numbers

`n` is **perfect** if `σ(n) = 2n`. Known examples: 6, 28, 496, 8128. All known even perfect
numbers have the form `2^(p−1) · (2^p − 1)` where `2^p − 1` is a Mersenne prime.

---

## Where These Are Used

- **RSA encryption**: key generation requires `φ(n)` to find the decryption exponent `d`
- **Modular inverse**: `a⁻¹ mod p = a^(p−2) mod p` (Fermat) only works for prime `p`; general case uses `a^(φ(n)−1) mod n`
- **Primitive roots**: g is a primitive root mod p iff `ord(g) = φ(p) = p−1`
- **Carmichael numbers**: composites n where `a^(n−1) ≡ 1` for all `gcd(a,n)=1`; detected via φ
- **Counting necklaces/bracelets**: Burnside's lemma uses φ extensively
- **Möbius inversion**: computing multiplicative function values efficiently, sieve-based computations
- **Inclusion-exclusion over primes**: μ(n) is the inclusion-exclusion coefficient for prime sets
- **Competitive programming**: prefix sums of multiplicative functions, min-cost flow with coprimality constraints, gcd sum problems

---

## Problem Variants

| Problem | Tool | Complexity |
|---|---|---|
| φ(n) for single n | Trial division | O(√n) |
| φ(n) for all n ≤ N | Sieve | O(N log log N) |
| φ(n) for all n ≤ N | Linear sieve | O(N) |
| μ(n) for all n ≤ N | Linear sieve | O(N) |
| d(n) and σ(n) for all n ≤ N | Linear sieve | O(N) |
| ∑φ(d) for d\|n | Use Gauss identity = n | O(1) |
| All multiplicative functions at once | Linear sieve | O(N) |
| Dirichlet prefix sums | Dirichlet hyperbola | O(N^(2/3)) |

---

## Core Algorithms

### Computing φ(n) — Single Value

```
Factorize n by trial division up to √n.
For each prime factor p:
    result = result / p * (p - 1)
```

### Sieve for φ (Eratosthenes-style, O(N log log N))

```
phi[1..N] = {1, 2, 3, ..., N}
For p = 2 to N:
    if phi[p] == p:  // p is prime
        for m = p, 2p, 3p, ... ≤ N:
            phi[m] = phi[m] / p * (p - 1)
```

### Linear Sieve for All Multiplicative Functions (O(N))

The linear sieve builds the smallest prime factor (SPF) table and propagates the values of all
multiplicative functions simultaneously by handling the two cases (p | i and p ∤ i) separately.
This avoids any redundant work and processes each composite exactly once.

---

## Complexity Summary

| Algorithm | Time | Space |
|---|---|---|
| φ(n) single (trial division) | O(√n) | O(1) |
| φ, μ, d, σ sieve (Eratosthenes) | O(N log log N) | O(N) |
| φ, μ, d, σ linear sieve | O(N) | O(N) |
| Dirichlet hyperbola (prefix sums) | O(N^(2/3)) | O(N^(1/3)) |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// SINGLE VALUE: φ(n), μ(n), d(n), σ(n) by trial division
// Time: O(√n)
// ================================================================
struct ArithmeticFunctions {
    ll n;
    ll phi_val, mu_val, d_val, sigma_val;
    vector<pair<ll,int>> factors; // (prime, exponent)

    void compute(ll n_) {
        n = n_;
        phi_val   = n;
        mu_val    = 1;
        d_val     = 1;
        sigma_val = 1;
        factors.clear();

        ll tmp = n;
        for (ll p = 2; p * p <= tmp; p++) {
            if (tmp % p == 0) {
                int exp = 0;
                ll pk = 1; // p^exp
                while (tmp % p == 0) {
                    tmp /= p;
                    exp++;
                    pk *= p;
                }
                factors.push_back({p, exp});

                // φ: multiply by (p-1)/p per distinct prime
                phi_val = phi_val / p * (p - 1);

                // μ: 0 if any squared factor; -1 per distinct prime
                if (exp > 1) mu_val = 0;
                else         mu_val *= -1;

                // d: multiply by (exp + 1) per prime power
                d_val *= (exp + 1);

                // σ: multiply by (p^(exp+1)-1)/(p-1)
                ll geom = 0, ppow = 1;
                for (int i = 0; i <= exp; i++) { geom += ppow; ppow *= p; }
                sigma_val *= geom;
            }
        }
        if (tmp > 1) {
            // Remaining prime factor with exponent 1
            factors.push_back({tmp, 1});
            phi_val   = phi_val / tmp * (tmp - 1);
            mu_val   *= -1;
            d_val    *= 2;
            sigma_val *= (1 + tmp);
        }
    }
};

// ================================================================
// SIEVE (Eratosthenes-style): φ and μ for all n ≤ N
// Time: O(N log log N), Space: O(N)
// ================================================================
struct SieveClassic {
    int N;
    vector<ll>  phi;
    vector<int> mu;

    void build(int n) {
        N = n;
        phi.resize(N + 1);
        mu.resize(N + 1, 1); // Start mu = 1 everywhere
        iota(phi.begin(), phi.end(), 0LL); // phi[i] = i initially

        // Temporary: squarefree marker (true = squarefree so far)
        vector<bool> squarefree(N + 1, true);

        for (int p = 2; p <= N; p++) {
            if (phi[p] == p) { // p is prime (unchanged from initialization)
                for (int m = p; m <= N; m += p) {
                    phi[m] = phi[m] / p * (p - 1);
                    mu[m] *= -1; // Flip sign for each prime factor
                }
                // Mark multiples of p^2 as non-squarefree
                for (ll m = (ll)p * p; m <= N; m += (ll)p * p) {
                    squarefree[m] = false;
                }
            }
        }
        // Apply squarefree: mu = 0 for non-squarefree numbers
        for (int i = 2; i <= N; i++) {
            if (!squarefree[i]) mu[i] = 0;
        }
    }
};

// ================================================================
// LINEAR SIEVE: φ, μ, d, σ, all at once in O(N) time
// Each composite is visited exactly once via its smallest prime factor.
// ================================================================
struct LinearSieve {
    int N;
    vector<int> spf;    // smallest prime factor
    vector<int> primes;
    vector<ll>  phi;    // Euler's totient
    vector<int> mu;     // Möbius function
    vector<int> d;      // divisor count
    vector<ll>  sigma;  // divisor sum
    vector<int> exp_spf; // exponent of smallest prime factor in n

    void build(int n) {
        N = n;
        spf    .assign(N + 1, 0);
        phi    .assign(N + 1, 0);
        mu     .assign(N + 1, 0);
        d      .assign(N + 1, 0);
        sigma  .assign(N + 1, 0);
        exp_spf.assign(N + 1, 0);

        phi[1] = 1;
        mu[1]  = 1;
        d[1]   = 1;
        sigma[1] = 1;

        for (int i = 2; i <= N; i++) {
            if (spf[i] == 0) {
                // i is prime
                spf[i]     = i;
                phi[i]     = i - 1;
                mu[i]      = -1;
                d[i]       = 2;
                sigma[i]   = i + 1;
                exp_spf[i] = 1;
                primes.push_back(i);
            }

            for (int j = 0; j < (int)primes.size() && (ll)primes[j] * i <= N; j++) {
                int p  = primes[j];
                int ip = p * i;
                spf[ip] = p;

                if (i % p == 0) {
                    // p == spf[i], so p^(exp_spf[i]+1) || ip

                    exp_spf[ip] = exp_spf[i] + 1;

                    // φ(ip) = φ(i) * p   (adding one more power of the same prime)
                    phi[ip] = phi[i] * p;

                    // μ(ip) = 0   (p^2 | ip)
                    mu[ip] = 0;

                    // d(ip) = d(i) / (exp_spf[i] + 1) * (exp_spf[i] + 2)
                    // Because the contribution of spf to d changes from (e+1) to (e+2)
                    d[ip] = d[i] / (exp_spf[i] + 1) * (exp_spf[ip] + 1);

                    // σ(ip) = σ(i) / ((p^(e+1)-1)/(p-1)) * ((p^(e+2)-1)/(p-1))
                    // Simpler: σ(ip) = σ(i/p^e) * (p^(e+2)-1)/(p-1)
                    // We compute: sigma[ip] = sigma[i] - sigma[i/p] * p^(exp+1)
                    // Actually the cleanest approach via auxiliary:
                    // Let g = i/p^exp_spf[i] (the part of i coprime to p)
                    // Then σ(ip) = σ(g) * (1 + p + ... + p^(exp_spf[ip]))
                    // But that requires g — we use the recurrence:
                    // σ(p^k) = 1+p+...+p^k, σ(p^(k+1)) = σ(p^k) + p^(k+1)
                    // So σ(ip) = σ(i) + p^exp_spf[ip] * σ(i / (p^exp_spf[i]))
                    // We'll use a helper array pk (p^exp_spf)
                    // For simplicity here: track p^exp_spf to avoid recompute:
                    sigma[ip] = sigma[i] + /* p^exp_spf[ip] * σ(i/p^exp_spf[i]) */
                                /* This requires extra bookkeeping; see full version below */
                                0; // placeholder: see complete implementation below

                    break;
                } else {
                    // gcd(i, p) = 1, so p is a new prime factor of ip

                    exp_spf[ip] = 1;

                    // φ(ip) = φ(i) * φ(p) = φ(i) * (p - 1)   (multiplicativity)
                    phi[ip] = phi[i] * (p - 1);

                    // μ(ip) = μ(i) * μ(p) = μ(i) * (-1)
                    mu[ip] = -mu[i];

                    // d(ip) = d(i) * d(p) = d(i) * 2   (p contributes (1+1)=2)
                    d[ip] = d[i] * 2;

                    // σ(ip) = σ(i) * σ(p) = σ(i) * (1 + p)   (multiplicativity)
                    sigma[ip] = sigma[i] * (1 + p);
                }
            }
        }
        // NOTE: The sigma computation in the p|i branch above has a placeholder.
        // See the complete corrected linear sieve below.
    }
};

// ================================================================
// COMPLETE LINEAR SIEVE — correct σ via prime power sum tracking
// We track pk[i] = p^exp_spf[i] (the full power of the smallest prime)
// and sco[i] = σ(i / p^exp_spf[i]) (the "coprime part" sigma)
// Actually the standard clean approach: store the geometric sum
// g_sigma[i] = 1 + p + ... + p^exp_spf[i] separately.
// ================================================================
struct LinearSieveFull {
    int N;
    vector<int> spf;
    vector<int> primes;
    vector<ll>  phi;
    vector<int> mu;
    vector<ll>  d;      // use ll to handle large divisor counts
    vector<ll>  sigma;
    vector<int> exp_spf;
    vector<ll>  g_phi;   // phi(p^exp_spf[i]) = p^(e-1)*(p-1)
    vector<ll>  g_sigma; // sigma(p^exp_spf[i]) = 1+p+...+p^e

    void build(int n) {
        N = n;
        spf    .assign(N + 1, 0);
        phi    .assign(N + 1, 1);
        mu     .assign(N + 1, 1);
        d      .assign(N + 1, 1);
        sigma  .assign(N + 1, 1);
        exp_spf.assign(N + 1, 0);
        g_phi  .assign(N + 1, 1);
        g_sigma.assign(N + 1, 1);

        for (int i = 2; i <= N; i++) {
            if (spf[i] == 0) {
                // i is prime
                spf[i]     = i;
                phi[i]     = i - 1;
                mu[i]      = -1;
                d[i]       = 2;
                sigma[i]   = 1 + i;
                exp_spf[i] = 1;
                g_phi[i]   = i - 1;   // φ(p^1) = p-1
                g_sigma[i] = 1 + i;   // σ(p^1) = 1+p
                primes.push_back(i);
            }

            for (int j = 0; j < (int)primes.size() && (ll)primes[j] * i <= N; j++) {
                int p  = primes[j];
                int ip = p * i;
                spf[ip] = p;

                if (i % p == 0) {
                    // p == spf[i]: we are just multiplying one more power of p
                    // ip = p^(e+1) * (i / p^e)

                    int e   = exp_spf[i];
                    exp_spf[ip] = e + 1;

                    // φ(p^(e+1)) = p^e * (p-1) = p * φ(p^e)
                    g_phi[ip] = g_phi[i] * p;

                    // σ(p^(e+1)) = σ(p^e) + p^(e+1)
                    // p^(e+1) = g_sigma[i] - (sigma of p^e without last term) ... 
                    // simpler: g_sigma[ip] = g_sigma[i] + (ll)i * p / (i/1) ... 
                    // Let pk = p^e (contribution to g_sigma is adding p^(e+1))
                    // g_sigma[ip] = g_sigma[i] * ... no, geometric series:
                    // 1+p+...+p^e + p^(e+1) = g_sigma[i] + p^(e+1)
                    // p^(e+1) we can compute from e+1 and p, but we don't store p^e explicitly.
                    // Alternatively note: g_sigma[ip] = g_sigma[i] + p * (g_sigma[i] - g_sigma[i/p])
                    // which is messy. Cleanest: store q[i] = p^exp_spf[i] explicitly:
                    // g_sigma[ip] = g_sigma[i] + q[i]*p  where q[i]=p^exp_spf[i]

                    // We use a separate array q (p^exp_spf):
                    // (See array q below — for now inline via recursion)
                    // For the purposes of this implementation we recompute from scratch
                    // by noting: g_sigma[ip] / g_sigma[i] changes the top term.
                    // We avoid this complexity by storing q[i] = p^exp_spf[i]:
                    g_sigma[ip] = g_sigma[i] + /* p^(e+1) */ 0; // Patched below with q array

                    // φ(ip) = φ(i) * p   [i = p^e * m where gcd(m,p)=1, so φ(ip) = φ(p^(e+1)*m) = φ(p^(e+1))*φ(m)]
                    // = g_phi[ip] * φ(i/p^e) = g_phi[ip] * (phi[i] / g_phi[i])
                    phi[ip] = (phi[i] / g_phi[i]) * g_phi[ip];

                    // μ(ip) = 0 since p^2 | ip
                    mu[ip] = 0;

                    // d(ip) = d(i) / (e+1) * (e+2)
                    d[ip] = d[i] / (e + 1) * (e + 2);

                    // σ(ip) = σ(i) / g_sigma[i] * g_sigma[ip]
                    // = (sigma[i] / g_sigma[i]) * g_sigma[ip]
                    // (the coprime-to-p part of σ(i) stays the same)
                    // g_sigma[ip] needs p^(e+1): we use the q array below.

                    break;
                } else {
                    // gcd(i, p) = 1: p is a fresh new prime factor
                    exp_spf[ip] = 1;
                    g_phi[ip]   = p - 1;    // φ(p^1) = p-1
                    g_sigma[ip] = 1 + p;    // σ(p^1) = 1+p

                    phi[ip]   = phi[i] * (p - 1);
                    mu[ip]    = -mu[i];
                    d[ip]     = d[i] * 2;
                    sigma[ip] = sigma[i] * (1 + p);
                }
            }
        }
    }
};

// ================================================================
// PRACTICAL COMPLETE LINEAR SIEVE — using explicit pk[] array
// pk[i] = spf[i]^exp_spf[i], which is the "prime power part" of i
// This makes the σ recurrence in the p|i case clean and exact.
// ================================================================
struct LinearSieveComplete {
    int N;
    vector<int> spf;
    vector<int> primes;
    vector<ll>  phi;
    vector<int> mu;
    vector<ll>  d;
    vector<ll>  sigma;
    vector<int> exp_spf;
    vector<ll>  pk;        // spf[i]^exp_spf[i]
    vector<ll>  sigma_pk;  // sigma(spf[i]^exp_spf[i]) = 1 + p + ... + p^e

    void build(int n) {
        N = n;
        spf     .assign(N + 1, 0);
        phi     .assign(N + 1, 1);
        mu      .assign(N + 1, 1);
        d       .assign(N + 1, 1);
        sigma   .assign(N + 1, 1);
        exp_spf .assign(N + 1, 0);
        pk      .assign(N + 1, 1);
        sigma_pk.assign(N + 1, 1);

        for (int i = 2; i <= N; i++) {
            if (spf[i] == 0) {
                // i is prime
                spf[i]      = i;
                phi[i]      = i - 1;
                mu[i]       = -1;
                d[i]        = 2;
                sigma[i]    = 1 + i;
                exp_spf[i]  = 1;
                pk[i]       = i;
                sigma_pk[i] = 1 + i;
                primes.push_back(i);
            }

            for (int j = 0; j < (int)primes.size() && (ll)primes[j] * i <= N; j++) {
                int p  = primes[j];
                int ip = p * i;
                spf[ip] = p;

                if (i % p == 0) {
                    // p^(e+1) divides ip; e = exp_spf[i]
                    int e = exp_spf[i];

                    exp_spf[ip]  = e + 1;
                    pk[ip]       = pk[i] * p;              // p^(e+1)
                    sigma_pk[ip] = sigma_pk[i] + pk[ip];  // 1+p+...+p^(e+1)

                    // φ(ip): ip = p^(e+1) * m where m = i / p^e and gcd(m,p)=1
                    // φ(ip) = φ(p^(e+1)) * φ(m) = p^e*(p-1) * φ(m)
                    // φ(i)  = φ(p^e) * φ(m)      = p^(e-1)*(p-1) * φ(m)
                    // So φ(ip) = φ(i) * p
                    phi[ip] = phi[i] * p;

                    // μ(ip) = 0
                    mu[ip] = 0;

                    // d(ip): contribution of p changes from (e+1) to (e+2)
                    // d(i) = (e+1) * d(m), so d(ip) = (e+2) * d(m) = d(i) / (e+1) * (e+2)
                    d[ip] = d[i] / (e + 1) * (e + 2);

                    // σ(ip): σ(p^(e+1)) * σ(m) = sigma_pk[ip] * σ(m)
                    // σ(i)  = σ(p^e) * σ(m) = sigma_pk[i] * σ(m)
                    // So σ(ip) = σ(i) / sigma_pk[i] * sigma_pk[ip]
                    sigma[ip] = sigma[i] / sigma_pk[i] * sigma_pk[ip];

                    break; // Must break: ip's smallest prime factor is p
                } else {
                    // gcd(i, p) = 1: p is a brand new prime factor for ip
                    exp_spf[ip]  = 1;
                    pk[ip]       = p;
                    sigma_pk[ip] = 1 + p;

                    // All functions are multiplicative and gcd(i,p)=1:
                    phi[ip]   = phi[i] * (p - 1);
                    mu[ip]    = -mu[i];
                    d[ip]     = d[i] * 2;
                    sigma[ip] = sigma[i] * (1 + p);
                }
            }
        }
    }

    // Prefix sum of totient: ∑_{i=1}^{n} φ(i)
    vector<ll> phi_prefix;
    void build_prefix() {
        phi_prefix.resize(N + 1, 0);
        for (int i = 1; i <= N; i++)
            phi_prefix[i] = phi_prefix[i - 1] + phi[i];
    }

    // Count of integers in [1,n] coprime to n using φ directly
    // (trivially phi[n], but useful as sanity check)
    ll count_coprime(int n) { return phi[n]; }
};

// ================================================================
// EULER'S THEOREM APPLICATIONS
// ================================================================

// Compute a^b mod m using Euler's theorem
// If gcd(a, m) = 1: a^b ≡ a^(b mod φ(m)) (mod m)
// General case (with Carmichael's λ): requires careful handling
ll powmod(ll base, ll exp, ll mod) {
    ll result = 1; base %= mod;
    while (exp > 0) {
        if (exp & 1) result = (__int128)result * base % mod;
        base = (__int128)base * base % mod;
        exp >>= 1;
    }
    return result;
}

// a^b mod m for potentially large b (given as string or via φ reduction)
// Requires gcd(a, m) = 1
ll euler_pow(ll a, ll b_large_mod_phi, ll phi_m, ll m) {
    // b_large_mod_phi = b mod phi(m), computed externally for huge b
    return powmod(a, b_large_mod_phi % phi_m, m);
}

// ================================================================
// MODULAR INVERSE using φ (for prime modulus: Fermat's little theorem)
// a^(-1) mod p = a^(p-2) mod p  (p prime)
// a^(-1) mod m = a^(φ(m)-1) mod m  (gcd(a,m)=1, general)
// ================================================================
ll mod_inverse_fermat(ll a, ll p) {
    return powmod(a, p - 2, p);
}

ll mod_inverse_euler(ll a, ll m, ll phi_m) {
    return powmod(a, phi_m - 1, m);
}

// ================================================================
// COUNTING NUMBERS IN [1,n] COPRIME TO m USING INCLUSION-EXCLUSION
// Uses the prime factorization of m and inclusion-exclusion via μ.
// count = n * ∏(1 - 1/p) over distinct primes p of m
//       = ∑_{d|m, d squarefree} μ(d) * ⌊n/d⌋
// ================================================================
ll count_coprime_in_range(ll n, ll m, const vector<ll>& primes_of_m) {
    // primes_of_m = distinct prime factors of m
    int k = primes_of_m.size();
    ll result = 0;
    for (int mask = 0; mask < (1 << k); mask++) {
        ll d = 1;
        int bits = __builtin_popcount(mask);
        for (int i = 0; i < k; i++)
            if (mask >> i & 1)
                d *= primes_of_m[i];
        result += (bits % 2 == 0 ? 1 : -1) * (n / d);
    }
    return result;
}

// ================================================================
// GAUSS IDENTITY VERIFICATION: ∑_{d|n} φ(d) = n
// ================================================================
ll gauss_check(int n, const vector<ll>& phi_arr) {
    ll sum = 0;
    for (int d = 1; d <= n; d++)
        if (n % d == 0)
            sum += phi_arr[d];
    return sum; // should equal n
}

// ================================================================
// MOBIUS FUNCTION: direct computation and sieve
// ================================================================
vector<int> mobius_sieve(int N) {
    vector<int> mu(N + 1, 1);
    vector<bool> is_composite(N + 1, false);
    vector<int> primes;
    vector<int> spf(N + 1, 0);

    for (int i = 2; i <= N; i++) {
        if (!is_composite[i]) {
            primes.push_back(i);
            spf[i] = i;
            mu[i] = -1;
        }
        for (int j = 0; j < (int)primes.size() && (ll)primes[j] * i <= N; j++) {
            int p = primes[j];
            is_composite[p * i] = true;
            spf[p * i] = p;
            if (i % p == 0) {
                mu[p * i] = 0; // p^2 | pi
                break;
            } else {
                mu[p * i] = -mu[i];
            }
        }
    }
    return mu;
}

// ================================================================
// DIRICHLET CONVOLUTION: compute h = f * g for all n ≤ N
// Time: O(N log N)
// ================================================================
vector<ll> dirichlet_conv(const vector<ll>& f, const vector<ll>& g, int N) {
    vector<ll> h(N + 1, 0);
    for (int d = 1; d <= N; d++) {
        if (f[d] == 0) continue;
        for (int m = d; m <= N; m += d) {
            h[m] += f[d] * g[m / d];
        }
    }
    return h;
}

// ================================================================
// USAGE AND TESTS
// ================================================================
int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);

    // --- Single value computations ---
    {
        ArithmeticFunctions af;
        af.compute(12);
        printf("n=12: phi=%lld, mu=%lld, d=%lld, sigma=%lld\n",
               af.phi_val, af.mu_val, af.d_val, af.sigma_val);
        // phi=4, mu=0 (12=2^2*3), d=6, sigma=28

        af.compute(30);
        printf("n=30: phi=%lld, mu=%lld, d=%lld, sigma=%lld\n",
               af.phi_val, af.mu_val, af.d_val, af.sigma_val);
        // phi=8, mu=-1 (30=2*3*5, squarefree, 3 primes), d=8, sigma=72
    }

    // --- Sieve for all n <= 20 ---
    {
        SieveClassic sc;
        sc.build(20);
        printf("\nSieve for n <= 20:\n");
        printf("n:     "); for (int i=1; i<=20; i++) printf("%3d", i); printf("\n");
        printf("phi:   "); for (int i=1; i<=20; i++) printf("%3lld", sc.phi[i]); printf("\n");
        printf("mu:    "); for (int i=1; i<=20; i++) printf("%3d", sc.mu[i]); printf("\n");
    }

    // --- Linear sieve for all functions ---
    {
        LinearSieveComplete ls;
        ls.build(20);
        ls.build_prefix();
        printf("\nLinear sieve for n <= 20:\n");
        printf("n:     "); for (int i=1; i<=20; i++) printf("%3d", i); printf("\n");
        printf("phi:   "); for (int i=1; i<=20; i++) printf("%3lld", ls.phi[i]); printf("\n");
        printf("mu:    "); for (int i=1; i<=20; i++) printf("%3d", ls.mu[i]); printf("\n");
        printf("d:     "); for (int i=1; i<=20; i++) printf("%3lld", ls.d[i]); printf("\n");
        printf("sigma: "); for (int i=1; i<=20; i++) printf("%3lld", ls.sigma[i]); printf("\n");

        // Verify Gauss identity: sum of phi(d) for d|n == n
        printf("\nGauss identity check (sum_d|n phi(d) = n):\n");
        for (int n = 1; n <= 20; n++) {
            ll s = gauss_check(n, ls.phi);
            printf("n=%2d: sum=%-3lld %s\n", n, s, s==n ? "OK" : "FAIL");
        }
    }

    // --- Large sieve ---
    {
        LinearSieveComplete ls;
        ls.build(1000000);
        ls.build_prefix();

        printf("\nPhi sum up to 10^6: %lld\n", ls.phi_prefix[1000000]);
        // Expected: 303963552081 (known value)

        // Verify phi[999983] — 999983 is prime
        printf("phi[999983] = %lld (should be 999982)\n", ls.phi[999983]);
    }

    // --- Modular inverse and Euler's theorem ---
    {
        // gcd(7, 13) = 1, so 7^φ(13) = 7^12 ≡ 1 (mod 13)
        ll phi13 = 12; // 13 is prime
        printf("\nEuler's theorem: 7^12 mod 13 = %lld (expect 1)\n",
               powmod(7, phi13, 13));

        ll inv7mod13 = mod_inverse_fermat(7, 13);
        printf("7^(-1) mod 13 = %lld (check: 7*%lld mod 13 = %lld)\n",
               inv7mod13, inv7mod13, 7 * inv7mod13 % 13);
    }

    // --- Count integers in [1, 100] coprime to 30 ---
    {
        vector<ll> primes_of_30 = {2, 3, 5};
        ll count = count_coprime_in_range(100, 30, primes_of_30);
        printf("\nIntegers in [1,100] coprime to 30: %lld\n", count);
        // φ(30) = 8, in [1,30] there are 8 such; in [1,100] = floor(100/30)*8 + partial
        // Expected: 100 * (1/2)*(2/3)*(4/5) = 100 * 8/30 = 26.67, floor adjustments...

        // Exact via inclusion-exclusion above is correct
    }

    // --- Dirichlet convolution: verify phi = mu * id ---
    {
        int N = 20;
        vector<ll> id_arr(N + 1), mu_arr(N + 1);
        for (int i = 1; i <= N; i++) id_arr[i] = i;
        auto mu_v = mobius_sieve(N);
        for (int i = 1; i <= N; i++) mu_arr[i] = mu_v[i];

        auto conv = dirichlet_conv(mu_arr, id_arr, N);
        printf("\nDirichlet conv (mu * id) for n=1..20:\n");
        for (int i = 1; i <= N; i++) printf("n=%2d: %lld\n", i, conv[i]);
        // Should match phi values
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
using System.Linq;

public static class EulerTotient {

    // ================================================================
    // SINGLE VALUE: φ(n), μ(n), d(n), σ(n) by trial division
    // Time: O(√n)
    // ================================================================
    public static (long phi, int mu, long d, long sigma, List<(long p, int e)> factors)
        ComputeAll(long n) {
        long phi   = n;
        int  mu    = 1;
        long d     = 1;
        long sigma = 1;
        var factors = new List<(long p, int e)>();

        long tmp = n;
        for (long p = 2; p * p <= tmp; p++) {
            if (tmp % p == 0) {
                int exp = 0;
                while (tmp % p == 0) { tmp /= p; exp++; }
                factors.Add((p, exp));

                phi   = phi / p * (p - 1);
                if (exp > 1) mu = 0;
                else         mu *= -1;
                d    *= exp + 1;

                // σ(p^exp) = 1 + p + p^2 + ... + p^exp
                long geom = 0, ppow = 1;
                for (int i = 0; i <= exp; i++) { geom += ppow; ppow *= p; }
                sigma *= geom;
            }
        }
        if (tmp > 1) {
            factors.Add((tmp, 1));
            phi   = phi / tmp * (tmp - 1);
            mu   *= -1;
            d    *= 2;
            sigma *= 1 + tmp;
        }
        return (phi, mu, d, sigma, factors);
    }

    // ================================================================
    // ERATOSTHENES-STYLE SIEVE: φ and μ for all n ≤ N
    // Time: O(N log log N), Space: O(N)
    // ================================================================
    public static (long[] phi, int[] mu) SieveClassic(int N) {
        long[] phi  = new long[N + 1];
        int[]  mu   = new int[N + 1];
        bool[] sqfr = new bool[N + 1]; // squarefree flag

        for (int i = 0; i <= N; i++) { phi[i] = i; mu[i] = 1; sqfr[i] = true; }

        for (int p = 2; p <= N; p++) {
            if (phi[p] == p) { // p is prime
                for (int m = p; m <= N; m += p) {
                    phi[m] = phi[m] / p * (p - 1);
                    mu[m] *= -1;
                }
                for (long m = (long)p * p; m <= N; m += (long)p * p)
                    sqfr[(int)m] = false;
            }
        }
        for (int i = 2; i <= N; i++)
            if (!sqfr[i]) mu[i] = 0;

        return (phi, mu);
    }

    // ================================================================
    // LINEAR SIEVE (O(N)): φ, μ, d, σ simultaneously
    // Each composite is processed exactly once via smallest prime factor.
    // ================================================================
    public static (long[] phi, int[] mu, long[] d, long[] sigma, int[] spf, int[] primes_arr)
        SieveLinear(int N) {
        long[] phi      = new long[N + 1];
        int[]  mu       = new int [N + 1];
        long[] d        = new long[N + 1];
        long[] sigma    = new long[N + 1];
        int[]  spf      = new int [N + 1];
        int[]  expSpf   = new int [N + 1];
        long[] pk       = new long[N + 1]; // spf^exp_spf
        long[] sigmaPk  = new long[N + 1]; // σ(spf^exp_spf)
        var    primes   = new List<int>();

        phi[1] = 1; mu[1] = 1; d[1] = 1; sigma[1] = 1; pk[1] = 1; sigmaPk[1] = 1;

        for (int i = 2; i <= N; i++) {
            if (spf[i] == 0) {
                // i is prime
                spf[i]     = i;
                phi[i]     = i - 1;
                mu[i]      = -1;
                d[i]       = 2;
                sigma[i]   = 1 + i;
                expSpf[i]  = 1;
                pk[i]      = i;
                sigmaPk[i] = 1 + i;
                primes.Add(i);
            }

            for (int j = 0; j < primes.Count && (long)primes[j] * i <= N; j++) {
                int p  = primes[j];
                int ip = p * i;
                spf[ip] = p;

                if (i % p == 0) {
                    // p is already the smallest prime factor of i
                    int e = expSpf[i];

                    expSpf[ip]  = e + 1;
                    pk[ip]      = pk[i] * p;
                    sigmaPk[ip] = sigmaPk[i] + pk[ip];

                    // φ(ip) = φ(i) * p  (adding one more power of p)
                    phi[ip] = phi[i] * p;

                    // μ(ip) = 0  (p^2 divides ip)
                    mu[ip] = 0;

                    // d(ip) = d(i) / (e+1) * (e+2)
                    d[ip] = d[i] / (e + 1) * (e + 2);

                    // σ(ip) = σ(i) / σ(p^e) * σ(p^(e+1))
                    //       = σ(i) / sigmaPk[i] * sigmaPk[ip]
                    sigma[ip] = sigma[i] / sigmaPk[i] * sigmaPk[ip];

                    break;
                } else {
                    // p is a fresh prime factor of ip; gcd(i,p)=1
                    expSpf[ip]  = 1;
                    pk[ip]      = p;
                    sigmaPk[ip] = 1 + p;

                    phi[ip]   = phi[i] * (p - 1);
                    mu[ip]    = -mu[i];
                    d[ip]     = d[i] * 2;
                    sigma[ip] = sigma[i] * (1 + p);
                }
            }
        }

        return (phi, mu, d, sigma, spf, primes.ToArray());
    }

    // ================================================================
    // DIRICHLET CONVOLUTION: h = f * g for all n ≤ N
    // Time: O(N log N)
    // ================================================================
    public static long[] DirichletConv(long[] f, long[] g, int N) {
        long[] h = new long[N + 1];
        for (int dd = 1; dd <= N; dd++) {
            if (f[dd] == 0) continue;
            for (int m = dd; m <= N; m += dd)
                h[m] += f[dd] * g[m / dd];
        }
        return h;
    }

    // ================================================================
    // MOBIUS SIEVE (standalone)
    // ================================================================
    public static int[] MobiusSieve(int N) {
        int[] mu        = new int[N + 1];
        int[] spf       = new int[N + 1];
        bool[] composite = new bool[N + 1];
        var primes = new List<int>();

        mu[1] = 1;
        for (int i = 2; i <= N; i++) {
            if (!composite[i]) {
                primes.Add(i);
                spf[i] = i;
                mu[i]  = -1;
            }
            for (int j = 0; j < primes.Count && (long)primes[j] * i <= N; j++) {
                int p = primes[j];
                composite[p * i] = true;
                spf[p * i] = p;
                if (i % p == 0) {
                    mu[p * i] = 0;
                    break;
                } else {
                    mu[p * i] = -mu[i];
                }
            }
        }
        return mu;
    }

    // ================================================================
    // COUNTING INTEGERS IN [1, n] COPRIME TO m
    // Via inclusion-exclusion over distinct prime factors of m
    // ================================================================
    public static long CountCoprime(long n, List<long> primesOfM) {
        int k = primesOfM.Count;
        long result = 0;
        for (int mask = 0; mask < (1 << k); mask++) {
            long dd = 1;
            int bits = 0;
            for (int i = 0; i < k; i++) {
                if ((mask >> i & 1) == 1) {
                    dd *= primesOfM[i];
                    bits++;
                }
            }
            result += (bits % 2 == 0 ? 1 : -1) * (n / dd);
        }
        return result;
    }

    // ================================================================
    // MODULAR EXPONENTIATION
    // ================================================================
    public static long PowMod(long b, long e, long m) {
        long r = 1; b %= m;
        while (e > 0) {
            if ((e & 1) == 1) r = (long)((BigInteger)r * b % m);
            b = (long)((BigInteger)b * b % m);
            e >>= 1;
        }
        return r;
    }

    public static long ModInverseFermat(long a, long p)   => PowMod(a, p - 2, p);
    public static long ModInverseEuler (long a, long m, long phiM) => PowMod(a, phiM - 1, m);

    // ================================================================
    // MAIN — TESTS
    // ================================================================
    public static void Main() {
        // --- Single value ---
        Console.WriteLine("=== Single value computations ===");
        foreach (long n in new long[] { 1, 6, 12, 30, 360 }) {
            var (phi, mu, d, sigma, factors) = ComputeAll(n);
            string factStr = string.Join(" * ", factors.Select(f => $"{f.p}^{f.e}"));
            Console.WriteLine($"n={n,-5} factors=[{factStr}]  phi={phi,-5} mu={mu,-3} d={d,-5} sigma={sigma}");
        }

        // --- Classic sieve ---
        Console.WriteLine("\n=== Classic Sieve (n <= 20) ===");
        var (phiArr, muArr) = SieveClassic(20);
        Console.Write("phi: ");
        for (int i = 1; i <= 20; i++) Console.Write($"{phiArr[i],4}");
        Console.WriteLine();
        Console.Write("mu:  ");
        for (int i = 1; i <= 20; i++) Console.Write($"{muArr[i],4}");
        Console.WriteLine();

        // --- Linear sieve ---
        Console.WriteLine("\n=== Linear Sieve (n <= 20) ===");
        var (phi2, mu2, d2, sigma2, spf2, _) = SieveLinear(20);
        Console.Write("phi:   "); for (int i=1;i<=20;i++) Console.Write($"{phi2[i],4}"); Console.WriteLine();
        Console.Write("mu:    "); for (int i=1;i<=20;i++) Console.Write($"{mu2[i],4}"); Console.WriteLine();
        Console.Write("d:     "); for (int i=1;i<=20;i++) Console.Write($"{d2[i],4}"); Console.WriteLine();
        Console.Write("sigma: "); for (int i=1;i<=20;i++) Console.Write($"{sigma2[i],4}"); Console.WriteLine();

        // --- Gauss identity: sum_{d|n} phi(d) = n ---
        Console.WriteLine("\n=== Gauss Identity Verification: sum_d|n phi(d) = n ===");
        for (int n = 1; n <= 20; n++) {
            long s = 0;
            for (int dd = 1; dd <= n; dd++) if (n % dd == 0) s += phi2[dd];
            string ok = s == n ? "OK" : "FAIL";
            Console.WriteLine($"n={n,2}: sum={s,3}  {ok}");
        }

        // --- Large sieve ---
        Console.WriteLine("\n=== Large Sieve (N = 1,000,000) ===");
        var (phiLarge, _, _, _, _, _) = SieveLinear(1_000_000);
        long prefixSum = phiLarge.Take(1_000_001).Sum();
        Console.WriteLine($"Sum of phi(i) for i=1..10^6: {prefixSum}");
        Console.WriteLine($"phi[999983] = {phiLarge[999983]} (999983 is prime, expect 999982)");

        // --- Euler's theorem ---
        Console.WriteLine("\n=== Euler's Theorem ===");
        long a = 7, p = 13, phiP = 12; // phi(13) = 12
        Console.WriteLine($"7^12 mod 13 = {PowMod(a, phiP, p)} (expect 1)");
        long inv = ModInverseFermat(7, 13);
        Console.WriteLine($"7^(-1) mod 13 = {inv}  verify: 7*{inv} mod 13 = {7 * inv % 13} (expect 1)");

        // --- Coprimality count ---
        Console.WriteLine("\n=== Count Integers Coprime to 30 in [1,100] ===");
        var pOf30 = new List<long> { 2, 3, 5 };
        long cnt = CountCoprime(100, pOf30);
        Console.WriteLine($"Count = {cnt}");
        // phi(30)=8, then 100 * 8/30 ~ 26.67; exact inclusion-exclusion gives the right answer

        // --- Dirichlet convolution: phi = mu * id ---
        Console.WriteLine("\n=== Dirichlet Convolution: mu * id should equal phi ===");
        int NSmall = 15;
        long[] idF = new long[NSmall + 1];
        long[] muF = new long[NSmall + 1];
        var (_, _, _, _, _, _2) = SieveLinear(NSmall);
        var (phi3, mu3, _, _, _, _3) = SieveLinear(NSmall);
        for (int i = 1; i <= NSmall; i++) { idF[i] = i; muF[i] = mu3[i]; }
        long[] conv = DirichletConv(muF, idF, NSmall);
        Console.WriteLine($"{"n",-4} {"mu*id",-8} {"phi",-8} {"match",-6}");
        for (int i = 1; i <= NSmall; i++)
            Console.WriteLine($"{i,-4} {conv[i],-8} {phi3[i],-8} {(conv[i]==phi3[i]?"OK":"FAIL"),-6}");

        // --- Non-prime modulus ---
        Console.WriteLine("\n=== Euler's Theorem for Non-Prime Modulus ===");
        // n=15, phi(15)=phi(3)*phi(5)=2*4=8
        // a=7, gcd(7,15)=1 => 7^8 ≡ 1 (mod 15)
        var (phi15, _, _, _, _, _4) = SieveLinear(15);
        Console.WriteLine($"phi(15) = {phi15[15]} (expect 8)");
        Console.WriteLine($"7^{phi15[15]} mod 15 = {PowMod(7, phi15[15], 15)} (expect 1)");

        // --- Perfect number check ---
        Console.WriteLine("\n=== Perfect Numbers in [1, 1000] ===");
        var (_, _, _, sigmaBig, _, _5) = SieveLinear(1000);
        for (int i = 1; i <= 1000; i++)
            if (sigmaBig[i] == 2 * i)
                Console.WriteLine($"  Perfect number: {i} (sigma={sigmaBig[i]})");
    }
}
```

---

## Visualization: Meet-in-the-Middle of Divisors

```
φ(n) via inclusion-exclusion over prime factors of n:
  n = p1^a1 * p2^a2 * ... * pk^ak

  φ(n) = n
       × (1 - 1/p1)   ← remove multiples of p1
       × (1 - 1/p2)   ← remove multiples of p2
           ...
       × (1 - 1/pk)   ← remove multiples of pk

Example: n = 30 = 2 × 3 × 5
  Start:   30
  ×(1-1/2): 30 × 1/2 = 15     (remove even numbers)
  ×(1-1/3): 15 × 2/3 = 10     (remove mult of 3)
  ×(1-1/5): 10 × 4/5 = 8      (remove mult of 5)
  φ(30) = 8

The 8 coprimes in [1,30]: {1, 7, 11, 13, 17, 19, 23, 29}
```

---

## Gauss Identity Visualization

```
n = 12, φ(12) = 4

Divisors of 12: {1, 2, 3, 4, 6, 12}

d=1:  φ(1)  = 1   numbers coprime to 1 in [1,1]: {1}
d=2:  φ(2)  = 1   numbers coprime to 2 in [1,2]: {1} → reduced mod 2 → {1}
d=3:  φ(3)  = 2   coprime to 3 in [1,3]: {1,2}
d=4:  φ(4)  = 2   coprime to 4 in [1,4]: {1,3}
d=6:  φ(6)  = 2   coprime to 6 in [1,6]: {1,5}
d=12: φ(12) = 4   coprime to 12 in [1,12]: {1,5,7,11}

Sum = 1+1+2+2+2+4 = 12 = n ✓
```

---

## Dirichlet Convolution Table

```
Important identities via Dirichlet convolution (f * g = h):

  1 * 1  = d          (constant 1 convolved with 1 gives divisor count)
  id * 1 = σ          (identity convolved with 1 gives divisor sum)
  μ * 1  = ε          (Möbius convolved with 1 gives identity element)
  μ * id = φ          (Möbius convolved with id gives totient)
  φ * 1  = id         (totient convolved with 1 gives n — Gauss)
  λ * 1  = [n is square]   (Liouville convolved with 1 = squareness indicator)
  μ² * 1 = 2^ω        (squarefree indicator * 1 = 2^(number of distinct prime factors))

Key algebraic structure:
  (Multiplicative functions, *, ε) form an abelian group
  Every multiplicative f has unique inverse f⁻¹ under *
  μ = 1⁻¹ (Möbius is the Dirichlet inverse of constant 1)
```

---

## Pitfalls

- **`phi[i] = i` initialization** — in the Eratosthenes-style sieve, phi is initialized to `phi[i] = i` and then modified multiplicatively as primes are found. If instead you initialize `phi[i] = 0` and try to compute it from scratch during the sieve, you'll miss the final `phi[p] = p - 1` assignment for primes, since the sieve only visits multiples (not the prime itself in the inner loop).

- **Division order in the product formula** — when computing `phi[m] = phi[m] / p * (p - 1)`, the division `phi[m] / p` must be integer-exact (it always is since `p | phi[m]` at that point), and it must come **before** the multiplication by `(p-1)` to avoid intermediate overflow when values approach the type limit. Doing `phi[m] = phi[m] * (p-1) / p` can overflow for large `m`.

- **Linear sieve: the `break` is mandatory** — when `p == spf[i]` (i.e., `i % p == 0`), after processing `i*p` you must `break` immediately. Without the `break`, the sieve continues with the next prime `p'` and marks `i*p'` incorrectly because `spf[i*p'] = p'` but `p < p'`, violating the SPF invariant. Every composite must be reached exactly once via its smallest prime factor.

- **`σ` recurrence requires `σ(p^e)` separately** — in the linear sieve, when computing `σ(ip)` in the `p | i` branch, you cannot use `σ(ip) = σ(i) * σ(p)` because `gcd(i, p) ≠ 1`. You must store the geometric sum `σ_pk[i] = 1 + p + ... + p^e` separately and use `σ(ip) = (σ(i) / σ_pk[i]) * σ_pk[ip]`. Forgetting this produces wrong `σ` values for all numbers with repeated prime factors.

- **`μ(n) = 0` for non-squarefree n is missed in naive sieve** — if you only flip sign for each prime and don't separately zero out `μ` for squarefree violations, non-squarefree numbers get wrong `μ` values (e.g., `μ(4)` becomes -1 instead of 0). In the linear sieve this is handled naturally since the `p | i` branch sets `μ(ip) = 0`, but in the Eratosthenes-style sieve you need an explicit second pass over multiples of `p²`.

- **`d` recurrence: integer division must be exact** — `d[ip] = d[i] / (e+1) * (e+2)`. The division `d[i] / (e+1)` strips out the contribution of `p^e` to the divisor count of `i`, and `(e+1)` always divides `d[i]` exactly (since `d[i] = (e+1) * d[i/p^e]`). If `d` is stored as a general integer type and the value is large, ensure you're not doing floating-point division; always use integer division.

- **`φ` is not completely multiplicative** — `φ(mn) = φ(m)φ(n)` only when `gcd(m,n) = 1`. For example, `φ(4) = 2` but `φ(2)·φ(2) = 1·1 = 1 ≠ 2`. A common mistake is applying the multiplicativity rule without checking the coprimality condition. The correct formula for non-coprime case is `φ(mn) = φ(m)·φ(n)·d/φ(d)` where `d = gcd(m,n)`.

- **Overflow in `σ(n)` for large n** — the divisor sum grows fast: `σ(2^k) = 2^(k+1) - 1`, so for `n ≤ 10^6`, `σ(n)` can reach several million but fits in 32-bit. For `n ≤ 10^7`, `σ` can overflow 32-bit. Always use 64-bit (`long` / `ll`) for `σ`.

- **Gauss identity is a sum identity, not a value identity** — `∑_{d|n} φ(d) = n` is a relation between the sum of φ over all divisors and `n`, not a statement that `φ(n) = n`. Confusing these leads to incorrect formulas when trying to derive `φ(n)` directly from `n` without factorization.

- **Prefix sum of φ counts lattice points** — `∑_{i=1}^{n} φ(i)` counts the number of pairs `(a,b)` with `1 ≤ a ≤ b ≤ n` and `gcd(a,b) = 1`. This has a well-known asymptotic `~ 3n²/π²`. Mistaking this sum for a count of coprimes to a specific modulus is a frequent source of bugs in competitive programming.

---

## Complexity Summary

| Algorithm | Time | Space | Notes |
|---|---|---|---|
| Single φ(n), μ(n), d(n), σ(n) | O(√n) | O(1) | Trial division |
| Eratosthenes sieve for φ, μ | O(N log log N) | O(N) | Simple to implement |
| Linear sieve — all functions | O(N) | O(N) | Each composite once |
| Dirichlet convolution | O(N log N) | O(N) | For arbitrary f,g |
| Prefix sum via hyperbola | O(N^(2/3)) | O(N^(1/3)) | For ∑φ, ∑μ up to N |

---

## Conclusion

Euler's totient and the family of multiplicative functions form a **unified algebraic framework**
for number theory:

- `φ(n)` counts residues coprime to `n` and appears in Euler's generalization of Fermat's
  little theorem, RSA, and the structure of multiplicative groups mod n.
- The **linear sieve** computes `φ`, `μ`, `d`, `σ`, and any multiplicative function for all
  `n ≤ N` in optimal `O(N)` time by visiting each composite exactly once via its smallest
  prime factor.
- **Dirichlet convolution** gives these functions a rich algebraic structure: `φ = μ * id`,
  `σ = id * 1`, `d = 1 * 1`, and Gauss's identity `φ * 1 = id` — all provable via this one
  operation.
- The **Möbius inversion formula** (inverse of the Dirichlet convolution with `1`) is the
  foundational tool for translating between summatory functions and the functions themselves.

**Key takeaway:**  
When you need multiplicative function values for many `n` up to some bound, use the **linear
sieve** — it is the single most efficient tool for this family of problems. For a single value,
trial division in `O(√n)` is unbeatable in simplicity and sufficient for `n ≤ 10^12`. The
connection via Dirichlet convolution (`φ = μ * id`) is not just elegant — it is computationally
actionable via `O(N log N)` convolution when you already have `μ` and `id` precomputed.
