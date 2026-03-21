# SMT Solver (Z3 Overview) — Constraint Logic

## Origin & Motivation

Satisfiability Modulo Theories (SMT) extends SAT by allowing formulas over **typed variables** with **background theories** — arithmetic, arrays, bitvectors, strings, uninterpreted functions, and more. Where SAT asks "is there a boolean assignment satisfying this formula?", SMT asks "is there an assignment over integers / reals / bitvectors / ... satisfying this formula?".

Z3 was developed at Microsoft Research by de Moura and Bjørner (2008) and is the most widely used SMT solver. It implements the **DPLL(T)** architecture: a CDCL SAT engine drives the search while **theory solvers** check consistency of partial assignments in their respective theories and feed conflicts back to the SAT engine as theory lemmas.

Complexity: **undecidable** in general (first-order arithmetic). Decidable for many useful fragments: linear arithmetic, bitvectors, quantifier-free theories. Each decidable fragment has its own complexity class.

---

## Where It Is Used

- Program verification and symbolic execution (KLEE, angr, Triton)
- Hardware model checking (bounded, unbounded)
- Security: bug finding, exploit generation, binary analysis
- Compiler optimization and synthesis
- Type checking and protocol verification (TLA+, Dafny, F*)
- Competitive programming: encoding constraint problems
- Machine learning fairness and neural network verification

---

## Theories Supported by Z3

| Theory | Sorts / Operations | Decidability |
|---|---|---|
| Linear Integer Arithmetic (LIA) | Z, +, -, *, constants | Decidable (PSPACE) |
| Linear Real Arithmetic (LRA) | R, +, -, *, / | Decidable (polynomial) |
| Nonlinear Arithmetic (NIA/NRA) | Z/R, *, / between vars | Undecidable in general |
| Bitvectors (BV) | Fixed-width integers, shifts, bitops | Decidable (NP-complete) |
| Arrays | select, store | Decidable (quantifier-free) |
| Uninterpreted Functions (UF) | f(x)=f(y) => x=y style | Decidable (PTIME congruence) |
| Strings / RegEx | concat, length, match | Decidable (fragments) |
| Floating Point (FP) | IEEE 754 ops | Decidable |
| Datatypes | Algebraic / recursive types | Decidable (quantifier-free) |

---

## DPLL(T) Architecture

```
                    ┌──────────────────────────┐
  Formula F  ──►   │   Boolean Abstraction     │
                    │   (propositional skeleton) │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │     CDCL SAT Engine       │
                    │  (drives search over      │
                    │   boolean assignments)    │
                    └────────────┬─────────────┘
                    conflict ◄───┤──► partial assignment
                    lemma        │
                    ┌────────────▼─────────────┐
                    │    Theory Solver(s)       │
                    │  LIA | BV | Arrays | UF  │
                    │  Check: is assignment    │
                    │  T-consistent?           │
                    └──────────────────────────┘
```

1. CDCL assigns truth values to **abstract boolean variables** representing theory atoms (`x + y <= 5`, `a[i] == 3`, ...).
2. The current partial assignment is sent to theory solvers for **T-consistency** check.
3. If T-inconsistent, the theory solver returns a **theory lemma** (a clause over atoms) that rules out this combination. CDCL learns this clause and backtracks.
4. If T-consistent and all atoms assigned: **SAT** with a model. If CDCL exhausts all possibilities: **UNSAT**.

---

## Core Algorithms Inside Z3

### Congruence Closure (UF + Equality)

Maintains an equivalence relation over terms. When `f(a) = b` and `a = c` are asserted, derives `f(c) = b` automatically. Implemented with union-find + a congruence closure algorithm in O(n log n).

### Simplex (Linear Arithmetic)

Z3 uses a **dual simplex** variant adapted for incremental solving (Dutertre and de Moura, 2006). Maintains a tableau; pivots on infeasible rows. Supports incremental assert/retract without full re-solve. Integer arithmetic handled by **Gomory cuts** and **branch-and-bound** on top of LP relaxation.

