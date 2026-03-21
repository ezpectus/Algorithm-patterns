# Bitap Algorithm — Approximate String Matching

## Origin & Motivation

The Bitap algorithm (also called **Shift-Or**, **Shift-And**, or **Wu-Manber algorithm**) was introduced by Baeza-Yates and Gonnet in 1992, with extensions for approximate matching by Wu and Manber in the same year. It solves the **approximate string matching** problem: find all positions in text `T` where pattern `P` occurs with at most `k` errors (substitutions, insertions, or deletions — Levenshtein distance ≤ k).

The key insight: encode the matching state as a **bitmask** of length `m` (pattern length), where bit `i` represents "the first `i+1` characters of `P` have been matched ending at the current text position." All state transitions become **bitwise shift and OR/AND operations** — extremely fast on modern hardware. For `k = 0` (exact matching), the algorithm runs in **O(n * ⌈m/w⌉)** time where `w` is the word size (64 on modern hardware), giving practical speedups of up to 64x over naive matching.

For approximate matching with `k` errors, `k + 1` bitmasks are maintained simultaneously, yielding **O(k * n * ⌈m/w⌉)** time.

Complexity: **O(n * ⌈m/w⌉)** exact, **O(k * n * ⌈m/w⌉)** approximate, **O(m * σ)** preprocessing.

---

## Where It Is Used

- `grep` with approximate matching (`agrep` tool uses Bitap internally)
- Spell checking and fuzzy search
- DNA sequence alignment (short reads, error-tolerant matching)
- Text editors: fuzzy find
- Competitive programming: pattern matching with wildcards or bounded errors
- Full-text search engines: fuzzy query matching

---

## Exact Matching — Shift-And

### Preprocessing

For each character `c` in the alphabet, build a bitmask `B[c]` of length `m`:

```
B[c][i] = 1  if P[i] == c
B[c][i] = 0  if P[i] != c
```

### State Bitmask

`R` is an m-bit integer where **bit `i` = 1** means "we have matched the first `i+1` characters of `P` ending at the current text position."

### Transition (Shift-And)

```
For each text character T[j]:
    R = ((R << 1) | 1) & B[T[j]]
    // Shift left: extend all partial matches by one position
    // OR 1:       allow new match starting at current position
    // AND B[c]:   keep only states where T[j] matches P[i]

If bit (m-1) of R is set: match found starting at position j-m+1
```

### Shift-Or (complement convention, used in agrep)

```
D[c][i] = 0  if P[i] == c,  1 otherwise   (complement of B)
R = ~0  (all bits 1 = no active matches)

For each T[j]:
    R = (R << 1) | D[T[j]]

If bit (m-1) of R is 0: match found
```

Both conventions are equivalent — Shift-And uses 1 for active states, Shift-Or uses 0.

---

## Approximate Matching — Wu-Manber Extension

Maintain `k+1` bitmasks `R[0], R[1], ..., R[k]` where `R[d]` represents states reachable with exactly `d` errors.

### Transition with Levenshtein Distance

For each text character `T[j]`, update all error levels from lowest to highest, saving the previous values:

```
prev_0 = R[0]
R[0] = ((R[0] << 1) | 1) & B[T[j]]      // 0 errors: exact match only

for d = 1 to k:
    old_d   = R[d]         // save before overwrite
    old_dm1 = prev_{d-1}   // previous position's R[d-1]

    match_sub = ((old_d   << 1) | 1) & B[T[j]]  // match or substitution
    insert_t  =  (old_dm1 << 1) | 1             // insert char in T
    delete_p  =   old_dm1                        // delete char from P

    R[d] = match_sub | insert_t | delete_p
    prev_d = old_d
```

If bit `(m-1)` of `R[d]` is set: match with exactly `d` errors ending at position `j`.

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Preprocessing (build B[c]) | O(m * σ) | O(σ * ⌈m/64⌉) words |
| Exact match per character | O(⌈m/64⌉) | — |
| Exact match total | O(n * ⌈m/64⌉) | O(σ * ⌈m/64⌉) |
| Approx match per character | O(k * ⌈m/64⌉) | — |
| Approx match total | O(k * n * ⌈m/64⌉) | O((k + σ) * ⌈m/64⌉) |

