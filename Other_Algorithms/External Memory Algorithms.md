# External Memory Algorithms

## Origin & Motivation

**External Memory Algorithms** are designed for data too large to fit in RAM, where the dominant cost is **I/O operations** (disk block transfers) rather than CPU instructions. The formal model was introduced by Aggarwal and Vitter in 1988: the **I/O model** (also called the disk-access model, DAM).

**The I/O model:**
```
N = number of elements (total data size)
M = number of elements that fit in internal memory (RAM)
B = number of elements per disk block (transfer unit)

A single I/O operation reads or writes one block of B elements.
Cost = number of I/O operations (block transfers), NOT CPU operations.
```

**The problem it solves:** When `N` is much larger than `M` (e.g. sorting a terabyte of data with 16GB RAM), an algorithm with optimal CPU complexity but poor I/O behavior (e.g. quicksort doing N random accesses) becomes catastrophically slow — each I/O takes milliseconds (disk seek) vs nanoseconds (RAM access), a factor of 10⁵–10⁶. External memory algorithms minimize the **number of block transfers**, achieving bounds like `O((N/B) log_{M/B}(N/B))` for sorting instead of `O(N log N)` comparisons.

Complexity: External sort **O((N/B) log_{M/B}(N/B))** I/Os, external B-tree search **O(log_B N)** I/Os per operation.

---

## Where It Is Used

- Database systems: B-trees for indexes, external sort-merge for `ORDER BY` and joins
- File systems: directory structures (B-trees, B+ trees) for fast lookup on disk
- Big data processing: external merge sort underlies sort-based shuffle in MapReduce/Spark
- Geographic information systems: R-trees (spatial B-trees) for disk-resident spatial data
- Scientific computing: out-of-core matrix algorithms for datasets exceeding RAM
- Search engines: external sorting of posting lists, inverted index construction

---

## Core Structure: The I/O Model

### Key Parameters

```
N = problem size (elements)
M = memory size (elements that fit in RAM)
B = block size (elements transferred per I/O)

Assumption: M < N  (data doesn't fit in memory)
Assumption: B <= M/2  (at least 2 blocks fit in memory, often M/B >> 1)
```

### Fundamental Bounds

```
Scanning N elements:        O(N/B) I/Os    (sequential read)
Sorting N elements:         O((N/B) log_{M/B}(N/B)) I/Os    (external sort)
Searching (static, sorted): O(log_B N) I/Os    (B-tree)
```

The term `log_{M/B}(N/B)` is the number of **merge passes** — using base `M/B` instead of base 2 dramatically reduces the height: with `M/B = 100`, `log_{100}(10⁹) ≈ 4.5` passes instead of `log_2(10⁹) ≈ 30`.

---

## External Merge Sort

### Algorithm

```
ExternalMergeSort(N elements, block size B, memory M):

  Phase 1 — Create sorted runs:
    while input remains:
        read M elements into memory       (M/B I/Os)
        sort internally (e.g. quicksort)  (0 I/Os — in RAM)
        write sorted run to disk          (M/B I/Os)
    → produces ceil(N/M) sorted runs, each of size M

  Phase 2 — k-way merge:
    k = M/B - 1    (reserve 1 block for output buffer)
    while more than 1 run remains:
        merge groups of k runs into one larger sorted run
        each merge pass: read all N elements, write all N elements
        → O(N/B) I/Os per pass
    number of passes = ceil(log_k(N/M)) = O(log_{M/B}(N/B))

  Total I/Os: O(N/B) [Phase 1] + O((N/B) * log_{M/B}(N/B)) [Phase 2]
            = O((N/B) log_{M/B}(N/B))
```

### Why k-way Merge with k = M/B - 1?

During a merge pass, we need one input buffer per run being merged plus one output buffer:

```
Memory layout during merge:
  [buffer for run 1] [buffer for run 2] ... [buffer for run k] [output buffer]
  k + 1 buffers, each of size B, total = (k+1)*B <= M
  → k <= M/B - 1
```