### Bit-Blasting (Bitvectors)

Each bitvector operation is reduced to a boolean circuit — addition becomes a ripple-carry adder in CNF, multiplication becomes an array of adders, etc. The resulting CNF is handed to the CDCL engine. For large bitvectors, Z3 also uses word-level reasoning to avoid full blasting.

### Nelson-Oppen (Theory Combination)

When multiple theories appear simultaneously (e.g., LIA + Arrays + UF), Z3 uses the **Nelson-Oppen** framework: theories share only equalities over shared variables. Each theory propagates implied equalities to others. Requires theories to be **stably-infinite** and **disjoint** for correctness.

---

## Complexity by Fragment

| Fragment | Complexity |
|---|---|
| Propositional SAT | NP-complete |
| QF_LRA (quantifier-free linear reals) | PTIME (simplex) |
| QF_LIA (quantifier-free linear integers) | NP-complete |
| QF_BV (quantifier-free bitvectors) | NP-complete |
| QF_UF (quantifier-free uninterpreted functions) | PTIME (congruence closure) |
| QF_AUFLIA (arrays + UF + LIA) | NP-complete |
| Full first-order arithmetic | Undecidable |

---

## Implementation (C++ — Z3 C API)

```cpp
#include <z3.h>
#include <cstdio>

// -------------------------------------------------------
// 1. Basic integer constraint solving
// -------------------------------------------------------
void example_linear_arithmetic() {
    Z3_config  cfg = Z3_mk_config();
    Z3_context ctx = Z3_mk_context(cfg);
    Z3_del_config(cfg);

    Z3_solver solver = Z3_mk_solver(ctx);
    Z3_solver_inc_ref(ctx, solver);

    // Declare integer variables x, y
    Z3_sort int_sort = Z3_mk_int_sort(ctx);
    Z3_ast x = Z3_mk_const(ctx, Z3_mk_string_symbol(ctx, "x"), int_sort);
    Z3_ast y = Z3_mk_const(ctx, Z3_mk_string_symbol(ctx, "y"), int_sort);

    // Assert: x + y == 10
    Z3_ast ten = Z3_mk_int(ctx, 10, int_sort);
    Z3_ast sum = Z3_mk_add(ctx, 2, (Z3_ast[]){x, y});
    Z3_solver_assert(ctx, solver, Z3_mk_eq(ctx, sum, ten));

    // Assert: x >= 3
    Z3_ast three = Z3_mk_int(ctx, 3, int_sort);
    Z3_solver_assert(ctx, solver,
        Z3_mk_ge(ctx, x, three));

    // Assert: y >= 4
    Z3_ast four = Z3_mk_int(ctx, 4, int_sort);
    Z3_solver_assert(ctx, solver,
        Z3_mk_ge(ctx, y, four));

    Z3_lbool result = Z3_solver_check(ctx, solver);
    if (result == Z3_L_TRUE) {
        Z3_model model = Z3_solver_get_model(ctx, solver);
        Z3_ast x_val, y_val;
        Z3_model_eval(ctx, model, x, 1, &x_val);
        Z3_model_eval(ctx, model, y, 1, &y_val);
        printf("SAT: x=%s, y=%s\n",
            Z3_ast_to_string(ctx, x_val),
            Z3_ast_to_string(ctx, y_val));
    } else {
        printf("UNSAT\n");
    }

    Z3_solver_dec_ref(ctx, solver);
    Z3_del_context(ctx);
}

// -------------------------------------------------------
// 2. Bitvector reasoning (overflow-safe arithmetic)
// -------------------------------------------------------
void example_bitvector() {
    Z3_config  cfg = Z3_mk_config();
    Z3_context ctx = Z3_mk_context(cfg);
    Z3_del_config(cfg);

    Z3_solver solver = Z3_mk_solver(ctx);
    Z3_solver_inc_ref(ctx, solver);

    // 8-bit unsigned variables a, b
    Z3_sort bv8 = Z3_mk_bv_sort(ctx, 8);
    Z3_ast a = Z3_mk_const(ctx, Z3_mk_string_symbol(ctx, "a"), bv8);
    Z3_ast b = Z3_mk_const(ctx, Z3_mk_string_symbol(ctx, "b"), bv8);

    // Assert: a + b overflows (unsigned)
    // i.e., bvadd_no_overflow is false
    Z3_ast no_overflow = Z3_mk_bvadd_no_overflow(ctx, a, b, false);
    Z3_solver_assert(ctx, solver, Z3_mk_not(ctx, no_overflow));

    // Assert: a > 200, b > 200
    Z3_ast c200 = Z3_mk_unsigned_int(ctx, 200, bv8);
    Z3_solver_assert(ctx, solver, Z3_mk_bvugt(ctx, a, c200));
    Z3_solver_assert(ctx, solver, Z3_mk_bvugt(ctx, b, c200));

    Z3_lbool result = Z3_solver_check(ctx, solver);
    printf("BV overflow example: %s\n",
        result == Z3_L_TRUE ? "SAT (overflow found)" : "UNSAT");

    if (result == Z3_L_TRUE) {
        Z3_model model = Z3_solver_get_model(ctx, solver);
        Z3_ast av, bv;
        Z3_model_eval(ctx, model, a, 1, &av);
        Z3_model_eval(ctx, model, b, 1, &bv);
        printf("  a=%s, b=%s\n",
            Z3_ast_to_string(ctx, av),
            Z3_ast_to_string(ctx, bv));
    }

    Z3_solver_dec_ref(ctx, solver);
    Z3_del_context(ctx);
}

// -------------------------------------------------------
// 3. Array theory (out-of-bounds access detection)
// -------------------------------------------------------
void example_arrays() {
    Z3_config  cfg = Z3_mk_config();
    Z3_context ctx = Z3_mk_context(cfg);
    Z3_del_config(cfg);

    Z3_solver solver = Z3_mk_solver(ctx);
    Z3_solver_inc_ref(ctx, solver);

    Z3_sort int_sort = Z3_mk_int_sort(ctx);
    Z3_sort arr_sort = Z3_mk_array_sort(ctx, int_sort, int_sort);

    // Array A, index i, size n=10
    Z3_ast A = Z3_mk_const(ctx, Z3_mk_string_symbol(ctx, "A"), arr_sort);
    Z3_ast i = Z3_mk_const(ctx, Z3_mk_string_symbol(ctx, "i"), int_sort);
    Z3_ast n = Z3_mk_int(ctx, 10, int_sort);
    Z3_ast zero = Z3_mk_int(ctx, 0, int_sort);

    // Assert: access A[i] is in-bounds (0 <= i < 10)
    // Then ask: can i be out of bounds?
    // Negate: i < 0 OR i >= 10
    Z3_ast oob_lo = Z3_mk_lt(ctx, i, zero);
    Z3_ast oob_hi = Z3_mk_ge(ctx, i, n);
    Z3_ast oob    = Z3_mk_or(ctx, 2, (Z3_ast[]){oob_lo, oob_hi});
    Z3_solver_assert(ctx, solver, oob);

    // Additional constraint: some program logic sets i = 5 (in-bounds)
    Z3_ast five = Z3_mk_int(ctx, 5, int_sort);
    Z3_solver_assert(ctx, solver, Z3_mk_eq(ctx, i, five));

    Z3_lbool result = Z3_solver_check(ctx, solver);
    printf("Out-of-bounds example: %s\n",
        result == Z3_L_TRUE ? "SAT (OOB reachable)" : "UNSAT (safe)");

    Z3_solver_dec_ref(ctx, solver);
    Z3_del_context(ctx);
}

// -------------------------------------------------------
// 4. Incremental solving (push/pop)
// -------------------------------------------------------
void example_incremental() {
    Z3_config  cfg = Z3_mk_config();
    Z3_context ctx = Z3_mk_context(cfg);
    Z3_del_config(cfg);

    Z3_solver solver = Z3_mk_solver(ctx);
    Z3_solver_inc_ref(ctx, solver);

    Z3_sort int_sort = Z3_mk_int_sort(ctx);
    Z3_ast x = Z3_mk_const(ctx, Z3_mk_string_symbol(ctx, "x"), int_sort);

    // Persistent base constraint: x > 0
    Z3_solver_assert(ctx, solver,
        Z3_mk_gt(ctx, x, Z3_mk_int(ctx, 0, int_sort)));

    // Temporary constraint scope 1: x < 5
    Z3_solver_push(ctx, solver);
    Z3_solver_assert(ctx, solver,
        Z3_mk_lt(ctx, x, Z3_mk_int(ctx, 5, int_sort)));
    printf("x > 0 and x < 5: %s\n",
        Z3_solver_check(ctx, solver) == Z3_L_TRUE ? "SAT" : "UNSAT");
    Z3_solver_pop(ctx, solver, 1);   // remove x < 5

    // Temporary constraint scope 2: x < 0
    Z3_solver_push(ctx, solver);
    Z3_solver_assert(ctx, solver,
        Z3_mk_lt(ctx, x, Z3_mk_int(ctx, 0, int_sort)));
    printf("x > 0 and x < 0: %s\n",
        Z3_solver_check(ctx, solver) == Z3_L_TRUE ? "SAT" : "UNSAT");
    Z3_solver_pop(ctx, solver, 1);

    Z3_solver_dec_ref(ctx, solver);
    Z3_del_context(ctx);
}

int main() {
    example_linear_arithmetic();
    example_bitvector();
    example_arrays();
    example_incremental();
    return 0;
}
```

