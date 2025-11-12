# Red-Black Tree  
*Self-Balancing BST — O(log n) Guarantee with Color Magic*

---

## Origin & Motivation  

**Red-Black Tree** is a **self-balancing binary search tree** that guarantees **O(log n)** time for `insert`, `delete`, and `search`.

- **Origin**:  
  - Rudolf Bayer (1972) — invented as **"symmetric binary B-tree"**  
  - Renamed **"Red-Black Tree"** by Leo J. Guibas and Robert Sedgewick (1978)  
- **Motivation**:  
  - **AVL trees** are too strict (perfect balance) → frequent rotations  
  - **Red-Black** allows **relaxed balance** → fewer rotations, better **cache performance**  
  - **Guaranteed O(log n)** worst-case → safe for real-time systems

---

## Where It’s Used  

| Domain | Use Case |
|-------|---------|
| **C++ STL** | `std::map`, `std::set` |
| **Java** | `TreeMap`, `TreeSet` |
| **Linux Kernel** | `struct rb_tree` — scheduling, virtual memory |
| **Databases** | B+ tree variant inspiration |
| **File Systems** | Metadata indexing |
| **Compilers** | Symbol tables |

---

## Core Properties — The 5 Rules  

Every Red-Black Tree satisfies:

| Rule | Description |
|------|-----------|
| 1 | Every node is **red** or **black** |
| 2 | **Root** is **black** |
| 3 | All **leaves (NIL)** are **black** |
| 4 | **Red node** → both children are **black** (no two reds in a row) |
| 5 | Every path from node to descendant leaves has **same number of black nodes** |

> **Black height** (`bh`) is constant → tree height ≤ `2 × log₂(n+1)`

---

## Core Idea — Balance via Color & Rotations  

### Operations  
- **Search**: Standard BST → **O(log n)**  
- **Insert**: BST insert → **fix violations** via **recoloring + rotations**  
- **Delete**: BST delete → **fix violations** (more complex)

### Rotations  
```text
Left Rotate(x):     Right Rotate(y):
     x                y
    / \              / \
   a   y    →       x   c
      / \          / \
     b   c        a   b
```

## Insert Algorithm — Step by Step  

1. **Insert like BST** → new node is **red**  
2. **Fix violations**:  
   - If **parent is black** → **done**  
   - If **parent is red** → **Rule 4 violation** → fix via:  
     - **Recoloring**  
     - **Rotations** (left/right)  
     - **Case analysis** based on **uncle’s color**

---

## Insert Cases  

| **Case** | **Uncle** | **Action** |
|--------|---------|-----------|
| **Case 1** | **Red** | Recolor **parent**, **uncle**, **grandparent** → **recurse** |
| **Case 2** | **Black (triangle)** | **Rotate parent** → reduce to **Case 3** |
| **Case 3** | **Black (line)** | **Rotate grandparent** + **swap colors** |

> **Goal**: Push red-red violation **up** until root or resolved

---

## Delete Algorithm — High Level  

1. **Standard BST delete**  
2. If **deleted node was black** → **black height violation**  
3. **Fix** via:  
   - **Borrow black** from sibling  
   - **Rotations + recoloring**  
   - **6 symmetric cases** (left/right, sibling red/black)

> **Delete is harder** — but **still O(log n)**

---

## Full Implementation (C++)  

