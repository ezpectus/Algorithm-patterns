# 🧠 Suffix Automaton + DP — Substring Enumeration Engine

## 📜 Origin & Motivation

The **Suffix Automaton (SAM)** is a compact deterministic finite automaton that recognizes all substrings of a given string.  
It is built in **O(n)** time and space, and supports powerful operations:

- Count **distinct substrings**
- Enumerate substrings in **lexicographic order**
- Find the **k-th substring**
- Solve **substring matching** and **LCS** problems

Unlike suffix trees, SAM is **smaller**, **faster**, and easier to implement.

> Suffix Automaton is the go-to structure for substring analytics, lexicographic enumeration, and compressed pattern matching.

---

## 🧩 Where It’s Used

- 🔢 Count distinct substrings  
- 🧮 Find k-th lexicographic substring  
- 🔍 Substring matching  
- 🧠 Longest common substring (LCS)  
- 🏆 Competitive programming: substring analytics in O(n)

---

## 🧱 Core Idea

- Each state represents a set of substrings ending at some position  
- Transitions represent character extensions  
- `len[v]` — length of longest string in state `v`  
- `link[v]` — suffix link to previous state  
- `transitions[v][c]` — transition from state `v` via character `c`

> The automaton is built incrementally by extending the string one character at a time.

---

## 🔢 Task 1: Count Distinct Substrings

### 🧠 Insight

Every path from the initial state to any other state represents a substring.  
We count all such paths using **DP over the DAG**.

## 🚀 Implementation (C++)

```cpp
struct State {
    int len, link;
    map<char, int> next;
};

vector<State> st;
vector<long long> dp;
int sz, last;

void sa_init() {
    st.clear(); dp.clear();
    st.push_back({0, -1});
    dp.push_back(-1);
    sz = 1; last = 0;
}

void sa_extend(char c) {
    int cur = sz++;
    st.push_back({st[last].len + 1, 0});
    dp.push_back(-1);
    int p = last;
    while (p != -1 && !st[p].next.count(c)) {
        st[p].next[c] = cur;
        p = st[p].link;
    }
    if (p == -1) {
        st[cur].link = 0;
    } else {
        int q = st[p].next[c];
        if (st[p].len + 1 == st[q].len) {
            st[cur].link = q;
        } else {
            int clone = sz++;
            st.push_back({st[p].len + 1, st[q].link, st[q].next});
            dp.push_back(-1);
            while (p != -1 && st[p].next[c] == q) {
                st[p].next[c] = clone;
                p = st[p].link;
            }
            st[q].link = st[cur].link = clone;
        }
    }
    last = cur;
}

long long count(int v) {
    if (dp[v] != -1) return dp[v];
    long long res = 1;
    for (auto [c, u] : st[v].next)
        res += count(u);
    return dp[v] = res;
}

long long count_distinct_substrings(const string& s) {
    sa_init();
    for (char c : s) sa_extend(c);
    return count(0) - 1; // exclude empty string
}
```
## 🔢 Task 1: Count Distinct Substrings

### 🧠 Insight

Every path from the initial state to any other state represents a substring.  
We count all such paths using **DP over the DAG** — memoizing the number of substrings reachable from each state.

---

## 🚀 Implementation (C#)

```csharp
public class SuffixAutomaton {
    private class State {
        public int Len, Link;
        public Dictionary<char, int> Next = new();
    }

    private List<State> st = new();
    private List<long> dp = new();
    private int last;

    public SuffixAutomaton() {
        st.Add(new State { Len = 0, Link = -1 });
        dp.Add(-1);
        last = 0;
    }

    public void Extend(char c) {
        int cur = st.Count;
        st.Add(new State { Len = st[last].Len + 1 });
        dp.Add(-1);

        int p = last;
        while (p != -1 && !st[p].Next.ContainsKey(c)) {
            st[p].Next[c] = cur;
            p = st[p].Link;
        }

        if (p == -1) {
            st[cur].Link = 0;
        } else {
            int q = st[p].Next[c];
            if (st[p].Len + 1 == st[q].Len) {
                st[cur].Link = q;
            } else {
                int clone = st.Count;
                st.Add(new State {
                    Len = st[p].Len + 1,
                    Link = st[q].Link,
                    Next = new Dictionary<char, int>(st[q].Next)
                });
                dp.Add(-1);

                while (p != -1 && st[p].Next[c] == q) {
                    st[p].Next[c] = clone;
                    p = st[p].Link;
                }

                st[q].Link = st[cur].Link = clone;
            }
        }

        last = cur;
    }

    private long Count(int v) {
        if (dp[v] != -1) return dp[v];
        long res = 1;
        foreach (var u in st[v].Next.Values)
            res += Count(u);
        return dp[v] = res;
    }

    public long CountDistinctSubstrings(string s) {
        foreach (char c in s) Extend(c);
        return Count(0) - 1; // exclude empty string
    }

    public long[] PrecomputeCounts() {
        long[] cnt = new long[st.Count];
        for (int i = 0; i < cnt.Length; i++)
            cnt[i] = Count(i);
        return cnt;
    }

    public string KthSubstring(long k) {
        long[] cnt = PrecomputeCounts();
        int v = 0;
        var res = new StringBuilder();

        while (k > 0) {
            foreach (var kv in st[v].Next.OrderBy(p => p.Key)) {
                char c = kv.Key;
                int u = kv.Value;
                if (cnt[u] >= k) {
                    res.Append(c);
                    v = u;
                    k--;
                    break;
                } else {
                    k -= cnt[u];
                }
            }
        }

        return res.ToString();
    }
}
```


