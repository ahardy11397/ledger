# PLAN — Ledger: a Memelang capstone (beta proof program)

Build a complete banking-ledger CLI in Memelang. This is the Phase-4 beta
capstone: it proves Memelang can carry a fresh-CS-graduate-level backend
program end to end.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compile with
`uv run meme --help` from /mnt/ssd/projects/meme. Key capabilities you will
use: records, `Map<K, V>`, `Set<T>`, `Optional<T>` with `or else`,
`read/write ... as json to file`, `read json record[s] ... from file ...
or else nothing`, `format <float> with N decimals`, `current epoch seconds`,
`assert`, `test <name>:` blocks, `use <module>`, argv, `exit with code N`.

## Ground rules (non-negotiable)

- Work ONLY in this repo (/mnt/ssd/projects/ledger). Do not modify the
  compiler repo.
- NO git commits. Leave all changes uncommitted for coordinator review.
- Money is stored as integer CENTS (Int). Floats appear only in reports,
  rendered via `format <float> with 2 decimals`.
- All persisted state is JSON in this repo's `data/` directory (created on
  demand). Never write outside the repo.
- The deterministic heuristic parser is the only parser: every construct
  must parse with `--provider heuristic`.
- Determinism: tests must NOT assert exact wall-clock values (assert
  relational properties only). Transaction timestamps use `current epoch
  seconds`.

## CLI surface (argv-routed in main.meme; unknown command → usage text +
exit 1)

    ledger add <account-name>              # fails if exists; prints account id
    ledger deposit <account> <cents>       # fails: unknown account, cents <= 0
    ledger withdraw <account> <cents>      # fails: unknown account, cents <= 0,
                                           #       insufficient funds
    ledger transfer <from> <to> <cents>    # fails: unknown account(s), cents <= 0,
                                           #       insufficient funds, from == to
    ledger balance <account>               # prints "<account> <name>: $<dollars.cc>"
    ledger statement <account> <yyyy-mm>   # monthly report (see below)
    ledger list                            # all accounts, sorted by name

Error handling contract: validation failures print one clear line to stderr
and `exit with code 1`. Missing/corrupt data files are handled through the
Optional JSON readers (`or else nothing`), never a traceback.

## Data model (modules; thin main + logic modules — main.meme only routes argv)

- `ledger.meme` — records `Account` (id: Int, name: String, balance_cents:
  Int) and `Transaction` (id: Int, from_id: Int, to_id: Int, amount_cents:
  Int, epoch: Int; use from_id = 0 for deposits, to_id = 0 for withdrawals);
  loader/saver functions using the Optional JSON readers
  (`read json records ... from file ... or else nothing` / `write ... as
  json to file`); next-id allocation; find-by-name; apply-and-validate
  functions returning `Optional<String>` (nothing = ok, some = error
  message) — THE user-locked error channel for operation results.
- `statement.meme` — monthly statement: filter transactions by account and
  month (month bucketing computed in-language from epoch seconds: UTC, days
  per month incl. leap years — no date builtins), running balance, opening
  and closing balance, printed via `format ... with 2 decimals`; totals as
  cents math, floats only at print time.
- `main.meme` — argv router only; loads modules with `use`.

`Set<String>` of account names for uniqueness; `Map<Int, Account>` for
lookup; a `List<Transaction>` append-only log.

## Tests (tests/meme/*.meme — `meme test` discovers them)

Use `use` of the logic modules. `test <name>:` blocks with `assert` covering
at minimum: add (ok, duplicate rejected), deposit (ok, zero/negative
rejected, unknown account rejected), withdraw (ok, insufficient funds,
unknown account), transfer (ok, self-transfer rejected, insufficient funds),
balance math across multiple operations, statement month-bucketing (use
FIXED epoch values, not the clock), corrupt-data-file tolerance (Optional
reader returns nothing → clean error). Tests write scratch data into
`tests/tmp/` (each test uses its own file names; delete after).

## Build + acceptance ("done means")

1. `uv run meme test` from this repo — all cases pass (python target).
2. `uv run meme test tests/meme --target rust` and `--target cpp`,
   `--target go`, `--target c`, `--target ts` — ALL SIX targets green
   (user-locked requirement).
3. `uv run meme build main.meme -o ledger_bin` — native binary works:
   run an end-to-end demo (add 2 accounts, deposit, withdraw, transfer,
   statement) and show the output in your final summary.
4. `uv run pytest -q` in /mnt/ssd/projects/meme must still be untouched —
   you did not modify the compiler.
5. A `README.md` documenting the CLI, data files, and how to run tests.

## Final report

List files created, test case count per target, the demo transcript, and
any deviations from this plan with reasons.
