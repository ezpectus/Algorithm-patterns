# 🧠 Palindromic Tree (Eertree) — Online Engine for Palindrome Detection
## 📜 Origin & Motivation
Palindromic Tree, also known as Eertree, is a data structure invented around 2014 by Mikhail Rubinchik and Dmitry Sokolov. 
It was designed to solve problems involving online palindrome detection, counting distinct palindromic substrings, and efficient updates as characters are added one by one.

Core idea: maintain a tree of palindromic substrings where each node represents a unique palindrome, and edges represent extensions by characters.
Unlike brute-force or DP approaches, Eertree enables real-time palindrome tracking with O(n) total complexity.

## 🧩 Where It’s Used

- 🔍 Online detection of palindromic substrings
- 🧮 Counting distinct palindromes in a string
- 📈 Real-time updates as characters are added
- 🧠 Efficient preprocessing for palindrome-related queries
- 🏆 Competitive programming: Codeforces, AtCoder, HackerRank

## 🔁 When to Use Eertree vs Alternatives

| Task / Scenario                        | Eertree | Manacher’s | DP  | Hashing |
|---------------------------------------|---------|------------|-----|---------|
| Online palindrome detection           | ✅      | ❌         | ❌  | ❌      |
| Count distinct palindromic substrings | ✅      | ❌         | ✅  | ✅      |
| Real-time updates                     | ✅      | ❌         | ❌  | ❌      |
| Longest palindromic substring         | ❌      | ✅         | ✅  | ✅      |
| Offline analysis                      | ✅      | ✅         | ✅  | ✅      |

---

## 🧱 Core Idea

- Each node in the tree represents a **unique palindrome**
- Two root nodes:
  - Length -1: imaginary node for even-length palindromes
  - Length 0: empty string
- For each new character:
  - Traverse suffix links to find the longest suffix-palindrome that can be extended
  - Create a new node if the new palindrome is unique
  - Maintain suffix links and transition edges
- Guarantees **O(n)** total time for building the tree

---

## 🚀 Implementation (C++)

```cpp
struct Node {
    int len, suffLink;
    map<char, int> next;
    int count;

    Node(int l) : len(l), suffLink(0), count(0) {}
};

class Eertree {
    vector<Node> tree;
    string s;
    int suff;

public:
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
        while (true) {
            int pos = s.size() - 2 - tree[curr].len;
            if (pos >= 0 && s[pos] == c) break;
            curr = tree[curr].suffLink;
        }

        if (tree[curr].next.count(c)) {
            suff = tree[curr].next[c];
            tree[suff].count++;
            return;
        }

        int newLen = tree[curr].len + 2;
        tree.emplace_back(newLen);
        int newNode = tree.size() - 1;
        tree[curr].next[c] = newNode;

        if (newLen == 1) {
            tree[newNode].suffLink = 1;
        } else {
            int link = tree[curr].suffLink;
            while (true) {
                int pos = s.size() - 2 - tree[link].len;
                if (pos >= 0 && s[pos] == c) break;
                link = tree[link].suffLink;
            }
            tree[newNode].suffLink = tree[link].next[c];
        }

        tree[newNode].count = 1;
        suff = newNode;
    }

    int countDistinct() {
        return tree.size() - 2;
    }
};
```


## 🚀 Implementation (C#)
```csharp
public class Eertree {
    private class Node {
        public int Len, SuffLink, Count;
        public Dictionary<char, int> Next = new();
        public Node(int len) { Len = len; Count = 0; }
    }

    private List<Node> tree = new();
    private StringBuilder s = new();
    private int suff;

    public Eertree() {
        tree.Add(new Node(-1)); // imaginary root
        tree.Add(new Node(0));  // empty string root
        tree[0].SuffLink = 0;
        tree[1].SuffLink = 0;
        suff = 1;
    }

    public void AddChar(char c) {
        s.Append(c);
        int curr = suff;
        while (true) {
            int pos = s.Length - 2 - tree[curr].Len;
            if (pos >= 0 && s[pos] == c) break;
            curr = tree[curr].SuffLink;
        }

        if (tree[curr].Next.ContainsKey(c)) {
            suff = tree[curr].Next[c];
            tree[suff].Count++;
            return;
        }

        int newLen = tree[curr].Len + 2;
        var newNode = new Node(newLen);
        tree.Add(newNode);
        int newNodeIndex = tree.Count - 1;
        tree[curr].Next[c] = newNodeIndex;

        if (newLen == 1) {
            newNode.SuffLink = 1;
        } else {
            int link = tree[curr].SuffLink;
            while (true) {
                int pos = s.Length - 2 - tree[link].Len;
                if (pos >= 0 && s[pos] == c) break;
                link = tree[link].SuffLink;
            }
            newNode.SuffLink = tree[link].Next[c];
        }

        newNode.Count = 1;
        suff = newNodeIndex;
    }

    public int CountDistinct() => tree.Count - 2;
}
```

## ⏱️ Complexity Analysis

- **Insertion per character:**  
  Each character is processed in amortized **O(1)** time due to suffix link traversal and bounded node creation.  
  Even in worst-case scenarios, the number of traversals is limited by the depth of the tree.

- **Total build time:**  
  For a string of length **n**, the entire Eertree is built in **O(n)** time.  
  This includes all node creations, suffix link updates, and edge insertions.

- **Space complexity:**  
  - **O(n)** nodes — each unique palindrome creates one node  
  - Each node stores:
    - `len`: length of the palindrome  
    - `suffLink`: pointer to longest proper suffix-palindrome  
    - `next`: map of transitions by character  
    - `count`: frequency of occurrence  
  - Total space: **O(n)** for nodes + **O(n)** for transitions

- **Query time:**  
  - **O(1)** to add a character  
  - **O(n)** to count all distinct palindromes  
  - **O(n)** to propagate counts (if needed for frequency analysis)

---

## ⚠️ Pitfalls

- 🧩 **Suffix link traversal**  
  Must correctly find the longest suffix-palindrome that can be extended.  
  Mistakes here lead to incorrect tree structure and duplicate nodes.

- ⚠️ **Edge cases**  
  - First character must link to root correctly  
  - Palindromes of length 1 and 2 require special handling  
  - Empty string and imaginary root must be initialized properly

- 🔁 **Duplicate palindromes**  
  Each node must represent a **unique** palindrome.  
  Use suffix links and transition maps to avoid redundant creation.

- 🧠 **Count propagation**  
  If you want to count total occurrences of each palindrome (not just distinct ones),  
  you must propagate `count` values **in reverse topological order** — from longer to shorter palindromes via suffix links.

---

## ✅ Conclusion

**Palindromic Tree (Eertree)** is a **real-time engine for palindrome detection**, combining elegance and efficiency:

- 🔍 Tracks all **unique palindromic substrings** in a string  
- 🧠 Supports **online updates** with amortized **O(1)** time per character  
- 📊 Enables **fast counting**, **frequency analysis**, and **structural queries**  
- 🏆 A must-have tool for **dynamic string problems** in competitive programming

👉 **Key takeaway:**  
Eertree transforms brute-force palindrome detection into a **scalable, online-capable architecture** — ideal for real-time analysis, substring tracking, and structural string queries.


---
