# Contribution #1: Implement lcase, ucase, and initcap string functions

**Contribution Number:** 1
**Issue:** https://github.com/Eventual-Inc/Daft/issues/3792
**Status:** Phase II Complete

---

## Why I Chose This Issue

Daft is a distributed dataframe library for processing large-scale and multimodal data. I want to work on it because I'm drawn to data infrastructure and want to learn how a tool like this is built.

This issue tracks adding a set of string functions to bring Daft to parity with PySpark, originally for Daft's Spark Connect support. From that list I chose `lcase`, `ucase`, and `initcap`: SQL and Spark-style helpers for lowercasing, uppercasing, and capitalizing the first letter of each word. They matter because these are widely-expected string operations that users coming from Spark, SQL, or other dataframe libraries will reach for, and their absence is a gap in Daft's expression API.

I specifically chose these three because they map closely to Python's own string methods (`str.lower()`, `str.upper()`, and `str.title()`), so the core logic is approachable and Python-focused, making them a well-scoped first contribution. There is also strong source material to learn from: Daft already implements similar string functions, so I can follow those existing patterns for how functions are defined, registered, and tested.

---

## Reproduction Process

### Steps to Reproduce

The bug is that `lcase`, `ucase`, and `initcap` don't exist yet. These steps show they're missing while `upper` and `lower` work. The released package is enough, with no source build needed.

1. Create a virtual environment (Python 3.10+):
   ```powershell
   py -3.14 -m venv daft-repro-venv
   ```

2. Install Daft:
   ```powershell
   daft-repro-venv\Scripts\python -m pip install daft
   ```

3. Save this as `repro.py`:
   ```python
   import daft
   df = daft.from_pydict({"x": ["hello world"]})

   # These exist and work:
   from daft.functions import upper, lower
   print("upper ->", df.select(upper(df["x"])).to_pydict())
   print("lower ->", df.select(lower(df["x"])).to_pydict())

   # These don't exist in the Python API:
   for name in ["ucase", "lcase", "initcap"]:
       try:
           __import__("daft.functions", fromlist=[name]).__dict__[name]
           print(f"{name}: found")
       except (ImportError, KeyError):
           print(f"{name}: missing")

   # And they don't exist in SQL:
   for fn in ["ucase", "lcase", "initcap"]:
       try:
           daft.sql(f"SELECT {fn}(x) AS r FROM df").to_pydict()
           print(f"SQL {fn}: found")
       except Exception as e:
           print(f"SQL {fn}: {str(e)[:60]}")
   ```

4. Run it:
   ```powershell
   daft-repro-venv\Scripts\python repro.py
   ```

5. Output:
   ```
   upper -> {'x': ['HELLO WORLD']}
   lower -> {'x': ['hello world']}
   ucase: missing
   lcase: missing
   initcap: missing
   SQL ucase: Unsupported SQL: 'Function `ucase` not found'
   SQL lcase: Unsupported SQL: 'Function `lcase` not found'
   SQL initcap: Unsupported SQL: 'Function `initcap` not found'
   ```

---

## Reproduction Evidence

- The reproduction script `repro.py` is committed locally on the branch `feat/initcap-ucase-lcase` in my local clone of my Daft fork (`AngelGalindo7/Daft`), at `C:\Users\agali\Documents\Daft`. It is not pushed to GitHub yet.
- Tested on Windows 11, Python 3.14, Daft 0.7.15.

---

## Implementation Plan

Daft's string functions live in `src/daft-functions-utf8/src/`, one file per function, all built from the same `ScalarUDF` template. `upper`, `lower`, and `capitalize` already follow it, so I'll copy that pattern. PR #7070 added six sibling functions from this same issue, so I'll use it as a reference.

- **`ucase` / `lcase`:** aliases for `upper` / `lower`. Add UDFs named `"ucase"` and `"lcase"` that reuse the same uppercase/lowercase logic, and register them in `lib.rs`.

- **`initcap`:** new file `initcap.rs` based on `capitalize.rs`, but uppercasing the first letter of every word instead of just the first letter of the string. The `heck` crate's `to_title_case` splits on case and punctuation, so it can't be reused here.

- **Register** the three functions in `src/daft-functions-utf8/src/lib.rs`.

- **Python API:** add wrappers in `daft/functions/str.py` and export them in `daft/functions/__init__.py`.

- **Tests:** add coverage in `tests/expressions/test_utf8.py` and `tests/series/test_utf8_ops.py`, covering empty and multi-word strings.

Writing the code needs a source build (`make build`), which I'll set up in Phase III.


# Contribution #2: Fix confusing default in the IFC import options dialogue

**Contribution Number:** 2
**Issue:** https://github.com/FreeCAD/FreeCAD/issues/30732
**Fork branch:** https://github.com/AngelGalindo7/FreeCAD/tree/ifc-import-default-representation

---

## Phase I Complete

### Why I Left My Previous Issue

My first issue (Contribution #1, Daft #3792) got flooded with comments splitting the work across several people and PRs. It became cluttered and hard to claim cleanly. With no response I moved to an unclaimed issue I can own end to end.

### Why I Chose This Issue

FreeCAD is an open-source parametric 3D CAD modeler. I chose this issue because it's a well scoped, self contained fix in FreeCAD's BIM workbench.

---

## Phase II Complete

### Reproduction Process

**Steps to Reproduce**

1. Download and open the latest FreeCAD release.
2. Switch to the BIM workbench.
3. Open (import) any `.ifc` file.
4. In the import dialogue, the "Representation type" dropdown shows `Load the shape (slower)` as the default — even though `Load 3D representation only, no shape` is the one marked `(default)`.

### Reproduction Evidence

Branch in my fork: https://github.com/AngelGalindo7/FreeCAD/tree/ifc-import-default-representation

### Implementation Plan

Two approaches:

- **A:** change the code's fallback default to `1` (3D representation only), so it matches the `(default)` label.
- **B:** keep the code on `0` (Load the shape) and move the `(default)` label to that option instead.

Going with **A**, because the `.ui` files and the Native IFC preferences page already default to `1` — the code was the only place out of step. Fix: change `PARAMS.GetInt("ShapeMode", 0)` to `PARAMS.GetInt("ShapeMode", 1)` in `ifc_import.py`.

