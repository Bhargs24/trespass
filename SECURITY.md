# Security policy

## Reporting

Please report vulnerabilities privately via
[GitHub security advisories](https://github.com/Bhargs24/trespass/security/advisories/new)
rather than a public issue. You will get an acknowledgement within a week.

A **wrong verdict is a security bug** here: a policy trespass calls ISOLATED
that a real Postgres leaks on, or a VULNERABLE whose reproduction does not
reproduce, deserves the same private report as a code vulnerability — ideally
with the schema and intent file that show it.

## Scope notes

- trespass reads DDL text. It never connects to a database, needs no
  credentials, and the runtime has zero dependencies.
- Z3 and Postgres are used only by the test suite, to validate the solver and
  the verdicts.
