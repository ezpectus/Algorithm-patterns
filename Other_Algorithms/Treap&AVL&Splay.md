# Treap / AVL / Splay — Self-Balancing BST Trio  
*Guaranteed O(log n) — Randomized, Rigid, Adaptive*

---

## Origin & Motivation  
All three structures solve **one core problem**:  
> **Keep a Binary Search Tree (BST) balanced → O(log n) operations**

Without balance → **O(n)** worst case → **linked list**

**Solution**: **Self-balancing BSTs**

---

### **Treap (Tree + Heap)**  
- **Invented**: **1989** by **R. Seidel & C. Aragon**  
- **Idea**:  
  - **Key** → BST order  
  - **Random Priority** → **Max-Heap** order  
- **Balance**: **Randomness** → expected O(log n)

> **“Let probability do the balancing”**

---

### **AVL Tree**  
- **Invented**: **1962** by **Adelson-Velsky & Landis**  
- **First self-balancing BST**  
- **Rule**:  
  - **Height difference ≤ 1**  
  - Store **height** in node  
- **Balance**: **Rotations** on violation

> **“Enforce balance strictly — no exceptions”**

---

### **Splay Tree**  
- **Invented**: **1985** by **Sleator & Tarjan**  
- **Idea**:  
  - **After access → splay node to root**  
  - **Frequently used nodes → near root**  
- **Balance**: **Amortized O(log n)**

> **“Adapt to usage — hot data rises”**

---

## Where They’re Used  

| Domain | Treap | AVL | Splay |
|-------|-------|-----|-------|
| **Competitive Programming** | Yes | Yes | No (rare) |
| **Databases / Indexes** | No | Yes | Yes (caching) |
| **Randomized Algorithms** | Yes | No | No |
| **Memory-Constrained Systems** | Yes | No | Yes |
| **Adaptive Access Patterns** | No | No | Yes |

---

## When to Use  

| Requirement | Treap | AVL | Splay |
|-----------|-------|-----|-------|
| **Simple implementation** | Yes | No | No |
| **Guaranteed height** | **Probabilistic** | **Strict** | **Amortized** |
| **Fast insert/delete** | Yes | No | No |
| **Frequent searches** | Yes | Yes | **Adaptive** |
| **Caching / locality** | No | No | Yes |

---

## Core Idea — Deep Dive  

---

### **Treap — Random Heap on BST**

```text
          50 (p=10)
         /  \
      30(p=20) 70(p=5)
```

- **BST**: `left < node < right`  
- **Heap**: `parent.priority > child.priority`  
- **Insert**: insert as BST → **rotate up by priority**  
- **Delete**: **rotate down** → remove leaf

### **AVL — Strict Height Control**  

```text
Balance factor = h_left - h_right
|balance| ≤ 1
```

## AVL Rotations — 4 Cases  
Every **insert/delete** may unbalance the tree → requires **rebalance along the path**.  

| Case | Fix | Rotation |
|------|-----|----------|
| **LL** | Right rotate | Single rotation |
| **RR** | Left rotate | Single rotation |
| **LR** | Left → Right | Double rotation |
| **RL** | Right → Left | Double rotation |

**Summary:**  
- LL → single right rotation  
- RR → single left rotation  
- LR → left rotation on child, then right rotation on parent  
- RL → right rotation on child, then left rotation on parent

---

## Splay Tree — Access = Move to Root  
Accessing a node triggers **splay(x)** → node becomes **root**.  

Splay steps:

- Zig (parent of root)
- Zig-Zig (same direction)
- Zig-Zag (opposite direction)
- Amortized O(log n) → even if one splay is O(n)

