# Sqrt Decomposition

## Origin & Motivation

**Sqrt Decomposition** is a general algorithmic technique that splits a problem into blocks of size `B ≈ √n`, achieving **O(√n)** per query or update instead of O(n) brute force, at the cost of O(√n) extra space or preprocessing. It is the canonical "between O(1) and O(n)" tradeoff.

The intuition: divide n elements into √n blocks of √n each. Maintain a per-block aggregate (sum, min, max, XOR, ...). A query spanning `[l, r]` has at most O(√n) complete blocks (answered in O(1) each via the aggregate) and at most O(√n) partial elements at the boundaries (answered by brute force). Total: O(√n) per query. Updates modify one element and one block aggregate in O(1).

Unlike segment trees (O(log n) per operation, O(n) space, complex lazy propagation), sqrt decomposition is **simpler to implement correctly**, handles a wider variety of operations (including those that are hard to make lazy), and is competitive for n ≤ 10^6 where √n ≈ 1000.

Complexity: **O(√n)** per query/update, **O(n)** build, **O(√n)** extra space.

---

## Where It Is Used

- Range sum / min / max queries with point updates
- Range updates with range queries (more complex block maintenance)
- Mo's algorithm (offline range queries via block-sorted traversal)
- Sqrt decomposition of queries (batch processing)
- Heavy-light-style decomposition on sequences
- Competitive programming: operations too complex for lazy segment tree
- Approximate nearest neighbor (grid-based sqrt bucketing)

---

## Core Pattern

```
Block size B = ceil(sqrt(n))
Number of blocks K = ceil(n / B)

block[i] = aggregate of A[i*B .. (i+1)*B - 1]

Query [l, r]:
    if l and r in same block: brute force O(B)
    else:
        brute force left partial block:  O(B)
        answer complete middle blocks:   O(K) = O(n/B)
        brute force right partial block: O(B)
    Total: O(B + n/B)  optimized at B = sqrt(n): O(sqrt(n))

Point update A[i] = val:
    A[i] = val
    recompute block[i / B]
    Total: O(B) or O(1) depending on operation
```

---

## Variants and Applications

### 1. Range Sum Query + Point Update — O(√n)

Block stores sum. Query sums partial elements + complete block sums. Update replaces one element and recomputes one block sum in O(1) (subtract old, add new).

### 2. Range Min/Max Query + Point Update — O(√n)

Block stores min/max. Query brute-forces partial blocks, takes min/max of complete block values. Update recomputes block min/max in O(B) = O(√n).

### 3. Range Update + Point Query — O(√n) / O(1)

Maintain a "lazy delta" per block. Range update [l, r] by `+val`:
- Partial blocks: update elements directly (O(B))
- Complete blocks: add val to block's lazy delta (O(1) per block)
Point query A[i] = A[i] + block_delta[i / B] in O(1).

### 4. Range Update + Range Query — O(√n)

Combine both: each block has a lazy delta and a block sum. Range sum query accounts for both individual elements and deltas.

### 5. Frequency / Order Statistics

Count elements in [l, r] with value in [a, b]:
- Keep each block sorted
- Binary search on complete blocks: O(√n * log √n)
- Brute force partial blocks: O(√n)

Update: re-sort the block containing the modified element: O(√n log √n).
This replaces a merge sort tree or wavelet tree with simpler code at the same complexity.

---

## Complexity Analysis

| Operation | Block size B | Time per op | Optimal B |
|---|---|---|---|
| Range query | B | O(B + n/B) | B = √n → O(√n) |
| Point update (sum/XOR) | B | O(1) | Any B |
| Point update (min/max) | B | O(B) | B = √n → O(√n) |
| Range update + range query | B | O(B + n/B) | B = √n → O(√n) |
| Freq query (sorted blocks) | B | O(B + √n log B) | B = √n → O(√n log n) |

For q queries on array of size n: total O(q√n) after O(n) build.

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// 1. RANGE SUM + POINT UPDATE — O(sqrt(n)) per query, O(1) update
// ================================================================
struct SqrtSum {
    int n, B;
    vector<ll> A, block;

