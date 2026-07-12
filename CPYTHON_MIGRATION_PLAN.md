# pyRevit → CPython: Implementation Plan

> **Companion to `CPYTHON_MIGRATION_RESEARCH.md`** (the *why*; refs like "R§8" point into it). Tactical, phased end-to-end from the current draft PR to CPython-as-default. Supersedes the sequencing in the earlier three docs.
> **Destination:** `CPY3123` (CPython 3.12 + pythonnet 3) as the **default** engine; IronPython kept as opt-in legacy, then sunset.

## How to read this

Seven phases, each with **goal · key tasks · critical files · depends-on · exit criteria · verification**. Phases 0–1 and 5 are engine plumbing and tooling; 2–4 are the `forms` long pole; 6 is the flip. **Size** is relative effort (1–5). Phases gate by exit criteria, not calendar. Everything verifiable plugs into the one Phase 0 harness (the `check_py3_compat` checker + the in-Revit per-engine parity buttons).

**Invariants (every phase):**
- Nothing changes engine selection or public API signatures until Phase 6.
- Every change keeps working on the default IronPython 2 engine.
- CPython behavior lands incrementally; unported symbols keep raising `PyRevitCPythonNotSupported` via the facade `__getattr__` (R§8.1).
- Extend the checker/suite as each phase adds an idiom class — a red cell is the coverage signal.

---

## Phase 0 — Foundation & harness  ·  Size 3  ·  (draft PR `fix/improve-python3-support`)

**Goal.** Land the language layer, the first two bridge classes, and the reusable test harness (R§10). Mostly done in the draft PR.

**Key tasks.** Land Stages 0–5 (Py2 syntax residuals; the `framework.add_reference_to_file` CLR shim; `clr.Reference` out-param ports; sort hardening; extended checker rules). Keep the CI `py3-compat` gate at zero.

**Critical files.** `dev/scripts/check_py3_compat.py`, `pyrevit/unittests/test_py3_compat.py`, `pyrevit/framework.py`, `revit/db/{query,create}.py`, the two DevTools parity buttons.

**Depends on.** Nothing.

**Exit criteria.** PR merged to `develop`; checker at zero, CI-gated; in-Revit suite green on IPY2/IPY342/CPython with only the documented skips.

**Verification.** `pipenv run check-py3`; run both parity buttons under IPY2, re-attach IPY342, re-run.

---

## Phase 1 — Bridge backlog: collections, `__namespace__`, object-param seams  ·  Size 3

**Goal.** Clear the largest pythonnet marshaling classes and resolve the `__namespace__` semantics the forms modeless tier depends on (R§10.1).

**Key tasks.**
- **Tiered checker rules first:** a `List[T](pylist)` generic-collection classifier (55 sites, tiered 46/7/2) and a `__namespace__`-missing rule (36 sites, 33 missing).
- **Port the 55 generic-collection sites** (~16 lib first) to explicit `List[T]([...])`; wrap returned .NET collections in `list()` where indexed/mutated.
- **Settle `__namespace__`.** Question (b) is settled: re-defining any .NET-derived class collides at the pinned fork commit (R§10.1). In order: (1) run the question-(a) per-interface matrix — ~6–8 interfaces, with/without `__namespace__` × across a reload — via the DevTools button + parity suite; (2) land the reload fix: **primary — port the #2055 type-cache method into `pyrevitlabs/pythonnet` and re-vendor the DLLs, using Autodesk's shipped implementation as the source: `autodesk-forks/pythonnet@11b56b4` ("DYN-2934: Cache class definitions", branch `dynamo_py3`, incl. tests — R§10.1)**; interim — the import-guard pattern, *without* Dynamo's random-suffix namespaces (R§10.1); (3) **only then** apply `__namespace__` per the rule to the 33 sites — deterministic names before the fix make collisions *more* certain.
- Survey/spot-fix the smaller classes: `IDisposable` `with` (4), `.Item[...]` indexers, enum→int, overload/`System.Func` typing.
- **Fix the .NET-side `object`-param seams** (R§10.1 field bug — `print_table` silently empty under CPython): `PyObject` branch in `ToRows`/`ToList`, a visible warning instead of the silent return, and an audit of the remaining `object`-typed APIs (`inject_*`/`add_style`). Invisible to the AST checker; covered by the suite.

**Critical files.** `check_py3_compat.py` (new rules), `revit/db/*`, `revit/events.py`, `dev/pyRevitLabs.PyRevit.Runtime/ScriptOutput.cs`, the `pyRevitLabs.Python.Net` submodule (`ClassDerived.cs` is the cherry-pick site) + the re-vendored `dev/libs/{netfx,netcore}/pyRevitLabs.PythonNet.dll`.

**Depends on.** Phase 0.

