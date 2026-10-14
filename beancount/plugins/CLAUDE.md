# beancount/plugins/ — bundled plugins

A ledger enables a plugin with `plugin "beancount.plugins.<module>"` (optionally `plugin "module"
"config string"`). The loader imports the module and runs it between the built-in `documents` and
`pad`/`balance` steps (see `beancount/CLAUDE.md`).

## Contract

```python
__plugins__ = ("my_function",)            # names (or function objects), run in order

def my_function(entries, options_map, config=None):   # config only when the directive has one
    ...
    return new_entries, errors            # always both; errors may be []
```

- Receive booked, complete, sorted entries; return the full list (not a diff). The loader re-sorts
  after each plugin, but keep it sorted anyway.
- Never mutate directives; build new ones with `._replace`. New entries need meta
  (`data.new_metadata(filename, lineno)` — reuse the source entry's meta where possible).
- Errors are namedtuples `(source, message, entry)`, defined per module
  (`collections.namedtuple("LeafOnlyError", "source message entry")`).
- An exception inside a plugin becomes a `LoadError` and the load continues without that plugin's
  output; an `ImportError` likewise. Only `SystemExit` stops the load.
- `loader.combine_plugins(mod_a, mod_b, ...)` concatenates `__plugins__` tuples — this is how the
  meta-plugins below are built.

## Bundled plugins

Meta-plugins:
- `auto` = `auto_accounts` + `implicit_prices` (lax mode; enabled by `bean-check --auto`).
- `pedantic` = `check_commodity`, `coherent_cost`, `leafonly`, `noduplicates`, `nounused`,
  `onecommodity`, `sellgains`, `unique_prices`, `check_drained`.

Transformations (add entries):
| Module | Effect |
|---|---|
| `auto_accounts` | inserts `Open` for accounts used without one |
| `implicit_prices` | synthesizes `Price` directives from posting prices and costs |
| `check_closing` | posting meta `closing: TRUE` → zero `Balance` assertion for that currency the next day |
| `check_drained` | zero `Balance` assertion (per currency used) the day after each balance-sheet account `Close` |
| `close_tree` | closing an account also closes its descendants |
| `currency_accounts` | currency-trading postings so that every currency balances (config: base account, e.g. `"Equity:CurrencyAccounts"`) |

Validations (return errors only):
| Module | Checks |
|---|---|
| `check_average_cost` | NONE-booking accounts keep their average cost basis on reductions |
| `check_commodity` | every currency used has a `Commodity` directive (config: ignore map) |
| `coherent_cost` | a currency is either held at cost or converted at price, not both |
| `commodity_attr` | `Commodity` directives have the configured attributes with allowed values |
| `leafonly` | no postings to accounts that have children |
| `noduplicates` | no identical entries (stable hash without meta) |
| `nounused` | every opened account is used |
| `onecommodity` | one commodity per account (opt out with `onecommodity: FALSE` on the account's `Open`) |
| `sellgains` | declared gains on lot sales match proceeds minus cost |
| `unique_prices` | at most one price per (date, base, quote) |

## Adding a plugin

1. `beancount/plugins/<name>.py` with copyright header, module docstring (shown in docs),
   `__plugins__`, and an error namedtuple.
2. `<name>_test.py` using `@loader.load_doc(expect_errors=...)` with a ledger that enables the
   plugin in its docstring.
3. Add both files to `meson.build` (`py.install_sources` lists, module and tests sections).
4. If it should be part of `pedantic` or `auto`, add it to that `combine_plugins` call.
