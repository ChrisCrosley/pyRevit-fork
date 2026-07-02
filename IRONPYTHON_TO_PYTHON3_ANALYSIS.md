# pyRevit: IronPython Dependence & the Path to a Python 3 Default

> **Audience:** pyRevit core maintainers.
> **Status:** Analysis / decision-support document. Describes current-state findings and
> weighs migration options. No code changes are proposed by the document itself.
> **Scope note:** Line/file references reflect the state of the branch this was written
> against; treat them as pointers, not guarantees. PR references are point-in-time.

---

## 1. Executive Summary

### 1.1 The question in one paragraph
pyRevit defaults to IronPython 2.7.12 because IronPython *is* a .NET language: a Python
object is a CLR object, so scripts get native, zero-marshaling access to the Revit .NET API,
can subclass .NET types, and can data-bind WPF to Python objects. That same property is why
IronPython has been hard to leave. This document maps how much of pyRevit still genuinely
depends on IronPython, what has already been migrated to C#, and what it would take to make
CPython 3 a first-class (or default) engine without losing functionality.

### 1.2 Key findings
- **IronPython is being demoted, not deleted.** The C# Roslyn loader and the in-process C#
  bootstrap (PR #3438) move session startup, extension parsing, assembly generation, and UI
  construction off IronPython. IronPython becomes one opt-in script engine among several.
- **The syntax port is nearly done.** `pyrevitlib` has almost no Python-2-only syntax left;
  the residual idioms are concentrated in one vendored dependency.
- **The real coupling is .NET interop, most of which is engine-agnostic.** `import clr` /
  `System.*` work under both IronPython and pythonnet. The only widely-used IronPython-*only*
  API is `clr.AddReferenceToFileAndPath`, centralized in `framework.py`.
- **`forms/` is the long pole.** ~4,100 lines of IronPython-only WPF, currently stubbed off
  under CPython. It is the one place "run the same Python on a different engine" breaks down.
- **All engines are pyRevit-controlled forks.** IronPython 2, IronPython 3, and pythonnet are
  all submodules under `pyrevitlabs/*`. No layer of this depends on an outside maintainer
  except Autodesk's Revit API, which is engine-agnostic.

### 1.3 Recommendation at a glance
**Destination:** CPython 3.12 + pythonnet 3 — the "PythonNet3" equivalent, pyRevit's
`CPY3123` — as the **default** engine. Everything between here and there (IronPython 3, the
dual-engine period, the `compat` shims, the `clr.AddReferenceToFileAndPath` wrapper) is
**transitional scaffolding** to be removed once migration completes. The one durable exception
is an **opt-in legacy IronPython engine** kept for backward-compatibility with the existing
IronPython-2 extension ecosystem, then sunset when that ecosystem has moved.

Concretely: continue the C#-chrome / Python-scripting trajectory, and reimplement `forms` as a
**C# UI layer with a thin Python API** (Option B, §7) — the one piece that must be solved
before CPython can become the default. See §2.4 for the runtime vocabulary and §9 for options
and sequencing.

---

## 2. Background: Why pyRevit Depends on IronPython

### 2.1 IronPython *is* .NET — native CLR/Revit-API integration
The Revit API is a .NET API. IronPython compiles Python to CLR IL and runs in-process on the
same runtime as Revit, so scripts manipulate live Revit objects with no marshaling layer,
subclass .NET types directly, and let WPF's binding engine reflect over Python objects. This
is the core reason IronPython was chosen and the core reason it is hard to replace.

### 2.2 Historical and library-ecosystem inertia
pyRevit predates viable CPython-in-Revit support, so `pyrevitlib/` and the vendored
`site-packages/` were written for IronPython 2.7.12. That target is why Python-2 backports
(`six`, `enum`, `pathlib2`, `scandir`, ...) still appear in `site-packages/`.

### 2.3 The three engines today
| Engine key | Kernel | Role |
|---|---|---|
| `IPY2712PR` | IronPython 2.7.12 (pyRevit fork) | Default |
| `IPY342` | IronPython 3.4.2 (pyRevit fork) | Opt-in, install-level |
| `CPY3123` | CPython 3.12.3 (embedded) via pythonnet | Opt-in, per-script (`#! python3`) |

### 2.4 The runtimes, disambiguated
"Python 3" is ambiguous here — it can mean two completely different runtimes. This section
fixes the vocabulary used throughout the document.

| Name | What it *actually* is | Language level | .NET model | C-extensions (numpy) | In pyRevit |
|---|---|---|---|---|---|
| **CPython** | The standard/reference Python interpreter (in C). "Normal" Python. | current (3.12) | none by itself | ✓ | embedded as `CPY3123` |
| **IronPython 2.x** | Python *reimplemented in C#/.NET*; Python objects **are** CLR objects | Python 2.7 (EOL) | native (is .NET) | ✗ | default `IPY2712PR` (2.7.12) |
| **IronPython 3.x** | Same reimplementation, community-revived for Python 3 | ~Python 3.4 | native (is .NET) | ✗ | opt-in `IPY342` (3.4.2) |
| **pythonnet (Python.NET)** | **Not an interpreter** — a *bridge* letting CPython call .NET | (follows CPython) | bridge/marshaling | ✓ | the layer over `CPY3123` |
| **"PythonNet3"** (Dynamo's term) | Shorthand for **CPython 3.x + pythonnet 3.x** together | current | bridge (v3, improved) | ✓ | = pyRevit's `CPY3123` |

Three traps to avoid:
- **"Python 3" ≠ one thing.** IronPython 3 and CPython 3 are different runtimes. In pyRevit,
  `#! python3` selects **CPython**, never IronPython 3.
- **pythonnet is a bridge, not a Python.** "CPython via pythonnet" is the real runtime;
  "pythonnet" alone is just the interop layer.
- **pythonnet has two eras.** pythonnet **2.x** (≤ Python 3.8, weak interop) vs pythonnet
  **3.x** (Python 3.7–3.12+, strong interop). Dynamo brands the latter "PythonNet3."
  **pyRevit's `CPY3123` already runs on pythonnet 3.x** (it must, to host CPython 3.12), so
  pyRevit is already the "PythonNet3" equivalent — see §6.4.

The two axes that actually matter: **(a) .NET-native (IronPython) vs bridged (CPython+pythonnet)**
— decides WPF/marshaling behavior; and **(b) can it run C-extensions?** — only CPython can,
which is the entire reason to move.

**Framing — destination vs scaffolding.** The migration target is **PythonNet3** (`CPY3123`).
IronPython 3, the dual-engine period, and the `compat`/CLR shims are *transitional scaffolding*.
IronPython 3 in particular is **optional** scaffolding — valuable only if it accelerates the
Python 2→3 *syntax* migration while preserving native .NET integration; it is not a destination
(capped ~Python 3.4, no numpy, small-team fork) and can be skipped if IPy2 code is ported
straight to PythonNet3. The only potentially-durable IronPython presence is an opt-in legacy
compatibility engine for un-migrated extensions.

---

## 3. Current Architecture & the Migration Already Underway

### 3.1 The legacy IronPython loading sequence
Revit reads the `.addin` manifest → loads `pyRevitLoader.dll` → the loader starts an
IronPython engine and runs `pyrevitloader.py` → `sessionmgr.load_session()` builds the UI.
Historically, IronPython was the bootstrap that constructed everything.

### 3.2 The C# Roslyn loader (`pyRevitAssemblyBuilder`) — landed, opt-in
A C# project pair (`pyRevitAssemblyBuilder`, `pyRevitExtensionParser`) reimplements extension
parsing, command-type generation (via **Roslyn** source compilation instead of IronPython
`Reflection.Emit`), UI construction, hook registration, and session management in native .NET.
Gated by `user_config.new_loader`; falls back to the legacy Python path. The legacy Python
loader is logged as deprecated.

### 3.3 The in-process C# bootstrap (PR #3438) — in-flight
Removes the IronPython bootstrap entirely: the C# `IExternalApplication` calls the C# session
manager directly for startup and reload. Python startup logic is split into C#-invoked
`session_preload.py` / `session_postload.py` phases. Eliminates the `new_loader` toggle and
the legacy `uimaker.py` / `asmmaker.py`. Drops Revit < 2021. This is the decisive step that
turns "IronPython is the foundation" into "IronPython is one engine among several."

> As of writing, this branch is **pre-#3438**: `session_preload.py`/`session_postload.py` are
> absent, `uimaker.py`/`asmmaker.py` are present, and the IronPython bootstrap is still in place.

### 3.4 Engine selection & the `#! python3` shebang
Engine type is resolved per script (`scriptruntime.cs`):
1. Explicit `"type"` in engine configs (honored for CPython, or IronPython only when
   `type_explicit` is set).
2. Otherwise, **shebang detection**: a first line containing `python3` or `cpython`
   → `ScriptEngineType.CPython`; anything else → `ScriptEngineType.IronPython`.

Critical consequences:
- **`#! python3` → CPython, not IronPython 3.** There is no shebang that selects IronPython 3;
  the `ScriptEngineType` enum has no `IronPython3` member.
- **CPython vs IronPython is per-button** (they coexist in-process).
  **IPy2 vs IPy3 is global** — a build/config choice (`#if IPY342`) that rewrites the `.addin`
  manifest; the two IronPythons cannot run simultaneously.
- The startup-script helper `create_ipyengine_configs(...)` sets only `clean`/`full_frame`/
  `persistent` and **no** `"type"`, so extension startup scripts also resolve engine by
  shebang despite the misleading name.

---

## 4. Quantifying the IronPython Coupling

### 4.1 Inventory: `site-packages/` (vendored third-party)
~35 packages, almost entirely pure-Python (only `sqlalchemy` carries optional C-extension
artifacts, with a pure-Python fallback). Groups: HTTP/web (`requests`, `urllib3`, `werkzeug`,
`websocket`, `slackclient`), data/files (`xlrd`, `xlsxwriter`, `sqlalchemy`, `bson`,
`sorted*`), parsing/util (`pyparsing`, `docopt`, `pytz`), and **Python-2 backports** (`six`,
`enum`, `pathlib`/`pathlib2`, `scandir`, `importlib_resources`, `unicodecsv`). The backports
are the tell-tale of an IronPython-2.7 target.

### 4.2 Inventory: `pyrevitlib/` (first-party)
Five packages: `pyrevit` (~200 files; sub-packages `coreutils`, `revit`, `interop`, `routes`,
`runtime`, `loader`, `forms`, `output`, `telemetry`, ...), plus `rpw`, `rpws`, `rjm`,
`rsparam`.

### 4.3 Syntax portability — near-complete
Genuine Python-2-only syntax is nearly absent: no `print` statements, no `except X, e`, no
`has_key`, no `xrange`. Residuals: one `.iteritems()`, a few `basestring`, and a handful of
`unicode()` calls **almost entirely inside the vendored `coreutils/markdown/` package**.
`compat.py` already provides `PY2`/`PY3`/`IRONPY` branches — the tree was written dual-target
on purpose.

### 4.4 CLR interop: engine-agnostic vs IronPython-only
`import clr`, `clr.AddReference`, and `System.*` work under **both** IronPython and pythonnet —
these are .NET dependencies, not IronPython dependencies. The one widely-used **IronPython-only**
API is **`clr.AddReferenceToFileAndPath`** (~17 call sites across ~9 files), and CLR loading is
already funneled through `framework.py` (~30 `AddReference` calls in ~174 lines) — a single,
well-located place to shim for pythonnet.

### 4.5 The `revit/` API-wrapper marshaling hotspots
The `revit/` layer is mostly engine-agnostic .NET with a bounded, enumerable set of
marshaling-sensitive spots: ~18 `.NET` event hookups, generic-collection constructions, and
**2 `out`/`ref` sites** using `clr.Reference` (`revit/db/query.py`, `revit/db/create.py`).
It is already partially dual-targeted (`query.py` branches on `PY3`; `events.py` gates
`ExternalEvent.Create` behind `compat.IRONPY`).

---

## 5. The `forms/` Layer — the Long Pole

### 5.1 Current state
`forms/__init__.py` dispatches by engine: `_ipy.py` (**~4,105 lines**, the real WPF
implementation) under IronPython, `_cpy.py` (**~103 lines of stubs**) under CPython.

### 5.2 What `#! python3` users get today: forms wholesale unavailable
The CPython facade imports the stubs **and** installs a module-level `__getattr__` that raises
`PyRevitCPythonNotSupported` for any name not stubbed. So:
- The 13 explicitly-stubbed symbols (`WPFWindow`, `SelectFromList`, `CommandSwitchWindow`,
  `ProgressBar`, `ask_for_*`, `pick_file`, `pick_folder`, `show_balloon`, ...) raise when used.
- **Everything else** (`alert`, `toast`, all `select_*`, `check_*` validators, `WPFPanel`,
  dockable panels, ...) raises on access via `__getattr__`.

Importing `pyrevit.forms` succeeds (stubs load no WPF), but **any actual use fails**. Note the
gate is broader than strictly necessary: `alert` (a dialog), `pick_file` (a file dialog), and
the pure-logic `check_*` validators have no WPF-binding dependency and could work under
pythonnet, but are stubbed off anyway. Today's behavior is a blanket "forms off under CPython."

### 5.3 Why a pythonnet port is hard: the runtime-model mismatch
IronPython *is* .NET (a Python object is a CLR object, visible to WPF's reflection/binding
engine). pythonnet is a *bridge* (Python objects are proxies, not natively visible to .NET
reflection). Almost everything hard about `forms` flows from that asymmetry.

### 5.4 The specific blockers
- **`wpf.LoadComponent(self, ...)`** — IronPython's `wpf` module (`IronPython.Wpf.dll`) has no
  pythonnet equivalent. It parses XAML *into* the existing instance, auto-wires `x:Name`
  controls as attributes, and binds XAML-declared handlers to Python methods — three things
  pythonnet's `XamlReader.Load` (returns a new graph; requires `FindName`) cannot do together.
  The base class and every dialog subclass depend on this.
- **WPF data-binding to Python objects** — `{Binding SomeProp}` against Python objects works
  under IronPython, silently resolves to nothing under pythonnet. This is the deepest blocker.
- **`pyevent.make_event()`** — fabricates a .NET-compatible event for `INotifyPropertyChanged`
  using IronPython reflection magic; no pythonnet analog.
- **.NET subclassing lifecycle** — `class WPFWindow(_WPFMixin, Window)` plus "construct empty,
  then `LoadComponent(self)`" is not how pythonnet subclassing works.
- **Threading / GIL** — IronPython has no GIL; `forms` freely spins threads and marshals via
  the Dispatcher. Under CPython the GIL interacts with the WPF Dispatcher and Revit's
  single-threaded API context — a real source of hangs/reentrancy to re-validate.

---

## 6. The Two "Python 3" Paths Compared

### 6.1 IronPython 3 (`IPY342`)
Real, buildable, shipped as an opt-in engine (its own `IronPython.Wpf`, runtime built via
`#if IPY342`). Upstream is community-maintained (IronLanguages, not Microsoft), alive but slow
(3.4.0 in 2022, 3.4.1 mid-2024, 3.4.2 Dec 2024), pinned near the Python 3.4 language level
with some backports (f-strings). **Because it is still a .NET language, the `forms` blockers
and `revit/` marshaling issues largely evaporate** — migration is mostly a Python 2→3 syntax
bump. It does **not** provide the C-extension ecosystem (no numpy/pandas).

### 6.2 CPython 3.12 (`CPY3123`) via pythonnet
The Revit-API *interaction idioms change* (full migration checklist in §6.5, corroborated by
Dynamo's real-world migration):
- **`out`/`ref` params become return tuples** (`res, fam = doc.LoadFamily(...)`), replacing the
  IronPython `clr.Reference[T]()` pattern used in `query.py`/`create.py` today.
- **Collections need explicit generic construction** (`List[ElementId]([...])`), and are now
  **"views" not deep copies** — wrap in `list()` where mutation/Python-list methods are needed.
- **Enum → int is no longer implicit** — use `int(cat)` or the enum member.
- **`super().__init__()` is required** when subclassing .NET types.
- **Indexer properties** need the `get_` accessor (`elem.get_Space(phase)`), and `IEnumerable`
  cannot be bracket-indexed.
- **`with` on `IDisposable`**, **overload resolution**, and **interface implementation** differ.
Unlocks the full pip/PyPI ecosystem (already works today via `#! python3`).

### 6.3 Side-by-side trade-offs
| | IronPython 3 | CPython / pythonnet |
|---|---|---|
| Modern language level | ✗ (~3.4) | ✓ (3.12) |
| C-extension ecosystem (numpy) | ✗ never | ✓ |
| `forms` port cost | Low (syntax bump) | High (rewrite) |
| Revit-API code churn | Minimal | Real (idioms change) |
| Maintenance | pyRevit fork; small upstream | pyRevit fork + embedded CPython |

### 6.4 pythonnet version matters — pyRevit is already on the good one
pythonnet has two eras: **2.x** (≤ Python 3.8, weak .NET interop) and **3.x** (Python 3.7–3.12+,
much-improved interop — `out`-params as tuples, LINQ/extension methods, operator overloading,
better overload resolution). Dynamo brands their pythonnet-3 engine "PythonNet3" and reports
that most "previously impossible" limitations were **pythonnet-2.x** problems that pythonnet 3
fixed. Because pyRevit runs **CPython 3.12**, its fork is necessarily **pythonnet 3.x**
(confirmed by the `Py.GIL()` / `Python.Runtime` API in `CPythonEngine.cs`). **pyRevit's
`CPY3123` is already the "PythonNet3" equivalent** — it already has the improved interop and is
not stuck with the old-pythonnet limitations. The target is therefore *not* "adopt pythonnet 3"
(done); it is "port pyRevit's code and make `CPY3123` the default."

### 6.5 Prior art: Dynamo's IronPython → PythonNet3 migration
Dynamo performed essentially this migration and documented it publicly:
- *PythonNet3: a new Dynamo Python to fix everything* —
  https://dynamobim.org/pythonnet3-a-new-dynamo-python-to-fix-everything/
- *Dynamo PythonNet3 upgrade: a practical guide to migrating your Dynamo graphs* —
  https://dynamobim.org/dynamo-pythonnet3-upgrade-a-practical-guide-to-migrating-your-dynamo-graphs/

Relevance to this plan:
- **Same path.** Dynamo's staged route was **IronPython2 → IronPython3 → PythonNet3** — the
  trajectory in §9.4.
- **Validates Option B.** Even on pythonnet 3, Dynamo reports WPF-in-Python still needs
  "namespace manipulation and property declaration boilerplate, substantially more verbose than
  IronPython" — i.e. WPF stays painful in Python, arguing to move it to C# (§7) rather than
  reimplement it in Python (§9.1).
- **Concrete API migration checklist** (what real scripts hit, expanding §6.2): `out`/`ref` →
  return tuple; enum→int now explicit; `super().__init__()` required for .NET subclasses;
  collections are views (use `list()`); explicit `List[T][...]` casts; LINQ lambdas need
  `System.Func[...]`; indexer `get_` accessors; `IEnumerable` not bracket-indexable;
  `with`/`IDisposable` needs explicit `.Dispose()`; interface implementation needs a
  `__namespace__` attribute; COM/GAC access lost on .NET Core (use Python libs like `openpyxl`).
- **Ecosystem churn is a real cost** — Dynamo's upgrade jumped numpy 1.24→2.1 and pandas
  1.5→2.2, whose own breaking changes broke scripts independent of the engine swap (see §8.5).
- **Coexistence & security corroboration** — CPython and PythonNet3 engines coexist per-node
  (pyRevit already selects engine per-button, supporting incremental migration), and Dynamo
  cites Python 2's 15-year lack of security updates (see §8.6).

---

## 7. Proposed Direction: C# UI Layer + Thin Python API (Option B)

> **Framing:** "C# UI layer" is shorthand. In practice this is **C#-assisted, mostly-Python** —
> most of `forms` stays pure Python under pythonnet, and C# is used surgically for a few
> high-leverage primitives. See §7.10 for the verified breakdown of what needs C# and what does
> not.

### 7.1 The proven precedent: `ScriptOutput` / `PyRevitOutputWindow`
The output console is **already** a C#-hosted WPF window (`ScriptOutput.cs`) driven from Python
via a thin wrapper (`output/__init__.py`, forwarding through `__getattr__`). Scripts call
`script.get_output()` and get the same object on any engine, because they are just calling
.NET methods. Option B generalizes this pattern from the console to dialogs.

### 7.2 MVVM primer (why the ViewModel must be .NET-visible)
- **Model** = domain data (Revit elements). **View** = XAML. **ViewModel** = properties +
  commands the View binds to, raising `INotifyPropertyChanged` so the UI updates automatically.
- WPF binding reflects over .NET objects. A **Python** ViewModel is visible to binding under
  IronPython but **not** under pythonnet. The rule: **the binding target must be a .NET object.**
  Example: a batch-renumber dialog whose preview grid recomputes live as inputs change and
  whose Apply button auto-disables on a numbering collision — this reactivity needs a
  .NET-visible ViewModel.

### 7.3 `forms.DataGrid` — the imperative "dump data" path (no C# required)
Backed by a C# window over a `System.Data.DataTable`. Author fills columns/rows from Python,
reads selection/edits back. Works on every engine; **no ViewModel, no C#**:
```python
grid = forms.DataGrid(title="Renumber Sheets",
                      columns=[forms.Column("sheet", readonly=True),
                               forms.Column("new", width=120)])
grid.set_data(rows=[{...}, {...}])     # bulk load — one boundary crossing
if grid.show_dialog():
    for row in grid.rows: apply(row["sheet"], row["new"])
```

### 7.4 `forms.BindableModel` — the MVVM / data-binding path
A C#-defined `INotifyPropertyChanged` base with a dynamic property bag + `ICommand` factory,
driven from Python. Assignments raise change notifications, so `{Binding}` reacts live.
Collection items must also be `.NET`-visible (`BindableRow` / another `BindableModel`).

### 7.5 Where the Python/C# boundary lands
| Layer | Owner |
|---|---|
| XAML files (layout, styles, cell templates, localization) | Python-side author (declarative) |
| Data, event handlers, command logic, validation | Python |
| Window host + `XamlReader` loader + `FindName` | C# (pyRevit) |
| `DataGrid`/`DataTable`, `BindableModel`/`BindableRow`/`ICommand` | C# (pyRevit) |
| Delegate marshaling for callbacks (C#→Python) | Runtime (IPy native; pythonnet `Func`/`Action`) |

The only cross-boundary mechanic is "invoke a Python callable" — supported by both engines.

### 7.6 Authoring a custom XAML dialog in pure Python
You do **not** have to write C#. In pure Python (all engines) you can author XAML, load it,
find named controls, attach handlers, read/write control state, and dump rows into a
`DataGrid`. The **only** capability that needs a .NET binding target is declarative `{Binding}`
to your data — covered by `BindableModel`/`DataTable`. You would reach for hand-written C# only
for a heavily-typed, logic-rich ViewModel, and even that is optional.

### 7.7 Concrete code comparison

All snippets below are **illustrative** — the current-state ones reflect real pyRevit APIs;
the Option A / Option B ones sketch the shape each approach would take. The point is the
*developer experience*, not copy-pasteable code.

**Note on built-in dialogs:** for callers of built-in dialogs (`forms.alert`,
`forms.SelectFromList`, ...), Options A and B look **identical** — the Python caller API is
preserved in both, and the difference is purely internal (Option A reimplements each dialog in
Python; Option B in C#). The developer-visible divergence appears when **authoring a custom
dialog**, so the scenario below is a custom live-preview dialog (a batch-renumber tool, modeled
on the existing `ReValue` button).

#### Baseline — today, on IronPython (works)
```python
#! (ironpython — default)
from pyrevit import forms

class RenumberWindow(forms.WPFWindow):
    def __init__(self, xaml):
        forms.WPFWindow.__init__(self, xaml)     # loads XAML *into self*, auto-wires controls
        self.preview_dg.ItemsSource = self._rows  # bind grid directly to Python row objects

    def on_prefix_changed(self, sender, args):    # handler named in XAML resolves to this method
        self._recompute()
        self.preview_dg.Items.Refresh()

RenumberWindow("renumber.xaml").show_dialog()
```
```xml
<!-- renumber.xaml -->
<TextBox  x:Name="prefix_tb" TextChanged="on_prefix_changed"/>
<DataGrid x:Name="preview_dg">
  <DataGrid.Columns>
    <DataGridTextColumn Binding="{Binding new_number}"/>   <!-- binds to Python object attr -->
  </DataGrid.Columns>
</DataGrid>
```
Three IronPython-only conveniences make this terse: XAML loads *into* `self`, `x:Name` controls
auto-appear as `self.<name>`, and `{Binding}` resolves against Python objects. Under
`#! python3` today this class raises `PyRevitCPythonNotSupported` at construction.

#### Option A — the same custom dialog, reimplemented in Python on pythonnet
```python
#! python3
from System.Windows.Markup import XamlReader
from System.IO import File

class RenumberWindow:                              # composition, not inheritance
    def __init__(self, xaml_path):
        with File.OpenRead(xaml_path) as fs:
            self.window = XamlReader.Load(fs)      # returns a NEW object graph (not self)
        # no auto-wiring — every control fetched by name:
        self.prefix_tb  = self.window.FindName("prefix_tb")
        self.preview_dg = self.window.FindName("preview_dg")
        # events wired by hand (XAML handler names do NOT resolve to Python methods):
        self.prefix_tb.TextChanged += self.on_prefix_changed
        # {Binding} to Python objects does NOT work under pythonnet -> feed .NET-visible rows:
        self.preview_dg.ItemsSource = to_datatable(self._rows).DefaultView

    def on_prefix_changed(self, sender, args):
        self._recompute()
        self.preview_dg.ItemsSource = to_datatable(self._rows).DefaultView   # re-push

    def show(self):
        self.window.ShowDialog()
```
Every custom-dialog author must now: use composition, `FindName` each control, wire events
manually, and convert Python data to a `DataTable` (or hand-roll an `ICustomTypeDescriptor`
adapter) because binding to Python objects is dead. This boilerplate repeats in every extension.

#### Option B — the same dialog, engine-agnostic, via pyRevit's C# layer

*MVVM path (`forms.BindableModel`):*
```python
#! python3   (identical code runs on IronPython too)
from pyrevit import forms

class RenumberVM(forms.BindableModel):
    def __init__(self, sheets):
        super().__init__()
        self._sheets = sheets
        self.prefix  = "A"                     # assignment auto-raises PropertyChanged
        self.preview = self.observable([])     # .NET ObservableCollection
        self.apply_cmd = self.command(self._apply, can_execute=self._valid)
        self._recompute()

    def _on_property_changed(self, name):      # react to input changes
        if name == "prefix":
            self._recompute()

    def _recompute(self):
        self.preview.reset(forms.BindableRow(r)
                           for r in renumber(self._sheets, self.prefix))

    def _valid(self):  return not collisions(self.preview)   # drives Apply enable/disable
    def _apply(self):  ...; self.close()

forms.WPFWindow("renumber.xaml", context=RenumberVM(sheets)).show_dialog()
```
```xml
<!-- renumber.xaml — no code-behind, no FindName, no manual wiring -->
<TextBox  Text="{Binding prefix, UpdateSourceTrigger=PropertyChanged}"/>
<DataGrid ItemsSource="{Binding preview}"/>
<Button   Content="Apply" Command="{Binding apply_cmd}"/>
```

*No-ViewModel path (`forms.DataGrid`) — when you just need to show/edit a table:*
```python
#! python3
from pyrevit import forms

grid = forms.DataGrid(title="Renumber Sheets", columns=["sheet", "old", "new"])
grid.set_data(rows=[{"sheet": s.name, "old": s.number, "new": proposed[s]}
                    for s in sheets])          # bulk load = one boundary crossing
if grid.show_dialog():
    for row in grid.rows:                       # read back edited values
        apply_number(row["sheet"], row["new"])
```

#### Where Option B's cost lives — the C# side (authored once)
The ergonomics above depend on a small amount of C# pyRevit ships. `BindableModel` is a thin
Python wrapper (same forwarding trick as `PyRevitOutputWindow`) over a C# core that WPF binding
can see:
```csharp
// Illustrative. The crux is exposing dynamic members to WPF's binding engine
// (via DynamicObject / ICustomTypeDescriptor) and raising change notifications.
public class BindableCore : DynamicObject, INotifyPropertyChanged {
    private readonly Dictionary<string, object> _bag = new();
    public event PropertyChangedEventHandler PropertyChanged;

    public void Set(string name, object value) {
        _bag[name] = value;
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));  // UI updates
    }
    public object Get(string name) => _bag.TryGetValue(name, out var v) ? v : null;

    public ICommand Command(Func<bool> canExec, Action exec) => new RelayCommand(canExec, exec);
}
```
```python
# Thin Python wrapper — mirrors output/__init__.py's PyRevitOutputWindow forwarding
class BindableModel(object):
    def __init__(self):
        object.__setattr__(self, "_core", runtime.BindableCore())
    def __setattr__(self, name, value):     # self.prefix = x  -> raises PropertyChanged
        if name.startswith("_"):
            object.__setattr__(self, name, value)
        else:
            self._core.Set(name, value)
    def __getattr__(self, name):
        return self._core.Get(name)
```
This C# lives in one place and serves **every** engine, versus Option A's adapter logic that
each Python author reinvents.

#### Developer-experience summary
| Concern | Baseline (IPy) | Option A (CPython, Python) | Option B (C# layer) |
|---|---|---|---|
| Works under `#! python3` | ✗ | ✓ | ✓ |
| Works under IronPython | ✓ | (n/a) | ✓ (one codebase) |
| XAML load model | into `self` | `XamlReader` + `FindName` | into window (C#-handled) |
| Control access | auto `self.<name>` | manual `FindName` per control | auto / `FindName` (C#-handled) |
| Event wiring | XAML handler names | manual `+=` in Python | `{Binding}` command / handlers |
| Bind to your data | Python objects (native) | must convert to `DataTable`/adapter | `BindableModel`/`BindableRow` |
| Boilerplate location | none | repeated per extension | written once in C# |
| "Show a table" quick path | `SelectFromList` | hand-rolled | `forms.DataGrid` |

### 7.8 The C# forms layer is a convenience, not a gate
Under pythonnet 3 users can build **arbitrary WPF entirely in Python** without `pyrevit.forms`
(`clr.AddReference("PresentationFramework")` → `XamlReader.Load` → `FindName` → `+=` handlers →
`ShowDialog()`). The C# API does not lock anyone in; it coexists with hand-rolled WPF (a single
command can use both). What `forms` *buys* is absorbing two kinds of baggage authors would
otherwise re-solve every time:
- **The general pythonnet-WPF tax** (§7.7 / §6.5): composition + `FindName`, manual event
  wiring, no `{Binding}` to Python objects, `super().__init__()`, `__namespace__` for interfaces.
- **Revit-host concerns** — window ownership/parenting to the Revit window
  (`WindowInteropHelper` + `AdWindows.ComponentManager.ApplicationWindow`), assembly references
  (`PresentationFramework`/`PresentationCore`/`WindowsBase`/`System.Xaml`, incl. the .NET 8
  Desktop runtime on Revit 2025+), MahApps + pyRevit dark-theme resources, and — the sharpest
  edge — **modeless windows that call the Revit API**, which must marshal back via
  `ExternalEvent`/`IExternalEventHandler` (pythonnet needs the `__namespace__` interface trick +
  GIL care). Modal dialogs are simple DIY; modeless-with-API-callbacks is genuinely tricky.

So the value proposition is "paved road + open dirt road": `forms` saves ~95% of authors from
re-implementing ownership, `ExternalEvent` marshaling, theming, and binding adapters, while the
raw-pythonnet escape hatch stays fully available for power users and exotic UI.

### 7.9 Incremental migration path
The port is **not** a 4,100-line big bang. The existing `_cpy.py` backend + the module-level
`__getattr__` catch-all in `forms/__init__.py` are already an incremental scaffold: implement
one symbol (real code in `_cpy.py`, or a thin wrapper forwarding to a new C# element) and add
it to `__all__`; everything not yet done keeps raising a precise
`PyRevitCPythonNotSupported("pyrevit.forms.X")`. Each release moves a few functions from stub to
real with nothing else breaking.

Suggested ordering (easy → hard):

| Tier | Elements | Effort | Notes |
|---|---|---|---|
| 0 | `check_*` validators | trivial | pure logic, no WPF |
| 1 | `alert`, `pick_file`/`pick_folder`/`save_file`, `toast`, `show_balloon` | easy | native dialogs; no XAML/binding |
| 2 | `SelectFromList`, `CommandSwitchWindow`, `ask_for_one_item`, `select_*` family | medium | data-bound list → `DataGrid`/`DataTable` |
| 3 | `ask_for_string`/`_date`/`_number_slider`/`_color` | medium | small value-input windows |
| 4 | `ProgressBar`, dockable panels (`WPFPanel`, open/close/toggle) | harder | modeless + threading + `ExternalEvent` |
| 5 | `WPFWindow`/`WPFPanel` base classes | hardest | user-subclassable custom-dialog API |

Planning notes:
- **Built-in dialogs are cleanly incremental; the subclassable base is one deliberate step.**
  Leaf dialogs (Tiers 0–4) are independent, added one at a time. `WPFWindow`/`WPFPanel` (Tier 5)
  is infrastructure users subclass — it can't be half-shipped. Under **Option B this is easier**:
  built-ins are C# and need no Python-subclassable base, so Tiers 0–4 ship *without* exposing
  `WPFWindow`, and the custom-authoring API (`forms.WPFWindow`/`BindableModel`) becomes a
  distinct final milestone.
- **Hold parity with `_ipy.py`.** Keep public signatures and observable behavior identical
  across engines; `_ipy.py` is the reference. A small per-element parity test is worthwhile.
- **Low risk by construction.** Per-button engine selection lets authors keep IronPython for
  any not-yet-ported dialog (no forced migration); the `PyRevitCPythonNotSupported` errors are a
  precise, actionable coverage signal (could point at a live coverage matrix); and new elements
  are testable side-by-side against `_ipy`.

### 7.10 How much actually needs C# (the hybrid reality)
"C# UI layer" overstates it. Verified against `_ipy.py`, **most of `forms` can stay pure Python**
under pythonnet, and C# is warranted only for a small, high-leverage subset. The realistic
recommendation is **C#-assisted, mostly-Python**, not a wholesale C# rewrite.

**Stays pure Python — no C# (native dialogs / logic / orchestration):**

| Element | Backing (verified in `_ipy.py`) |
|---|---|
| `alert`, `alert_ifnot`, `inform_wip` | `UI.TaskDialog` (Revit native) |
| `pick_file`, `save_file` | `Forms.OpenFileDialog` / `SaveFileDialog` |
| `pick_folder` | `CPDialogs.CommonOpenFileDialog` / `Forms.FolderBrowserDialog` |
| `ask_for_color` | `Forms.ColorDialog` |
| `pick_excel_file`, `save_excel_file` | wrappers over the above |
| `toast`, `show_balloon` | native toast / InfoCenter |
| `check_*` validators | pure logic, no UI |
| `select_*` / `ask_*` orchestration | thin Python delegating to the above/below |

These are `.NET` calls with no XAML-into-self and no Python-object binding — they port directly
(only the §6.2 idiom cleanups). `alert` and `pick_file` alone are among the most-used forms
functions and need zero C#.

**Needs the WPF-host layer — mostly still Python; C# only where it pays:**

| Element(s) | What it needs | Pure-Python viable? | C# leverage |
|---|---|---|---|
| `ask_for_string`, `ask_for_date`, `ask_for_number_slider` | small window, imperative control read | ✓ yes | low |
| `SelectFromList`, `CommandSwitchWindow`, `ask_for_one_item`, underlying `select_*` | data-bound list | ✓ via `DataTable` | medium (nicer binding) |
| `SearchPrompt` | filtered list + keyboard handling | ✓ mostly | low–medium |
| `ProgressBar`, `WarningBar`, `TemplatePromptBar` | **modeless** window + threading + `ExternalEvent` | ✗ tricky (GIL, `__namespace__`) | **high** |
| dockable panels (`WPFPanel`, register/get/open/close/toggle) | modeless dockable + API context | ✗ tricky | **high** |
| `WPFWindow` / `WPFPanel` (user-subclassable base) | XAML-into-self, auto-wire, ownership, theming | partial (composition works, verbose) | **high** (ergonomics + plumbing once) |
| reactive / MVVM custom dialogs | declarative `{Binding}` to Python view models | ✗ | **high** (`BindableModel`) |

**The two C# primitives worth building** (everything else calls into them or stock .NET):
1. **Window-host / plumbing base** — writes the hardest Revit-host mechanics *once*: modeless
   `ExternalEvent`/`IExternalEventHandler` marshaling, window ownership/parenting to the Revit
   window, and MahApps/dark-theme resources. High leverage — every modeless dialog needs it.
2. **`BindableModel` / bindable collections** — clean declarative MVVM binding to Python-driven
   data; the one thing pure pythonnet can't do gracefully (`DataTable` covers tabular cases
   without it).

**Consequence for sequencing:** Tiers 0–3 of §7.9 need **no C# at all**; C# arrives late and
surgically at the modeless/`ProgressBar` tier (→ the host base) and reactive custom dialogs
(→ `BindableModel`). It is a dial, not a binary — pick how much ergonomics to buy.

---

## 8. Decision Factors

### 8.1 Does it remove the IronPython dependence?
It **demotes, not deletes.** pyRevit's core becomes engine-independent, but the large installed
base of IronPython-2 extensions means IronPython stays as an opt-in compatibility engine —
likely for years. "Without losing functionality" is true for pyRevit's *features*; it is not
automatic for the *third-party extension ecosystem*.

### 8.2 Ecosystem unlock (numpy/pandas)
Real and valuable — but **already works today**, opt-in, via `#! python3`. The plan makes
CPython the default and removes the IPy gate. Friction: pyRevit ships an **embedded** CPython
(no bundled pip by default), so "how do I install numpy into pyRevit's Python" is a genuine UX
cost; C-extension packages need the right ABI/install path.

### 8.3 Performance implications of the C# forms API
- **Neutral-to-positive for typical dialogs.** Compiled XAML (BAML) can beat runtime
  `wpf.LoadComponent` parsing; binding to typed .NET objects beats IronPython's dynamic
  Python-object binding for large grids.
- **From IronPython callers:** no change (.NET→.NET).
- **From CPython callers:** per-call marshaling + GIL overhead across the pythonnet bridge —
  negligible for construct-once/read-once dialogs, but real if the API is chatty. **Mitigation
  is design:** bulk APIs (`set_data`) over per-item calls, and batched UI updates (the
  `ScriptOutput` `freeze()`/`unfreeze()` + flush pattern already demonstrates this).
- **Watch:** GIL × Dispatcher × Revit single-threaded context for background-threaded dialogs
  (e.g. progress bars) needs testing, not just porting.

### 8.4 Ownership & feasibility
pyRevit controls the loader, the wrapper library, **and all engines**: IronPython 2, IronPython
3, and pythonnet are pyRevit forks (submodules under `pyrevitlabs/*`, e.g.
`pyrevitlabs/pythonnet` branch `pyrevit-5-main`). The Revit-API-interaction fixes are pyRevit's
own Python code; even bridge-level changes can be made in the pythonnet fork and re-vendored.
The only external, unchangeable component is Autodesk's Revit API — and it is engine-agnostic.

### 8.5 Risks & downsides
1. **Ecosystem backward-compat break** (dominant risk): community extensions are largely
   IronPython 2; a CPython default forces mass migration or a permanently split ecosystem.
2. **Maintenance surface grows before it shrinks:** IPy2 and/or IPy3 + CPython + the C# layer,
   all at once, through a long transition.
3. **pythonnet becomes load-bearing**, with its own quirks (GIL × Dispatcher × Revit API).
4. **The `forms` rewrite is large and compatibility-sensitive**; custom-`WPFWindow` authors
   face a migration; CPython has no dialogs until it lands.
5. **Behavior/perf differences** across the ~25 `revit/` modules require per-case validation.
6. **Embedded-CPython packaging friction** (pip/site-packages, C-extension ABIs, version pins).
7. **Package-version churn** — moving to CPython pulls in modern package majors (Dynamo's
   upgrade jumped numpy 1.24→2.1, pandas 1.5→2.2), whose own breaking changes can break user
   scripts independently of the engine swap.

### 8.6 Security considerations
- **Neither engine is a security boundary.** Both run arbitrary Python at full Revit-process
  privilege (file system, network, `Process.Start`). IronPython's old CAS/AppDomain "sandbox"
  was never a real boundary and is gone in modern .NET. Primary defense in both worlds is
  **source trust of installed extensions**, not runtime isolation.
- **Interpreter patch velocity favors CPython** (funded security team, CVE process, fast
  releases) — *but* only if pyRevit keeps the **embedded** `CPY3123` current by re-vendoring;
  pinning accumulates known CVEs. IronPython's security fixes are slow/volunteer.
- **Supply-chain surface favors IronPython** (ironically): its inability to run most
  C-extensions limits PyPI exposure. A CPython default opens the full pip/PyPI surface
  (typosquatting, malicious packages) and **native C-extension execution** (unmanaged code,
  no CLR verification). This is the largest real security delta.
- **Execution model:** IronPython runs verifiable managed IL; CPython + native `.pyd`
  extensions introduce native memory-safety as a possible vuln class.
- **Maintenance:** pyRevit must track security fixes in *both* forks (IronPython, pythonnet)
  and refresh the embedded CPython. A CPython default raises the stakes on vetting third-party
  extensions **and their pip dependencies**.

> **Historical note — why Microsoft dropped IronPython:** a strategic de-prioritization
> (~2010), not a security failure. IronPython was the flagship of the DLR; when Microsoft wound
> down the dynamic-language bet (and its champion left), IronPython/IronRuby moved to community
> stewardship. Its inability to run CPython's C-extension ecosystem had already capped adoption.

---

## 9. Recommendation & Options

### 9.1 Option A — reimplement `forms` on pythonnet, in Python
Rewrite `_cpy.py` with `XamlReader` + `FindName` + an `ICustomTypeDescriptor`/
`INotifyPropertyChanged` adapter for binding. **Pro:** stays in Python. **Con:** re-solves the
entire WPF-binding problem per engine; permanently maintains engine-specific UI code
(`_ipy.py` + a heavy `_cpy.py`).

### 9.2 Option B — C# UI layer + thin Python API *(recommended)*
Move the WPF layer into C# (`forms.DataGrid`, `forms.BindableModel`), Python calls in. **Pro:**
one implementation for all engines; kills the `_ipy`/`_cpy` divergence; aligns with the C#
loader/bootstrap direction; MahApps/ControlzEx already vendored; the `ScriptOutput` precedent
proves it. **Con:** large, compatibility-sensitive C# rewrite; migration for custom-dialog
authors.

### 9.3 Option C — IronPython 3 as default, keep `forms` as-is
Switch the default IronPython engine to `IPY342`. **Pro:** cheapest way off Python 2; `forms`
and `revit/` carry across as a syntax bump; no marshaling rewrite. **Con:** capped at ~Python
3.4; **no** numpy/pandas ecosystem; bets the default on a slow, small-team fork.

### 9.4 Suggested sequencing
**Destination: `CPY3123` (CPython 3.12 + pythonnet 3) as the default engine.** Steps 1–4 are
scaffolding on the way there.
1. Land the C# bootstrap (#3438); confirm IronPython is no longer the startup dependency.
2. Finish the `pyrevitlib` syntax port and shim `clr.AddReferenceToFileAndPath` in
   `framework.py`; port the `out`-param sites, collection constructions, and the
   enum/indexer/`with`/interface idioms in `revit/` (checklist in §6.5).
3. Re-vendor Python-3 releases in `site-packages/`; drop the Py2 backports.
4. Build the Option B `forms` C# layer + thin Python API; migrate built-in dialogs first,
   provide the callback bridge for custom `WPFWindow` authors.
5. Flip the default to `CPY3123`; keep an **opt-in legacy IronPython engine** for un-migrated
   extensions, then sunset it once the ecosystem has moved.

**On IronPython 3:** treat it as *optional* scaffolding. It helps only if it lets you do the
Python 2→3 *syntax* migration while keeping IronPython's native .NET integration (so
forms/raw-API code keeps working mid-port). It is **not** a destination (capped ~Python 3.4, no
numpy, small-team fork). If IPy2 code can be ported straight to PythonNet3, IPy3 can be skipped.

---

## Appendix A — File & code reference index
- Engine selection / shebang: `dev/pyRevitLabs.PyRevit.Runtime/scriptruntime.cs`,
  `ScriptEngines.cs` (`ScriptEngineType` enum), `IronPythonEngine.cs` (`#if IPY342`),
  `CPythonEngine.cs`, `ScriptEngineManager.cs`.
- C# loader / bootstrap: `dev/pyRevitLoader/pyRevitAssemblyBuilder/**`
  (`SessionManager/SessionManagerService.cs`, `AssemblyMaker/CommandTypeGenerator.cs`),
  `dev/pyRevitLoader/Source/PyRevitLoaderApplication.cs`, `Source/ScriptExecutor.cs`.
- Session dispatch: `pyrevitlib/pyrevit/loader/sessionmgr.py`
  (`_new_session`, `_new_session_csharp`, `execute_extension_startup_script`).
- Coupling: `pyrevitlib/pyrevit/compat.py`, `framework.py`, `revit/db/query.py`,
  `revit/db/create.py`, `revit/events.py`.
- Forms: `pyrevitlib/pyrevit/forms/__init__.py`, `_ipy.py`, `_cpy.py`,
  `pyrevitlib/pyrevit/output/__init__.py`, `dev/pyRevitLabs.PyRevit.Runtime/ScriptOutput.cs`.
- Config: `pyRevitfile` (`[engines.*]`), `.gitmodules` (engine forks), `docs/architecture.md`,
  `docs/ci-cd.md` (vendored DLLs).

## Appendix B — Related PRs
- **#3438** — in-process C# bootstrap; removes the IronPython bootstrap; `session_preload.py`/
  `session_postload.py`; drops the `new_loader` toggle and `uimaker.py`/`asmmaker.py`; requires
  Revit 2021+; gated for a 7.0 release. *(Pre-#3438 on this branch.)*
- C# Roslyn loader (`pyRevitAssemblyBuilder`) — landed, opt-in via `user_config.new_loader`.

## Appendix C — Glossary
- **`IPY2712PR`** — pyRevit fork of IronPython 2.7.12 (default engine).
- **`IPY342`** — pyRevit fork of IronPython 3.4.2 (opt-in, install-level).
- **`CPY3123`** — embedded CPython 3.12.3, driven via pythonnet (opt-in, per-script).
- **Roslyn loader** — C# path that generates command types as C# source compiled by Roslyn,
  replacing IronPython `Reflection.Emit`.
- **pythonnet** — CPython↔.NET *bridge* (not an interpreter); pyRevit fork
  `pyrevitlabs/pythonnet`. **v3.x** has the improved interop; pyRevit runs 3.x. See §2.4.
- **PythonNet3** — Dynamo's name for **CPython 3.x + pythonnet 3.x** together; pyRevit's
  `CPY3123` equivalent and the migration destination. See §2.4 / §6.4.
- **MVVM** — Model-View-ViewModel; WPF's binding-centric UI pattern.
- **BAML** — compiled/binary XAML produced at build time.
- **CAS** — .NET Framework Code Access Security (deprecated; never a real IronPython sandbox).
