# DAWG — Directed Acyclic Word Graph

## Origin & Motivation

The Directed Acyclic Word Graph (DAWG), also known as the **Suffix Automaton (SAM)**, was introduced by Blumer, Blumer, Haussler, Ehrenfeucht, Chen, and Seiferas in 1985. It is the **minimal deterministic finite automaton** (DFA) that accepts exactly all suffixes of a string `s`. It is simultaneously:

- The smallest automaton recognizing all suffixes of `s`
- A compact representation of all substrings of `s`
- The structure underlying efficient substring search, distinct substring counting, and longest common substring computation

The DAWG has at most **2n − 1 nodes** and **3n − 4 edges** for a string of length n ≥ 3 — linear in the input size. It is constructed incrementally in **O(n)** time (O(n log σ) with sorted maps) by the online algorithm of Blumer et al., which is equivalent to the Suffix Automaton construction used in competitive programming.

The key distinction from a trie or suffix tree: the DAWG **merges equivalent states** — states that recognize the same set of right extensions — making it minimal and compact.

Complexity: **O(n)** build time and space (O(n log σ) with maps).

---

## Where It Is Used

- Substring search: does pattern P occur in text T? O(|P|) after O(|T|) build
- Count of distinct substrings of a string
- Longest common substring of two strings
- Number of occurrences of each substring
- Lexicographically k-th substring
- String matching with wildcards (extended DAWG)
- Bioinformatics: genome indexing, repeat detection
- Data compression (LZ78 variants)
- Competitive programming: string DP on automaton states

---

## Key Concepts

**State (node):** Each state represents an **equivalence class** of substrings — all substrings that have the same set of right extensions (positions where they end + 1 in `s`). The set of right extensions is called `endpos(v)`.

**Suffix link:** For every non-initial state `v`, the suffix link `link[v]` points to the state representing the longest proper suffix of `v`'s substrings that belongs to a different (strictly larger) `endpos` class. The suffix links form a tree rooted at the initial state — the **suffix link tree**, equivalent to the suffix tree of the reversed string.

**Transitions:** `trans[v][c]` = state reached by reading character `c` from state `v`. Transitions encode substring extension.

**Terminal states:** States corresponding to suffixes of `s` (their `endpos` sets contain position `n`). Found by following suffix links from the last state added.

**len[v]:** Length of the longest substring in the equivalence class of state `v`.  
**link[v]:** Suffix link — parent in suffix link tree.  
**minlen[v] = len[link[v]] + 1:** Length of the shortest substring in the class.

---

## Construction Algorithm

```
Initialize:
    last = initial state (len=0, link=-1)

For each character c = s[i]:
    cur = new state (len = last.len + 1)
    p = last
    while p != -1 and trans[p][c] undefined:
        trans[p][c] = cur
        p = link[p]

    if p == -1:
        link[cur] = initial
    else:
        q = trans[p][c]
        if len[q] == len[p] + 1:
            link[cur] = q
        else:
            clone = copy of q (same transitions and link)
            clone.len = len[p] + 1
            while p != -1 and trans[p][c] == q:
                trans[p][c] = clone
                p = link[p]
            link[q] = clone
            link[cur] = clone

    last = cur
```

Each character addition takes amortized O(1) time (O(log σ) with sorted maps).

---

## Complexity Analysis

| Property | Bound | Notes |
|---|---|---|
| States | ≤ 2n − 1 | Tight for string aaa...ab |
| Transitions | ≤ 3n − 4 | Tight for specific strings |
| Build time | O(n) / O(n log σ) | With array / map for transitions |
| Build space | O(n σ) / O(n log σ) | With array / map |
| Substring search | O(m) | After O(n) build |
| Distinct substrings | O(n) | Count transitions reachable from initial |
| k-th substring | O(m log n) | With cnt[] on states |
| LCS of two strings | O(n + m) | Build on one, traverse with other |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// SUFFIX AUTOMATON (DAWG)
// ================================================================
struct SuffixAutomaton {
    struct State {
        int len, link;
        map<char, int> next;
        bool is_clone;
        ll cnt; // number of times this state's substrings appear
    };

