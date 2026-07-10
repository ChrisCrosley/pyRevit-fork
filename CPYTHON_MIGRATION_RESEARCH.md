# pyRevit → CPython: Research & Analysis

> **Audience:** pyRevit core maintainers.
> **Status:** Research / decision-support. This is the **single source of truth** for the
> IronPython → CPython+pythonnet migration. It supersedes and consolidates the earlier
> `IRONPYTHON_TO_PYTHON3_ANALYSIS.md`, `DEPENDENCY_MANAGEMENT_ANALYSIS.md`, and the
> `Refactor pyrevit.forms…` note. The tactical companion is
> **`CPYTHON_MIGRATION_PLAN.md`** — read this for *why*, that for *what to do next*.
> **Scope note:** File/line references reflect the state of the
> `claude/ironpython-cpython-transition` branch; treat them as pointers, not guarantees.

---

## 1. Executive Summary

pyRevit defaults to IronPython 2.7.12 because IronPython *is* a .NET language — a Python
object is a CLR object, so scripts get zero-marshaling access to the Revit .NET API, can
subclass .NET types, and can data-bind WPF to Python objects. That property is also why
IronPython has been hard to leave. This document maps how much of pyRevit still genuinely
depends on IronPython and what it takes to make **CPython 3.12 + pythonnet 3 (`CPY3123`)** the
default engine without losing functionality.

**Findings that drive everything:**