```cpp
#include <iostream>
using namespace std;

struct Node {
    int data;
    bool isRed;
    Node *left, *right, *parent;
    
    Node(int val) : data(val), isRed(true), left(nullptr), right(nullptr), parent(nullptr) {}
};

class RedBlackTree {
private:
    Node* root;
    Node* NIL;

    void leftRotate(Node* x) {
        Node* y = x->right;
        x->right = y->left;
        if (y->left != NIL) y->left->parent = x;
        y->parent = x->parent;
        if (x->parent == NIL) root = y;
        else if (x == x->parent->left) x->parent->left = y;
        else x->parent->right = y;
        y->left = x;
        x->parent = y;
    }

    void rightRotate(Node* y) {
        Node* x = y->left;
        y->left = x->right;
        if (x->right != NIL) x->right->parent = y;
        x->parent = y->parent;
        if (y->parent == NIL) root = x;
        else if (y == y->parent->right) y->parent->right = x;
        else y->parent->left = x;
        x->right = y;
        y->parent = x;
    }

    void insertFixup(Node* z) {
        while (z->parent && z->parent->isRed) {
            if (z->parent == z->parent->parent->left) {
                Node* uncle = z->parent->parent->right;
                
                // Case 1: Uncle is red
                if (uncle->isRed) {
                    z->parent->isRed = false;
                    uncle->isRed = false;
                    z->parent->parent->isRed = true;
                    z = z->parent->parent;
                } else {
                    // Case 2: Uncle black, z is right child (triangle)
                    if (z == z->parent->right) {
                        z = z->parent;
                        leftRotate(z);
                    }
                    // Case 3: Uncle black, z is left child (line)
                    z->parent->isRed = false;
                    z->parent->parent->isRed = true;
                    rightRotate(z->parent->parent);
                }
            } else {
                Node* uncle = z->parent->parent->left;
                
                if (uncle->isRed) {
                    z->parent->isRed = false;
                    uncle->isRed = false;
                    z->parent->parent->isRed = true;
                    z = z->parent->parent;
                } else {
                    if (z == z->parent->left) {
                        z = z->parent;
                        rightRotate(z);
                    }
                    z->parent->isRed = false;
                    z->parent->parent->isRed = true;
                    leftRotate(z->parent->parent);
                }
            }
        }
        root->isRed = false;  // Root always black
    }

    void transplant(Node* u, Node* v) {
        if (u->parent == NIL) root = v;
        else if (u == u->parent->left) u->parent->left = v;
        else u->parent->right = v;
        v->parent = u->parent;
    }

    Node* minimum(Node* node) {
        while (node->left != NIL) node = node->left;
        return node;
    }

    void deleteFixup(Node* x) {
        while (x != root && !x->isRed) {
            if (x == x->parent->left) {
                Node* w = x->parent->right;
                if (w->isRed) {
                    w->isRed = false;
                    x->parent->isRed = true;
                    leftRotate(x->parent);
                    w = x->parent->right;
                }
                if (!w->left->isRed && !w->right->isRed) {
                    w->isRed = true;
                    x = x->parent;
                } else {
                    if (!w->right->isRed) {
                        w->left->isRed = false;
                        w->isRed = true;
                        rightRotate(w);
                        w = x->parent->right;
                    }
                    w->isRed = x->parent->isRed;
                    x->parent->isRed = false;
                    w->right->isRed = false;
                    leftRotate(x->parent);
                    x = root;
                }
            } else {
                Node* w = x->parent->left;
                if (w->isRed) {
                    w->isRed = false;
                    x->parent->isRed = true;
                    rightRotate(x->parent);
                    w = x->parent->left;
                }
                if (!w->right->isRed && !w->left->isRed) {
                    w->isRed = true;
                    x = x->parent;
                } else {
                    if (!w->left->isRed) {
                        w->right->isRed = false;
                        w->isRed = true;
                        leftRotate(w);
                        w = x->parent->left;
                    }
                    w->isRed = x->parent->isRed;
                    x->parent->isRed = false;
                    w->left->isRed = false;
                    rightRotate(x->parent);
                    x = root;
                }
            }
        }
        x->isRed = false;
    }

public:
    RedBlackTree() {
        NIL = new Node(0);
        NIL->isRed = false;
        root = NIL;
    }

    void insert(int key) {
        Node* z = new Node(key);
        Node* y = NIL;
        Node* x = root;

        while (x != NIL) {
            y = x;
            if (z->data < x->data) x = x->left;
            else x = x->right;
        }

        z->parent = y;
        if (y == NIL) root = z;
        else if (z->data < y->data) y->left = z;
        else y->right = z;

        z->left = z->right = NIL;
        insertFixup(z);
    }

    void remove(int key) {
        Node* z = search(root, key);
        if (z == NIL) return;

        Node* y = z;
        Node* x;
        bool yOriginalColor = y->isRed;

        if (z->left == NIL) {
            x = z->right;
            transplant(z, z->right);
        } else if (z->right == NIL) {
            x = z->left;
            transplant(z, z->left);
        } else {
            y = minimum(z->right);
            yOriginalColor = y->isRed;
            x = y->right;
            if (y->parent == z) {
                x->parent = y;
            } else {
                transplant(y, y->right);
                y->right = z->right;
                y->right->parent = y;
            }
            transplant(z, y);
            y->left = z->left;
            y->left->parent = y;
            y->isRed = z->isRed;
        }

        if (!yOriginalColor) {
            deleteFixup(x);
        }
        delete z;
    }

    Node* search(Node* node, int key) {
        while (node != NIL && key != node->data) {
            if (key < node->data) node = node->left;
            else node = node->right;
        }
        return node;
    }

    void inorder() { inorderHelper(root); cout << endl; }
    void inorderHelper(Node* node) {
        if (node != NIL) {
            inorderHelper(node->left);
            cout << node->data << (node->isRed ? "(R) " : "(B) ");
            inorderHelper(node->right);
        }
    }
};
```