Choosing `k = M/B - 1` maximizes the merge fan-in, minimizing the number of passes `log_k(N/M)`.

---

## External B-Tree

### Algorithm

An external B-tree is a B-tree where **each node is exactly one disk block**, holding up to `B-1` keys and `B` child pointers (fan-out `B`). This gives height `O(log_B N)`.

```
Node structure (fits in one block of B elements):
  keys[0..k-1]       (k <= B-1 keys, sorted)
  children[0..k]     (k+1 child block pointers, internal nodes only)

Search(key):
    node = root
    while node is not a leaf:
        read node from disk         (1 I/O)
        find child via binary search on keys (in-memory, 0 I/O)
        node = child
    read leaf node                  (1 I/O)
    binary search for key in leaf

Height = O(log_B N)  →  Search = O(log_B N) I/Os
```

### Insert with Node Splitting

```
Insert(key):
    if root is full (B-1 keys):
        create new root, split old root → height increases by 1
    InsertNonFull(root, key):
        read node                   (1 I/O)
        if node is leaf:
            insert key in sorted position
            write node               (1 I/O)
        else:
            find child i for key
            if child i is full:
                SplitChild(node, i)  (2 I/Os: read+write both halves)
            InsertNonFull(child i, key)

Total I/Os per insert: O(log_B N)  (one read+write per level, plus occasional splits)
```

---

## Buffer Trees (Brief Overview)

For algorithms that perform many operations with **amortized** I/O guarantees better than `O(log_B N)` per operation, **Buffer Trees** (Arge, 1995) attach a buffer of size `Θ(M)` to each internal node. Operations are inserted into the root buffer and lazily pushed down when a buffer fills, achieving:

```
Amortized I/O per operation: O((1/B) log_{M/B}(N/B))
```

This is a factor of `B` better than `O(log_B N)` per operation — important for batch processing of many operations (e.g. priority queue with N total operations costing `O((N/B) log_{M/B}(N/B))` total, vs `O(N log_B N)` for N individual B-tree operations).

---

## Complexity Summary

| Operation | I/O Complexity | Notes |
|---|---|---|
| Scan N elements | O(N/B) | Sequential access |
| External sort | O((N/B) log_{M/B}(N/B)) | k = M/B-1 way merge |
| B-tree search | O(log_B N) | One I/O per level |
| B-tree insert/delete | O(log_B N) | Plus split/merge overhead |
| Buffer tree operation (amortized) | O((1/B) log_{M/B}(N/B)) | Batched operations |
| Binary search (unsorted blocks) | O(log_2 N) | NOT O(log_B N) — naive |

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;

// ================================================================
// SIMULATED DISK
// Tracks block reads/writes to measure I/O complexity.
// Block size B, memory size M (in elements).
// ================================================================
const int BLOCK_SIZE = 8;   // B: elements per disk block
const int MEMORY     = 32;  // M: elements that fit in RAM (M/B = 4 buffers)

struct DiskBlock {
    vector<int> data;
    DiskBlock() : data(BLOCK_SIZE, INT_MAX) {}
};

struct Disk {
    vector<DiskBlock> blocks;
    ll reads = 0, writes = 0;

    int alloc() {
        blocks.push_back(DiskBlock());
        return (int)blocks.size() - 1;
    }
    const DiskBlock& read(int id)  { reads++;  return blocks[id]; }
    void write(int id, const DiskBlock& b) { writes++; blocks[id] = b; }
    void reset() { reads = writes = 0; }
};

