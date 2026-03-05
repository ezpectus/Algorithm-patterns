# FM-Index — Substring Query Index over BWT

## Origin & Motivation

FM-Index (Full-text index in Minute space), introduced by Ferragina and Manzini in 2000, is a compressed full-text index built on top of the **Burrows-Wheeler Transform (BWT)**. It supports fast substring search while using space proportional to the compressed size of the input, making it practical for large texts such as DNA sequences or document corpora.

The core insight: BWT rearranges characters to cluster equal characters together, enabling compression. FM-Index exploits this structure to answer **count** and **locate** queries without decompressing the text.

---

## Where It's Used

- Bioinformatics (short-read alignment: BWA, Bowtie)
- Full-text search engines over compressed corpora
- Suffix array-based string processing
- Compressed self-indexes replacing suffix trees

---

## Key Concepts

| Concept | Role |
|---|---|
| BWT | Permutation of input enabling compression and backward search |
| Suffix Array (SA) | Sorted array of all suffixes; BWT derived from it |
| F column | First column of BWT matrix (sorted characters) |
| L column | Last column of BWT matrix (the BWT itself) |
| C[c] | Number of characters in BWT strictly smaller than c |
| Occ(c, i) | Number of occurrences of character c in L[1..i] |
| LF-mapping | Maps row i in L to corresponding row in F: LF(i) = C[L[i]] + Occ(L[i], i) |

---

## Core Idea

**Backward Search** — the fundamental operation of FM-Index:

Given pattern `P[1..m]`, search is performed right-to-left over pattern characters while maintaining a range `[lo, hi]` in the suffix array. The range represents all suffixes that have the current suffix of P as a prefix.

```
Initialize: [lo, hi] = [1, n]
For i = m downto 1:
    c = P[i]
    lo = C[c] + Occ(c, lo - 1) + 1
    hi = C[c] + Occ(c, hi)
    if lo > hi: pattern not found
Count = hi - lo + 1
```

Each step costs `O(1)` if Occ is answered in `O(1)` (via precomputed rank structure or wavelet tree), giving total count query time `O(m)`.

**Locate** (finding actual positions) requires a sampled suffix array — every `k`-th SA entry is stored. To locate, follow LF-mapping steps until a sampled position is hit; cost is `O(m + occ * k)`.

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Build (SA + BWT) | O(n) or O(n log n) | O(n) |
| Count query | O(m) | — |
| Locate query | O(m + occ * k) | O(n/k) for SA samples |
| Index space | O(n log sigma) bits | Approaches nH_k compressed |

- `n` — text length
- `m` — pattern length
- `occ` — number of occurrences
- `k` — SA sample rate
- `sigma` — alphabet size
- `H_k` — k-th order empirical entropy of the text

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// Build suffix array via O(n log n) prefix doubling
vector<int> buildSA(const string& s) {
    int n = s.size();
    vector<int> sa(n), rank_(n), tmp(n);
    iota(sa.begin(), sa.end(), 0);
    for (int i = 0; i < n; i++) rank_[i] = s[i];

    for (int gap = 1; gap < n; gap <<= 1) {
        auto cmp = [&](int a, int b) {
            if (rank_[a] != rank_[b]) return rank_[a] < rank_[b];
            int ra = a + gap < n ? rank_[a + gap] : -1;
            int rb = b + gap < n ? rank_[b + gap] : -1;
            return ra < rb;
        };
        sort(sa.begin(), sa.end(), cmp);
        tmp[sa[0]] = 0;
        for (int i = 1; i < n; i++)
            tmp[sa[i]] = tmp[sa[i-1]] + (cmp(sa[i-1], sa[i]) ? 1 : 0);
        rank_ = tmp;
    }
    return sa;
}

// Build BWT from suffix array
string buildBWT(const string& s, const vector<int>& sa) {
    int n = s.size();
    string bwt(n, '\0');
    for (int i = 0; i < n; i++)
        bwt[i] = sa[i] == 0 ? s[n-1] : s[sa[i]-1];
    return bwt;
}

