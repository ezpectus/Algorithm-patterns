# SAT — Boolean Satisfiability

## Origin & Motivation

Boolean Satisfiability (SAT) was the first problem proven NP-complete by Cook in 1971. Given a propositional formula — typically in **Conjunctive Normal Form (CNF)**: a conjunction of clauses, each clause a disjunction of literals — the question is whether there exists an assignment of boolean variables that makes the entire formula true.

Despite NP-completeness, modern SAT solvers based on the **DPLL** algorithm (Davis–Putnam–Logemann–Loveland, 1962) extended with **CDCL** (Conflict-Driven Clause Learning, 1990s) solve industrial instances with millions of variables in seconds. The key insight: most real instances have structure that allows massive pruning of the search space.

Complexity: **O(2^n)** worst case. In practice CDCL solvers run in polynomial time on structured instances.

---

## Where It Is Used

- Hardware and software verification (model checking)
- Automated planning and scheduling
- Cryptanalysis and equivalence checking
- AI (constraint satisfaction, configuration)
- Competitive programming (reduction target for NP-hard problems)
- Package dependency resolution (APT, Cargo use SAT backends)

---

## Problem Definition

**Input:** A CNF formula `F = C_1 ∧ C_2 ∧ ... ∧ C_m`  
Each clause `C_i = (l_i1 ∨ l_i2 ∨ ... ∨ l_ik)`, each literal `l` is either variable `x` or its negation `¬x`.

**Output:** An assignment `{x_i → true/false}` satisfying F, or UNSAT.

**Example:**
```
Variables: x1, x2, x3
Formula:   (x1 ∨ ¬x2) ∧ (¬x1 ∨ x3) ∧ (x2 ∨ ¬x3)
```

---

## Core Algorithms

### DPLL (Davis–Putnam–Logemann–Loveland)

Recursive backtracking search with two key propagation rules:

**Unit propagation:** If a clause has exactly one unassigned literal, that literal must be true. Assign it and propagate.

**Pure literal elimination:** If a variable appears only positively (or only negatively) across all clauses, assign it to satisfy all those clauses.

```
DPLL(F):
    F = unit_propagate(F)
    if F contains empty clause: return UNSAT
    if F is empty: return SAT
    l = choose_literal(F)          // branching heuristic
    if DPLL(F ∧ {l}) == SAT: return SAT
    return DPLL(F ∧ {¬l})
```

### CDCL (Conflict-Driven Clause Learning)

Extends DPLL with:
- **Implication graph** tracking which assignments caused which propagations
- **Conflict analysis** — when a conflict occurs, analyze the implication graph to derive a new **learned clause** that prevents the same conflict
- **Non-chronological backjumping** — jump back to the earliest decision level implicated in the conflict, not just one level
- **VSIDS heuristic** — variable activity scores updated on conflict; branch on highest-activity variable

These extensions transform worst-case exponential DPLL into a solver that handles industrial instances with 10^6+ variables.

---

## Complexity Analysis

| Algorithm | Time (worst case) | Space | Notes |
|---|---|---|---|
| Naive enumeration | O(2^n) | O(n) | No pruning |
| DPLL | O(2^n) | O(n * m) | Unit propagation prunes heavily |
| DPLL + pure literal | O(2^n) | O(n * m) | Minor practical gain |
| CDCL | O(2^n) | O(n * m + learned) | Practical: near-polynomial on structured instances |

- `n` — number of variables
- `m` — number of clauses
- Learned clauses grow the formula but each learned clause cuts a subtree of the search space permanently

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// CNF formula in DIMACS-style:
// Variables numbered 1..n. Literal l: positive = var l, negative = var -l.
// Clause = vector of literals. Formula = vector of clauses.

struct SAT {
    int n;                         // number of variables
    vector<vector<int>> clauses;   // CNF formula
    vector<int> assign;            // 0=unset, 1=true, -1=false
    vector<int> trail;             // assignment stack
    // For each literal: list of clause indices it appears in
    vector<vector<int>> watch;     // watch[lit_id] = clause indices

    // Literal encoding: var v (1-indexed) -> positive lit = 2*v, negative = 2*v+1
    int pos(int v) { return 2 * v; }
    int neg(int v) { return 2 * v + 1; }
    int lit(int l) { return l > 0 ? pos(l) : neg(-l); }
    int var_of(int lid) { return lid / 2; }
    bool is_neg(int lid) { return lid & 1; }

    SAT(int n, const vector<vector<int>>& cls) : n(n), clauses(cls),
        assign(n + 1, 0), watch(2 * (n + 1)) {
        for (int i = 0; i < (int)clauses.size(); i++)
            for (int l : clauses[i])
                watch[lit(l)].push_back(i);
    }

