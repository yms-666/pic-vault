# Maker Not Screenshot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop classifying images with camera Make (or archived whitelist source in the name) as screenshots, and migrate existing misclassified files out of `screenshots/`.

> Related follow-up (2026-08-02): run-stream stall fix is documented in `docs/superpowers/specs/2026-08-02-run-stream-stall-design.md`.

**Architecture:** Extend `classify_capture` with a maker-evidence gate before screenshot keywords/GPS; add helpers to parse archived source tokens; add `--fix-maker-screenshots` that reclassifies matching files via existing `reclassify_paths`.

**Tech Stack:** Python 3, PIL EXIF, existing PicVault test runners (`run_tests.py`, `test_bugbot_fixes.py`).

---

### Task 1: Failing unit tests

**Files:**
- Modify: `scripts/tests/run_tests.py` (Make without GPS helper + classify case)
- Modify: `scripts/tests/test_bugbot_fixes.py` (classify + fix-maker migration)

- [x] Write tests for: Make+no GPS → normal; screenshot keyword + Make → normal; archived `_sony_` name → maker evidence; fix command moves stock file to by-date

### Task 2: Helpers + classify_capture v8

**Files:**
- Modify: `scripts/rename_organize.py`

- [x] Add `archived_stem_name`, `source_from_archived_filename`, `has_maker_evidence`
- [x] Insert maker gate in `classify_capture` after camera-filename check
- [x] Update module docstring / classification comment to v8

### Task 3: Stock fix CLI

**Files:**
- Modify: `scripts/rename_organize.py` (`main`, new `fix_maker_screenshots`)

- [x] Implement scan + `reclassify_paths(..., 'to_normal')`
- [x] Wire `--fix-maker-screenshots`

### Task 4: Docs + verify

**Files:**
- Modify: `outputs/PLAN.md` classification section → v8

- [x] Update PLAN.md image rules
- [x] Run `python3 scripts/tests/run_tests.py` and `python3 scripts/tests/test_bugbot_fixes.py`