- For `m ≤ 64`: all bitmasks fit in one `uint64_t` → O(1) per character
- For `m > 64`: use arrays of `uint64_t`; O(⌈m/64⌉) per character

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using u64 = uint64_t;

// ================================================================
// EXACT MATCHING — Shift-And, m <= 64
// Returns start positions of all exact matches.
// ================================================================
vector<int> bitap_exact(const string& text, const string& pattern) {
    int n = text.size(), m = pattern.size();
    if (m == 0 || m > 64) return {};

    // Build B[c]: bit i is set if pattern[i] == c
    u64 B[256] = {};
    for (int i = 0; i < m; i++)
        B[(unsigned char)pattern[i]] |= (1ULL << i);

    vector<int> matches;
    u64 R = 0;
    u64 match_mask = 1ULL << (m - 1);

    for (int j = 0; j < n; j++) {
        // Extend all active states, start new state at bit 0,
        // keep only states where text[j] matches pattern[i]
        R = ((R << 1) | 1ULL) & B[(unsigned char)text[j]];
        if (R & match_mask)
            matches.push_back(j - m + 1);
    }
    return matches;
}

// ================================================================
// APPROXIMATE MATCHING — Levenshtein distance <= k, m <= 64
// Returns {start_approx, error_count} for each match position.
// ================================================================
struct ApproxMatch {
    int pos;
    int errors;
};

vector<ApproxMatch> bitap_approx(
    const string& text,
    const string& pattern,
    int k)
{
    int n = text.size(), m = pattern.size();
    if (m == 0 || m > 64 || k < 0) return {};
    k = min(k, m);

    u64 B[256] = {};
    for (int i = 0; i < m; i++)
        B[(unsigned char)pattern[i]] |= (1ULL << i);

    u64 match_mask = 1ULL << (m - 1);
    vector<u64> R(k + 1, 0); // R[d] = state with d errors

    vector<ApproxMatch> matches;

    for (int j = 0; j < n; j++) {
        u64 bc = B[(unsigned char)text[j]];

        // Update from highest d downward, using saved previous values
        u64 prev = R[0];
        R[0] = ((R[0] << 1) | 1ULL) & bc;

        for (int d = 1; d <= k; d++) {
            u64 old_d   = R[d];
            u64 old_dm1 = prev;
            prev = old_d;

            // Substitution/match: advance in both P and T
            u64 match_sub = ((old_d << 1) | 1ULL) & bc;
            // Insertion in T: advance in T, consume error (stay in P)
            u64 insert_t  = (old_dm1 << 1) | 1ULL;
            // Deletion from P: advance in P, consume error (stay in T)
            u64 delete_p  = old_dm1;

            R[d] = match_sub | insert_t | delete_p;
        }

        // Report minimum-error match at this text position
        for (int d = 0; d <= k; d++) {
            if (R[d] & match_mask) {
                // Approximate start: exact start requires backtracking
                matches.push_back({j - m + 1, d});
                break; // report minimum error count
            }
        }
    }
    return matches;
}

// ================================================================
// WILDCARD MATCHING — '?' matches any single character
// ================================================================
vector<int> bitap_wildcard(const string& text, const string& pattern) {
    int n = text.size(), m = pattern.size();
    if (m == 0 || m > 64) return {};

    u64 B[256] = {};
    for (int i = 0; i < m; i++) {
        if (pattern[i] == '?') {
            for (int c = 0; c < 256; c++)
                B[c] |= (1ULL << i);
        } else {
            B[(unsigned char)pattern[i]] |= (1ULL << i);
        }
    }

    vector<int> matches;
    u64 R = 0, match_mask = 1ULL << (m - 1);
    for (int j = 0; j < n; j++) {
        R = ((R << 1) | 1ULL) & B[(unsigned char)text[j]];
        if (R & match_mask) matches.push_back(j - m + 1);
    }
    return matches;
}

// ================================================================
// MULTI-WORD BITAP — pattern length > 64
// ================================================================
struct BitapLong {
    int m, W; // W = ceil(m/64)
    vector<vector<u64>> B;

