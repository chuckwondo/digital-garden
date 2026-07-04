---
title: Python test suite structure
tags: [python, testing, pytest, project-structure]
stage: budding
planted: 2026-07-04
tended: 2026-07-04
---

*Where tests live, whether `tests/` is a package, and how to share code across them: it all follows from your import mode.*

**Thesis:** whether your `tests/` directory should be a *package* (carry an `__init__.py`) is not a standalone decision. It falls out of your **import mode**, and once you pick the modern default the answer is usually "no".

## The modern default

For a new project, the combination current pytest guidance points to:

- **src layout** (`src/yourpkg/`): forces tests to run against the *installed* package, not the working copy, so packaging mistakes surface.
- **`tests/` as a sibling** of `src/`, outside the package.
- **`importlib` import mode** (`--import-mode=importlib`): recommended for new projects. It pairs with src layout because the legacy `prepend` mode inserts the rootdir onto `sys.path` and can import your package twice (once installed, once via the insertion), which collides under `--doctest-modules`.
- **No `__init__.py` in `tests/`**: under importlib it is not needed, and omitting it is the recommended default.

## Import modes in one breath

- `prepend` / `append` (legacy): pytest inserts the test's rootdir into `sys.path`. Consequence: without `__init__.py`, two test files that share a basename collide, so you need unique names or packages.
- `importlib` (recommended): pytest imports each test module under a unique name and does **not** touch `sys.path`. Duplicate basenames are fine, no `__init__.py` needed. The trade-off: a test module can no longer import a sibling helper module by bare name for free (see [[sharing code between test files]]).

## Package or not?

| You want | Do |
|---|---|
| the default (importlib, no cross-file plain imports) | **no `__init__.py`** |
| `from tests.sub.x import y`, deep nested test packages | add `__init__.py` (make `tests` a package) |

Adding `__init__.py` is not free: it changes every test module's identity (`test_foo` becomes `tests.test_foo`) and risks tests being packaged or installed. Add it only for a concrete need.

## Sharing code between test files

The real question hiding under "package or not". Three idiomatic routes, lowest-ceremony first:

| Route | Best for | Cost |
|---|---|---|
| `conftest.py` **fixtures** | shared *setup / state* | none structurally; injected by parameter name |
| `pythonpath = ["tests"]` + `tests/_helpers.py` | shared *plain functions* | one ini line; no package, no `__init__`, no fixture indirection |
| `tests/__init__.py` package | deep nested test packages, explicit `tests.*` imports | changes module identity suite-wide; heaviest |

Key gotcha, verified below: under importlib, `from conftest import some_function` does **not** work (conftest is special-loaded by pytest, not placed on `sys.path`). Share *fixtures* through conftest; share *plain functions* through `pythonpath` plus a helper module.

## When to reach for each

- Start with **no `conftest.py` at all**. Add `tests/conftest.py` when your first genuinely shared *fixture* appears, not before. It is also the home for pytest hooks and config.
- To dedup shared *plain helpers* (object builders, fakes, canned data), use `pythonpath = ["tests"]` plus `tests/_helpers.py`. Prefer this over a factory fixture (which forces a fixture parameter onto every consuming test) and over making `tests` a package.
- Reach for a `tests` package only when you actually have nested test subpackages.

## Verify, do not trust (as of pytest 8.x)

```python
# tests/conftest.py
import pytest

def plain():
    return "x"

@pytest.fixture
def fx():
    return "x"
```

```python
# tests/test_probe.py
def test_import():      # FAILS under importlib: ModuleNotFoundError: No module named 'conftest'
    from conftest import plain

def test_fixture(fx):   # works: fixtures inject regardless of import mode
    assert fx == "x"
```

```toml
# pyproject.toml: lets `import _helpers` resolve with no __init__.py
[tool.pytest.ini_options]
pythonpath = ["tests"]
```

## Related

- [[pytest import modes]]
- [[src layout]]
- [[sharing code between test files]]
- [[tests as a package or not]]
- [[pytest fixtures vs helper functions]]

---

*Sources: pytest "Good Integration Practices" and "Import modes". Claims verified against pytest 8.x on 2026-07-04.*