    vector<State> st;
    int last;

    void init() {
        st.clear();
        st.push_back({0, -1, {}, false, 0}); // initial state
        last = 0;
    }

    void extend(char c) {
        // Check if transition already exists (for efficiency)
        if (st[last].next.count(c)) {
            int q = st[last].next[c];
            if (st[q].len == st[last].len + 1) {
                last = q;
                return;
            }
            int clone = st.size();
            st.push_back({st[last].len + 1, st[q].link,
                          st[q].next, true, 0});
            while (last != -1 && st[last].next.count(c) &&
                   st[last].next[c] == q) {
                st[last].next[c] = clone;
                last = st[last].link;
            }
            st[q].link = clone;
            last = clone;
            return;
        }

        int cur = st.size();
        st.push_back({st[last].len + 1, -1, {}, false, 1});
        int p = last;

        while (p != -1 && !st[p].next.count(c)) {
            st[p].next[c] = cur;
            p = st[p].link;
        }

        if (p == -1) {
            st[cur].link = 0;
        } else {
            int q = st[p].next[c];
            if (st[q].len == st[p].len + 1) {
                st[cur].link = q;
            } else {
                int clone = st.size();
                st.push_back({st[p].len + 1, st[q].link,
                              st[q].next, true, 0});
                while (p != -1 && st[p].next.count(c) &&
                       st[p].next[c] == q) {
                    st[p].next[c] = clone;
                    p = st[p].link;
                }
                st[q].link = clone;
                st[cur].link = clone;
            }
        }

        last = cur;
    }

    // Build from string
    void build(const string& s) {
        init();
        for (char c : s) extend(c);
    }

    // ----------------------------------------------------------------
    // Count occurrences of each state's substrings
    // Topological sort by len (decreasing), propagate cnt to link
    // ----------------------------------------------------------------
    void compute_cnt() {
        int n = st.size();
        // Mark non-clone states as cnt=1 (each suffix position)
        for (int i = 0; i < n; i++)
            if (!st[i].is_clone) st[i].cnt = 1;

        // Sort by len descending
        vector<int> order(n);
        iota(order.begin(), order.end(), 0);
        sort(order.begin(), order.end(), [&](int a, int b){
            return st[a].len > st[b].len;
        });

        // Propagate counts up the suffix link tree
        for (int v : order)
            if (st[v].link != -1)
                st[st[v].link].cnt += st[v].cnt;
    }

    // ----------------------------------------------------------------
    // Count distinct substrings
    // Each transition in the DAG (ignoring suffix links) corresponds
    // to exactly one distinct substring length from that state.
    // Total = sum over all states of (len[v] - len[link[v]])
    // ----------------------------------------------------------------
    ll count_distinct_substrings() {
        ll result = 0;
        for (int v = 1; v < (int)st.size(); v++)
            result += st[v].len - st[st[v].link].len;
        return result;
    }

    // ----------------------------------------------------------------
    // Substring search: does pattern p occur in s?
    // O(|p|) after build
    // ----------------------------------------------------------------
    bool contains(const string& p) {
        int cur = 0;
        for (char c : p) {
            if (!st[cur].next.count(c)) return false;
            cur = st[cur].next[c];
        }
        return true;
    }

    // ----------------------------------------------------------------
    // Count occurrences of pattern p in s
    // ----------------------------------------------------------------
    ll count_occurrences(const string& p) {
        int cur = 0;
        for (char c : p) {
            if (!st[cur].next.count(c)) return 0;
            cur = st[cur].next[c];
        }
        return st[cur].cnt;
    }

    // ----------------------------------------------------------------
    // Longest Common Substring of s (built) and t (traversal)
    // ----------------------------------------------------------------
    string lcs(const string& t) {
        int cur = 0, curlen = 0;
        int best = 0, best_pos = 0;

        for (int i = 0; i < (int)t.size(); i++) {
            char c = t[i];
            while (cur != 0 && !st[cur].next.count(c)) {
                cur = st[cur].link;
                curlen = st[cur].len;
            }
            if (st[cur].next.count(c)) {
                cur = st[cur].next[c];
                curlen++;
            }
            if (curlen > best) {
                best = curlen;
                best_pos = i;
            }
        }

        return t.substr(best_pos - best + 1, best);
    }

