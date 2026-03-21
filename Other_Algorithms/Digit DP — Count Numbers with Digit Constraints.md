# Digit DP — Count Numbers with Digit Constraints

## Origin & Motivation

Digit DP is a dynamic programming technique for counting integers in a range `[L, R]` that satisfy some property expressible digit by digit. The key observation: properties like "sum of digits ≤ k", "no two adjacent digits are equal", or "digits are non-decreasing" depend only on the individual digits of the number, not its full value. Processing digits from most significant to least significant, the DP state tracks only the information needed to continue building the number: how many digits have been placed, whether the current prefix is still equal to the upper bound, and any accumulated property (digit sum, last digit, etc.).

The technique reduces what would be an O(R) iteration to **O(log R * states)** — logarithmic in the value of the bound, polynomial in the number of states.

The standard framework: count numbers in `[0, N]` satisfying property P, then answer for `[L, R]` = `count(R) - count(L-1)`.

Complexity: **O(D * S)** per query where D = number of digits, S = number of states per position.

---

## Where It Is Used

- Count integers in [L, R] with digit sum divisible by k
- Count integers with no two equal adjacent digits
- Count integers whose digits are non-decreasing / non-increasing
- Count integers not containing digit d
- Count integers with digit sum = k
- Count "lucky numbers" (only digits 4 and 7)
- Count balanced numbers (sum of first half digits = sum of second half)
- Competitive programming: any "count numbers satisfying property" problem

---

## Standard Framework

### count(N) — count integers in [0, N] satisfying P

Build N's digit representation. Process digit by digit from most significant.

**State:** `(position, tight, ...property_state...)`

- `position` — current digit index (0 = most significant)
- `tight` — whether current prefix equals N's prefix (constrains next digit)
- `property_state` — whatever the property needs (digit sum, last digit, etc.)
- `started` — optional flag: whether any non-zero digit has been placed yet (handles leading zeros)

**Transition:**

```
For digit d from 0 to limit:
    limit = tight ? N[pos] : 9
    new_tight = tight && (d == limit)
    new_state = update(property_state, d, started || d > 0)
    dp[pos+1][new_tight][new_state] += dp[pos][tight][property_state]
```

**Base:** `dp[0][true][initial_state] = 1`

**Answer:** Sum over all valid final states at position D (all digits placed).

### Template (Top-Down with Memoization)

```
solve(pos, tight, property_state, started):
    if pos == D: return is_valid(property_state, started)
    if memo[pos][tight][property_state][started] cached: return it

    limit = tight ? digits[pos] : 9
    result = 0
    for d = 0 to limit:
        result += solve(pos+1,
                        tight && d == limit,
                        update(property_state, d, started || d>0),
                        started || d>0)
    return memo[pos][tight][...] = result
```

---

## State Design

The key design decision is what to store in `property_state`:

| Property | State needed | State size |
|---|---|---|
| Digit sum | running sum | O(9*D) |
| Digit sum mod k | sum mod k | O(k) |
| Last digit | last digit placed | O(10) |
| No two equal adjacent | last digit | O(10) |
| Non-decreasing digits | last digit | O(10) |
| Contains digit d | boolean flag | O(2) |
| Count of specific digit | count | O(D) |
| Divisible by k | number mod k | O(k) |
| Balanced (half-sum equal) | (half1_sum - half2_sum) | O(9*D) |

Total states = D * 2 (tight) * product of all state dimensions. Keep this manageable (≤ 10^6).

---

## Complexity Analysis

| Parameter | Bound |
|---|---|
| Digits D | log₁₀(N) + 1 ≈ 18 for N ≤ 10^18 |
| Tight flag | 2 values |
| Started flag | 2 values |
| Property states | problem-specific, typically O(1)–O(D*k) |
| Total states | O(D * 2 * S) |
| Per-state work | O(10) (try each digit) |
| Total time | O(D * S * 10) |

