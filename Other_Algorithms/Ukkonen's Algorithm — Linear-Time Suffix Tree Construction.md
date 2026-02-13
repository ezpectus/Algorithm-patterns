# Ukkonen's Algorithm — Linear-Time Suffix Tree Construction

## Origin & Motivation

Ukkonen's algorithm was introduced by Esko Ukkonen in 1995.

It provides an online method for constructing a suffix tree in linear time—it's a compressed prefix tree (trie) containing all the suffixes of a string.

Previous algorithms (Weiner 1973, McCreight 1976) also ran in O(n), but were offline and much more complex to implement.

Ukkonen's algorithm, however, constructs the tree letter by letter, from left to right, making it a true online algorithm and significantly easier to understand and code.

**Main goal**: to enable solving a huge number of string problems with O(n) preprocessing and very fast queries afterwards.

## Core Idea

### Implicit → Explicit Suffix Tree
- The algorithm first builds **implicit** suffix trees for each prefix `S[1..i]`
- Leaves remain **open** (their end coincides with the current end of the string)
- A unique terminator `$` is added at the end, which forces all suffixes to become explicit leaves

### Phases and Extensions
- A total of **n+1** phases (one for each character + terminator)
- In phase **i+1**, up to **i+1** extensions are performed — inserting all suffixes ending with a new character

### Active Point
- Current insertion point: `(active_node, active_edge, active_length)`
- Allows continuation without returning to the root each time

### Suffix Links
- Analogous to the failure function in Aho-Corasick
- Lead from the internal node representing the string 'αx' to the node representing 'x'
- Allows quick jumps to the desired location for shorter suffixes

### Key Tricks for Achieving O(n)
- **Open ends** — leaves are automatically extended when characters are added
- **Extension Rules (Rule 1/2/3)**:
- Rule 1: suffix already ends in a leaf → do nothing (auto-extend)
- Rule 2: no suitable edge → create a new leaf (or split an existing edge)
- Rule 3: extension already exists → abort the expansion phase for this suffix
- **Skip/count** — fast traversal of long edges
- **Once a leaf, always a leaf** — leaves are never split
- Amortization via suffix links makes the overall time Linear

## Where It's Used

- Substring search in **O(m)** (m is the length of the sample)
- Longest repeating substring
- Longest common substring of two strings
- Longest palindromic substring
- Bioinformatics: pattern searching in DNA/proteins, genome assembly
- Data compression (based on the Burrows–Wheeler transform)
- Full-text index, autocompletion, plagiarism check

## Ukkonen's Algorithm (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

const int ALPHABET_SIZE = 26;  // a-z
const char TERMINATOR = '$';

struct Node {
    int start, end;                    // edge label s[start..end-1]
    int suffixLink = 0;
    int children[ALPHABET_SIZE]{-1};
    Node(int st = 0, int en = 0) : start(st), end(en) {}
};

class UkkonenSuffixTree {
private:
    string s;
    vector<Node> nodes;
    int root = 0;
    int activeNode = 0, activeEdge = -1, activeLen = 0;
    int remainder = 0;
    int currentEnd = 0;

    int newNode(int st, int en) {
        nodes.emplace_back(st, en);
        return nodes.size() - 1;
    }

    int edgeLength(int nodeIdx) {
        return min(nodes[nodeIdx].end, currentEnd) - nodes[nodeIdx].start;
    }

public:
    UkkonenSuffixTree(const string& str) : s(str + TERMINATOR) {
        nodes.reserve(2 * s.size());
        nodes.emplace_back();          // dummy node 0
        root = newNode(0, 0);
        nodes[root].suffixLink = root;
        build();
    }

    void build() {
        for (char ch : s) {
            currentEnd++;
            remainder++;
            int lastNew = 0;

            while (remainder > 0) {
                if (activeLen == 0) activeEdge = currentEnd - remainder;

                int chIdx = s[currentEnd - remainder] - 'a';
                int& child = nodes[activeNode].children[chIdx];

                if (child == -1) {
                    // Rule 2 — new leaf
                    child = newNode(currentEnd - remainder, INT_MAX);
                    if (lastNew) nodes[lastNew].suffixLink = activeNode;
                    lastNew = 0;
                } else {
                    int next = child;
                    int len = edgeLength(next);

                    if (activeLen >= len) {
                        activeEdge += len;
                        activeLen -= len;
                        activeNode = next;
                        continue;
                    }

                    if (s[nodes[next].start + activeLen] == s[currentEnd - 1]) {
                        // Rule 3 — already exists
                        activeLen++;
                        if (lastNew) nodes[lastNew].suffixLink = activeNode;
                        remainder--;
                        continue;
                    }

                    // Rule 2 — split edge
                    int split = newNode(nodes[next].start, nodes[next].start + activeLen);
                    nodes[activeNode].children[chIdx] = split;

                    int leaf = newNode(currentEnd - 1, INT_MAX);
                    nodes[split].children[s[currentEnd - 1] - 'a'] = leaf;

                    nodes[next].start += activeLen;
                    nodes[split].children[s[nodes[next].start] - 'a'] = next;

                    if (lastNew) nodes[lastNew].suffixLink = split;
                    lastNew = split;
                }

                remainder--;

                if (activeNode == root && activeLen > 0) activeLen--;
                else if (activeNode != root) activeNode = nodes[activeNode].suffixLink;
            }
        }

        // Close open ends
        for (auto& node : nodes)
            if (node.end == INT_MAX) node.end = s.size();
    }