**Exit criteria.** New checker rules at zero for `pyrevitlib`; the `__namespace__` rule documented and applied to all lib sites; suite gains passing collection + interface-callback tests on CPython. Extension sites tracked, not necessarily fixed.

**Verification.** Suite tests: a `List[ElementId]` built from a Python list into a Revit API call; a Python-implemented `IExternalEventHandler`/`ISelectionFilter` receiving a live callback; **reload twice** (expected-fail at the pinned commit; becomes the regression test once the fix lands); the **second-click experiment** — a script-body `ISelectionFilter` under `#! python3`, confirming or clearing the R§10.1 suspected live bug; **per-engine output tests** — `print_table`, `print_html_table`, and `inject_to_head`/`add_style` with an `attribs` dict produce visible output under CPython.

---

## Phase 2 — `forms` structural refactor (IronPython-only, behavior-preserving)  ·  Size 4

**Goal.** Split `pyrevit.forms` into the shared-package structure (R§9.1) with **zero behavior change** — the skeleton that makes CPython enablement incremental and drift-proof.

**Target layout.** Only `backends/` is per-engine; every feature module is written once and gets its one engine-specific operation from `_backend`:

```
forms/
  __init__.py            # facade: builds public namespace, explicit __all__
  _backend.py            # selects & re-exports the active engine backend
  backends/
    __init__.py
    _ipy.py              # IronPython core: wpf.LoadComponent, pyevent, clr refs
    _cpy.py              # CPython core: XamlReader loader, event shim (Phase 4)
  base.py                # _WPFMixin, WPFWindow, WPFPanel, WindowToggler, constants
  reactive.py            # reactive, Reactive, TemplateListItem, ParamDef
  dockable.py            # _WPFPanelProvider + register/get/open/close/toggle panel fns
  dialogs.py             # TemplateUserInputWindow, SelectFromList, CommandSwitchWindow,
                         #   GetValueWindow, SearchPrompt
  promptbars.py          # TemplatePromptBar, WarningBar, ProgressBar
  selection.py           # *Option classes + select_* Revit element helpers
  alerts.py              # alert, alert_ifnot, ask_for_* wrappers, ask_to_use_selected,
                         #   ask_for_color, inform_wip
  checks.py              # check_workshared/selection/familydoc/modeldoc/... guards
  pickers.py             # pick_file/pick_folder/save_file/pick_excel_file/save_excel_file
  notify.py              # toast, show_balloon, result_item_result_clicked
  utils.py               # (existing) bitmap/xaml helpers — guard wpf import, route
                         #   XAML load via backend (key task 4)
  toaster.py             # (existing, unchanged — pure subprocess, engine-agnostic)
  settings_window.py     # (existing, unchanged here — subclasses WPFWindow, so it works
                         #   under CPython once Phase 4 Tier 2 lands)
  *.xaml, pyrevit-toast.exe
```