For most problems: D ≈ 18, S ≤ 1000 → ~180,000 operations per query.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// GENERIC DIGIT DP TEMPLATE
// ================================================================

// ----------------------------------------------------------------
// Problem 1: Count numbers in [0, N] with digit sum == target
// ----------------------------------------------------------------
ll count_digit_sum(ll N, int target) {
    if (N < 0) return 0;
    string s = to_string(N);
    int D = s.size();

    // memo[pos][tight][sum][started]
    // sum up to 9*18 = 162
    const int MAXSUM = 170;
    vector<vector<vector<vector<ll>>>> memo(
        D, vector<vector<vector<ll>>>(
            2, vector<vector<ll>>(
                MAXSUM, vector<ll>(2, -1))));

    function<ll(int,bool,int,bool)> solve =
        [&](int pos, bool tight, int sum, bool started) -> ll {
        if (sum > target) return 0;
        if (pos == D) return started && sum == target ? 1 : 0;

        ll& ret = memo[pos][tight][sum][started];
        if (ret != -1) return ret;

        int limit = tight ? (s[pos] - '0') : 9;
        ret = 0;
        for (int d = 0; d <= limit; d++) {
            bool new_tight   = tight && (d == limit);
            bool new_started = started || (d > 0);
            int  new_sum     = (started || d > 0) ? sum + d : 0;
            ret += solve(pos+1, new_tight, new_sum, new_started);
        }
        return ret;
    };

    return solve(0, true, 0, false);
}

// ----------------------------------------------------------------
// Problem 2: Count numbers in [L, R] with no two equal adjacent digits
// ----------------------------------------------------------------
ll count_no_adjacent_equal(ll N) {
    if (N < 0) return 0;
    string s = to_string(N);
    int D = s.size();

    // memo[pos][tight][last_digit][started]
    // last_digit: 0-9 when started, 10 when not started
    vector<vector<vector<vector<ll>>>> memo(
        D, vector<vector<vector<ll>>>(
            2, vector<vector<ll>>(
                11, vector<ll>(2, -1))));

    function<ll(int,bool,int,bool)> solve =
        [&](int pos, bool tight, int last, bool started) -> ll {
        if (pos == D) return started ? 1 : 0;

        ll& ret = memo[pos][tight][last][started];
        if (ret != -1) return ret;

        int limit = tight ? (s[pos] - '0') : 9;
        ret = 0;
        for (int d = 0; d <= limit; d++) {
            if (started && d == last) continue; // adjacent equal
            bool new_tight   = tight && (d == limit);
            bool new_started = started || (d > 0);
            int  new_last    = (new_started) ? d : 10;
            ret += solve(pos+1, new_tight, new_last, new_started);
        }
        return ret;
    };

    return solve(0, true, 10, false);
}

// ----------------------------------------------------------------
// Problem 3: Count numbers in [L, R] divisible by k
// (classic approach: [L,R] = floor(R/k) - floor((L-1)/k)
//  but digit DP version handles more complex divisibility)
// Count numbers in [0, N] with digit sum divisible by k
// ----------------------------------------------------------------
ll count_digit_sum_div_k(ll N, int k) {
    if (N < 0 || k <= 0) return 0;
    string s = to_string(N);
    int D = s.size();

    // memo[pos][tight][sum_mod_k][started]
    vector<vector<vector<vector<ll>>>> memo(
        D, vector<vector<vector<ll>>>(
            2, vector<vector<ll>>(
                k, vector<ll>(2, -1))));

    function<ll(int,bool,int,bool)> solve =
        [&](int pos, bool tight, int rem, bool started) -> ll {
        if (pos == D) return (started && rem == 0) ? 1 : 0;

        ll& ret = memo[pos][tight][rem][started];
        if (ret != -1) return ret;

        int limit = tight ? (s[pos] - '0') : 9;
        ret = 0;
        for (int d = 0; d <= limit; d++) {
            bool new_tight   = tight && (d == limit);
            bool new_started = started || (d > 0);
            int  new_rem     = new_started ? (rem + d) % k : 0;
            ret += solve(pos+1, new_tight, new_rem, new_started);
        }
        return ret;
    };

    return solve(0, true, 0, false);
}