// ================================================================
// EXTERNAL MERGE SORT — O((N/B) log_{M/B}(N/B)) I/Os
//
// Phase 1: split input into runs of size M, sort each in RAM, write to disk.
// Phase 2: repeatedly merge pairs of runs until one run remains.
//          (2-way merge shown for clarity; production code uses
//           k = M/B - 1 way merge for fewer passes)
// ================================================================
void external_merge_sort(vector<int>& arr, Disk& disk) {
    int n = (int)arr.size();
    int B = BLOCK_SIZE, M = MEMORY;

    // ---- Phase 1: create sorted runs of size M ----
    vector<vector<int>> run_blocks; // run_blocks[r] = list of block ids for run r
    for (int lo = 0; lo < n; lo += M) {
        int hi = min(lo + M, n);
        vector<int> buf(arr.begin()+lo, arr.begin()+hi);
        sort(buf.begin(), buf.end()); // in-memory sort, 0 I/Os

        vector<int> blks;
        for (int b = 0; b * B < (int)buf.size(); b++) {
            int bid = disk.alloc();
            DiskBlock blk;
            for (int j = 0; j < B; j++) {
                int idx = b*B + j;
                blk.data[j] = (idx < (int)buf.size()) ? buf[idx] : INT_MAX;
            }
            disk.write(bid, blk);
            blks.push_back(bid);
        }
        run_blocks.push_back(blks);
    }

    // ---- Phase 2: pairwise merge passes ----
    // Each pass halves the number of runs.
    // Number of passes = ceil(log2(num_runs)) ≈ O(log_{M/B}(N/B)) for k-way
    while (run_blocks.size() > 1) {
        vector<vector<int>> next_runs;
        for (size_t i = 0; i+1 < run_blocks.size(); i += 2) {
            // Read both runs fully (sequential, O(run_size/B) I/Os each)
            vector<int> merged;
            for (int bid : run_blocks[i]) {
                const auto& blk = disk.read(bid);
                for (int x : blk.data) if (x != INT_MAX) merged.push_back(x);
            }
            vector<int> r2;
            for (int bid : run_blocks[i+1]) {
                const auto& blk = disk.read(bid);
                for (int x : blk.data) if (x != INT_MAX) r2.push_back(x);
            }
            // Merge two sorted sequences
            vector<int> out(merged.size()+r2.size());
            std::merge(merged.begin(),merged.end(),r2.begin(),r2.end(),out.begin());

            // Write merged run back to disk
            vector<int> new_blks;
            for (int b = 0; b*B < (int)out.size(); b++) {
                int bid = disk.alloc();
                DiskBlock blk;
                for (int j = 0; j < B; j++) {
                    int idx = b*B+j;
                    blk.data[j] = (idx < (int)out.size()) ? out[idx] : INT_MAX;
                }
                disk.write(bid, blk);
                new_blks.push_back(bid);
            }
            next_runs.push_back(new_blks);
        }
        if (run_blocks.size() % 2 == 1) next_runs.push_back(run_blocks.back());
        run_blocks = next_runs;
    }

    // ---- Read final sorted run back into arr ----
    arr.clear();
    for (int bid : run_blocks[0]) {
        const auto& blk = disk.read(bid);
        for (int x : blk.data) if (x != INT_MAX) arr.push_back(x);
    }
}

// ================================================================
// EXTERNAL B-TREE — O(log_B N) I/Os per search/insert
//
// Each node holds up to 2*ORDER-1 keys and 2*ORDER children
// (simulating one disk block of fan-out B = 2*ORDER).
// ================================================================
struct ExtBTree {
    static const int ORDER = BLOCK_SIZE/2; // min degree; max keys = 2*ORDER-1

    struct Node {
        vector<int> keys;
        vector<int> children; // node ids (simulated disk block ids)
        bool leaf = true;
    };

    Disk& disk;
    vector<Node> nodes; // simulated disk blocks (one Node = one block)
    int root_id;

    ExtBTree(Disk& d) : disk(d) {
        root_id = new_node(true);
    }

    int new_node(bool leaf) {
        nodes.push_back(Node());
        nodes.back().leaf = leaf;
        return (int)nodes.size() - 1;
    }

    // Search for key — O(log_B N) I/Os (one read per level)
    bool search(int key) { return search_node(root_id, key); }

