# beancount/parser/ — syntax, options, booking, printing

## Parsing: C extension + Python builder

```
lexer.l (flex)  +  grammar.y (bison)  +  parser.c / tokens.c / decimal.c   ==>  _parser extension module
       |  grammar actions call methods on a Python object
       v
grammar.Builder (grammar.py)  -> creates core.data directives, collects errors and options
       |
parser.py: parse_file / parse_string / parse_many / parse_one / parse_doc  -> (entries, errors, options_map)
```

- `_parser` is built by meson (`meson.build`: `py.extension_module('_parser', ...)` from
  `decimal.c`, `parser.c`, `tokens.c` + the flex/bison outputs). `_parser.pyi` is its type stub.
  - `parser.c`: the Python module glue (Parser object, `parse()` entry).
  - `tokens.c`: token value conversion (e.g. `validate_decimal_number`), unit-tested in C by
    `tokens_test.c` (`meson test -C build`).
  - `decimal.c`: cached reference to the `Decimal` type; `macros.h`: helper macros (MSVC-safe).
- `grammar.Builder` methods (`open`, `close`, `balance`, `transaction`, `posting`, `cost_spec`,
  `amount`, `option`, `include`, `plugin`, `pushtag` / `poptag`, `pushmeta` / `popmeta`, ...) are
  the grammar's semantic actions. Changing syntax means changing **both** `grammar.y` and the
  matching Builder method (argument order is positional).
- `lexer.py`: `lex_iter(file)` / `lex_iter_string` expose the token stream (used by
  `bean-doctor lex`).
- **Source hash check**: `hashsrc.hash_parser_source_files()` hashes the C/flex/bison sources;
  meson compiles it into the module (`-DPARSER_SOURCE_HASH`), and importing `parser.py` warns
  "The Beancount parser C extension module is out-of-date" when they differ. With the editable
  install the module is rebuilt on import, so the warning means the rebuild did not happen (venv
  `Scripts` not on PATH, or a non-editable install).
- Windows: the build defines `YY_NO_UNISTD_H`; bison/flex come from `winflexbison-bin`
  (`win_bison`, `win_flex`). Bison runs with `-Wall -Werror`: grammar conflicts fail the build.

### The parser's output is incomplete on purpose

Elided parts become `core.number.MISSING`, and costs are `CostSpec` (what was written), not `Cost`:

```
Assets:Account                 -> posting.units = MISSING
Assets:Account     USD         -> posting.units = Amount(MISSING, "USD")
Assets:Stock  10 HOOL {USD}    -> cost = CostSpec(MISSING, None, "USD", ...)
```

`is_posting_incomplete` / `is_entry_incomplete` detect this. Only booking produces complete entries;
never feed parser output directly to reports — use the loader.

## options.py

`OPTION_GROUPS` declares every `option "name" "value"` with its default and converter
(`options_validate_*`). Notable: `booking_method` (default `STRICT`), `operating_currency`,
`name_assets/liabilities/equity/income/expenses` (root names), `inferred_tolerance_default`,
`inferred_tolerance_multiplier`, `infer_tolerance_from_cost`, `account_rounding`,
`plugin_processing_mode` (`default` | `raw`), `documents`, `insert_pythonpath`. Read-only
computed entries: `filename`, `include`, `input_hash`, `dcontext`, `commodities`.
`get_account_types(options_map)` returns the root names. `bean-doctor list-options` prints them all.

## Booking (called by the loader after parsing)

- `booking.py` — `book(entries, options_map)`: per-account booking method from `Open.booking`
  (else `options_map["booking_method"]`), calls `booking_full.book`, then
  `validate_missing_eliminated` (no `MISSING` may remain). `validate_inventory_booking` exists
  but is not called by `book`.
- `booking_full.py` — the algorithm (its module docstring explains why): per transaction,
  1. `categorize_by_currency` groups postings by the currency they weigh in, inferring missing
     currencies;
  2. `book_reductions` matches reducing postings against the running inventory of each account
     using the account's method; augmentations keep their `CostSpec`;
  3. `interpolate_group` fills at most one missing number per currency group, converts remaining
     `CostSpec`s to `Cost` (a lot without an explicit date gets the transaction date).
  Errors: `SelfReduxError`, `CategorizationError`, `ReductionError`, `InterpolationError`.
- `booking_method.py` — one function per `Booking` value (`booking_method_STRICT`, `FIFO`, `LIFO`,
  `HIFO`, `AVERAGE`, `NONE`, `STRICT_WITH_SIZE`), dispatched through `_BOOKING_METHODS`. A new
  method = a function here + an enum value in `core/data.py` + an entry in `_BOOKING_METHODS`,
  with tests in `booking_method_test.py`.

## Output

- `printer.py`: `format_entry(entry, dcontext)`, `print_entry`, `print_entries`, `format_error`,
  `print_errors`. Output must re-parse to equal entries (round-trip; `bean-doctor roundtrip`
  checks it on a file).
- `context.py`: `render_entry_context` — balances before/after an entry (`bean-doctor context`).

## Tests

- `@parser.parse_doc(expect_errors=False, allow_incomplete=False)` decorates a test; the docstring
  is parsed (not booked) and the test receives `(self, entries, errors, options_map)`. Most `grammar_test.py` / `booking_*_test.py`
  cases use it.
- `cmptest.TestCase.assertEqualEntries(expected, actual)` compares entries (expected may be a
  ledger string); `assertIncludesEntries` / `assertExcludesEntries`.
- `booking_full_test.py` has table-driven cases (input, expected booked output) — follow that style.
