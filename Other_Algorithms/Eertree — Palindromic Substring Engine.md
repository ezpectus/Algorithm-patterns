# Eertree — Palindromic Substring Engine

---

## Origin & Motivation

**Eertree** (Palindromic Tree) was introduced in **2014** by **Mikhail Rubinchik and Arseny Malkov**.  
It’s a **specialized structure** for **online construction** of all **distinct palindromic substrings** in a string — in **linear time**.

Unlike **suffix trees** or **tries**, Eertree is built around **palindromic symmetry**, maintaining links between palindromes and their extensions.

---

## Where It’s Used

| Domain | Use Case |
|--------|----------|
| Bioinformatics | Palindromic DNA patterns |
| String Analytics | Counting, indexing palindromes |
| Competitive Programming | Rare but powerful problems |
| Text Mining | Symmetry-based features |
| Algorithmic Research | Structure design, suffix symmetry |

---

## When to Use Eertree vs Alternatives

| Task | Eertree | Suffix Tree | Trie | Hashing |
|------|---------|-------------|------|---------|
| Count distinct palindromic substrings | Yes | No | No | No |
| Online palindrome tracking | Yes | No | No | No |
| Substring search | No | Yes | Yes | Yes |
| Longest palindromic suffix | Yes | No | No | Yes (with tricks) |
| All suffixes | No | Yes | No | No |

---

## Core Idea

* Each node represents a **unique palindrome**  
* Edges represent **character extensions** that preserve palindromicity  
* Two special roots:  
  * Length **−1**: imaginary node for even-length palindromes  
  * Length **0**: empty string  
* For each new character:  
  * Traverse **suffix links** to find longest palindromic suffix  
  * **Extend** if possible, **create new node** if needed  
  * Maintain **suffix links** for efficient traversal  

---

## Implementation (C++)

```cpp
struct Node {
    int len, suffLink;
    map<char, int> next;
    Node(int l) : len(l), suffLink(0) {}
};

class Eertree {
public:
    vector<Node> tree;
    string s;
    int suff;

    Eertree() {
        tree.emplace_back(-1); // imaginary root
        tree.emplace_back(0);  // empty string root
        tree[0].suffLink = 0;
        tree[1].suffLink = 0;
        suff = 1;
    }

    void addChar(char c) {
        s += c;
        int curr = suff;
        int pos = s.size() - 1;

        while (true) {
            int curlen = tree[curr].len;
            if (pos - curlen - 1 >= 0 && s[pos - curlen - 1] == c)
                break;
            curr = tree[curr].suffLink;
        }

        if (tree[curr].next.count(c)) {
            suff = tree[curr].next[c];
            return;
        }

        int newNode = tree.size();
        tree.emplace_back(tree[curr].len + 2);
        tree[curr].next[c] = newNode;

        if (tree[newNode].len == 1) {
            tree[newNode].suffLink = 1;
        } else {
            int link = tree[curr].suffLink;
            while (true) {
                int curlen = tree[link].len;
                if (pos - curlen - 1 >= 0 && s[pos - curlen - 1] == c)
                    break;
                link = tree[link].suffLink;
            }
            tree[newNode].suffLink = tree[link].next[c];
        }

        suff = newNode;
    }
};
```

## Implementation (C#)

```csharp
public class Node {
    public int Len, SuffLink;
    public Dictionary<char, int> Next = new();
    public Node(int len) { Len = len; }
}

public class Eertree {
    private List<Node> tree = new();
    private string s = "";
    private int suff;

    public Eertree() {
        tree.Add(new Node(-1)); // imaginary root
        tree.Add(new Node(0));  // empty string root
        tree[0].SuffLink = 0;
        tree[1].SuffLink = 0;
        suff = 1;
    }

    public void AddChar(char c) {
        s += c;
        int curr = suff;
        int pos = s.Length - 1;

        while (true) {
            int curlen = tree[curr].Len;
            if (pos - curlen - 1 >= 0 && s[pos - curlen - 1] == c)
                break;
            curr = tree[curr].SuffLink;
        }

        if (tree[curr].Next.ContainsKey(c)) {
            suff = tree[curr].Next[c];
            return;
        }

        int newNode = tree.Count;
        tree.Add(new Node(tree[curr].Len + 2));
        tree[curr].Next[c] = newNode;

        if (tree[newNode].Len == 1) {
            tree[newNode].SuffLink = 1;
        } else {
            int link = tree[curr].SuffLink;
            while (true) {
                int curlen = tree[link].Len;
                if (pos - curlen - 1 >= 0 && s[pos - curlen - 1] == c)
                    break;
                link = tree[link].SuffLink;
            }
            tree[newNode].SuffLink = tree[link].Next[c];
        }

        suff = newNode;
    }
}
```
## Complexity Analysis

| Phase             | Complexity                     |
|-------------------|--------------------------------|
| **Add character** | O(log n) (due to map)          |
| **Total build**   | O(n log n) worst-case          |
| **Space**         | O(n) nodes for palindromes     |

> With **hash maps** or **array-based transitions**, time can be reduced to **amortized O(n)**.

---

## Why It’s Rare and Powerful

* Tracks **all distinct palindromic substrings** in **online fashion**  
* Supports **dynamic updates** — character-by-character  
* **Elegant suffix-link structure** mirrors suffix automaton logic  
* **Rarely needed**, but when it fits — **nothing else compares**

---

## Impact of Design Choices

| Choice             | Effect |
|--------------------|--------|
| **Map vs array**   | Tradeoff between generality and speed |
| **Suffix links**   | Enable fast traversal and extension |
| **Two roots**      | Handle even/odd palindromes uniformly |
| **Online updates** | Ideal for streaming or incremental input |

---

## Pitfalls

* Not suitable for **general substring search**  
* Requires **careful suffix-link logic**  
* **Map-based transitions** may slow down performance  
* Hard to **debug** due to implicit symmetry logic

---

## Conclusion

Eertree is a **palindromic engine**:

* Elegant structure for **rare but powerful tasks**  
* Tracks **all palindromic substrings** in **linear time**  
* Ideal for **symmetry-based string analysis**  
* **Suffix-link architecture** mirrors automaton logic  
* **Competitive edge** in niche problems

> **Key takeaway**:  
> **Eertree is the cherry on top when you need online palindromic tracking.**  
> It’s rare, but when the task fits — it’s **unbeatable**.



---