## Full Implementation (C++) 
```cpp
#include <bits/stdc++.h>
using namespace std;

// ===================== TREAP =====================
struct Treap {
    int key, priority;
    Treap *left, *right;
    Treap(int k) : key(k), priority(rand()), left(nullptr), right(nullptr) {}
};

class TreapTree {
    Treap* root = nullptr;

    pair<Treap*, Treap*> split(Treap* node, int key) {
        if (!node) return {nullptr, nullptr};
        if (node->key <= key) {
            auto [l, r] = split(node->right, key);
            node->right = l;
            return {node, r};
        } else {
            auto [l, r] = split(node->left, key);
            node->left = r;
            return {l, node};
        }
    }

    Treap* merge(Treap* l, Treap* r) {
        if (!l || !r) return l ? l : r;
        if (l->priority > r->priority) {
            l->right = merge(l->right, r);
            return l;
        } else {
            r->left = merge(l, r->left);
            return r;
        }
    }

public:
    void insert(int key) {
        auto [l, r] = split(root, key);
        root = merge(merge(l, new Treap(key)), r);
    }

    void erase(int key) {
        auto [l, r] = split(root, key);
        auto [ll, lr] = split(l, key - 1);
        root = merge(ll, r);
    }

    bool find(int key) {
        auto [l, r] = split(root, key);
        auto [ll, lr] = split(l, key - 1);
        bool exists = lr != nullptr;
        root = merge(merge(ll, lr), r);
        return exists;
    }
};

// ===================== AVL =====================
struct AVL {
    int key, height;
    AVL *left, *right;
    AVL(int k) : key(k), height(1), left(nullptr), right(nullptr) {}
};

class AVLTree {
    AVL* root = nullptr;

    int height(AVL* node) { return node ? node->height : 0; }
    int balance(AVL* node) { return height(node->left) - height(node->right); }

    void updateHeight(AVL* node) {
        node->height = max(height(node->left), height(node->right)) + 1;
    }

    AVL* rotateRight(AVL* y) {
        AVL* x = y->left;
        y->left = x->right;
        x->right = y;
        updateHeight(y);
        updateHeight(x);
        return x;
    }

    AVL* rotateLeft(AVL* x) {
        AVL* y = x->right;
        x->right = y->left;
        y->left = x;
        updateHeight(x);
        updateHeight(y);
        return y;
    }

    AVL* balanceNode(AVL* node) {
        updateHeight(node);
        int bal = balance(node);
        if (bal > 1) {
            if (balance(node->left) < 0)
                node->left = rotateLeft(node->left);
            return rotateRight(node);
        }
        if (bal < -1) {
            if (balance(node->right) > 0)
                node->right = rotateRight(node->right);
            return rotateLeft(node);
        }
        return node;
    }

    AVL* insert(AVL* node, int key) {
        if (!node) return new AVL(key);
        if (key < node->key)
            node->left = insert(node->left, key);
        else
            node->right = insert(node->right, key);
        return balanceNode(node);
    }

public:
    void insert(int key) { root = insert(root, key); }
};

// ===================== SPLAY =====================
struct Splay {
    int key;
    Splay *left, *right, *parent;
    Splay(int k) : key(k), left(nullptr), right(nullptr), parent(nullptr) {}
};

class SplayTree {
    Splay* root = nullptr;

    void rotate(Splay* x) {
        Splay* p = x->parent;
        if (!p) return;
        Splay* g = p->parent;
        if (p->left == x) {
            p->left = x->right;
            if (x->right) x->right->parent = p;
            x->right = p;
        } else {
            p->right = x->left;
            if (x->left) x->left->parent = p;
            x->left = p;
        }
        x->parent = g;
        p->parent = x;
        if (g) {
            if (g->left == p) g->left = x;
            else g->right = x;
        } else {
            root = x;
        }
    }

    void splay(Splay* x) {
        while (x->parent) {
            Splay* p = x->parent;
            Splay* g = p->parent;
            if (g) {
                if ((p->left == x) == (g->left == p))
                    rotate(p);
                else
                    rotate(x);
            }
            rotate(x);
        }
    }

public:
    void insert(int key) {
        if (!root) {
            root = new Splay(key);
            return;
        }
        Splay* node = root;
        while (true) {
            if (key < node->key) {
                if (!node->left) {
                    node->left = new Splay(key);
                    node->left->parent = node;
                    splay(node->left);
                    return;
                }
                node = node->left;
            } else {
                if (!node->right) {
                    node->right = new Splay(key);
                    node->right->parent = node;
                    splay(node->right);
                    return;
                }
                node = node->right;
            }
        }
    }
};
```

