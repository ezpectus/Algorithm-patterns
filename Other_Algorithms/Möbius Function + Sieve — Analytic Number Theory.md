# Möbius Function + Sieve — Analytic Number Theory

## Origin & Motivation

The Möbius function `μ(n)` was introduced by August Ferdinand Möbius in 1832. It is the central object of **multiplicative number theory**, encoding the inclusion-exclusion principle over divisors in a single arithmetic function. The **Möbius inversion formula** is the number-theoretic analogue of inclusion-exclusion: if `g(n) = Σ_{d|n} f(d)`, then `f(n) = Σ_{d|n} μ(d) * g(n/d)`.

Together with the **linear sieve** (or Euler's sieve), all multiplicative functions — Euler's totient φ, number of divisors d(n), sum of divisors σ(n), Liouville's λ, Möbius μ itself — can be computed for all integers up to N in **O(N)** time, which is asymptotically optimal.

Complexity: **O(N)** for linear sieve of all multiplicative functions up to N, **O(N log log N)** for standard sieve of Eratosthenes.

---

## Where It Is Used

- Counting integers in [1, N] coprime to a set of primes (inclusion-exclusion via Möbius)
- Computing Euler's totient φ(n) for all n up to N
- Multiplicative function evaluation (d(n), σ(n), λ(n))
- Summing f(n) over all n in a range (Dirichlet series, prefix sums)
- Counting squarefree integers, Möbius sieve for number theory
- Competitive programming: divisor sums, coprimality counting, Dirichlet convolution
- Inverting Dirichlet series: recovering f from g = f * 1 (Möbius inversion)

---

## Möbius Function — Definition

```
μ(1)    = 1
μ(n)    = 0           if n has a squared prime factor (p² | n for some prime p)
μ(n)    = (-1)^k      if n = p₁ * p₂ * ... * pₖ  (product of k distinct primes)
```

Examples:
```
μ(1)=1, μ(2)=-1, μ(3)=-1, μ(4)=0, μ(5)=-1,
μ(6)=1, μ(7)=-1, μ(8)=0, μ(9)=0, μ(10)=1
```

**Key property:** `Σ_{d|n} μ(d) = [n == 1]` (1 if n=1, 0 otherwise). This is the fundamental identity that powers Möbius inversion.

---

## Multiplicative Functions

A function `f: ℕ → ℂ` is **multiplicative** if `f(1) = 1` and `f(ab) = f(a)f(b)` whenever `gcd(a,b) = 1`.

| Function | Formula | Meaning |
|---|---|---|
| `μ(n)` | See above | Möbius function |
| `φ(n)` | `n * Π(1 - 1/p)` | Euler's totient: integers ≤ n coprime to n |
| `d(n)` | `Π(eᵢ + 1)` | Number of divisors |
| `σ(n)` | `Π(pᵢ^{eᵢ+1} - 1)/(pᵢ - 1)` | Sum of divisors |
| `λ(n)` | `(-1)^Ω(n)` | Liouville: (-1)^(total prime factors with multiplicity) |
| `ω(n)` | count of distinct prime factors | Not multiplicative but additive |
| `id(n)` | `n` | Identity function |
| `1(n)` | `1` | Constant 1 |

**Dirichlet convolution:** `(f * g)(n) = Σ_{d|n} f(d) * g(n/d)`

Key identities:
```
φ = μ * id       (φ(n) = Σ_{d|n} μ(d) * (n/d))
d  = 1 * 1       (d(n) = Σ_{d|n} 1)
σ  = 1 * id      (σ(n) = Σ_{d|n} d)
μ * 1 = ε        (ε(n) = [n==1], Möbius inversion identity)
```

---

## Möbius Inversion Formula

If `g(n) = Σ_{d|n} f(d)`, then:
```
f(n) = Σ_{d|n} μ(d) * g(n/d) = Σ_{d|n} μ(n/d) * g(d)
```

**Applications:**
- From `d(n) = Σ_{d|n} 1`, invert to get `1 = Σ_{d|n} μ(d) * d(n/d)`
- From `n = Σ_{d|n} φ(d)`, invert to get `φ(n) = Σ_{d|n} μ(d) * (n/d)`
- Count integers in [1, N] with property P: express as sum, apply inversion

---

## Linear Sieve (Euler's Sieve)

The linear sieve generates all primes up to N and computes any multiplicative function in **O(N)** time. Each composite number is marked exactly once — by its **smallest prime factor (SPF)**.

```
Algorithm:
  primes = [], spf[i] = 0 for all i
  for i = 2 to N:
    if spf[i] == 0:                 // i is prime
      spf[i] = i
      primes.append(i)
      f[i] = f_at_prime(i)          // base case: prime
    for p in primes while p <= spf[i] and i*p <= N:
      spf[i*p] = p
      f[i*p] = f_combine(i, p)      // use multiplicativity
```

The key invariant: `spf[i*p] = p` since `p <= spf[i]`, so `p` is the smallest prime of `i*p`. Each number is marked by exactly one prime — O(N) total.

---

## Complexity Analysis

| Algorithm | Time | Space | Notes |
|---|---|---|---|
| Sieve of Eratosthenes | O(N log log N) | O(N) | Primes only |
| Linear sieve (primes) | O(N) | O(N) | Primes + SPF |
| Linear sieve (multiplicative f) | O(N) | O(N) | Any mult. function |
| Möbius sieve | O(N log N) | O(N) | Via divisor iteration |
| Prefix sum of mult. f | O(N) | O(N) | After linear sieve |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const int MAXN = 10000001;

// ================================================================
// LINEAR SIEVE — computes primes, SPF, and multiplicative functions
// in O(N) time.
// ================================================================
int  spf[MAXN];     // smallest prime factor
int  mu[MAXN];      // Möbius function
int  phi_arr[MAXN]; // Euler's totient
ll   prefix_mu[MAXN];  // prefix sum of mu
ll   prefix_phi[MAXN]; // prefix sum of phi
vector<int> primes;

void linear_sieve(int N) {
    // Initialize
    mu[1]  = 1;
    phi_arr[1] = 1;

    for (int i = 2; i <= N; i++) {
        if (spf[i] == 0) { // i is prime
            spf[i]  = i;
            mu[i]   = -1;          // prime: mu = -1
            phi_arr[i] = i - 1;   // prime: phi = p-1
            primes.push_back(i);
        }

        for (int j = 0; j < (int)primes.size()
             && primes[j] <= spf[i]
             && (ll)i * primes[j] <= N; j++)
        {
            int p = primes[j];
            int ip = i * p;
            spf[ip] = p;

            if (i % p == 0) {
                // p | i: p appears at least twice in ip
                // mu(ip) = 0 (has p^2 factor)
                mu[ip] = 0;
                // phi(ip) = phi(i) * p  (since p | i => phi(ip/gcd)... )
                // phi(p^{k+1}) = p^k * (p-1), phi(p^k) = p^{k-1}*(p-1)
                // phi(ip) = phi(i) * p  (because p | i)
                phi_arr[ip] = phi_arr[i] * p;
            } else {
                // gcd(i, p) = 1: multiplicative
                // mu(ip) = mu(i) * mu(p) = mu(i) * (-1)
                mu[ip] = -mu[i];
                // phi(ip) = phi(i) * phi(p) = phi(i) * (p-1)
                phi_arr[ip] = phi_arr[i] * (p - 1);
            }
        }
    }

    // Prefix sums
    prefix_mu[0]  = 0;
    prefix_phi[0] = 0;
    for (int i = 1; i <= N; i++) {
        prefix_mu[i]  = prefix_mu[i-1]  + mu[i];
        prefix_phi[i] = prefix_phi[i-1] + phi_arr[i];
    }
}

// ================================================================
// NUMBER OF DIVISORS d(n) via linear sieve
// ================================================================
int d_arr[MAXN]; // number of divisors
int d_exp[MAXN]; // exponent of spf in n (for multiplicativity)

void sieve_divisors(int N) {
    d_arr[1] = 1; d_exp[1] = 0;

    for (int i = 2; i <= N; i++) {
        if (spf[i] == i) { // prime
            d_arr[i] = 2;
            d_exp[i] = 1;
        }
    }

    for (int i = 2; i <= N; i++) {
        if (spf[i] == i) continue; // prime, already done

        int p = spf[i];
        int prev = i / p;

        if (spf[prev] == p) {
            // p | prev: exponent of p in i = exp(prev) + 1
            d_exp[i] = d_exp[prev] + 1;
            // d(i) = d(prev) / (d_exp[prev]+1) * (d_exp[prev]+2)
            d_arr[i] = d_arr[prev] / (d_exp[prev] + 1) * (d_exp[prev] + 2);
        } else {
            // p does not divide prev: gcd(p, prev) = 1
            d_exp[i] = 1;
            d_arr[i] = d_arr[prev] * 2;
        }
    }
}

// ================================================================
// SUM OF DIVISORS σ(n) via sieve
// ================================================================
ll sigma_arr[MAXN];
ll sigma_pk[MAXN]; // contribution from the spf prime power in n

void sieve_sigma(int N) {
    sigma_arr[1] = 1;
    sigma_pk[1]  = 1;

    for (int i = 2; i <= N; i++) {
        if (spf[i] == i) { // prime
            sigma_arr[i] = i + 1;
            sigma_pk[i]  = i + 1;
        }
    }

    // For composites: done via linear sieve relationship
    // Actually we recompute using SPF decomposition:
    for (int i = 2; i <= N; i++) {
        if (spf[i] == i) continue;
        int p = spf[i], prev = i / p;

        if (spf[prev] == p) {
            // p | prev: sigma_pk(i) = sigma_pk(prev)*p + 1
            // since p^{e+1}+...+1 = p*(p^e+...+1) + 1... wait:
            // sigma_pk encodes 1 + p + p^2 + ... + p^e for the prime power part
            // sigma_pk(p^e) = (p^{e+1}-1)/(p-1)
            // sigma_pk(i) where spf^e || i: sigma_pk(i) = sigma_pk(i/p)*p + 1? No:
            // Let's store sigma_pk as sum 1+p+...+p^{v_p(i)}
            // sigma_pk(i) = sigma_pk(prev) * p + 1? No.
            // sigma_pk(prev) = 1+p+...+p^{e-1}, sigma_pk(i) = 1+p+...+p^e
            // = sigma_pk(prev) + p^e
            // But we don't store p^e separately.
            // Simpler: sigma(i) = sigma(i/p^e) * sigma(p^e)
            // Use d_exp to get e, then compute p^{e+1}-1)/(p-1) directly.
            int e = d_exp[i];
            ll pk = 1, geo = 1;
            for (int k = 0; k <= e; k++) { geo += pk; pk *= p; }
            // geo = 1 + p + ... + p^e = (p^{e+1}-1)/(p-1)
            sigma_pk[i] = geo - 1; // store (1+p+...+p^e)
            // prev2 = i / p^e (the cofactor coprime to p)
            int tmp = i;
            while (tmp % p == 0) tmp /= p;
            sigma_arr[i] = sigma_arr[tmp] * sigma_pk[i];
        } else {
            sigma_pk[i]  = p + 1; // = 1 + p
            sigma_arr[i] = sigma_arr[prev] * sigma_pk[i];
        }
    }
}

// ================================================================
// MÖBIUS INVERSION — count squarefree integers in [1, N]
// squarefree(N) = Σ_{d=1}^{sqrt(N)} μ(d) * floor(N / d^2)
// ================================================================
ll count_squarefree(ll N) {
    ll result = 0;
    for (ll d = 1; d * d <= N; d++)
        result += mu[d] * (N / (d * d));
    return result;
}

// ================================================================
// COUNT INTEGERS in [1, N] coprime to m
// = Σ_{d | m} μ(d) * floor(N / d)   (inclusion-exclusion via Möbius)
// ================================================================
ll count_coprime(ll N, ll m) {
    // Factor m, iterate over divisors
    vector<ll> divs = {1};
    ll tmp = m;
    for (ll p = 2; p * p <= tmp; p++) {
        if (tmp % p == 0) {
            int sz = divs.size();
            while (tmp % p == 0) tmp /= p;
            for (int i = 0; i < sz; i++) divs.push_back(divs[i] * p);
        }
    }
    if (tmp > 1) {
        int sz = divs.size();
        for (int i = 0; i < sz; i++) divs.push_back(divs[i] * tmp);
    }

    // Apply Möbius: only squarefree divisors of m matter
    ll result = 0;
    for (ll d : divs) {
        ll mu_d = 1;
        ll t = d;
        for (ll p = 2; p * p <= t; p++) {
            if (t % p == 0) { mu_d = -mu_d; t /= p; }
        }
        if (t > 1) mu_d = -mu_d;
        result += mu_d * (N / d);
    }
    return result;
}

// ================================================================
// SUM OF φ(i) for i in [1, N] — using prefix_phi
// = total coprime pairs (a,b) with a <= b <= N
// ================================================================
ll sum_phi(ll N) {
    if (N <= MAXN - 1) return prefix_phi[N];
    // Else use formula or extend sieve
    return 0;
}

// ================================================================
// MERTENS FUNCTION M(N) = Σ_{i=1}^{N} μ(i)
// ================================================================
ll mertens(ll N) {
    if (N <= MAXN - 1) return prefix_mu[N];
    return 0;
}

// ================================================================
// DIRICHLET CONVOLUTION f*g computed for all n up to N
// O(N log N)
// ================================================================
vector<ll> dirichlet_conv(const vector<ll>& f, const vector<ll>& g, int N) {
    vector<ll> h(N + 1, 0);
    for (int d = 1; d <= N; d++) {
        if (!f[d]) continue;
        for (int k = 1; (ll)d * k <= N; k++)
            h[d * k] += f[d] * g[k];
    }
    return h;
}

// ================================================================
// Usage
// ================================================================
int main() {
    const int N = 20;
    linear_sieve(N);

    // Print mu, phi, prefix sums
    printf("n   mu  phi  prefix_mu  prefix_phi\n");
    for (int i = 1; i <= N; i++)
        printf("%-4d%-4d%-5d%-11lld%lld\n",
               i, mu[i], phi_arr[i], prefix_mu[i], prefix_phi[i]);

    // Verify: Σ_{d|n} mu(d) = [n==1]
    printf("\nVerify Σ_{d|n} mu(d) = [n==1]:\n");
    for (int n = 1; n <= 10; n++) {
        int sum = 0;
        for (int d = 1; d <= n; d++)
            if (n % d == 0) sum += mu[d];
        printf("  n=%d: sum=%d (expect %d)\n", n, sum, n==1?1:0);
    }

    // Verify: Σ_{d|n} phi(d) = n
    printf("\nVerify Σ_{d|n} phi(d) = n:\n");
    for (int n = 1; n <= 10; n++) {
        int sum = 0;
        for (int d = 1; d <= n; d++)
            if (n % d == 0) sum += phi_arr[d];
        printf("  n=%d: sum=%d (expect %d)\n", n, sum, n);
    }

    // Count squarefree integers
    for (ll n : {100LL, 1000LL, 1000000LL}) {
        printf("\nSquarefree in [1,%lld] = %lld\n",
               n, count_squarefree(n));
        // approx 6/pi^2 * n ≈ 0.6079 * n
    }

    // Count coprime to 12
    printf("\nIntegers in [1,100] coprime to 12: %lld\n",
           count_coprime(100, 12));
    // phi(12)=4, so approx 100*4/12 = 33

    // Mertens function
    printf("\nMertens M(n) for small n:\n");
    for (int n : {1,2,5,10,20})
        printf("  M(%d) = %lld\n", n, mertens(n));

    // Dirichlet convolution: 1*1 = d (number of divisors)
    {
        int n2 = 15;
        vector<ll> one(n2+1, 1); one[0]=0;
        auto dconv = dirichlet_conv(one, one, n2);
        printf("\nd(n) = (1*1)(n) via Dirichlet conv:\n");
        for (int i = 1; i <= n2; i++)
            printf("  d(%d)=%lld\n", i, dconv[i]);
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class MobiusSieve {

    // ================================================================
    // LINEAR SIEVE — O(N) computation of primes, μ, φ
    // ================================================================
    public static (int[] mu, int[] phi, long[] prefMu, long[] prefPhi,
                   List<int> primes, int[] spf)
    LinearSieve(int N) {
        var spf    = new int[N + 1];
        var mu     = new int[N + 1];
        var phi    = new int[N + 1];
        var prefMu = new long[N + 1];
        var prefPhi= new long[N + 1];
        var primes = new List<int>();

        mu[1]  = 1;
        phi[1] = 1;

        for (int i = 2; i <= N; i++) {
            if (spf[i] == 0) {      // i is prime
                spf[i]  = i;
                mu[i]   = -1;
                phi[i]  = i - 1;
                primes.Add(i);
            }
            foreach (int p in primes) {
                if ((long)i * p > N || p > spf[i]) break;
                int ip = i * p;
                spf[ip] = p;
                if (i % p == 0) {
                    mu[ip]  = 0;
                    phi[ip] = phi[i] * p;
                } else {
                    mu[ip]  = -mu[i];
                    phi[ip] = phi[i] * (p - 1);
                }
            }
        }

        for (int i = 1; i <= N; i++) {
            prefMu[i]  = prefMu[i-1]  + mu[i];
            prefPhi[i] = prefPhi[i-1] + phi[i];
        }

        return (mu, phi, prefMu, prefPhi, primes, spf);
    }

    // ================================================================
    // COUNT SQUAREFREE INTEGERS in [1, N]
    // Σ_{d=1}^{sqrt(N)} μ(d) * floor(N/d²)
    // ================================================================
    public static long CountSquarefree(long N, int[] mu) {
        long result = 0;
        for (long d = 1; d * d <= N; d++)
            result += mu[d] * (N / (d * d));
        return result;
    }

    // ================================================================
    // COUNT INTEGERS in [1, N] coprime to m
    // Σ_{d | m, d squarefree} μ(d) * floor(N/d)
    // ================================================================
    public static long CountCoprime(long N, long m) {
        // Get squarefree divisors of m with their Möbius values
        var divMu = new List<(long d, int mu)> { (1, 1) };
        long tmp = m;
        for (long p = 2; p * p <= tmp; p++) {
            if (tmp % p != 0) continue;
            int sz = divMu.Count;
            while (tmp % p == 0) tmp /= p;
            for (int i = 0; i < sz; i++)
                divMu.Add((divMu[i].d * p, -divMu[i].mu));
        }
        if (tmp > 1)
        {
            int sz = divMu.Count;
            for (int i = 0; i < sz; i++)
                divMu.Add((divMu[i].d * tmp, -divMu[i].mu));
        }

        long result = 0;
        foreach (var (d, muVal) in divMu)
            result += muVal * (N / d);
        return result;
    }

    // ================================================================
    // DIRICHLET CONVOLUTION (f * g)(n) for n = 1..N, O(N log N)
    // ================================================================
    public static long[] DirichletConv(long[] f, long[] g, int N) {
        var h = new long[N + 1];
        for (int d = 1; d <= N; d++) {
            if (f[d] == 0) continue;
            for (int k = 1; (long)d * k <= N; k++)
                h[d * k] += f[d] * g[k];
        }
        return h;
    }

    public static void Main() {
        int N = 20;
        var (mu, phi, prefMu, prefPhi, primes, spf) = LinearSieve(N);

        Console.WriteLine("n    mu   phi  prefix_mu  prefix_phi");
        for (int i = 1; i <= N; i++)
            Console.WriteLine($"{i,-5}{mu[i],-5}{phi[i],-5}{prefMu[i],-11}{prefPhi[i]}");

        // Verify Σ_{d|n} mu(d) = [n==1]
        Console.WriteLine("\nVerify Σ_{d|n}μ(d)=[n==1]:");
        for (int n = 1; n <= 10; n++) {
            int sum = 0;
            for (int d = 1; d <= n; d++) if (n % d == 0) sum += mu[d];
            Console.WriteLine($"  n={n}: {sum} (expect {(n==1?1:0)})");
        }

        // Verify Σ_{d|n} phi(d) = n
        Console.WriteLine("\nVerify Σ_{d|n}φ(d)=n:");
        for (int n = 1; n <= 10; n++) {
            int sum = 0;
            for (int d = 1; d <= n; d++) if (n % d == 0) sum += phi[d];
            Console.WriteLine($"  n={n}: {sum}");
        }

        // Count squarefree
        var (mu2, _, _, _, _, _) = LinearSieve(1000);
        Console.WriteLine($"\nSquarefree in [1,100]:  {CountSquarefree(100, mu2)}");
        Console.WriteLine($"Squarefree in [1,1000]: {CountSquarefree(1000, mu2)}");

        // Coprime
        Console.WriteLine($"\nCoprime to 12 in [1,100]: {CountCoprime(100, 12)}");

        // Dirichlet: 1*1 = d(n)
        Console.WriteLine("\nd(n) = (1*1)(n):");
        var one = new long[N + 1]; for (int i = 1; i <= N; i++) one[i] = 1;
        var dn  = DirichletConv(one, one, N);
        for (int i = 1; i <= N; i++) Console.Write($"d({i})={dn[i]} ");
        Console.WriteLine();
    }
}
```

---

## Key Identities and Applications

### 1. Euler's Totient via Möbius

```
φ(n) = n * Π_{p|n} (1 - 1/p) = Σ_{d|n} μ(d) * (n/d)

Verification: φ(12) = 12*(1-1/2)*(1-1/3) = 4
Via Möbius: divisors of 12 = {1,2,3,4,6,12}, squarefree = {1,2,3,6}
  μ(1)*12 + μ(2)*6 + μ(3)*4 + μ(6)*2 = 12 - 6 - 4 + 2 = 4 ✓
```

### 2. Counting GCD Pairs

Count pairs (a,b) with 1 ≤ a,b ≤ N and gcd(a,b) = k:

```
f(k) = #{(a,b): gcd(a,b)=k} = Σ_{d=1}^{N/k} μ(d) * floor(N/(dk))^2

Total pairs with gcd = 1:
  Σ_{d=1}^{N} μ(d) * floor(N/d)^2
```

### 3. Sum of φ(i) for 1 ≤ i ≤ N

```
Σ_{i=1}^{N} φ(i) = (1/2) * (1 + Σ_{d=1}^{N} μ(d) * floor(N/d)^2)

This counts lattice points (a,b) with gcd(a,b)=1 and 1≤a,b≤N.
```

### 4. Squarefree Count

```
#{n ≤ N : n is squarefree} = Σ_{d=1}^{sqrt(N)} μ(d) * floor(N/d²)

Asymptotically: 6N/π² ≈ 0.6079 * N
```

### 5. Dirichlet Series Connection

The Dirichlet series of μ is the inverse of ζ(s):
```
Σ_{n=1}^∞ μ(n)/n^s = 1/ζ(s)

This is why Möbius inversion works: μ is the "Dirichlet inverse" of 1.
```

---

## Pitfalls

- **μ(n) = 0 for non-squarefree n** — in the linear sieve, when `i % p == 0` (p already divides i), the product `ip` has p² as a factor, so `mu[ip] = 0`. Forgetting this case and computing `mu[ip] = -mu[i]` gives wrong values for all multiples of p² and cascades into wrong prefix sums.
- **phi[ip] = phi[i] * p when p | i** — the formula `phi(n) = phi(n/p) * p` holds when p divides n/p (i.e., p² | n). When `gcd(i, p) = 1`, use `phi[ip] = phi[i] * (p-1)` (multiplicativity). Confusing these two cases gives wrong totient values for prime powers.
- **Linear sieve inner loop condition** — the loop must break when `primes[j] > spf[i]` to guarantee each composite is marked exactly once. Without this condition, some composites are marked multiple times (still gives correct SPF) but the O(N) bound is lost.
- **prefix_mu is not monotone** — μ alternates between -1, 0, 1. The prefix sum (Mertens function) can be negative and oscillates. Do not assume it is non-decreasing or that `prefix_mu[N] > 0`.
- **Dirichlet convolution is O(N log N)** — the double loop `for d, for k=1..N/d` iterates N*H(N) ≈ N*ln(N) times. This is fine for N ≤ 10^6 but too slow for N = 10^7. For large N, use the linear sieve formula for each specific function instead.
- **count_squarefree iterates to sqrt(N)** — the sum `Σ_{d=1}^{sqrt(N)} μ(d) * floor(N/d²)` requires μ for d up to sqrt(N). For N = 10^{12}, sqrt(N) = 10^6 — precompute μ up to 10^6 with the linear sieve.
- **count_coprime builds squarefree divisors** — the inclusion-exclusion over divisors of m only uses squarefree divisors (since μ(d) = 0 for non-squarefree d). The code must deduplicate prime factors of m before building divisors; otherwise the squarefree divisor list is wrong for numbers like m = 12 = 4 * 3.

---

## Complexity Summary

| Task | Method | Time |
|---|---|---|
| All primes up to N | Sieve of Eratosthenes | O(N log log N) |
| All primes + SPF | Linear sieve | O(N) |
| μ(n), φ(n) for all n ≤ N | Linear sieve | O(N) |
| d(n), σ(n) for all n ≤ N | Linear sieve | O(N) |
| Σ_{i=1}^{N} μ(i) | Prefix sum after sieve | O(N) precomp, O(1) query |
| Count squarefree in [1,N] | Möbius sum | O(sqrt(N)) |
| Count coprime to m in [1,N] | Inclusion-exclusion | O(2^ω(m)) |
| Dirichlet convolution f*g | Double loop | O(N log N) |

---

## Conclusion

The Möbius function and linear sieve are the **core toolkit of analytic number theory in algorithms**:

- The linear sieve computes all multiplicative functions (μ, φ, d, σ) simultaneously in O(N) — optimal for precomputation.
- Möbius inversion converts divisor-sum identities into exact formulas, enabling inclusion-exclusion over arithmetic structure.
- Dirichlet convolution is the algebraic operation underlying all multiplicative function identities — `φ = μ * id`, `d = 1 * 1`, `σ = 1 * id`.
- Practical applications: counting coprime pairs, squarefree integers, GCD-constrained pairs — all reduce to Möbius sums.

**Key takeaway:**  
Build the linear sieve once for n up to 10^7. It gives primes, SPF, μ, φ, and their prefix sums in O(N) time and O(N) space. For queries on individual large n (up to 10^{18}), use trial division via SPF up to sqrt(n) combined with Pollard's Rho for complete factorization, then apply the multiplicative formulas directly.
