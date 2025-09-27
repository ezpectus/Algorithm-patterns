# 🧠 Fenwick Tree (Binary Indexed Tree) — Prefix Sum Engine

## 📜 Origin & Motivation
The **Fenwick Tree**, introduced by Peter Fenwick in 1994, is a compact data structure for maintaining prefix sums efficiently.  
It supports:
- **Prefix queries** in O(log n)  
- **Point updates** in O(log n)  

Unlike segment trees, it is simpler to implement and uses only O(n) memory.  
It is the go‑to structure for cumulative frequency tables, inversion counting, and competitive programming tasks.

---

## 🧩 Where It’s Used
- 📊 Prefix sums with updates  
- 🔢 Frequency tables (e.g., counting inversions)  
- ⚡ Order statistics (k‑th element queries with binary lifting)  
- 🎮 Range updates + point queries (with trick of using two BITs)  
- 🏆 Competitive programming: fast, memory‑efficient alternative to segment trees  

---

## 🔁 When to Use Fenwick Tree vs Alternatives

| Task / Scenario                  | Use Fenwick Tree | Use Segment Tree | Use Prefix Array |
|----------------------------------|------------------|------------------|------------------|
| Static prefix sums               | ❌               | ❌               | ✅               |
| Dynamic point updates + prefix   | ✅               | ✅               | ❌               |
| Range queries (sum/min/max)      | ✅ (sum only)    | ✅               | ❌               |
| Complex operations (min/gcd/xor) | ❌               | ✅               | ❌               |
| Memory‑sensitive                 | ✅               | ❌               | ✅               |

---

## 🧱 Core Idea
- Store partial sums in a clever way using **least significant bit (LSB)**.  
- Each index `i` is responsible for a range of size `LSB(i)`.  
- Update: climb upward by adding `LSB(i)`.  
- Query: climb downward by subtracting `LSB(i)`.  

---

## 🚀 Implementation (C++)

```cpp
struct Fenwick {
    int n;
    vector<long long> bit;

    Fenwick(int n) : n(n), bit(n+1, 0) {}

    void update(int idx, long long delta) {
        for (++idx; idx <= n; idx += idx & -idx)
            bit[idx] += delta;
    }

    long long query(int idx) {
        long long res = 0;
        for (++idx; idx > 0; idx -= idx & -idx)
            res += bit[idx];
        return res;
    }

    long long rangeQuery(int l, int r) {
        return query(r) - (l ? query(l-1) : 0);
    }
};
```

## 🚀 Implementation (C#)
```cpp
public class Fenwick {
    private readonly long[] bit;
    private readonly int n;

    public Fenwick(int n) {
        this.n = n;
        bit = new long[n + 1];
    }

    public void Update(int idx, long delta) {
        for (idx++; idx <= n; idx += idx & -idx)
            bit[idx] += delta;
    }

    public long Query(int idx) {
        long res = 0;
        for (idx++; idx > 0; idx -= idx & -idx)
            res += bit[idx];
        return res;
    }

    public long RangeQuery(int l, int r) {
        return Query(r) - (l > 0 ? Query(l - 1) : 0);
    }
}
```

## ⏱️ Complexity Analysis

- **Preprocessing:** O(n) if built from array (O(n log n) with naive updates).  
- **Update:** O(log n).  
- **Prefix query:** O(log n).  
- **Range query:** O(log n).  
- **Space:** O(n).  

---

## ⚠️ Pitfalls

- **Indexing:** BIT is usually 1‑based internally; be careful with 0‑based input.  
- **Range updates:** Need two BITs or a trick with differential arrays.  
- **Overflow:** Use `long`/`long long` for large sums.  
- **Non‑commutative ops:** Works only for invertible, associative operations like sum/xor.  

---

## ✅ Conclusion

Fenwick Tree is a **Prefix Sum Engine**:

- ⚡ Supports O(log n) updates and queries  
- 📊 Ideal for frequency tables and inversion counting  
- 🔗 Can be extended for range updates with two BITs  
- 🛡️ Simpler and more memory‑efficient than segment trees  

👉 **Key takeaway:** Fenwick Tree is the lightweight workhorse for prefix sums and frequency queries — elegant, efficient, and indispensable in competitive programming.


---
