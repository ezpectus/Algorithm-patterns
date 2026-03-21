# Eertree Extensions — Palindromic Factorization

## Origin & Motivation

The Eertree (Palindromic Tree) was introduced by Mikhail Rubinchik and Arseny M. Shur in 2013 as an online data structure for all distinct palindromic substrings of a string. It builds in **O(n)** time and space, storing exactly `n + 2` nodes (one per distinct palindrome plus two root sentinels).

**Palindromic factorization** extends the Eertree with additional DP layers to answer: "in how many ways can the string be factored into palindromes?", "what is the minimum number of palindromes needed?", and "what is the factorization achieving that minimum?" These DP problems are solved in **O(n log n)** using the **series decomposition** of the Eertree — a structural property that groups palindromes into arithmetic series by their difference of lengths, enabling batch DP transitions.

Complexity: **O(n log n)** for palindromic factorization DP, **O(n)** for the Eertree itself.

---

## Where It Is Used

- Minimum palindrome factorization (fewest palindromes covering the string)
- Count of palindrome factorizations (number of ways)
- Palindromic richness and palindromic complexity analysis
- Competitive programming: palindrome partition DP
- Bioinformatics: palindromic structure in DNA sequences
- Combinatorics on words: palindromic defect, rich strings

---

## Key Definitions

**Palindromic factorization:** A sequence `w_1 w_2 ... w_k` where each `w_i` is a non-empty palindrome and their concatenation equals `s`.

**Palindromic tree (Eertree):** A trie-like structure where each node represents a distinct palindromic substring of `s`. Two root nodes: `odd_root` (length -1, imaginary) and `even_root` (length 0, empty palindrome). Each node `v` stores:
- `len[v]` — length of the palindrome
- `link[v]` — suffix link (longest proper palindromic suffix)
- `diff[v] = len[v] - len[link[v]]` — difference in lengths
- `series_link[v]` — link to the longest palindromic suffix with a different `diff`

**Series:** A maximal sequence of nodes `v, link[v], link[link[v]], ...` with the same `diff`. Within a series, palindromes form an arithmetic progression of lengths.

**Series link (`slink`):** Points to the first node outside the current series. Enables batch DP transitions in O(1) amortized per character.

---

## Eertree Structure

```
Nodes:
  odd_root:  len = -1, link = odd_root
  even_root: len = 0,  link = odd_root
  v (palindrome w): len[v], link[v] = longest proper pal-suffix of w

Edges:
  edge[v][c] = node for palindrome c + v + c (extension by character c)

Series decomposition:
  diff[v]   = len[v] - len[link[v]]
  slink[v]  = link[v]  if diff[v] != diff[link[v]]
              else slink[link[v]]
```

---

## Palindromic Factorization DPs

### DP 1 — Minimum number of palindromes

`dp[i]` = minimum number of palindromes in a factorization of `s[0..i-1]`.

Naive: for each palindromic suffix `s[j..i-1]`, update `dp[i] = min(dp[i], dp[j] + 1)`.
This is O(n²) if done directly. With series links: O(n log n).

For each node `v` in the series ending at position `i`, define:
`g[v]` = minimum `dp[i - len[slink[v]] - diff[v]]` among all nodes in the series.

```
g[v] = dp[i - len[slink[v]] - diff[v]]   if diff[v] != diff[link[v]]
     = min(g[link[v]], dp[i - len[slink[v]] - diff[v]])  otherwise
```

Then: `dp[i] = min over all palindromic suffixes v of (g[v] + 1)`

### DP 2 — Count of palindrome factorizations

`cnt[i]` = number of distinct palindrome factorizations of `s[0..i-1]`.

Similarly uses series decomposition with an auxiliary `h[v]` array storing partial sums within each series.

### DP 3 — Longest palindromic factorization

Variant: maximize the number of parts (always achievable as n single characters), or maximize the length of the longest part — solvable with the same framework.

---

## Complexity Analysis

| Operation | Time | Space |
|---|---|---|
| Build Eertree | O(n) | O(n * sigma) or O(n log sigma) with maps |
| Naive palindrome partition DP | O(n²) | O(n) |
| Series-link DP (min palindromes) | O(n log n) | O(n) |
| Series-link DP (count factorizations) | O(n log n) | O(n) |
| Distinct palindromic substrings count | O(n) | O(n) |

The O(n log n) bound comes from the fact that the number of distinct `diff` values along any suffix-link chain is O(log n) — the series decomposition partitions the chain into O(log n) series.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

const int MOD = 1e9 + 7;