    SqrtSum(vector<ll>& a) : n(a.size()), B(max(1,(int)sqrt(n))),
        A(a), block(ceil((double)n/B), 0)
    {
        for (int i = 0; i < n; i++)
            block[i / B] += A[i];
    }

    // Point update: A[i] = val
    void update(int i, ll val) {
        block[i / B] += val - A[i];
        A[i] = val;
    }

    // Range sum [l, r]
    ll query(int l, int r) {
        ll sum = 0;
        int bl = l / B, br = r / B;
        if (bl == br) {
            for (int i = l; i <= r; i++) sum += A[i];
        } else {
            // Left partial block
            for (int i = l; i < (bl+1)*B; i++) sum += A[i];
            // Complete middle blocks
            for (int b = bl+1; b < br; b++) sum += block[b];
            // Right partial block
            for (int i = br*B; i <= r; i++) sum += A[i];
        }
        return sum;
    }
};

// ================================================================
// 2. RANGE MIN + POINT UPDATE — O(sqrt(n)) per query and update
// ================================================================
struct SqrtMin {
    int n, B;
    vector<ll> A, block;

    SqrtMin(vector<ll>& a) : n(a.size()), B(max(1,(int)sqrt(n))),
        A(a), block(ceil((double)n/B), LLONG_MAX)
    {
        for (int i = 0; i < n; i++)
            block[i / B] = min(block[i / B], A[i]);
    }

    // Point update: O(B)
    void update(int i, ll val) {
        A[i] = val;
        int b = i / B;
        block[b] = LLONG_MAX;
        int lo = b * B, hi = min(n-1, lo + B - 1);
        for (int j = lo; j <= hi; j++) block[b] = min(block[b], A[j]);
    }

    // Range min [l, r]: O(sqrt(n))
    ll query(int l, int r) {
        ll res = LLONG_MAX;
        int bl = l / B, br = r / B;
        if (bl == br) {
            for (int i = l; i <= r; i++) res = min(res, A[i]);
        } else {
            for (int i = l; i < (bl+1)*B; i++) res = min(res, A[i]);
            for (int b = bl+1; b < br; b++) res = min(res, block[b]);
            for (int i = br*B; i <= r; i++) res = min(res, A[i]);
        }
        return res;
    }
};

// ================================================================
// 3. RANGE ADD UPDATE + RANGE SUM QUERY — O(sqrt(n))
// ================================================================
struct SqrtRangeAdd {
    int n, B;
    vector<ll> A;       // individual elements
    vector<ll> lazy;    // lazy delta per block
    vector<ll> block_sum; // block sum without lazy

    SqrtRangeAdd(vector<ll>& a) : n(a.size()), B(max(1,(int)sqrt(n))),
        A(a), lazy((n+B-1)/B, 0), block_sum((n+B-1)/B, 0)
    {
        for (int i = 0; i < n; i++) block_sum[i/B] += A[i];
    }

    // Range add [l, r] += val
    void update(int l, int r, ll val) {
        int bl = l / B, br = r / B;
        if (bl == br) {
            for (int i = l; i <= r; i++) { A[i] += val; block_sum[bl] += val; }
        } else {
            // Left partial block
            for (int i = l; i < (bl+1)*B; i++) { A[i] += val; block_sum[bl] += val; }
            // Complete middle blocks: just update lazy
            for (int b = bl+1; b < br; b++) {
                lazy[b] += val;
                block_sum[b] += val * B;
            }
            // Right partial block
            for (int i = br*B; i <= r; i++) { A[i] += val; block_sum[br] += val; }
        }
    }

    // Range sum [l, r]
    ll query(int l, int r) {
        ll sum = 0;
        int bl = l / B, br = r / B;
        if (bl == br) {
            for (int i = l; i <= r; i++) sum += A[i] + lazy[bl];
        } else {
            for (int i = l; i < (bl+1)*B; i++) sum += A[i] + lazy[bl];
            for (int b = bl+1; b < br; b++) sum += block_sum[b];
            for (int i = br*B; i <= r; i++) sum += A[i] + lazy[br];
        }
        return sum;
    }

    // Point query A[i]
    ll point_query(int i) { return A[i] + lazy[i / B]; }
};

