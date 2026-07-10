# pyRevit → CPython: Implementation Plan

> **Companion to `CPYTHON_MIGRATION_RESEARCH.md`** (the *why*; section refs below like "R§8"
> point into it). This is the tactical *what-to-do*, phased end-to-end from the current draft PR
> to CPython-as-default. Supersedes the sequencing in the earlier three docs.
> **Destination:** `CPY3123` (CPython 3.12 + pythonnet 3) as the **default** engine; IronPython
> kept as opt-in legacy, then sunset.

## How to read this

Seven phases, each with **goal · key tasks · critical files · depends-on · exit criteria ·
verification**. Phases 0–1 and 5 are largely engine-plumbing/tooling; 2–4 are the `forms` long
pole; 6 is the flip. **Size** is relative effort (1–5). Phases gate the next by *exit criteria*,
not calendar. Everything verifiable plugs into the **one** harness from Phase 0 (the
`check_py3_compat` checker + the in-Revit `test_py3_compat` per-engine parity buttons).

**Guiding invariants (hold across every phase):**
- Nothing changes engine *selection* or public API signatures until Phase 6.
- Every change keeps working on the default IronPython 2 engine.
- New CPython behavior lands **incrementally**; anything not yet ported keeps raising
  `PyRevitCPythonNotSupported("pyrevit.forms.<name>")` via the facade `__getattr__` (R§8.1), so
  partial rollout degrades cleanly.
- Extend the Phase 0 checker/suite as each phase adds a new idiom class — a red cell is the
  coverage signal for the work.

---

## Phase 0 — Foundation & harness  ·  Size 3  ·  (draft PR `fix/improve-python3-support`)

**Goal.** Land the language layer, the first 2 bridge classes, and the reusable test harness.
This is the "first-ish step" (R§10) and is mostly done in the draft PR.

**Key tasks.**
- Land Stages 0–5: Py2 syntax residuals; `framework.add_reference_to_file` CLR shim; `clr.Reference`
  out-param ports; heterogeneous-sort hardening; extended checker rules (`PY3-VIEW`, `PY2-MODULE`,
  `PY2-BUILTIN`, `PY2-NEXT`).
- Keep the CI `py3-compat` gate at zero.

**Critical files.** `dev/scripts/check_py3_compat.py`, `pyrevit/unittests/test_py3_compat.py`,
`pyrevit/framework.py`, `revit/db/{query,create}.py`, and the two DevTools parity buttons.

**Depends on.** Nothing (foundation).

**Exit criteria.** PR merged to `develop`; checker at zero and CI-gated; the in-Revit suite green
on IPY2 / IPY342 / CPython with only the documented skips (interop assemblies missing on .NET 8;
markdown deprecated).

**Verification.** `pipenv run check-py3`; run both DevTools parity buttons under an IPY2 attach,
then re-attach IPY342 and re-run (R§10 procedure).

---

## Phase 1 — Bridge backlog: collections & `__namespace__`  ·  Size 3

**Goal.** Clear the two largest/most-blocking pythonnet marshaling classes and, critically,
**resolve the `__namespace__` semantics** the forms modeless tier depends on (R§8.3.4, R§10.1).

**Key tasks.**
- **Tiered checker rules first** (the proven approach, R§10.1): ship a `List[T](pylist)`
  generic-collection classifier (prototype already reached 55 sites, tiered 46/7/2) and a
  `__namespace__`-missing rule (36 sites, 33 missing). Rules enumerate the work and gate
  regressions.
- **Port the 55 generic-collection sites** (~16 lib first, then ~39 ext) to explicit
  `List[T]([...])`; wrap returned .NET collections in `list()` where indexed/mutated.
- **Settle `__namespace__`** by resolving the two open questions on
  `FamilyLoaderOptionsHandler` (R§10.1): (a) which interfaces actually require it under pyRevit's
  pythonnet fork; (b) whether pinning a CLR type collides on reload (the type map survives a
  `sys.modules` clear — the same shared-process-state class as R§5/R§6). Produce a documented
  rule + a reload-safe pattern, then apply to the 33 missing sites (lib before ext).
- Survey/spot-fix the smaller classes as found: `IDisposable` `with` (4), `.Item[...]` indexers,
  enum→int, overload/`System.Func` typing.

**Critical files.** `dev/scripts/check_py3_compat.py` (new rules), `revit/db/*`, `revit/events.py`
(the inconsistent `__namespace__` file), the DevTools "Test CPython Namespace" button, plus the
~39 extension sites.

