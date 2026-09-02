# Changelog

All notable changes to trespass. The format follows
[Keep a Changelog](https://keepachangelog.com/); versions follow
[SemVer](https://semver.org/).

## 0.2.0 — 2026-09-02

The soundness release: every verdict the tool can print is now backed in both
directions, and every install instruction works as written. 460 tests; the
solver is differentially tested against Z3 and the verdicts validated against
a real Postgres in CI. Published to PyPI as `trespass-rls`.

### Fixed
- **False proofs are gone.** `IS TRUE` / `IS FALSE` / `IS DISTINCT FROM` all
  collapsed into one shared opaque atom, so a genuinely leaky policy like
  `... or (is_public is true and not (deleted is true))` encoded as
  `X AND NOT X` and was reported as a *proof of isolation*. Boolean tests are
  now modeled precisely through a new two-valued `IsTrue` node (differentially
  tested against Z3), and `IS [NOT] DISTINCT FROM` desugars into null tests
  plus equality.
- **False alarms on Supabase's own recommended idiom are gone.**
  `(select auth.uid()) = user_id` — the initplan pattern Supabase's docs and
  dashboard linter tell everyone to write — degraded to an opaque value and
  produced a false CRITICAL whose counterexample does not reproduce. A scalar
  subselect with no FROM clause now resolves to its inner expression.
- **Real migration folders parse.** Any operator lexes (`||`, `@>`, `?`, …);
  a clause using syntax outside the grammar (`CASE`, `ARRAY[...]`) degrades
  to one opaque span instead of aborting the whole analysis; `$1` is no
  longer mistaken for a dollar-quote tag.
- **Verdicts only claim what they can prove.** A counterexample resting on an
  unmodeled predicate or value reports as UNKNOWN; a concrete boolean column
  set to true stays a hard VULNERABLE with its precondition stated.
- **Declared intent is validated.** A typo'd table or tenant column warns
  loudly instead of silently disabling every check.
- Schema qualifiers are respected (`auth.users` no longer collides with
  `users`); non-public schemas are not assumed PostgREST-reachable without an
  explicit grant.
- The distribution is `trespass-rls` (the bare PyPI name belongs to an
  unrelated project); the CLI and import name remain `trespass`.

## 0.1.0 — 2026-08-29

Initial release: the from-scratch three-valued solver, the DDL parser, intent
files and inference, the analyzer, and terminal/JSON/SARIF reports.
