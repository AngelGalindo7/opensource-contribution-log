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

## Environment Setup

**Branch:** `feat/initcap-ucase-lcase` in my clone of my fork (`AngelGalindo7/Daft`).

**Setup approach:** README instructions. For Phase II I installed the released wheel (`pip install daft`) into a fresh venv instead of building from source — the bug is that the functions are *missing*, so the released package proves that on its own.

**Challenges encountered:**

- Daft's core is Rust, so a source build means a Rust toolchain plus the `make build` workflow. That isn't needed to prove the functions are missing, so I reproduced against the released wheel and deferred the source build to Phase III, when I actually need to compile changes.
- The function machinery spans three layers (Rust UDFs, Python wrappers, and the SQL registry), and it wasn't obvious at first where a "missing function" actually lives. Resolved by reading PR #7070, which added six sibling functions from this same issue and touches every layer I'll need.

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

### Expected vs. Actual Behavior

- **Expected:** `ucase`, `lcase`, and `initcap` are importable from `daft.functions` and callable from Daft SQL, like their siblings `upper` and `lower` — this is what issue #3792 asks for, for PySpark parity.
- **Actual:** all three are missing from the Python API, and Daft SQL raises `Unsupported SQL: 'Function `ucase` not found'` (same for the other two), while `upper` and `lower` work normally.

### Files and Functions Involved

- `src/daft-functions-utf8/src/` — one file per string function (`upper.rs`, `lower.rs`, `capitalize.rs`), each built from the same `ScalarUDF` template; the new functions belong here.
- `src/daft-functions-utf8/src/lib.rs` — where UDFs are registered, and why SQL can't find these names today.
- `daft/functions/str.py` and `daft/functions/__init__.py` — the Python wrappers and exports that make functions importable from `daft.functions`.

---

## Reproduction Evidence

- The reproduction script `repro.py` is committed locally on the branch `feat/initcap-ucase-lcase` in my local clone of my Daft fork (`AngelGalindo7/Daft`), at `C:\Users\agali\Documents\Daft`. It is not pushed to GitHub yet.
- Tested on Windows 11, Python 3.14, Daft 0.7.15.

---

## Implementation Plan

**Understand.** Issue #3792 asks for PySpark-parity string functions. `ucase`/`lcase` are the Spark/SQL names for uppercase/lowercase; `initcap` uppercases the first letter of each word. Root cause of the gap (not just the symptom of "function not found" errors): these names were never implemented or registered in the `daft-functions-utf8` crate, so neither the Python API nor the SQL registry can resolve them. There's no partial or broken version to repair — the fix is additive.

**Match.** Daft already has the exact pattern I need: `upper`, `lower`, and `capitalize` in `src/daft-functions-utf8/src/` are one-file-per-function `ScalarUDF`s built from the same template. `capitalize.rs` is the closest analogue for `initcap` — same shape, just first-letter-of-string instead of first-letter-of-each-word. PR #7070, which added six sibling functions from this same issue, is my reference for every layer that has to change.

**Plan.**

- **`ucase` / `lcase`:** aliases for `upper` / `lower`. Add UDFs named `"ucase"` and `"lcase"` that reuse the same uppercase/lowercase logic.
- **`initcap`:** new file `initcap.rs` based on `capitalize.rs`, but uppercasing the first letter of every word instead of just the first letter of the string.
- **Register** the three functions in `src/daft-functions-utf8/src/lib.rs`.
- **Python API:** add wrappers in `daft/functions/str.py` and export them in `daft/functions/__init__.py`.
- **Tests:** add coverage in `tests/expressions/test_utf8.py` and `tests/series/test_utf8_ops.py`.

**Review.** Edge cases to handle before calling it done: empty strings, multi-word strings, multiple spaces or punctuation between words, and null values. I also checked whether the `heck` crate's `to_title_case` could implement `initcap` — it can't, because it splits on case and punctuation, which rewrites the string instead of just changing casing.

**Evaluate.** Success is the Phase II repro script flipping from `missing` to `found` for all three names, the SQL calls resolving, and the new tests passing. Writing the code needs a source build (`make build`), which I'll set up in Phase III.


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

### Environment Setup

**Branch:** https://github.com/AngelGalindo7/FreeCAD/tree/ifc-import-default-representation

**Setup approach:** README/release instructions. The bug lives in FreeCAD's Python BIM module, so I worked against the installed FreeCAD 1.1.1 release rather than doing a source build — the Python files run as-is in the release, so they can be inspected and tested in place.

**Challenges:** FreeCAD's full source build is a heavy C++/Qt toolchain, which would have been a lot of setup for a one-line Python fix. Resolved by confirming the fix is entirely in `ifc_import.py` (Python), which the installed release executes directly, so no compile was needed to reproduce or test the change.

### Reproduction Process

**Steps to Reproduce**

1. Download and open the latest FreeCAD release.
2. Switch to the BIM workbench.
3. Open (import) any `.ifc` file.
4. In the import dialogue, the "Representation type" dropdown shows `Load the shape (slower)` as the default — even though `Load 3D representation only, no shape` is the one marked `(default)`.

**Expected vs. Actual**