## Full Implementation (C#) 
```cpp
using System;

public class RedBlackTree
{
    private class Node
    {
        public int Data { get; set; }
        public bool IsRed { get; set; }
        public Node Left { get; set; }
        public Node Right { get; set; }
        public Node Parent { get; set; }

        public Node(int data)
        {
            Data = data;
            IsRed = true;
        }
    }

    private Node root;
    private readonly Node NIL;

    public RedBlackTree()
    {
        NIL = new Node(0) { IsRed = false };
        root = NIL;
    }

    private void LeftRotate(Node x)
    {
        Node y = x.Right;
        x.Right = y.Left;
        if (y.Left != NIL) y.Left.Parent = x;
        y.Parent = x.Parent;
        if (x.Parent == NIL) root = y;
        else if (x == x.Parent.Left) x.Parent.Left = y;
        else x.Parent.Right = y;
        y.Left = x;
        x.Parent = y;
    }

    private void RightRotate(Node y)
    {
        Node x = y.Left;
        y.Left = x.Right;
        if (x.Right != NIL) x.Right.Parent = y;
        x.Parent = y.Parent;
        if (y.Parent == NIL) root = x;
        else if (y == y.Parent.Right) y.Parent.Right = x;
        else y.Parent.Left = x;
        x.Right = y;
        y.Parent = x;
    }

    public void Insert(int key)
    {
        Node z = new Node(key);
        Node y = NIL;
        Node x = root;

        while (x != NIL)
        {
            y = x;
            x = key < x.Data ? x.Left : x.Right;
        }

        z.Parent = y;
        if (y == NIL) root = z;
        else if (key < y.Data) y.Left = z;
        else y.Right = z;

        z.Left = z.Right = NIL;
        InsertFixup(z);
    }

    private void InsertFixup(Node z)
    {
        while (z.Parent?.IsRed == true)
        {
            if (z.Parent == z.Parent.Parent.Left)
            {
                Node uncle = z.Parent.Parent.Right;
                if (uncle.IsRed)
                {
                    z.Parent.IsRed = false;
                    uncle.IsRed = false;
                    z.Parent.Parent.IsRed = true;
                    z = z.Parent.Parent;
                }
                else
                {
                    if (z == z.Parent.Right)
                    {
                        z = z.Parent;
                        LeftRotate(z);
                    }
                    z.Parent.IsRed = false;
                    z.Parent.Parent.IsRed = true;
                    RightRotate(z.Parent.Parent);
                }
            }
            else
            {
                Node uncle = z.Parent.Parent.Left;
                if (uncle.IsRed)
                {
                    z.Parent.IsRed = false;
                    uncle.IsRed = false;
                    z.Parent.Parent.IsRed = true;
                    z = z.Parent.Parent;
                }
                else
                {
                    if (z == z.Parent.Left)
                    {
                        z = z.Parent;
                        RightRotate(z);
                    }
                    z.Parent.IsRed = false;
                    z.Parent.Parent.IsRed = true;
                    LeftRotate(z.Parent.Parent);
                }
            }
        }
        root.IsRed = false;
    }

    public void Delete(int key)
    {
        Node z = Search(root, key);
        if (z == NIL) return;

        Node y = z;
        Node x;
        bool yOriginalColor = y.IsRed;

        if (z.Left == NIL)
        {
            x = z.Right;
            Transplant(z, z.Right);
        }
        else if (z.Right == NIL)
        {
            x = z.Left;
            Transplant(z, z.Left);
        }
        else
        {
            y = Minimum(z.Right);
            yOriginalColor = y.IsRed;
            x = y.Right;
            if (y.Parent == z)
                x.Parent = y;
            else
            {
                Transplant(y, y.Right);
                y.Right = z.Right;
                y.Right.Parent = y;
            }
            Transplant(z, y);
            y.Left = z.Left;
            y.Left.Parent = y;
            y.IsRed = z.IsRed;
        }

        if (!yOriginalColor)
            DeleteFixup(x);
    }

    private void DeleteFixup(Node x)
    {
        while (x != root && !x.IsRed)
        {
            if (x == x.Parent.Left)
            {
                Node w = x.Parent.Right;
                if (w.IsRed)
                {
                    w.IsRed = false;
                    x.Parent.IsRed = true;
                    LeftRotate(x.Parent);
                    w = x.Parent.Right;
                }
                if (!w.Left.IsRed && !w.Right.IsRed)
                {
                    w.IsRed = true;
                    x = x.Parent;
                }
                else
                {
                    if (!w.Right.IsRed)
                    {
                        w.Left.IsRed = false;
                        w.IsRed = true;
                        RightRotate(w);
                        w = x.Parent.Right;
                    }
                    w.IsRed = x.Parent.IsRed;
                    x.Parent.IsRed = false;
                    w.Right.IsRed = false;
                    LeftRotate(x.Parent);
                    x = root;
                }
            }
            else
            {
                Node w = x.Parent.Left;
                if (w.IsRed)
                {
                    w.IsRed = false;
                    x.Parent.IsRed = true;
                    RightRotate(x.Parent);
                    w = x.Parent.Left;
                }
                if (!w.Right.IsRed && !w.Left.IsRed)
                {
                    w.IsRed = true;
                    x = x.Parent;
                }
                else
                {
                    if (!w.Left.IsRed)
                    {
                        w.Right.IsRed = false;
                        w.IsRed = true;
                        LeftRotate(w);
                        w = x.Parent.Left;
                    }
                    w.IsRed = x.Parent.IsRed;
                    x.Parent.IsRed = false;
                    w.Left.IsRed = false;
                    RightRotate(x.Parent);
                    x = root;
                }
            }
        }
        x.IsRed = false;
    }

    private Node Minimum(Node node)
    {
        while (node.Left != NIL) node = node.Left;
        return node;
    }

    private void Transplant(Node u, Node v)
    {
        if (u.Parent == NIL) root = v;
        else if (u == u.Parent.Left) u.Parent.Left = v;
        else u.Parent.Right = v;
        v.Parent = u.Parent;
    }

    private Node Search(Node node, int key)
    {
        while (node != NIL && key != node.Data)
            node = key < node.Data ? node.Left : node.Right;
        return node;
    }

    public void Inorder() { InorderHelper(root); Console.WriteLine(); }
    private void InorderHelper(Node node)
    {
        if (node != NIL)
        {
            InorderHelper(node.Left);
            Console.Write($"{node.Data}{(node.IsRed ? "(R)" : "(B)")} ");
            InorderHelper(node.Right);
        }
    }
}
```