// ================================================================
// 4. FREQUENCY QUERY — count elements in [l,r] with value in [a,b]
// Each block kept sorted. O(sqrt(n) * log(sqrt(n))) per query.
// O(sqrt(n) * log(sqrt(n))) per update (re-sort one block).
// ================================================================
struct SqrtFreq {
    int n, B;
    vector<int> A;
    vector<vector<int>> sorted_block;

    SqrtFreq(vector<int>& a) : n(a.size()), B(max(1,(int)sqrt(n))),
        A(a), sorted_block((n+B-1)/B)
    {
        for (int i = 0; i < n; i++) sorted_block[i/B].push_back(A[i]);
        for (auto& b : sorted_block) sort(b.begin(), b.end());
    }

    // Count elements in [l, r] with value in [lo_val, hi_val]
    int query(int l, int r, int lo_val, int hi_val) {
        int cnt = 0;
        int bl = l / B, br = r / B;
        if (bl == br) {
            for (int i = l; i <= r; i++)
                if (A[i] >= lo_val && A[i] <= hi_val) cnt++;
        } else {
            // Partial left block
            for (int i = l; i < (bl+1)*B; i++)
                if (A[i] >= lo_val && A[i] <= hi_val) cnt++;
            // Complete middle blocks: binary search
            for (int b = bl+1; b < br; b++) {
                auto& sb = sorted_block[b];
                int lo_idx = (int)(lower_bound(sb.begin(), sb.end(), lo_val) - sb.begin());
                int hi_idx = (int)(upper_bound(sb.begin(), sb.end(), hi_val) - sb.begin());
                cnt += hi_idx - lo_idx;
            }
            // Partial right block
            for (int i = br*B; i <= r; i++)
                if (A[i] >= lo_val && A[i] <= hi_val) cnt++;
        }
        return cnt;
    }

    // Point update: A[idx] = val
    void update(int idx, int val) {
        int b = idx / B;
        // Remove old value from sorted block
        auto& sb = sorted_block[b];
        sb.erase(lower_bound(sb.begin(), sb.end(), A[idx]));
        // Insert new value
        sb.insert(lower_bound(sb.begin(), sb.end(), val), val);
        A[idx] = val;
    }
};

// ================================================================
// 5. SQRT DECOMPOSITION ON QUERIES (Sqrt of queries)
// Process queries in batches of sqrt(q). Within a batch,
// handle "pending updates" via brute force.
// ================================================================
struct SqrtQueryBatch {
    // Example: offline queries with updates
    // Each query [l,r,time] answered using elements up to that time
    // Batch size B_q = sqrt(q): within batch, check pending updates O(B_q)
    // Across batches: rebuild structure O(n) per batch

    int n;
    vector<int> A;

    SqrtQueryBatch(vector<int>& a) : n(a.size()), A(a) {}

    // Process mixed updates and queries offline
    // update = {0, i, val}: A[i] = val
    // query  = {1, l, r}:   sum of A[l..r]
    vector<ll> process(vector<tuple<int,int,int>>& ops) {
        int q = ops.size();
        int Bq = max(1, (int)sqrt(q));
        vector<ll> answers;

        // For each query, brute-force within its batch window
        // Segment tree or Fenwick handles cross-batch queries
        // (simplified: just brute force everything for demo)

        vector<int> cur = A;
        vector<pair<int,int>> pending_updates; // {index, value}

        for (int i = 0; i < q; i++) {
            auto [type, x, y] = ops[i];
            if (type == 0) {
                // Update
                pending_updates.push_back({x, y});
            } else {
                // Query [x, y]: sum
                ll sum = 0;
                for (int j = x; j <= y; j++) sum += cur[j];
                // Also check pending updates in this batch
                for (auto [idx, val] : pending_updates)
                    if (idx >= x && idx <= y) sum += val - cur[idx];
                answers.push_back(sum);
            }

            // Flush batch
            if ((i+1) % Bq == 0) {
                for (auto [idx, val] : pending_updates) cur[idx] = val;
                pending_updates.clear();
            }
        }
        return answers;
    }
};