**Depends on.** Phase 0 (checker + harness).

**Exit criteria.** Both new checker rules at zero for `pyrevitlib`; the `__namespace__` rule
documented (needed-per-interface + reload-safe) and applied to all lib sites; the in-Revit suite
gains passing collection + interface-callback tests on CPython. Extension sites tracked (not
necessarily all fixed — most are IronPython-only until they run under CPython).

**Verification.** Add suite tests: construct a `List[ElementId]` from a Python list and pass to a
Revit API call; register an `IExternalEventHandler`/`ISelectionFilter` implemented in Python and
receive a live callback; **reload the command twice** and confirm no duplicate-type-name error.

---

## Phase 2 — `forms` structural refactor (IronPython-only, behavior-preserving)  ·  Size 4

**Goal.** Split `pyrevit.forms` into the shared-package structure (R§9.1) **without changing any
behavior** — a pure reorganization on IronPython. This is the skeleton that makes CPython
enablement incremental and drift-proof.

**Key tasks (mirrors the refactor note's steps 1–2, corrected).**
1. **Rebase on Phase 0** so the extraction inherits the already-fixed `_ipy.py` (the `__bool__`
   aliases and `list()`-wrapped views) — do **not** reintroduce them.
2. **Carve the skeleton without moving logic:** add `_backend.py` + `backends/{_ipy,_cpy}.py`;
   `_ipy` backend delegates `load_xaml_component` → `wpf.LoadComponent` and
   `make_property_changed_event` → `pyevent.make_event`, and re-exports the assembly refs. Point
   `_ipy.py`'s two `load_xaml` methods **and `utils.py`** at the backend. IronPython behavior
   byte-for-byte unchanged.
3. **Extract shared feature modules** from `_ipy.py` into `base`, `reactive`, `dialogs`,
   `promptbars`, `selection`, `alerts`, `checks`, `pickers`, `notify`, `dockable`. Rebuild
   `__init__.py` as the namespace assembler with an explicit `__all__` reproducing today's
   surface exactly (R§8.1 — there is no `__all__` today, so the surface is "every non-underscore
   name").
4. **Fix `utils.py`'s top-level import** (`from pyrevit.framework import wpf`) — guard it so
   `import pyrevit.forms.utils` doesn't hard-fail under CPython (re-routing the body is not
   enough; R§8.1). Route XAML load through the backend. Use `framework.add_reference_to_file`
   (Phase 0) for the backend assembly refs — don't reinvent.

**Critical files.** `forms/__init__.py`, `forms/_ipy.py` (source of extraction), `forms/_cpy.py`,
`forms/utils.py`, new `forms/_backend.py` + `forms/backends/{_ipy,_cpy}.py` + the 10 feature
modules. Reference idioms: `compat.py`, `runtime/types.py`, `coreutils/git.py` (engine-conditional
`clr.AddReference`).

**Depends on.** Phase 0 (fixed `_ipy.py`, the CLR shim).

**Exit criteria.** IronPython-only end to end; a `dir(forms)` diff before/after (run under
IronPython) shows an identical public surface; every name in the new `__all__` imports; under
CPython, `import pyrevit.forms` and `import pyrevit.forms.utils` do **not** raise at import time
and do **not** eagerly load `wpf`.

**Verification.** Attach the dev clone (`pyrevit clones add dev <repo>`;
`pyrevit attach dev default --installed`) and exercise high-traffic IronPython paths unchanged:
`alert`, `SelectFromList`, `CommandSwitchWindow`, a custom `WPFWindow` subclass, `ProgressBar`,
`select_sheets`/`select_views`, `ask_for_string`. Add a `dir(forms)` parity check to the suite.
`pipenv run pyrevit check`; ruff/black clean.

---

## Phase 3 — Binding prototype gate (R§9.4)  ·  Size 1

**Goal.** Decide the reactive/binding backend bet **before** building it. Highest-risk, smallest
effort — do it early.

**Key tasks.** Build a throwaway spike on pyRevit's pythonnet fork demonstrating:
1. **WPF binding fidelity to a Python / `DynamicObject` source** using `SelectFromList.xaml` as
   the fixture — `{Binding name}`, the `DataTrigger`s the parameter pickers use, `DataTemplate`
   resolution, validation, and `ICollectionView` sort/group. Not just the happy path.
2. **A modeless `ProgressBar` driving Revit API calls through `ExternalEvent`** — measure
   GIL × Dispatcher × Revit-context (R§8.3.5), reusing the Phase 1 `__namespace__` pattern.

**Depends on.** Phase 1 (`__namespace__` pattern), Phase 2 (structure to host the spike cleanly).

**Exit criteria — a decision, recorded in the research doc:**
- **Pass** → the `reactive` backend is a **pure-Python `INotifyPropertyChanged` shim**; Phase 4's
  data-bound tier stays pure Python.
- **Fail** → the `reactive` backend is a **C# `BindableModel`/`DataTable` adapter** (R§9.2, the
  `ScriptOutput`/`PyRevitOutputWindow` pattern); build the two C# primitives (window-host base +
  `BindableModel`/bindable collections) and expose them through the backend. Feature modules are
  untouched either way.