1. **IronPython is being demoted, not deleted.** The C# Roslyn loader and the in-process C#
   bootstrap (PR #3438) move session startup, extension parsing, assembly generation, and UI
   construction off IronPython. IronPython becomes one opt-in script engine among several.
2. **The syntax/language port is essentially done** (draft PR `fix/improve-python3-support`),
   along with the test harness that guards it and the first 2 of ~8 pythonnet "bridge" classes.
3. **The engine/session model changes structurally.** IronPython caches *one engine per
   extension* (isolation by construction). CPython runs *one process-global interpreter* with
   one shared, never-evicted `sys.modules`. This reshapes both **dependency management** (§6)
   and **module-name isolation** (§5).
4. **`forms/` is the long pole, and its hard core is WPF data-binding to Python objects** — not
   the XAML loader. This is the deepest technical risk and is prototype-gated (§8–§9).

**Destination:** `CPY3123` as the **default** engine, IronPython kept as an **opt-in legacy
engine** until the ecosystem moves, then sunset. The flip is a *designed mechanism*
(per-extension engine declaration, §11.3), not a config change.

---

## 2. Runtime Vocabulary (disambiguated)

"Python 3" is ambiguous — it can mean two different runtimes. This vocabulary is used
throughout.

| Name | What it *actually* is | Language level | .NET model | C-ext (numpy) | In pyRevit |
| --- | --- | --- | --- | --- | --- |
| **CPython** | The reference interpreter (in C). "Normal" Python. | 3.12 | none by itself | ✓ | embedded as `CPY3123` |
| **IronPython 2.x** | Python reimplemented in C#/.NET; Python objects **are** CLR objects | 2.7 (EOL) | native (is .NET) | ✗ | default `IPY2712PR` |
| **IronPython 3.x** | Same reimplementation, community-revived | ~3.4 | native (is .NET) | ✗ | opt-in `IPY342` |
| **pythonnet** | **Not an interpreter** — a *bridge* letting CPython call .NET | follows CPython | bridge/marshaling | ✓ | the layer over `CPY3123` |
| **PythonNet3** (Dynamo's term) | CPython 3.x + pythonnet 3.x together | current | bridge (v3) | ✓ | = pyRevit's `CPY3123` |

Traps: (a) `#! python3` selects **CPython**, never IronPython 3 — the `ScriptEngineType` enum
has no IronPython3 member. (b) pythonnet is a bridge, not a Python. (c) pythonnet **2.x** (weak
interop) vs **3.x** (strong interop — out-params as tuples, LINQ, better overloads); pyRevit
runs **3.x**, so most "pythonnet can't do X" lore describes the 2.x era.

The two axes that matter: **(a) .NET-native (IronPython) vs bridged (CPython+pythonnet)** —
decides WPF/marshaling behavior; **(b) can it run C-extensions?** — only CPython can, which is
the entire reason to move.

---

## 3. The Three Engines & Selection

| Engine key | Kernel | Role |
| --- | --- | --- |
| `IPY2712PR` | IronPython 2.7.12 (pyRevit fork) | Default |
| `IPY342` | IronPython 3.4.2 (pyRevit fork) | Opt-in, install-level |
| `CPY3123` | CPython 3.12.3 (embedded) via pythonnet | Opt-in, per-script (`#! python3`) |

Engine type is resolved per script (`scriptruntime.cs`): explicit `"type"` in engine configs,
else **shebang detection** (`python3`/`cpython` → CPython; anything else → IronPython).
Consequences: CPython vs IronPython is **per-button** (they coexist in-process); IPy2 vs IPy3 is
**global** (`#if IPY342`, rewrites the `.addin` manifest — the two IronPythons cannot run
simultaneously). pyRevit controls the loader, wrapper library, **and all engines** (IronPython
2, IronPython 3, and pythonnet are pyRevit forks under `pyrevitlabs/*`) — the only external,
unchangeable component is Autodesk's engine-agnostic Revit API.

---

## 4. Migration Already Underway

- **Legacy IronPython bootstrap:** Revit reads `.addin` → `pyRevitLoader.dll` → starts an
  IronPython engine → `pyrevitloader.py` → `sessionmgr.load_session()` builds the UI.
- **C# Roslyn loader (`pyRevitAssemblyBuilder`, landed, opt-in):** reimplements extension
  parsing, command-type generation (Roslyn source compilation instead of IronPython
  `Reflection.Emit`), UI construction, and session management in native .NET. Gated by
  `user_config.new_loader`.
- **In-process C# bootstrap (PR #3438, in-flight):** removes the IronPython bootstrap entirely;
  the C# `IExternalApplication` calls the C# session manager directly; Python startup splits
  into `session_preload.py`/`session_postload.py`. Drops Revit < 2021. This is the decisive step
  that turns "IronPython is the foundation" into "IronPython is one engine among several."

---

## 5. Engine Lifecycle & Isolation — the model changes structurally

Verified in `dev/pyRevitLabs.PyRevit.Runtime/` (`ScriptEngineManager.cs`, `ScriptEngines.cs`,
`IronPythonEngine.cs`, `CPythonEngine.cs`):

**IronPython:** engines are cached in a dictionary keyed by
`SessionUUID : EngineType : CommandExtension` — **all commands in one extension share one cached
engine; each extension gets its own.** `clean: true` forces a fresh engine per run;
`persistent: true` lets globals survive; the default scrubs scope references after each run but
keeps the engine (and its imported modules) warm. Search paths are set *wholesale* per engine.

**CPython:** there is **one process-global interpreter** — `PythonEngine.Initialize()` runs
once (parameterless), and pythonnet has **no sub-interpreter support**. Every execution gets a
fresh disposable scope (`Py.CreateScope` → exec → `Dispose`). `clean`/`persistent` have no
CPython meaning. "Refresh engine" is a full `PythonEngine.Shutdown()` — it resets state for
*every* CPython command at once.

| | IronPython (default) | IronPython (`clean`) | CPython `#! python3` |
| --- | --- | --- | --- |
| Script globals across runs | scrubbed unless `persistent` | gone (new engine) | always gone (scope disposed) |
| `sys.modules` | cached per extension's engine | fresh | cached **process-wide**, shared by all commands in all extensions |
| `sys.path` | per extension's engine | fresh | rebuilt per run from a baseline + current bundle paths |
| Isolation boundary | **extension** | command execution | **none** (one interpreter) |

**Consequences of one shared interpreter:**

- **`sys.modules` is shared and never evicted.** Extensions ship `lib/` dirs with generic
  module names (`utils.py`, `config.py`). Per-extension IronPython engines make that safe by
  construction; under one shared interpreter, **first import wins** and later extensions
  silently receive the first extension's module. Order-dependent "works unless that other button
  ran first" bugs.
  - **Worked example:** `A.extension/lib/get_door.py` and a *different*
    `B.extension/lib/get_door.py`, both `import get_door`. IronPython: isolated. CPython:
    whichever runs first caches `sys.modules['get_door']`; the other silently gets it —
    `AttributeError`, or silently-wrong results if both define the name. Its own `lib/` on the
    rebuilt `sys.path` is never consulted because `import` checks `sys.modules` first.
- **`sys.path` is rebuilt per run** (`CPythonEngine.SetupSearchPaths`): baseline snapshot →
  `PYTHONPATH` → the command's `SearchPaths`. A known ordering bug: the baseline is snapshotted
  per engine-wrapper *on first use* from the *current* `sys.path`, so if extension B's first
  CPython run happens after A ran, A's bundle paths bake into B's baseline. **Fix:** snapshot the
  pristine baseline once, globally, immediately after `PythonEngine.Initialize()`.
- **`clean`/`persistent` bundles** (smartbutton state) have **no CPython equivalent**.

**Mitigations (prerequisites for the flip):** (1) fix the baseline snapshot; (2) a per-run
**module-eviction policy** — after each execution, drop `sys.modules` entries whose `__file__`
lives under an extension directory, keeping stdlib/pip modules warm; (3) recommend namespaced
lib packages (`import myext_lib.get_door`). **But see §6.5 — the eviction policy must never touch
C-extension packages.**

---

## 6. Dependency Management

The migration changes how pyRevit packages its **own** dependencies *and* how extensions
(including third-party) manage theirs. Two layers are usually conflated; separate them.

### 6.1 The model today (verified in code)

- **First-party bundle code — per-component `lib/`.** Each component may carry a `lib/`
  (`COMP_LIBRARY_DIR_NAME='lib'`, `extensions/__init__.py`); registered in `genericcomps.py`,
  inherited down the bundle tree, added to `sys.path` **first** (highest precedence).
  Whole-extension shared libs use the `.lib` extension type.
- **Third-party — one shared vendored tree.** `MISC_LIB_DIR = HOME_DIR/site-packages`
  (`pyrevit/__init__.py`), appended **last** to every extension/script path
  (`extensionmgr.py`, `sessionmgr.py`; C# path `Constants.SITE_PACKAGES_DIR` in
  `CommandTypeGenerator.cs`). All extensions share the one tree.
- **No per-extension third-party mechanism exists.** `extension.json`'s `dependencies` means
  *other pyRevit extensions*, not pip packages. No `requirements.txt`, no pip step.
- **The embedded CPython ships no package machinery.** `release/cengines/CPY3123/` is a standard
  CPython **embeddable** distro: `python312.dll`, stdlib as `python312.zip`, the `.pyd`
  C-extensions — **no pip, no `Lib/`, no `site-packages/`**. Its `python312._pth` starts the
  interpreter *isolated* and leaves `import site` commented out, so normal `site-packages`
  discovery never runs; pyRevit re-adds its own folders manually. **"How does a user get numpy
  into pyRevit's Python" has no built-in answer today.**

### 6.2 The question: three extensions import numpy

Under one process-global interpreter with one shared, never-evicted `sys.modules`:

- **Same version from the shared tree →** fine, and *optimal*: import once, cached process-wide,
  everyone shares the warm module. The behavior to design *toward*.
- **Different versions, each bundled per-extension `lib/` → silent first-import-wins.** `import`
  consults `sys.modules` before `sys.path`; whichever extension runs first caches its numpy
  process-wide; the next `import numpy` returns the *first* version, its own `lib/` never
  consulted. Order-dependent.
- **C-extensions physically cannot coexist in two versions in one process.** numpy's dtype
  registry, `ndarray` type identity, and C-API are **process-global** C state, not namespaced by
  `sys.modules`. Hard CPython limit. IronPython dodged it by never running C-extensions *and*
  per-extension engines; CPython gives up both.

### 6.3 Can IronPython's "one environment per extension" be recreated? — No

- **In-process multi-interpreter is blocked:** pythonnet has one process-global interpreter, no
  sub-interpreters.
- **Faking it via per-extension `sys.modules`/`sys.path` swap** isolates *pure-Python code* but
  breaks on C-extensions (shared/corrupt C state or a forbidden second init), breaks
  cross-extension object flow (`isinstance`, C-API), and discards the import-once warm-cache win.
- **Sub-interpreters (per-interpreter GIL, 3.12/3.13):** not viable — pythonnet + many
  C-extensions don't support them.

### 6.4 Recommended model

| What you want isolated | Mechanism under CPython |
| --- | --- |
| First-party extension **code** (`utils.py`, `get_door.py` collisions) | §5 eviction (refined per §6.5) + namespaced `lib` packages. Cheap, safe; restores IronPython-like isolation for *code*. |
| Third-party **packages** (numpy, pandas) | **Do not isolate.** One curated, version-pinned shared environment — one numpy for everyone. |
| The rare genuine version conflict | **Out-of-process worker** (subprocess venv) — the only real per-extension third-party isolation, at the cost of in-process Revit API access; opt-in for compute-heavy/API-light tools. |

Concrete pieces:
1. **Single curated shared environment (platform-SDK model)** — pyRevit owns one pinned set,
   ABI-matched to CPython 3.12 `win_amd64`, on a security re-vendoring cadence.
2. **Managed writable user-site + a `pyrevit` CLI pip command** (`pyrevit env pip install …`) —
   solves the "how do I get numpy" UX cost. Still one shared tree, one version per package.
   Requires wiring pip into the embeddable distro (absent today).
3. **Declaration + conflict-check, not isolation** — extensions *declare* pip requirements that
   resolve against the shared env and **fail loud** on incompatible demands (vs today's silent
   first-wins). Extends the `dependencies` concept to pip.
4. **Per-extension bundling documented as an unsupported footgun** — works in single-extension
   dev, breaks under multi-extension production; C-extensions can't be isolated at all.
5. **Per-engine `site-packages` split** — a modern Py3 tree for CPython and a *frozen* Py2 tree
   for the legacy IronPython engine (the Py2 backports `six`, `pathlib2`, `scandir`,
   `unicodecsv`, … live only in the frozen tree, deleted wholesale when legacy sunsets).

### 6.5 The eviction-policy hazard (a latent bug in the §5 mitigation)

The §5 module-eviction rule ("evict `sys.modules` entries under an extension dir") correctly
fixes first-party name collisions (§5 `get_door` example) — safe because that code is
pure-Python and re-importable. It is **actively unsafe** the moment it touches a C-extension:
numpy/pandas run C init exactly once; evicting and re-importing corrupts/crashes them (numpy
refuses re-init). If an extension bundles numpy under its own `lib/`, the naive rule targets it.
**Required refinement:** evict pure-Python *extension-namespace code*; **never** evict
site-packages/pip/C-extension modules — maintain an explicit extension-namespace allowlist to
evict and keep everything else warm. **Evict code; never evict packages.**

---

## 7. Quantifying the IronPython Coupling

**`site-packages/` (vendored, 33 packages):** almost entirely pure-Python (only `sqlalchemy`
carries optional C-ext artifacts with a pure-Python fallback). The **Python-2 backports** (`six`,
`enum`, `pathlib2`, `scandir`, `importlib_resources`, `unicodecsv`) are the telltale of an
IronPython-2.7 target and are the frozen-tree candidates (§6.4.5).

**`pyrevitlib/` (first-party):** `pyrevit` (~200 files, mixed), `rpw` (frozen legacy, §7.1),
`rpws`/`rjm`/`rsparam` (pure-Python, compatible). Within `pyrevit`: `coreutils` (syntax + 2 CLR
shim sites), `revit` (bounded §12 idiom port), `interop` (9 CLR shim sites), `routes`/`extensions`
(compatible), `runtime`/`loader` (superseded by the C# loader), `forms` (**major work**, §8),
plus `framework.py` (*the* CLR-shim hub, 30 `AddReference` calls) and `compat.py` (already
`PY2`/`PY3`/`IRONPY` branched).

**Syntax portability — cleared by the draft PR (§10).** The residual Py2-only idioms (`iteritems`,
bare `unicode`, `__nonzero__`, heterogeneous sorts, `ifilterfalse`, Py3 view/iterator escapes)
are fixed and guarded by a CI checker.

**CLR interop — engine-agnostic vs IronPython-only.** `import clr`, `clr.AddReference`, `System.*`
work under **both** engines. The one widely-used IronPython-*only* API is
`clr.AddReferenceToFileAndPath` — now shimmed by the PR's `framework.add_reference_to_file`
(§10), removing it from all call sites outside `framework.py`.

**`revit/` marshaling hotspots** — a bounded, enumerable set: ~18 event hookups, generic-collection
constructions, 2 `out`/`ref` sites (`query.py`, `create.py`, ported by the PR). Already partially
dual-targeted (`query.py` branches on `PY3`; `events.py` gates on `IRONPY`).

**Shipped `extensions/`** (the parity corpus): 441 first-party scripts; **167 import `forms`**, 35
author custom WPF, and **zero end-user tools carry `#! python3`** — they break first under any
default flip and double as the parity test corpus.

### 7.1 `rpw` and `markdown` — vendored-library policy

- **`pyrevitlib/rpw`** (revitpythonwrapper 1.7.4, unmaintained): **engine-locked legacy.** Its
  `rpw.ui.forms` subclasses WPF and loads `IronPython.Wpf` — cannot run on CPython. Supported
  only under the opt-in legacy IronPython engine; `pyrevit.forms` is the documented replacement.
  Core's one dependence (`revit/db/pickling.py: from rpw import doc`) is trivially replaceable.
- **`pyrevit.coreutils.markdown`** (python-markdown 2.6.8): **deprecated, portable orphan.** Pure
  Python (no `clr`/WPF), so engine-agnostic, but orphaned when the output window moved rendering
  to C# (`ScriptOutput.cs`) — zero first-party runtime consumers. The PR keeps its syntax fixes
  but stops investing (removed from the checker/suite). Known crash: markdown conversion
  overflows the stack under IPY342 in Revit (fat DLR frames + recursion-heavy parser). Unbundling
  candidate; `#! python3` scripts should use pip `markdown`.

Both are deletion candidates when the legacy engine sunsets.

### 7.2 Dependency modernization — swap, drop, or bump

Beyond compatibility, the CPython move opens deliberate modernization. Consumer counts below
are first-party `.py` (telemetry server is Go, so vendored packages it doesn't use are dead
weight). Correction to the generic §12 checklist: **pyRevit has no Excel COM** — `interop/xl.py`
is pure-Python `xlrd`/`xlsxwriter`, and there is no `Microsoft.Office.Interop.Excel`/`GetActiveObject`
in first-party code, so "Excel COM → openpyxl" does **not** apply; the real Excel issue is xlrd's
`.xlsx` deprecation below.

**The headline: CPython turns the worst `interop` dependency — .NET/native assemblies missing on
.NET 8 (Revit 2025+) — into clean pip installs.** Today `interop.rhino`/`dxf`/`ifc` load managed
(and, for Rhino, **native**) assemblies that ship only in the netfx lib set, so they can't import
on Revit 2025+ on *any* engine (§10.1 known issue). This is a fork in strategy: **build netcore
copies** vs **retire the .NET wrapper for a pip-native Python library**.

| Today | Modern alternative | Used by | Fit / cost |
| --- | --- | --- | --- |
| `xlrd`+`xlsxwriter` (`interop/xl.py`, file I/O) | **openpyxl** | `xl.py` (71 lines) | **Clean swap.** Also fixes xlrd 2.0 dropping `.xlsx`. |
| `IxMilia.Dxf` (.NET, missing on .NET 8) via `interop/dxf.py` | **ezdxf** (pip) | thin 10-line loader; consumers use `IxMilia` API | **Strategic swap** — becomes `pip install ezdxf`; rewrite the DXF-building consumers (small surface). |
| `Rhino3dmIO` (.NET + **native** `rhino3dmio_native.dll`, missing on .NET 8) via `interop/rhino.py` | **rhino3dm** (pip, McNeel) | thin 10-line loader; 1 DevTools test | **Strategic swap** — kills native-binary + netcore + opt-in-risk at once; rewrite consumers. |
| `Ifc.Net` via `interop/ifc.py` | ~~ifcopenshell~~ — **not a swap** | `ifc.py` (318 lines) + 1 dev example | **Keep or netcore-build.** `ifc.py` uses `Ifc4` schema types to configure Revit's *native* IFC exporter (parses Revit's JSON, builds `IFCExportOptions`); ifcopenshell is a standalone authoring toolkit, a different job. This one still needs a netcore assembly, not a pip lib. |
| Autodesk Desktop Connector (`interop/adc.py`) | none (proprietary) | ADC integration | Stays a managed-assembly load. |

**site-packages modernization:**

| Class | Packages | Action |
| --- | --- | --- |
| Py2 backports → stdlib | `enum`, `pathlib.py`+`pathlib2`, `scandir`, `unicodecsv`, `importlib_resources`, `pytz`→`zoneinfo` | **Drop from the Py3 tree** (stdlib in 3.12); keep only in the frozen legacy tree. |
| Py2-only | `six.py`; `pyevent.py` | Drop `six`; `pyevent.py` is IronPython-only (§8.3.1) → frozen tree, replaced by the forms reactive backend. |
| Orphaned (0 first-party consumers) | `slackclient`, `bson`, `sqlalchemy`, `munch` | **Unbundle candidates** — dead weight in the Python tree. If Slack is revived, `slackclient` is the *deprecated* SDK → `slack_sdk`. |
| Unmaintained but used | `docopt` (2 consumers) | → `argparse` if touched anyway; low priority. |
| Keep (maintained, pure-Python) | `requests`+stack, `werkzeug`, `websocket`, `xlsxwriter`, `pyparsing`, `natsort`, `sortedcontainers`, `filelock`, … | Re-vendor current majors (or pip) into the curated Py3 env; watch major-version breaking changes (§11). |

---

## 8. The `forms/` Layer — the Long Pole

### 8.1 Current state (verified)

`forms/__init__.py` (20-line facade) dispatches on `compat.IRONPY` to `_ipy.py` (**4,105 lines**,
the real WPF implementation) or `_cpy.py` (**103 lines of stubs** + a module-level `__getattr__`
that raises `PyRevitCPythonNotSupported` for anything not stubbed). Also in scope:
`settings_window.py` (705 lines, subclasses `WPFWindow`, **not engine-gated**), `utils.py` (63
lines, imports the IronPython-only `wpf` module **at module top → hard-fails on import under
CPython**), `toaster.py` (portable subprocess), and **21 `.xaml` files** authored against the
LoadComponent-into-self model. So the port scope is **~4,900 Python lines + 21 XAML files**.

Under `#! python3` today, importing `pyrevit.forms` succeeds (stubs load no WPF) but **any actual
use raises**.

### 8.2 Why a pythonnet port is hard — the runtime-model mismatch

IronPython *is* .NET (a Python object is a CLR object, visible to WPF's reflection/binding
engine). pythonnet is a *bridge* (Python objects are proxies, not natively visible to .NET
reflection). Almost everything hard about `forms` flows from that asymmetry.

### 8.3 The blockers, ranked — and the crux, validated in code

The `Refactor pyrevit.forms…` note argued the engine-specific surface is *tiny* — essentially
only `wpf.LoadComponent` and `pyevent.make_event()` — and everything else "runs identically under
pythonnet." **The first half is right; the second half is not, and the code proves it.**

1. **WPF data-binding to Python objects — THE deepest blocker, and not confined to exotic MVVM.**
   The workhorse dialogs bind XAML directly to Python objects:
   - `SelectFromList` sets `self.list_lb.ItemsSource = ObservableCollection[TemplateListItem](…)`
     and `SelectFromList.xaml` binds `{Binding name}`, `{Binding checked}`, `{Binding checkable}`.
   - Parameter pickers bind `{Binding displayvalue}`, `{Binding istype}`, `{Binding isbuiltin}`;
     the color swatch binds `{Binding hex_color}`; the image list binds `{Binding item}`.
   - `TemplateListItem(Reactive)` → `Reactive(ComponentModel.INotifyPropertyChanged)` via
     `pyevent.make_event()`; `@reactive` properties raise `OnPropertyChanged`.
   Under IronPython these resolve because the item *is* a CLR object with reflectable properties.
   Under pythonnet, WPF binding resolves via `TypeDescriptor`/`PropertyDescriptor`, which
   generally **cannot see Python-defined properties**. Fabricating the `INotifyPropertyChanged`
   event is **necessary but not sufficient** — the event says "something changed," but binding
   still can't find `name` on a Python object. So `SelectFromList` and the parameter/color/image
   selectors are **not free**; they sit on the same unproven crux as full MVVM. This is
   asserted-not-proven on both sides and is the reason for the §9.3 prototype gate.
2. **`wpf.LoadComponent(self, …)`** — no pythonnet `wpf` module. The CPython loader must parse
   with `XamlReader`/`Application.LoadComponent`, copy the tree onto `target`, walk the
   name-scope to wire `x:Name` controls as attributes, and connect XAML-named handlers to Python
   methods — three things `XamlReader.Load` doesn't do together. Largest piece of *new* code;
   viable in pure Python (this is the §7.6-style "author XAML in pure Python" path).
3. **`.NET subclassing lifecycle** — `class WPFWindow(_WPFMixin, Window)` + "construct empty,
   then `LoadComponent(self)`" isn't how pythonnet subclassing works; needs `super().__init__()`
   and, for interfaces, `__namespace__` (ties to the §10 bridge backlog).
4. **Modeless dialogs that call the Revit API** (`ProgressBar`, `WarningBar`, dockable panels)
   must marshal back via `ExternalEvent`/`IExternalEventHandler` — needs `__namespace__` + GIL
   care. **Depends on the §10 `__namespace__` resolution.**
5. **Threading / GIL × WPF Dispatcher × Revit single-threaded context** — a real source of
   hangs/reentrancy to re-validate, not just port.

---

## 9. The Chosen UI Direction (reconciled)

The two prior documents pointed in different directions; both are partly right, and the shared
structure reconciles them.

### 9.1 Adopt the shared-package structure (from the refactor note)

Split `pyrevit.forms` along **two axes**: a horizontal *engine-core vs shared-feature* boundary,
and a vertical *feature-area* boundary within the shared code. Only the engine core is duplicated
per engine; feature modules are written once. Target layout: `__init__.py` (facade/namespace
assembler) + `_backend.py` + `backends/{_ipy,_cpy}.py` (the tiny engine core:
`load_xaml_component`, `make_property_changed_event`, assembly refs) + feature modules (`base`,
`reactive`, `dialogs`, `promptbars`, `selection`, `alerts`, `checks`, `pickers`, `notify`,
`dockable`) + existing `utils`/`toaster`/`settings_window`/`*.xaml`. This kills the
`_ipy`/`_cpy` monolith divergence and is the right skeleton regardless of the binding bet.

### 9.2 But keep the original analysis's C# hedge for the binding tier

The original analysis's *chosen* direction was a **C#-assisted UI API** (`forms.DataGrid` over a
`DataTable`; `forms.BindableModel` — a C# `INotifyPropertyChanged`/`DynamicObject` core WPF
binding *can* see), precisely because binding-to-Python-objects (§8.3.1) may be unsolvable in pure
Python under pythonnet. The `ScriptOutput`/`PyRevitOutputWindow` console already proves the
pattern: a C#-hosted WPF window driven from Python via a thin `__getattr__` wrapper, working on
every engine.

### 9.3 Resolution — prototype-gated, structure fits both (the decision)

- **Native/leaf tier** (alert, pickers, `ask_for_color`, toast, `check_*`, orchestration): plain
  `.NET`/Revit-API calls, port directly under pythonnet with only §12 idiom cleanups. No C#. Both
  documents agree here.
- **XAML-into-self base** (`WPFWindow`/`WPFPanel`): the pure-Python `load_xaml_component`
  (XamlReader + name-scope wiring) is viable. Adopt.
- **Data-bound / reactive tier** (`SelectFromList`, selectors, MVVM custom dialogs): **gate on a
  binding prototype** (§9.4). Design the `reactive` backend so it resolves to **either** a
  pure-Python `INotifyPropertyChanged` shim (if the prototype proves binding to Python objects
  works on pyRevit's pythonnet 3 fork) **or** a C# `BindableModel`/`DataTable` adapter (if it
  doesn't). The shared-package structure makes this swap invisible to the feature modules.

### 9.4 The prototype gate (must pass before committing the binding backend)

A sharply-scoped spike must demonstrate, **on pyRevit's pythonnet fork**:
1. **WPF binding fidelity to a Python (or `DynamicObject`) source** — not just `{Binding name}`
   happy path, but `DataTemplate` resolution, `DataTrigger`s (the parameter pickers use them),
   validation, and `ICollectionView` sort/group. `SelectFromList`'s real XAML is the fixture.
2. **The chatty-callback path under the GIL** — a **modeless `ProgressBar` driving Revit API
   calls through `ExternalEvent`** (the §8.3.4 combination), measuring GIL × Dispatcher ×
   Revit-context.

**Fallback if it fails:** `DataTable`-backed binding + imperative control access covers the
native/leaf and list tiers; only reactive-MVVM custom dialogs are lost, and the approach degrades
to "C# host plumbing + `DataGrid`" rather than collapsing.

---

## 10. What the Draft PR Establishes (`fix/improve-python3-support`)

The first step of the journey. It splits the work along one clean axis and clears the first part:

- **Language layer (breaks on *any* Py3 engine, IPy3 included):** `iteritems`, bare `unicode`,
  `__nonzero__`/`__bool__`, heterogeneous sorts, `ifilterfalse`, Py3 view/iterator escapes —
  **cleared comprehensively**, including the language residuals *inside* `forms/_ipy.py` (the
  `__bool__` aliases and the `list()`-wrapped views in `search_matches`/grouped select).
- **Bridge layer (pythonnet-only): 2 of ~8 classes cleared** — IronPython-only CLR loading
  (Stage 2 → new public `framework.add_reference_to_file`, the **exact shim the forms backend
  will reuse**) and `clr.Reference` out-params (Stage 3 → `query.intersect_curves`,
  `create.load_family`, with the `__namespace__` + tuple-return conventions).
- **Test harness (the reusable safety net):** an AST checker (`dev/scripts/check_py3_compat.py`,
  CI-gated at zero) and an in-Revit `test_py3_compat` suite run by two DevTools buttons as a
  **per-engine parity dashboard** (IPY2 / IPY342 / CPython). The forms port and the bridge
  backlog should plug into *this*.
- **Honest scope:** does **not** touch the forms WPF port or engine selection; surveys the
  remaining bridge classes as measured future work (§10.1).

### 10.1 The measured bridge backlog

| Bridge idiom | Sites | Notes |
| --- | --- | --- |
| **Generic collection construction** (`List[T](pylist)`) | **55** | 46 high-confidence / 7 review / 2 already-.NET; ~16 lib, ~39 ext. Largest class; natural first slice. |
| **Interface impls needing `__namespace__`** | **36** | only 3 carry it; `ISelectionFilter` (~15), `IExternalEventHandler` (~8), `IDuplicateTypeNamesHandler` (~6), … Applied inconsistently even within one file (`events.py`). **Two open questions** (from `FamilyLoaderOptionsHandler.__namespace__` in Stage 3): (a) is it needed per-interface? (b) does pinning it collide on reload — the CLR type map survives a `sys.modules` clear? **These are the same shared-process-state class as §5/§6, and they gate the forms modeless tier (§8.3.4).** |
| **Collections returned as views** (wrap in `list()`) | unsized | not cleanly AST-detectable. |
| **`.Item[…]` / bracket-index on `IEnumerable`** | 2+ | plain `[i]` not statically distinguishable from Python indexing. |
| **`with` on `IDisposable`** | 4 | modern pythonnet may already cover; verify in-Revit. |
| **Overload resolution / LINQ `System.Func[…]`** | 1+ | mostly unsurveyed. |
| **Enum→int not implicit; `super().__init__()`; COM/GAC lost on .NET Core** | unsized | `super()` largely in `forms`; COM is a library-rewrite concern (Excel COM → `openpyxl`). |

These are **migration-readiness totals, not active-bug counts** — most extension sites are
IronPython-only today, so the bridge bites only when they run under CPython.

---

## 11. Decision Factors, Risks, Security

**Ecosystem unlock** is real but **already works today** (opt-in via `#! python3`); the plan
makes it the default and removes the gate. **Backward-compat break is the dominant risk:**
community *and* first-party extensions are largely IronPython 2; a naive default flip forces mass
migration — mitigated by the per-extension declaration (§11.3).

**Maintenance surface grows before it shrinks** (IPy2 and/or IPy3 + CPython + the C# layer + two
`site-packages` trees through the transition). **pythonnet becomes load-bearing** with its own
quirks (GIL × Dispatcher × Revit API) and the shared-interpreter isolation model (§5).
**Package-version churn** — CPython pulls modern majors (Dynamo's jump: numpy 1.24→2.1, pandas
1.5→2.2), whose breaking changes can break user scripts independently of the engine swap.

**Security:** neither engine is a security boundary (both run arbitrary Python at full
Revit-process privilege). Interpreter patch velocity favors CPython **only if pyRevit keeps the
embedded `CPY3123` current by re-vendoring.** The largest real delta: a CPython default opens the
full pip/PyPI surface (typosquatting, malicious packages) and **native C-extension execution**
(unmanaged code, no CLR verification). pyRevit must track security fixes in *both* forks
(IronPython, pythonnet) and refresh the embedded CPython.

### 11.1 Prior art — Dynamo's PythonNet3 migration

Dynamo performed essentially this migration (IronPython2 → IronPython3 → PythonNet3) and
documented it. Relevance: **same destination, shorter route** (pyRevit skips the IPy3 stage —
`CPY3123` already ships); the §12 checklist mirrors what Dynamo users actually hit; and Dynamo
reports WPF-in-Python stays painful even on pythonnet 3 ("namespace manipulation and property
declaration boilerplate"), which **argues for the §9.2 C# hedge**.

---

## 12. The Revit-API Migration Checklist (per §10.1 bridge classes)

What changes for Revit-API code moving IronPython → pythonnet 3:
- **`out`/`ref` params → return tuples** (`res, fam = doc.LoadFamily(…)`) — done for the 2 lib
  sites; 55-site collection class and 36-site `__namespace__` class are the backlog.
- **Collections need explicit generic construction** (`List[ElementId]([...])`), and are **views
  not copies** — wrap in `list()` where mutated.
- **Enum → int not implicit** (`int(cat)`); **`super().__init__()`** on .NET subclasses;
  **indexer properties** need `get_` accessors; **`with` on `IDisposable`** may need `.Dispose()`;
  **overload resolution differs**, LINQ needs explicit `System.Func[…]`; **interface impls need
  `__namespace__`**; **COM/GAC lost on .NET Core** (Revit 2025+) → Python-native libs.

In exchange: the full pip/PyPI C-extension ecosystem (numpy/pandas) unlocks.

---

## Appendix A — File & code reference index

- Engine lifecycle/selection: `dev/pyRevitLabs.PyRevit.Runtime/{ScriptEngineManager,ScriptEngines,IronPythonEngine,CPythonEngine,scriptruntime}.cs`.
- C# loader/bootstrap: `dev/pyRevitLoader/pyRevitAssemblyBuilder/**`
  (`AssemblyMaker/CommandTypeGenerator.cs`, `SessionManager/{Constants,SessionManagerService}.cs`),
  `dev/pyRevitLoader/Source/PyRevitLoaderApplication.cs`.
- Search paths / deps: `pyrevit/__init__.py` (`MAIN_LIB_DIR`, `MISC_LIB_DIR`),
  `extensions/{extensionmgr,genericcomps,extpackages,__init__}.py`, `loader/sessionmgr.py`,
  `release/cengines/CPY3123/python312._pth`, `pyRevitfile`.
- Coupling / shims: `pyrevit/compat.py`, `framework.py` (`add_reference_to_file`),
  `revit/db/{query,create}.py`, `revit/events.py`.
- Forms: `forms/__init__.py`, `_ipy.py` (`reactive`/`Reactive`/`TemplateListItem`/`SelectFromList`),
  `_cpy.py`, `utils.py`, `settings_window.py`, `output/__init__.py`,
  `dev/pyRevitLabs.PyRevit.Runtime/ScriptOutput.cs`.
- Test harness: `dev/scripts/check_py3_compat.py`, `pyrevit/unittests/test_py3_compat.py`.

## Appendix B — Glossary

`IPY2712PR` IronPython 2.7.12 (default) · `IPY342` IronPython 3.4.2 (opt-in) · `CPY3123` embedded
CPython 3.12.3 via pythonnet (opt-in) · **Roslyn loader** C# command-type generation replacing
`Reflection.Emit` · **pythonnet** CPython↔.NET bridge (v3.x in pyRevit) · **PythonNet3** Dynamo's
name for CPython 3.x + pythonnet 3.x (= `CPY3123`) · **MVVM** Model-View-ViewModel · **BAML**
compiled XAML.

## Appendix C — Related PRs

- **#3438** — in-process C# bootstrap; removes the IronPython bootstrap; requires Revit 2021+.
- **Roslyn loader** (`pyRevitAssemblyBuilder`) — landed, opt-in via `user_config.new_loader`.
- **`fix/improve-python3-support`** (draft) — language layer + CLR-loading & out-param bridge
  classes + the py3-compat checker/suite (§10).