    static void runTests() {
        cout << "Ukkonen — Construction Tests\n\n";

        {
            UkkonenSuffixTree st("banana");
            cout << "banana$ → nodes: " << st.nodes.size()
                 << (st.nodes.size() > 10 ? " PASS" : " FAIL") << "\n\n";
        }

        {
            UkkonenSuffixTree st("mississippi");
            cout << "mississippi$ → nodes: " << st.nodes.size()
                 << (st.nodes.size() > 15 ? " PASS" : " FAIL") << "\n\n";
        }

        cout << "Tests completed!\n";
    }
};

int main() {
    UkkonenSuffixTree::runTests();
    return 0;
}
```

## Ukkonen's Algorithm (C#)
```cpp
using System;
using System.Collections.Generic;

public class UkkonenSuffixTree
{
    private const char Terminator = '$';
    private string s;
    private List<Node> nodes = new();
    private int root;
    private int activeNode, activeEdge = -1, activeLen;
    private int remainder;
    private int currentEnd;

    private class Node
    {
        public int start, end;
        public int suffixLink;
        public int[] children = new int[27];
        public Node(int st, int en) { start = st; end = en; Array.Fill(children, -1); }
    }

    public UkkonenSuffixTree(string str)
    {
        s = str + Terminator;
        nodes.Add(null); // dummy
        root = NewNode(0, 0);
        nodes[root].suffixLink = root;
        Build();
    }

    private int NewNode(int st, int en)
    {
        nodes.Add(new Node(st, en));
        return nodes.Count - 1;
    }

    private int EdgeLength(int idx) => Math.Min(nodes[idx].end, currentEnd) - nodes[idx].start;

    private void Build()
    {
        currentEnd = 0;
        foreach (char ch in s)
        {
            currentEnd++;
            remainder++;
            int lastNew = 0;

            while (remainder > 0)
            {
                if (activeLen == 0) activeEdge = currentEnd - remainder;

                int chIdx = s[currentEnd - remainder] - 'a';
                if (chIdx < 0 || chIdx >= 26) chIdx = 26;

                ref int child = ref nodes[activeNode].children[chIdx];

                if (child == -1)
                {
                    child = NewNode(currentEnd - remainder, int.MaxValue);
                    if (lastNew != 0) nodes[lastNew].suffixLink = activeNode;
                    lastNew = 0;
                }
                else
                {
                    int next = child;
                    int len = EdgeLength(next);

                    if (activeLen >= len)
                    {
                        activeEdge += len;
                        activeLen -= len;
                        activeNode = next;
                        continue;
                    }

                    if (s[nodes[next].start + activeLen] == s[currentEnd - 1])
                    {
                        activeLen++;
                        if (lastNew != 0) nodes[lastNew].suffixLink = activeNode;
                        remainder--;
                        continue;
                    }

                    int split = NewNode(nodes[next].start, nodes[next].start + activeLen);
                    nodes[activeNode].children[chIdx] = split;

                    int leaf = NewNode(currentEnd - 1, int.MaxValue);
                    int leafIdx = s[currentEnd - 1] == Terminator ? 26 : s[currentEnd - 1] - 'a';
                    nodes[split].children[leafIdx] = leaf;

                    nodes[next].start += activeLen;
                    int contIdx = s[nodes[next].start] == Terminator ? 26 : s[nodes[next].start] - 'a';
                    nodes[split].children[contIdx] = next;

                    if (lastNew != 0) nodes[lastNew].suffixLink = split;
                    lastNew = split;
                }

                remainder--;

                if (activeNode == root && activeLen > 0) activeLen--;
                else if (activeNode != root) activeNode = nodes[activeNode].suffixLink;
            }
        }

        foreach (var node in nodes)
            if (node != null && node.end == int.MaxValue)
                node.end = s.Length;
    }

    public static void RunTests()
    {
        Console.WriteLine("Ukkonen — Construction Tests\n");

        var st1 = new UkkonenSuffixTree("banana");
        Console.WriteLine($"banana$ → nodes: {st1.nodes.Count-1} {(st1.nodes.Count > 10 ? "PASS" : "FAIL")}\n");

        var st2 = new UkkonenSuffixTree("mississippi");
        Console.WriteLine($"mississippi$ → nodes: {st2.nodes.Count-1} {(st2.nodes.Count > 15 ? "PASS" : "FAIL")}\n");

        Console.WriteLine("Tests completed!");
    }
}