    // Returns true if literal l is satisfied under current assignment
    bool lit_true(int l) {
        int v = abs(l);
        return (l > 0 && assign[v] == 1) || (l < 0 && assign[v] == -1);
    }

    // Returns true if literal l is falsified
    bool lit_false(int l) {
        int v = abs(l);
        return (l > 0 && assign[v] == -1) || (l < 0 && assign[v] == 1);
    }

    // Unit propagation: returns false on conflict
    bool propagate() {
        bool changed = true;
        while (changed) {
            changed = false;
            for (auto& clause : clauses) {
                int unset = 0, unset_lit = 0;
                bool satisfied = false;
                for (int l : clause) {
                    if (lit_true(l))  { satisfied = true; break; }
                    if (!lit_false(l)) { unset++; unset_lit = l; }
                }
                if (satisfied) continue;
                if (unset == 0) return false;   // empty clause = conflict
                if (unset == 1) {
                    // Unit clause: force unset_lit = true
                    int v = abs(unset_lit);
                    assign[v] = (unset_lit > 0) ? 1 : -1;
                    trail.push_back(v);
                    changed = true;
                }
            }
        }
        return true;
    }

    // Choose next unassigned variable (first unset — replace with VSIDS for speed)
    int choose() {
        for (int v = 1; v <= n; v++)
            if (assign[v] == 0) return v;
        return -1;
    }

    // DPLL recursive search
    bool dpll() {
        if (!propagate()) return false;

        int v = choose();
        if (v == -1) return true;   // all variables assigned

        // Save trail position for backtracking
        int saved = trail.size();

        // Try v = true
        assign[v] = 1;
        trail.push_back(v);
        if (dpll()) return true;

        // Backtrack
        while ((int)trail.size() > saved) {
            assign[trail.back()] = 0;
            trail.pop_back();
        }
        assign[v] = 0;

        // Try v = false
        assign[v] = -1;
        trail.push_back(v);
        if (dpll()) return true;

        // Backtrack
        while ((int)trail.size() > saved) {
            assign[trail.back()] = 0;
            trail.pop_back();
        }
        assign[v] = 0;

        return false;
    }

    bool solve() { return dpll(); }

    void print_assignment() {
        for (int v = 1; v <= n; v++)
            printf("x%d = %s\n", v, assign[v] == 1 ? "true" : "false");
    }
};

// -------------------------------------------------------
// 2-SAT (polynomial special case)
// Each clause has exactly 2 literals.
// Reduction to SCC on implication graph:
//   clause (a ∨ b) => implications (¬a -> b) and (¬b -> a)
// Satisfiable iff no variable x has x and ¬x in the same SCC.
// -------------------------------------------------------
struct TwoSAT {
    int n;
    vector<vector<int>> g, rg;  // implication graph and reverse
    vector<int> order, comp;
    vector<bool> vis;

    // Variable v (0-indexed): positive literal = 2v, negative = 2v+1
    TwoSAT(int n) : n(n), g(2*n), rg(2*n), comp(2*n), vis(2*n, false) {}

    // Add clause (a ∨ b) where literals are encoded as above
    // Use pos(v)=2v, neg(v)=2v+1
    void add_clause(int a, int b) {
        g[a^1].push_back(b);  // ¬a -> b
        g[b^1].push_back(a);  // ¬b -> a
        rg[b].push_back(a^1);
        rg[a].push_back(b^1);
    }

    void dfs1(int v) {
        vis[v] = true;
        for (int u : g[v]) if (!vis[u]) dfs1(u);
        order.push_back(v);
    }

    void dfs2(int v, int c) {
        comp[v] = c;
        for (int u : rg[v]) if (comp[u] == -1) dfs2(u, c);
    }

