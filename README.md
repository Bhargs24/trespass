# trespass

**Prove your tenants can't read each other's data.**

`trespass` is a formal analyzer for Postgres / Supabase [row-level security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html). Point it at your schema and it either **proves** that no user can reach another user's rows, or hands you the **exact query** that shows they can.

It is not a linter and not a pattern-matcher. It compiles every policy into three-valued logic, models what the developer *meant* separately from what the database *enforces*, and asks a solver whether the two can disagree. When they can, the disagreement is a real, reproducible exploit.

```console
$ trespass check schema.sql --intent app.intent

  documents  ·  SELECT  ·  role: authenticated
  ✗ VULNERABLE  critical   [tenant-read]

  Authenticated caller can select another user's rows

  Your policy:
      user_id = auth.uid() OR is_public

  Counterexample (from the solver):
      session  role = authenticated
      session  auth.uid() = attacker
      row      user_id = victim
      row      is_public = true
      holds for any row where `is_public` is true

  Reproduce it:
      select * from documents;
      -- returns rows owned by `victim` to `attacker`

  The intent for `documents` says SELECT is owner. The policy is more
  permissive than that: the solver found a caller who is not the row's owner
  and can still reach it. This holds for any row where `is_public` is true.

  Fix:
      Tighten the policy to owner-only, matching the intent (owner):
        using (user_id = auth.uid())

  ────────────────────────────────────────────────────────────
  1 table  ·  1 policy  ·  1 vulnerable  ·  0 unknown  ·  0 proved
  1 proven way for one user to reach another's data.
```

Exit status is non-zero, so a broken policy fails your CI the way a failing test does.

---

## Why this exists

In 2026, most software is written by people who can't read it. Four out of five people building on tools like Lovable, Bolt and v0 have no engineering background, and independent audits keep finding the same thing: **around 90% of these apps ship with a security flaw, and the single largest class is broken access control** — one user able to read or delete another user's data. In one scan of 1,072 Supabase-backed apps, 172 allowed *unauthenticated deletion* of rows.

Broken access control is also the class every existing scanner **structurally cannot catch**, and the reason is not a lack of effort. Deciding whether user B *should* be able to read user A's row requires knowing who is *supposed* to see what. That is **intent**, and intent is not in the code. A tool that only reads the code inherits the exact blind spot the code was written with: if the agent confidently built the wrong permission model, every code-derived check passes.

`trespass` is built around that gap. It asks you — in six lines of config — who owns each table and who may touch it, then proves whether the policies actually enforce that. It is the one input a code-only tool can never recover on its own.

---

## Install

```bash
pip install trespass          # from PyPI
# or, from source:
pipx install git+https://github.com/bhargavraghavendra/trespass
```