**Verification.** The spike itself is the verification; capture results (and any perf numbers) in
`CPYTHON_MIGRATION_RESEARCH.md` §9.4.

---

## Phase 4 — `forms` CPython enablement, per tier  ·  Size 5

**Goal.** Turn on real CPython forms **incrementally, tier by tier** (not the refactor note's
big-bang step 4). Each tier: implement `backends/_cpy.py` support as needed, move symbols from
stub to real, add to `__all__`, port the shipped extensions that use them, and keep the
`__getattr__` fallback for the rest.

**Tiers (easy → hard; R§8.3 / R§9.3):**

| Tier | Elements | Backend need | Notes |
| --- | --- | --- | --- |
| 0 | `check_*` validators | none | pure logic |
| 1 | `alert`, `pick_file`/`pick_folder`/`save_file`, `ask_for_color`, `toast`, `show_balloon` | none | native dialogs; §12 idiom cleanups only |
| 2 | `WPFWindow`/`WPFPanel` base via pure-Python `load_xaml_component` (XamlReader + name-scope wiring) | XAML loader | the largest *new* code; no binding yet |
| 3 | `ask_for_string`/`_date`/`_number_slider` | XAML loader | small imperative windows |
| 4 | `SelectFromList`, `CommandSwitchWindow`, `select_*`, parameter/color/image selectors | **binding backend (Phase 3 decision)** | the data-bound tier — the crux |
| 5 | `ProgressBar`, `WarningBar`, dockable panels (`WPFPanel` register/open/close/toggle) | `ExternalEvent`/`__namespace__` + GIL | modeless; depends on Phase 1 |

Then `settings_window.py` (subclasses `WPFWindow`) works once Tier 2 lands; verify it under
CPython.

**Key tasks.** Implement the pure-Python XAML loader in `backends/_cpy.py` (parse via
`XamlReader`/`Application.LoadComponent`, copy tree onto `target`, walk the name-scope to wire
`x:Name` controls + connect handler methods — R§8.3.2). Wire the reactive backend per Phase 3.
Port shipped extensions (the R§7 corpus: 167 forms-consumers) as each tier lands; freeze the two
`rpw.ui.forms` tools onto legacy (R§7.1).

**Critical files.** `forms/backends/_cpy.py` (grows from stub to real), the feature modules (they
should need *no* per-engine edits), `forms/settings_window.py`.

**Depends on.** Phase 2 (structure), Phase 3 (binding decision), Phase 1 (`__namespace__` for
Tier 5).

**Exit criteria.** Every public `_ipy.py` symbol either works under CPython or appears in a
published **coverage matrix**; the R§7 parity corpus passes under both engines for shipped tiers;
`settings_window.py`/`utils.py` no longer hard-require IronPython. Tiers 0–4 complete is the gate
for Phase 6.

**Verification.** For each tier, run the same functions from a `#! python3` pushbutton and diff
observable behavior against the IronPython reference via the parity suite. Focus first on Tier 2
XAML load + named-control binding (the risk center), then Tier 4 binding.

---

## Phase 5 — Dependency & packaging mechanism  ·  Size 2

**Goal.** Make third-party dependencies work sanely under one shared interpreter, and split the
vendored tree per engine. Must ship **before** the Phase 6 flip. (R§6)

**Key tasks.**
- **Split `site-packages/` per engine family** (R§6.4.5): a modern Py3 tree for CPython
  (re-vendored current releases) + a *frozen* Py2 tree for legacy IronPython (the backports live
  only there). Each engine resolves only its own tree — a loader-path change, since resolution
  paths already differ.
- **Wire pip into the embeddable distro + a managed writable user-site**, and add a `pyrevit`
  CLI command (`pyrevit env pip install …`) — the distro ships none today (R§6.1).