// ================================================================
// Usage
// ================================================================
int main() {
    // Range sum + point update
    {
        printf("=== Range Sum + Point Update ===\n");
        vector<ll> A = {1,2,3,4,5,6,7,8,9,10};
        SqrtSum sq(A);
        printf("sum[2,7] = %lld\n", sq.query(2,7));  // 3+4+5+6+7+8=33
        sq.update(5, 100);                             // A[5] = 100
        printf("After update A[5]=100:\n");
        printf("sum[2,7] = %lld\n", sq.query(2,7));  // 3+4+5+100+7+8=127
        printf("sum[0,9] = %lld\n", sq.query(0,9));  // 1+2+3+4+5+100+7+8+9+10=149
    }

    // Range min + point update
    {
        printf("\n=== Range Min + Point Update ===\n");
        vector<ll> A = {5,3,8,1,9,2,7,4,6,10};
        SqrtMin sq(A);
        printf("min[1,6] = %lld\n", sq.query(1,6));  // min(3,8,1,9,2,7)=1
        sq.update(3, 10);                              // A[3] = 10
        printf("After update A[3]=10:\n");
        printf("min[1,6] = %lld\n", sq.query(1,6));  // min(3,8,10,9,2,7)=2
    }

    // Range add + range sum
    {
        printf("\n=== Range Add + Range Sum ===\n");
        vector<ll> A = {1,2,3,4,5,6,7,8,9,10};
        SqrtRangeAdd sq(A);
        printf("sum[2,7] = %lld\n", sq.query(2,7));  // 33
        sq.update(3, 6, 10);                           // A[3..6] += 10
        printf("After range_add[3,6]=+10:\n");
        printf("sum[2,7] = %lld\n", sq.query(2,7));  // 33+40=73
        printf("sum[0,9] = %lld\n", sq.query(0,9));  // 55+40=95
        printf("A[5] = %lld\n", sq.point_query(5));  // 6+10=16
    }

    // Frequency query
    {
        printf("\n=== Frequency Query ===\n");
        vector<int> A = {3,1,4,1,5,9,2,6,5,3};
        SqrtFreq sq(A);
        printf("count in [1,8] with val in [3,6] = %d\n",
               sq.query(1, 8, 3, 6));  // 4,5,2,6,5,3 -> 4,5,6,5,3 in range = {4,5,6,5,3} = 5? 
        // A[1..8] = {1,4,1,5,9,2,6,5} val in [3,6] = {4,5,6,5} = 4
        sq.update(0, 5);
        printf("After update A[0]=5:\n");
        printf("count in [0,4] with val in [3,6] = %d\n",
               sq.query(0, 4, 3, 6));
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

// ================================================================
// RANGE SUM + POINT UPDATE
// ================================================================
public class SqrtSum {
    private readonly int n, B;
    private readonly long[] A, block;

    public SqrtSum(long[] a) {
        n = a.Length; B = Math.Max(1, (int)Math.Sqrt(n));
        A = (long[])a.Clone();
        block = new long[(n + B - 1) / B];
        for (int i = 0; i < n; i++) block[i / B] += A[i];
    }

    public void Update(int i, long val) {
        block[i / B] += val - A[i]; A[i] = val;
    }

    public long Query(int l, int r) {
        long sum = 0;
        int bl = l / B, br = r / B;
        if (bl == br) { for (int i=l;i<=r;i++) sum+=A[i]; return sum; }
        for (int i=l; i<(bl+1)*B; i++) sum+=A[i];
        for (int b=bl+1; b<br; b++) sum+=block[b];
        for (int i=br*B; i<=r; i++) sum+=A[i];
        return sum;
    }
}

// ================================================================
// RANGE ADD + RANGE SUM
// ================================================================
public class SqrtRangeAdd {
    private readonly int n, B;
    private readonly long[] A, lazy, blockSum;

    public SqrtRangeAdd(long[] a) {
        n=a.Length; B=Math.Max(1,(int)Math.Sqrt(n));
        A=(long[])a.Clone();
        int K=(n+B-1)/B;
        lazy=new long[K]; blockSum=new long[K];
        for(int i=0;i<n;i++) blockSum[i/B]+=A[i];
    }

    public void Update(int l, int r, long val) {
        int bl=l/B, br=r/B;
        if(bl==br) { for(int i=l;i<=r;i++){A[i]+=val;blockSum[bl]+=val;} return; }
        for(int i=l;i<(bl+1)*B;i++){A[i]+=val;blockSum[bl]+=val;}
        for(int b=bl+1;b<br;b++){lazy[b]+=val;blockSum[b]+=val*B;}
        for(int i=br*B;i<=r;i++){A[i]+=val;blockSum[br]+=val;}
    }

    public long Query(int l, int r) {
        long sum=0;
        int bl=l/B, br=r/B;
        if(bl==br){for(int i=l;i<=r;i++)sum+=A[i]+lazy[bl];return sum;}
        for(int i=l;i<(bl+1)*B;i++)sum+=A[i]+lazy[bl];
        for(int b=bl+1;b<br;b++)sum+=blockSum[b];
        for(int i=br*B;i<=r;i++)sum+=A[i]+lazy[br];
        return sum;
    }

    public long PointQuery(int i) => A[i] + lazy[i/B];
}

// ================================================================
// FREQUENCY QUERY (sorted blocks)
// ================================================================
public class SqrtFreq {
    private readonly int n, B;
    private readonly int[] A;
    private readonly List<int>[] sortedBlock;

    public SqrtFreq(int[] a) {
        n=a.Length; B=Math.Max(1,(int)Math.Sqrt(n));
        A=(int[])a.Clone();
        int K=(n+B-1)/B;
        sortedBlock=new List<int>[K];
        for(int i=0;i<K;i++) sortedBlock[i]=new();
        for(int i=0;i<n;i++) sortedBlock[i/B].Add(A[i]);
        foreach(var b in sortedBlock) b.Sort();
    }

    private static int CountInRange(List<int> sb, int lo, int hi) {
        int l=(int)(System.Linq.Enumerable.TakeWhile(sb,x=>x<lo).Count());
        // Use BinarySearch instead:
        int lo_idx=sb.BinarySearch(lo);
        if(lo_idx<0) lo_idx=~lo_idx;
        int hi_idx=sb.BinarySearch(hi+1);
        if(hi_idx<0) hi_idx=~hi_idx;
        return hi_idx-lo_idx;
    }

    public int Query(int l, int r, int loVal, int hiVal) {
        int cnt=0, bl=l/B, br=r/B;
        if(bl==br){for(int i=l;i<=r;i++)if(A[i]>=loVal&&A[i]<=hiVal)cnt++;return cnt;}
        for(int i=l;i<(bl+1)*B;i++)if(A[i]>=loVal&&A[i]<=hiVal)cnt++;
        for(int b=bl+1;b<br;b++) cnt+=CountInRange(sortedBlock[b],loVal,hiVal);
        for(int i=br*B;i<=r;i++)if(A[i]>=loVal&&A[i]<=hiVal)cnt++;
        return cnt;
    }

    public void Update(int idx, int val) {
        int b=idx/B;
        sortedBlock[b].Remove(A[idx]);
        int ins=sortedBlock[b].BinarySearch(val);
        if(ins<0) ins=~ins;
        sortedBlock[b].Insert(ins,val);
        A[idx]=val;
    }
}

public class Program {
    public static void Main() {
        // Range sum
        var sq = new SqrtSum(new long[]{1,2,3,4,5,6,7,8,9,10});
        Console.WriteLine($"sum[2,7]={sq.Query(2,7)}");  // 33
        sq.Update(5, 100);
        Console.WriteLine($"sum[2,7] after update={sq.Query(2,7)}");  // 127

        // Range add + sum
        var ra = new SqrtRangeAdd(new long[]{1,2,3,4,5,6,7,8,9,10});
        Console.WriteLine($"sum[2,7]={ra.Query(2,7)}");
        ra.Update(3,6,10);
        Console.WriteLine($"sum[2,7] after range_add={ra.Query(2,7)}");
        Console.WriteLine($"A[5]={ra.PointQuery(5)}");

        // Freq query
        var sf = new SqrtFreq(new[]{3,1,4,1,5,9,2,6,5,3});
        Console.WriteLine($"freq[1,8] in [3,6] = {sf.Query(1,8,3,6)}");
    }
}
```

---

## Block Size Optimization

The standard choice B = √n is optimal when query and update costs are symmetric. In practice:

```
Query cost:  O(B + n/B)     minimized at B = sqrt(n)
Update cost: O(1) or O(B)

If update_cost >> query_cost: increase B (fewer blocks, faster queries)
If query_cost >> update_cost: decrease B (smaller partial scans)

For q queries, n elements: total cost O(q * (B + n/B))
Optimal B = sqrt(n) → O(q * sqrt(n))

If updates cost O(B): balance B + n/B vs B → B = sqrt(n) still optimal
If offline: may adjust B based on ratio of queries to updates
```

---

## Pitfalls

- **Block size B = 0 for small arrays** — when n=1, `sqrt(n)=1`, which is fine. But `(int)sqrt(0)=0` causes division by zero. Always set `B = max(1, (int)sqrt(n))`.
- **Last block may be smaller than B** — the last block has `n - (K-1)*B` elements where K = ceil(n/B). When iterating block indices, the right boundary of the last block is `min(n-1, (b+1)*B - 1)`, not `(b+1)*B - 1`. Out-of-bounds access in the last block is the most common sqrt bug.
- **Range update + range query: lazy must be applied at query time** — when using lazy deltas for range updates, elements in partial blocks are updated directly (lazy=0 for their block), but elements in complete blocks accumulate via `lazy[b]`. At query time, partial block elements must have `A[i] + lazy[b]` evaluated, not just `A[i]`.
- **Sorted block update: use proper insertion, not re-sort** — for frequency queries, maintaining sorted blocks requires removing the old value and inserting the new one at the correct position. Re-sorting the entire block on every update costs O(B log B) instead of O(log B). Use `lower_bound` + erase/insert for O(log B) updates.
- **Mo's algorithm is a different technique** — Mo's uses sqrt-sized blocks for offline range query ordering (sorting queries by block of left endpoint, then right endpoint). It is not the same as the sqrt decomposition array structure. Both use B ≈ √n but for fundamentally different purposes.
- **Floating-point sqrt** — `(int)sqrt(n)` may give n-1 or n+1 due to floating-point imprecision for perfect squares. Use `(int)sqrt((double)n + 0.5)` or compute B as the smallest integer with B² ≥ n: `B = 1; while(B*B < n) B++;`.

---

## Comparison with Segment Tree

| Property | Sqrt Decomposition | Segment Tree |
|---|---|---|
| Time per op | O(√n) | O(log n) |
| Build time | O(n) | O(n) |
| Space | O(n) | O(n) |
| Lazy propagation | Not needed for simple ops | Required for range updates |
| Implementation complexity | Low | Moderate to high |
| Range update + query | O(√n) | O(log n) with lazy |
| Non-standard aggregates | Easy (brute force partial) | Hard (may not be lazy-able) |
| Constant factor | Large (√n ≈ 300–1000) | Small (log n ≈ 17–20) |

**When to prefer sqrt decomposition:**
- Operation is hard to make lazy (e.g., sort elements in range)
- n ≤ 10^5 and constant factor matters
- Implementation time is limited
- Operation involves full block recomputation anyway

---

## Conclusion

Sqrt Decomposition is the **universal fallback for range query problems** — when the operation is too complex for lazy segment trees, sqrt gives O(√n) per operation with minimal implementation effort:

- The core idea is always the same: split into √n blocks, maintain a per-block aggregate, handle partial blocks by brute force and complete blocks via the aggregate.
- Range sum with point updates, range min/max, range add with range queries, and frequency queries all fit this template with only the aggregate and update logic changing.
- The optimal block size B = √n balances partial-block brute force (O(B)) against complete-block aggregation (O(n/B)), both becoming O(√n).
- For n = 10^6 and q = 10^6 queries: total O(q√n) = O(10^9) — marginal but often fast enough with small constants.

**Key takeaway:**  
When a problem requires range queries with updates and the operation resists lazy propagation (e.g., k-th order statistic, range sorting, range XOR with non-invertible aggregate), reach for sqrt decomposition before attempting complex segment tree extensions. The implementation is ~20 lines, the complexity is provably O(√n) per operation, and the correctness argument is straightforward.
