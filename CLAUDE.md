# CLAUDE.md — Beancount (working copy)

Beancount is a double-entry accounting system that reads a plain-text ledger, turns it into
immutable Python directives, books and validates them, and exposes them to reports, queries and
scripts. This repository is a copy of upstream https://github.com/beancount/beancount (v3,
`beancount/VERSION` = 3.2.3), set up as a VS Code Python project. License: GPL-2.0-only.

Area guides (read the one for the code you touch):

| Path | What it covers |
|---|---|
| [beancount/CLAUDE.md](beancount/CLAUDE.md) | package entry points: `api.py`, `loader.py`, the load pipeline |
| [beancount/core/CLAUDE.md](beancount/core/CLAUDE.md) | data model: directives, Amount, Position, Inventory, prices, realization |
| [beancount/parser/CLAUDE.md](beancount/parser/CLAUDE.md) | C lexer/grammar, Builder, options, booking, printer |
| [beancount/ops/CLAUDE.md](beancount/ops/CLAUDE.md) | built-in transformations (pad, balance, documents) and validation |
| [beancount/plugins/CLAUDE.md](beancount/plugins/CLAUDE.md) | plugin contract and the bundled plugins |
| [beancount/scripts/CLAUDE.md](beancount/scripts/CLAUDE.md) | `bean-check`, `bean-doctor`, `bean-example`, `bean-format` |
| [beancount/utils/CLAUDE.md](beancount/utils/CLAUDE.md) | generic helpers and the test utilities |

The existing `docs.md` files in each folder are upstream's general overviews; the CLAUDE.md files
focus on contracts, invariants and how to change the code safely.

## Setup (Windows, done once)

Requires Python >= 3.9 (the venv uses 3.13) and a C compiler (MSVC from Visual Studio 2022
"Desktop development with C++"; meson finds it without a developer prompt).

```powershell
python -m venv .venv
.venv\Scripts\python -m pip install meson-python meson ninja "winflexbison-bin>=2.5.25.1" pytest ruff mypy types-python-dateutil types-regex
$env:PATH = "$PWD\.venv\Scripts;$env:PATH"          # win_bison / win_flex / meson / ninja
.venv\Scripts\python -m pip install --no-build-isolation -Cbuild-dir=build -Csetup-args=-Dtests=enabled -e .
```

The editable install compiles the C parser into `build/` and **rebuilds it on import** whenever
`lexer.l`, `grammar.y` or the `.c/.h` files change — keep the venv `Scripts` folder on PATH (the
VS Code settings and tasks already do). `--no-build-isolation` is required for editable
meson-python installs.

## Everyday commands

| Task | Command |
|---|---|
| all Python tests | `.venv\Scripts\python -m pytest beancount` (~30 s, ~1100 tests) |
| one module | `.venv\Scripts\python -m pytest beancount/core/inventory_test.py` |
| C tests | `.venv\Scripts\meson test -C build` |
| check a ledger | `.venv\Scripts\bean-check examples\example.beancount` |
| lint / format | `ruff check beancount`, `ruff format beancount` (config: `pyproject.toml [tool.ruff]`) |
| types | `mypy .` (`[tool.mypy]`) |

VS Code: `.vscode/` has the interpreter, pytest discovery, launch configs (bean-check / bean-doctor /
bean-format on the current file, pytest on the current file) and tasks for install, tests, lint, types.
Upstream CI (`.github/workflows/tests.yaml`) runs the same install + `pytest beancount` +
`meson test` on Linux, macOS and Windows.

## How the system fits together

```
text ledger
  -> parser (C extension _parser: lexer.l + grammar.y)  calls  grammar.Builder  -> incomplete directives
  -> loader: include files, merge options, sort by data.entry_sortkey
  -> parser.booking.book: match lots + interpolate missing numbers (CostSpec -> Cost, MISSING filled)
  -> loader.run_transformations: plugins in order
         ops.documents  ->  user `plugin` directives  ->  auto plugins (--auto)  ->  ops.pad, ops.balance
  -> ops.validation.validate
  -> (entries, errors, options_map)   consumed by core.realization, core.prices, scripts, external tools
```

## Repository layout

| Path | Contents |
|---|---|
| `beancount/` | the installed package (only this is shipped; see `meson.build`) |
| `bin/` | thin launcher scripts mirroring `[project.scripts]` |
| `examples/` | sample ledgers (`example.beancount`, tutorial, vesting) and an ingest example layout |
| `docs/` | documentation site sources |
| `tools/` | maintainer scripts (benchmarks, dependency checks, doc conversion) — not installed |
| `experiments/` | prototypes (duckdb, sql, visidata, docs exporters) — not installed, not tested by CI |
| `meson.build`, `meson.options` | build definition for the package and the C extension |

## Conventions that matter

- **Tests live next to the code** as `*_test.py` (pytest, `unittest.TestCase` style). Most tests
  write the ledger in the test's docstring: `@parser.parse_doc()` (parse only) or
  `@loader.load_doc()` (full pipeline), then compare with `cmptest.TestCase.assertEqualEntries`.
- **Every installed file is listed in `meson.build`** (`py.install_sources`), tests included. A new
  module or test file that is not listed works in the editable install but is missing from wheels.
- Directives are immutable `NamedTuple`s; change them with `._replace(...)`, never in place.
- Numbers are `decimal.Decimal` everywhere (`core.number.D`); never floats.
- Python code is formatted with `ruff format`; imports one per line (`[tool.ruff.lint.isort]
  force-single-line`).
- Every source file carries `__copyright__` / `__license__ = "GNU GPLv2"` headers.

## Git remotes

- `origin` — this copy (github.com/diqp2001/beancount)
- `upstream` — github.com/beancount/beancount; sync with `git fetch upstream` and
  `git merge upstream/master`.
