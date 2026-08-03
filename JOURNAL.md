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

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
All 5 sub-tasks from `PLAN.md` are implemented and committed on
`fix/150-tech-detector-vendored-files`:
1. Normalized path separators (`\` → `/`) in `_should_skip_file`.
2. Rewrote matching to check whole path segments (via a `SKIP_DIRECTORIES`
   set) instead of leading-slash substrings — this is the actual fix for
   #150, since it naturally handles root-level, nested, and Windows-path
   cases without special-casing any of them.
3. Broadened the skip list (`target`, `.next`, `.nuxt`, `.pytest_cache`,
   `.mypy_cache`, `.ruff_cache`, `.tox`, `.eggs`) plus a `*.egg-info` suffix
   check, since that pattern doesn't fit the exact-segment-match model.
4. Fixed `test_vendor_files_excluded` (previously had zero assertions) and
   7 other dead tests discovered along the way (see below) — all in
   `tests/unit/test_tech_detector.py`.
5. Added 5 new tests covering the specific edge cases from `PLAN.md`:
   nested vendored dirs (regression guard), Windows backslash paths (root
   + nested), the `rebuild/` substring-false-positive case, `*.egg-info`,
   and a file literally named `build` with no extension.

**Unplanned but necessary side-work:** the test file already failed
`ruff`/`mypy` pre-commit hooks on ~30 pre-existing issues (no return-type
annotations on any test method, 8 dead tests with unused variables and no
real assertions) — `disallow_untyped_defs = true` applies file-wide, so I
couldn't land any commit touching this file without also fixing those.
Added `-> None` / `TechDetector` annotations to all 27 test methods, and
gave each of the 8 previously-assertion-less tests a real assertion based
on the tool's actual (verified, not assumed) behavior. One of those —
`test_case_insensitive_extension_matching` — documents a genuine separate
bug (extension matching is case-sensitive) via `xfail(strict=True)` rather
than either fixing it (out of scope) or asserting the current broken
behavior as if it were correct.

Also confirmed the full `pytest tests/unit -m unit` failure set is
byte-for-byte identical before and after my change (51 pre-existing
failures, unrelated modules — confirmed via diffing sorted `FAILED` lines
from both runs), except for the two tests my fix makes pass. Full-repo
`make lint` (173 pre-existing errors) and `black --check .` (51 files) are
also pre-existing and untouched by this branch — my two changed files pass
`ruff`, `black`, and `mypy` individually.

**Next steps:**
Open a draft PR against `ascherj/pathreview` for peer/mentor feedback,
fill out the PR template (documenting the pre-existing failures above),
then mark ready for review once any feedback is addressed.

**Blockers:**
None currently — the pre-existing test-file lint debt turned out to be
resolvable within scope rather than a true blocker, since it only required
mechanical type annotations and restoring real assertions to already-named
tests (not new test-writing scope creep).

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/645 (currently draft — opened for peer/mentor feedback per this week's process; will mark ready for review once feedback is addressed)

**Branch:** `fix/150-tech-detector-vendored-files`

**What you built:**
Fixed `TechDetector._should_skip_file` (`agent/tools/tech_detector.py`) so
vendored/build directories are excluded from language detection regardless
of whether they're at the repo root, nested, or use Windows backslash
paths — replacing leading-slash substring matching with normalized,
segment-based directory matching, and broadening the skip list.

**Tests added or updated:**
`tests/unit/test_tech_detector.py` — fixed 8 pre-existing dead tests
(added real assertions in place of unused-variable placeholders, verified
each assertion against actual tool output rather than the test's original
aspirational comment), added 5 new tests for the specific edge cases named
in `PLAN.md`'s Edge Cases section, and added type annotations across all
27 test methods to satisfy the repo's `disallow_untyped_defs` mypy setting.

**Self-review confirmation:** [x] make check passes (on changed files;
pre-existing repo-wide lint/format debt documented in PR description and
confirmed unrelated) [x] make test-unit passes (pre-existing unrelated
failures documented; zero new failures introduced, verified by diffing
failure sets before/after)

**Draft PR feedback received from:** (pending — PR opened as draft for
peer/mentor review per this week's process)
