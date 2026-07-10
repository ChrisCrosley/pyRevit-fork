# pyRevit: Dependency Management Under the CPython Transition

> **Audience:** pyRevit core maintainers.
> **Status:** Analysis / decision-support. Companion to `IRONPYTHON_TO_PYTHON3_ANALYSIS.md`
> (referred to below as "the parent doc"). No code changes are proposed by this document.
> **Scope:** How the move to CPython 3.12 + pythonnet (`CPY3123`) as the default engine
> changes the way pyRevit packages/manages its **own** third-party dependencies
> (`site-packages/`) and the way **extensions — including third-party** — manage theirs.
> Framed by one question: *what happens if three different extensions all import numpy?*
> **Scope note:** Line/file references reflect the current state of the
> `claude/ironpython-cpython-transition` branch; treat them as pointers, not guarantees.

---

## 1. Executive Summary

The parent doc is thorough on the engine/session *lifecycle* (its §3.5) and on splitting
pyRevit's own vendored tree (its §9.2 step 3), but it treats **dependency management** as a
footnote. This document fills that gap. Two findings drive everything:

1. **The isolation boundary disappears.** IronPython runs one cached engine *per extension* —
   each with its own `sys.modules` and `sys.path`, so extensions cannot see each other's
   modules. CPython + pythonnet runs **one process-global interpreter** with **one shared
   `sys.modules`** that is never evicted. Isolation goes from "per extension, by construction"
   to "none." (Confirmed in code — §3.)
2. **Two distinct dependency layers get conflated.** pyRevit's *own* vendored packages and
   *extension* third-party packages are different problems with different answers. The parent
   plan addresses the first (a Py2/Py3 tree split) and is nearly silent on the second — which
   is exactly where the "3 extensions import numpy" question lands.

**Recommendation in one line:** adopt a **single curated, version-pinned shared environment**
for third-party packages (one numpy for everyone), restore IronPython-like isolation only for
first-party *extension code* via the parent doc's §3.5 mitigations, and offer an
**out-of-process worker** as the sole real escape hatch for genuinely conflicting dependencies.
Do **not** try to recreate "one environment per extension" under CPython — for the packages
that matter (C-extensions) it is technically impossible in-process, and where it is possible it
is unsafe or slow.

---

## 2. The two assumptions this analysis was asked to confirm

**"The engine/session framework changes entirely under CPython." — Confirmed.**
It is a different lifecycle model, not a tuning change:

- **IronPython** (`dev/pyRevitLabs.PyRevit.Runtime/ScriptEngineManager.cs`,
  `IronPythonEngine.cs`): engines are cached in a dictionary keyed by
  `SessionUUID : EngineType : CommandExtension`. Every extension gets its **own** engine; all
  buttons in one extension share it. `SetupSearchPaths` sets the path list *wholesale* per
  engine. Each engine has its own `sys.modules`.
- **CPython** (`CPythonEngine.cs`): `Start` calls `PythonEngine.Initialize()` **once,
  parameterless, process-global**. pythonnet has no sub-interpreter support, so there is exactly
  **one** interpreter, **one** never-evicted `sys.modules`, and a `sys.path` rebuilt *per run*
  (`StoreSearchPaths` → `RestoreSearchPaths` → `SetupSearchPaths`). `clean` / `persistent` from
  `bundle.yaml` have no CPython meaning; "refresh" is a full `PythonEngine.Shutdown()` that
  resets state for *every* CPython command at once.

**"CPython eventually becomes the default." — Confirmed** as the parent plan's explicit
destination (its §9), flipped via a per-extension `engine` declaration (its §9.3), with
IronPython demoted to an opt-in legacy engine.

---

## 3. The dependency model today (verified in code)

**First-party bundle code — per-component `lib/`.**
Each bundle component may carry a `lib/` folder (`COMP_LIBRARY_DIR_NAME = 'lib'`,
`pyrevitlib/pyrevit/extensions/__init__.py`). It is registered in
`genericcomps.py` (`GenericUIComponent._update_from_directory`) and inherited down the bundle
tree, so an extension-root `lib/` is visible to every button under it. It is added to
`sys.path` **first** — highest precedence. Whole-extension shared libraries use the separate
`.lib` extension type (`LIB_EXTENSION_POSTFIX = '.lib'`).