    BitapLong(const string& p) : m(p.size()), W((p.size()+63)/64),
        B(256, vector<u64>((p.size()+63)/64, 0))
    {
        for (int i = 0; i < m; i++)
            B[(unsigned char)p[i]][i/64] |= 1ULL << (i%64);
    }

    // Shift multi-word bitmask left by 1 and OR bit 0 with carry_in
    vector<u64> shift1(const vector<u64>& R, u64 carry_in = 1) const {
        vector<u64> res(W);
        u64 carry = carry_in;
        for (int w = 0; w < W; w++) {
            res[w] = (R[w] << 1) | carry;
            carry  = R[w] >> 63;
        }
        return res;
    }

    bool is_match(const vector<u64>& R) const {
        return (R[(m-1)/64] >> ((m-1)%64)) & 1;
    }

    vector<int> search(const string& text) const {
        vector<u64> R(W, 0);
        vector<int> matches;
        for (int j = 0; j < (int)text.size(); j++) {
            auto s = shift1(R);
            auto& bc = B[(unsigned char)text[j]];
            for (int w = 0; w < W; w++) R[w] = s[w] & bc[w];
            if (is_match(R)) matches.push_back(j - m + 1);
        }
        return matches;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // Exact matching
    {
        auto m = bitap_exact("abcabcabc", "abc");
        printf("Exact 'abc' in 'abcabcabc': ");
        for (int p : m) printf("%d ", p);
        printf("\n");
    }
    {
        auto m = bitap_exact("hello world", "xyz");
        printf("Exact 'xyz': %s\n", m.empty() ? "not found" : "found");
    }

    // Approximate matching k=1
    {
        printf("\nApprox 'bce' in 'abcdefg' (k=1):\n");
        for (auto& [p, e] : bitap_approx("abcdefg", "bce", 1))
            printf("  pos~%d errors=%d\n", p, e);
    }
    {
        printf("Approx 'sitting' in 'kitten' (k=3):\n");
        for (auto& [p, e] : bitap_approx("kitten", "sitting", 3))
            printf("  pos~%d errors=%d\n", p, e);
    }

    // Wildcard
    {
        auto m = bitap_wildcard("abcdefg", "a?c");
        printf("\nWildcard 'a?c' in 'abcdefg': ");
        for (int p : m) printf("%d ", p);
        printf("\n");
    }

    // Multi-word
    {
        string pat(80, 'a');
        string txt = string(50, 'b') + pat + string(50, 'b');
        BitapLong bl(pat);
        auto m = bl.search(txt);
        printf("\nMulti-word (len=80): found at %d\n",
               m.empty() ? -1 : m[0]);
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public static class BitapAlgorithm {

    // ================================================================
    // EXACT MATCHING — Shift-And, pattern length <= 64
    // ================================================================
    public static List<int> ExactMatch(string text, string pattern) {
        int n = text.Length, m = pattern.Length;
        if (m == 0 || m > 64) return new();

        var B = new ulong[256];
        for (int i = 0; i < m; i++)
            B[pattern[i]] |= 1UL << i;

        var matches = new List<int>();
        ulong R = 0, matchMask = 1UL << (m - 1);

        for (int j = 0; j < n; j++) {
            R = ((R << 1) | 1UL) & B[text[j]];
            if ((R & matchMask) != 0) matches.Add(j - m + 1);
        }
        return matches;
    }

    // ================================================================
    // APPROXIMATE MATCHING — k errors (Levenshtein)
    // ================================================================
    public record ApproxMatch(int Pos, int Errors);

    public static List<ApproxMatch> ApproximateMatch(
        string text, string pattern, int k)
    {
        int n = text.Length, m = pattern.Length;
        if (m == 0 || m > 64 || k < 0) return new();
        k = Math.Min(k, m);

        var B = new ulong[256];
        for (int i = 0; i < m; i++)
            B[pattern[i]] |= 1UL << i;

        ulong matchMask = 1UL << (m - 1);
        var R    = new ulong[k + 1];
        var matches = new List<ApproxMatch>();

        for (int j = 0; j < n; j++) {
            ulong bc = B[text[j]];
            ulong prev = R[0];
            R[0] = ((R[0] << 1) | 1UL) & bc;

            for (int d = 1; d <= k; d++) {
                ulong oldD   = R[d];
                ulong oldDm1 = prev;
                prev = oldD;

                ulong matchSub = ((oldD   << 1) | 1UL) & bc;
                ulong insertT  =  (oldDm1 << 1) | 1UL;
                ulong deleteP  =   oldDm1;

                R[d] = matchSub | insertT | deleteP;
            }

            for (int d = 0; d <= k; d++) {
                if ((R[d] & matchMask) != 0) {
                    matches.Add(new ApproxMatch(j - m + 1, d));
                    break;
                }
            }
        }
        return matches;
    }

    // ================================================================
    // WILDCARD MATCHING — '?' matches any single character
    // ================================================================
    public static List<int> WildcardMatch(string text, string pattern) {
        int n = text.Length, m = pattern.Length;
        if (m == 0 || m > 64) return new();

        var B = new ulong[256];
        for (int i = 0; i < m; i++) {
            if (pattern[i] == '?')
                for (int c = 0; c < 256; c++) B[c] |= 1UL << i;
            else
                B[pattern[i]] |= 1UL << i;
        }

        var matches = new List<int>();
        ulong R = 0, matchMask = 1UL << (m - 1);
        for (int j = 0; j < n; j++) {
            R = ((R << 1) | 1UL) & B[text[j]];
            if ((R & matchMask) != 0) matches.Add(j - m + 1);
        }
        return matches;
    }

    // ================================================================
    // MULTI-WORD BITAP
    // ================================================================
    public class BitapLong {
        private readonly int m, W;
        private readonly ulong[][] B;

        public BitapLong(string pattern) {
            m = pattern.Length;
            W = (m + 63) / 64;
            B = new ulong[256][];
            for (int c = 0; c < 256; c++) B[c] = new ulong[W];
            for (int i = 0; i < m; i++)
                B[pattern[i]][i / 64] |= 1UL << (i % 64);
        }

        private ulong[] Shift1(ulong[] R) {
            var res = new ulong[W];
            ulong carry = 1;
            for (int w = 0; w < W; w++) {
                res[w] = (R[w] << 1) | carry;
                carry  = R[w] >> 63;
            }
            return res;
        }

        private bool IsMatch(ulong[] R) =>
            ((R[(m-1)/64] >> ((m-1)%64)) & 1) == 1;

        public List<int> Search(string text) {
            var R = new ulong[W];
            var matches = new List<int>();
            for (int j = 0; j < text.Length; j++) {
                var s  = Shift1(R);
                var bc = B[text[j]];
                for (int w = 0; w < W; w++) R[w] = s[w] & bc[w];
                if (IsMatch(R)) matches.Add(j - m + 1);
            }
            return matches;
        }
    }

    public static void Main() {
        // Exact
        Console.WriteLine($"Exact 'abc': {string.Join(", ", ExactMatch("abcabcabc","abc"))}");
        Console.WriteLine($"Exact 'xyz': {(ExactMatch("hello world","xyz").Count==0?"not found":"found")}");

        // Approximate
        Console.WriteLine("\nApprox 'bce' in 'abcdefg' (k=1):");
        foreach (var m in ApproximateMatch("abcdefg","bce",1))
            Console.WriteLine($"  pos~{m.Pos} errors={m.Errors}");

        Console.WriteLine("Approx 'sitting' in 'kitten' (k=3):");
        foreach (var m in ApproximateMatch("kitten","sitting",3))
            Console.WriteLine($"  pos~{m.Pos} errors={m.Errors}");

        // Wildcard
        Console.WriteLine($"\nWildcard 'a?c': {string.Join(", ", WildcardMatch("abcdefg","a?c"))}");

        // Multi-word
        string pat = new('a', 80);
        string txt = new('b', 50) + pat + new('b', 50);
        var bl = new BitapLong(pat);
        var res = bl.Search(txt);
        Console.WriteLine($"\nMulti-word (len=80): found at {(res.Count > 0 ? res[0] : -1)}");
    }
}
```

---

## Shift-Or vs Shift-And Convention

| Property | Shift-And | Shift-Or |
|---|---|---|
| `B[c][i]` value | 1 if `P[i]==c` | 0 if `P[i]==c` (complement) |
| Initial `R` | `0` | `~0` (all ones) |
| Transition | `R = ((R<<1)\|1) & B[c]` | `R = (R<<1) \| D[c]` |
| Match condition | `R & (1<<(m-1)) != 0` | `R & (1<<(m-1)) == 0` |
| Active state meaning | Bit = 1 | Bit = 0 |

Both are equivalent — choose whichever matches the problem framing.

---

## Comparison with Other Matching Algorithms

| Algorithm | Preprocessing | Search | Notes |
|---|---|---|---|
| Naive | O(1) | O(nm) | No structure |
| KMP | O(m) | O(n) | Linear, deterministic |
| Boyer-Moore | O(m+σ) | O(n/m) best | Fast in practice for large σ |
| Rabin-Karp | O(m) | O(n) expected | Hash-based, easy to extend |
| **Bitap exact** | O(mσ/64) | O(n⌈m/64⌉) | Best for m ≤ 64 |
| **Bitap approx** | O(mσ/64) | O(kn⌈m/64⌉) | Only practical approx bitwise method |
| DP Levenshtein | O(m) | O(nm) | General, no bit tricks |

---

## Pitfalls

- **Pattern length limit** — single-word Bitap works only for `m ≤ 64`. For longer patterns, use the multi-word variant. Silently truncating bits beyond position 63 causes missed matches and wrong positions without any runtime error.
- **State update order in approximate matching** — `R[d]` must be computed from the **previous** text position's `R[d-1]`, not the current step's updated value. Always save `prev = R[d]` before overwriting, then use `prev` as `old_dm1` for the next level. Using the updated `R[d-1]` mixes current and previous positions and breaks the Levenshtein recurrence.
- **Report minimum error per position** — check error levels from `d=0` upward and break after the first match. Without breaking, multiple entries are reported for the same text position at different error counts, inflating the result list.
- **Wildcard sets all 256 byte values** — `B[c] |= (1<<i)` must run for all `c` in `[0, 255]`, not just printable ASCII `[32, 126]`. Non-ASCII bytes in binary text files or Unicode-encoded strings will miss the wildcard without this.
- **Multi-word carry propagation** — shifting a multi-word bitmask left by 1 must propagate bit 63 of word `w` into bit 0 of word `w+1`. Forgetting carry gives correct results within each 64-bit word but wrong results at word boundaries — a bug that only manifests for patterns of length 65, 129, etc.
- **Approximate match start position is approximate** — `j - m + 1` gives the exact end position but the start position depends on the edit script. With deletions, the actual matched region in the text can be shorter than `m`. For exact start positions, run a second Bitap pass from right to left on the matched region, or use the DP backtracking.
- **Deletion vs insertion direction** — deletion from P (one character of P is skipped) uses `R[d-1]` with no shift (the text position does not advance). Insertion in T (one extra character in T is consumed) uses `(R[d-1] << 1) | 1` (advance in T). Swapping these operations counts the wrong edit type and gives wrong Levenshtein distances.

---

## Conclusion

The Bitap algorithm is the **canonical bit-parallel string matching method**:

- For exact matching with `m ≤ 64`, it achieves O(n) with a constant factor of one shift + one AND per character — matching the performance floor of any word-RAM algorithm.
- For approximate matching, it extends cleanly to k errors with O(kn) total work, no auxiliary data structures, and simple code — making it the standard for `agrep`-style tools.
- Wildcards, character classes, and case-insensitive matching reduce to preprocessing changes only — the inner loop remains unchanged.
- The multi-word extension handles arbitrary pattern lengths at the cost of O(⌈m/64⌉) per character.

**Key takeaway:**  
Bitap is the first choice for approximate matching when `m ≤ 64`. Its three-line inner loop — shift, OR 1, AND bitmask — is the most cache-friendly and branch-free possible implementation of DFA simulation. For `m > 64` or `k > m/4`, classical DP or Myers' algorithm may outperform the multi-word variant due to better cache behavior on wide rows.
