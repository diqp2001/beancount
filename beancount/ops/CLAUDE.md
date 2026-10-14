# beancount/ops/ — built-in transformations and validation

Functions over lists of directives. Three modules are **plugins run on every load** (see the loader
order in `beancount/CLAUDE.md`); the rest are libraries used by plugins, scripts and external tools.

## Run by the loader

| Module | `__plugins__` | When | What |
|---|---|---|---|
| `documents.py` | `process_documents`, `verify_document_files_exist` | first (`PLUGINS_PRE`) | creates `Document` directives from the `option "documents"` folders (files named `YYYY-MM-DD.name` under account-shaped paths), checks that declared documents exist |
| `pad.py` | `pad` | after user plugins (`PLUGINS_POST`) | turns each `Pad` into a padding transaction (flag `P`) so that the next `Balance` of the account holds; `PadError` for unused pads |
| `balance.py` | `check` | last plugin | checks every `Balance` against the running balance at the start of its date, within `get_balance_tolerance`; failing balances get `diff_amount` set and a `BalanceError` |
| `validation.py` | — | after all plugins (`validate`) | `BASIC_VALIDATIONS`: open/close consistency, postings only to active accounts, currency constraints from `Open`, duplicate balances / commodities, document paths, every transaction balances |

Because pad and balance are plugins, `plugin_processing_mode "raw"` disables them, and a user
plugin that runs before them sees unpadded entries.

**Gotcha (upstream)**: `validation.validate` does `validation_tests = VALIDATIONS;
validation_tests += extra_validations`, which extends the module-level list in place, so
`extra_validations` accumulate across loads in one process. Pass a copy if you rely on it.
`HARDCORE_VALIDATIONS` (`validate_data_types`) is defined but not part of `VALIDATIONS`.

## Libraries

| Module | Use |
|---|---|
| `summarize.py` | period reports: `open` (summarize history into opening balances), `close` (move income/expenses to equity), `clear`, `clamp` (= open + close over a date range), `cap`, `truncate`, `conversions`, `transfer_balances`, `balance_by_account` |
| `basicops.py` | `filter_tag`, `filter_link`, `group_entries_by_link`, `get_common_accounts` |
| `compress.py` | merge many small transactions into one (`compress`, `merge`) — import-time helper |
| `find_prices.py` | which currencies need prices (held at cost, converted, priced) — feeds price fetchers |
| `lifetimes.py` | per-commodity holding intervals; `required_daily_prices` / `required_weekly_prices` |

## Changing things

- A new automatic transformation belongs in a plugin, not here, unless every ledger needs it (then
  it must also be added to `loader.PLUGINS_PRE` / `PLUGINS_POST` — a behaviour change for all users).
- Error objects are namedtuples `(source, message, entry)`; `source` is a meta dict (filename,
  lineno) so `printer.format_error` can point at the line.
- Tests: `pad_test.py`, `balance_test.py`, `validation_test.py` use `@loader.load_doc(...)` with the
  ledger in the docstring, so they exercise the real pipeline order.
