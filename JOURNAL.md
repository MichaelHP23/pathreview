# Module 3 Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/150

**Issue title:** Tech detector counts vendored and build-output files, skewing language detection

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The `tech_detector` tool (`agent/tools/tech_detector.py`) scans a repository's
file list to infer which languages and frameworks a project uses. It already
tries to skip vendored and build directories, but every skip pattern requires a
leading slash (e.g. `/node_modules/`), so files in a *top-level* vendored folder
— like `node_modules/react/index.js` — never match and get counted as the
developer's own code. Windows-style paths using backslashes miss the patterns
entirely too. The result is that dependency and build-output code inflates the
detected language/framework list, polluting the review's picture of the
candidate's actual skills. A successful fix normalizes path separators and
matches vendored directories as path segments (including at the repo root),
broadens the skip list, and adds unit tests covering root-level, nested, and
Windows-path vendored files.

**Branch name:** fix/150-tech-detector-vendored-files

**Setup confirmation:** [ ] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

---

### "Is this right for me?" — scope reasoning

- **Scope is bounded to one module.** The change lives in a single file
  (`agent/tools/tech_detector.py`) plus its unit test
  (`tests/unit/test_tech_detector.py`). No cross-module or API-surface changes.
- **No new dependencies.** The fix is pure string/path logic against the
  existing file list — no libraries to add.
- **The bug is understood and reproducible.** Root cause is the leading-slash
  requirement in `_should_skip_file`; a file path like `node_modules/react/x.js`
  demonstrates it immediately.
- **Effort matches Tier 1.** Estimated 2–3 hours: normalize separators, match
  path segments, extend the skip list, and cover it with unit tests.
- **Clear definition of done.** Vendored/build files (root-level and nested,
  forward- and back-slash) are excluded from language detection, proven by tests.