    // ----------------------------------------------------------------
    // Lexicographically k-th distinct substring (1-indexed)
    // Requires compute_dp_cnt() — number of distinct substrings
    // reachable from each state (including current state's contribution)
    // ----------------------------------------------------------------
    vector<ll> dp_cnt; // dp_cnt[v] = distinct substrings reachable from v

    void compute_dp_cnt() {
        int n = st.size();
        dp_cnt.assign(n, 0);

        // Topological order (DAG of transitions, not suffix link tree)
        vector<int> order;
        vector<int> indeg(n, 0);
        for (int v = 0; v < n; v++)
            for (auto& [c, u] : st[v].next)
                indeg[u]++;

        queue<int> q;
        for (int v = 0; v < n; v++)
            if (indeg[v] == 0) q.push(v);

        while (!q.empty()) {
            int v = q.front(); q.pop();
            order.push_back(v);
            for (auto& [c, u] : st[v].next)
                if (--indeg[u] == 0) q.push(u);
        }

        // Process in reverse topological order
        for (int i = order.size()-1; i >= 0; i--) {
            int v = order[i];
            dp_cnt[v] = (v != 0) ? 1 : 0; // count the empty suffix only for init=0
            for (auto& [c, u] : st[v].next)
                dp_cnt[v] += dp_cnt[u];
        }
        // dp_cnt[0] = total distinct substrings (including empty)
    }

    string kth_substring(ll k) {
        // k is 1-indexed among non-empty distinct substrings
        compute_dp_cnt();
        string result;
        int cur = 0;
        while (k > 0) {
            for (auto& [c, u] : st[cur].next) {
                if (k <= dp_cnt[u]) {
                    result += c;
                    cur = u;
                    k--;
                    break;
                }
                k -= dp_cnt[u];
            }
        }
        return result;
    }

    // ----------------------------------------------------------------
    // All terminal states (states corresponding to suffixes of s)
    // Terminal = all states on the suffix link path from `last` to root
    // ----------------------------------------------------------------
    vector<bool> terminal_states() {
        int n = st.size();
        vector<bool> term(n, false);
        int v = last;
        while (v != -1) {
            term[v] = true;
            v = st[v].link;
        }
        return term;
    }

    // ----------------------------------------------------------------
    // Number of distinct substrings of length exactly L
    // ----------------------------------------------------------------
    ll count_length(int L) {
        ll result = 0;
        for (int v = 1; v < (int)st.size(); v++) {
            int lo = st[st[v].link].len + 1;
            int hi = st[v].len;
            if (lo <= L && L <= hi) result += st[v].cnt; // occurrences
            // For distinct count: just check if L in [lo, hi]
        }
        return result; // returns occurrences summed, not distinct count
    }
};

// ================================================================
// GENERALIZED SUFFIX AUTOMATON (DAWG for multiple strings)
// ================================================================
struct GeneralizedSAM {
    SuffixAutomaton sam;

    void init() { sam.init(); }

    // Add a new string to the automaton
    // Call reset_last() between strings to avoid spurious transitions
    void add_string(const string& s) {
        sam.last = 0; // reset to initial state between strings
        for (char c : s) sam.extend(c);
    }

    // LCS of all added strings:
    // Build generalized SAM, then for each state track which strings
    // pass through it — the LCS length = max len[v] where all strings visit v
};

