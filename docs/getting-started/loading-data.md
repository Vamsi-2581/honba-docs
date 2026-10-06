# Loading Data: `honba data load`

Reads a Parquet bar file for one instrument and prints a summary. Nothing is
written; use it to check a file before a backtest or before `honba serve`.

```bash
honba data load <file.parquet> <SYMBOL> [--exchange NSE|BSE]
```

```bash
honba data load bars/INFY.parquet INFY                    # NSE (default)
honba data load bars/INFY.parquet INFY --exchange BSE     # BSE
honba data load bars/INFY.parquet NSE:INFY                # qualified
```

Output (stdout):

```text
<count> bars loaded from <file> (NSE:INFY)
first: ts_event=<ns> close=<price>
last:  ts_event=<ns> close=<price>
```

The `first`/`last` lines are omitted for an empty file. Only `.parquet` sources
are supported; any other extension fails with
`unsupported source extension: <file> (expected .parquet)`.

## Symbol forms

| Form | Where | Example |
|---|---|---|
| Input: bare symbol | CLI argument | `INFY` (exchange from `--exchange`, default NSE) |
| Input: qualified | CLI argument | `NSE:INFY`, `BSE:INFY` |
| Display | ids in output, REST paths, `InstrumentId` | `INFY.NSE` |

Input is `EXCHANGE:SYMBOL`; the display form is `SYMBOL.EXCHANGE`. Do not pass
`INFY.NSE` to the CLI, and do not use `NSE:INFY` in a REST path. Exchange codes
are case-insensitive (`nse` works). The summary line above prints the
qualified form `NSE:INFY`.

## Errors

| Cause | Message |
|---|---|
| Exchange not NSE or BSE | `unknown exchange "XYZ" (supported: NSE, BSE)` |
| Qualifier and `--exchange` disagree (`BSE:INFY --exchange NSE`) | `symbol qualifier BSE conflicts with exchange NSE` |
| Empty symbol, whitespace, a second `:` | `invalid symbol "..." (expected SYMBOL or EXCHANGE:SYMBOL)` |

A qualifier and `--exchange` that agree (`NSE:INFY --exchange NSE`) are accepted.
Exit status is non-zero on any error.

Name files `SYMBOL.EXCHANGE.parquet` (for example `INFY.NSE.parquet`) when they
are meant for [`honba serve`](../api/rest.md).
