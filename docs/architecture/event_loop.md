# Runtime: `honba.event_loop`

`honba._honba` runs async Rust work on one tokio runtime per interpreter. `honba.event_loop` is
the only module that creates and destroys it. Source of truth: ADR 0015 and
`honba/python/src/honba/event_loop.py`.

```python
from honba import event_loop

with event_loop.running():          # starts the runtime, stops it on exit
    ...                             # in-process calls reuse the started runtime

info = event_loop.start()           # or explicitly
print(info.flavor, info.worker_threads, info.generation)  # multi-thread <n> 1
event_loop.stop()
```

## API

| Function | Behavior |
|---|---|
| `start(worker_threads=None)` | Starts the runtime and returns a `RuntimeInfo`. `worker_threads` defaults to one per core. Idempotent for the owner: a second call returns the running runtime. |
| `stop()` | Stops the runtime this module started and joins its threads. Returns whether it stopped one. Idempotent. |
| `is_running()` | Whether a runtime is running, whoever started it. |
| `info()` | The running `RuntimeInfo`, or `None`. |
| `running(worker_threads=None)` | Context manager. Starts the runtime if needed; on exit it stops it only if this block started it. |

`RuntimeInfo` is a frozen dataclass with `flavor` (`"multi-thread"`), `worker_threads` and
`generation`.

## Rules

- **One runtime per interpreter.** `start` while a runtime is running does nothing new for the
  owner. A runtime started outside the module (a direct `honba._honba.runtime_start` call) is never
  adopted or stopped: `start()` then raises `EventLoopError`.
- **Guard on conflicting requests.** `start(worker_threads=N)` raises `EventLoopError` when the
  owned runtime runs with a different explicit count; call `stop()` first. `worker_threads=0`
  raises `ValueError`.
- **Restartable.** The runtime can be started again after `stop()`. `RuntimeInfo.generation`
  counts starts.
- **Deterministic shutdown.** `stop()` is registered with `atexit`, so audit streams are flushed
  before the process ends. It blocks until in-flight calls finish; there is no timeout.
- **No pyo3 `auto-initialize`.** The extension is loaded into a host interpreter and never
  starts one. A test (`crates/honba-py/tests/no_auto_initialize.rs`) fails if the feature turns on.
- **No runtime is required.** Without a started loop the in-process REST path
  (`InprocTransport`, see [Python SDK](../api/python-sdk.md)) builds a throwaway current-thread
  runtime per call. Answers are identical either way; starting the loop only avoids that
  per-call cost.

## Deprecated shims

`honba._honba.initialize_runtime()` and `get_runtime_handle()` remain but are deprecated:

- `initialize_runtime()` starts the runtime if none is running and is now idempotent (it used to
  leak a runtime on every call). A runtime started this way is not owned by `honba.event_loop`.
- `get_runtime_handle()` returns the flavour (`"tokio-multi-thread"`) and raises `RuntimeError`
  when no runtime is running (it used to return a constant string).

Use `event_loop.start()` and `event_loop.info()` instead.

## Limits

- The runtime is per process; sub-interpreters are not supported.
- `pyo3-async-runtimes` is not handed the runtime, and no async Python API is built on it yet
  (`honba.async_run` does not exist).