struct FMIndex {
    string bwt;
    vector<int> sa;
    // C[c] = number of chars in BWT strictly less than c
    array<int, 256> C{};
    // Occ[c][i] = occurrences of c in bwt[0..i-1]  (prefix counts, size n+1)
    // For large sigma use wavelet tree; here we use flat array for simplicity
    vector<array<int, 256>> Occ;
    int n;

    FMIndex(const string& text) {
        // Append sentinel '$' (smallest char, value 0 conceptually)
        string s = text + "$";
        n = s.size();
        sa = buildSA(s);
        bwt = buildBWT(s, sa);

        // Build C
        C.fill(0);
        for (char c : bwt) C[(unsigned char)c]++;
        // prefix sum: C[c] = number of chars strictly less than c
        int total = 0;
        for (int i = 0; i < 256; i++) {
            int cnt = C[i];
            C[i] = total;
            total += cnt;
        }

        // Build Occ prefix counts (n+1 rows x 256 cols)
        // Note: for large n/sigma use a compressed structure
        Occ.assign(n + 1, {});
        Occ[0].fill(0);
        for (int i = 0; i < n; i++) {
            Occ[i+1] = Occ[i];
            Occ[i+1][(unsigned char)bwt[i]]++;
        }
    }

    // Returns count of pattern occurrences in original text
    int count(const string& pattern) const {
        int m = pattern.size();
        int lo = 0, hi = n - 1;
        for (int i = m - 1; i >= 0 && lo <= hi; i--) {
            unsigned char c = pattern[i];
            lo = C[c] + Occ[lo][c];
            hi = C[c] + Occ[hi + 1][c] - 1;
        }
        return max(0, hi - lo + 1);
    }

    // Returns sorted list of starting positions of all occurrences
    // Uses suffix array directly (locate via SA range)
    vector<int> locate(const string& pattern) const {
        int m = pattern.size();
        int lo = 0, hi = n - 1;
        for (int i = m - 1; i >= 0 && lo <= hi; i--) {
            unsigned char c = pattern[i];
            lo = C[c] + Occ[lo][c];
            hi = C[c] + Occ[hi + 1][c] - 1;
        }
        vector<int> positions;
        for (int i = lo; i <= hi; i++)
            positions.push_back(sa[i]);
        sort(positions.begin(), positions.end());
        return positions;
    }
};