**Key tasks.**
1. **Rebase on Phase 0** so the extraction inherits the already-fixed `_ipy.py` (the `__bool__` aliases, `list()`-wrapped views).
2. **Carve the skeleton without moving logic:** `_backend.py` + `backends/{_ipy,_cpy}.py`; the `_ipy` backend delegates to `wpf.LoadComponent`/`pyevent.make_event` and re-exports the assembly refs. Point `_ipy.py`'s `load_xaml` methods and `utils.py` at it. IronPython behavior byte-for-byte unchanged.
3. **Extract the feature modules** from `_ipy.py`; rebuild `__init__.py` as the namespace assembler with an explicit `__all__` reproducing today's surface exactly (there is no `__all__` today — the surface is every non-underscore name, R§8.1).
4. **Guard `utils.py`'s top-level `wpf` import** so it doesn't hard-fail under CPython (re-routing the body isn't enough); route XAML load via the backend; use `framework.add_reference_to_file` for the backend assembly refs.

**Critical files.** `forms/__init__.py`, `_ipy.py` (source of extraction), `_cpy.py`, `utils.py`, new `_backend.py` + `backends/` + the 10 feature modules. Reference idioms: `compat.py`, `runtime/types.py`, `coreutils/git.py`.

**Depends on.** Phase 0.

**Exit criteria.** IronPython-only end to end; a `dir(forms)` diff shows an identical public surface; every `__all__` name imports; under CPython, `import pyrevit.forms` and `import pyrevit.forms.utils` neither raise nor eagerly load `wpf`.

**Verification.** Attach the dev clone (`pyrevit clones add dev <repo>`; `pyrevit attach dev default --installed`); exercise high-traffic paths unchanged: `alert`, `SelectFromList`, `CommandSwitchWindow`, a custom `WPFWindow` subclass, `ProgressBar`, `select_sheets`, `ask_for_string`. Add a `dir(forms)` parity check to the suite. `pipenv run pyrevit check`; ruff/black clean.

---

## Phase 3 — Binding prototype gate (R§9.4)  ·  Size 1

**Goal.** Decide the reactive/binding backend before building it — highest risk, smallest effort, do it early. The spike decides *only* the binding backend; the C# window-host plumbing for the modeless tier ships unconditionally either way (R§9.3).

**Key tasks.** A throwaway spike on pyRevit's pythonnet fork:
1. **Binding fidelity to a Python/`DynamicObject` source** — `SelectFromList.xaml` as the fixture: `{Binding}`, `DataTrigger`s, `DataTemplate`s, validation, `ICollectionView` sort/group.
2. **Performance on realistic loads** — a ~2,000-item `SelectFromList` and a chatty `ICommand.CanExecute` re-query loop (every call crosses the bridge under the GIL). Correct-but-unusable is a fail.
3. **A modeless `ProgressBar` driving Revit API calls through `ExternalEvent`** — GIL × Dispatcher × Revit-context (R§8.3.5), on the Phase 1 `__namespace__` pattern.

**Depends on.** Phase 1 (`__namespace__`), Phase 2 (structure).

**Exit criteria — a decision, recorded in R§9.4. Pass = fidelity AND performance:**
- **Both pass** → pure-Python `INotifyPropertyChanged` shim.
- **Fidelity passes, perf fails** → hybrid: shim for small dialogs; C# `BindableModel`/bindable collections for large-`ItemsSource`/command-heavy ones.
- **Fidelity fails** → the C# `BindableModel`/`DataTable` adapter (R§9.2).

Feature modules are untouched in every outcome — the backend seam hides the choice.

**Verification.** The spike is the verification; record results, including the perf numbers, in R§9.4.

---

## Phase 4 — `forms` CPython enablement, per tier  ·  Size 5

**Goal.** Turn on CPython forms tier by tier. Per tier: implement `backends/_cpy.py` support, move symbols from stub to real, extend `__all__`, port the shipped extensions that use them; the `__getattr__` fallback covers the rest.

**Tiers (easy → hard; R§8.3 / R§9.3):**

| Tier | Elements | Backend need | Notes |
| --- | --- | --- | --- |
| 0 | `check_*` validators | none | pure logic |
| 1 | `alert`, `pick_file`/`pick_folder`/`save_file`, `ask_for_color`, `toast`, `show_balloon` | none | native dialogs; R§12 idiom cleanups only |
| 2 | `WPFWindow`/`WPFPanel` base via pure-Python `load_xaml_component` | XAML loader | the largest *new* code; no binding yet |
| 3 | `ask_for_string`/`_date`/`_number_slider` | XAML loader | small imperative windows |
| 4 | `SelectFromList`, `CommandSwitchWindow`, `select_*`, parameter/color/image selectors | **binding backend (Phase 3 decision)** | the data-bound tier — the crux |
| 5 | `ProgressBar`, `WarningBar`, dockable panels | **C# window-host base** (unconditional) + `ExternalEvent`/`__namespace__` + GIL | modeless; depends on Phase 1 |

`settings_window.py` (subclasses `WPFWindow`) works once Tier 2 lands; verify under CPython.

**Key tasks.** The pure-Python XAML loader in `backends/_cpy.py` (`XamlReader` parse, copy the tree onto the target, name-scope wiring of `x:Name` controls + handlers — R§8.3.2). Wire the reactive backend per Phase 3. **Build the Tier 5 C# window-host base** (`ExternalEvent` marshaling, ownership/parenting, theming — R§9.2). Port the R§7 corpus (167 forms-consumers) as tiers land; freeze the two `rpw.ui.forms` tools onto legacy (R§7.1).

**Critical files.** `forms/backends/_cpy.py` (grows from stub to real), the feature modules (no per-engine edits expected), `settings_window.py`.

**Depends on.** Phases 1 (Tier 5), 2, 3.

**Exit criteria.** Every public `_ipy.py` symbol works under CPython or appears in a published **coverage matrix**; the parity corpus passes under both engines for shipped tiers; `settings_window.py`/`utils.py` no longer hard-require IronPython. Tiers 0–4 gate Phase 6.

**Verification.** Per tier, run the same functions from a `#! python3` pushbutton and diff observable behavior against IronPython via the parity suite. Risk centers: Tier 2 XAML load, Tier 4 binding.

---

## Phase 5 — Dependency, packaging & interop modernization  ·  Size 2–3

**Goal.** Sane third-party dependencies under one shared interpreter; per-engine vendored trees; modernized interop libraries. Ships before the Phase 6 flip. (R§6, R§7.2)

**Packaging & isolation.**
- **Split `site-packages/` per engine family:** a modern Py3 tree for CPython + a frozen Py2 tree for legacy IronPython; each engine resolves only its own (a loader-path change).
- **Wire pip into the embedded distro + a managed writable user-site**; add `pyrevit env pip install …` (the distro ships no pip today).
- **Curate + pin the shared Py3 env** (platform-SDK model), ABI-matched to CPython 3.12 win-amd64.
- **Declaration + conflict-check:** extensions declare pip requirements in metadata (`"requirements": ["numpy>=2,<3"]`); pyRevit installs at extension-install time (never at script runtime), validates at load, fails loud on conflicts (author-facing semantics: R§6.4). Document top-level bundling as the footgun and namespaced vendoring as the offline fallback.
- **Ship the R§5 isolation mitigations:** the `sys.path` baseline-snapshot fix + the eviction policy split per R§6.5 (evict extension code, never packages).

**Dependency modernization (R§7.2).**
- Drop from the Py3 tree (stdlib in 3.12): `enum`, `pathlib`(2), `scandir`, `unicodecsv`, `importlib_resources`, `pytz`→`zoneinfo`, `six`; `pyevent.py` → frozen tree only.
- Unbundle the orphans (zero first-party consumers): `slackclient`, `bson`, `sqlalchemy`, `munch`.
- Swap `xlrd`+`xlsxwriter` → **openpyxl** in `interop/xl.py`.

**Interop netcore strategy (per module, R§7.2).**
- `dxf.py`: retire `IxMilia.Dxf` for **ezdxf** (pip); rewrite the small consumer surface.
- `rhino.py`: retire `Rhino3dmIO` (native) for **rhino3dm** (pip).
- `ifc.py`: **build a netcore `Ifc.Net`** — not an ifcopenshell swap; it configures Revit's native IFC exporter.
- `adc.py`: stays a managed-assembly load (proprietary; already on the Phase 0 shim).

**Critical files.** `pyRevitfile`, the CLI (`dev/pyRevitLabs/pyRevitLabs.PyRevit/`), `CPythonEngine.cs` (baseline fix + eviction hook), `pyrevit/__init__.py`, `extensions/extpackages.py`, `interop/{xl,dxf,rhino,ifc}.py` and their consumers.

**Depends on.** Phase 0. Runs in parallel with Phases 2–4.

**Exit criteria.** Per-engine tree resolution works; `pyrevit env pip install numpy` works; the eviction policy drops a colliding `get_door` while keeping numpy warm; the baseline bug is fixed (B-after-A case); dropped backports/orphans gone from the Py3 tree; `interop.dxf`/`rhino` work on a .NET 8 host via their pip replacements (or are explicitly deferred with `ifc`).

**Verification.** Suite tests: same-named `lib/get_door.py` in two extensions — each run gets its own; numpy shared and never evicted across extensions; B-after-A baseline pristine. Interop: exercise `dxf`/`rhino`/`xl` from `#! python3` on Revit 2025+ (the case that fails today).

---

## Phase 6 — Flip the default  ·  Size 4

**Goal.** Make `CPY3123` the default via a designed mechanism; keep legacy opt-in, then sunset. (R§11.3)

**Key tasks.**
- **Per-extension `engine` declaration** in extension metadata: `ironpython` keeps today's behavior; `cpython` makes CPython the default for the extension's no-shebang scripts. Per-script shebangs still win.
- Ship the declaration for **≥1 major release** with the IronPython default deprecated-but-unchanged, alongside the coverage matrix + a migration guide (the R§12 checklist; the checker doubles as a check-your-extension tool).
- Shipped extensions declare `cpython` once ported (Phase 4).
- Only then let a global default change affect undeclared extensions — deferrable, stageable, or scoped to new extensions.

**Depends on.** Phase 4 (Tiers 0–4 + matrix), Phase 5.

**Exit criteria.** Declaration shipped ≥1 major release; all shipped extensions declared; coverage matrix complete for Tiers 0–4; isolation mitigations shipped. Then flip; sunset legacy (delete the frozen Py2 tree, `rpw`, vendored `markdown`) once the ecosystem moves.

**Verification.** A no-shebang script in a `cpython`-declared extension runs on CPython; `ironpython`-declared or undeclared extensions unchanged; the shipped-extension corpus passes under the flipped default.

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
- **`framework.add_reference_to_file`** (Phase 0) for all portable assembly loading, including the forms backend.
- The **facade `__getattr__` + `__all__`** scaffold (R§8.1) for incremental forms rollout.
- The **`ScriptOutput`/`PyRevitOutputWindow`** C#-hosted-window pattern for the Tier 5 host base and, if Phase 3 lands there, the C# binding backend.
- The **`compat.py`** `PY2`/`PY3`/`IRONPY` branches and the engine-conditional `clr.AddReference` idioms.
