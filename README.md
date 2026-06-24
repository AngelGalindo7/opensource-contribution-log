# Contribution #1: Implement lcase, ucase, and initcap string functions

**Contribution Number:** 1
**Student:** Angel Galindo
**Issue:** https://github.com/Eventual-Inc/Daft/issues/3792
**Status:** Phase I Complete

---

## Why I Chose This Issue

Daft is a distributed dataframe library for processing large-scale and multimodal data. I want to work on it because I'm drawn to data infrastructure and want to learn how a tool like this is built.

This issue asks Daft to add the `lcase`, `ucase`, and `initcap` string functions — common SQL-style helpers for lowercasing, uppercasing, and title-casing text. It matters because these are widely-expected string operations that users coming from SQL or other dataframe libraries will reach for, and their absence is a gap in Daft's expression API.

I specifically chose `lcase`, `ucase`, and `initcap` because they map closely to Python's own string methods (`str.lower()`, `str.upper()`, and `str.title()`), so the core logic is approachable and Python-focused, making them a well-scoped and easy-to-implement first contribution. There's also strong source material to learn from: Daft already implements similar string functions, so I can follow those existing patterns for how functions are defined, registered, and tested, and reference how other dataframe/SQL libraries implement the same operations.