- **Curate + pin the shared Py3 env** (platform-SDK model, R§6.4.1), ABI-matched to CPython 3.12
  `win_amd64`.
- **Declaration + conflict-check** (R§6.4.3): let extensions declare pip requirements resolving
  against the shared env, failing loud on incompatible demands. Document per-extension bundling
  as an unsupported footgun.
- **Ship the R§5 isolation mitigations:** fix the `sys.path` baseline snapshot; add the module
  eviction policy **split per R§6.5** (evict extension-namespace *code*; never evict
  site-packages/pip/C-ext modules).

**Critical files.** `pyRevitfile` (`[deployments]`, `[engines.*]`), the CLI
(`dev/pyRevitLabs/pyRevitLabs.PyRevit/`), `CPythonEngine.cs` (`StoreSearchPaths` baseline fix +
the eviction hook), `pyrevit/__init__.py` (`MISC_LIB_DIR`), `extensions/extpackages.py`
(`dependencies` → pip declaration).

**Depends on.** Phase 0 (engine correctness). Can run in parallel with Phases 2–4.

**Exit criteria.** Each engine resolves only its own tree; `pyrevit env pip install numpy` works
into the CPython env; the eviction policy demonstrably drops a colliding `get_door` (R§5 example)
while keeping numpy warm across runs; the baseline snapshot bug is fixed (verified by the B-after-A
ordering case).

**Verification.** Suite tests: two extensions with a same-named `lib/get_door.py` — assert each
run gets its own; import numpy across two extensions in a session — assert one shared instance,
never evicted; run B after A — assert B's baseline `sys.path` is pristine.

---

## Phase 6 — Flip the default  ·  Size 4

**Goal.** Make `CPY3123` the default via a *designed mechanism*, keep legacy IronPython opt-in,
then sunset. (R§11)

**Key tasks (R§11.3).**
- **Per-extension `engine` declaration** in extension/bundle metadata: `ironpython` keeps today's
  behavior indefinitely; `cpython` gets the new default for the extension's no-shebang scripts.
  Per-script shebangs still win.
- Ship the declaration for **≥1 major release** with the IronPython default announced deprecated
  but *unchanged*, alongside the published forms coverage matrix + a migration guide (the R§12
  checklist is most of it; the Phase 0 checker doubles as a "check your extension" tool).
- Have all shipped extensions declare `cpython` once ported (Phase 4).
- Only then let a global default change affect extensions that declare *nothing* — deferrable,
  staged by major version, or scoped to extensions created after a cutoff.

**Depends on.** Phases 4 (Tiers 0–4 + coverage matrix) and 5 (isolation mitigations + packaging).

**Exit criteria.** Declaration shipped ≥1 major release; all shipped extensions declare an engine;
coverage matrix complete for Tiers 0–4; R§5 isolation mitigations shipped. Then flip; sunset
legacy (and delete the frozen Py2 tree, `rpw`, vendored `markdown`) once the ecosystem has moved.

**Verification.** A no-shebang script in a `cpython`-declared extension runs on CPython; an
`ironpython`-declared (or undeclared) extension is unchanged; the R§7 shipped-extension corpus
passes under the flipped default.

---

## Dependency graph (at a glance)

```
Phase 0 (foundation + harness)
  ├─► Phase 1 (bridge: collections, __namespace__) ─┐
  ├─► Phase 2 (forms structural refactor) ──► Phase 3 (binding gate) ─┐
  └─► Phase 5 (deps & packaging)  [parallel]                          │
                                     Phases 1+2+3 ─► Phase 4 (forms CPython, per tier)
                                              Phase 4 (Tiers 0–4) + Phase 5 ─► Phase 6 (flip)
```

## What to reuse (don't reinvent)

- The **checker + parity suite** (Phase 0) — every later phase adds rules/tests here.
- **`framework.add_reference_to_file`** (Phase 0) for all portable assembly loading, incl. the
  forms backend.
- The **facade `__getattr__` + `__all__`** scaffold (R§8.1) for incremental forms rollout.
- The **`ScriptOutput`/`PyRevitOutputWindow`** C#-hosted-window-via-thin-Python-wrapper pattern
  if Phase 3 lands on the C# binding backend.
- The existing **`compat.py` `PY2`/`PY3`/`IRONPY`** branches and the engine-conditional
  `clr.AddReference` idioms in `runtime/types.py` / `coreutils/git.py`.