## 🔢 Task 2: Find k-th Lexicographic Substring

### 🧠 Insight

- Precompute cnt[v] — number of substrings starting from state v
- Traverse automaton in lexicographic order
- At each state, subtract counts until reaching the k-th

## 🚀 Implementation (C++)
```cpp
string kth_substring(int v, long long k) {
    string res;
    while (k > 0) {
        for (auto [c, u] : st[v].next) {
            if (cnt[u] >= k) {
                res += c;
                v = u;
                k--;
                break;
            } else {
                k -= cnt[u];
            }
        }
    }
    return res;
}
```
## 🚀 Implementation (C#)

```cpp
public string KthSubstring(long k) {
        long[] cnt = PrecomputeCounts();
        int v = 0;
        var res = new StringBuilder();

        while (k > 0) {
            foreach (var kv in st[v].Next.OrderBy(p => p.Key)) {
                char c = kv.Key;
                int u = kv.Value;
                if (cnt[u] >= k) {
                    res.Append(c);
                    v = u;
                    k--;
                    break;
                } else {
                    k -= cnt[u];
                }
            }
        }

        return res.ToString();
    }
```

Before calling kth_substring, run count(v) for all states to fill cnt[].
## ⏱️ Complexity Analysis

| Operation           | Complexity       | Description                                                                 |
|---------------------|------------------|-----------------------------------------------------------------------------|
| Build SAM           | O(n)             | Constructs the automaton in linear time using suffix links and cloning     |
| Count substrings    | O(n)             | DP over DAG — each state visited once, memoized                            |
| k-th substring      | O(k × log σ)     | Traverses lexicographic paths, subtracting counts at each transition       |
| Space               | O(n)             | Number of states is ≤ 2n, transitions are sparse                           |
| Stability           | High             | Deterministic construction, no recursion or branching                      |
| Scalability         | Excellent        | Handles strings up to 10⁶ with ease — ideal for analytics and enumeration  |

### 🧠 Why O(k × log σ) for k-th substring?

- At each step, we scan transitions in lexicographic order  
- For alphabet size σ, this costs O(log σ) per decision (map or sorted array)  
- We make k decisions → total cost is O(k × log σ)

---

## ⚠️ Pitfalls

### ❌ Static Structure
Suffix Automaton is **not updatable** — once built, it cannot accommodate insertions or deletions.  
Use dynamic alternatives (e.g., suffix trees or online automata) for mutable strings.

### 🧠 Clone Handling
During construction, **cloning states** is essential to preserve suffix link invariants.  
Incorrect clone logic leads to invalid transitions and broken DAG structure.

### 🧮 Lexicographic Traversal
To find the k-th substring, transitions must be **sorted by character**.  
Use `map<char, int>` for automatic ordering, or `array<int, σ>` with manual sorting.

### 📦 Alphabet Optimization
For small alphabets (e.g., lowercase English), use `array<int, 26>` for speed.  
For large or dynamic alphabets, prefer `unordered_map` or `map<char, int>`.

> Engineering trade-off: speed vs flexibility vs memory.

---

## ✅ Conclusion

**Suffix Automaton + DP** is a **Substring Enumeration Engine** — designed for high-performance substring analytics.

### ⚡ Capabilities

- Count all **distinct substrings** in O(n)  
- Find the **k-th lexicographic substring**  
- Solve **LCS**, **substring matching**, and **offline analytics**

### 🧠 Design Philosophy

- **Compress substring space** into minimal deterministic states  
- **Traverse DAG** with memoized DP for analytics  
- **Control lexicographic flow** via ordered transitions

### 🛡️ Use Cases

- 📊 Substring counting  
- 🔢 Lexicographic enumeration  
- 🧠 LCS and pattern matching  
- 🏆 Competitive programming: substring analytics, suffix-based problems

> 👉 **Key takeaway:** Suffix Automaton turns substring chaos into compressed order — fast, elegant, and powerful.
> When **brute-force fails** and **trie explodes**, **SAM + DP** is your precision tool.


---
