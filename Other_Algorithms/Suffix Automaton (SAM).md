# 🧠 Suffix Automaton (SAM) — Compressed Suffix Engine

## 📜 Origin & Motivation
A **Suffix Automaton** is a minimal deterministic finite automaton (DFA) that recognizes all substrings of a given string.  
It was introduced in the 1980s as a compact way to represent all substrings while supporting efficient queries.

Instead of building a full suffix tree (heavy, O(n) nodes but large constants), SAM compresses the structure into at most **2n−1 states** and **3n−4 transitions** for a string of length `n`.  
This makes it one of the most elegant and powerful string data structures.

---

## 🧩 What It Can Do
- 📊 **Count unique substrings** in O(n)  
- 🔗 **Find LCS (Longest Common Substring)** between two strings in O(n)  
- 🔍 **Substring existence queries** in O(1) per character (walk the automaton)  
- 📏 **Longest repeated substring**  
- 🔄 **Lexicographic k‑th substring**  
- ⚡ General purpose: a compressed suffix engine for deep optimizations

---

## 🔁 When to Use SAM vs Other Structures

| Task / Scenario              | Use SAM | Use Suffix Tree | Use Rolling Hash |
|------------------------------|---------|-----------------|------------------|
| Substring existence queries  | ✅      | ✅              | ✅               |
| Count distinct substrings    | ✅      | ✅              | ❌               |
| LCS between two strings      | ✅      | ✅              | ❌               |
| Fast substring comparison    | ❌      | ❌              | ✅               |
| Memory efficiency            | ✅      | ❌              | ✅               |

---

## 🧱 Core Construction Idea
- Each **state** represents a set of end positions of substrings (endpos set).  
- Each **transition** corresponds to extending substrings by one character.  
- Each state stores:
  - `len` → length of the longest substring in this state  
  - `link` → suffix link (like failure link in Aho–Corasick)  
  - `next` → transitions by characters  

**Construction:**  
- Process the string left to right.  
- Extend automaton with each new character in O(1) amortized.  
- Maintain suffix links to ensure minimality.

---

## 🚀 Implementation (C++)

```cpp
struct SAM {
    struct State {
        int len, link;
        map<char,int> next;
    };

    vector<State> st;
    int last;

    SAM(int n) {
        st.resize(2*n);
        st[0].len = 0;
        st[0].link = -1;
        last = 0;
    }

    int sz = 1;

    void extend(char c) {
        int cur = sz++;
        st[cur].len = st[last].len + 1;
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
                st[clone] = st[q];
                st[clone].len = st[p].len + 1;
                while (p != -1 && st[p].next[c] == q) {
                    st[p].next[c] = clone;
                    p = st[p].link;
                }
                st[q].link = st[cur].link = clone;
            }
        }
        last = cur;
    }
};
```
- Build: build(s) constructs SAM in O(n) amortized.
- contains: checks if t is a substring of s.
- countDistinctSubstrings: sums len[v] - len[link[v]] over states.
- longestCommonSubstring: walks over second string to find LCS with s.


---