// ================================================================
// EERTREE (PALINDROMIC TREE) with Series Links
// ================================================================
struct Eertree {
    struct Node {
        map<char, int> children;
        int link;       // suffix link
        int len;        // palindrome length
        int diff;       // len[v] - len[link[v]]
        int slink;      // series link: first ancestor with different diff
        int cnt;        // occurrences (for counting)
        // DP auxiliary values per node, updated per position
        ll g_min;       // for min-palindromes DP
        ll h_cnt;       // for count-factorizations DP
    };

    vector<Node> t;
    int last;           // last added node
    string s;

    // Node indices: 0 = odd_root (len=-1), 1 = even_root (len=0)
    Eertree() {
        // odd root
        t.push_back({.link=0, .len=-1, .diff=0, .slink=0,
                     .cnt=0, .g_min=0, .h_cnt=0});
        // even root
        t.push_back({.link=0, .len=0, .diff=0, .slink=0,
                     .cnt=0, .g_min=0, .h_cnt=0});
        last = 1;
    }

    int new_node(int link, int len) {
        Node v{};
        v.link  = link;
        v.len   = len;
        v.diff  = len - t[link].len;
        v.slink = (v.diff == t[link].diff) ? t[link].slink : link;
        v.cnt   = 0;
        v.g_min = 0;
        v.h_cnt = 0;
        t.push_back(v);
        return (int)t.size() - 1;
    }

    // Get suffix link for current last node when adding s[i]
    int get_link(int v, int i) {
        while (i - t[v].len - 1 < 0 || s[i - t[v].len - 1] != s[i])
            v = t[v].link;
        return v;
    }

    // Add character s[i], return the node for the new palindromic suffix
    int add_char(int i) {
        char c = s[i];
        int cur = get_link(last, i);
        if (!t[cur].children.count(c)) {
            int q = get_link(t[cur].link, i);
            t[cur].children[c] = new_node(
                t[q].children.count(c) ? t[q].children[c] : 1,
                t[cur].len + 2
            );
        }
        last = t[cur].children[c];
        t[last].cnt++;
        return last;
    }

    // Count occurrences: propagate cnt through suffix links
    void propagate_counts() {
        for (int v = (int)t.size()-1; v >= 2; v--)
            t[t[v].link].cnt += t[v].cnt;
    }

    int size() { return (int)t.size() - 2; } // distinct palindromes
};

// ================================================================
// DP: MINIMUM PALINDROME FACTORIZATION using series links
// dp[i] = min palindromes to cover s[0..i-1]
// ================================================================
vector<int> min_palindrome_partition(const string& str) {
    int n = str.size();
    Eertree et;
    et.s = str;

    const int INF = 1e9;
    vector<int> dp(n + 1, INF);
    dp[0] = 0;

    for (int i = 0; i < n; i++) {
        int v = et.add_char(i);

        // Process all palindromic suffixes via series links
        for (int u = v; u > 1; u = et.t[u].slink) {
            // Series of nodes with same diff ending at u
            // g_min[u] = min dp[i+1 - len[slink[u]] - diff[u] - diff[u]*k]
            //           for k = 0, 1, 2, ... within the series

            int sl = et.t[u].slink;
            int base_pos = i + 1 - et.t[sl].len - et.t[u].diff;

            et.t[u].g_min = (base_pos >= 0 && dp[base_pos] < INF)
                          ? dp[base_pos] : INF;

            if (et.t[u].diff == et.t[et.t[u].link].diff)
                et.t[u].g_min = min(et.t[u].g_min,
                                    et.t[et.t[u].link].g_min);

            if (et.t[u].g_min < INF)
                dp[i + 1] = min(dp[i + 1], (int)et.t[u].g_min + 1);
        }
    }

    return dp;
}

// ================================================================
// DP: COUNT PALINDROME FACTORIZATIONS (mod MOD)
// cnt[i] = number of ways to factor s[0..i-1] into palindromes
// ================================================================
vector<ll> count_palindrome_factorizations(const string& str) {
    int n = str.size();
    Eertree et;
    et.s = str;

    vector<ll> dp(n + 1, 0);
    dp[0] = 1;

    for (int i = 0; i < n; i++) {
        int v = et.add_char(i);

        for (int u = v; u > 1; u = et.t[u].slink) {
            int sl = et.t[u].slink;
            int base_pos = i + 1 - et.t[sl].len - et.t[u].diff;

            et.t[u].h_cnt = (base_pos >= 0) ? dp[base_pos] : 0;

            if (et.t[u].diff == et.t[et.t[u].link].diff)
                et.t[u].h_cnt = (et.t[u].h_cnt +
                                  et.t[et.t[u].link].h_cnt) % MOD;

            dp[i + 1] = (dp[i + 1] + et.t[u].h_cnt) % MOD;
        }
    }

    return dp;
}

