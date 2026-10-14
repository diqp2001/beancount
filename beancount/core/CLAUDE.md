# beancount/core/ — the data model

Pure Python, no I/O. Everything here operates on lists of directives (`data.Directives`) and is
used by every other package. Read `data.py` first.

## data.py — directives

All directives are `typing.NamedTuple`s whose first two fields are `meta` and `date`:

| Type | Main fields |
|---|---|
| `Open` | account, currencies (constraint list), booking (`Booking` enum or None) |
| `Close` | account |
| `Commodity` | currency |
| `Pad` | account, source_account |
| `Balance` | account, amount, tolerance, diff_amount (set by `ops.balance` on failure) |
| `Transaction` | flag, payee, narration, tags (frozenset), links (frozenset), postings (list) |
| `Note`, `Event`, `Query`, `Price`, `Document`, `Custom` | see the class docstrings |

`Posting(account, units: Amount, cost: Cost | CostSpec | None, price: Amount | None, flag, meta)`.
`TxnPosting(txn, posting)` pairs a posting with its transaction (used by realization).

- `meta` is a dict that always has `filename` and `lineno` (`new_metadata(filename, lineno)`);
  user metadata keys are added to it. Postings may have `meta=None`.
- `Booking`: STRICT (default, option `booking_method`), STRICT_WITH_SIZE, NONE, AVERAGE, FIFO,
  LIFO, HIFO.
- **Ordering**: `entry_sortkey = (date, SORT_ORDER, lineno)` with
  `SORT_ORDER = {Open: -2, Balance: -1, Document: 1, Close: 2}` — on a given date, Open comes first,
  then Balance assertions (a balance checks the state at the *start* of the day), then everything
  else, then Documents, then Close. Code that builds entries must keep the list sorted
  (`data.sorted`).
- Immutability: use `entry._replace(...)`; postings lists are shared — copy before mutating.

## Numbers and amounts

- `number.py`: `D(str)` builds a `Decimal`; `ZERO`, `ONE`, `HALF`; `MISSING` is the sentinel the
  parser puts in elided numbers (it must never survive booking). Never use floats.
- `amount.py`: `Amount(number, currency)` plus `add`, `sub`, `mul`, `div`, `abs`; arithmetic only
  between equal currencies.
- `position.py`: `Cost(number, currency, date, label)` (a booked lot) vs `CostSpec(number_per,
  number_total, currency, date, label, merge)` (what the user wrote, possibly partial).
  `Position(units: Amount, cost: Cost | None)`.
- `inventory.py`: `Inventory` — a list of positions keyed by (currency, cost); `add_amount`,
  `add_position` return a `MatchResult` (CREATED / REDUCED / AUGMENTED / IGNORED); `reduce(reducer)`
  maps positions with the `convert` functions; `check_invariants` for tests.
- `convert.py`: the four views of a position — `get_units`, `get_cost` (cost basis),
  `get_weight` (what the posting contributes to the transaction balance: cost if any, else price,
  else units) and `get_value` (market value from a `PriceMap`).

## Balancing and tolerances

`interpolate.py`: `compute_residual(postings)` sums weights; `infer_tolerances(postings,
options_map)` derives per-currency tolerances from the precision of the numbers written (options
`inferred_tolerance_default`, `inferred_tolerance_multiplier`, `infer_tolerance_from_cost`);
`get_residual_postings` creates rounding postings (option `account_rounding`). A transaction
balances when every currency's residual is within its tolerance.

## Accounts

- `account.py`: account names are `:`-separated strings (`Assets:US:Bank`); helpers `join`, `split`,
  `parent`, `leaf`, `root`, `has_component`, `walk`, `AccountTransformer`.
- `account_types.py`: the five roots come from the options (`name_assets`, ... — renameable);
  `get_account_type`, `is_balance_sheet_account`, `get_account_sign` (+1 Assets/Expenses, -1
  Liabilities/Equity/Income).

## Reporting structures

- `realization.py`: `realize(entries)` builds a tree of `RealAccount` (a dict subclass of child
  accounts with `.account`, `.txn_postings`, `.balance` Inventory); `compute_balance`,
  `iterate_with_balance`, `dump_balances`.
- `prices.py`: `build_price_map(entries)` → `PriceMap` keyed by (base, quote) with sorted
  (date, price) lists; each pair is also available inverted (when both directions were declared,
  the one with fewer points is converted into the other); `get_price(price_map, (base, quote), date)`
  returns the latest price at or before `date`; `get_latest_price`.
- `getters.py`: collect accounts, tags, links, payees, currencies, open/close per account, date
  ranges from a list of entries.
- `compare.py`: stable hashes of entries (`hash_entry`, `exclude_meta`), `compare_entries`,
  `includes_entries` / `excludes_entries` — used by tests and `plugins.noduplicates`.
- `display_context.py`: tracks number precision per currency to render aligned output
  (`options_map["dcontext"]`).
- `flags.py`: one-character flags (`*` okay, `!` warning, `P` padding, `S` summarize, ...).

## Tests

`*_test.py` next to each module; pure unit tests, no files. `inventory_test.py`, `data_test.py` and
`interpolate_test.py` are the best references for expected semantics.
