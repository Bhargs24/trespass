# Contributing

Thanks for looking. `trespass` is small on purpose and easy to hack on.

## Setup

```bash
pip install -e ".[dev]"
pytest && ruff check src tests && mypy
```

## The bar

Every change keeps three things true:

1. **The runtime stays dependency-free.** The solver and parser use only the
   standard library. Z3 and Postgres are test-only.
2. **New logic comes with a proof it is right.** If you extend the solver,
   extend `tests/test_solver_differential.py` so Z3 checks it. If you add a
   check, add a vulnerable *and* a safe example under `examples/` — they are
   executed as tests.
3. **Findings never lie.** A `VULNERABLE` verdict must carry a reproducible
   witness. When you are unsure, the verdict is `UNKNOWN`, not a guess.

## Good first issues

- More policy patterns in `examples/` (both directions).
- Widening the modeled fragment (e.g. `IS DISTINCT FROM`) — with matching
  differential coverage.
- A `--fix` mode that rewrites a leaky policy to match the intent.
