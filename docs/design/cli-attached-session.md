# Design: CLI Connection to a Live Revit Session

Status: Draft / proposal
Owner: (tbd)
Related area: `pyrevitlib/pyrevit/routes/`, `dev/pyRevitLabs/pyRevitCLI/`

## Goal

Let the pyRevit CLI (and, through it, an external coding agent or a test
harness) connect to an **already-open** Revit model and:

1. run arbitrary Python against the live API context and get results back,
2. later, trigger existing pyRevit commands/buttons,

without launching a new Revit process.

This is distinct from today's `pyrevit run` (`PyRevitRunner.cs`), which boots a
**new headless Revit** via journal playback and quits it. That flow cannot
observe or drive a session the user already has open. This proposal is the
complementary "attach to the running session" mode.

## What already exists (and what we are reusing)

The hard, Revit-specific problem is already solved by the **Routes** feature.
We are not building an in-process server or a thread-marshaling scheme from
scratch — we are adding an auth gate, one execution endpoint, a results
contract, and a CLI client.

- **In-process HTTP server.** `routes/server/server.py` runs a threaded
  `HTTPServer` on a configurable host/port, started as a daemon thread when a
  session loads. Flask-like route registration lives in `routes/api.py`.
- **Correct thread marshaling.** The Revit API is single-threaded and
  non-reentrant; requests arrive on HTTP worker threads. `routes/server/handler.py`
  wraps handlers in an `IExternalEventHandler`, with the `ExternalEvent`
  created once on the main thread at load. When a handler's signature includes
  `uiapp`/`uidoc`/`doc`, the request is marshaled onto the API thread via
  `ExternalEvent.Raise()` and the worker blocks until Revit signals completion.
  `Denied`/`TimedOut` (e.g. a modal dialog is open) are already handled.
- **Instance discovery.** `routes/server/serverinfo.py` writes a per-Revit
  pickle keyed to the process id and exposes `get_registered_servers()`, which
  enumerates every running Revit and its port. Port discovery for the CLI is
  therefore already available.
- **CLI config surface.** `pyrevit configs routes` (`PyRevitCLI.cs:740`) already
  enables the server, sets the port, and toggles the built-in core API.
- **Command invocation primitive.** `sessionmgr.execute_command(unique_id)` and
  `execute_command_cls` resolve a pyRevit command by id and run its `.Execute()`
  path — the same path the ribbon uses. `find_all_available_commands()`
  enumerates commands and their unique ids.

What does **not** exist yet: any authentication, an arbitrary-exec endpoint, a
results/output-capture contract, a command-trigger route, and a CLI client that
*calls* the server (today the CLI only *configures* it).

## Fundamental constraints (accept these; do not fight them)

- **One API thread.** All API-context work is serialized through the external
  event on Revit's main thread. A long handler freezes the Revit UI. Suitable
  for probing and short automation; not for heavy batch work.
- **Modal state blocks execution.** While Revit shows a modal dialog or is
  mid-transaction-group, `Raise()` returns `Denied`/`TimedOut`. Calls fail
  until the user clears the dialog. Surface this clearly to the caller.
- **Results must be JSON-serializable.** A live `Element` cannot cross the HTTP
  boundary. Handlers return serialized data (the existing IronPython-safe
  dumper is in `handler.py`).
- **No implicit transaction.** See the transactions section below.

## Authentication

The Routes server currently has **no auth** (there is a literal `# TODO: auth`
in the core API). Exposing arbitrary execution requires closing that gap.

### Model: human-gated pairing token (device-pairing pattern)

1. User enables Routes and clicks a **"Connect / Allow agent"** button in the
   Revit session.
2. The button mints a high-entropy random token, holds it in memory for the
   session, and **displays it** (short code or copyable string).
3. The CLI prompts the user to paste it; the CLI keeps it for its session.
4. The CLI sends `Authorization: Bearer <token>` on every request.
5. The server validates the header **before dispatching a route** and returns
   `401` when it is missing or wrong.

### Why this shape

- The **manual paste is the security property**, not friction: it proves a
  human at the Revit machine consented at that moment — the right gate in front
  of remote code execution against a live model.
- It composes with a **`127.0.0.1` binding**. Localhost blocks off-machine
  access; the token additionally blocks other local processes/users on a shared
  workstation. Defense in depth.
- **Do not store the token in the `serverinfo` pickle.** That file is how the
  CLI discovers the port and is readable by any local process; putting the
  token there would defeat the gate. Port discovery is automatic; the token is
  human-gated. Keep them separate.

### Requirements

- Constant-time token comparison.
- Session-scoped lifetime: invalidate on toggle-off and on session reload.
- Token shown once; regenerated per Connect.
- A lighter fallback (localhost-only, no token) is acceptable for a single-user
  machine but is explicitly weaker; the token model is the default once more
  than one process/person can reach the port.

## Transactions and undo history

The Routes API uses **no transactions today** — confirmed: there are zero
transaction references anywhere in `pyrevitlib/pyrevit/routes/`. The model:

- A handler that requests API context is simply *called* on the main thread in
  a **valid API context** where opening a transaction is permitted — but
  **nothing opens one for it**. This matches the rest of pyRevit: normal
  pushbutton scripts are not auto-wrapped in a transaction either.
- **Reads** (query elements, read parameters) need no transaction, leave no
  undo entry, and cannot corrupt the model. This is the safe, ideal case for an
  agent probing API returns.
- **Writes** require the handler to open a `Transaction` itself. pyRevit ships
  the helper (`pyrevit/revit/db/transaction.py`): a context manager that does
  `Start()`/`Commit()` and **auto-rolls-back on exception**, downgrading to a
  `SubTransaction` when one is already open.

