# DPLL — Davis–Putnam–Logemann–Loveland Algorithm

## Origin & Motivation

DPLL was introduced by Davis and Putnam in 1960 and refined by Logemann and Loveland in 1962. It is the foundational algorithm for deciding **propositional satisfiability (SAT)** on CNF formulas. Every modern CDCL SAT solver (MiniSat, CaDiCaL, Kissat) is a direct extension of DPLL with conflict learning and non-chronological backtracking layered on top.

The algorithm is a systematic backtracking search over variable assignments, heavily pruned by two rules applied exhaustively before each branch: **unit propagation** and **pure literal elimination**. These two rules alone eliminate the vast majority of the search space on structured instances.

Complexity: **O(2^n)** worst case. In practice, unit propagation prunes the tree to a fraction of this bound on most real inputs.

---

## Where It Is Used

- Core engine of all modern CDCL SAT solvers
- Hardware verification (bounded model checking)
- Planning and scheduling (SAT encoding)
- Constraint solving in compilers and package managers
- Teaching: minimal correct SAT decision procedure

---

## Problem Definition

**Input:** CNF formula `F = C_1 ∧ C_2 ∧ ... ∧ C_m`  
Each clause `C_i = (l_1 ∨ l_2 ∨ ... ∨ l_k)`, each literal `l` is `x` or `¬x` for some variable `x`.

**Output:** A satisfying assignment, or UNSAT.

**Three termination conditions checked at each recursive call:**

| Condition | Meaning | Return |
|---|---|---|
| F contains empty clause | Some clause is falsified | UNSAT |
| F is empty (all clauses satisfied) | All clauses true | SAT |
| Neither | Continue branching | recurse |

---

## Core Rules

### Unit Propagation (BCP — Boolean Constraint Propagation)

If a clause contains exactly one **unassigned** literal (all others falsified), that literal **must** be assigned true. Assign it, simplify F, and repeat. This is applied to fixpoint before every branch.

```
Clause: (¬x1 ∨ x2 ∨ ¬x3),  x1=true, x3=true  →  only x2 unassigned
Force:  x2 = true
```

### Pure Literal Elimination

If a variable appears only with positive polarity across all remaining clauses, assign it true (satisfies all its clauses). If only negative, assign it false. Remove all satisfied clauses.

```
x5 appears only as ¬x5 in remaining clauses → assign x5 = false
```

### Branching

Choose an unassigned variable `v`. Recursively try `v = true`, then `v = false` if the first fails.

---

## Algorithm (Pseudocode)

```
DPLL(F, assignment):
    F = unit_propagate(F, assignment)
    if empty clause in F:    return UNSAT
    if F is empty:           return SAT, assignment

    F = pure_literal_elim(F, assignment)
    if empty clause in F:    return UNSAT
    if F is empty:           return SAT, assignment

    v = choose_variable(F)   // heuristic

    if DPLL(F[v=true],  assignment ∪ {v=true})  == SAT: return SAT
    if DPLL(F[v=false], assignment ∪ {v=false}) == SAT: return SAT
    return UNSAT
```

---

## Branching Heuristics

| Heuristic | Strategy | Notes |
|---|---|---|
| DLCS | Most frequent variable in all clauses | Simple, decent quality |
| DLIS | Literal that satisfies the most clauses | Slightly better than DLCS |
| Jeroslow–Wang | Weight literals by 2^(−clause_size) | Prefer short clauses |
| MOM | Most clauses of minimum size | Very effective on 3-SAT |
| VSIDS (CDCL) | Variable activity bumped on conflict | Standard in modern solvers |

---

## Complexity Analysis

| Scenario | Bound | Reason |
|---|---|---|
| Worst case | O(2^n * n * m) | Full binary tree, each node costs O(n*m) |
| With watched literals | O(2^n * m / 64) | BCP in O(1) amortized via two-pointer scheme |
| 2-SAT (k=2) | O(n + m) | SCC reduces to polynomial |
| Horn-SAT (≤1 pos literal/clause) | O(n * m) | Unit propagation alone decides |
| Random 3-SAT near threshold | Exponential | Phase transition at ratio m/n ≈ 4.267 |

- `n` — variables, `m` — clauses

---

## Implementation (C++)