## Full Implementation (C#)
```cpp
using System;

// ===================== TREAP =====================
public class TreapNode {
    public int Key, Priority;
    public TreapNode Left, Right;
    public TreapNode(int k) { Key = k; Priority = new Random().Next(); }
}

public class Treap {
    private TreapNode root;

    private (TreapNode, TreapNode) Split(TreapNode node, int key) {
        if (node == null) return (null, null);
        if (node.Key <= key) {
            var (l, r) = Split(node.Right, key);
            node.Right = l;
            return (node, r);
        } else {
            var (l, r) = Split(node.Left, key);
            node.Left = r;
            return (l, node);
        }
    }

    private TreapNode Merge(TreapNode l, TreapNode r) {
        if (l == null || r == null) return l ?? r;
        if (l.Priority > r.Priority) {
            l.Right = Merge(l.Right, r);
            return l;
        } else {
            r.Left = Merge(l, r.Left);
            return r;
        }
    }

    public void Insert(int key) {
        var (l, r) = Split(root, key);
        root = Merge(Merge(l, new TreapNode(key)), r);
    }
}

// ===================== AVL =====================
public class AVLNode {
    public int Key, Height;
    public AVLNode Left, Right;
    public AVLNode(int k) { Key = k; Height = 1; }
}

public class AVLTree {
    private AVLNode root;

    private int Height(AVLNode node) => node?.Height ?? 0;
    private int Balance(AVLNode node) => Height(node.Left) - Height(node.Right);

    private void UpdateHeight(AVLNode node) {
        node.Height = Math.Max(Height(node.Left), Height(node.Right)) + 1;
    }

    private AVLNode RotateRight(AVLNode y) {
        var x = y.Left;
        y.Left = x.Right;
        x.Right = y;
        UpdateHeight(y);
        UpdateHeight(x);
        return x;
    }

    private AVLNode RotateLeft(AVLNode x) {
        var y = x.Right;
        x.Right = y.Left;
        y.Left = x;
        UpdateHeight(x);
        UpdateHeight(y);
        return y;
    }

    private AVLNode Balance(AVLNode node) {
        UpdateHeight(node);
        int bal = Balance(node);
        if (bal > 1) {
            if (Balance(node.Left) < 0) node.Left = RotateLeft(node.Left);
            return RotateRight(node);
        }
        if (bal < -1) {
            if (Balance(node.Right) > 0) node.Right = RotateRight(node.Right);
            return RotateLeft(node);
        }
        return node;
    }

    private AVLNode Insert(AVLNode node, int key) {
        if (node == null) return new AVLNode(key);
        if (key < node.Key) node.Left = Insert(node.Left, key);
        else node.Right = Insert(node.Right, key);
        return Balance(node);
    }

    public void Insert(int key) { root = Insert(root, key); }
}

// ===================== SPLAY =====================
public class SplayNode {
    public int Key;
    public SplayNode Left, Right, Parent;
    public SplayNode(int k) { Key = k; }
}

public class SplayTree {
    private SplayNode root;

    private void Rotate(SplayNode x) {
        var p = x.Parent;
        if (p == null) return;
        var g = p.Parent;
        if (p.Left == x) {
            p.Left = x.Right;
            if (x.Right != null) x.Right.Parent = p;
            x.Right = p;
        } else {
            p.Right = x.Left;
            if (x.Left != null) x.Left.Parent = p;
            x.Left = p;
        }
        x.Parent = g;
        p.Parent = x;
        if (g != null) {
            if (g.Left == p) g.Left = x;
            else g.Right = x;
        } else {
            root = x;
        }
    }

    private void Splay(SplayNode x) {
        while (x.Parent != null) {
            var p = x.Parent;
            var g = p.Parent;
            if (g != null) {
                if ((p.Left == x) == (g.Left == p)) Rotate(p);
                else Rotate(x);
            }
            Rotate(x);
        }
    }

    public void Insert(int key) {
        if (root == null) { root = new SplayNode(key); return; }
        var node = root;
        while (true) {
            if (key < node.Key) {
                if (node.Left == null) {
                    node.Left = new SplayNode(key) { Parent = node };
                    Splay(node.Left);
                    return;
                }
                node = node.Left;
            } else {
                if (node.Right == null) {
                    node.Right = new SplayNode(key) { Parent = node };
                    Splay(node.Right);
                    return;
                }
                node = node.Right;
            }
        }
    }
}
```

## Complexity Analysis  

| Operation | **Treap** | **AVL** | **Splay** |
|----------|-----------|---------|-----------|
| **Insert** | **O(log n)** *expected* | **O(log n)** *worst-case* | **O(log n)** *amortized* |
| **Delete** | **O(log n)** *expected* | **O(log n)** *worst-case* | **O(log n)** *amortized* |
| **Search** | **O(log n)** *expected* | **O(log n)** *worst-case* | **O(log n)** *amortized* |
| **Space** | **O(n)** | **O(n)** | **O(n)** |

> **Treap**: *Probabilistic* — relies on random priorities  
> **AVL**: *Strict* — height diff ≤ 1 always  
> **Splay**: *Amortized* — single op may be O(n), but sequence is O(log n)

---

## Pitfalls & Fixes  

| Tree | **Issue** | **Fix / Mitigation** |
|------|-----------|-----------------------|
| **Treap** | **Rare O(n) degradation** (bad RNG) | `srand(time(NULL))` + use **high-quality RNG** (`mt19937`) |
| **AVL** | **Too many rotations** on bulk inserts | Use **Red-Black Tree** for bulk operations |
| **Splay** | **Single operation O(n)** | **Accept amortized** — real-world access patterns make it fast |

---

## Conclusion  
**Treap / AVL / Splay — three philosophies of balance:**

- **Treap** → **Simple + Random = Reliable**  
  > Minimal code, no rotations, expected O(log n)  
- **AVL** → **Strict + Rigid = Guaranteed**  
  > Ironclad O(log n) worst-case, height-controlled  
- **Splay** → **Adaptive + Smart = Fast on real data**  
  > Frequently accessed nodes rise to root

**Key takeaway:**  
> **Pick your fighter:**  
> - Need **simplicity & speed in CP**? → **Treap**  
> - Need **guaranteed O(log n)**? → **AVL**  
> - Need **adaptive caching / locality**? → **Splay**

---

## Fichka Library Entry  
**Category:** `Data Structures / BST`  
**Pattern:** `Self-balancing binary search trees (randomized / strict / adaptive)`  
**Tags:** `treap`, `avl`, `splay`, `balanced`, `rotations`, `logarithmic`, `amortized`, `split-merge`


---
