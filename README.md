# Ledger — a banking-ledger CLI written in Memelang

A small but complete double-entry-ish banking ledger: accounts, deposits,
withdrawals, transfers, balances, and monthly statements. Everything is
written in [Memelang](https://github.com/egoles/meme) (Structured English,
compiled by the offline `heuristic` parser) and emits identical behavior on
all six backends: **Python, Rust, C++, Go, C, and TypeScript**.

This is the Phase-4 beta capstone: it demonstrates that Memelang can carry a
fresh-CS-graduate-level backend program end to end — typed records, modules,
`Optional<T>` error channels, JSON persistence, deterministic date math, and
a cross-target test suite.

## Source layout

| File | Role |
|---|---|
| `main.meme` | argv router only (loads `ledger` + `statement` modules) |
| `ledger.meme` | domain: `Account`/`Transaction` records, JSON loaders/savers, id allocation, find-by-name, validate/apply operation pairs (`Optional<String>` error channel), exact money rendering |
| `statement.meme` | monthly statements: UTC calendar math (no date builtins), chronological replay, opening/closing balances |
| `tests/meme/*.meme` | 36 test cases (`meme test` discovers them) |
| `tests/meme/ledger.meme`, `tests/meme/statement.meme` | symlinks to the root modules (modules resolve relative to the importing file's directory) |

## CLI

```
ledger add <account-name>              # fails if exists; prints account id
ledger deposit <account> <cents>       # fails: unknown account, cents <= 0
ledger withdraw <account> <cents>      # fails: unknown account, cents <= 0,
                                       #       insufficient funds
ledger transfer <from> <to> <cents>    # fails: unknown account(s), cents <= 0,
                                       #       insufficient funds, from == to
ledger balance <account>               # prints "<id> <name>: $<dollars.cc>"
ledger statement <account> <yyyy-mm>   # monthly report
ledger list                            # all accounts, sorted by name
```

Validation failures print one clear line to **stderr** and exit with code `1`.
Unknown commands print the usage text to stderr and exit `1`. Reports go to
stdout.

## Data files

All persisted state is JSON under `data/` (created on demand, gitignored):

- `data/accounts.json` — `List<Account>` (`id`, `name`, `balance_cents`)
- `data/transactions.json` — append-only `List<Transaction>`
  (`id`, `from_id`, `to_id`, `amount_cents`, `epoch`; `from_id = 0` marks a
  deposit, `to_id = 0` marks a withdrawal)

Money is stored as **integer cents** everywhere. Missing or corrupt data
files are handled through the Optional JSON readers
(`read json records ... from file ... or else nothing`) — they degrade to
empty state or a clean one-line error, never a traceback.

## Running

The compiler lives at `/mnt/ssd/projects/meme` (run everything from this
repo; `--provider heuristic` is the only parser used):

```bash
# compile to Python and run the CLI
uv run --project /mnt/ssd/projects/meme meme compile main.meme \
    --provider heuristic --target python -o ledger.py
python3 ledger.py add alice

# native binary (C11 backend)
uv run --project /mnt/ssd/projects/meme meme build main.meme -o ledger_bin \
    --provider heuristic
./ledger_bin balance alice

# other targets: --target rust|cpp|go|c|ts
```

## Tests

```bash
uv run --project /mnt/ssd/projects/meme meme test              # python (default)
uv run --project /mnt/ssd/projects/meme meme test --target rust
uv run --project /mnt/ssd/projects/meme meme test --target cpp
uv run --project /mnt/ssd/projects/meme meme test --target go
uv run --project /mnt/ssd/projects/meme meme test --target c
uv run --project /mnt/ssd/projects/meme meme test --target ts
```

36 cases across five files:

- `01_ledger_core_test.meme` — 8 cases — add (ok / duplicate / empty name), deposit
  validation, withdrawal funds check, find-by-name
- `02_money_math_test.meme` — 4 cases — balance math across multiple operations, exact
  dollar rendering, id allocation
- `03_statement_test.meme` — 11 cases — month bucketing and replay with **fixed epoch
  values** (no wall-clock assertions), leap years, transfers seen from both
  sides, gap months, empty replay, statement year validation
- `04_persistence_test.meme` — 6 cases — JSON roundtrips, missing files, corrupt and
  wrong-shape files (Optional reader returns nothing → clean behavior)
- `05_operations_test.meme` — 8 cases — full deposit / withdraw / transfer semantics:
  ok paths, unknown account, zero/negative amounts, insufficient funds,
  self-transfer rejection

Notes:

- Tests write scratch JSON with per-test unique filenames into the
  compiler-managed test working directory, which is deleted automatically
  after each run.
- Every test file parses with the offline `heuristic` provider only.
- Generated `.meme.lock` sidecars and `build.lock` are gitignored build
  artifacts; they are safe to delete and regenerate. If a root `build.lock`
  exists, delete it before running `meme test` (the test runner can pick it
  up as a lockfile cache).

## Design notes

- **Error channel:** all operation validation returns `Optional<String>`
  (`nothing` = ok, `some` = error message). `main.meme` prints the message
  to stderr and exits 1.
- **Value semantics:** `ledger.meme` ships validate/apply pairs
  (`validate_transfer` + `apply_transfer`, etc.). `apply_*` return a *new*
  list built with an explicit element-wise copy — the plain
  `define list out as src` copy idiom aliases backing storage on the Go, C,
  and TypeScript backends — so the caller's list is never mutated and all
  six backends behave identically.
- **Determinism:** statements replay the transaction log chronologically
  (stable insertion sort on `(epoch, id)`); tests assert only relational
  properties and fixed epochs, never the clock.
- **Money rendering:** `dollars_text` renders cents exactly with integer
  math (`1234` → `"12.34"`), avoiding binary-float rounding entirely.
  `format <float> with N decimals` is referenced by a documented shim
  (`dollars_float_preview`) that keeps the runtime float-format helper
  linked in every backend (some backends emit it lazily and the JSON
  serializer depends on it).
- **stderr:** the language has no stderr print, so error lines are written
  with `append ... to file "/dev/stderr"` (verified to produce real stderr
  bytes with identical exit codes on all six backends).