**Build:** `g++ -o smt smt.cpp -lz3`

---

## Implementation (C# — Microsoft.Z3 NuGet)

```csharp
// NuGet: Microsoft.Z3
using System;
using Microsoft.Z3;

public class SMTExamples {

    // -------------------------------------------------------
    // 1. Linear integer arithmetic
    // -------------------------------------------------------
    static void LinearArithmetic() {
        using var ctx    = new Context();
        using var solver = ctx.MkSolver();

        IntExpr x = ctx.MkIntConst("x");
        IntExpr y = ctx.MkIntConst("y");

        // x + y == 10, x >= 3, y >= 4
        solver.Assert(ctx.MkEq(ctx.MkAdd(x, y), ctx.MkInt(10)));
        solver.Assert(ctx.MkGe(x, ctx.MkInt(3)));
        solver.Assert(ctx.MkGe(y, ctx.MkInt(4)));

        Status status = solver.Check();
        Console.WriteLine($"LIA: {status}");
        if (status == Status.SATISFIABLE) {
            Model m = solver.Model;
            Console.WriteLine($"  x={m.Eval(x)}, y={m.Eval(y)}");
        }
    }

    // -------------------------------------------------------
    // 2. Bitvector overflow detection
    // -------------------------------------------------------
    static void BitvectorOverflow() {
        using var ctx    = new Context();
        using var solver = ctx.MkSolver();

        BitVecExpr a = ctx.MkBVConst("a", 8);
        BitVecExpr b = ctx.MkBVConst("b", 8);

        // a + b overflows unsigned 8-bit
        BoolExpr overflow = ctx.MkNot(ctx.MkBVAddNoOverflow(a, b, false));
        solver.Assert(overflow);
        solver.Assert(ctx.MkBVUGT(a, ctx.MkBV(200, 8)));
        solver.Assert(ctx.MkBVUGT(b, ctx.MkBV(200, 8)));

        Status status = solver.Check();
        Console.WriteLine($"BV overflow: {status}");
        if (status == Status.SATISFIABLE) {
            Model m = solver.Model;
            Console.WriteLine($"  a={m.Eval(a)}, b={m.Eval(b)}");
        }
    }

    // -------------------------------------------------------
    // 3. Array out-of-bounds
    // -------------------------------------------------------
    static void ArrayBounds() {
        using var ctx    = new Context();
        using var solver = ctx.MkSolver();

        ArraySort arrSort = ctx.MkArraySort(ctx.IntSort, ctx.IntSort);
        Expr      A = ctx.MkConst("A", arrSort);
        IntExpr   i = ctx.MkIntConst("i");

        // i < 0 OR i >= 10  (out of bounds)
        solver.Assert(ctx.MkOr(
            ctx.MkLt(i, ctx.MkInt(0)),
            ctx.MkGe(i, ctx.MkInt(10))
        ));
        // Program forces i = 5
        solver.Assert(ctx.MkEq(i, ctx.MkInt(5)));

        Status status = solver.Check();
        Console.WriteLine($"Array OOB: {status}");  // UNSAT = safe
    }

    // -------------------------------------------------------
    // 4. Incremental solving with push/pop
    // -------------------------------------------------------
    static void Incremental() {
        using var ctx    = new Context();
        using var solver = ctx.MkSolver();

        IntExpr x = ctx.MkIntConst("x");
        solver.Assert(ctx.MkGt(x, ctx.MkInt(0)));  // permanent: x > 0

        solver.Push();
        solver.Assert(ctx.MkLt(x, ctx.MkInt(5)));  // temporary: x < 5
        Console.WriteLine($"x>0 && x<5: {solver.Check()}");   // SAT
        solver.Pop();

        solver.Push();
        solver.Assert(ctx.MkLt(x, ctx.MkInt(0)));  // temporary: x < 0
        Console.WriteLine($"x>0 && x<0: {solver.Check()}");   // UNSAT
        solver.Pop();
    }

    // -------------------------------------------------------
    // 5. Quantified formula (forall)
    // -------------------------------------------------------
    static void Quantifiers() {
        using var ctx    = new Context();
        using var solver = ctx.MkSolver();

        IntExpr  x    = ctx.MkIntConst("x");
        FuncDecl f    = ctx.MkFuncDecl("f", ctx.IntSort, ctx.IntSort);
        Expr     fx   = f.Apply(x);

        // Assert: forall x. f(x) > x
        BoolExpr body  = ctx.MkGt((ArithExpr)fx, x);
        BoolExpr forall = ctx.MkForall(new Expr[]{x}, body);
        solver.Assert(forall);

        // Ask: is there x such that f(x) == x?  (should be UNSAT)
        IntExpr  x2 = ctx.MkIntConst("x2");
        solver.Assert(ctx.MkEq(f.Apply(x2), x2));

        Console.WriteLine($"Quantifier example: {solver.Check()}"); // UNSAT
    }

    // -------------------------------------------------------
    // 6. Optimization (Z3 Optimize — MaxSMT / weighted objectives)
    // -------------------------------------------------------
    static void Optimization() {
        using var ctx = new Context();
        using var opt = ctx.MkOptimize();

        IntExpr x = ctx.MkIntConst("x");
        IntExpr y = ctx.MkIntConst("y");

        opt.Assert(ctx.MkLe(ctx.MkAdd(x, y), ctx.MkInt(10)));
        opt.Assert(ctx.MkGe(x, ctx.MkInt(0)));
        opt.Assert(ctx.MkGe(y, ctx.MkInt(0)));

        // Maximize x + 2*y
        opt.MkMaximize(ctx.MkAdd(x, ctx.MkMul(ctx.MkInt(2), y)));

        Status status = opt.Check();
        Console.WriteLine($"Optimization: {status}");
        if (status == Status.SATISFIABLE) {
            Model m = opt.Model;
            Console.WriteLine($"  x={m.Eval(x)}, y={m.Eval(y)}");
            // x=0, y=5 => objective=10
        }
    }

    public static void Main() {
        LinearArithmetic();
        BitvectorOverflow();
        ArrayBounds();
        Incremental();
        Quantifiers();
        Optimization();
    }
}
```