```cpp
#include <bits/stdc++.h>
using namespace std;

// CNF formula. Literal l: positive = l, negative = -l. Variables 1..n.
// Clause = vector<int>. Formula = vector of clauses.
// assign[v]: 0=unset, 1=true, -1=false.

struct DPLL {
    int n;
    vector<vector<int>> clauses;
    vector<int> assign;

    DPLL(int n, vector<vector<int>> cls)
        : n(n), clauses(move(cls)), assign(n + 1, 0) {}

    bool lit_true(int l)  const { return l > 0 ? assign[l]  ==  1 : assign[-l] == -1; }
    bool lit_false(int l) const { return l > 0 ? assign[l]  == -1 : assign[-l] ==  1; }

    // -------------------------------------------------------
    // Unit propagation to fixpoint.
    // Returns false on conflict (empty clause found).
    // Appends forced literals to `forced` for backtracking.
    // -------------------------------------------------------
    bool unit_propagate(vector<int>& forced) {
        bool changed = true;
        while (changed) {
            changed = false;
            for (const auto& clause : clauses) {
                bool sat = false;
                int unset_cnt = 0, unset_lit = 0;
                for (int l : clause) {
                    if (lit_true(l))  { sat = true; break; }
                    if (!lit_false(l)) { unset_cnt++; unset_lit = l; }
                }
                if (sat) continue;
                if (unset_cnt == 0) return false;   // conflict
                if (unset_cnt == 1) {
                    // Unit clause: force unset_lit
                    int v = abs(unset_lit);
                    if (assign[v] != 0) {
                        // Already assigned to the opposite value: conflict
                        if (assign[v] != (unset_lit > 0 ? 1 : -1)) return false;
                        continue;
                    }
                    assign[v] = (unset_lit > 0) ? 1 : -1;
                    forced.push_back(v);
                    changed = true;
                }
            }
        }
        return true;
    }

    // -------------------------------------------------------
    // Pure literal elimination.
    // For each unassigned variable, check if it appears only
    // positive or only negative in unsatisfied clauses.
    // Appends assigned variables to `forced`.
    // -------------------------------------------------------
    void pure_literal_elim(vector<int>& forced) {
        vector<int> has_pos(n + 1, 0), has_neg(n + 1, 0);
        for (const auto& clause : clauses) {
            bool sat = false;
            for (int l : clause) if (lit_true(l)) { sat = true; break; }
            if (sat) continue;
            for (int l : clause) {
                if (assign[abs(l)] != 0) continue;
                if (l > 0) has_pos[l]  = 1;
                else       has_neg[-l] = 1;
            }
        }
        for (int v = 1; v <= n; v++) {
            if (assign[v] != 0) continue;
            if (has_pos[v] && !has_neg[v]) {
                assign[v] = 1; forced.push_back(v);
            } else if (!has_pos[v] && has_neg[v]) {
                assign[v] = -1; forced.push_back(v);
            }
        }
    }

    // -------------------------------------------------------
    // Check formula status under current assignment.
    // Returns: 1=SAT, -1=UNSAT(conflict), 0=unresolved
    // -------------------------------------------------------
    int status() const {
        bool all_sat = true;
        for (const auto& clause : clauses) {
            bool sat = false, has_unset = false;
            for (int l : clause) {
                if (lit_true(l))  { sat = true; break; }
                if (!lit_false(l)) has_unset = true;
            }
            if (!sat && !has_unset) return -1;  // empty clause
            if (!sat) all_sat = false;
        }
        return all_sat ? 1 : 0;
    }

    // -------------------------------------------------------
    // Branching heuristic: DLIS
    // Choose literal that satisfies the most unresolved clauses.
    // -------------------------------------------------------
    int choose_literal() const {
        vector<int> score(2 * (n + 1), 0); // score[2v]=pos, score[2v+1]=neg
        for (const auto& clause : clauses) {
            bool sat = false;
            for (int l : clause) if (lit_true(l)) { sat = true; break; }
            if (sat) continue;
            for (int l : clause) {
                if (assign[abs(l)] != 0) continue;
                if (l > 0) score[2 * l]++;
                else       score[2 * (-l) + 1]++;
            }
        }
        int best = -1, best_score = -1;
        for (int v = 1; v <= n; v++) {
            if (assign[v] != 0) continue;
            if (score[2*v]   > best_score) { best = v;  best_score = score[2*v];   }
            if (score[2*v+1] > best_score) { best = -v; best_score = score[2*v+1]; }
        }
        return best; // positive = branch true first, negative = branch false first
    }

    // -------------------------------------------------------
    // DPLL recursive solve.
    // Returns true if satisfying assignment exists.
    // -------------------------------------------------------
    bool solve() {
        // Unit propagation
        vector<int> forced;
        if (!unit_propagate(forced)) {
            for (int v : forced) assign[v] = 0;
            return false;
        }

        // Pure literal elimination
        pure_literal_elim(forced);

        int s = status();
        if (s == 1)  { return true; }
        if (s == -1) { for (int v : forced) assign[v] = 0; return false; }

        // Branch
        int lit = choose_literal();
        int v   = abs(lit);
        int val = (lit > 0) ? 1 : -1;

        // Try preferred polarity
        assign[v] = val;
        if (solve()) return true;
        assign[v] = 0;

        // Try opposite polarity
        assign[v] = -val;
        if (solve()) return true;
        assign[v] = 0;

        // Undo propagated assignments before returning UNSAT to caller
        for (int u : forced) assign[u] = 0;
        return false;
    }

    void print() const {
        for (int v = 1; v <= n; v++)
            printf("x%d = %s\n", v, assign[v] == 1 ? "true" : "false");
    }
};

// -------------------------------------------------------
// Watched Literals (two-pointer BCP) — production BCP
// Each clause watches two unfalsfied literals.
// Only re-examines a clause when one of its watched literals
// becomes false — amortized O(1) per assignment.
// -------------------------------------------------------
struct WatchedBCP {
    int n;
    vector<vector<int>> clauses;
    // watch[lit_id] = list of clause indices watching this literal
    // lit_id: positive literal of var v = 2*v, negative = 2*v+1
    vector<vector<int>> watch;
    vector<int> assign;     // 0=unset, 1=true, -1=false
    vector<int> watched;    // watched[i] = index of second watch in clause i

    int enc(int l) { return l > 0 ? 2*l : 2*(-l)+1; }
    bool lt(int lid) {
        int v = lid/2;
        return (lid&1) ? assign[v]==-1 : assign[v]==1;
    }
    bool lf(int lid) {
        int v = lid/2;
        return (lid&1) ? assign[v]==1 : assign[v]==-1;
    }

    WatchedBCP(int n, const vector<vector<int>>& cls)
        : n(n), clauses(cls), watch(2*(n+1)), assign(n+1, 0),
          watched(cls.size(), 1)
    {
        for (int i = 0; i < (int)clauses.size(); i++) {
            if (clauses[i].size() >= 1) watch[enc(clauses[i][0])].push_back(i);
            if (clauses[i].size() >= 2) watch[enc(clauses[i][1])].push_back(i);
        }
    }

    // Assign literal l, propagate, push units to queue
    // Returns false on conflict
    bool assign_lit(int l, vector<int>& trail) {
        int v = abs(l);
        if (assign[v] != 0) return assign[v] == (l>0 ? 1 : -1);
        assign[v] = l > 0 ? 1 : -1;
        trail.push_back(v);

        // The falsified literal's watcher list needs updating
        int false_lid = enc(-l);
        auto& w = watch[false_lid];
        for (int i = 0; i < (int)w.size(); ) {
            int ci = w[i];
            auto& clause = clauses[ci];
            // Find another literal to watch
            bool found = false;
            for (int j = 0; j < (int)clause.size(); j++) {
                int lid = enc(clause[j]);
                if (lid == false_lid) continue;
                if (lf(lid)) continue;  // also false, skip
                // Move watch to clause[j]
                watch[lid].push_back(ci);
                w.erase(w.begin() + i);
                found = true;
                break;
            }
            if (!found) {
                // All other literals false: check for unit or conflict
                bool sat = false;
                int unit_lit = 0;
                for (int lx : clause) {
                    if (lt(enc(lx))) { sat = true; break; }
                    if (!lf(enc(lx))) unit_lit = lx;
                }
                if (!sat) {
                    if (unit_lit == 0) return false;  // conflict
                    if (!assign_lit(unit_lit, trail)) return false;
                }
                i++;
            }
        }
        return true;
    }
};

// -------------------------------------------------------
// Usage
// -------------------------------------------------------
int main() {
    // (x1 ∨ ¬x2) ∧ (¬x1 ∨ x3) ∧ (x2 ∨ ¬x3) ∧ (¬x1 ∨ ¬x2 ∨ ¬x3)
    int n = 3;
    vector<vector<int>> clauses = {
        {1, -2}, {-1, 3}, {2, -3}, {-1, -2, -3}
    };

    DPLL dpll(n, clauses);
    if (dpll.solve()) {
        puts("SAT");
        dpll.print();
    } else {
        puts("UNSAT");
    }

    return 0;
}
```

