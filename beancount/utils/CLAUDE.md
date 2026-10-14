# beancount/utils/ — generic helpers and test support

No accounting logic here; nothing in `utils` imports `core`, `parser` or `ops` business code.

| Module | Use |
|---|---|
| `test_utils.py` | test support used across the repo (below) |
| `misc_utils.py` | `log_time` (context manager behind loader timings), `groupby`, `filter_type`, `uniquify`, `is_sorted`, screen size, `LineFileProxy` |
| `file_utils.py` | `find_files`, `guess_file_format`, `path_greedy_split`, `touch_file` |
| `encryption.py` | GPG-encrypted ledgers (`is_encrypted_file`, `read_encrypted_file`; needs `gpg` on PATH) — used by `loader.load_encrypted_file` and includes |
| `bisect_key.py` | `bisect_left_with_key` / `bisect_right_with_key` (price lookups) |
| `defdict.py` | `ImmutableDictWithDefault` |
| `import_utils.py` | `import_symbol("pkg.mod.name")` |
| `invariants.py` | instrument a class's methods with invariant checks in tests (`instrument_invariants`) |
| `memo.py` | `memoize_recent_fileobj` (time-bounded cache of fetched files) |
| `pager.py` | `ConditionalPager` (pipe long output to `$PAGER`) |
| `table.py` | table rendering to text / CSV / HTML |
| `errors.h`, `errors.cc` | C++ helpers kept for the C++ port experiments; not part of the `_parser` build |

## Test utilities (`test_utils.py`)

- `@docfile` / `@docfile_extra(...)`: write the test's docstring to a temporary file and pass its
  path — for testing scripts that take a filename.
- `TestCase` / `TestTempdirMixin` / `TmpFilesTestBase` / `tempdir()` / `create_temporary_files()`:
  temporary directories and file trees.
- `ClickTestCase`: run a click command and capture output (`run_with_args`).
- `capture("stdout")`, `patch(obj, attr, value)`, `environ(var, value)`: context managers.
- `skipIfRaises(ImportError)`, `make_failing_importer(...)`: optional-dependency tests.
- `find_repository_root()`, `subprocess_env()`: run `bin/` scripts from tests.

Ledger-level helpers live elsewhere: `loader.load_doc`, `parser.parse_doc`,
`parser.cmptest.TestCase.assertEqualEntries`.