**Zero runtime dependencies.** The solver, the SQL parser, and the report renderer are all written from scratch on the standard library. `git clone && python -m trespass` works on any machine, forever. (Z3 and Postgres are used only to *test* the tool — see [Correctness](#correctness-three-independent-checks).)

Requires Python 3.10+.

---

## Use it

```bash
# analyze a single schema, a migrations folder, or stdin
trespass check schema.sql
trespass check supabase/migrations/
cat schema.sql | trespass check -

# declare intent for hard verdicts (see below)
trespass check schema.sql --intent app.intent

# machine-readable output
trespass check schema.sql --json
trespass check schema.sql --sarif > trespass.sarif   # annotates a GitHub PR

# CI knobs
trespass check schema.sql --fail-on unknown   # also fail on undecided policies
trespass check schema.sql --no-default-grants # ignore Supabase's default grants
```

### Gate your pull requests

Drop this in `.github/workflows/authz.yml` to block a merge that breaks tenant
isolation:

```yaml
name: authz
on: pull_request
jobs:
  trespass:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install trespass
      - run: trespass check supabase/migrations/ --intent app.intent
```

A proven hole exits non-zero and fails the check. Add `--sarif` and upload the
output with `github/codeql-action/upload-sarif` to see each finding inline on the
diff.

### Intent: the part that makes verdicts possible

Without any configuration, `trespass` infers likely ownership from column names (`user_id` looks owned, `org_id` looks tenant-scoped) and runs a conservative pass — enough to catch the unconditional disasters (missing RLS, `using (true)`, anonymous writes), while reporting anything ambiguous as `UNKNOWN` rather than crying wolf.

Declaring intent turns ambiguity into proof. It is an INI file:

```ini
# app.intent
[documents]
tenant = user_id      # each row is owned by the user in this column
select = owner        # only the owner may read
insert = owner
update = owner
delete = owner

[articles]
tenant = author_id
select = public       # anyone may read — on purpose
insert = owner        # but only the author may write
update = owner
delete = owner

[invoices]
tenant  = org_id
identity = jwt:org_id # ownership is the caller's org claim, not their user id
select  = owner
```

The same schema that is `UNKNOWN` under inference becomes a hard `VULNERABLE` (or a proved `ISOLATED`) once you have said what you meant. That is the whole idea: **the gap between declared intent and enforced policy is the vulnerability.**

---

## What it proves

| Verdict | Meaning |
|---|---|
| **`VULNERABLE`** | The solver found a concrete caller and row that breaks the intent. Ships with the session values, the row values, and the SQL to reproduce it. |
| **`ISOLATED`** | The solver proved no such caller exists. This is a proof over the modeled logic, not a clean scan. |
| **`UNKNOWN`** | A policy leaned on something not modeled precisely (a subquery, an inequality). Reported honestly rather than assumed safe. |
| **`INFO`** | Context — a positive isolation proof, a dead policy, a reachability note. |

`UNKNOWN` is a feature. A security tool that turns "I couldn't decide this" into "looks fine" is how scanners lose your trust. `trespass` never reports `VULNERABLE` without a reproduction, and never reports `ISOLATED` without a proof.

The checks, today:

- **RLS disabled on an internet-reachable table** — the classic catastrophe. Unconditional.
- **Tenant read/update/delete** — a caller reaching rows they don't own.
- **Insert / update forgery** — a missing `WITH CHECK` letting a caller create or re-attribute rows to someone else.
- **Anonymous writes** — the `anon` role able to change stored data.
- **Claim confusion** — a "tenant filter" that compares the wrong JWT claim and isolates nothing.
- **Dead policies** — a policy referencing a column that doesn't exist, silently protecting nothing.

---

## How it works

The pipeline is four small stages, each independently testable:

```
 .sql ──▶  parser  ──▶  schema model  ──▶  encoder  ──▶  SMT solver
          (hand-      (tables, RLS,      (policy ×      (proof, or a
           written)    policies,          intent  →      counterexample
                       grants)            3-valued       witness)
                                          formula)
```

Three ideas do the real work:

**1. Three-valued logic, modeled faithfully.** Postgres shows a row only when a policy evaluates to `TRUE` — not `FALSE`, and not `NULL`. That distinction is the source of a whole family of bugs (`user_id = auth.uid()` is `NULL`, not `FALSE`, when `auth.uid()` is null for an anonymous visitor). `trespass` evaluates every policy in [Kleene logic](https://en.wikipedia.org/wiki/Three-valued_logic) so those cases come out right.

**2. Intent as a separate source of truth.** The policy is compiled from the schema; the intent is stated by you. The analyzer asks the solver a single, precise question — *can a policy make a row visible that the intent does not permit?* — using a `deny` constraint that captures "not permitted" as *not `TRUE`* (which is `FALSE` **or** `NULL`), so an anonymous caller reaching an owner-only row is caught even though the ownership check evaluates to `NULL` for them.

**3. A purpose-built solver.** Row-level-security policies live in a tiny, decidable fragment: quantifier-free equality logic with uninterpreted functions, plus `NULL`. `trespass` ships its own decision procedure for exactly that fragment — Boolean enumeration over the finitely-many model shapes, with a union-find congruence closure to keep function applications consistent. The formulas are small enough that this is exact, and small enough that it can be checked against a real SMT solver on thousands of random inputs.

There is a longer write-up in [docs/how-it-works.md](docs/how-it-works.md).

---

## Correctness: three independent checks

A tool that claims to *prove* things has to earn it. `trespass` is validated three ways, and all three run in CI:

1. **Unit tests** pin the semantics — three-valued equality, distinct literals, congruence, and the canonical safe/unsafe policy shapes.
2. **Differential testing against Z3.** The custom solver is the product; [Z3](https://github.com/Z3Prover/z3) is the oracle. Thousands of random three-valued formulas are thrown at both, and any disagreement fails the build. (This is not hypothetical — it caught a real congruence-over-nullness bug during development.)
3. **Dynamic validation against real Postgres.** Every canonical policy is stood up in an actual Postgres instance, seeded with an attacker row and a victim row, and queried as a non-owner role. The tool's verdict must match what Postgres actually does. The counterexamples aren't theoretical — they're confirmed to leak.

```
unit logic  ✓        the rules are what we think they are
vs. Z3      ✓        the solver is correct on 3,000+ random formulas
vs. Postgres ✓       the verdicts match a real database
```

---

## What it does *not* do (yet)

Honesty is the point of the tool, so:

- It models equality-based policies precisely. Inequalities (`<`, `>`), arithmetic, and subqueries (`EXISTS`, `IN (SELECT …)`) become opaque atoms — findings that depend on them are `UNKNOWN`, never a false proof.
- It reasons about a single row at a time, which is the right model for tenant isolation but not for aggregate leaks.
- It assumes a standard Supabase exposure model (PostgREST + the default `anon` / `authenticated` grants). Turn that off with `--no-default-grants` to rely only on grants written in your SQL.
- It reads DDL. It does not connect to your database, and it never needs credentials.

None of these produce false alarms — they produce `UNKNOWN`, which is the tool telling you where it stopped being sure.

---

## Development

```bash
git clone https://github.com/bhargavraghavendra/trespass
cd trespass
pip install -e ".[dev]"

pytest                 # 440+ tests, ~6s (Postgres tests skip without a DSN)
ruff check src tests   # lint
mypy                   # strict type-checking, zero errors

# run the dynamic Postgres validation locally
TRESPASS_PG_DSN=postgresql://postgres:postgres@localhost:5432/postgres \
  pytest tests/test_postgres.py
```

Layout:

```
src/trespass/
  smt/            the from-scratch solver and its term/formula algebra
  sql/            lexer, AST, and a recursive-descent parser for the DDL fragment
  schema.py       the resolved model: tables, RLS state, policies, grants
  encode.py       SQL expression  →  three-valued SMT formula
  intent.py       declared and inferred authorization intent
  analyze.py      the checks: policy × intent  →  findings
  findings.py     verdicts and reproducible witnesses
  report.py       terminal / JSON / SARIF renderers
  cli.py          the command line
```

---

## License

MIT. See [LICENSE](LICENSE).

The name is the whole thesis: this tool exists to find out who can **trespass** into whose data — and to prove, when the answer is nobody, that it really is nobody.
