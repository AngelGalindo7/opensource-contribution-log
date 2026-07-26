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

---

## Phase III Complete

### Implementation Notes

Applied Approach A: a one-line change in `ifc_import.py` (`get_options`), setting the `ShapeMode` fallback from `0` to `1`. The import dialogue now opens on `Load 3D representation only, no shape (default)`, matching the `(default)` label, the `.ui` files, and the preferences page.

### Code Changes

Branch: https://github.com/AngelGalindo7/FreeCAD/tree/ifc-import-default-representation

Commit: https://github.com/AngelGalindo7/FreeCAD/commit/c5bf4b0aad26dd7ecc5c4486e76ff7b98ddae3be

### Testing Strategy

Manual before/after check in FreeCAD 1.1.1:

1. Open the BIM workbench and import an `.ifc` file.
2. Before the change, the import dialogue opens with `Load the shape (slower)` selected.
3. After the change, it opens with `Load 3D representation only, no shape (default)` selected — confirming the default now matches the `(default)` label.

---

## Phase IV Complete

**Status:** Merged

### Pull Request

https://github.com/FreeCAD/FreeCAD/pull/31154

### Summary of Contribution

Fixed the confusing default in the IFC import options dialogue: a one-line change in `ifc_import.py` setting the `ShapeMode` fallback from `0` to `1`, so the dialogue opens on `Load 3D representation only, no shape (default)` — matching the `(default)` label, the `.ui` files, and the preferences page.

### Feedback / Next Steps

PR submitted upstream to FreeCAD; awaiting maintainer review. Will respond to any requested changes.

**Update (2026-07-12) — staying in Phase IV:** Auto-merge is enabled and CI is passing. Merged the latest `main` in, so there are no conflicts. Just waiting for the PR to land.

**Update (2026-07-19):** PR merged.


# Contribution #3: Add `airflowctl tasks states-for-dag-run` CLI command

**Contribution Number:** 3
**Issue:** https://github.com/apache/airflow/issues/66175

---

## Phase I Complete

### Why I Chose This Issue

Apache Airflow is an open-source workflow orchestration platform for scheduling and monitoring data pipelines. I chose this issue because it's a well scoped, additive CLI feature — adding an `airflowctl tasks states-for-dag-run` command — that follows an existing pattern from sibling `airflowctl tasks` commands, and the issue spells out where the code goes and asks for unit and integration tests. It's unassigned with no comments, in an actively maintained project.


# Contribution #4: Fix tekken tokenizer conversion crash in `convert_hf_to_gguf`

**Contribution Number:** 4
**Issue:** https://github.com/ggml-org/llama.cpp/issues/25359
**Fork branch:** https://github.com/AngelGalindo7/llama.cpp/tree/fix/convert-tekken-vocab

---

## Phase I Complete

### Why I Chose This Issue

llama.cpp is the C/C++ inference engine that runs LLMs locally, and the substrate a lot of the ecosystem (Ollama, LM Studio, Jan) is built on. I picked it as a project to contribute to consistently rather than one-off, because outside contributors actually get merged there.

I chose this issue because it's a small, provable regression in the model-conversion path — a missing `return` — and it sits in the niche I want to build in: conversion and tokenizer support, where every new model release generates fresh work. It was unassigned with no comments, and the whole repro is two JSON files and a download, no GPU needed.

---

## Phase II Complete

### Reproduction Process

**Steps to Reproduce**

1. Clone llama.cpp and install the conversion requirements.
2. Download a HF-layout model that ships `tekken.json` but no `tokenizer.json`:
   ```powershell
   hf download mistralai/Voxtral-Mini-3B-2507 --exclude consolidated.safetensors --local-dir models/Voxtral-Mini-3B-2507
   ```
3. Convert it without `--mistral-format`:
   ```powershell
   py convert_hf_to_gguf.py models/Voxtral-Mini-3B-2507 --outfile voxtral.gguf --outtype q8_0
   ```
4. The mistral vocab is written successfully, then the run keeps going through the sentencepiece → llama_hf → gpt2 ladder and dies on the last one:
   ```
   AttributeError: 'MistralCommonTokenizer' object has no attribute 'vocab'
   ```

### Reproduction Evidence

Full before/after tracebacks are in the PR description. Tested on Windows 11, no GPU needed.

### Implementation Plan

`LlamaModel.set_vocab` in `conversion/llama.py` has a tekken branch that calls `self._set_vocab_mistral()` without returning, so a successful vocab write still falls through to the remaining tokenizer paths, and `_set_vocab_gpt2` chokes on the `MistralCommonTokenizer` wrapper. The `is_mistral_format` branch directly above it does return.

Tracing the history: the `return` was there in #14862, then dropped by the refactor in #14737 that renamed `set_vocab_tekken()` to `_set_vocab_mistral()`. #17114 carried the behavior over when it split the converter into the `conversion/` package.

Fix: add the `return` back to the tekken branch, restoring the behavior from #14862.

---

## Phase III Complete

### Implementation Notes

A one-line change in `conversion/llama.py` (`LlamaModel.set_vocab`): `self._set_vocab_mistral()` becomes `return self._set_vocab_mistral()`, so the tekken branch stops falling through into the other tokenizer paths.

### Code Changes

Branch: https://github.com/AngelGalindo7/llama.cpp/tree/fix/convert-tekken-vocab

Commit: https://github.com/AngelGalindo7/llama.cpp/commit/cb324ae70b1e04b45ef1961afceaf2a4b956f2ac

### Testing Strategy

Before/after run of the same conversion command on Voxtral-Mini-3B-2507:

1. Before the change, the run crashes with `AttributeError: 'MistralCommonTokenizer' object has no attribute 'vocab'`.
2. After the change, it writes `voxtral.gguf` (273 tensors, 4.3G) and reports `Model successfully exported` — confirming the tekken vocab path now ends where it should.

---

## Phase IV Complete

**Status:** Awaiting review

### Pull Request

https://github.com/ggml-org/llama.cpp/pull/25947

### Summary of Contribution

Fixed a crash converting HF-layout tekken models that have no `tokenizer.json`: a one-line change in `conversion/llama.py` adding the missing `return` to the tekken branch of `set_vocab`, so conversion stops falling through into the sentencepiece/llama_hf/gpt2 paths after the mistral vocab is already written.

### Feedback / Next Steps

PR submitted upstream to llama.cpp; awaiting maintainer review. Will respond to any requested changes.

**Update (2026-07-25) — staying in Phase IV:** Checks are passing and there are no conflicts. Just waiting on maintainer review.