// ================================================================
// RECONSTRUCT minimum palindrome partition path
// ================================================================
vector<string> reconstruct_min_partition(const string& s) {
    auto dp = min_palindrome_partition(s);
    int n = s.size();

    // Backtrack: find which palindromic suffix was used at each step
    // Rebuild Eertree to get palindromic suffixes at each position
    Eertree et2;
    et2.s = s;
    vector<int> last_node(n + 1, -1);

    // Store last node at each position for backtracking
    // (simplified: rerun and record suffix palindromes)
    vector<vector<pair<int,int>>> pal_suffixes(n); // {len, start}

    Eertree et3;
    et3.s = s;
    for (int i = 0; i < n; i++) {
        int v = et3.add_char(i);
        for (int u = v; u > 1; u = et3.t[u].link) {
            int len = et3.t[u].len;
            pal_suffixes[i].push_back({len, i - len + 1});
        }
    }

    // Backtrack dp
    vector<string> result;
    int pos = n;
    while (pos > 0) {
        for (auto [len, start] : pal_suffixes[pos - 1]) {
            if (start >= 0 && dp[start] + 1 == dp[pos]) {
                result.push_back(s.substr(start, len));
                pos = start;
                break;
            }
        }
    }
    reverse(result.begin(), result.end());
    return result;
}

// ================================================================
// ALL DISTINCT PALINDROMIC SUBSTRINGS with their occurrences
// ================================================================
vector<pair<string,int>> all_palindromes(const string& str) {
    Eertree et;
    et.s = str;
    for (int i = 0; i < (int)str.size(); i++) et.add_char(i);
    et.propagate_counts();

    vector<pair<string,int>> result;
    for (int v = 2; v < (int)et.t.size(); v++) {
        // Reconstruct palindrome string from node:
        // The palindrome at node v is the longest palindromic suffix
        // when it was created. We can recover by tracing suffix links
        // but simpler: store start position during build.
        // For demo: just show len and count
        result.push_back({"len=" + to_string(et.t[v].len),
                           et.t[v].cnt});
    }
    return result;
}

