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

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/MichaelHP23/pathreview/commit/e26c0c5

**Reproduction summary:**

Root cause is `TechDetector._should_skip_file` in `agent/tools/tech_detector.py`
(lines 143–164). Every skip pattern (`"/node_modules/"`, `"/vendor/"`,
`"/dist/"`, `"/build/"`, etc.) requires a leading `/`, so it only matches a
vendored directory when it's *nested* under something else — a vendored
directory sitting at the repo root, or any path using Windows-style
backslashes, never matches and gets counted as first-party code.

Reproduced two ways:

1. **Existing tests already fail.** Running
   `pytest tests/unit/test_tech_detector.py -v -k "node_modules or vendor_files or build_directory"`
   against the current `main`/branch code gives:
   ```
   FAILED test_node_modules_excluded  — AssertionError: assert 'JavaScript' == 'Python'
   FAILED test_build_directory_excluded — AssertionError: assert 'JavaScript' == 'Python'
   PASSED test_vendor_files_excluded  (passes only because it has no assertion — dead test)
   ```
   Both failing tests use root-level vendored paths
   (`"node_modules/package1/index.js"`, `"build/generated.js"`) with no
   leading slash — exactly the case the issue describes.

2. **Manual repro of the Windows-path case** (not covered by any existing
   test):
   ```python
   from agent.tools.tech_detector import TechDetector
   d = TechDetector()
   d.execute({"files": ["src\\main.py", "node_modules\\package\\index.js", "utils.py"]})
   # -> {'primary_language': 'JavaScript', 'all_languages': ['JavaScript', 'Python'], ...}
   ```
   A single Python file plus one Windows-path `node_modules` file flips
   `primary_language` from `Python` to `JavaScript`.

I considered adding a new failing test directly to `tests/unit/test_tech_detector.py`
to serve as the reproduction artifact, but the file already fails the repo's
`mypy`/`ruff` pre-commit hooks on ~30 pre-existing issues unrelated to this
change (missing return-type annotations on every test method, a few unused
local variables) — `disallow_untyped_defs = true` in `pyproject.toml` applies
repo-wide with no test-file exclusion. Fixing all of that pre-existing debt is
out of scope for a reproduction commit, so I'm documenting the reproduction
here instead (as this section's guidance explicitly allows) and will add the
new/fixed tests as part of the actual fix commit in Week 9, at which point
I'll also add the missing type annotations to whichever test methods I touch
so the hook passes cleanly.

**PLAN.md link:** https://github.com/MichaelHP23/pathreview/blob/fix/150-tech-detector-vendored-files/PLAN.md

**Walkthrough video (recommended):**

**Blockers or open questions:**
- `test_vendor_files_excluded` has no assertion at all — it's a dead test
  that will need a real assertion added alongside the fix, even though the
  issue itself doesn't mention it.
- Need to decide the full replacement skip list content (issue asks to
  "broaden" it) — current plan is `node_modules`, `vendor`, `dist`, `build`,
  `.git`, `__pycache__`, `.venv`, `venv`, plus likely additions like `target`
  (Rust/Java build output) and `.next`/`.nuxt` (JS framework build output) —
  open to narrowing this in review if it's judged too broad.