## 🚀Implementation C#
```cpp
using System;
using System.Collections.Generic;

public class SuffixAutomaton
{
    private class State
    {
        public int Len;                 // length of the longest string in this state
        public int Link;                // suffix link
        public Dictionary<char,int> Next = new Dictionary<char,int>();
    }

    private readonly List<State> st = new List<State>();
    private int last;                  // index of the state representing the whole current string

    public SuffixAutomaton(int maxLenHint = 0)
    {
        st.Add(new State { Len = 0, Link = -1 });
        last = 0;
        // Optional: pre-allocate to reduce reallocations
        if (maxLenHint > 0) st.Capacity = Math.Max(2 * maxLenHint, 4);
    }

    public void Extend(char c)
    {
        int cur = NewState();
        st[cur].Len = st[last].Len + 1;

        int p = last;
        while (p != -1 && !st[p].Next.ContainsKey(c))
        {
            st[p].Next[c] = cur;
            p = st[p].Link;
        }

        if (p == -1)
        {
            st[cur].Link = 0;
        }
        else
        {
            int q = st[p].Next[c];
            if (st[p].Len + 1 == st[q].Len)
            {
                st[cur].Link = q;
            }
            else
            {
                int clone = NewState();
                st[clone].Len  = st[p].Len + 1;
                st[clone].Link = st[q].Link;
                st[clone].Next = new Dictionary<char,int>(st[q].Next);

                while (p != -1 && st[p].Next.TryGetValue(c, out int to) && to == q)
                {
                    st[p].Next[c] = clone;
                    p = st[p].Link;
                }

                st[q].Link    = clone;
                st[cur].Link  = clone;
            }
        }

        last = cur;
    }

    public void Build(string s)
    {
        foreach (char c in s) Extend(c);
    }

    // O(m) check: does t appear as a substring in the built string?
    public bool Contains(string t)
    {
        int v = 0;
        foreach (char c in t)
        {
            if (!st[v].Next.TryGetValue(c, out v)) return false;
        }
        return true;
    }

    // Count distinct substrings of the built string in O(states)
    public long CountDistinctSubstrings()
    {
        long sum = 0;
        for (int i = 1; i < st.Count; i++) // skip state 0 (root)
        {
            int link = st[i].Link == -1 ? 0 : st[i].Link;
            sum += st[i].Len - st[link].Len;
        }
        return sum;
    }

    // Longest Common Substring with t in O(|t|)
    public string LongestCommonSubstring(string t)
    {
        int v = 0, l = 0;
        int bestLen = 0, bestEnd = -1;

        for (int i = 0; i < t.Length; i++)
        {
            char c = t[i];

            while (v != -1 && !st[v].Next.ContainsKey(c))
            {
                v = st[v].Link;
                if (v == -1) { v = 0; l = 0; break; }
                l = Math.Min(l, st[v].Len);
            }

            if (st[v].Next.TryGetValue(c, out int to))
            {
                v = to;
                l++;
            }
            else
            {
                // stay at root
                v = 0;
                l = 0;
            }

            if (l > bestLen)
            {
                bestLen = l;
                bestEnd = i;
            }
        }

        return bestLen == 0 ? "" : t.Substring(bestEnd - bestLen + 1, bestLen);
    }

    private int NewState()
    {
        st.Add(new State());
        return st.Count - 1;
    }
}

```
- Build: call Build(s) to construct SAM in O(n).
- Contains: Contains(t) checks substring existence in O(|t|).
- CountDistinctSubstrings: uses the classic sum of len[v] - len[link[v]].
- LongestCommonSubstring: walks over t using SAM of s, tracking the best match length.


---

## ⏱️ Complexity Analysis

- **Construction:** O(n) amortized  
  - Each character is processed once.  
  - Each transition is created at most once.  
  - Cloning happens only when necessary, and each state is cloned at most once.  

- **Space:** O(n)  
  - Number of states ≤ 2n − 1.  
  - Number of transitions ≤ 3n − 4.  
  - Each state stores:  
    - `len` (int)  
    - `link` (int)  
    - outgoing transitions (map/array)  

- **Queries:**  
  - **Substring existence:** O(m) for string of length m (walk transitions).  
  - **Count distinct substrings:** O(n) after construction using formula  
    ```
    sum(len[v] - len[link[v]]) over all states v
    ```  
  - **Longest Common Substring (LCS):** O(n) with traversal of the second string.  
  - **Other queries:**  
    - Longest repeated substring → O(n)  
    - Lexicographic k‑th substring → O(n log n) with preprocessing  

---

## ⚠️ Pitfalls

- **Memory:**  
  - Although linear, constants are higher than rolling hash.  
  - For large alphabets, `map<char,int>` is heavy; prefer arrays for fixed alphabets.  

- **Implementation complexity:**  
  - Cloning states is subtle.  
  - Off‑by‑one errors in suffix links are common.  
  - Debugging requires careful tracking of `len` and `link`.  

- **Alphabet size:**  
  - `map<char,int>` → flexible but slower.  
  - `array[26]` → much faster for lowercase letters.  
  - Trade‑off: generality vs performance.  

- **Practical note:**  
  - Pre‑reserve `2n` states to avoid reallocations.  
  - For competitive programming, use arrays for speed.  
  - For general text (Unicode), maps/dictionaries are safer.  

---

## ✅ Conclusion

Suffix Automaton is a **compressed suffix engine**:

- ⚡ Stores all substrings in O(n) space  
- 📊 Enables counting and distinct substring queries  
- 🔗 Solves LCS and repeated substring problems efficiently  
- 🛡️ Provides deterministic guarantees  

👉 **Key takeaway:**  
SAM is the **“architectural monster”** of string algorithms — compact, powerful, and the go‑to structure when you need deep substring analysis.  
It combines the power of suffix trees with the elegance of automata, making it one of the most versatile tools in advanced string processing.



---