    bool search_node(int id, int key) {
        if (id < 0) return false;
        Node& nd = nodes[id];
        disk.reads++; // 1 I/O: read this node/block
        int i = 0;
        while (i < (int)nd.keys.size() && key > nd.keys[i]) i++;
        if (i < (int)nd.keys.size() && nd.keys[i] == key) return true;
        if (nd.leaf) return false;
        return search_node(nd.children[i], key);
    }

    // Insert key — O(log_B N) I/Os amortized (set semantics: no duplicates)
    void insert(int key) {
        if ((int)nodes[root_id].keys.size() == 2*ORDER-1) {
            int new_root = new_node(false);
            nodes[new_root].children.push_back(root_id);
            split_child(new_root, 0);
            root_id = new_root;
        }
        insert_nonfull(root_id, key);
    }

    // Split full child i of parent — 1 read + 2 writes (3 I/Os)
    void split_child(int parent_id, int i) {
        int full_id = nodes[parent_id].children[i];
        int new_id  = new_node(nodes[full_id].leaf);
        disk.reads++; disk.writes += 2; // read full node, write both halves

        Node& full = nodes[full_id];
        Node& nw   = nodes[new_id];
        int mid = ORDER - 1;
        int sep = full.keys[mid];

        nw.keys.assign(full.keys.begin()+mid+1, full.keys.end());
        full.keys.resize(mid);
        if (!full.leaf) {
            nw.children.assign(full.children.begin()+mid+1, full.children.end());
            full.children.resize(mid+1);
        }
        nodes[parent_id].keys.insert(nodes[parent_id].keys.begin()+i, sep);
        nodes[parent_id].children.insert(nodes[parent_id].children.begin()+i+1, new_id);
    }

    // Insert into a node known not to be full — 1 read + 1 write per level
    void insert_nonfull(int id, int key) {
        Node& nd = nodes[id];
        disk.reads++; disk.writes++; // read node, (eventually) write it back
        if (nd.leaf) {
            auto it = lower_bound(nd.keys.begin(), nd.keys.end(), key);
            if (it != nd.keys.end() && *it == key) return; // duplicate: skip
            nd.keys.insert(it, key);
        } else {
            int i = (int)(lower_bound(nd.keys.begin(), nd.keys.end(), key) - nd.keys.begin());
            if (i < (int)nd.keys.size() && nd.keys[i] == key) return; // duplicate
            if ((int)nodes[nd.children[i]].keys.size() == 2*ORDER-1) {
                split_child(id, i);
                if (key > nodes[id].keys[i]) i++;
            }
            insert_nonfull(nd.children[i], key);
        }
    }

    // In-order traversal for verification
    void inorder(int id, vector<int>& out) {
        if (id < 0) return;
        Node& nd = nodes[id];
        for (int i = 0; i < (int)nd.keys.size(); i++) {
            if (!nd.leaf) inorder(nd.children[i], out);
            out.push_back(nd.keys[i]);
        }
        if (!nd.leaf) inorder(nd.children.back(), out);
    }
    vector<int> toSorted() { vector<int> v; inorder(root_id, v); return v; }
};

