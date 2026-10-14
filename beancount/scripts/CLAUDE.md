# beancount/scripts/ — command-line programs

Each CLI is a `click` command in a module here, wired in `pyproject.toml [project.scripts]` (and
mirrored by the launchers in `bin/`). The scripts only orchestrate: the logic lives in `loader`,
`parser`, `core` and `ops`.

| Command | Module | What it does |
|---|---|---|
| `bean-check FILE` | `check.py` | full load (parse, book, plugins, validation); prints the errors, exit status 1 if there are any, else 0. Flags: `-v` timings, `-C/--no-cache`, `--cache-filename`, `-a/--auto` (adds `plugins.auto`), `--json` (errors as JSON) |
| `bean-doctor <cmd>` | `doctor.py` | debugging toolbox (click group; command names use dashes): `lex` (alias `dump-lexer`), `parse`, `roundtrip` (print + re-parse must be equal), `directories` (document folders vs accounts), `list-options`, `print-options`, `context FILE LOCATION` (balances before/after an entry), `linked` (entries linked/tagged with one), `region` (balances of a line range), `missing-open`, `display-context` |
| `bean-example` | `example.py` | generates a realistic multi-year ledger (salary, taxes, retirement, investments, expenses) — used for demos and as a large test input |
| `bean-format FILE` | `format.py` | re-aligns amounts and currencies in a ledger (text-level, no load) |
| `treeify` | `../tools/treeify.py` | renders a column of hierarchical names as a tree (stand-alone tool) |

Helpers: `deps.py` (`check_dependencies` — versions of installed dependencies, a diagnostic
library with no command of its own), `directories.py` (document-folder validation, used by
`bean-doctor directories`).

## Changing or adding a CLI

- Keep `main` as the click entry point; a new command needs `[project.scripts]`, a `bin/` launcher,
  and the module + its test in `meson.build`.
- Tests (`*_test.py`) use `test_utils.ClickTestCase` (`self.run_with_args(main, ...)`) and
  `@test_utils.docfile` to pass a temporary ledger written from the docstring.
- `bean-check` honours the loader pickle cache: pass `-C` when testing loader or plugin changes on a
  real ledger.
- VS Code: `.vscode/launch.json` has debug configurations for bean-check / bean-doctor parse /
  bean-format on the current file.
