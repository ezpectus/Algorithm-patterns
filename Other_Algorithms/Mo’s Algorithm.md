# 🧠 Mo’s Algorithm — Offline Query Block Engine

## 📜 Origin & Motivation
When we have **many range queries** (e.g., frequency, sum, xor) on a static array,  
naive O(n) per query is too slow (O(nq)).  
Segment trees or Fenwick trees help for associative operations, but not for more complex queries (like frequency of a value, number of distinct elements, etc.).

**Mo’s Algorithm** is an offline technique:  
- Sort queries in a special order (by blocks of size √n).  
- Move a sliding window [L, R] across the array, adding/removing elements.  
- Each query is answered in O(1) amortized per add/remove.  

Total complexity: **O((n + q)√n)**.

---

## 🧩 Where It’s Used
- 📊 Frequency queries (count occurrences of x in [L, R])  
- 🔢 Number of distinct elements in a range  
- ⚡ Range XOR / sum / gcd when updates are simple  
- 🎮 Competitive programming: offline optimization without segment trees  

---

## 🔁 When to Use Mo’s Algorithm vs Alternatives

| Task / Scenario                  | Use Mo’s Algorithm | Use Segment Tree | Use Fenwick Tree |
|----------------------------------|--------------------|------------------|------------------|
| Range sum/min/max                | ❌                 | ✅               | ✅               |
| Frequency / distinct count       | ✅                 | ❌               | ❌               |
| Offline queries (all known)      | ✅                 | ❌               | ❌               |
| Online queries (arrives in real time) | ❌            | ✅               | ✅               |
| Complex add/remove operations    | ✅                 | ❌               | ❌               |

---

## 🧱 Core Idea
1. Choose block size ≈ √n.  
2. Sort queries by:  
   - Primary key = L / block size  
   - Secondary key = R (increasing or decreasing depending on block parity to reduce moves).  
3. Maintain current [L, R] and a data structure supporting:  
   - `add(x)` when extending range  
   - `remove(x)` when shrinking range  
4. Process queries in sorted order, adjusting [L, R] with add/remove.  

---

## 🚀 Implementation Sketch (C++)

```cpp
struct Query {
    int l, r, idx;
};

int BLOCK;

bool cmp(const Query& a, const Query& b) {
    int blockA = a.l / BLOCK, blockB = b.l / BLOCK;
    if (blockA != blockB) return blockA < blockB;
    return (blockA & 1) ? (a.r > b.r) : (a.r < b.r);
}

vector<long long> moAlgorithm(const vector<int>& arr, vector<Query>& queries) {
    int n = arr.size();
    BLOCK = max(1, (int)sqrt(n));
    sort(queries.begin(), queries.end(), cmp);

    vector<long long> ans(queries.size());
    int curL = 0, curR = -1;
    long long curAns = 0;

    auto add = [&](int pos) {
        curAns += arr[pos]; // example: sum
    };
    auto remove = [&](int pos) {
        curAns -= arr[pos];
    };

    for (auto& q : queries) {
        while (curL > q.l) add(--curL);
        while (curR < q.r) add(++curR);
        while (curL < q.l) remove(curL++);
        while (curR > q.r) remove(curR--);
        ans[q.idx] = curAns;
    }
    return ans;
}
```

## 🚀 Implementation Sketch (C#)
```cpp
public class Query {
    public int L, R, Id;
}

public class MoAlgorithm {
    private int BLOCK;

    public long[] Process(int[] arr, List<Query> queries) {
        int n = arr.Length;
        BLOCK = (int)Math.Sqrt(n);
        queries.Sort((a, b) => {
            int blockA = a.L / BLOCK, blockB = b.L / BLOCK;
            if (blockA != blockB) return blockA.CompareTo(blockB);
            return (blockA % 2 == 0) ? a.R.CompareTo(b.R) : b.R.CompareTo(a.R);
        });

        long[] ans = new long[queries.Count];
        int curL = 0, curR = -1;
        long curAns = 0;

        Action<int> add = (pos) => { curAns += arr[pos]; };   // example: sum
        Action<int> remove = (pos) => { curAns -= arr[pos]; };

        foreach (var q in queries) {
            while (curL > q.L) add(--curL);
            while (curR < q.R) add(++curR);
            while (curL < q.L) remove(curL++);
            while (curR > q.R) remove(curR--);
            ans[q.Id] = curAns;
        }
        return ans;
    }
}
```

## ⏱️ Complexity Analysis

- **Preprocessing:** O(q log q)  
  - Sorting queries by block and R coordinate.  
  - Sorting dominates when q is large.  

- **Query processing:** O((n + q)√n)  
  - Each element is added/removed O(√n) times on average.  
  - Total moves of pointers L and R across all queries = O((n + q)√n).  
  - Each add/remove must be O(1) (or close) for this bound to hold.  

- **Space:** O(n + q)  
  - Store the array, queries, and auxiliary frequency counters.  
  - Frequency array size depends on value range (e.g., O(maxValue) if direct indexing, or use hash map).  

---

## ⚠️ Pitfalls

- **Offline only:**  
  - All queries must be known in advance.  
  - Cannot handle online queries arriving in real time.  

- **Block size choice:**  
  - Typically `BLOCK = √n`.  
  - Tuning matters:  
    - Too small → too many block switches.  
    - Too large → too many moves inside blocks.  
  - Some variants use `BLOCK = max(1, n / √q)` for better balance.  

- **Add/remove complexity:**  
  - Must be O(1) or close.  
  - Example:  
    - Frequency count updates → O(1).  
    - Distinct count → maintain counter of non‑zero frequencies.  
    - XOR/sum → direct add/remove.  
  - If add/remove is O(log n), complexity worsens.  

- **Edge cases:**  
  - Empty ranges (L > R).  
  - 0‑based vs 1‑based indexing.  
  - Queries with L = R (single element).  

- **Memory for frequency array:**  
  - If values are large (e.g., up to 1e9), cannot use direct array.  
  - Use hash map or coordinate compression.  

- **Answer ordering:**  
  - Queries are processed in sorted order, but answers must be restored to original order using `idx`.  

---

## ✅ Conclusion

Mo’s Algorithm is an **Offline Query Block Engine**:

- ⚡ Handles many range queries in **O((n + q)√n)**.  
- 📊 Works for frequency, distinct count, XOR, sum, etc.  
- 🔗 Avoids heavy segment tree or Fenwick tree implementations.  
- 🛡️ Deterministic and simple once `add/remove` are defined.  

👉 **Key takeaway:**  
Mo’s Algorithm is the go‑to tool when you need to process many offline range queries efficiently **without building complex trees**.  
It shines when:  
- Queries are offline.  
- Add/remove operations are O(1).  
- The problem requires frequency/distinct‑based answers that segment trees cannot handle directly.


---
