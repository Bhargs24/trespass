# How trespass works

This is the longer version of the [README](../README.md)'s "How it works" — the
actual formal-methods content, for anyone who wants to see the machine.

## The problem, stated precisely

A Postgres row-level-security policy is a boolean expression evaluated once per
row. A row is **visible** to a query when the policy evaluates to `TRUE` — and,
crucially, *only* then. If it evaluates to `FALSE` or to `NULL`, the row is
hidden.

Tenant isolation is the claim: *for every caller and every row, the caller can
see the row only if they are supposed to.* Written as a question a solver can
answer, a **hole** is a model of:

```
        policy(row, session)  is TRUE          -- the database will show it
   AND  intended(row, session)  is not TRUE    -- but the intent forbids it
```

If that conjunction is satisfiable, the satisfying assignment *is* the exploit:
concrete values for the caller and the row that the database will leak. If it is
unsatisfiable, no such caller exists — a proof of isolation.

Everything in `trespass` is in service of asking that one question exactly.

## Why "is not TRUE" and not "is FALSE"

This is the subtlety that most hand-written checks get wrong.

Consider the textbook owner-only policy `user_id = auth.uid()` and an
**anonymous** caller, for whom `auth.uid()` is `NULL`. For any row:

```
user_id = auth.uid()   →   <something> = NULL   →   NULL
```

The policy is `NULL`, so the row is correctly hidden. But the *intended*
predicate for an anonymous caller is also `user_id = auth.uid()`, which is `NULL`
too. If you encoded "the intent forbids it" as `NOT intended` you would get
`NOT NULL = NULL`, and the whole check would evaluate to `NULL` and silently
find nothing.

The fix is to treat "not permitted" as **not `TRUE`** — that is, `FALSE` *or*
`NULL`. `trespass`'s solver takes a set of `deny` formulas that must each
evaluate to something other than `TRUE`, which is exactly this. An anonymous
caller reaching an owner-only row is then caught, because the intent predicate is
`NULL` (not `TRUE`) for them, so any row the policy exposes is a hole.

## The logic fragment

Strip the syntax from real RLS policies and what remains is small:

- **equalities** between columns, session values (`auth.uid()`), literals, and
  JSON claims (`auth.jwt() ->> 'org_id'`);
- **null tests** (`IS NULL`, `IS NOT NULL`);
- **boolean tests** (`IS [NOT] TRUE / FALSE / UNKNOWN`) — two-valued projections
  of a three-valued formula, which is exactly why policies use them on nullable
  boolean columns;
- **null-safe equality** (`IS [NOT] DISTINCT FROM`), desugared into null tests
  and equality;
- **boolean structure** (`AND`, `OR`, `NOT`);
- **`IN` lists**, which are just disjunctions of equalities;
- **scalar subselects with no `FROM` clause** — `(select auth.uid())`, the
  initplan idiom Supabase's docs recommend, evaluates to its inner expression
  and is modeled as such.

That is quantifier-free first-order logic over the theory of equality with
uninterpreted functions (**QF_UF** / EUF), plus an explicit `NULL` and Kleene
evaluation. It is decidable, and the formulas are tiny.

Anything outside the fragment — an inequality, arithmetic, a real subquery — is
not forced into it. A predicate becomes an **opaque atom** the solver may set
freely; an unmodeled *value* becomes a marked term. Two occurrences of the same
source text share one symbol; two different texts never do (sharing a symbol
between different predicates would let `X AND NOT X` fake an isolation proof).

Verdicts then follow one rule. `ISOLATED` is only claimed when the solver
exhausted every model — opaque atoms included, set adversarially. `VULNERABLE`
is only claimed when the counterexample stands on modeled atoms and concrete row
values (a boolean column being `true` is a row that can really exist, and is
reported as an explicit precondition). A counterexample that needs an unmodeled
predicate to hold, or an unmodeled value to take a convenient shape — a subquery
that might be correlated with the caller's session, say — is reported as
`UNKNOWN` instead. This is how the tool stays sound in both directions: it never
proves safety it doesn't have, and never claims an exploit it can't demonstrate.

## The decision procedure

Because the fragment is small, `trespass` does not need a general SMT solver at
runtime. It enumerates the finitely-many *shapes* a model can take:

1. **Null assignment.** Each nullable term is either null or not.
2. **Equality assignment.** Each equality atom is either true or false.
3. **Opaque assignment.** Each opaque atom is true, false, or (if nullable) null.

For each shape:

- A **union-find** merges the terms an "equal" atom joins, then closes under
  **congruence**: if `x` and `y` are equal and non-null, then `f(x)` and `f(y)`
  must be equal *and* share null-ness. (Skipping the null-ness half of that rule
  is a real bug — it was caught by the Z3 differential test, and there is now a
  regression case for it.)
- The shape is rejected if it merges two distinct literals, or violates a decided
  disequality.
- The asserted formulas are evaluated in Kleene logic; if they are all `TRUE`
  and every `deny` formula is not `TRUE`, the shape is a model, returned as a
  witness.

Small-model reasoning makes this **complete**: a formula can only observe its own
atoms, so ranging over their consistent assignments ranges over every model that
could matter. A budget guard turns pathologically large inputs into `UNKNOWN`
rather than a slow wrong answer.

## Kleene evaluation

The three-valued truth tables, for completeness:

```
NOT:  T→F   F→T   N→N

AND:  F if any operand is F;  else N if any is N;  else T
OR:   T if any operand is T;  else N if any is N;  else F

a = b:   N if a or b is null;  else T if same value, F otherwise
a IS NULL:   T if a is null, F otherwise   (always two-valued)
```

Row visibility is `policy evaluates to T`. That single definition, applied
consistently, is what makes the null-handling correct instead of merely
plausible.

## Why validate against Z3 and Postgres

A from-scratch solver is a liability unless it is checked, so it is checked twice
over, by things that were built independently of it:

- **Z3** encodes the same three-valued logic (each node as an `(is_true,
  is_false)` pair of booleans, both false meaning null) and must agree on
  satisfiability for thousands of random formulas. Z3 is a mature, widely-trusted
  solver; agreeing with it on the whole fragment is strong evidence of
  correctness.
- **Postgres** is the ground truth that actually matters. Each canonical policy
  is applied to a real database, seeded with two users' rows, and queried as a
  non-owner. Whatever Postgres does, `trespass` must have predicted. This closes
  the loop from "the logic is internally consistent" to "the logic matches the
  system we are reasoning about."

The result is a tool whose proofs you can believe, because it has been made to
agree with both the theory (Z3) and the reality (Postgres) on every case it
claims to decide.