// ----------------------------------------------------------------
// Problem 4: Count numbers in [0, N] with non-decreasing digits
// ----------------------------------------------------------------
ll count_non_decreasing(ll N) {
    if (N < 0) return 0;
    string s = to_string(N);
    int D = s.size();

    // memo[pos][tight][last_digit][started]
    vector<vector<vector<vector<ll>>>> memo(
        D, vector<vector<vector<ll>>>(
            2, vector<vector<ll>>(
                10, vector<ll>(2, -1))));

    function<ll(int,bool,int,bool)> solve =
        [&](int pos, bool tight, int last, bool started) -> ll {
        if (pos == D) return started ? 1 : 0;

        ll& ret = memo[pos][tight][last][started];
        if (ret != -1) return ret;

        int limit = tight ? (s[pos] - '0') : 9;
        int lo    = started ? last : 0; // must be >= last digit
        ret = 0;
        for (int d = lo; d <= limit; d++) {
            bool new_tight   = tight && (d == limit);
            bool new_started = started || (d > 0);
            ret += solve(pos+1, new_tight, d, new_started);
        }
        return ret;
    };

    return solve(0, true, 0, false);
}

// ----------------------------------------------------------------
// Problem 5: Count "balanced" numbers — sum of first half digits
//            equals sum of second half digits. Handles even-length.
// ----------------------------------------------------------------
ll count_balanced(ll N) {
    if (N < 0) return 0;
    string s = to_string(N);
    int D = s.size();

    // memo[pos][tight][diff][started]
    // diff = sum_first_half - sum_second_half, offset by 9*D/2
    int offset = 9 * D; // safe offset
    int states = 2 * offset + 1;

    vector<vector<vector<vector<ll>>>> memo(
        D, vector<vector<vector<ll>>>(
            2, vector<vector<ll>>(
                states, vector<ll>(2, -1))));

    function<ll(int,bool,int,bool)> solve =
        [&](int pos, bool tight, int diff, bool started) -> ll {
        if (pos == D) return (started && diff == offset) ? 1 : 0;

        ll& ret = memo[pos][tight][diff][started];
        if (ret != -1) return ret;

        int limit = tight ? (s[pos] - '0') : 9;
        ret = 0;
        for (int d = 0; d <= limit; d++) {
            bool new_tight   = tight && (d == limit);
            bool new_started = started || (d > 0);
            // first half: positions 0..D/2-1 add d, second half subtract d
            int delta = 0;
            if (new_started) {
                delta = (pos < D/2) ? d : -d;
            }
            int new_diff = diff + delta;
            if (new_diff < 0 || new_diff >= states) continue;
            ret += solve(pos+1, new_tight, new_diff, new_started);
        }
        return ret;
    };

    return solve(0, true, offset, false);
}

// ----------------------------------------------------------------
// Problem 6: Count numbers containing at least one digit d
// Complement: count numbers NOT containing d, then subtract from total
// ----------------------------------------------------------------
ll count_without_digit(ll N, int forbidden) {
    if (N < 0) return 0;
    string s = to_string(N);
    int D = s.size();

    vector<vector<vector<ll>>> memo(
        D, vector<vector<ll>>(2, vector<ll>(2, -1)));

    function<ll(int,bool,bool)> solve =
        [&](int pos, bool tight, bool started) -> ll {
        if (pos == D) return started ? 1 : 0;

        ll& ret = memo[pos][tight][started];
        if (ret != -1) return ret;

        int limit = tight ? (s[pos] - '0') : 9;
        ret = 0;
        for (int d = 0; d <= limit; d++) {
            if (d == forbidden && (started || d > 0)) continue; // skip forbidden
            bool new_tight   = tight && (d == limit);
            bool new_started = started || (d > 0);
            ret += solve(pos+1, new_tight, new_started);
        }
        return ret;
    };

    return solve(0, true, false);
}

