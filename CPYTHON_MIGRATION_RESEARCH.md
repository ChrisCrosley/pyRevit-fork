# pyRevit → CPython: Research & Analysis

> **Audience:** pyRevit core maintainers.
> **Status:** Research / decision-support. The **single source of truth** for the IronPython → CPython+pythonnet migration; supersedes and consolidates the earlier `IRONPYTHON_TO_PYTHON3_ANALYSIS.md`, `DEPENDENCY_MANAGEMENT_ANALYSIS.md`, and the `Refactor pyrevit.forms…` note. The tactical companion is **`CPYTHON_MIGRATION_PLAN.md`** — this doc is the *why*, that one the *what next*.
> **Scope note:** File/line references reflect the `claude/ironpython-cpython-transition` branch; treat them as pointers, not guarantees.

---

## 1. Executive Summary

pyRevit defaults to IronPython 2.7.12 because IronPython *is* a .NET language — a Python object is a CLR object, so scripts get zero-marshaling access to the Revit .NET API, can subclass .NET types, and can data-bind WPF to Python objects. That same property is why IronPython has been hard to leave. This document maps how much of pyRevit still genuinely depends on IronPython and what it takes to make **CPython 3.12 + pythonnet 3 (`CPY3123`)** the default engine without losing functionality.

**Findings that drive everything:**

1. **IronPython is being demoted, not deleted.** The C# Roslyn loader and the in-process C# bootstrap (PR #3438) move session startup, extension parsing, assembly generation, and UI construction off IronPython, leaving it one opt-in script engine among several.
2. **The syntax/language port is essentially done** (draft PR `fix/improve-python3-support`), along with the test harness that guards it and the first 2 of ~8 pythonnet "bridge" classes.
3. **The engine/session model changes structurally.** IronPython caches *one engine per extension* (isolation by construction); CPython runs *one process-global interpreter* with one shared, never-evicted `sys.modules`. This reshapes **dependency management** (§6) and **module-name isolation** (§5).
4. **`forms/` is the long pole, and its hard core is WPF data-binding to Python objects** — not the XAML loader. The deepest technical risk; prototype-gated (§8–§9).

**Destination:** `CPY3123` as the **default** engine, IronPython kept as an **opt-in legacy engine** until the ecosystem moves, then sunset. The flip is a *designed mechanism* (per-extension engine declaration, §11.3), not a config change.

---

## 2. Runtime Vocabulary (disambiguated)

"Python 3" is ambiguous — it can mean two different runtimes.