class Program
{
    static void Main() => UkkonenSuffixTree.RunTests();
}
```

## Time Complexity

- **O(n) for fixed-size (small) alphabet**  
  When the alphabet size |Σ| is constant (e.g. 26 for lowercase English letters, DNA alphabet 4, etc.), the algorithm runs in **strictly linear time O(n)**.

- **Amortized O(1) per extension**  
  There are exactly **n+1** phases and up to **n+1** extensions in total across the entire algorithm.  
  Although a single extension might look expensive (walking down edges, creating nodes), the **suffix links** and **active point** mechanism cause most work to be amortized away.  
  Every time we traverse an edge or create a new internal node, we "pay" for it — but suffix links allow us to skip large portions of work in future extensions.  
  This classic amortization argument shows that the **total number of edge traversals and node creations** across all extensions is bounded by **O(n)**.

- **Total work bounded by O(n)**  
  - Number of phases: n+1  
  - Number of extensions performed: at most n+1 (but many are terminated early by Rule 3)  
  - Number of newly created internal nodes: < n  
  - Total edge traversals (skip/count operations): amortized O(1) per extension thanks to decreasing active length and suffix links  
  → overall **O(n)** time for constant alphabet

- **General alphabet (arbitrary |Σ|)**  
  When using hash maps or balanced trees instead of fixed-size arrays for children, each child lookup / insertion becomes **O(log |Σ|)** in the worst case (or expected O(1) with good hashing).  
  Since we perform O(n) such operations in total, the complexity becomes **O(n log |Σ|)**.

**Summary**:  
Fixed/small alphabet → **O(n)** (optimal)  
Arbitrary alphabet → **O(n log |Σ|)** (still very practical for most real-world cases)

## Space Complexity

- **At most 2n − 1 nodes in the final suffix tree**  
  It is a well-known property of suffix trees (for strings with terminator):  
  - Number of leaves = n+1 (one per suffix + terminator)  
  - Number of internal nodes ≤ n  
  - Total nodes ≤ 2n − 1 (tight bound in worst case, e.g., all distinct characters)

- **Edge labels stored as (start, end) pairs**  
  Instead of copying substrings, each edge stores two integers: start index and end index in the original string.  
  → each edge uses **O(1)** space regardless of label length  
  → total space for all edge labels = **O(number of edges) = O(n)**

- **Auxiliary data structures**  
  - Active point: 3 integers → O(1)  
  - Suffix links, remainder, currentEnd, etc.: O(1)  
  - Children arrays/maps: already counted in nodes

**Overall space complexity**: **O(n)**  
(even for very large alphabets when using maps — the constant factor grows, but asymptotic remains linear)

## Key Concepts Recap

- **Implicit suffix tree**  
  Intermediate representation during construction where leaves are "open-ended" — they implicitly extend to the current end of the string being processed. This avoids creating explicit leaf nodes until the very end.

- **Active point (active_node, active_edge, active_length)**  
  The "cursor" that remembers exactly where we are in the tree.  
  It dramatically reduces redundant traversals — we never restart from the root unless necessary.

- **Suffix links**  
  Pointers that connect internal nodes in a way that mirrors suffix relationships.  
  They are the key to amortization: after handling one suffix, we can jump directly to the correct place for the next (shorter) suffix without re-traversing from root.

- **Open ends**  
  Leaves do not store an explicit end position during construction — they are considered to extend to `currentEnd`.  
  Only at the very end do we "close" them by setting end = string length.

- **Extension Rules 1/2/3**  
  - **Rule 1**: the suffix is already represented by an existing leaf → do nothing (implicit extension)  
  - **Rule 2**: we reach a point where no continuation exists → either add a new leaf or split an edge and add a leaf  
  - **Rule 3**: the next character already matches → we can stop this extension early (showstopper rule — huge savings)

- **Once a leaf, always a leaf**  
  Fundamental invariant: once a node becomes a leaf, it will never be split again.  
  This guarantees progress and bounds the number of internal nodes created.

## ✅ Conclusion

Ukkonen's algorithm is widely regarded as one of the most beautiful and technically deep achievements in string algorithm design.

- It transforms what appears at first glance to be a **quadratic** problem (naive suffix trie construction) into a **strictly linear-time** solution for constant alphabets.
- It achieves this through extraordinarily clever use of **amortization**, **structural invariants**, and **link pointers** — concepts that appear again and again in advanced string and graph algorithms.
- It remains **the standard reference** for online suffix tree construction and one of very few ways to build a full suffix tree in linear time in practice.
- It directly enables or heavily influences:
  - suffix arrays and enhanced suffix arrays
  - Burrows–Wheeler Transform (basis of bzip2)
  - modern read aligners and genome assembly tools
  - fast exact and approximate string matching engines
  - longest common substring / repeated substring solvers
  - many data compression and bioinformatics pipelines

Above all, Ukkonen's algorithm stands as a classic demonstration of a powerful principle in algorithm design:

**Deep insight into the mathematical structure of the problem — combined with careful amortization and elegant pointer
tricks — can turn an apparently hard task into one of the most efficient and beautiful algorithms known in computer science.**


---