// ================================================================
// Usage
// ================================================================
int main() {
    SuffixAutomaton sa;

    // --- Basic usage ---
    sa.build("abcbc");
    sa.compute_cnt();

    printf("String: 'abcbc'\n");
    printf("States: %d\n", (int)sa.st.size());
    printf("Distinct substrings: %lld\n", sa.count_distinct_substrings());
    printf("Contains 'bcb': %s\n", sa.contains("bcb") ? "yes" : "no");
    printf("Contains 'abd': %s\n", sa.contains("abd") ? "yes" : "no");
    printf("Occurrences of 'bc': %lld\n", sa.count_occurrences("bc"));
    printf("Occurrences of 'b':  %lld\n", sa.count_occurrences("b"));

    // --- LCS ---
    sa.build("abcde");
    string lcs_result = sa.lcs("zbcdf");
    printf("\nLCS of 'abcde' and 'zbcdf': '%s'\n", lcs_result.c_str());

    // --- k-th substring ---
    sa.build("aab");
    sa.compute_cnt();
    printf("\nSubstrings of 'aab' in lex order:\n");
    // Distinct substrings of "aab": a, aa, aab, ab, b — 5 total
    printf("Distinct count: %lld\n", sa.count_distinct_substrings());
    for (int k = 1; k <= sa.count_distinct_substrings(); k++)
        printf("  %d-th: '%s'\n", k, sa.kth_substring(k).c_str());

    // --- Terminal states ---
    sa.build("abc");
    auto term = sa.terminal_states();
    printf("\nTerminal states of 'abc': ");
    for (int v = 0; v < (int)sa.st.size(); v++)
        if (term[v]) printf("%d(len=%d) ", v, sa.st[v].len);
    printf("\n");

    // --- Generalized SAM ---
    printf("\n--- Generalized SAM ---\n");
    GeneralizedSAM gsam;
    gsam.init();
    gsam.add_string("abcd");
    gsam.add_string("bcde");
    printf("States in generalized SAM: %d\n", (int)gsam.sam.st.size());
    printf("LCS of 'abcd' and 'bcde': '%s'\n",
           gsam.sam.lcs("bcde").c_str());

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

public class SuffixAutomaton {
    public class State {
        public int Len, Link;
        public Dictionary<char, int> Next = new();
        public bool IsClone;
        public long Cnt;
    }

    public List<State> St = new();
    public int Last;

    public void Init() {
        St.Clear();
        St.Add(new State { Len = 0, Link = -1, IsClone = false, Cnt = 0 });
        Last = 0;
    }

    public void Extend(char c) {
        int cur = St.Count;
        St.Add(new State { Len = St[Last].Len + 1, Link = -1,
                           IsClone = false, Cnt = 1 });
        int p = Last;

        while (p != -1 && !St[p].Next.ContainsKey(c)) {
            St[p].Next[c] = cur;
            p = St[p].Link;
        }

        if (p == -1) {
            St[cur].Link = 0;
        } else {
            int q = St[p].Next[c];
            if (St[q].Len == St[p].Len + 1) {
                St[cur].Link = q;
            } else {
                int clone = St.Count;
                St.Add(new State {
                    Len     = St[p].Len + 1,
                    Link    = St[q].Link,
                    Next    = new Dictionary<char,int>(St[q].Next),
                    IsClone = true,
                    Cnt     = 0
                });
                while (p != -1 && St[p].Next.ContainsKey(c) &&
                       St[p].Next[c] == q) {
                    St[p].Next[c] = clone;
                    p = St[p].Link;
                }
                St[q].Link  = clone;
                St[cur].Link = clone;
            }
        }
        Last = cur;
    }

    public void Build(string s) { Init(); foreach (char c in s) Extend(c); }

    public void ComputeCnt() {
        int n = St.Count;
        for (int i = 0; i < n; i++) if (!St[i].IsClone) St[i].Cnt = 1;
        var order = Enumerable.Range(0, n)
            .OrderByDescending(v => St[v].Len).ToArray();
        foreach (int v in order)
            if (St[v].Link != -1) St[St[v].Link].Cnt += St[v].Cnt;
    }

    public long CountDistinctSubstrings() {
        long res = 0;
        for (int v = 1; v < St.Count; v++)
            res += St[v].Len - St[St[v].Link].Len;
        return res;
    }

    public bool Contains(string p) {
        int cur = 0;
        foreach (char c in p) {
            if (!St[cur].Next.TryGetValue(c, out cur)) return false;
        }
        return true;
    }

    public long CountOccurrences(string p) {
        int cur = 0;
        foreach (char c in p) {
            if (!St[cur].Next.TryGetValue(c, out cur)) return 0;
        }
        return St[cur].Cnt;
    }

    public string LCS(string t) {
        int cur = 0, curLen = 0, best = 0, bestPos = 0;
        for (int i = 0; i < t.Length; i++) {
            char c = t[i];
            while (cur != 0 && !St[cur].Next.ContainsKey(c)) {
                cur = St[cur].Link;
                curLen = St[cur].Len;
            }
            if (St[cur].Next.TryGetValue(c, out int nx)) {
                cur = nx; curLen++;
            }
            if (curLen > best) { best = curLen; bestPos = i; }
        }
        return best == 0 ? "" : t.Substring(bestPos - best + 1, best);
    }

    public bool[] TerminalStates() {
        var term = new bool[St.Count];
        int v = Last;
        while (v != -1) { term[v] = true; v = St[v].Link; }
        return term;
    }

    public string KthSubstring(long k) {
        // Compute dp_cnt
        int n = St.Count;
        var dpCnt = new long[n];
        var indeg = new int[n];
        foreach (var s in St)
            foreach (var (_, u) in s.Next) indeg[u]++;

        var queue = new Queue<int>();
        for (int v = 0; v < n; v++) if (indeg[v] == 0) queue.Enqueue(v);
        var order = new List<int>();
        while (queue.Count > 0) {
            int v = queue.Dequeue(); order.Add(v);
            foreach (var (_, u) in St[v].Next)
                if (--indeg[u] == 0) queue.Enqueue(u);
        }

        for (int i = order.Count - 1; i >= 0; i--) {
            int v = order[i];
            dpCnt[v] = v != 0 ? 1 : 0;
            foreach (var (_, u) in St[v].Next) dpCnt[v] += dpCnt[u];
        }

        var result = new System.Text.StringBuilder();
        int cur = 0;
        while (k > 0) {
            foreach (var (c, u) in St[cur].Next.OrderBy(p => p.Key)) {
                if (k <= dpCnt[u]) {
                    result.Append(c); cur = u; k--; break;
                }
                k -= dpCnt[u];
            }
        }
        return result.ToString();
    }

    public static void Main() {
        var sa = new SuffixAutomaton();

        sa.Build("abcbc");
        sa.ComputeCnt();
        Console.WriteLine($"String: 'abcbc'");
        Console.WriteLine($"States: {sa.St.Count}");
        Console.WriteLine($"Distinct substrings: {sa.CountDistinctSubstrings()}");
        Console.WriteLine($"Contains 'bcb': {sa.Contains("bcb")}");
        Console.WriteLine($"Contains 'abd': {sa.Contains("abd")}");
        Console.WriteLine($"Occurrences 'bc': {sa.CountOccurrences("bc")}");

        sa.Build("abcde");
        Console.WriteLine($"\nLCS('abcde','zbcdf'): '{sa.LCS("zbcdf")}'");

        sa.Build("aab");
        sa.ComputeCnt();
        long total = sa.CountDistinctSubstrings();
        Console.WriteLine($"\nDistinct substrings of 'aab': {total}");
        for (long k = 1; k <= total; k++)
            Console.WriteLine($"  {k}-th: '{sa.KthSubstring(k)}'");

        sa.Build("abc");
        var term = sa.TerminalStates();
        Console.Write("Terminal states: ");
        for (int v = 0; v < sa.St.Count; v++)
            if (term[v]) Console.Write($"{v}(len={sa.St[v].Len}) ");
        Console.WriteLine();
    }
}
```

---

## Structural Properties

### State Count Bound

| String | States | Transitions |
|---|---|---|
| `abcde...` (all distinct) | 2n - 1 | 2n - 2 |
| `aaa...a` (all same) | n + 1 | n |
| `aaa...ab` (tight for states) | 2n - 1 | 3n - 4 |

### Suffix Link Tree = Suffix Tree of Reverse

The suffix link tree of the DAWG of `s` is isomorphic to the suffix tree of `reverse(s)`. This deep duality connects the two fundamental string data structures.

### endpos Sets and Suffix Links

```
endpos(v) = set of ending positions of substrings in class v
endpos(link[v]) ⊋ endpos(v)   (strict superset)

For states u, v (neither ancestor of the other in suffix link tree):
endpos(u) ∩ endpos(v) = ∅
```

This means the suffix link tree has the structure of a partition tree over ending positions — exactly the structure of a suffix tree.

---

## DAWG vs Suffix Tree vs Suffix Array

| Structure | Space | Build | Substring search | Distinct substrings | LCS |
|---|---|---|---|---|---|
| Suffix Trie | O(n²) | O(n²) | O(m) | O(n²) | O(n+m) |
| Suffix Tree | O(n) | O(n) | O(m) | O(n) | O(n+m) |
| Suffix Array | O(n) | O(n) | O(m log n) | O(n) | O(n+m) |
| DAWG (SAM) | O(n) | O(n) | O(m) | O(n) | O(n+m) |

DAWG and suffix tree are asymptotically equivalent but differ in practice: DAWG has a cleaner online construction, while suffix tree has simpler substring extraction (each edge stores a range in the original string).

---

## Pitfalls

- **Clone state cnt must be 0** — when cloning state `q` into `clone`, set `clone.cnt = 0`. The clone represents the same substrings as `q` but is not a new occurrence endpoint. Only non-clone states are initialized with `cnt = 1`. Giving `clone` a nonzero initial count inflates occurrence counts.
- **Suffix link of initial state is -1, not 0** — the initial state (index 0) has `link = -1` (no parent). When traversing suffix links, always check `p != -1` before accessing `st[p].link`. Indexing `st[-1]` is undefined behavior in C++ and an exception in C#.
- **Topological sort for cnt propagation must respect suffix link tree** — propagating `cnt` requires processing states from deepest (largest `len`) to shallowest. Sorting by `len` descending is sufficient since `len[link[v]] < len[v]` always. Using BFS order of transitions (not suffix links) gives wrong propagation.
- **Generalized SAM needs last reset between strings** — when building a generalized SAM for multiple strings, reset `last = 0` (the initial state) between strings. Without this reset, the extension of the second string continues from where the first ended, creating spurious transitions between unrelated strings.
- **k-th substring requires sorted transitions** — when traversing the automaton to find the k-th lexicographic substring, transitions must be iterated in sorted character order. `map<char,int>` preserves order; `unordered_map` does not. Using `unordered_map` gives arbitrary ordering and wrong k-th answers.
- **dp_cnt includes the empty string at state 0** — the DP counting distinct substrings reachable from each state counts the empty string at the initial state. When answering "k-th non-empty substring", do not count the empty string. Initialize `dp_cnt[0] = 0` (not 1) and `dp_cnt[v] = 1` for all other states before summing children.
- **LCS traversal must follow suffix links on mismatch** — when the current character of `t` has no transition from the current DAWG state, follow suffix links (reducing `curLen` to `len[link[cur]]`) until a matching transition is found or the initial state is reached. Jumping directly to the initial state on mismatch misses the longest possible match.

---

## Conclusion

The DAWG (Suffix Automaton) is the **most compact and versatile string index structure**:

- **Minimal automaton** — recognizes all suffixes with the fewest possible states (proved optimal by Myhill-Nerode theorem).
- **Universal substring index** — substring search, occurrence counting, distinct substring enumeration, and LCS all reduce to O(m) or O(n) traversals after O(n) build.
- **Dual structure** — the suffix link tree is the suffix tree of the reversed string, giving a deep connection between forward and backward string structure.
- **Online construction** — characters are processed left to right with O(1) amortized work per character, enabling streaming applications.

**Key takeaway:**  
When the problem involves any combination of substring search, counting, enumeration, or comparison on a fixed text, the DAWG is the canonical choice. It matches the suffix tree in asymptotic complexity while having a simpler, fully online construction algorithm. The suffix link tree is as powerful as the suffix tree for most competitive programming applications — treat them as two views of the same underlying structure.