**Build:** `dotnet add package Microsoft.Z3`

---

## SMT-LIB2 Format (Universal Input)

All SMT solvers accept the standard **SMT-LIB2** text format. Z3 can be invoked as a command-line tool on `.smt2` files:

```smt2
; Linear arithmetic example
(set-logic QF_LIA)
(declare-const x Int)
(declare-const y Int)
(assert (= (+ x y) 10))
(assert (>= x 3))
(assert (>= y 4))
(check-sat)
(get-model)
```

```smt2
; Bitvector example
(set-logic QF_BV)
(declare-const a (_ BitVec 8))
(declare-const b (_ BitVec 8))
(assert (not (bvadd-no-overflow a b false)))
(assert (bvugt a #xC8))
(assert (bvugt b #xC8))
(check-sat)
(get-model)
```

```bash
z3 input.smt2
```

---

## Pitfalls

- **Non-linear arithmetic triggers incompleteness** — as soon as two symbolic variables are multiplied (`x * y`), Z3 falls back to incomplete heuristics (e.g., model-based projection or NL arithmetic solver). It may return `unknown` rather than SAT or UNSAT. Restructure to linear where possible or use bitvectors with bit-blasting.
- **Quantifiers are undecidable in general** — Z3 uses E-matching and MBQI (model-based quantifier instantiation) for quantified formulas. These are heuristics: Z3 may loop, time out, or return `unknown` even on satisfiable instances. For verification, prefer quantifier-free encodings.
- **Push/Pop invalidates the model** — after `solver.Pop()`, previously computed models are no longer valid. Always call `Check()` again after any `Pop()` before accessing the model.
- **Theory combination requires stably-infinite theories** — Nelson-Oppen works correctly only when all combined theories are stably-infinite (every satisfiable formula has an infinite model). Finite theories (bitvectors) require special handling. Z3 manages this internally, but hand-combining theories via axioms can violate this requirement silently.
- **Soft assertions in MaxSMT must be weighted** — Z3's `Optimize` supports hard constraints (`Assert`) and soft constraints (`AssertSoft` with a weight). Mixing them requires explicitly setting weights; a soft constraint with weight 0 is ignored, not treated as hard.
- **Model evaluation requires `model_completion=true`** — when calling `model.Eval(expr)`, pass `true` as the second argument to get a concrete value for uninterpreted constants. Without it, Z3 may return the expression unevaluated for variables not directly constrained.
- **Context is not thread-safe** — a single Z3 `Context` (or `Z3_context`) must not be used from multiple threads concurrently. Create one context per thread; contexts are independent and can be used in parallel.
- **String theory is slow** — the string/regex solver in Z3 uses automata-based reasoning and is significantly slower than arithmetic or bitvector solvers. Avoid large regex constraints; prefer length bounds and simple containment queries.

---

## Conclusion

SMT solvers, and Z3 in particular, are **constraint logic engines** that sit one level above SAT:

- The DPLL(T) architecture cleanly separates boolean search (CDCL) from theory reasoning (simplex, congruence closure, bit-blasting), making it modular and extensible.
- Z3 covers the full spectrum from linear arithmetic and bitvectors (decidable, fast) to quantified first-order logic (undecidable, heuristic).
- Incremental solving via push/pop enables interactive usage patterns like symbolic execution engines that explore paths one by one.
- The SMT-LIB2 format is the universal interface — code targeting it is portable across Z3, CVC5, Yices2, and Bitwuzla.

**Key takeaway:**  
Use Z3 (or any SMT solver) when your constraint problem involves typed variables over arithmetic, bitvectors, or arrays — encodings that would require an exponential number of boolean variables in raw SAT. Stay in quantifier-free fragments of decidable theories (QF_LIA, QF_BV, QF_UF, QF_AUFLIA) to guarantee termination and predictable performance. Reach for quantifiers and nonlinear arithmetic only when necessary, and set timeouts to handle potential non-termination.