| Name | What it *actually* is | Language level | .NET model | C-ext (numpy) | In pyRevit |
| --- | --- | --- | --- | --- | --- |
| **CPython** | The reference interpreter (in C). "Normal" Python. | 3.12 | none by itself | ✓ | embedded as `CPY3123` |
| **IronPython 2.x** | Python reimplemented in C#/.NET; Python objects **are** CLR objects | 2.7 (EOL) | native (is .NET) | ✗ | default `IPY2712PR` |
| **IronPython 3.x** | Same reimplementation, community-revived | ~3.4 | native (is .NET) | ✗ | opt-in `IPY342` |
| **pythonnet** | **Not an interpreter** — a *bridge* letting CPython call .NET | follows CPython | bridge/marshaling | ✓ | the layer over `CPY3123` |
| **PythonNet3** (Dynamo's term) | CPython 3.x + pythonnet 3.x together | current | bridge (v3) | ✓ | = pyRevit's `CPY3123` |

Traps: (a) `#! python3` selects **CPython**, never IronPython 3 — the `ScriptEngineType` enum has no IronPython3 member. (b) pythonnet is a bridge, not a Python. (c) pythonnet **2.x** (weak interop) vs **3.x** (strong interop — out-params as tuples, LINQ, better overloads); pyRevit runs **3.x**, so most "pythonnet can't do X" lore describes the 2.x era.

The two axes that matter: **.NET-native vs bridged** — decides WPF/marshaling behavior — and **can it run C-extensions?** — only CPython can, which is the entire reason to move.

---

## 3. The Three Engines & Selection

| Engine key | Kernel | Role |
| --- | --- | --- |
| `IPY2712PR` | IronPython 2.7.12 (pyRevit fork) | Default |
| `IPY342` | IronPython 3.4.2 (pyRevit fork) | Opt-in, install-level |
| `CPY3123` | CPython 3.12.3 (embedded) via pythonnet | Opt-in, per-script (`#! python3`) |

Engine type is resolved per script (`scriptruntime.cs`): explicit `"type"` in engine configs, else **shebang detection** (`python3`/`cpython` → CPython; anything else → IronPython). CPython vs IronPython is **per-button** (they coexist in-process); IPy2 vs IPy3 is **global** (`#if IPY342` rewrites the `.addin` manifest — the two IronPythons cannot run simultaneously). pyRevit controls the loader, wrapper library, **and all engines** (IronPython 2, IronPython 3, and pythonnet are pyRevit forks under `pyrevitlabs/*`); the only external, unchangeable component is Autodesk's engine-agnostic Revit API.

---

## 4. Migration Already Underway

- **Legacy IronPython bootstrap:** Revit reads `.addin` → `pyRevitLoader.dll` → starts an IronPython engine → `pyrevitloader.py` → `sessionmgr.load_session()` builds the UI.
- **C# Roslyn loader (`pyRevitAssemblyBuilder`, landed, opt-in):** reimplements extension parsing, command-type generation (Roslyn source compilation instead of IronPython `Reflection.Emit`), UI construction, and session management in native .NET. Gated by `user_config.new_loader`.
- **In-process C# bootstrap (PR #3438, in-flight):** removes the IronPython bootstrap entirely; the C# `IExternalApplication` calls the C# session manager directly; Python startup splits into `session_preload.py`/`session_postload.py`. Drops Revit < 2021. The decisive step that turns "IronPython is the foundation" into "IronPython is one engine among several."

---

## 5. Engine Lifecycle & Isolation — the model changes structurally

Verified in `dev/pyRevitLabs.PyRevit.Runtime/` (`ScriptEngineManager.cs`, `ScriptEngines.cs`, `IronPythonEngine.cs`, `CPythonEngine.cs`):

**IronPython:** engines are cached in a dictionary keyed by `SessionUUID : EngineType : CommandExtension` — **all commands in one extension share one cached engine; each extension gets its own.** `clean: true` forces a fresh engine per run; `persistent: true` lets globals survive; the default scrubs scope references after each run but keeps the engine (and its imported modules) warm. Search paths are set *wholesale* per engine.

**CPython:** **one process-global interpreter** — `PythonEngine.Initialize()` runs once (parameterless); pythonnet has **no sub-interpreter support**. Every execution gets a fresh disposable scope (`Py.CreateScope` → exec → `Dispose`). "Refresh engine" is a full `PythonEngine.Shutdown()`, resetting state for *every* CPython command at once.

| | IronPython (default) | IronPython (`clean`) | CPython `#! python3` |
| --- | --- | --- | --- |
| Script globals across runs | scrubbed unless `persistent` | gone (new engine) | always gone (scope disposed) |
| `sys.modules` | cached per extension's engine | fresh | cached **process-wide**, shared by all commands in all extensions |
| `sys.path` | per extension's engine | fresh | rebuilt per run from a baseline + current bundle paths |
| Isolation boundary | **extension** | command execution | **none** (one interpreter) |

**Consequences of one shared interpreter:**

- **`sys.modules` is shared and never evicted.** Extensions ship `lib/` dirs with generic module names (`utils.py`, `config.py`); per-extension IronPython engines make that safe by construction, but under one interpreter **first import wins** — later extensions silently receive the first extension's module. Order-dependent "works unless that other button ran first" bugs.
  - **Worked example:** `A.extension/lib/get_door.py` and a *different* `B.extension/lib/get_door.py`, both `import get_door`. IronPython: isolated. CPython: whichever runs first caches `sys.modules['get_door']`; the other silently gets it — `AttributeError`, or silently-wrong results. Its own `lib/` on the rebuilt `sys.path` is never consulted, because `import` checks `sys.modules` first.
- **`sys.path` is rebuilt per run** (`CPythonEngine.SetupSearchPaths`): baseline snapshot → `PYTHONPATH` → the command's `SearchPaths`. Known ordering bug: the baseline is snapshotted per engine-wrapper *on first use* from the *current* `sys.path`, so if extension B's first CPython run happens after A ran, A's bundle paths bake into B's baseline. **Fix:** snapshot the pristine baseline once, globally, right after `PythonEngine.Initialize()`.
- **The CLR dynamic-type table is a third shared namespace.** When Python subclasses or implements a .NET type, pythonnet emits a CLR type into one process-global dynamic module that **outlives any `sys.modules` eviction** — re-defining the class (on reload, or when two extensions use the same bare class name) re-fires type emission and collides. Settled from source in §10.1's reload-collision note.
- **`clean`/`persistent` bundles** (smartbutton state) have **no CPython equivalent**.

**Mitigations (prerequisites for the flip):** (1) fix the baseline snapshot; (2) a per-run **module-eviction policy** — drop `sys.modules` entries whose `__file__` lives under an extension directory, keeping stdlib/pip modules warm; (3) recommend namespaced lib packages (`import myext_lib.get_door`). **See §6.5 — the eviction policy must never touch C-extension packages.**

---

## 6. Dependency Management

The migration changes how pyRevit packages its **own** dependencies *and* how extensions (including third-party) manage theirs. Two layers, usually conflated.

### 6.1 The model today (verified in code)

- **First-party bundle code — per-component `lib/`** (`COMP_LIBRARY_DIR_NAME='lib'`, `extensions/__init__.py`); registered in `genericcomps.py`, inherited down the bundle tree, added to `sys.path` **first** (highest precedence). Whole-extension shared libs use the `.lib` extension type.
- **Third-party — one shared vendored tree.** `MISC_LIB_DIR = HOME_DIR/site-packages` (`pyrevit/__init__.py`), appended **last** to every extension/script path (`extensionmgr.py`, `sessionmgr.py`; C# path `Constants.SITE_PACKAGES_DIR` in `CommandTypeGenerator.cs`).
- **No per-extension third-party mechanism exists.** `extension.json`'s `dependencies` means *other pyRevit extensions*, not pip packages. No `requirements.txt`, no pip step.
- **The embedded CPython ships no package machinery.** `release/cengines/CPY3123/` is a standard CPython **embeddable** distro: `python312.dll`, stdlib as `python312.zip`, the `.pyd` C-extensions — **no pip, no `Lib/`, no `site-packages/`**. Its `python312._pth` starts the interpreter *isolated* with `import site` commented out, so `site-packages` discovery never runs; pyRevit re-adds its folders manually. **"How does a user get numpy into pyRevit's Python" has no built-in answer today.**

### 6.2 The question: three extensions import numpy

Under one process-global interpreter with one shared, never-evicted `sys.modules`:

- **Same version from the shared tree →** fine, and *optimal*: import once, cached process-wide, everyone shares the warm module. The behavior to design *toward*.
- **Different versions, each bundled per-extension `lib/` → silent first-import-wins.** `import` consults `sys.modules` before `sys.path`; whichever extension runs first caches its numpy process-wide, and the next `import numpy` returns that version, its own `lib/` never consulted.
- **C-extensions physically cannot coexist in two versions in one process.** numpy's dtype registry, `ndarray` type identity, and C-API are **process-global** C state, not namespaced by `sys.modules`. Hard CPython limit. IronPython dodged it by never running C-extensions *and* per-extension engines; CPython gives up both.

### 6.3 Can IronPython's "one environment per extension" be recreated? — No

- **In-process multi-interpreter is blocked:** pythonnet has one process-global interpreter, no sub-interpreters.
- **Faking it via per-extension `sys.modules`/`sys.path` swap** isolates *pure-Python code* but breaks on C-extensions (shared/corrupt C state or a forbidden second init), breaks cross-extension object flow (`isinstance`, C-API), and discards the import-once warm-cache win.
- **Sub-interpreters (per-interpreter GIL, 3.12/3.13):** not viable — pythonnet + many C-extensions don't support them.

### 6.4 Recommended model

| What you want isolated | Mechanism under CPython |
| --- | --- |
| First-party extension **code** (`utils.py`, `get_door.py` collisions) | §5 eviction (refined per §6.5) + namespaced `lib` packages. Cheap, safe; restores IronPython-like isolation for *code*. |
| Third-party **packages** (numpy, pandas) | **Do not isolate.** One curated, version-pinned shared environment — one numpy for everyone. |
| The rare genuine version conflict | **Out-of-process worker** (subprocess venv) — the only real per-extension third-party isolation, at the cost of in-process Revit API access; opt-in for compute-heavy/API-light tools. |

Concrete pieces:
1. **Single curated shared environment (platform-SDK model)** — pyRevit owns one pinned set, ABI-matched to CPython 3.12 `win_amd64`, on a security re-vendoring cadence.
2. **Managed writable user-site + a `pyrevit` CLI pip command** (`pyrevit env pip install …`) — solves the "how do I get numpy" UX cost. Still one shared tree, one version per package. Requires wiring pip into the embeddable distro (absent today).
3. **Declaration + conflict-check, not isolation** — extensions *declare* pip requirements that resolve against the shared env and **fail loud** on incompatible demands (vs today's silent first-wins). Extends the `dependencies` concept to pip.
4. **Per-extension bundling documented as an unsupported footgun** — works in single-extension dev, breaks under multi-extension production; C-extensions can't be isolated at all.
5. **Per-engine `site-packages` split** — a modern Py3 tree for CPython and a *frozen* Py2 tree for the legacy IronPython engine (the Py2 backports `six`, `pathlib2`, `scandir`, `unicodecsv`, … live only in the frozen tree, deleted wholesale when legacy sunsets).

#### What this means for third-party extension authors

*"My extension needs a pip package — do I commit it to my repo, or does pyRevit pip-install it?"* **Neither ship it nor ask users to install it manually — declare it, and pyRevit installs it.** The repo carries code only; metadata declares pip requirements as version *constraints* (e.g. `"requirements": ["numpy>=2,<3", "openpyxl"]`). pyRevit resolves against the shared environment and installs **at extension-install time** (never at script runtime — no network surprises mid-session), **validates at load time**, and fails loud on conflict: A demands `numpy<2`, B demands `numpy>=2` → an actionable error instead of silent first-import-wins. One version per package process-wide is the physical constraint (§6.2–§6.3), not a policy preference — the declaration model is the only one the runtime can honor.

| Author's option | Verdict | Why |
| --- | --- | --- |
| **Declare** pip requirements in extension metadata | **The supported path** | pyRevit installs into the shared env at install time, conflict-checks at load; deps keep receiving pip security updates instead of fossilizing in the repo. |
| **Vendor inside your own package namespace** (`myext_lib/vendored/pkg/`, imported as `from myext_lib.vendored import pkg`) | Acceptable fallback (offline installs, tiny unmaintained helpers) | The `sys.modules` key is `myext_lib.vendored.pkg` — collision-free by construction; the §6.5 eviction policy handles it as extension-namespace code. **Pure-Python only, never C-extensions.** |
| **Bundle top-level in `lib/`** (`import requests` resolves to your private copy) | **The documented footgun** | First-import-wins across extensions (§6.2); C-extensions are also ABI-coupled to the embedded CPython and cannot coexist with another version. Works in single-extension testing, breaks in multi-extension production. |
| Instruct users to pip-install into a system CPython + `PYTHONPATH` | Today's workaround, not a target | Works because `CPythonEngine` appends `PYTHONPATH` dirs, but manual, per-machine, ABI-fragile. Superseded by the declaration model. |

Until the Phase 5 mechanism ships, **no supported mechanism exists** — namespaced vendoring and `PYTHONPATH` are the interim guidance, and the migration guide should say so explicitly.

### 6.5 The eviction-policy hazard (a latent bug in the §5 mitigation)

The §5 eviction rule ("evict `sys.modules` entries under an extension dir") correctly fixes first-party name collisions — that code is pure-Python and re-importable. It is **actively unsafe** the moment it touches a C-extension: numpy/pandas run C init exactly once; evicting and re-importing corrupts or crashes them (numpy refuses re-init). If an extension bundles numpy under its own `lib/`, the naive rule targets it. **Required refinement:** evict pure-Python *extension-namespace code* only — maintain an explicit extension-namespace allowlist and keep everything else warm. **Evict code; never evict packages.**

---

## 7. Quantifying the IronPython Coupling

**`site-packages/` (vendored, 33 packages):** almost entirely pure-Python (only `sqlalchemy` carries optional C-ext artifacts with a pure-Python fallback). The **Python-2 backports** (`six`, `enum`, `pathlib2`, `scandir`, `importlib_resources`, `unicodecsv`) are the telltale of the IronPython-2.7 target — frozen-tree candidates (§6.4.5).

**`pyrevitlib/` (first-party):** `pyrevit` (~200 files, mixed), `rpw` (frozen legacy, §7.1), `rpws`/`rjm`/`rsparam` (pure-Python, compatible). Within `pyrevit`: `coreutils` (syntax + 2 CLR shim sites), `revit` (bounded §12 idiom port), `interop` (9 CLR shim sites), `routes`/`extensions` (compatible), `runtime`/`loader` (superseded by the C# loader), `forms` (**major work**, §8), plus `framework.py` (*the* CLR-shim hub, 30 `AddReference` calls) and `compat.py` (already `PY2`/`PY3`/`IRONPY` branched).

**Syntax portability — cleared by the draft PR (§10):** the residual Py2-only idioms (`iteritems`, bare `unicode`, `__nonzero__`, heterogeneous sorts, `ifilterfalse`, Py3 view/iterator escapes) are fixed and guarded by a CI checker.

**CLR interop:** `import clr`, `clr.AddReference`, `System.*` work under **both** engines. The one widely-used IronPython-*only* API, `clr.AddReferenceToFileAndPath`, is now shimmed by the PR's `framework.add_reference_to_file` (§10), removing it from all call sites outside `framework.py`.

**`revit/` marshaling hotspots** — bounded and enumerable: ~18 event hookups, generic-collection constructions, 2 `out`/`ref` sites (`query.py`, `create.py`, ported by the PR). Already partially dual-targeted (`query.py` branches on `PY3`; `events.py` gates on `IRONPY`).

**Shipped `extensions/`** (the parity corpus): 441 first-party scripts; **167 import `forms`**, 35 author custom WPF, **zero end-user tools carry `#! python3`** — they break first under any default flip and double as the parity test corpus.

### 7.1 `rpw` and `markdown` — vendored-library policy

- **`pyrevitlib/rpw`** (revitpythonwrapper 1.7.4, unmaintained): **engine-locked legacy.** Its `rpw.ui.forms` subclasses WPF and loads `IronPython.Wpf` — cannot run on CPython. Supported only under the opt-in legacy engine; `pyrevit.forms` is the documented replacement. Core's one dependence (`revit/db/pickling.py: from rpw import doc`) is trivially replaceable.
- **`pyrevit.coreutils.markdown`** (python-markdown 2.6.8): **deprecated, portable orphan.** Pure Python and engine-agnostic, but orphaned when the output window moved rendering to C# (`ScriptOutput.cs`) — zero first-party runtime consumers. The PR keeps its syntax fixes but stops investing (removed from checker/suite). Known crash: conversion overflows the stack under IPY342 in Revit (fat DLR frames + recursion-heavy parser). Unbundling candidate; `#! python3` scripts should use pip `markdown`.

Both are deletion candidates when the legacy engine sunsets.

### 7.2 Dependency modernization — swap, drop, or bump

Consumer counts below are first-party `.py` (the telemetry server is Go, so vendored packages it doesn't use are dead weight). One correction to the generic §12 checklist: **pyRevit has no Excel COM** — `interop/xl.py` is pure-Python `xlrd`/`xlsxwriter`, with no `Microsoft.Office.Interop.Excel`/`GetActiveObject` anywhere in first-party code — so "Excel COM → openpyxl" does not apply; the real Excel issue is xlrd's `.xlsx` drop, below.

**The headline: CPython turns the worst `interop` dependency — .NET/native assemblies missing on .NET 8 (Revit 2025+) — into clean pip installs.** `interop.rhino`/`dxf`/`ifc` load managed (for Rhino, also **native**) assemblies that ship only in the netfx lib set, so they can't import on Revit 2025+ on *any* engine. The strategy fork per module: **build netcore copies** vs **retire the .NET wrapper for a pip-native library**.

| Today | Modern alternative | Used by | Fit / cost |
| --- | --- | --- | --- |
| `xlrd`+`xlsxwriter` (`interop/xl.py`, file I/O) | **openpyxl** | `xl.py` (71 lines) | **Clean swap.** Also fixes xlrd 2.0 dropping `.xlsx`. |
| `IxMilia.Dxf` (.NET, missing on .NET 8) via `interop/dxf.py` | **ezdxf** (pip) | thin 10-line loader; consumers use `IxMilia` API | **Strategic swap** — becomes `pip install ezdxf`; rewrite the DXF-building consumers (small surface). |
| `Rhino3dmIO` (.NET + **native** `rhino3dmio_native.dll`, missing on .NET 8) via `interop/rhino.py` | **rhino3dm** (pip, McNeel) | thin 10-line loader; 1 DevTools test | **Strategic swap** — kills native-binary + netcore + opt-in-risk at once; rewrite consumers. |
| `Ifc.Net` via `interop/ifc.py` | ~~ifcopenshell~~ — **not a swap** | `ifc.py` (318 lines) + 1 dev example | **Keep or netcore-build.** `ifc.py` uses `Ifc4` schema types to configure Revit's *native* IFC exporter (parses Revit's JSON, builds `IFCExportOptions`); ifcopenshell is a standalone authoring toolkit — a different job. |
| Autodesk Desktop Connector (`interop/adc.py`) | none (proprietary) | ADC integration | Stays a managed-assembly load. |

**site-packages modernization:**

| Class | Packages | Action |
| --- | --- | --- |
| Py2 backports → stdlib | `enum`, `pathlib.py`+`pathlib2`, `scandir`, `unicodecsv`, `importlib_resources`, `pytz`→`zoneinfo` | **Drop from the Py3 tree** (stdlib in 3.12); frozen legacy tree only. |
| Py2-only | `six.py`; `pyevent.py` | Drop `six`; `pyevent.py` is IronPython-only (§8.3.1) → frozen tree, replaced by the forms reactive backend. |
| Orphaned (0 first-party consumers) | `slackclient`, `bson`, `sqlalchemy`, `munch` | **Unbundle candidates.** If Slack is revived, `slackclient` is the *deprecated* SDK → `slack_sdk`. |
| Unmaintained but used | `docopt` (2 consumers) | → `argparse` if touched anyway; low priority. |
| Keep (maintained, pure-Python) | `requests`+stack, `werkzeug`, `websocket`, `xlsxwriter`, `pyparsing`, `natsort`, `sortedcontainers`, `filelock`, … | Re-vendor current majors (or pip) into the curated Py3 env; watch major-version breaking changes (§11). |

---

## 8. The `forms/` Layer — the Long Pole

### 8.1 Current state (verified)

`forms/__init__.py` (20-line facade) dispatches on `compat.IRONPY` to `_ipy.py` (**4,105 lines**, the real WPF implementation) or `_cpy.py` (**103 lines of stubs** + a module-level `__getattr__` raising `PyRevitCPythonNotSupported` for anything not stubbed). Also in scope: `settings_window.py` (705 lines, subclasses `WPFWindow`, **not engine-gated**), `utils.py` (63 lines, imports the IronPython-only `wpf` module **at module top → hard-fails on import under CPython**), `toaster.py` (portable subprocess), and **21 `.xaml` files** authored against the LoadComponent-into-self model. Port scope: **~4,900 Python lines + 21 XAML files**. Under `#! python3` today, importing `pyrevit.forms` succeeds (stubs load no WPF) but **any actual use raises**.

### 8.2 Why a pythonnet port is hard

IronPython *is* .NET — Python objects are visible to WPF's reflection/binding engine. pythonnet is a *bridge* — Python objects are proxies, invisible to .NET reflection. Almost everything hard about `forms` flows from that asymmetry.

### 8.3 The blockers, ranked — and the crux, validated in code

The `Refactor pyrevit.forms…` note argued the engine-specific surface is *tiny* — essentially `wpf.LoadComponent` and `pyevent.make_event()` — and everything else "runs identically under pythonnet." **The first half is right; the second half is not:**

1. **WPF data-binding to Python objects — THE deepest blocker, not confined to exotic MVVM.** The workhorse dialogs bind XAML directly to Python objects:
   - `SelectFromList` sets `self.list_lb.ItemsSource = ObservableCollection[TemplateListItem](…)`; `SelectFromList.xaml` binds `{Binding name}`, `{Binding checked}`, `{Binding checkable}`.
   - Parameter pickers bind `{Binding displayvalue}`, `{Binding istype}`, `{Binding isbuiltin}`; the color swatch `{Binding hex_color}`; the image list `{Binding item}`.
   - `TemplateListItem(Reactive)` → `Reactive(ComponentModel.INotifyPropertyChanged)` via `pyevent.make_event()`; `@reactive` properties raise `OnPropertyChanged`.
   Under IronPython these resolve because the item *is* a CLR object with reflectable properties. Under pythonnet, WPF binding resolves via `TypeDescriptor`/`PropertyDescriptor`, which generally **cannot see Python-defined properties** — fabricating the `INotifyPropertyChanged` event is necessary but not sufficient (the event says "something changed"; binding still can't find `name`). So `SelectFromList` and the selectors sit on the same unproven crux as full MVVM — asserted-not-proven on both sides, hence the §9.3 prototype gate.
2. **`wpf.LoadComponent(self, …)`** — no pythonnet `wpf` module. The CPython loader must parse with `XamlReader`/`Application.LoadComponent`, copy the tree onto `target`, walk the name-scope to wire `x:Name` controls as attributes, and connect XAML-named handlers to Python methods — three things `XamlReader.Load` doesn't do together. Largest piece of *new* code; viable in pure Python.
3. **.NET subclassing lifecycle** — `class WPFWindow(_WPFMixin, Window)` + "construct empty, then `LoadComponent(self)`" isn't how pythonnet subclassing works; needs `super().__init__()` and, for interfaces, `__namespace__` (§10 backlog).
4. **Modeless dialogs that call the Revit API** (`ProgressBar`, `WarningBar`, dockable panels) must marshal back via `ExternalEvent`/`IExternalEventHandler` — needs `__namespace__` + GIL care; depends on the §10.1 resolution.
5. **Threading / GIL × WPF Dispatcher × Revit single-threaded context** — a real source of hangs/reentrancy to re-validate, not just port.

---

## 9. The Chosen UI Direction (reconciled)

The two prior documents pointed in different directions; both are partly right, and the shared structure reconciles them.

### 9.1 Adopt the shared-package structure (from the refactor note)

Split `pyrevit.forms` along **two axes**: a horizontal *engine-core vs shared-feature* boundary and a vertical *feature-area* boundary within the shared code. Only the engine core (`backends/`) is duplicated per engine; feature modules are written once, importing their one engine-specific operation (`load_xaml_component`, `make_property_changed_event`, assembly refs) from `_backend`. The annotated target layout is the plan's Phase 2 tree. This kills the `_ipy`/`_cpy` monolith divergence and is the right skeleton regardless of the binding bet.

### 9.2 But keep the original analysis's C# hedge for the binding tier

The original analysis's *chosen* direction was a **C#-assisted UI API** (`forms.DataGrid` over a `DataTable`; `forms.BindableModel` — a C# `INotifyPropertyChanged`/`DynamicObject` core WPF binding *can* see), precisely because binding-to-Python-objects (§8.3.1) may be unsolvable in pure Python under pythonnet. The `ScriptOutput`/`PyRevitOutputWindow` console already proves the pattern: a C#-hosted WPF window driven from Python via a thin `__getattr__` wrapper, working on every engine.

**Design rule for any C# API serving both engines** (learned from the pattern's one field failure, the `print_table` bug in §10.1): **never type-sniff an `object` parameter.** Under IronPython a Python list/dict *is* a CLR collection and `value as IEnumerable` succeeds; under pythonnet the same argument arrives as a `PyObject` proxy and the cast silently yields `null`. C# surfaces like `BindableModel`/`DataGrid` must either take **typed parameters** (forcing pythonnet conversion), handle **`PyObject` explicitly** (iterate under `Py.GIL()`), or have the thin Python wrapper convert before crossing — and must **fail loud, never silently no-op**, on an unrecognized argument shape.

#### What "host plumbing" actually is — the five mechanics

"Plumbing" = the Revit-host integration mechanics a window needs to live correctly inside the Revit process — everything *except* how data gets into the controls. It is identical whichever way `{Binding}` resolves, which is why it is separable from (and unconditional relative to) the §9.4 binding bet. In today's `_ipy.py`:

| # | Mechanic | Today (in `_ipy.py`) | First needed | Home under CPython |
| --- | --- | --- | --- | --- |
| 1 | **Window ownership/parenting** — child the WPF window to the Revit main window so it minimizes/restores with Revit and stays in front of it | `Interop.WindowInteropHelper(self).Owner = AdWindows.ComponentManager.ApplicationWindow` (`_ipy.py:501`) | Tier 2 (every window, modal included) | plain .NET calls → pure-Python base |
| 2 | **`ExternalEvent` / `IExternalEventHandler` marshaling** — a *modeless* window lives outside Revit's API context and cannot call the API directly; it must raise an `ExternalEvent` and do the API work in the `Execute(uiapp)` callback Revit invokes on the main thread | not needed by today's modal-heavy dialogs; required by `ProgressBar`-with-API and dockable panels | Tier 5 | **C# host base** (under pythonnet also needs `__namespace__` — Phase 1) |
| 3 | **Threading / Dispatcher marshaling** — background work thread + `Dispatcher.Invoke` pushing UI updates back onto the WPF UI thread | `self.pbar.Dispatcher.Invoke(System.Action(...), DispatcherPriority.Background)` (`_ipy.py:365`, `2016`) | Tier 5 (`ProgressBar`) | **C# host base** — the GIL × Dispatcher × Revit-context hang seam (§8.3.5) |
| 4 | **Theming / resource injection** — merge pyRevit's dark-mode brushes (`pyRevitDarkColor`, `pyRevitAccentBrush`, …) and MahApps styles into each window's `Resources` | `_ipy.py:211–227` | Tier 2 (every window) | plain .NET calls → pure-Python base |
| 5 | **Dockable-pane registration lifecycle** — `IDockablePaneProvider` + `RegisterDockablePane` with a stable `DockablePaneId`; Revit owns the pane's lifecycle thereafter | `_WPFPanelProvider` (`_ipy.py:656–732`) | Tier 5 | **C# host base** |

(Plus the mundane sixth: resolving `PresentationFramework`/`PresentationCore`/`WindowsBase`/`System.Xaml` — and the .NET 8 Desktop runtime bits on Revit 2025+ — via the Phase 0 CLR shim.)

Items 1 and 4 arrive at Tier 2 but are plain .NET calls that ride along in the pure-Python base. The **C# host base exists for items 2, 3, and 5** — the mechanics genuinely hazardous for a Python author to re-solve per dialog. This burden is also what separates the modeless tier from the leaf tier: `alert`/`pick_file`/`ask_for_color` are modal, native, and in-context, needing *none* of the five — hence pure Python, zero C#. The §10.1 reload-collision finding adds a fourth reason: with the `ExternalEvent` handler implemented once in C#, per-dialog Python never implements a .NET interface at all, sidestepping the derived-type re-definition hazard for the most common modeless case.

**Phase timing:** *researched* in Phase 1 (`__namespace__` per-interface + reload semantics), *prototyped* in Phase 3 (the spike's modeless-`ProgressBar` leg is effectively the plumbing prototype), *productionized* in Phase 4 as the Tier 5 C# window-host base.

### 9.3 Resolution — prototype-gated, structure fits both (the decision)

- **Native/leaf tier** (alert, pickers, `ask_for_color`, toast, `check_*`, orchestration): plain .NET/Revit-API calls; port directly under pythonnet with only §12 idiom cleanups. No C#.
- **XAML-into-self base** (`WPFWindow`/`WPFPanel`): the pure-Python `load_xaml_component` (XamlReader + name-scope wiring) is viable. Adopt.
- **Data-bound / reactive tier** (`SelectFromList`, selectors, MVVM custom dialogs): **gate on the §9.4 binding prototype.** The `reactive` backend resolves to **either** a pure-Python `INotifyPropertyChanged` shim (if the prototype proves binding to Python objects on pyRevit's pythonnet fork) **or** a C# `BindableModel`/`DataTable` adapter. The shared-package structure makes the swap invisible to the feature modules.
- **Modeless / host-plumbing tier** (`ProgressBar`, dockable panels): the **C# window-host base is unconditional** — plumbing items 2/3/5 of the §9.2 enumeration ship in C# regardless of the prototype outcome. Only the *binding backend* is gated; a spike pass does not mean "no C# anywhere."

**Why the gate ordering is not a pure-Python preference.** The prior UX argument for the C# layer compared it against *unassisted DIY pythonnet* (manual `FindName`, hand-wired events, hand-rolled `DataTable` conversion), under the premise that binding-to-Python-objects was impossible. It never compared C# against a *working, pyRevit-shipped* pure-Python `Reactive` shim — the world the spike tests for. If the shim works, it reproduces **today's IronPython authoring model exactly** (subclass `Reactive`, `@reactive` properties, `{Binding name}` in XAML): existing custom dialogs port nearly unchanged, no new API — the shim is the *better* author UX in that world, not a consolation. If it fails (or is too slow), `BindableModel` is the only way to get declarative binding and wins outright. Each option is the UX winner in the world where it is chosen. The "one implementation serves all engines" maintenance argument for C# is now carried by the shared-package structure itself (§9.1).

### 9.4 The prototype gate (must pass before committing the binding backend)

A sharply-scoped spike on **pyRevit's pythonnet fork**; **pass = fidelity AND performance** (correct-but-unusable is a fail):
1. **Binding fidelity to a Python (or `DynamicObject`) source** — not just `{Binding name}`, but `DataTemplate` resolution, `DataTrigger`s (the parameter pickers use them), validation, and `ICollectionView` sort/group. `SelectFromList`'s real XAML is the fixture.
2. **Performance on realistic loads** — a ~2,000-item `SelectFromList` (sheets-scale) and a chatty `ICommand.CanExecute` loop: `CommandManager.RequerySuggested` re-queries frequently on the UI thread, and with a Python source every call crosses the bridge and takes the GIL. The strongest surviving argument for the C# backend even if fidelity passes; measure, don't assume.
3. **A modeless `ProgressBar` driving Revit API calls through `ExternalEvent`** (the §8.3.4 combination) — measuring GIL × Dispatcher × Revit-context.

**Outcomes.** Both pass → pure-Python shim. Fidelity passes, perf fails → **hybrid**: shim for small value-input dialogs, C# `BindableModel`/bindable collections for large-`ItemsSource` and command-heavy dialogs (the backend seam hides the split). Fidelity fails → `DataTable`-backed binding + imperative control access covers the native/leaf and list tiers; only reactive-MVVM custom dialogs are lost — the approach degrades to "C# host plumbing + `DataGrid`" rather than collapsing.

---

## 10. What the Draft PR Establishes (`fix/improve-python3-support`)

The first step of the journey, split along one clean axis:

- **Language layer (breaks on *any* Py3 engine, IPy3 included):** `iteritems`, bare `unicode`, `__nonzero__`/`__bool__`, heterogeneous sorts, `ifilterfalse`, Py3 view/iterator escapes — **cleared comprehensively**, including the residuals *inside* `forms/_ipy.py` (the `__bool__` aliases; `list()`-wrapped views in `search_matches`/grouped select).
- **Bridge layer (pythonnet-only): 2 of ~8 classes cleared** — IronPython-only CLR loading (Stage 2 → public `framework.add_reference_to_file`, the **exact shim the forms backend will reuse**) and `clr.Reference` out-params (Stage 3 → `query.intersect_curves`, `create.load_family`, with the `__namespace__` + tuple-return conventions).
- **Test harness (the reusable safety net):** an AST checker (`dev/scripts/check_py3_compat.py`, CI-gated at zero) and an in-Revit `test_py3_compat` suite run by two DevTools buttons as a **per-engine parity dashboard** (IPY2 / IPY342 / CPython). The forms port and the bridge backlog plug into *this*.
- **Honest scope:** does not touch the forms WPF port or engine selection; surveys the remaining bridge classes as measured future work (§10.1).

### 10.1 The measured bridge backlog

| Bridge idiom | Sites | Notes |
| --- | --- | --- |
| **Generic collection construction** (`List[T](pylist)`) | **55** | 46 high-confidence / 7 review / 2 already-.NET; ~16 lib, ~39 ext. Largest class; natural first slice. |
| **Interface impls needing `__namespace__`** | **36** | only 3 carry it; `ISelectionFilter` (~15), `IExternalEventHandler` (~8), `IDuplicateTypeNamesHandler` (~6), … Applied inconsistently even within one file (`events.py`). Of the two Stage 3 open questions, **(b) is settled from source — reload re-definition collides at the pinned fork commit; see the reload-collision note below.** (a) — needed per-interface? — remains empirical. Same shared-process-state class as §5/§6; gates the forms modeless tier (§8.3.4). |
| **Collections returned as views** (wrap in `list()`) | unsized | not cleanly AST-detectable. |
| **`.Item[…]` / bracket-index on `IEnumerable`** | 2+ | plain `[i]` not statically distinguishable from Python indexing. |
| **`with` on `IDisposable`** | 4 | modern pythonnet may already cover; verify in-Revit. |
| **Overload resolution / LINQ `System.Func[…]`** | 1+ | mostly unsurveyed. |
| **Enum→int not implicit; `super().__init__()`; COM/GAC lost on .NET Core** | unsized | `super()` largely in `forms`; COM does not apply to pyRevit (§7.2). |
| **.NET-side `object`-typed API seams** (C# receiving Python objects) | 6+ confirmed | The **reverse direction** of every class above, and invisible to the AST checker — the Python call site is idiomatic (`output.print_table(data)`); the defect is C# type-sniffing an `object` param that receives a `PyObject` proxy and silently gets `null`. Confirmed in `ScriptOutput.cs`: `print_table`/`print_html_table` (`ToRows`/`ToList`) and the `object attribs` params of `inject_to_head`/`inject_to_body`/`inject_script`/`add_style`. Any other snake_case C# API taking `object` needs auditing. |

These are **migration-readiness totals, not active-bug counts** — most extension sites are IronPython-only today. The exception is the `object`-seam class, a **live bug today** for any `#! python3` script:

#### Known field bug: `output.print_table()` silently prints nothing under CPython

Reported in the field: *"if the script is running with CPython, `output.print_table()` has no output; only after removing the `#! python3` line the script works."* Traced and confirmed on this branch:

1. `output.print_table()` (`pyrevitlib/pyrevit/output/__init__.py:525`) is a thin forwarder — it passes the Python `list`-of-`list`s straight to the C# runtime output object.
2. `ScriptOutput.cs:477` `print_table(object table_data, …)` calls `ToRows(table_data)` and **returns silently** when the result is empty (`if (rows.Count == 0) return;`).
3. `ToRows` (`ScriptOutput.cs:780`) does `value as IEnumerable`. Under **IronPython** a Python list *is* a CLR `IEnumerable` — the cast succeeds. Under **CPython/pythonnet 3** a Python list passed to an `object` parameter arrives as a `PyObject` proxy, which does not implement `IEnumerable` — the cast yields `null`, `rows` is empty, no output, no error.

`print_html_table` shares `ToRows` but at least emits a visible "No table_data list" warning; `print_table` fails silently. The `attribs` dict params of `inject_*`/`add_style` hit the same seam. Typed-parameter APIs (`set_font`, `resize`, …) are unaffected — typed params force pythonnet conversion.

**Fix shape (Phase 1):** add a `PyObject` branch to `ToRows`/`ToList` iterating the proxy under the GIL (`ScriptOutput.cs` shares an assembly with `CPythonEngine.cs` and already references `Python.Runtime`); replace `print_table`'s silent return with the visible warning. A Python-side conversion would fix only that path; the C#-side fix protects every caller.

**Why this matters beyond `output`:** it is the one failure mode of the `ScriptOutput` thin-wrapper precedent the §9.2 forms C# layer is modeled on — see the design rule there. It also corrects this document's earlier framing of `output` as simply "compatible": the *hosting pattern* is proven on every engine, but `object`-typed API seams are engine-divergent.

#### The `__namespace__` reload collision — settled from source (fork pinned at `f591d92`)

**What `__namespace__` is.** Not Python — a pythonnet dunder. When Python subclasses or implements a .NET type, pythonnet emits a real CLR type behind it via `Reflection.Emit`; `__namespace__` sets that generated type's namespace. Irrelevant under IronPython (Python classes *are* CLR types natively), which is why it surfaces only now.

**Verified on this branch.** pyRevit does not ship stock PyPI pythonnet: the engine is the `pyrevitlabs/pythonnet` fork (branch `pyrevit-5-main`), vendored as committed DLLs (`dev/libs/{netfx,netcore}/pyRevitLabs.PythonNet.dll`), submodule pinned at `f591d92`. `src/runtime/Types/ClassDerived.cs` at that commit: `CreateDerivedType` caches only the assembly/module *builders* — **no generated-type cache, no existence guard** before `moduleBuilder.DefineType(name, …)`; the emitted type name is `namespaceStr + "." + name` when `__namespace__` is set and the **bare Python class name** otherwise; and every derived type in the process lands in **one cached dynamic module** (`Python.Runtime.Dynamic[.dll]`).

**Consequences — question (b) answered, more broadly than asked:**

- **Re-defining any .NET-derived class collides, with or without `__namespace__`.** Same type name, same reused module → the second `DefineType` throws "Duplicate type name within an assembly." `__namespace__` changes *who you collide with*, not *whether*: it prevents cross-extension name collisions while doing nothing for reload.
- **Without `__namespace__`, two extensions defining the same bare class name collide with each other** — the §5 `get_door` collision at the CLR type level. The CLR dynamic-type table is a **third process-global shared namespace** (after `sys.modules` and the `sys.path` baseline), and the §5 eviction mitigation **cannot protect it**: evicting the Python module doesn't evict the generated CLR type, and re-import re-fires `DefineType`.
- **Suspected live bug — unconfirmed, needs in-Revit reproduction.** Under scope-per-run, a *script-level* .NET-subclassing class body (e.g. a script-body `ISelectionFilter`) re-executes on **every button click**, so the source reading predicts a duplicate-type failure on the **second click** today. Confirm via the DevTools "Test CPython Namespace" button; if it reproduces, it joins `print_table` as an active `#! python3` bug.

**The fix and its two venues.** Upstream pythonnet PR **#2055** — authored by pyRevit's own maintainer, per the external review — adds exactly the missing machinery (generated-type caching, deterministic naming, no re-definition on reload) but was **closed unmerged upstream**, and its changes are absent from the pinned fork commit (confirmed from source). Because pyRevit ships its own fork, upstream's closure is not binding:

1. **Primary: cherry-pick #2055 into `pyrevitlabs/pythonnet` and re-vendor the two DLLs.** Fixes the whole class globally — including third-party scripts that subclass .NET at script level, which no pyRevit-side convention can reach.
2. **Belt-and-braces + interim: define-once conventions** — handler classes live in a runtime module the eviction policy never evicts (the class object stays identical across runs, so `DefineType` never re-fires), or an explicit define-once guard.

**Dynamo corroboration — verified against the live posts.** Dynamo hit this identical bug on the identical interface: their worked example is `MyFamilyLoadOptions` implementing `IFamilyLoadOptions` — the same interface as pyRevit's Stage 3 `FamilyLoaderOptionsHandler`. Verified verbatim from dynamobim.org:

- **Autodesk shipped the engine fix — production precedent for venue 1, and the fix commit is public and citable.** The "PythonNet3" post: the .NET-inheritance bug "is fixed starting in PythonNet3 version 1.1.0. Inheritance now works just as you'd expect… *(Thanks to [pythonnet/pythonnet#2055] for the method.)*" — note "1.1.0" is the version of Dynamo's engine *package* named PythonNet3, not a pythonnet-library version. The whole stack is open source and was traced for this document: the engine package is [DynamoDS/PythonNet3Engine](https://github.com/DynamoDS/PythonNet3Engine) (`DSPythonNet3.csproj` references NuGet **`Autodesk.pythonnet` 3.1.1-preview** — the fork's own version numbering; upstream pythonnet never shipped 3.1.x), whose source is [autodesk-forks/pythonnet](https://github.com/autodesk-forks/pythonnet), working branch `dynamo_py3`. **The fix is commit [`11b56b4`](https://github.com/autodesk-forks/pythonnet/commit/11b56b4) — "DYN-2934: Cache class definitions" (twastvedt, 2025-01-28)**: a static cache of generated derived types, deterministic base-chain naming (`CreateUniqueTypeName` → `Python.Runtime.Dynamic.Base__Iface1__…__SubClass`), and a `TryGetValue` existence check before `DefineType`, plus ~97 lines of subclass tests — an independent implementation of #2055's method (not a literal git cherry-pick; the "(#65)" in its title is a pre-public internal PR number). After the fix, their example needs nothing but a plain `__namespace__` — no guard. **For pyRevit's venue 1 this commit, not the closed PR, is the better porting source:** shipped, tested, and on an MIT-licensed public fork.
- **The documented pre-fix workaround is an import-guard, verified viable on pyRevit's pinned fork.** The community pattern (Cyril Poupin): a façade class whose `__new__` tries `__import__(cls.__namespace__)._InnerClass(*args)` and defines the inner `__namespace__`-pinned class only on `ImportError`. A `__namespace__`-registered CLR type's namespace becomes *importable*, so the guard queries **pythonnet's own type registry** rather than parallel bookkeeping. Verified at the pin: `ClassDerived.cs:291–295` calls `AssemblyManager.ScanAssembly(assembly)` after every `CreateType()`, so previously registered namespaces resolve through the import hook — the guard works on `f591d92` as-is. This is the concrete form of the venue-2 pattern.
- **Caution when borrowing:** Dynamo's random-suffix namespaces (`MyFamilyLoadOptions_xpkb`, `ViewModel_jhggsbUbwQpY` — "rename it each edition") are per-node-uniqueness and dev-iteration hacks for a graph environment; copied literally into a long-lived Revit session they leak one CLR type per variant, forever. The reusable part is the import-guard; the rename-on-edit caveat is this collision observed in the wild — a dev-time footgun only the engine fix removes.

**Sequencing consequence (binding on Phase 1):** do **not** blanket-apply `__namespace__` to the 33 missing sites before the reload fix lands — deterministic names without a type cache make reload collisions *more* certain. Question (a) — which interfaces need `__namespace__` at all — remains empirical: "need" is a property of whether the .NET side name-resolves or reflects on the generated type, not of the interface; settle it with a per-interface matrix (~6–8 interfaces, with/without `__namespace__` × across a reload) via the DevTools button + parity suite. Dynamo's WPF viewmodel (`class ViewModel(INotifyPropertyChanged)` carrying the same `__namespace__` + rename-on-edit treatment) is consistent with the name-resolution hypothesis but doesn't decide it — the matrix does. (An upstream issue on forms-under-CPython, pyrevitlabs/pyRevit#3033, reportedly tracks the same gap; not independently verified from this session.) The finding also strengthens the §9.2 Tier 5 decision: with the `ExternalEvent` handler in C#, per-dialog Python never implements a .NET interface and sidesteps this entire class.

---

## 11. Decision Factors, Risks, Security

**Ecosystem unlock** is real but **already works today** (opt-in via `#! python3`); the plan makes it the default and removes the gate. **Backward-compat break is the dominant risk:** community *and* first-party extensions are largely IronPython 2; a naive default flip forces mass migration — mitigated by the per-extension declaration (§11.3).

**Maintenance surface grows before it shrinks** (IPy2 and/or IPy3 + CPython + the C# layer + two `site-packages` trees through the transition). **pythonnet becomes load-bearing**, with its own quirks (GIL × Dispatcher × Revit API) and the shared-interpreter isolation model (§5). **Package-version churn:** CPython pulls modern majors (Dynamo's jump: numpy 1.24→2.1, pandas 1.5→2.2) whose breaking changes can break user scripts independently of the engine swap.

**Security:** neither engine is a security boundary (both run arbitrary Python at full Revit-process privilege). Interpreter patch velocity favors CPython **only if pyRevit keeps the embedded `CPY3123` current by re-vendoring.** The largest real delta: a CPython default opens the full pip/PyPI surface (typosquatting, malicious packages) and **native C-extension execution** (unmanaged code, no CLR verification). pyRevit must track security fixes in *both* forks (IronPython, pythonnet) and refresh the embedded CPython.

### 11.1 Prior art — Dynamo's PythonNet3 migration

Dynamo performed essentially this migration (IronPython2 → IronPython3 → PythonNet3) and documented it. Relevance: **same destination, shorter route** (pyRevit skips the IPy3 stage — `CPY3123` already ships); the §12 checklist mirrors what Dynamo users actually hit; Dynamo reports WPF-in-Python stays painful even on pythonnet 3 ("takes more effort compared to IronPython"), arguing for the §9.2 C# hedge; and Dynamo already made the exact engine-fix move recommended in §10.1 — PythonNet3 1.1.0 shipped pythonnet #2055's method, the production precedent for cherry-picking it into `pyrevitlabs/pythonnet`.

---

## 12. The Revit-API Migration Checklist (per §10.1 bridge classes)

What changes for Revit-API code moving IronPython → pythonnet 3:
- **`out`/`ref` params → return tuples** (`res, fam = doc.LoadFamily(…)`) — done for the 2 lib sites; the 55-site collection class and 36-site `__namespace__` class are the backlog.
- **Collections need explicit generic construction** (`List[ElementId]([...])`), and are **views not copies** — wrap in `list()` where mutated.
- **Enum → int not implicit** (`int(cat)`); **`super().__init__()`** on .NET subclasses; **indexer properties** need `get_` accessors; **`with` on `IDisposable`** may need `.Dispose()`; **overload resolution differs**, LINQ needs explicit `System.Func[…]`; **interface impls need `__namespace__`**; **COM/GAC lost on .NET Core** (Revit 2025+) → Python-native libs.

In exchange: the full pip/PyPI C-extension ecosystem (numpy/pandas) unlocks.

---

## Appendix A — File & code reference index

- Engine lifecycle/selection: `dev/pyRevitLabs.PyRevit.Runtime/{ScriptEngineManager,ScriptEngines,IronPythonEngine,CPythonEngine,scriptruntime}.cs`.
- C# loader/bootstrap: `dev/pyRevitLoader/pyRevitAssemblyBuilder/**` (`AssemblyMaker/CommandTypeGenerator.cs`, `SessionManager/{Constants,SessionManagerService}.cs`), `dev/pyRevitLoader/Source/PyRevitLoaderApplication.cs`.
- Search paths / deps: `pyrevit/__init__.py` (`MAIN_LIB_DIR`, `MISC_LIB_DIR`), `extensions/{extensionmgr,genericcomps,extpackages,__init__}.py`, `loader/sessionmgr.py`, `release/cengines/CPY3123/python312._pth`, `pyRevitfile`.
- Coupling / shims: `pyrevit/compat.py`, `framework.py` (`add_reference_to_file`), `revit/db/{query,create}.py`, `revit/events.py`.
- Forms: `forms/__init__.py`, `_ipy.py` (`reactive`/`Reactive`/`TemplateListItem`/`SelectFromList`), `_cpy.py`, `utils.py`, `settings_window.py`, `output/__init__.py`, `dev/pyRevitLabs.PyRevit.Runtime/ScriptOutput.cs`.
- Test harness: `dev/scripts/check_py3_compat.py`, `pyrevit/unittests/test_py3_compat.py`.

## Appendix B — Glossary

`IPY2712PR` IronPython 2.7.12 (default) · `IPY342` IronPython 3.4.2 (opt-in) · `CPY3123` embedded CPython 3.12.3 via pythonnet (opt-in) · **Roslyn loader** C# command-type generation replacing `Reflection.Emit` · **pythonnet** CPython↔.NET bridge (v3.x in pyRevit) · **PythonNet3** Dynamo's name for CPython 3.x + pythonnet 3.x (= `CPY3123`) · **MVVM** Model-View-ViewModel · **BAML** compiled XAML.

## Appendix C — Related PRs

- **#3438** — in-process C# bootstrap; removes the IronPython bootstrap; requires Revit 2021+.
- **Roslyn loader** (`pyRevitAssemblyBuilder`) — landed, opt-in via `user_config.new_loader`.
- **`fix/improve-python3-support`** (draft) — language layer + CLR-loading & out-param bridge classes + the py3-compat checker/suite (§10).