**Third-party code — one shared vendored tree.**
`MISC_LIB_DIR = op.join(HOME_DIR, 'site-packages')` (`pyrevitlib/pyrevit/__init__.py`) is
appended **last** to every extension's and every script's search paths
(`extensions/extensionmgr.py`, `loader/sessionmgr.py`; on the C# loader path,
`Constants.SITE_PACKAGES_DIR` under the discovered pyRevit root in
`pyRevitAssemblyBuilder/AssemblyMaker/CommandTypeGenerator.cs`). Because it is appended after
the bundle `lib/` dirs, a bundle can shadow a vendored package, and all extensions share the
one tree.

**No per-extension third-party mechanism exists.**
The `dependencies` field in `extension.json` (`extensions/extpackages.py`) resolves to **other
pyRevit extensions**, not pip packages — it drives an inter-extension install graph. There is
no `requirements.txt` convention, no pip step, no per-extension `site-packages`. (Grep of the
repo confirms: no `requirements.txt`, no `pip install`, no `ensurepip`.)

**The embedded CPython ships no package machinery.**
`release/cengines/CPY3123/` is a standard CPython **embeddable** distribution: `python312.dll`,
the stdlib as `python312.zip`, the `.pyd` C-extensions — and **no pip, no `Lib/`, no
`site-packages/`**. Its `python312._pth` is:

```
python312.zip
.

# Uncomment to run site.main() automatically
#import site
```

So the interpreter starts **isolated** (path built only from `._pth`, relative to the DLL), and
`import site` is disabled — the normal `site-packages` discovery machinery never runs.
`CPythonEngine` re-introduces pyRevit's `pyrevitlib/` and `site-packages/` folders manually via
the runtime's `SearchPaths`. **Consequence: "how does a user get numpy into pyRevit's Python"
has no built-in answer today** — the distro has no installer.

**CPython `sys.path` order per run** (from `CPythonEngine.SetupSearchPaths`):
`[ native baseline: python312.zip + engine dir ]` → `[ existing PYTHONPATH dirs ]` →
`[ SearchPaths: script dir, component lib/ + bin/, library extensions, pyrevitlib/, site-packages/ ]`.
Stdlib wins collisions (it is first); the shared `site-packages/` is last.

---

## 4. The question: three extensions import numpy

Under one process-global interpreter with one shared, never-evicted `sys.modules`, decompose
the cases:

### 4.1 Same version, from the shared tree — fine, and optimal
All three `import numpy`; the first triggers the load, it is cached process-wide, the other two
get the cached module. This is not a problem — it is the **best** behavior: import once, share
the warm module. It is the model to design *toward*, not away from.

### 4.2 Different versions, each bundled in its own `lib/` — silent first-import-wins
`sys.path` is rebuilt per run, so each extension's `lib/` is correctly on the path *for its own
run*. But `import` consults `sys.modules` **before** `sys.path`. Whichever extension runs first
caches its numpy process-wide; the next extension's `import numpy` finds it already loaded and
receives the **first** extension's version — its own `lib/` is never consulted. The bug is
**order-dependent** ("works unless that other button ran first in this session") and invisible
in single-extension testing.

### 4.3 C-extensions physically cannot coexist in two versions in one process
Even if you *wanted* to force isolation, you cannot load numpy 1.24 and numpy 2.1 into one
CPython process. numpy's dtype registry, the identity of the `ndarray` type object, and its
C-API tables are **process-global** C state, not something `sys.modules` namespaces. This is a
hard CPython limitation, not a pyRevit one. IronPython side-stepped it entirely by (a) never
running C-extensions and (b) giving each extension its own engine. CPython gives up both at
once.

### 4.4 The same failure hits pure-Python first-party code — worked example
The collision is not exotic; it happens with ordinary bundle libraries. Suppose:

- `A.extension/lib/get_door.py` — defines `get_door()` one way,
- `B.extension/lib/get_door.py` — a *different* `get_door()`.

Both scripts do `import get_door`.

| | IronPython (today) | CPython (shared interpreter) |
| --- | --- | --- |
| A runs, then B | isolated — separate engines, separate `sys.modules`; each gets its own file | A caches `sys.modules['get_door'] = <A's module>`; B's `import get_door` returns **A's** module — B's `lib/` never consulted |
| Symptom | none | `AttributeError` on a B-only function, **or silently-wrong results** if both define the same name differently |
| Determinism | n/a | order-dependent on which button ran first this session |

This is the headline symptom of losing per-extension engines, and it is exactly the parent
doc's §3.5 `sys.modules` hazard made concrete.

---

## 5. The eviction-policy hazard (a bug latent in the parent plan)

The parent doc's §3.5 mitigation is: *after each run, evict `sys.modules` entries whose
`__file__` lives under an extension directory.* That correctly fixes §4.4 — after A runs,
`get_door` is dropped, so B's run re-imports B's version. It is safe there **because
`get_door` is pure-Python, first-party, and re-importable.**

But the same rule is **actively unsafe** the moment it touches a C-extension:

- numpy / pandas run their C initialization exactly **once** per process. Evicting them from
  `sys.modules` and re-importing corrupts or crashes the C state (numpy explicitly refuses
  re-init).
- If an extension bundles numpy inside its own `lib/` (i.e. **under an extension directory**),
  the eviction rule *as written in §3.5* would target it.

**Required refinement (belongs in §3.5):** the eviction policy must key on *pure-Python
extension-namespace code*, and must **never** evict site-packages / pip / C-extension modules —
even when bundled under an extension dir. Practically: maintain an explicit allowlist of
extension namespaces to evict, and keep everything else warm. Evict **code**; never evict
**packages**.

---

## 6. Should "one environment per extension" be adopted for CPython? — No

"One environment per extension" means, in IronPython, one cached engine per extension → its own
`sys.modules` + `sys.path`. Recreating that under CPython needs one of three things, and each
fails for the case that matters:

1. **Multiple real interpreters in-process — blocked.** pythonnet hosts one process-global
   interpreter; there is no "N engines" dictionary to build. Hard wall.
2. **Fake it by swapping `sys.modules` / `sys.path` per extension — isolates pure-Python code,
   breaks exactly where you need it.** C-extension state is process-global, not in
   `sys.modules`; two swapped-in `numpy` wrappers point at the same C state (corruption) or
   force a forbidden second init (crash). Cross-extension object flow (`isinstance`, C-API)
   breaks. And you discard the import-once warm-cache win, paying hundreds of ms to re-import
   heavy libs per extension.
3. **Sub-interpreters (per-interpreter GIL, 3.12/3.13) — not viable now.** pythonnet does not
   support running under sub-interpreters, and many C-extensions lack multi-phase init /
   sub-interpreter support; object sharing across them is restricted. Long-horizon at best.

**Resolution — split the goal.** "One environment per extension" bundles two separate wishes;
serve them separately:

| What you want isolated | Right mechanism under CPython |
| --- | --- |
| First-party extension **code** (`lib/utils.py`, `get_door.py` collisions, §4.4) | §3.5 eviction (refined per §5) + namespaced `lib` packages. Cheap, safe; restores IronPython-like isolation for code. |
| Third-party **packages** (numpy, pandas) | **Do not isolate.** One curated, version-pinned shared env — one numpy for everyone. The only option that is both possible and performant. |
| The rare genuine version conflict | **Out-of-process worker** (subprocess with its own venv). The only real per-extension third-party isolation — at the cost of in-process Revit API access. |

---

## 7. Recommended model

1. **Single curated shared environment (platform-SDK model).** pyRevit owns one pinned set of
   third-party packages for the CPython engine, ABI-matched to CPython 3.12 `win_amd64`, on a
   security re-vendoring cadence. All extensions import the same versions. This leans into §4.1
   (warm cache) and matches today's shared-`site-packages/` shape — but with the added
   responsibilities of version pins and native-ABI matching that the parent doc's §8.5.7 numpy
   1.24→2.1 churn warning implies.

