# beancount/ — package entry points and the load pipeline

## Public surface

- `beancount/__init__.py` reads `VERSION` into `__version__` and re-exports everything from
  `api.py` (`import beancount as bn; bn.load_file(...)`).
- `api.py` is the v3 public API: a curated `__all__` of directive types (`bn.Transaction`, ... via
  `core.data.dtypes`), `Amount`, `Position`, `Cost`, `CostSpec`, `Inventory`, `D`, `ZERO`,
  `PriceMap` / `build_price_map` / `get_price`, `realize`, the conversion helpers (`get_units`,
  `get_cost`, `get_weight`, `get_value`), account-type predicates, `load_file` / `load_doc` /
  `load_encrypted_file` and the printers. **Adding a symbol here is an API commitment** — upstream
  promises not to remove symbols on v3.

## loader.py — the one place that produces a usable ledger

```
load_file(filename, log_timings=None, log_errors=None, extra_validations=None, encoding=None)
load_string(string, ...)        load_encrypted_file(...)        (all return entries, errors, options_map)
```

`_load(sources, ...)` does, in order:

1. `_parse_recursive` — parse each source with `parser.parse_file/parse_string`, follow `include`
   directives (recursive glob patterns, resolved from the including file's directory; a glob with
   no match is an error), skip files already parsed ("Duplicate filename parsed" — this also
   breaks include cycles), merge the per-file options (`aggregate_options_map`). Then
   `entries.sort(key=data.entry_sortkey)`.
2. `parser.booking.book(entries, options_map)` — lot matching + interpolation; errors appended.
3. `run_transformations` — plugins, with `options_map["pythonpath"]` temporarily prepended to
   `sys.path`:
   - `plugin_processing_mode = "default"`: `PLUGINS_PRE` (`ops.documents`) → the ledger's own
     `plugin "..."` directives in file order → `PLUGINS_AUTO` (filled only by `--auto`, i.e.
     `plugins.auto`) → `PLUGINS_POST` (`ops.pad`, `ops.balance`).
   - `"raw"`: only the ledger's plugins — pad/balance/documents do not run unless listed.
   - each plugin module must expose `__plugins__`; each function is called as
     `fn(entries, options_map)` or `fn(entries, options_map, config_string)` and returns
     `(entries, errors)`. Import errors and exceptions become `LoadError`s (the load continues);
     `SystemExit` propagates. Entries are re-sorted after every plugin.
4. `ops.validation.validate(entries, options_map, log_timings, extra_validations)`.
5. `options_map["input_hash"]` = hash of all included files.

### Caching

`load_file` is wrapped by `pickle_cache_function` (set up by `initialize(use_cache, cache_filename)`):
the result is pickled to `.{filename}.picklecache` next to the ledger when the load took longer than
`PICKLE_CACHE_THRESHOLD` (1 s), and reused while `compute_input_hash` (mtime and size of every
included file) is unchanged (`needs_refresh`). Editing Beancount's own code or a plugin does not
invalidate it. `bean-check --no-cache` / `-C` disables it. When debugging "my change has no
effect", delete the `.picklecache` file first.

### Test helper

`load_doc(expect_errors=False)` decorates a test method: the method's docstring is loaded as a
ledger and the method receives `(self, entries, errors, options_map)`. `expect_errors=True` fails
when there are no errors, `None` skips the check. See `beancount/utils/CLAUDE.md` for the others.

## Where to change things

| Want to... | Go to |
|---|---|
| a new directive / field | `core/data.py`, then the grammar (`parser/grammar.y` + `grammar.Builder`), printer, tests |
| a new option | `parser/options.py` (`OPTION_GROUPS`), then whoever reads `options_map` |
| a transformation on every load | a plugin (`plugins/`), not a loader change |
| a new check | a plugin returning errors, or an `extra_validations` function |
| a new CLI | `scripts/` + `[project.scripts]` in `pyproject.toml` + `meson.build` + `bin/` |
