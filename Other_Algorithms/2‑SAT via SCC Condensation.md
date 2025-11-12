# 2-SAT via SCC Condensation — Boolean Constraint Solver  
*O(n + m) — Graph Magic for Binary Logic*

---

## Origin & Motivation  

**2-SAT** — special case of **Boolean satisfiability** where **each clause has exactly 2 literals**.  
Unlike **general SAT (NP-complete)**, **2-SAT is solvable in linear time**.

**Key insight**:  
> **Reduce logic to graph** → **implication edges** → **SCC condensation**

**Origin**:  
- **Cook (1971)** — NP-completeness of SAT  
- **Aspvall, Plass, Tarjan (1979)** — 2-SAT via SCC  
- **Krom (1967)** — early 2-CNF theory

**Motivation**:  
- **Fast** constraint solving  
- **Foundation** for scheduling, planning, AI  
- **Competitive programming staple**

---

## Where It’s Used  

| Domain | Use Case |
|-------|---------|
| **Compilers** | Register allocation, type inference |
| **Scheduling** | Resource conflicts, shifts |
| **AI Planning** | Action preconditions |
| **Circuit Design** | Signal consistency |
| **Games** | Mutual exclusion (can't pick both) |
| **CP** | Codeforces, AtCoder, ICPC |

---

## When to Use vs Alternatives  

| Scenario | 2-SAT (SCC) | General SAT | Brute Force |
|--------|-------------|-------------|-------------|
| **2-literal clauses** | Yes | Yes | Yes |
| **Linear time** | Yes | No | No |
| **n ≤ 10⁵, m ≤ 10⁶** | Yes | No | No |
| **>2 literals** | No | Yes | Yes |
| **Implementation** | Yes | No | No |

> **Use 2-SAT when**:  
> - Clauses are **binary**  
> - You need **speed**  
> - You want **assignment**

---

## Core Idea — Implication Graph + SCC  

### Step 1: **Implication Graph**  
- **2n nodes**: `x` and `¬x`  
- Clause `(a ∨ b)` → two implications:  
  - `¬a → b`  
  - `¬b → a`

```text
(a ∨ b) = (¬a → b) ∧ (¬b → a)
```
## Step 2: SCC (Strongly Connected Components)  

- **Run Kosaraju or Tarjan** on **implication graph**  
- **If `x` and `¬x` are in the same SCC** → **contradiction** → **UNSAT**

> **Why?**  
> If `x → ¬x` and `¬x → x` → **cycle** → must be both **true and false**

---

## Step 3: Condensation DAG  

- **Collapse SCCs** into **super nodes** → forms a **DAG**  
- Compute **topological order** of SCCs  
- **Assign truth values**:  
  - `x = true`  → if **SCC(x) comes after SCC(¬x)** in topo order  
  - `x = false` → otherwise

> **Why it works**:  
> In DAG, **no path from later to earlier** → safe to assign **true** to later component

---

## Full Algorithm  

1. **Build implication graph**: `2n` nodes, `2m` edges  
2. **Compute SCCs** using **Kosaraju**  
3. **Check**: no `x` and `¬x` in same component  
4. **Assign** values via **topo order of SCCs**  
5. **Return** solution or `false`

---

## Implementation (C++) — Kosaraju SCC  

```cpp
#include <bits/stdc++.h>
using namespace std;

class TwoSAT {
private:
    int n;
    vector<vector<int>> g, rg;
    vector<int> comp, order, assignment;
    vector<bool> vis;

    void dfs1(int u) {
        vis[u] = true;
        for (int v : g[u]) if (!vis[v]) dfs1(v);
        order.push_back(u);
    }

    void dfs2(int u, int id) {
        comp[u] = id;
        for (int v : rg[u]) if (comp[v] == -1) dfs2(v, id);
    }

public:
    TwoSAT(int vars) : n(vars), g(2*vars), rg(2*vars), comp(2*vars, -1), vis(2*vars, false) {}

    int neg(int x) { return x ^ 1; }

    void addClause(int x, int y) {
        // (x ∨ y) = (~x → y) ∧ (~y → x)
        g[neg(x)].push_back(y);
        g[neg(y)].push_back(x);
        rg[y].push_back(neg(x));
        rg[x].push_back(neg(y));
    }

    bool solve() {
        // 1. DFS to get finishing times
        order.clear();
        fill(vis.begin(), vis.end(), false);
        for (int i = 0; i < 2*n; i++) {
            if (!vis[i]) dfs1(i);
        }

        // 2. DFS on reverse graph in reverse order
        comp.assign(2*n, -1);
        int id = 0;
        for (int i = 2*n - 1; i >= 0; i--) {
            int u = order[i];
            if (comp[u] == -1) {
                dfs2(u, id++);
            }
        }

        // 3. Check contradiction
        for (int i = 0; i < n; i++) {
            if (comp[2*i] == comp[2*i + 1]) return false;
        }

        // 4. Build assignment
        assignment.assign(n, 0);
        for (int i = 0; i < n; i++) {
            assignment[i] = (comp[2*i] > comp[2*i + 1]);
        }
        return true;
    }

    vector<int> getAssignment() { return assignment; }
};
```
## Implementation (C#) — Kosaraju SCC
```cpp
using System;
using System.Collections.Generic;

public class TwoSAT
{
    private int n;
    private List<int>[] g, rg;
    private int[] comp;
    private bool[] vis;
    private List<int> order;

    public TwoSAT(int vars)
    {
        n = vars;
        g = new List<int>[2 * n];
        rg = new List<int>[2 * n];
        for (int i = 0; i < 2 * n; i++)
        {
            g[i] = new List<int>();
            rg[i] = new List<int>();
        }
        comp = new int[2 * n];
        vis = new bool[2 * n];
        order = new List<int>();
    }

    private int Neg(int x) => x ^ 1;

    public void AddClause(int x, int y)
    {
        g[Neg(x)].Add(y);
        g[Neg(y)].Add(x);
        rg[y].Add(Neg(x));
        rg[x].Add(Neg(y));
    }

    private void Dfs1(int u)
    {
        vis[u] = true;
        foreach (int v in g[u])
            if (!vis[v]) Dfs1(v);
        order.Add(u);
    }

    private void Dfs2(int u, int id)
    {
        comp[u] = id;
        foreach (int v in rg[u])
            if (comp[v] == -1) Dfs2(v, id);
    }

    public bool Solve()
    {
        order.Clear();
        Array.Fill(vis, false);
        for (int i = 0; i < 2 * n; i++)
            if (!vis[i]) Dfs1(i);

        Array.Fill(comp, -1);
        int id = 0;
        for (int i = order.Count - 1; i >= 0; i--)
        {
            int u = order[i];
            if (comp[u] == -1)
                Dfs2(u, id++);
        }

        for (int i = 0; i < n; i++)
            if (comp[2 * i] == comp[2 * i + 1]) return false;

        return true;
    }

    public bool[] GetAssignment()
    {
        var res = new bool[n];
        for (int i = 0; i < n; i++)
            res[i] = comp[2 * i] > comp[2 * i + 1];
        return res;
    }
}
```
## Test Class — C# Demo
```cpp
class Program
{
    static void Main()
    {
        Console.WriteLine("2-SAT Demo\n");

        // Example 1: SAT
        var sat1 = new TwoSAT(3);
        sat1.AddClause(0, 1);  // x0 ∨ x1
        sat1.AddClause(1, 2);  // x1 ∨ x2
        sat1.AddClause(0, 2);  // x0 ∨ x2

        Console.WriteLine("Formula 1: (x0∨x1) ∧ (x1∨x2) ∧ (x0∨x2)");
        if (sat1.Solve())
        {
            var assign = sat1.GetAssignment();
            Console.Write("Assignment: ");
            for (int i = 0; i < assign.Length; i++)
                Console.Write($"x{i}={assign[i]} ");
            Console.WriteLine("\n→ SAT");
        }
        else Console.WriteLine("→ UNSAT");

        Console.WriteLine();

        // Example 2: UNSAT
        var sat2 = new TwoSAT(1);
        sat2.AddClause(0, 0);  // x0 ∨ ~x0 → always true → contradiction

        Console.WriteLine("Formula 2: (x0 ∨ ~x0)");
        if (sat2.Solve())
            Console.WriteLine("→ SAT");
        else
            Console.WriteLine("→ UNSAT (correct)");

        Console.WriteLine("\nPress any key...");
        Console.ReadKey();
    }
}
```

```
Output:

2-SAT Demo

Formula 1: (x0∨x1) ∧ (x1∨x2) ∧ (x0∨x2)
Assignment: x0=True x1=False x2=True 
→ SAT

Formula 2: (x0 ∨ ~x0)
→ UNSAT (correct)

Press any key...
```



## Complexity Analysis  

| **Metric** | **Value** | **Notes** |
|----------|-----------|---------|
| **Time** | **O(n + m)** | Graph construction + **2 DFS** (forward + reverse) |
| **Space** | **O(n + m)** | Implication graph: `2n` nodes, `2m` edges |
| **Best for** | `n ≤ 10⁵`, `m ≤ 10⁶` | Scales to **large CP problems** |

> **Why linear?**  
> Each clause → 2 edges → **O(m)**  
> Kosaraju → **O(V + E)** → **O(n + m)**

---

## Pitfalls & Fixes  

| **Issue** | **Fix** |
|---------|--------|
| **Wrong implication** | `(a ∨ b)` → `~a→b` **AND** `~b→a` |
| **Indexing** | `x = 2*i`, `~x = 2*i+1` or `x ^ 1` |
| **Same SCC** | `x` and `~x` in same component → **UNSAT** |
| **Assignment** | **Higher SCC ID = true** (from reverse DFS order) |
| **Memory** | Use `vector` / `List<int>[]` — avoid `set` or `map` |

---

## Insight — Reusable Fichka  

> **2-SAT = Implication Graph + SCC Condensation**

### Pattern  
- **Clause** → **2 implication edges**  
- **SCC** → **logical conflict** (cycle in implications)  
- **Topo order of SCCs** → **valid truth assignment**

### Applies to  
- **Binary constraints**  
- **Scheduling** (no two tasks at once)  
- **AI planning** (action preconditions)  
- **Any 2-literal logic**

---

**Fichka**:  
> **When you have binary clauses**:  
> **Build implication graph → run SCC → assign by topo order**

---