    // Returns true if satisfiable; assigns[v] = true means var v is true
    bool solve(vector<bool>& assigns) {
        for (int v = 0; v < 2*n; v++) if (!vis[v]) dfs1(v);
        fill(comp.begin(), comp.end(), -1);
        int c = 0;
        for (int i = 2*n-1; i >= 0; i--)
            if (comp[order[i]] == -1) dfs2(order[i], c++);

        assigns.resize(n);
        for (int v = 0; v < n; v++) {
            if (comp[2*v] == comp[2*v+1]) return false; // UNSAT
            assigns[v] = comp[2*v] > comp[2*v+1];
        }
        return true;
    }
};

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    // CNF: (x1 ∨ ¬x2) ∧ (¬x1 ∨ x3) ∧ (x2 ∨ ¬x3)
    int n = 3;
    vector<vector<int>> clauses = {
        {1, -2}, {-1, 3}, {2, -3}
    };

    SAT sat(n, clauses);
    if (sat.solve()) {
        puts("SAT");
        sat.print_assignment();
    } else {
        puts("UNSAT");
    }

    // 2-SAT example: (x0 ∨ x1) ∧ (¬x0 ∨ x1) ∧ (¬x1 ∨ x0)
    TwoSAT ts(2);
    ts.add_clause(0*2, 1*2);       // x0 ∨ x1
    ts.add_clause(0*2+1, 1*2);     // ¬x0 ∨ x1
    ts.add_clause(1*2+1, 0*2);     // ¬x1 ∨ x0
    vector<bool> asgn;
    if (ts.solve(asgn)) {
        puts("2-SAT: SAT");
        for (int i = 0; i < 2; i++)
            printf("x%d = %s\n", i, asgn[i] ? "true" : "false");
    } else {
        puts("2-SAT: UNSAT");
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

// -------------------------------------------------------
// DPLL SAT Solver
// -------------------------------------------------------
public class SAT {
    private readonly int n;
    private readonly List<int[]> clauses;
    private readonly int[] assign;   // 0=unset, 1=true, -1=false
    private readonly List<int> trail = new();

    public SAT(int n, List<int[]> clauses) {
        this.n = n;
        this.clauses = clauses;
        assign = new int[n + 1];
    }

    private bool LitTrue(int l)  =>
        l > 0 ? assign[l]  == 1 : assign[-l] == -1;

    private bool LitFalse(int l) =>
        l > 0 ? assign[l]  == -1 : assign[-l] == 1;

    private bool Propagate() {
        bool changed = true;
        while (changed) {
            changed = false;
            foreach (var clause in clauses) {
                int unset = 0, unsetLit = 0;
                bool sat = false;
                foreach (int l in clause) {
                    if (LitTrue(l))       { sat = true; break; }
                    if (!LitFalse(l))     { unset++; unsetLit = l; }
                }
                if (sat) continue;
                if (unset == 0) return false;
                if (unset == 1) {
                    int v = Math.Abs(unsetLit);
                    assign[v] = unsetLit > 0 ? 1 : -1;
                    trail.Add(v);
                    changed = true;
                }
            }
        }
        return true;
    }

    private int Choose() {
        for (int v = 1; v <= n; v++)
            if (assign[v] == 0) return v;
        return -1;
    }

    private bool Dpll() {
        if (!Propagate()) return false;
        int v = Choose();
        if (v == -1) return true;

        int saved = trail.Count;

        assign[v] = 1; trail.Add(v);
        if (Dpll()) return true;
        while (trail.Count > saved) { assign[trail[^1]] = 0; trail.RemoveAt(trail.Count - 1); }
        assign[v] = 0;

        assign[v] = -1; trail.Add(v);
        if (Dpll()) return true;
        while (trail.Count > saved) { assign[trail[^1]] = 0; trail.RemoveAt(trail.Count - 1); }
        assign[v] = 0;

        return false;
    }

    public bool Solve() => Dpll();

    public void PrintAssignment() {
        for (int v = 1; v <= n; v++)
            Console.WriteLine($"x{v} = {(assign[v] == 1 ? "true" : "false")}");
    }
}

// -------------------------------------------------------
// 2-SAT Solver (Kosaraju SCC)
// -------------------------------------------------------
public class TwoSAT {
    private readonly int n;
    private readonly List<int>[] g, rg;
    private readonly List<int> order = new();
    private readonly int[] comp;
    private readonly bool[] vis;

    public TwoSAT(int n) {
        this.n = n;
        g   = new List<int>[2 * n];
        rg  = new List<int>[2 * n];
        comp = new int[2 * n];
        vis  = new bool[2 * n];
        for (int i = 0; i < 2 * n; i++) { g[i] = new(); rg[i] = new(); comp[i] = -1; }
    }

    // Literal encoding: pos(v) = 2v, neg(v) = 2v+1
    public void AddClause(int a, int b) {
        g[a ^ 1].Add(b);  rg[b].Add(a ^ 1);
        g[b ^ 1].Add(a);  rg[a].Add(b ^ 1);
    }

    private void Dfs1(int v) {
        vis[v] = true;
        foreach (int u in g[v]) if (!vis[u]) Dfs1(u);
        order.Add(v);
    }

    private void Dfs2(int v, int c) {
        comp[v] = c;
        foreach (int u in rg[v]) if (comp[u] == -1) Dfs2(u, c);
    }

    public bool Solve(out bool[] assigns) {
        for (int v = 0; v < 2 * n; v++) if (!vis[v]) Dfs1(v);
        int c = 0;
        for (int i = 2 * n - 1; i >= 0; i--)
            if (comp[order[i]] == -1) Dfs2(order[i], c++);

        assigns = new bool[n];
        for (int v = 0; v < n; v++) {
            if (comp[2 * v] == comp[2 * v + 1]) return false;
            assigns[v] = comp[2 * v] > comp[2 * v + 1];
        }
        return true;
    }
}

public class Program {
    public static void Main() {
        // General SAT: (x1 ∨ ¬x2) ∧ (¬x1 ∨ x3) ∧ (x2 ∨ ¬x3)
        var clauses = new List<int[]> { new[]{1,-2}, new[]{-1,3}, new[]{2,-3} };
        var sat = new SAT(3, clauses);
        if (sat.Solve()) { Console.WriteLine("SAT"); sat.PrintAssignment(); }
        else Console.WriteLine("UNSAT");

        // 2-SAT: (x0 ∨ x1) ∧ (¬x0 ∨ x1) ∧ (¬x1 ∨ x0)
        var ts = new TwoSAT(2);
        ts.AddClause(0, 2);    // x0 ∨ x1
        ts.AddClause(1, 2);    // ¬x0 ∨ x1
        ts.AddClause(3, 0);    // ¬x1 ∨ x0
        if (ts.Solve(out var asgn)) {
            Console.WriteLine("2-SAT: SAT");
            for (int i = 0; i < 2; i++)
                Console.WriteLine($"x{i} = {asgn[i]}");
        } else Console.WriteLine("2-SAT: UNSAT");
    }
}
```

---

## Special Cases and Reductions

| Problem | Reduction to SAT | Complexity |
|---|---|---|
| 2-SAT | Implication graph + SCC | O(n + m) — polynomial |
| 3-SAT | Already CNF with k=3 | NP-complete, canonical form |
| Horn-SAT | All clauses have ≤ 1 positive literal | O(n * m) — polynomial via unit propagation only |
| XOR-SAT | All clauses are XOR constraints | Polynomial via Gaussian elimination |
| MAX-SAT | Maximize satisfied clauses | NP-hard optimization variant |

---

## Pitfalls

- **CNF conversion** — not all formulas are naturally in CNF. Tseitin transformation converts any formula to equisatisfiable CNF in linear time by introducing auxiliary variables. Converting directly via distribution produces exponential blowup.
- **Unit propagation termination** — naive propagation re-scans all clauses on every change: O(n * m) per propagation round. Industrial solvers use watched literals (two-pointer scheme per clause) to detect unit clauses in O(1) amortized per assignment.
- **Backtracking scope** — when backtracking, every assignment made during propagation after the branch point must be undone, not just the branching variable itself. Maintaining a trail and rewinding to a saved index handles this; forgetting propagated assignments causes wrong SAT verdicts.
- **2-SAT literal encoding** — the standard encoding `pos(v) = 2v`, `neg(v) = 2v+1` means `¬(lit) = lit XOR 1`. Any other encoding requires explicit negation logic and is error-prone. Stick to the XOR-1 convention.
- **2-SAT assignment direction** — after Kosaraju, variable v is true iff `comp[pos(v)] > comp[neg(v)]`. The direction depends on whether SCC numbers are assigned in topological or reverse-topological order. Reversing the comparison gives the wrong assignment (formula still satisfiable but assignments incorrect).
- **Recursive DPLL stack depth** — with n=10^5 variables, recursive DPLL overflows the call stack. Convert to iterative with an explicit decision stack, or increase stack size. CDCL solvers are iterative by design.
- **Learned clause explosion** — CDCL solvers must periodically delete learned clauses (e.g., keep only those with low LBD — Literal Block Distance). Without deletion, memory grows unboundedly and performance degrades due to cache pressure.

---

## Conclusion

SAT is the **canonical NP-complete problem** and the practical workhorse of combinatorial reasoning:

- DPLL with unit propagation handles small to medium structured instances cleanly.
- CDCL with VSIDS, non-chronological backjumping, and clause learning scales to industrial instances with millions of variables.
- 2-SAT and Horn-SAT are tractable special cases solvable in polynomial time.
- Tseitin transformation makes SAT a universal reduction target: encode your NP problem in CNF, hand it to a solver.

**Key takeaway:**  
For competitive programming, implement 2-SAT via SCC for polynomial problems. For NP-hard problems, encode in CNF and call an external CDCL solver (MiniSat, CaDiCaL, Kissat). For understanding, DPLL is the minimal correct implementation — every modern solver is DPLL with CDCL layered on top.