ll count_with_digit(ll L, ll R, int d) {
    auto total = [](ll n) -> ll {
        // count of numbers in [1,n]
        string s = to_string(n);
        return n; // total count = n itself; we want total - without_d
    };
    if (L > R) return 0;
    ll with_R = R - count_without_digit(R, d);
    ll with_Lm1 = (L-1) - count_without_digit(L-1, d);
    return with_R - with_Lm1;
}

// ================================================================
// QUERY WRAPPER: count in [L, R]
// ================================================================
ll query(ll L, ll R, function<ll(ll)> count_upto) {
    return count_upto(R) - count_upto(L - 1);
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Count numbers in [1, 100] with digit sum == 10
    printf("Digit sum == 10 in [1,100]: %lld\n",
        query(1, 100, [](ll n){ return count_digit_sum(n, 10); }));
    // 19,28,37,46,55,64,73,82,91,100 = 10 numbers

    // No adjacent equal digits in [1, 100]
    printf("No adjacent equal in [1,100]: %lld\n",
        query(1, 100, count_no_adjacent_equal));
    // 100 - {11,22,33,44,55,66,77,88,99,100} — 100 is fine, only doubles
    // Actually: all 1-digit + 2-digit non-equal + 100 = 9 + 81 + 1 = 91

    // Digit sum divisible by 3 in [1, 30]
    printf("Digit sum div 3 in [1,30]: %lld\n",
        query(1, 30, [](ll n){ return count_digit_sum_div_k(n, 3); }));
    // 3,6,9,12,15,18,21,24,27,30 = 10

    // Non-decreasing digits in [1, 50]
    printf("Non-decreasing digits in [1,50]: %lld\n",
        query(1, 50, count_non_decreasing));

    // Balanced numbers in [1, 1000]
    printf("Balanced in [1,1000]: %lld\n",
        query(1, 1000, count_balanced));

    // Numbers containing digit 7 in [1, 100]
    printf("Containing 7 in [1,100]: %lld\n", count_with_digit(1, 100, 7));
    // 7,17,27,37,47,57,67,70,71,72,73,74,75,76,77,78,79,87,97 = 19+1(77)=20? 
    // 7,17,27,37,47,57,67,70..79(10),87,97 = 19

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class DigitDP {

    // ================================================================
    // Count numbers in [0,N] with digit sum == target
    // ================================================================
    public static long CountDigitSum(long N, int target) {
        if (N < 0) return 0;
        var s = N.ToString();
        int D = s.Length;
        const int MAXSUM = 170;
        var memo = new long[D, 2, MAXSUM, 2];
        for (int i = 0; i < D; i++)
            for (int j = 0; j < 2; j++)
                for (int k = 0; k < MAXSUM; k++)
                    for (int l = 0; l < 2; l++)
                        memo[i,j,k,l] = -1;

        long Solve(int pos, int tight, int sum, int started) {
            if (sum > target) return 0;
            if (pos == D) return (started == 1 && sum == target) ? 1 : 0;
            ref long ret = ref memo[pos, tight, sum, started];
            if (ret != -1) return ret;

            int limit = tight == 1 ? (s[pos] - '0') : 9;
            ret = 0;
            for (int d = 0; d <= limit; d++) {
                int nt = (tight == 1 && d == limit) ? 1 : 0;
                int ns = (started == 1 || d > 0) ? 1 : 0;
                int nsum = ns == 1 ? sum + d : 0;
                ret += Solve(pos+1, nt, nsum, ns);
            }
            return ret;
        }

        return Solve(0, 1, 0, 0);
    }

    // ================================================================
    // Count numbers in [0,N] with no two adjacent equal digits
    // ================================================================
    public static long CountNoAdjacentEqual(long N) {
        if (N < 0) return 0;
        var s = N.ToString();
        int D = s.Length;
        var memo = new long[D, 2, 11, 2];
        for (int i = 0; i < D; i++)
            for (int j = 0; j < 2; j++)
                for (int k = 0; k < 11; k++)
                    for (int l = 0; l < 2; l++)
                        memo[i,j,k,l] = -1;

        long Solve(int pos, int tight, int last, int started) {
            if (pos == D) return started;
            ref long ret = ref memo[pos, tight, last, started];
            if (ret != -1) return ret;

            int limit = tight == 1 ? (s[pos] - '0') : 9;
            ret = 0;
            for (int d = 0; d <= limit; d++) {
                if (started == 1 && d == last) continue;
                int nt = (tight == 1 && d == limit) ? 1 : 0;
                int ns = (started == 1 || d > 0) ? 1 : 0;
                int nl = ns == 1 ? d : 10;
                ret += Solve(pos+1, nt, nl, ns);
            }
            return ret;
        }

        return Solve(0, 1, 10, 0);
    }

    // ================================================================
    // Count numbers in [0,N] with digit sum divisible by k
    // ================================================================
    public static long CountDigitSumDivK(long N, int k) {
        if (N < 0 || k <= 0) return 0;
        var s = N.ToString();
        int D = s.Length;
        var memo = new long[D, 2, k, 2];
        for (int i = 0; i < D; i++)
            for (int j = 0; j < 2; j++)
                for (int r = 0; r < k; r++)
                    for (int l = 0; l < 2; l++)
                        memo[i,j,r,l] = -1;

        long Solve(int pos, int tight, int rem, int started) {
            if (pos == D) return (started == 1 && rem == 0) ? 1 : 0;
            ref long ret = ref memo[pos, tight, rem, started];
            if (ret != -1) return ret;

            int limit = tight == 1 ? (s[pos] - '0') : 9;
            ret = 0;
            for (int d = 0; d <= limit; d++) {
                int nt = (tight == 1 && d == limit) ? 1 : 0;
                int ns = (started == 1 || d > 0) ? 1 : 0;
                int nr = ns == 1 ? (rem + d) % k : 0;
                ret += Solve(pos+1, nt, nr, ns);
            }
            return ret;
        }

        return Solve(0, 1, 0, 0);
    }

    // ================================================================
    // Count non-decreasing digit numbers in [0,N]
    // ================================================================
    public static long CountNonDecreasing(long N) {
        if (N < 0) return 0;
        var s = N.ToString();
        int D = s.Length;
        var memo = new long[D, 2, 10, 2];
        for (int i = 0; i < D; i++)
            for (int j = 0; j < 2; j++)
                for (int k = 0; k < 10; k++)
                    for (int l = 0; l < 2; l++)
                        memo[i,j,k,l] = -1;

        long Solve(int pos, int tight, int last, int started) {
            if (pos == D) return started;
            ref long ret = ref memo[pos, tight, last, started];
            if (ret != -1) return ret;

            int limit = tight == 1 ? (s[pos] - '0') : 9;
            int lo    = started == 1 ? last : 0;
            ret = 0;
            for (int d = lo; d <= limit; d++) {
                int nt = (tight == 1 && d == limit) ? 1 : 0;
                int ns = (started == 1 || d > 0) ? 1 : 0;
                ret += Solve(pos+1, nt, d, ns);
            }
            return ret;
        }

        return Solve(0, 1, 0, 0);
    }

    // Generic range query helper
    public static long Query(long L, long R, Func<long, long> countUpto) =>
        countUpto(R) - countUpto(L - 1);

    public static void Main() {
        Console.WriteLine($"Digit sum==10 in [1,100]: {Query(1,100, n=>CountDigitSum(n,10))}");
        Console.WriteLine($"No adjacent equal in [1,100]: {Query(1,100, CountNoAdjacentEqual)}");
        Console.WriteLine($"Digit sum div 3 in [1,30]: {Query(1,30, n=>CountDigitSumDivK(n,3))}");
        Console.WriteLine($"Non-decreasing in [1,50]: {Query(1,50, CountNonDecreasing)}");
    }
}
```

---

## Common Pitfalls

- **Leading zeros corrupt state** — the digit sum of `007` should equal the digit sum of `7`, not `0+0+7`. Use the `started` flag: only accumulate state after the first non-zero digit. Without it, `007` is incorrectly counted separately from `7`.
- **tight must be passed correctly** — `tight` is true only when the current prefix exactly equals the upper bound prefix. Once any digit `d < digits[pos]` is chosen, `tight` becomes false for all remaining positions. Propagating `tight = false` incorrectly (e.g., always passing `tight = true`) vastly over-constrains the search.
- **Memoization must not include tight=true states across calls** — when computing `count(R) - count(L-1)`, both calls reuse the same memo table. The `tight` dimension separates these cases: states with `tight=true` depend on the specific upper bound `N` and must not be shared between calls to `count(R)` and `count(L-1)`. Always reset memo between calls or include `N` as part of the key (the string representation handles this naturally).
- **answer = count(R) - count(L-1), not count(R) - count(L)** — the standard trick is `f(R) - f(L-1)` where `f(x)` counts in `[0, x]`. Using `f(L)` instead of `f(L-1)` excludes `L` from the range.
- **Avoid int overflow in state indexing** — if digit sum can reach 9 * 18 = 162 and you use it as an array index, allocate at least 163 slots. Allocating exactly `target + 1` causes out-of-bounds when intermediate sums exceed target during exploration (the DP explores sums greater than target before pruning).
- **started flag interaction with property state** — when `started = false`, the property state should reflect "no digits placed yet", not "zero placed". For digit sum: `sum = 0` is correct. For "last digit": use a sentinel value (e.g., -1 or 10) to indicate no digit placed. Confusing "last digit = 0" with "no digit placed" causes wrong transitions for numbers starting with 0 (leading zeros).
- **Bottom-up vs top-down** — top-down with memoization is easier to implement correctly but has recursive call overhead. Bottom-up iterates masks in order `0, 1, ..., 2^D - 1` by encoding (pos, tight, state) into a flat array. For very tight time limits, bottom-up avoids function call overhead.

---

## Complexity Summary

| Problem | State | Total states | Time per query |
|---|---|---|---|
| Digit sum = k | (pos, tight, sum, started) | 18 * 2 * 162 * 2 ≈ 12K | Fast |
| No adjacent equal | (pos, tight, last, started) | 18 * 2 * 11 * 2 ≈ 800 | Very fast |
| Digit sum mod k | (pos, tight, rem, started) | 18 * 2 * k * 2 | O(D*k) |
| Non-decreasing | (pos, tight, last, started) | 18 * 2 * 10 * 2 ≈ 720 | Very fast |
| Contains digit d | (pos, tight, found, started) | 18 * 2 * 2 * 2 ≈ 288 | Very fast |
| Balanced | (pos, tight, diff, started) | 18 * 2 * 324 * 2 ≈ 23K | Fast |

---

## Conclusion

Digit DP is the **canonical technique for counting integers with digit-level constraints**:

- Reduces O(N) iteration to O(D * S) where D ≈ 18 for 64-bit integers and S is the property state space — typically in the thousands.
- The framework is always the same: recurse over digit positions, pass `tight` and `started` flags, accumulate property state, memoize by (pos, tight, property_state, started).
- Range queries `[L, R]` reduce to `f(R) - f(L-1)` with a single `count_upto(N)` function.
- The only creative step is identifying what `property_state` to track — everything else is mechanical.

**Key takeaway:**  
When a problem says "count integers in [L, R] satisfying property P", and P depends on individual digits, use digit DP. Design the state as the minimal information needed to complete the number and check P. If the state space is ≤ 10^6, the solution is immediate. If the state space is larger, look for ways to compress it (e.g., use modular arithmetic instead of exact sums).
