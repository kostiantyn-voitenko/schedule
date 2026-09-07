# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

`schedule` — "Python job scheduling for humans." A zero-dependency, in-process periodic job scheduler with a fluent builder API (`schedule.every(10).minutes.do(job)`). The entire library lives in a single module, `schedule/__init__.py` (~950 lines). All tests live in a single file, `test_schedule.py`. Supports Python 3.7–3.12; the optional `pytz` extra enables timezone-aware `.at(..., tz=...)`.

## Commands

**Use `.venv311/` (Python 3.11) for all development.** It has the full `requirements-dev.txt` toolchain installed and matches CI's `format`/`docs` jobs. Prefix commands with `.venv311/bin/` or activate it. The older `.venv/` (Python 3.13) only has bare pytest and is useful solely as a "does 3.13 still import and pass the non-tz tests" smoke check.

Two environment traps, both verified:
- `requirements-dev.txt` **cannot be installed on Python ≥ 3.12**: `black==20.8b1` requires `typed-ast`, which has no wheels past cp311 and does not build on newer Pythons. Recreate the venv from `/opt/homebrew/opt/python@3.11/bin/python3.11` if needed.
- `pytest-flake8` (in `requirements-dev.txt`) is **incompatible with pytest ≥ 9** and crashes pytest at plugin load with `PluginValidationError ... Argument(s) {'path'}` even when `--flake8` is not passed. It is uninstalled from `.venv311`; if it reappears, `pip uninstall pytest-flake8` or run pytest with `-p no:flake8`. CI never uses it; the `--flake8` flag in `docs/development.rst` is stale — use the `tox.ini` command below.

```bash
pip install -r requirements-dev.txt       # dev tooling (pins black==20.8b1 and click==8.0.4 — keep them); Python 3.11 only

# Tests (what tox/CI runs)
py.test test_schedule.py schedule -v --cov schedule --cov-report term-missing
python -m pytest test_schedule.py -k test_tz_daily_dst     # single test / pattern
python -m pytest test_schedule.py::SchedulerTests::test_at_time

# Type check (CI runs this after tests; setup.cfg scopes mypy to the schedule package)
python -m mypy -p schedule --install-types --non-interactive

# Formatting — must be black 20.8b1 exactly, CI runs `black --check .`
black .

# Docs (Sphinx, warnings are errors in CI)
cd docs && make html                       # output in docs/_build/html

# Full matrix locally
tox                                        # py3{7..12}{,-pytz}
tox -e format / docs / setuppy
```

**pytz matters for test coverage.** The tox matrix runs every Python version twice, with and without `pytz`. Timezone tests call `self.make_tz_mock_job()` which skips when pytz is missing — in a venv without pytz, 41 of 81 tests silently skip. The expected healthy result in `.venv311` is **81 passed, 0 skipped, 99% coverage**; treat any skips as a broken environment, not a pass.

## Architecture

Two classes plus module-level shortcuts:

- **`Scheduler`** — owns `self.jobs: List[Job]`. `run_pending()` runs jobs whose `should_run` is true, in `next_run` order (jobs sort via `__lt__` on `next_run`). Intentionally does *not* catch up missed runs. `_run_job` cancels a job when its return value is `CancelJob` (class or instance).
- **`Job`** — a builder. `every(n)` creates an unconfigured `Job`; chained properties (`.seconds`, `.monday`, `.at()`, `.to()`, `.until()`, `.tag()`) mutate and return `self`; `.do(func, *args, **kwargs)` finalizes it: wraps func in `functools.partial`, calls `_schedule_next_run()`, and appends to the scheduler. Singular units (`.minute`, `.day`, weekdays) raise `IntervalError` unless `interval == 1`. Weekday properties set `start_day` and force `unit = "weeks"`.
- **Module-level API** (`schedule.every`, `run_pending`, `clear`, `repeat` decorator, ...) delegates to a global `default_scheduler`. Tests call `schedule.clear()` in `setUp` because of this shared state.

### Next-run computation (the tricky part)

`Job._schedule_next_run()` is where nearly all bugs and issues land (see `test_tz_daily_issue_*` tests). Key invariants:

- All arithmetic happens in `self.at_time_zone` (a pytz tz or `None`), then the result is converted back to a **naive local datetime** for backward compatibility. `next_run` and `last_run` are always naive local.
- `_move_to_at_time` replaces time components according to unit (`days` sets H:M:S, `hours` sets M:S, `minutes` sets S).
- `_correct_utc_offset(moment, fixate_time)` handles DST transitions: it normalizes via pytz and, when `fixate_time` is true (an `.at()` time was given), re-pins the wall-clock time; if that lands in a DST gap it schedules one offset later.
- `.at()` parsing is format-validated per unit with regexes (`HH:MM[:SS]` for days/weekdays, `[MM]:SS` for hours, `:SS` for minutes).
- `.until()` cancellation is checked twice in `Job.run()`: before running (if already overdue) and after rescheduling (if the *next* run would be overdue).

## Testing conventions

- Tests use `unittest.TestCase` under pytest with plain `assert`s.
- **Time is controlled with `mock_datetime(...)`**, a context manager in `test_schedule.py` that monkey-patches `datetime.datetime` (`now()`/`today()`) and can set `os.environ["TZ"]` + `time.tzset()` for a given POSIX TZ string (`TZ_BERLIN`, `TZ_AUCKLAND`, `TZ_CHATHAM`, `TZ_UTC` constants). The module sets the process TZ to Berlin at import for reproducibility. New time-dependent tests should follow this pattern rather than sleeping.
- `make_mock_job()` returns a `mock.Mock` with a `__name__` so `Job.__str__`/`__repr__` work.
- Timezone tests must go through `self.make_tz_mock_job()` so they skip cleanly in the no-pytz tox envs.

## Releasing

Documented in `docs/development.rst`: update `HISTORY.rst` and `AUTHORS.rst`, bump the version in **both** `setup.py` (`SCHEDULE_VERSION`) and `docs/conf.py`, merge to master, then tag `X.Y.Z` and upload with `twine`. Semantic versioning.