---

## Implementation (C#)

```csharp
using System;
using System.Collections.Generic;

public class DPLL {
    private readonly int n;
    private readonly List<int[]> clauses;
    private readonly int[] assign;  // 0=unset, 1=true, -1=false

    public DPLL(int n, List<int[]> clauses) {
        this.n = n;
        this.clauses = clauses;
        assign = new int[n + 1];
    }

    private bool LitTrue(int l)  => l > 0 ? assign[l]  ==  1 : assign[-l] == -1;
    private bool LitFalse(int l) => l > 0 ? assign[l]  == -1 : assign[-l] ==  1;

    // Unit propagation to fixpoint. Returns false on conflict.
    private bool UnitPropagate(List<int> forced) {
        bool changed = true;
        while (changed) {
            changed = false;
            foreach (var clause in clauses) {
                bool sat = false;
                int unsetCnt = 0, unsetLit = 0;
                foreach (int l in clause) {
                    if (LitTrue(l))       { sat = true; break; }
                    if (!LitFalse(l))     { unsetCnt++; unsetLit = l; }
                }
                if (sat) continue;
                if (unsetCnt == 0) return false;
                if (unsetCnt == 1) {
                    int v = Math.Abs(unsetLit);
                    if (assign[v] != 0) {
                        if (assign[v] != (unsetLit > 0 ? 1 : -1)) return false;
                        continue;
                    }
                    assign[v] = unsetLit > 0 ? 1 : -1;
                    forced.Add(v);
                    changed = true;
                }
            }
        }
        return true;
    }

    // Pure literal elimination.
    private void PureLiteralElim(List<int> forced) {
        var hasPos = new bool[n + 1];
        var hasNeg = new bool[n + 1];
        foreach (var clause in clauses) {
            bool sat = clause is { } c && Array.Exists(c, LitTrue);
            if (sat) continue;
            foreach (int l in clause) {
                if (assign[Math.Abs(l)] != 0) continue;
                if (l > 0) hasPos[l]  = true;
                else       hasNeg[-l] = true;
            }
        }
        for (int v = 1; v <= n; v++) {
            if (assign[v] != 0) continue;
            if (hasPos[v] && !hasNeg[v])  { assign[v] =  1; forced.Add(v); }
            else if (!hasPos[v] && hasNeg[v]) { assign[v] = -1; forced.Add(v); }
        }
    }

    // Formula status: 1=SAT, -1=conflict, 0=unresolved
    private int Status() {
        bool allSat = true;
        foreach (var clause in clauses) {
            bool sat = false, hasUnset = false;
            foreach (int l in clause) {
                if (LitTrue(l))  { sat = true; break; }
                if (!LitFalse(l)) hasUnset = true;
            }
            if (!sat && !hasUnset) return -1;
            if (!sat) allSat = false;
        }
        return allSat ? 1 : 0;
    }

    // DLIS branching: literal satisfying the most unresolved clauses
    private int ChooseLiteral() {
        var scorePos = new int[n + 1];
        var scoreNeg = new int[n + 1];
        foreach (var clause in clauses) {
            bool sat = false;
            foreach (int l in clause) if (LitTrue(l)) { sat = true; break; }
            if (sat) continue;
            foreach (int l in clause) {
                if (assign[Math.Abs(l)] != 0) continue;
                if (l > 0) scorePos[l]++;
                else       scoreNeg[-l]++;
            }
        }
        int best = 1, bestScore = -1;
        bool bestPos = true;
        for (int v = 1; v <= n; v++) {
            if (assign[v] != 0) continue;
            if (scorePos[v] > bestScore) { best = v; bestScore = scorePos[v]; bestPos = true;  }
            if (scoreNeg[v] > bestScore) { best = v; bestScore = scoreNeg[v]; bestPos = false; }
        }
        return bestPos ? best : -best;
    }

    private bool Solve() {
        var forced = new List<int>();

        if (!UnitPropagate(forced)) {
            foreach (int v in forced) assign[v] = 0;
            return false;
        }
        PureLiteralElim(forced);

        int s = Status();
        if (s ==  1) return true;
        if (s == -1) { foreach (int v in forced) assign[v] = 0; return false; }

        int lit = ChooseLiteral();
        int var_ = Math.Abs(lit);
        int val  = lit > 0 ? 1 : -1;

        assign[var_] = val;
        if (Solve()) return true;
        assign[var_] = 0;

        assign[var_] = -val;
        if (Solve()) return true;
        assign[var_] = 0;

        foreach (int v in forced) assign[v] = 0;
        return false;
    }

    public bool Run() => Solve();

    public void Print() {
        for (int v = 1; v <= n; v++)
            Console.WriteLine($"x{v} = {(assign[v] == 1 ? "true" : "false")}");
    }

    public static void Main() {
        var clauses = new List<int[]> {
            new[]{1,-2}, new[]{-1,3}, new[]{2,-3}, new[]{-1,-2,-3}
        };
        var dpll = new DPLL(3, clauses);
        if (dpll.Run()) { Console.WriteLine("SAT"); dpll.Print(); }
        else Console.WriteLine("UNSAT");
    }
}
```