## Complexity  

| **Operation** | **Time** | **Space** |
|---------------|----------|-----------|
| **Search** | **O(log n)** | O(1) |
| **Insert** | **O(log n)** | O(1) |
| **Delete** | **O(log n)** | O(1) |
| **Space** | **O(n)** | One node per element |

> **Amortized rotations**: ~1 per insert/delete  
> **Worst-case height**: ≤ `2 × log₂(n+1)` → **O(log n)** guaranteed

---

## Pitfalls & Fixes  

| **Issue** | **Fix** |
|---------|--------|
| **Double red (Rule 4 violation)** | Always fix via **recolor** → **rotate** |
| **Root not black** | Force `root.isRed = false` after `insertFixup` |
| **NIL handling** | Use **sentinel node** (always black) |
| **Parent pointers** | **Required** for rotations and fixup |
| **Delete complexity** | Use **transplant** + **fixup pattern** |

---

## Red-Black vs AVL  

| **Feature** | **Red-Black** | **AVL** |
|-----------|---------------|--------|
| **Balance** | Relaxed | Strict |
| **Insert/Delete** | **Faster** (fewer rotations) | Slower |
| **Search** | Slightly slower | **Faster** |
| **Memory** | `+parent` pointer | `+balance factor` |
| **Real-world** | **Preferred** (`std::map`, Java) | Rare |

> **Red-Black wins in practice**: fewer cache misses, better constants

---

## Insight — Reusable Fichka  

> **Self-Balancing BST with Color-Based Balance**

### Pattern  
- **Standard BST** + **5 color rules**  
- **Rotations** to fix **red-red violations**  
- **O(log n) worst-case** for all operations

### Applies to  
- `std::map` / `std::set`  
- **Ordered statistics**  
- **Interval trees**  
- Any **dynamic ordered set**

---

**Fichka**:  
> When you need **ordered operations** with **worst-case O(log n)**:  
> **Red-Black Tree = balance + simplicity + performance**

---

## Algorithm Deep Dive — How It Works  

### **Insert — The Color Dance**  
1. **Insert as BST** → new node = **red**  
2. **Check Rule 4**: if parent is red → **conflict**  
3. **Uncle check**:  
   - **Uncle red** → **recolor** (push conflict up)  
   - **Uncle black** → **rotate** + **swap colors**  
4. **Repeat** until root or resolved  
5. **Root always black**

> **Key insight**: **Red nodes are "temporary"** — they help balance without strict height

---

### **Delete — The Black Hole Fix**  
1. **Standard BST delete**  
2. If **removed node was black** → **missing black height**  
3. **Sibling borrow**:  
   - **Sibling red** → rotate → reduce to black sibling  
   - **Sibling black, both children black** → **recolor sibling red** → push deficit up  
   - **Sibling has red child** → **rotate + recolor** to restore balance  
4. **6 symmetric cases** → but **pattern is consistent**

> **Delete is harder** — but **still O(log n)** via **double black fixup**

---

**Final Verdict**:  
**O(log n). Self-balancing. Industry standard. Color magic.**


---