- **Expected:** the import dialogue opens with the option labeled `(default)` selected — `Load 3D representation only, no shape (default)`.
- **Actual:** it opens with `Load the shape (slower)` selected, contradicting the `(default)` label.

**Files and Functions Involved**

- `ifc_import.py` — `get_options()` reads the fallback with `PARAMS.GetInt("ShapeMode", 0)`; that `0` is the out-of-step default.
- The dialogue's `.ui` files and the Native IFC preferences page — both already default `ShapeMode` to `1`.

### Reproduction Evidence

Branch in my fork: https://github.com/AngelGalindo7/FreeCAD/tree/ifc-import-default-representation

### Implementation Plan

**Understand.** The dropdown's `(default)` label and the code's actual behavior disagree. Root cause (not just the visible symptom): the `.ui` files and the Native IFC preferences page define `ShapeMode = 1` as the default, but `get_options()` in `ifc_import.py` falls back to `0` when the parameter has never been saved — the code fallback was the one place out of step with every other definition of the default.

**Match.** The `.ui` files and the preferences page are the in-repo precedent: they already encode `1` as the intended default, so the fix imitates them instead of inventing new behavior.

**Plan.** Two approaches:

- **A:** change the code's fallback default to `1` (3D representation only), so it matches the `(default)` label.
- **B:** keep the code on `0` (Load the shape) and move the `(default)` label to that option instead.

Going with **A**, because the `.ui` files and the Native IFC preferences page already default to `1`. Fix: change `PARAMS.GetInt("ShapeMode", 0)` to `PARAMS.GetInt("ShapeMode", 1)` in `ifc_import.py`.

**Review.** Edge case: users who have already picked an option have `ShapeMode` saved in their parameters, so the fallback never fires for them — the change only affects first runs where the parameter is unset, which is exactly the situation the bug report describes. No other call sites read this fallback.

**Evaluate.** Success is a before/after manual check: after the change, the dialogue opens on the `(default)`-labeled option (verified in Phase III).

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

### Environment Setup

**Branch:** https://github.com/AngelGalindo7/llama.cpp/tree/fix/convert-tekken-vocab

**Setup approach:** README instructions — cloned llama.cpp and pip-installed the conversion requirements from the `requirements/` folder. No C++ build and no GPU are needed: the bug is in the pure-Python conversion script, which runs on CPU.

**Challenges encountered:**

- The repro model (Voxtral-Mini-3B-2507) ships its weights twice — sharded HF files plus a `consolidated.safetensors` copy. With 14GB RAM and limited disk, I excluded the consolidated file from the `hf download` (`--exclude consolidated.safetensors`); the converter only reads the HF shards, so this halves the download without changing the repro.
- I have no NVIDIA GPU, so before claiming the issue I confirmed the conversion path is CPU-only — it is, so the missing GPU never blocked reproduction.

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

**Expected vs. Actual**

- **Expected:** `convert_hf_to_gguf.py` detects `tekken.json`, writes the vocab through the mistral/tekken path, and finishes the conversion with a valid GGUF file.
- **Actual:** the tekken vocab is written successfully, but the run keeps going through the sentencepiece → llama_hf → gpt2 fallback ladder and crashes on the last one with `AttributeError: 'MistralCommonTokenizer' object has no attribute 'vocab'`.

**Files and Functions Involved**

- `conversion/llama.py` — `LlamaModel.set_vocab`: the tekken branch calls `self._set_vocab_mistral()` without `return`, so execution falls through into the remaining tokenizer paths.
- `_set_vocab_gpt2` — where the fall-through actually crashes: it expects a HF tokenizer with a `.vocab`, not the `MistralCommonTokenizer` wrapper.

### Reproduction Evidence

Full before/after tracebacks are in the PR description. Tested on Windows 11, no GPU needed.

### Implementation Plan

**Understand.** Root cause, not just the symptom: the `AttributeError` in `_set_vocab_gpt2` is downstream damage. The actual defect is a missing `return` in the tekken branch of `LlamaModel.set_vocab` in `conversion/llama.py` — a successful mistral vocab write still falls through into the remaining tokenizer paths, and the last one crashes on the `MistralCommonTokenizer` wrapper.

**Match.** The `is_mistral_format` branch directly above the tekken branch handles the identical "vocab already written, stop here" situation and does `return` — the fix makes the tekken branch match its sibling.

**Plan.** One-line change in the tekken branch: `self._set_vocab_mistral()` becomes `return self._set_vocab_mistral()`.

**Review.** Edge cases: models without `tekken.json` never enter this branch, so they're unaffected; the explicit `--mistral-format` path already returns, so its behavior doesn't change. The riskiest failure mode — silently skipping a vocab path some model still needs — doesn't apply, because the branch only fires after the mistral vocab has been fully written.

**Evaluate.** Success is the same conversion command completing and producing a loadable GGUF instead of crashing (verified in Phase III: `voxtral.gguf`, 273 tensors, `Model successfully exported`).

### Investigative Depth (git history)

I dated the bug through git history rather than guessing: the `return` was there when the tekken path was added in #14862, was dropped by the refactor in #14737 that renamed `set_vocab_tekken()` to `_set_vocab_mistral()`, and #17114 carried the broken behavior over when it split the converter into the `conversion/` package. The fix restores the original #14862 behavior.

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