---

## DPLL vs CDCL — Extension Path

| Feature | DPLL | CDCL |
|---|---|---|
| Backtracking | Chronological (one level) | Non-chronological (jump to conflict level) |
| Learned clauses | None | Derived from implication graph on conflict |
| Variable order | Static heuristic | Dynamic VSIDS (activity-based) |
| Restart policy | None | Luby sequence or geometric restarts |
| BCP | Naive re-scan | Watched literals (O(1) amortized) |
| Practical scale | ~100 variables | 10^6+ variables |

CDCL is DPLL with four additions: implication graph, conflict analysis, non-chronological backjumping, and clause learning. Every CDCL solver reduces to DPLL when learning is disabled.

---

## Pitfalls

- **Backtracking must undo propagated assignments** — when a branch fails, every variable forced by unit propagation after the branch point must be reset to unset. Forgetting propagated assignments produces false SAT results: the solver returns a partial assignment it believes is complete.
- **Pure literal elimination is optional** — it is always correct but not always profitable. On large instances it costs O(n * m) per call and rarely eliminates enough variables to justify the cost. Modern CDCL solvers omit it entirely.
- **Unit propagation is not deduplication** — the same variable may be forced multiple times if it appears in multiple unit clauses. Check if already assigned before forcing; if assigned to the opposite value, return conflict immediately.
- **Empty clause vs empty formula** — the termination check must test both. An empty clause (all literals falsified) means conflict. An empty formula (all clauses satisfied, no clause remains) means SAT. Testing only one of the two gives wrong results on boundary inputs.
- **DLIS heuristic recomputes from scratch** — the DLIS score loop above is O(n * m) per branch. For large formulas, cache scores and update incrementally, or use a simpler heuristic (first unassigned variable) during development.
- **Clause simplification vs formula copy** — passing a simplified copy of F at each recursive level is O(n * m) space per depth level. The implementation above avoids copying by working with the global assignment and checking lit_true / lit_false inline — this is the standard approach.
- **Stack overflow on deep recursion** — DPLL's recursion depth can reach `n` (one branch per variable). For n = 10^5, this overflows the default stack. Convert to an iterative loop with an explicit decision stack, or limit n to a few thousand for the recursive version.

---

## Conclusion

DPLL is the **algorithmic core of all SAT solving**:

- Unit propagation and pure literal elimination make it dramatically more efficient than naive backtracking on structured instances.
- Branching heuristics (DLIS, MOM, Jeroslow–Wang) reduce the tree size further by choosing high-impact variables first.
- Every modern industrial solver (MiniSat, Glucose, CaDiCaL) is DPLL extended with CDCL — the base algorithm is unchanged, only the learning and backjumping layers are added.
- 2-SAT and Horn-SAT are polynomial special cases where DPLL terminates without branching (unit propagation alone decides).

**Key takeaway:**  
Implement DPLL for correctness and understanding; switch to a CDCL solver (or call an external one) for any instance with more than a few hundred variables. The watched-literal BCP scheme is the single most impactful optimization — it reduces the constant factor of unit propagation from O(m) to O(1) amortized, which dominates runtime in practice.