// ================================================================
// Usage + stress tests
// ================================================================
int main() {
    // ---- External Merge Sort demo ----
    {
        printf("=== External Merge Sort ===\n");
        srand(42);
        int n = 100;
        vector<int> arr(n);
        for (auto& x : arr) x = rand() % 1000;
        vector<int> ref = arr;
        sort(ref.begin(), ref.end());

        Disk disk;
        external_merge_sort(arr, disk);

        printf("n=%d  B=%d  M=%d\n", n, BLOCK_SIZE, MEMORY);
        printf("I/Os: reads=%lld writes=%lld total=%lld\n",
               disk.reads, disk.writes, disk.reads+disk.writes);
        double theory = (double)n/BLOCK_SIZE * log2((double)n/BLOCK_SIZE) /
                        log2((double)MEMORY/BLOCK_SIZE);
        printf("Theoretical O((N/B)log_{M/B}(N/B)) ~ %.1f block ops\n", theory);
        printf("Sorted correctly: %s\n", (arr==ref)?"YES":"NO");
    }

    // ---- External B-Tree demo ----
    {
        printf("\n=== External B-Tree ===\n");
        Disk disk;
        ExtBTree bt(disk);

        srand(99);
        vector<int> vals;
        for (int i = 0; i < 50; i++) {
            int v = rand() % 200;
            bt.insert(v);
            vals.push_back(v);
        }
        sort(vals.begin(), vals.end());
        vals.erase(unique(vals.begin(), vals.end()), vals.end());

        auto sorted = bt.toSorted();
        printf("Order (fan-out) B=%d, inserted %d unique values\n",
               2*ExtBTree::ORDER, (int)vals.size());
        printf("I/Os used for inserts: %lld\n", disk.reads+disk.writes);
        printf("In-order traversal correct: %s\n", (sorted==vals)?"YES":"NO");

        disk.reset();
        int n_search = (int)vals.size() + 20;
        for (int v : vals) bt.search(v);
        for (int i = 0; i < 20; i++) bt.search(rand()%200);
        printf("Avg I/Os per search: %.2f  (theoretical O(log_B N) ~ %.2f)\n",
               (double)disk.reads/n_search,
               log((double)vals.size())/log(2*ExtBTree::ORDER));
    }

    // ---- Stress: external merge sort vs std::sort ----
    {
        printf("\n=== Stress: External Merge Sort, 500 trials ===\n");
        srand(7); int errors = 0;
        for (int t = 0; t < 500; t++) {
            int n = 1 + rand()%150;
            vector<int> arr(n), ref(n);
            for (int i=0;i<n;i++) arr[i]=ref[i]=rand()%1000;
            sort(ref.begin(), ref.end());
            Disk disk;
            external_merge_sort(arr, disk);
            if (arr != ref) errors++;
        }
        printf("Result: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }

    // ---- Stress: external B-tree insert + search ----
    {
        printf("\n=== Stress: External B-Tree, 300 trials ===\n");
        srand(123); int errors = 0;
        for (int t = 0; t < 300; t++) {
            Disk disk;
            ExtBTree bt(disk);
            set<int> ref;
            int n = 1 + rand()%60;
            for (int i=0;i<n;i++){
                int v = rand()%100;
                bt.insert(v); ref.insert(v);
            }
            auto sorted = bt.toSorted();
            vector<int> exp(ref.begin(), ref.end());
            if (sorted != exp) errors++;
            // search check
            for (int v : exp) if (!bt.search(v)) errors++;
            for (int i=0;i<10;i++){
                int v=rand()%100;
                bool found=bt.search(v), expected=ref.count(v)>0;
                if (found!=expected) errors++;
            }
        }
        printf("Result: %s\n", errors==0?"OK":("FAIL "+to_string(errors)).c_str());
    }
    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;
using System.Linq;

// ================================================================
// SIMULATED DISK — C#
// ================================================================
public class DiskBlock
{
    public int[] Data;
    public DiskBlock(int blockSize) { Data = new int[blockSize]; Array.Fill(Data, int.MaxValue); }
}

public class Disk
{
    public List<DiskBlock> Blocks = new();
    public long Reads = 0, Writes = 0;
    private readonly int _blockSize;

    public Disk(int blockSize) { _blockSize = blockSize; }

    public int Alloc() { Blocks.Add(new DiskBlock(_blockSize)); return Blocks.Count-1; }
    public DiskBlock Read(int id) { Reads++; return Blocks[id]; }
    public void Write(int id, DiskBlock b) { Writes++; Blocks[id]=b; }
    public void Reset() { Reads=0; Writes=0; }
}

// ================================================================
// EXTERNAL MERGE SORT — C#
// ================================================================
public static class ExternalMergeSort
{
    public static void Sort(List<int> arr, Disk disk, int B, int M)
    {
        int n = arr.Count;
        var runBlocks = new List<List<int>>();

        // Phase 1: sorted runs of size M
        for (int lo=0; lo<n; lo+=M)
        {
            int hi=Math.Min(lo+M,n);
            var buf=arr.GetRange(lo,hi-lo);
            buf.Sort();

            var blks=new List<int>();
            for (int b=0; b*B<buf.Count; b++)
            {
                int bid=disk.Alloc();
                var blk=new DiskBlock(B);
                for (int j=0;j<B;j++){
                    int idx=b*B+j;
                    blk.Data[j]=idx<buf.Count?buf[idx]:int.MaxValue;
                }
                disk.Write(bid,blk);
                blks.Add(bid);
            }
            runBlocks.Add(blks);
        }

        // Phase 2: pairwise merge passes
        while (runBlocks.Count>1)
        {
            var next=new List<List<int>>();
            for (int i=0;i+1<runBlocks.Count;i+=2)
            {
                var r1=new List<int>();
                foreach(int bid in runBlocks[i])
                    foreach(int x in disk.Read(bid).Data) if(x!=int.MaxValue) r1.Add(x);
                var r2=new List<int>();
                foreach(int bid in runBlocks[i+1])
                    foreach(int x in disk.Read(bid).Data) if(x!=int.MaxValue) r2.Add(x);

                // 2-way merge
                var outp=new List<int>(r1.Count+r2.Count);
                int a=0,b2=0;
                while(a<r1.Count&&b2<r2.Count) outp.Add(r1[a]<=r2[b2]?r1[a++]:r2[b2++]);
                while(a<r1.Count) outp.Add(r1[a++]);
                while(b2<r2.Count) outp.Add(r2[b2++]);

                var newBlks=new List<int>();
                for (int b=0;b*B<outp.Count;b++){
                    int bid=disk.Alloc();
                    var blk=new DiskBlock(B);
                    for(int j=0;j<B;j++){
                        int idx=b*B+j;
                        blk.Data[j]=idx<outp.Count?outp[idx]:int.MaxValue;
                    }
                    disk.Write(bid,blk);
                    newBlks.Add(bid);
                }
                next.Add(newBlks);
            }
            if (runBlocks.Count%2==1) next.Add(runBlocks[^1]);
            runBlocks=next;
        }

        // Read back
        arr.Clear();
        foreach(int bid in runBlocks[0])
            foreach(int x in disk.Read(bid).Data) if(x!=int.MaxValue) arr.Add(x);
    }
}

// ================================================================
// EXTERNAL B-TREE — C#
// ================================================================
public class ExtBTree
{
    public class Node
    {
        public List<int> Keys = new();
        public List<int> Children = new();
        public bool Leaf = true;
    }

    private readonly Disk _disk;
    private readonly List<Node> _nodes = new();
    private int _root;
    private readonly int _order;

    public ExtBTree(Disk disk, int blockSize)
    {
        _disk = disk;
        _order = blockSize/2;
        _root = NewNode(true);
    }

    private int NewNode(bool leaf)
    {
        _nodes.Add(new Node{Leaf=leaf});
        return _nodes.Count-1;
    }

    public bool Search(int key) => SearchNode(_root, key);

    private bool SearchNode(int id, int key)
    {
        if (id<0) return false;
        var nd=_nodes[id];
        _disk.Reads++;
        int i=0;
        while(i<nd.Keys.Count && key>nd.Keys[i]) i++;
        if (i<nd.Keys.Count && nd.Keys[i]==key) return true;
        if (nd.Leaf) return false;
        return SearchNode(nd.Children[i], key);
    }

    public void Insert(int key)
    {
        if (_nodes[_root].Keys.Count==2*_order-1)
        {
            int newRoot=NewNode(false);
            _nodes[newRoot].Children.Add(_root);
            SplitChild(newRoot,0);
            _root=newRoot;
        }
        InsertNonFull(_root,key);
    }

    private void SplitChild(int parentId,int i)
    {
        int fullId=_nodes[parentId].Children[i];
        int newId=NewNode(_nodes[fullId].Leaf);
        _disk.Reads++; _disk.Writes+=2;

        var full=_nodes[fullId]; var nw=_nodes[newId];
        int mid=_order-1;
        int sep=full.Keys[mid];

        nw.Keys.AddRange(full.Keys.GetRange(mid+1,full.Keys.Count-mid-1));
        full.Keys.RemoveRange(mid,full.Keys.Count-mid);

        if (!full.Leaf)
        {
            nw.Children.AddRange(full.Children.GetRange(mid+1,full.Children.Count-mid-1));
            full.Children.RemoveRange(mid+1,full.Children.Count-mid-1);
        }
        _nodes[parentId].Keys.Insert(i,sep);
        _nodes[parentId].Children.Insert(i+1,newId);
    }

    private void InsertNonFull(int id,int key)
    {
        var nd=_nodes[id];
        _disk.Reads++; _disk.Writes++;
        if (nd.Leaf)
        {
            int pos=nd.Keys.BinarySearch(key);
            if (pos>=0) return; // duplicate
            pos=~pos;
            nd.Keys.Insert(pos,key);
        }
        else
        {
            int i=0;
            while(i<nd.Keys.Count && key>nd.Keys[i]) i++;
            if (i<nd.Keys.Count && nd.Keys[i]==key) return;
            if (_nodes[nd.Children[i]].Keys.Count==2*_order-1)
            {
                SplitChild(id,i);
                if (key>_nodes[id].Keys[i]) i++;
            }
            InsertNonFull(nd.Children[i],key);
        }
    }

    public List<int> ToSorted()
    {
        var outp=new List<int>();
        void Inorder(int id){
            if(id<0)return;
            var nd=_nodes[id];
            for(int i=0;i<nd.Keys.Count;i++){
                if(!nd.Leaf) Inorder(nd.Children[i]);
                outp.Add(nd.Keys[i]);
            }
            if(!nd.Leaf) Inorder(nd.Children[^1]);
        }
        Inorder(_root);
        return outp;
    }
}

// ================================================================
// Usage
// ================================================================
public static class Program
{
    public static void Main()
    {
        const int B=8, M=32;

        // External merge sort
        var rng=new Random(42);
        var arr=Enumerable.Range(0,100).Select(_=>rng.Next(1000)).ToList();
        var refSorted=new List<int>(arr); refSorted.Sort();

        var disk=new Disk(B);
        ExternalMergeSort.Sort(arr,disk,B,M);
        Console.WriteLine($"External Merge Sort: I/Os={disk.Reads+disk.Writes}  correct={arr.SequenceEqual(refSorted)}");

        // External B-tree
        disk=new Disk(B);
        var bt=new ExtBTree(disk,B);
        var vals=new List<int>();
        for(int i=0;i<50;i++){int v=rng.Next(200);bt.Insert(v);vals.Add(v);}
        vals=vals.Distinct().OrderBy(x=>x).ToList();
        var sorted=bt.ToSorted();
        Console.WriteLine($"External B-Tree: I/Os={disk.Reads+disk.Writes}  correct={sorted.SequenceEqual(vals)}");
    }
}
```

---

## I/O Cost Walkthrough — Concrete Example

```
N = 1,000,000,000 elements (8 bytes each = 8 GB)
M = 1,000,000,000 / 1000 = 1,000,000 elements fit in RAM (8 MB... let's use 128MB → ~16M elements)
B = 4096 bytes / 8 bytes = 512 elements per block (typical disk block)

External sort:
  N/B = 1,000,000,000 / 512 ≈ 1,953,125 blocks
  M/B = 16,000,000 / 512 ≈ 31,250
  log_{M/B}(N/B) = log_{31250}(1953125) ≈ 1.36 → 2 passes

  Total I/Os ≈ 2 * 1,953,125 ≈ 3.9M block transfers
  At ~0.1ms per I/O (SSD): ≈ 390 seconds ≈ 6.5 minutes

  Compare: if algorithm did N random I/Os (one per element):
  1,000,000,000 * 0.1ms = 100,000 seconds ≈ 27 hours

  The I/O-aware algorithm is ~400x faster purely from block-aware access patterns.
```

This illustrates why the I/O model — not the RAM/comparison model — is the right cost model for large data: sequential block access dominates random access by orders of magnitude.

---

## Pitfalls

- **Block size B vs memory M confusion** — `B` is the unit of transfer (disk block / page size, typically 4KB–1MB), while `M` is total RAM available to the algorithm. The fan-in for merge sort is `M/B - 1`, not `M` or `B` alone. Confusing these gives wildly wrong pass-count estimates.
- **Sequential vs random I/O** — the I/O model counts *block transfers*, but in practice sequential transfers are far cheaper than random ones (no seek time on HDD, better prefetching on SSD). An algorithm with the same I/O *count* but random access pattern can be 10-100x slower in wall-clock time. Always prefer algorithms with sequential access patterns (merge sort) over random access (naive B-tree on HDD).
- **2-way merge vs k-way merge** — the reference implementation uses 2-way merge for clarity, giving `O((N/B) log₂(N/M))` passes. Production external sort uses `k = M/B - 1` way merge, giving `O((N/B) log_{M/B}(N/B))` — far fewer passes when `M/B` is large (typically thousands).
- **B-tree node size must match disk block size** — choosing `2*ORDER-1` keys per node such that the node's serialized size equals exactly one disk block (e.g. 4KB) is critical. A node spanning multiple blocks defeats the `O(log_B N)` guarantee — each "level" would cost multiple I/Os.
- **Duplicate key handling** — the reference B-tree uses set semantics (skip duplicate keys on insert). For a multiset/index where duplicates are meaningful (e.g. non-unique database index), either allow duplicates explicitly (store a list of record pointers per key) or append a unique tiebreaker (e.g. row ID) to make all keys distinct.
- **In-memory simulation hides real I/O latency variance** — a simulated `Disk` class with counters is useful for asymptotic analysis but does not model real disk behavior: SSD wear leveling, OS page cache, RAID striping, and network-attached storage all introduce additional layers that the simple I/O model abstracts away. Benchmark on real hardware for production tuning.
- **Memory for buffer tree** — buffer trees require `Θ(M)` space per buffer, with `O(N/M)` nodes having buffers — total buffer space is `O(N)`, not `O(M)`. This is intentional (buffers hold pending operations) but must be accounted for in capacity planning.

---

## Conclusion

External Memory Algorithms are essential whenever **data exceeds RAM**:

- The I/O model (parameters `N`, `M`, `B`) replaces the RAM model's operation count with block-transfer count as the cost metric — the right model when disk I/O dominates CPU time by 10⁵–10⁶x.
- External merge sort achieves `O((N/B) log_{M/B}(N/B))` I/Os by maximizing merge fan-in to `k = M/B - 1`, turning what would be `O(log₂(N/M))` passes into far fewer `O(log_{M/B}(N/B))` passes.
- External B-trees achieve `O(log_B N)` search by matching node size to block size, giving fan-out `B` and height `O(log_B N)` — for `B=512` and `N=10⁹`, height is about 3, meaning 3 I/Os per search.
- Buffer trees push the amortized cost per operation down by a further factor of `B`, essential for batch-processing workloads with millions of operations.

**Key takeaway:** every external memory algorithm's goal is to replace `O(log₂ N)`-style bounds (optimal in the RAM model) with `O(log_B N)` or `O((N/B) log_{M/B}(N/B))` bounds — exploiting the block size `B` to do `B` times more useful work per I/O. The fan-out/fan-in parameter `B` (or `M/B`) appearing in the base of the logarithm is the signature of an I/O-optimal algorithm.