// --- Usage ---
int main() {
    string text = "mississippi";
    FMIndex fm(text);

    string pat = "issi";
    cout << "Count of '" << pat << "': " << fm.count(pat) << "\n";
    for (int pos : fm.locate(pat))
        cout << "  at position " << pos << "\n";

    return 0;
}
```

**Notes on the C++ implementation:**
- The flat `Occ[n+1][256]` array is O(n * sigma) space — suitable for small alphabets. For DNA (sigma=4) or large texts, replace with a **wavelet tree** or **rank/select structure** for O(n log sigma) bits.
- SA is stored in full here (O(n log n) bits). In a compressed FM-Index, store only every k-th sample and recover positions via LF-mapping steps.

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class FMIndex {
    private readonly string bwt;
    private readonly int[] sa;
    private readonly int[] C;           // C[c] = chars strictly less than c in BWT
    private readonly int[,] Occ;        // Occ[i, c] = occurrences of c in bwt[0..i-1]
    private readonly int n;

    public FMIndex(string text) {
        string s = text + "$";
        n = s.Length;
        sa = BuildSA(s);
        bwt = BuildBWT(s, sa);

        // Build C array (256 ASCII)
        C = new int[256];
        foreach (char c in bwt) C[(byte)c]++;
        int total = 0;
        for (int i = 0; i < 256; i++) {
            int cnt = C[i];
            C[i] = total;
            total += cnt;
        }

        // Build Occ prefix table
        Occ = new int[n + 1, 256];
        for (int i = 0; i < n; i++) {
            for (int c = 0; c < 256; c++)
                Occ[i + 1, c] = Occ[i, c];
            Occ[i + 1, (byte)bwt[i]]++;
        }
    }

    public int Count(string pattern) {
        int lo = 0, hi = n - 1;
        for (int i = pattern.Length - 1; i >= 0 && lo <= hi; i--) {
            byte c = (byte)pattern[i];
            lo = C[c] + Occ[lo, c];
            hi = C[c] + Occ[hi + 1, c] - 1;
        }
        return Math.Max(0, hi - lo + 1);
    }

    public List<int> Locate(string pattern) {
        int lo = 0, hi = n - 1;
        for (int i = pattern.Length - 1; i >= 0 && lo <= hi; i--) {
            byte c = (byte)pattern[i];
            lo = C[c] + Occ[lo, c];
            hi = C[c] + Occ[hi + 1, c] - 1;
        }
        var positions = new List<int>();
        for (int i = lo; i <= hi; i++)
            positions.Add(sa[i]);
        positions.Sort();
        return positions;
    }

    // O(n log n) suffix array construction via prefix doubling
    private static int[] BuildSA(string s) {
        int n = s.Length;
        int[] sa = Enumerable.Range(0, n).ToArray();
        int[] rank = s.Select(c => (int)c).ToArray();
        int[] tmp = new int[n];

        for (int gap = 1; gap < n; gap <<= 1) {
            int g = gap;
            int[] r = rank;
            Array.Sort(sa, (a, b) => {
                if (r[a] != r[b]) return r[a].CompareTo(r[b]);
                int ra = a + g < n ? r[a + g] : -1;
                int rb = b + g < n ? r[b + g] : -1;
                return ra.CompareTo(rb);
            });
            tmp[sa[0]] = 0;
            for (int i = 1; i < n; i++) {
                int ra1 = r[sa[i - 1]], ra2 = r[sa[i]];
                int rb1 = sa[i - 1] + g < n ? r[sa[i - 1] + g] : -1;
                int rb2 = sa[i] + g < n ? r[sa[i] + g] : -1;
                tmp[sa[i]] = tmp[sa[i - 1]] + ((ra1 == ra2 && rb1 == rb2) ? 0 : 1);
            }
            Array.Copy(tmp, rank, n);
        }
        return sa;
    }

    private static string BuildBWT(string s, int[] sa) {
        int n = s.Length;
        char[] bwt = new char[n];
        for (int i = 0; i < n; i++)
            bwt[i] = sa[i] == 0 ? s[n - 1] : s[sa[i] - 1];
        return new string(bwt);
    }

    // --- Usage ---
    public static void Main() {
        var fm = new FMIndex("mississippi");
        string pat = "issi";
        Console.WriteLine($"Count of '{pat}': {fm.Count(pat)}");
        foreach (int pos in fm.Locate(pat))
            Console.WriteLine($"  at position {pos}");
    }
}
```

---

## Pitfalls

- **Sentinel character** must be strictly smaller than all characters in the text and unique. Typically `$` or `\0`. Without it, the suffix array construction and BWT are ill-defined.
- **Occ table size** — naive `O(n * sigma)` is fine for DNA but unusable for large alphabets. Use a wavelet tree or sampled rank structure.
- **LF-mapping off-by-one** — the range `[lo, hi]` is closed. The Occ query uses `Occ[lo]` (exclusive) and `Occ[hi+1]` (inclusive prefix count). Confusing these is the most common implementation bug.
- **Locate cost** — count is O(m) but locate is O(m + occ * k). For high-occurrence patterns with large k, this dominates. Tune k to balance space and locate speed.
- **SA construction** — the simple prefix-doubling approach above is O(n log² n) due to the inner sort. A proper O(n log n) variant avoids the sort by using a counting sort on the secondary key. DC3/SA-IS give O(n).
- **Pattern not found** — when `lo > hi` after any step, the range is empty. Return count = 0 and empty locate list immediately.

---

## Conclusion

FM-Index is the standard structure for **compressed full-text indexing**:

- Count queries run in **O(m)** independent of text length.
- Space approaches the compressed size of the text (nH_k + lower order terms).
- Locate queries are O(m + occ * k) with a space/time tradeoff via SA sampling rate k.
- The backbone of all major short-read DNA aligners (BWA-MEM, Bowtie2).

**Key takeaway:**  
FM-Index trades locate speed for dramatic space reduction versus suffix arrays or suffix trees. For count-heavy workloads on large texts, it is unmatched. For locate-heavy workloads, tune the SA sample rate or pair with additional acceleration structures.

---
