# 🧠 Aho-Corasick Automaton — Multi-pattern Matching Engine

---

## 📜 Origin & Motivation
The **Aho–Corasick algorithm** was introduced in 1975 by Alfred Aho and Margaret Corasick.  
It was designed to solve the problem of **matching multiple patterns simultaneously** in a text stream.

Instead of running KMP or Rabin–Karp separately for each pattern, Aho–Corasick builds a **Trie** of all patterns and augments it with **failure links** (similar to KMP’s prefix table).  
This transforms the Trie into a finite automaton that can scan the text in **linear time** relative to its length.

---

## 🧩 Where It’s Used
- **Dictionary-based filtering** — censorship, keyword blocking  
- **Spam detection** — scanning messages for blacklisted terms  
- **Intrusion detection systems** — matching against malicious signatures  
- **Bioinformatics** — searching DNA/RNA sequences for motifs  
- **Search engines** — autocomplete and multi-keyword matching  

---

## 🔁 When to Use Aho-Corasick vs Other Algorithms

| Task                  | Use Aho-Corasick | Use KMP | Use Rabin-Karp |
|-----------------------|------------------|---------|----------------|
| Single pattern        | ❌               | ✅      | ✅             |
| Multiple patterns     | ✅               | ❌      | ✅             |
| Real-time streaming   | ✅               | ✅      | ✅             |
| Worst-case guarantee  | ✅               | ✅      | ❌             |
| Large dictionary search | ✅             | ❌      | ❌             |

---

## 🧱 Trie + Failure Links Construction (C#)

```csharp
class AhoCorasick {
    class Node {
        public Dictionary<char, Node> Next = new();
        public Node? Fail;
        public List<string> Output = new();
    }

    private readonly Node root = new();

    public void AddPattern(string pattern) {
        var node = root;
        foreach (char c in pattern) {
            if (!node.Next.ContainsKey(c))
                node.Next[c] = new Node();
            node = node.Next[c];
        }
        node.Output.Add(pattern);
    }

    public void Build() {
        Queue<Node> q = new();
        foreach (var kv in root.Next) {
            kv.Value.Fail = root;
            q.Enqueue(kv.Value);
        }

        while (q.Count > 0) {
            var current = q.Dequeue();
            foreach (var kv in current.Next) {
                char c = kv.Key;
                var child = kv.Value;

                Node? f = current.Fail;
                while (f != null && !f.Next.ContainsKey(c))
                    f = f.Fail;

                child.Fail = f == null ? root : f.Next[c];
                child.Output.AddRange(child.Fail.Output);
                q.Enqueue(child);
            }
        }
    }

    public List<(int, string)> Search(string text) {
        var result = new List<(int, string)>();
        var node = root;

        for (int i = 0; i < text.Length; i++) {
            char c = text[i];
            while (node != root && !node.Next.ContainsKey(c))
                node = node.Fail!;

            if (node.Next.ContainsKey(c))
                node = node.Next[c];

            foreach (var pat in node.Output)
                result.Add((i - pat.Length + 1, pat));
        }
        return result;
    }
}
```


## 🧱 Trie + Failure Links Construction (C++)

```cpp
struct Node {
    map<char, Node*> next;
    Node* fail = nullptr;
    vector<string> output;
};

struct AhoCorasick {
    Node* root = new Node();

    void addPattern(const string& pat) {
        Node* node = root;
        for (char c : pat) {
            if (!node->next.count(c))
                node->next[c] = new Node();
            node = node->next[c];
        }
        node->output.push_back(pat);
    }

    void build() {
        queue<Node*> q;
        for (auto& kv : root->next) {
            kv.second->fail = root;
            q.push(kv.second);
        }

        while (!q.empty()) {
            Node* cur = q.front(); q.pop();
            for (auto& kv : cur->next) {
                char c = kv.first;
                Node* child = kv.second;

                Node* f = cur->fail;
                while (f && !f->next.count(c))
                    f = f->fail;

                child->fail = f ? f->next[c] : root;
                child->output.insert(child->output.end(),
                                     child->fail->output.begin(),
                                     child->fail->output.end());
                q.push(child);
            }
        }
    }

    vector<pair<int,string>> search(const string& text) {
        vector<pair<int,string>> res;
        Node* node = root;
        for (int i = 0; i < text.size(); i++) {
            char c = text[i];
            while (node != root && !node->next.count(c))
                node = node->fail;
            if (node->next.count(c))
                node = node->next[c];
            for (auto& pat : node->output)
                res.push_back({i - (int)pat.size() + 1, pat});
        }
        return res;
    }
};
```

## ⏱️ Complexity Analysis

### 🔧 Preprocessing
- **Building Trie:** `O(Σ|P_i|)`  
  where `Σ|P_i|` is the total length of all patterns.  
- **Building failure links:** `O(Σ|P_i|)`  
  each node is processed once when constructing failure transitions.

### 🔍 Search Phase
- **Time:** `O(n + z)`  
  - `n` = length of the text  
  - `z` = number of matches reported  
- Every character of the text is processed at most once, and each match is reported in constant time.

### 💾 Space
- **Memory usage:** proportional to the number of nodes in the Trie.  
- Each node stores:  
  - outgoing edges (children)  
  - one failure link  
  - output list of matched patterns  

---

## ✅ Conclusion

Aho–Corasick is not just a matcher — it’s a **multi-pattern automaton**.  
It transforms a set of patterns into a deterministic machine that:

- ⚡ Scans text in **linear time**  
- 📚 Handles **multiple patterns simultaneously**  
- 🛡️ Guarantees **worst-case performance**  
- 🌐 Powers real-world systems from **spam filters** to **genome analysis**  

This algorithm is a **dictionary-driven engine**:  
instead of brute-forcing each pattern, it compresses the entire set into a **Trie + failure links**,  
turning chaos into **deterministic flow control**.  

---


---