// ================================================================
// Usage
// ================================================================
int main() {
    // Minimum palindrome partition
    {
        string s = "abacaba";
        auto dp = min_palindrome_partition(s);
        printf("Min palindromes for '%s': %d\n", s.c_str(), dp[s.size()]);
        // "abacaba" is itself a palindrome → 1
        auto parts = reconstruct_min_partition(s);
        printf("Partition: ");
        for (auto& p : parts) printf("'%s' ", p.c_str());
        printf("\n");
    }
    {
        string s = "abacd";
        auto dp = min_palindrome_partition(s);
        printf("Min palindromes for '%s': %d\n", s.c_str(), dp[s.size()]);
        // "aba" + "c" + "d" = 3
        auto parts = reconstruct_min_partition(s);
        printf("Partition: ");
        for (auto& p : parts) printf("'%s' ", p.c_str());
        printf("\n");
    }

    // Count factorizations
    {
        string s = "aaa";
        auto cnt = count_palindrome_factorizations(s);
        printf("\nCount factorizations of '%s': %lld\n", s.c_str(), cnt[s.size()]);
        // "a"+"a"+"a", "a"+"aa", "aa"+"a", "aaa" = 4
    }
    {
        string s = "aaaa";
        auto cnt = count_palindrome_factorizations(s);
        printf("Count factorizations of '%s': %lld\n", s.c_str(), cnt[s.size()]);
        // 8
    }

    // Distinct palindromic substrings
    {
        string s = "abacaba";
        Eertree et;
        et.s = s;
        for (int i = 0; i < (int)s.size(); i++) et.add_char(i);
        et.propagate_counts();
        printf("\nDistinct palindromic substrings of '%s': %d\n",
               s.c_str(), et.size());
        for (int v = 2; v < (int)et.t.size(); v++)
            printf("  len=%d cnt=%d\n", et.t[v].len, et.t[v].cnt);
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class Eertree {
    public struct Node {
        public Dictionary<char, int> Children;
        public int Link, Len, Diff, Slink, Cnt;
        public long GMín, HCnt;
    }

    public List<Node> T = new();
    public int Last;
    public string S = "";

    public Eertree() {
        // odd root (index 0)
        T.Add(new Node { Children=new(), Link=0, Len=-1 });
        // even root (index 1)
        T.Add(new Node { Children=new(), Link=0, Len=0 });
        Last = 1;
    }

    private int NewNode(int link, int len) {
        int diff  = len - T[link].Len;
        int slink = (diff == T[link].Diff) ? T[link].Slink : link;
        T.Add(new Node {
            Children = new(), Link=link, Len=len,
            Diff=diff, Slink=slink, Cnt=0, GMín=0, HCnt=0
        });
        return T.Count - 1;
    }

    private int GetLink(int v, int i) {
        while (i - T[v].Len - 1 < 0 || S[i - T[v].Len - 1] != S[i])
            v = T[v].Link;
        return v;
    }

    public int AddChar(int i) {
        char c = S[i];
        int cur = GetLink(Last, i);
        if (!T[cur].Children.ContainsKey(c)) {
            int q   = GetLink(T[cur].Link, i);
            int lnk = T[q].Children.ContainsKey(c) ? T[q].Children[c] : 1;
            var node   = T[cur];
            node.Children[c] = NewNode(lnk, T[cur].Len + 2);
            T[cur] = node;
        }
        Last = T[cur].Children[c];
        var n2 = T[Last]; n2.Cnt++; T[Last] = n2;
        return Last;
    }

    public void PropagateCount() {
        for (int v = T.Count - 1; v >= 2; v--) {
            var lnk = T[v].Link;
            var n   = T[lnk]; n.Cnt += T[v].Cnt; T[lnk] = n;
        }
    }

    public int Size => T.Count - 2;
}

public class PalindromicFactorization {
    private const int MOD = 1_000_000_007;

    // ================================================================
    // MINIMUM PALINDROME PARTITION
    // ================================================================
    public static int[] MinPartition(string s) {
        int n = s.Length;
        var et = new Eertree { S = s };
        var dp = new int[n + 1];
        Array.Fill(dp, int.MaxValue / 2);
        dp[0] = 0;

        for (int i = 0; i < n; i++) {
            int v = et.AddChar(i);
            for (int u = v; u > 1; u = et.T[u].Slink) {
                int sl      = et.T[u].Slink;
                int basePos = i + 1 - et.T[sl].Len - et.T[u].Diff;

                long gMin = basePos >= 0 && dp[basePos] < int.MaxValue/2
                          ? dp[basePos] : int.MaxValue / 2;

                if (et.T[u].Diff == et.T[et.T[u].Link].Diff)
                    gMin = Math.Min(gMin, et.T[u].GMín);

                var nu = et.T[u]; nu.GMín = gMin; et.T[u] = nu;

                if (gMin < int.MaxValue / 2)
                    dp[i + 1] = Math.Min(dp[i + 1], (int)gMin + 1);
            }
        }
        return dp;
    }

    // ================================================================
    // COUNT PALINDROME FACTORIZATIONS
    // ================================================================
    public static long[] CountFactorizations(string s) {
        int n = s.Length;
        var et = new Eertree { S = s };
        var dp = new long[n + 1];
        dp[0] = 1;

        for (int i = 0; i < n; i++) {
            int v = et.AddChar(i);
            for (int u = v; u > 1; u = et.T[u].Slink) {
                int sl      = et.T[u].Slink;
                int basePos = i + 1 - et.T[sl].Len - et.T[u].Diff;

                long h = basePos >= 0 ? dp[basePos] : 0;
                if (et.T[u].Diff == et.T[et.T[u].Link].Diff)
                    h = (h + et.T[u].HCnt) % MOD;

                var nu = et.T[u]; nu.HCnt = h; et.T[u] = nu;
                dp[i + 1] = (dp[i + 1] + h) % MOD;
            }
        }
        return dp;
    }

    public static void Main() {
        // Min partition
        foreach (string s in new[]{"abacaba","abacd","amanaplanacanalpanama"}) {
            var dp = MinPartition(s);
            Console.WriteLine($"Min palindromes '{s}': {dp[s.Length]}");
        }

        Console.WriteLine();

        // Count factorizations
        foreach (string s in new[]{"aaa","aaaa","ab","abba"}) {
            var cnt = CountFactorizations(s);
            Console.WriteLine($"Count factorizations '{s}': {cnt[s.Length]}");
        }

        Console.WriteLine();

        // Distinct palindromic substrings
        string t = "abacaba";
        var et = new Eertree { S = t };
        for (int i = 0; i < t.Length; i++) et.AddChar(i);
        et.PropagateCount();
        Console.WriteLine($"Distinct palindromes in '{t}': {et.Size}");
        for (int v = 2; v < et.T.Count; v++)
            Console.WriteLine($"  len={et.T[v].Len} cnt={et.T[v].Cnt}");
    }
}
```

---

## Series Decomposition — Detail

The series decomposition is the structural insight enabling O(n log n) DP. Along any suffix-link chain from a node `v` to the root, consecutive nodes with the same `diff` form a **series**. Within a series, palindromes have lengths forming an arithmetic progression with common difference `d = diff[v]`.

```
Node chain: v → link[v] → link[link[v]] → ... → even_root
Example: palindromes of lengths 7, 5, 3, 1 with diff=2 form one series.
         Then lengths 3, 1 with diff=2 might be a separate series if the
         link from length 3 has diff ≠ 2 for the next node.
```

**Why O(log n) series per chain:** The shortest palindrome in a series of diff `d` has length ≥ d (since it must be longer than its suffix-link palindrome by exactly d). A new series has a strictly smaller diff. Diffs are positive integers that strictly decrease along the chain, and the minimum palindrome length ≥ sum of all diffs in the chain below it. This forces the number of distinct series to be O(log n).

**DP transition per series:** For each series, one auxiliary variable `g[u]` (or `h[u]`) accumulates the DP value for all palindromes in the series simultaneously, so the full series is processed in O(1) per node rather than O(series length).

---

## Pitfalls

- **Slink vs link confusion** — `slink[v]` skips the entire current series and lands at the first node with a different `diff`. Using `link[v]` in the DP outer loop instead of `slink[v]` degrades to O(n²) — the loop must iterate over series, not individual nodes.
- **g_min/h_cnt must be reset per position** — the auxiliary `g[u]` and `h[u]` arrays are per-position values, not permanent node properties. In the implementation above, they are stored in the node struct and overwritten at each position `i`. If a node is visited at position `i` and its auxiliary value is stale from position `i-1`, wrong DP values propagate. Ensure the update order visits the longest suffix first (which the outer loop from `v` downward to `slink` guarantees).
- **Eertree with map children is O(n log sigma)** — using `map<char,int>` for children gives O(log sigma) per lookup. For binary or small alphabets, use `int children[sigma]` arrays. For large alphabets (Unicode), `unordered_map` gives expected O(1) per lookup.
- **Suffix link of odd_root points to itself** — when traversing suffix links during `get_link`, the traversal must terminate at `odd_root` (index 0). If `odd_root.link = 0`, the loop `while s[i - len[v] - 1] != s[i]: v = link[v]` terminates because `len[odd_root] = -1` always satisfies `s[i - (-1) - 1] = s[i]`, i.e., `s[i] == s[i]`. This is the sentinel trick.
- **Propagate counts in reverse order** — `propagate_counts()` must iterate nodes from highest index to lowest (reverse creation order) because suffix links always point to earlier-created nodes. Iterating forward accumulates counts in the wrong order.
- **dp[0] = 0 for min, dp[0] = 1 for count** — the base case differs. For minimum palindromes, `dp[0] = 0` (empty string needs 0 palindromes). For counting factorizations, `dp[0] = 1` (one way to factor the empty string: the empty factorization). Swapping these gives wrong answers on all inputs.

---

## Complexity Summary

| Task | Time | Space |
|---|---|---|
| Build Eertree | O(n log sigma) | O(n log sigma) with map; O(n sigma) with array |
| Count distinct palindromic substrings | O(n) | O(n) |
| All palindromic substrings with occurrences | O(n) | O(n) |
| Minimum palindrome partition (naive DP) | O(n²) | O(n) |
| Minimum palindrome partition (series DP) | O(n log n) | O(n) |
| Count palindrome factorizations (series DP) | O(n log n) | O(n) |

---

## Conclusion

Eertree extensions for palindromic factorization represent the **state of the art for palindrome partition problems**:

- The base Eertree captures all n + O(1) distinct palindromic substrings in O(n) time and space with a single left-to-right scan.
- The series decomposition of suffix-link chains into O(log n) arithmetic progressions enables batch DP transitions, reducing palindromic partition DP from O(n²) to O(n log n).
- Both minimum palindrome count and total factorization count are handled by the same structural framework with only the DP recurrence changing.
- The series link `slink` is the key navigational primitive — it is to palindromic factorization DP what the failure function is to KMP.

**Key takeaway:**  
If a problem involves partitioning a string into palindromes — whether minimizing the count, counting all ways, or characterizing the structure — build the Eertree with series links and apply the appropriate DP. The O(n log n) complexity is tight and matches the information-theoretic lower bound for palindromic factorization problems where the answer itself can be Ω(n log n) bits.