### What "transaction history" means here

Every committed `Transaction` becomes **one named entry in Revit's undo
stack** — the *same* stack as manual user edits. There is no sandbox. So:

- Read probing is genuinely low-risk and untracked.
- Writes are real: they modify the live (possibly workshared, possibly saved)
  model. They are `Ctrl+Z`-undoable, but they happened.

### Safety knobs (cheap, built from existing helpers)

- **Dry-run mode:** wrap the executed code in a `Transaction` and always
  `RollBack()`. Changes are visible to queries *during* execution (so the agent
  can ask "if I did X, what would Y return?") but never persist.
- **Session grouping:** wrap an agent run in a `TransactionGroup` so its edits
  collapse into a single undo entry the user can revert in one step.

Default posture: **read-only.** Writes require an explicit opt-in flag (or the
dry-run rollback path).

## Output capture — the load-bearing design decision

Getting results back depends on **who owns stdout**. There are two execution
models with very different output stories.

### Model A — run logic *as* eval code (the workhorse)

The eval handler executes code directly in the routes-server engine on the API
thread. It wraps `exec` with a stdout/stderr capture, so everything the code
prints — plus its return value and any exception — comes back **in the HTTP
response, synchronously**. This is the natural "run Python, get results back"
path and should be the primary interface for tests, API probing, and any
"what does this return" question.

### Model B — trigger a pyRevit command/button

`sessionmgr.execute_command(...)` routes through `ScriptExecutor.ExecuteScript`,
which spins up a **separate** `ScriptRuntime` with its **own** `OutputStream`
bound to its **own** `ScriptOutput` window (`scriptruntime.cs`,
`ScriptOutput.GetForRuntime`). The command's `print()` / `script.get_output()`
write to *that* window inside Revit — **not** to the eval's captured stdout. A
trigger returns effectively `None` (or an `Execute` status), and the real
output either pops an output window on the user's screen or lands in the
runtime/telemetry log, both of which must be scraped **separately**.

### Consequence

For the canonical example — the DevTools unit-test button — the button script
is only a few lines calling `pyrevit.unittests.runner.run_module_tests`. The
right way for an agent to run those tests is **Model A**: run that same logic
through `/eval` and return structured pass/fail from the `TestResult` object
plus captured stdout. Do **not** click the button and scrape a window.

Therefore Phase 1's `/eval` must return a structured envelope:

```json
{
  "stdout": "...",
  "stderr": "...",
  "result": <json-serializable or null>,
  "exception": {"type": "...", "message": "...", "traceback": "..."} 
}
```

`result` is populated by convention (e.g. the value bound to a `result` name in
the exec namespace, or the value of a trailing expression). The `TestResult`
object is the reliable channel for test detail, since `run_module_tests` sends
its formatted detail to the output window rather than plain stdout.

Button-triggering (Model B) is intentionally **fire-and-forget** in v1;
capturing a triggered command's output means redirecting that child runtime's
`OutputStream` or reading the telemetry log — real work, deferred.

## Phased plan

### Phase 1 — read-only attached eval (highest value, safest)

Server (Python, new core-API routes):

- `GET /health` — liveness + host/version/pid/doc title (no API context needed
  beyond `uiapp` for doc title).
- `POST /eval` — body `{ "code": "...", "mode": "read"|"dryrun" }`. Runs on the
  API thread with `uiapp`/`uidoc`/`doc` in scope, captures stdout/stderr,
  returns the envelope above. `read` opens no transaction; `dryrun` wraps in a
  `Transaction` that always rolls back.
- Auth: bearer-token check in front of dispatch (`server.py`
  `_process_request`/`_handle_route`); `401` on failure.

Revit UI:

- A **Connect** button/smartbutton that mints + displays the token, starts
  listening (if not already), and shows host/port. Toggling off invalidates the
  token.

CLI (C#, `pyRevitCLI`):

- `pyrevit attach connect` — discover running Revits from `serverinfo`, prompt
  for the pasted token, store it for the session, verify against `/health`.
- `pyrevit attach eval <script.py | -->` `[--dry-run]` — POST code, print the
  returned envelope (stdout, then result/exception), set exit code from
  `exception`.
- Instance selection when multiple Revits are running (by pid/port/doc title).

### Phase 2 — write mode with transaction control

- `/eval` `mode: "write"` gated behind an explicit CLI `--write` flag.
- Optional `TransactionGroup` wrapping for a whole agent session.
- Clear surfacing of `Denied`/`TimedOut` (modal open) as a distinct, retryable
  error to the CLI.

### Phase 3 — command/button triggering (convenience, fire-and-forget)

- `GET /commands/` — list available commands + unique ids
  (`find_all_available_commands`).
- `POST /commands/<unique_id>` — run via `execute_command`. Documented as
  fire-and-forget: returns "executed", not the command's output.
- `pyrevit attach run-command <unique_id>` on the CLI.
- Optional later: capture the triggered runtime's output (redirect its
  `OutputStream` or read telemetry) — separate work item.

## Security summary

- Default bind `127.0.0.1`; token required by default.
- Human-in-the-loop Connect gesture per session; token shown once,
  session-scoped, constant-time compared, invalidated on toggle-off/reload.
- Read-only default; writes require explicit opt-in; dry-run available.
- Token never persisted to `serverinfo` or any world-readable location.

## Open questions

- Token transport for `dryrun`/`write` beyond a single machine (out of scope for
  v1; localhost only).
- Whether `result` is captured by trailing-expression evaluation, an explicit
  `result` name, or both.
- Timeout/cancellation for a long-running eval that is blocking the API thread.
- Multi-instance UX: default to sole running Revit, require explicit selection
  otherwise.