2. **A managed, writable user-site + a `pyrevit` CLI pip command.** Give the embedded CPython a
   real, pyRevit-managed writable package directory and a first-class install path (e.g.
   `pyrevit env pip install numpy`), resolving the parent doc's §8.2 "how do I get numpy" UX
   cost. This is still **one shared tree, one version per package** — it moves the tree from
   vendored to user-managed, it does not create isolation. It requires wiring pip into the
   embeddable distro (which ships none today, §3).

3. **Declaration + conflict-check, not isolation.** Let extensions *declare* pip requirements
   (extending the `dependencies` concept, §3, to packages) that resolve against the single
   shared env and **fail loud** when two extensions demand incompatible versions — instead of
   today's silent first-import-wins (§4.2). Loud-at-install/load beats order-dependent-at-run.

4. **Document per-extension bundling as an unsupported footgun.** Bundling third-party packages
   in a `lib/` works in single-extension dev and breaks under multi-extension production (§4.2,
   §4.3); C-extensions cannot be isolated at all; even pure-Python bundling is subject to
   first-wins on name collisions.

5. **Out-of-process worker as the escape hatch** for genuinely conflicting or heavy
   dependencies — opt-in, for compute-heavy / Revit-API-light tools, since it loses in-process
   API access.

6. **Refine the §3.5 eviction policy per §5** — evict extension-namespace *code*, never
   packages — and push **namespaced lib packages** (`import myext_lib.get_door`) in the
   migration guide so keys don't collide in the first place.

---

## 8. Suggested amendments to the parent plan

- **§3.5** — split the module-eviction policy into "evict extension-namespace code / never evict
  packages" (§5 above), and note the C-extension re-import hazard explicitly.
- **§9.2 step 3** — reframe the Py3 tree as a *curated, version-pinned platform SDK*, not merely
  a re-vendored copy; add the pip/user-site mechanism as an explicit deliverable (the embeddable
  distro ships no installer, §3).
- **§9.2 sequencing** — add a "Dependency Management" step: ship the pip/user-site mechanism and
  the declaration/conflict-check **before** the default flip (§9.3), alongside the §3.5
  isolation mitigations it depends on.
- **§8.5** — record that per-extension third-party isolation is *impossible in-process for
  C-extensions*, so the shared-environment model is a constraint, not a choice.

---

## Appendix — File & code reference index

- Engine lifecycle / caching: `dev/pyRevitLabs.PyRevit.Runtime/ScriptEngineManager.cs`,
  `IronPythonEngine.cs`, `CPythonEngine.cs` (`Start`, `StoreSearchPaths`, `RestoreSearchPaths`,
  `SetupSearchPaths`, `GetPythonDll`).
- Search-path construction (C# loader): `dev/pyRevitLoader/pyRevitAssemblyBuilder/AssemblyMaker/CommandTypeGenerator.cs`,
  `SessionManager/Constants.cs` (`PYREVIT_LIB_DIR`, `SITE_PACKAGES_DIR`), `SessionManager/SessionManagerService.cs`.
- Search-path construction (Python loader): `pyrevitlib/pyrevit/__init__.py`
  (`MAIN_LIB_DIR`, `MISC_LIB_DIR`), `extensions/extensionmgr.py`, `loader/sessionmgr.py`.
- Bundle `lib/` convention: `pyrevitlib/pyrevit/extensions/__init__.py`
  (`COMP_LIBRARY_DIR_NAME`, `LIB_EXTENSION_POSTFIX`), `extensions/genericcomps.py`.
- Extension metadata / `dependencies`: `pyrevitlib/pyrevit/extensions/extpackages.py`.
- Embedded CPython distro: `release/cengines/CPY3123/` (`python312.dll`, `python312.zip`,
  `python312._pth`); engine descriptor in `pyRevitfile` (`[engines.CPY3123]`).
