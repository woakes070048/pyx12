# pyx12

HIPAA X12 EDI document validator and converter. Parses ANSI X12N data files and validates against HIPAA Implementation Guidelines. Generates 997 (4010) / 999 (5010) acknowledgements.

## Project
- Python 3.11+, BSD
- Active branch: `dev/py3-modernize` — modernizing Python 2→3 patterns
- Tests live in `pyx12/test/`, use `unittest.TestCase` style
- Run tests: `.venv/Scripts/python.exe -m pytest pyx12/test/`
- Line length: 100 (ruff format), imports sorted via `ruff check --select I --fix`

## Tech stack
- **Language:** Python 3.11+ (CI matrix: 3.11, 3.12, 3.13, 3.14)
- **Runtime deps:** `defusedxml>=0.7` (only)
- **Build backend:** setuptools (>=61) via `pyproject.toml`; no `setup.py`
- **Package manager:** `uv` — install Python packages with `uv pip install ...`, not pip
- **Local environment:** `.venv/` at project root. Always invoke tools via `.venv/Scripts/python.exe -m <tool>` — never `source .venv/Scripts/activate`, since shell state does not persist across Bash tool calls
- **Test runner:** pytest (+ pytest-cov for coverage); tests written in `unittest.TestCase` style
- **Format / lint:** ruff (line length 100); `ruff format` + `ruff check --select I` for imports
- **CI:** GitHub Actions — `.github/workflows/main.yml` runs the matrix; `release.yml` and `publish-to-test-pypi.yml` handle PyPI
- **XML parsing:** `defusedxml.ElementTree` (never `xml.etree.ElementTree` directly — DTD/entity DoS protection)
- **Package data:** XML maps + DTDs/XSDs in `pyx12/map/` are loaded via `importlib.resources`

## Coding style
- No comments unless the WHY is non-obvious (hidden constraint, subtle invariant, workaround)
- No unnecessary abstractions — match scope to the task; three similar lines beats a premature helper
- No error handling for scenarios that can't happen; validate only at system boundaries
- No backwards-compatibility shims for removed code
- No docstrings beyond a single short line when truly needed
- Prefer functional programming patterns over object oriented patterns

## Response style
- Trailing summaries and recaps of what was done are welcome
- Some emojis
- Reference code locations as `file:line` (or markdown links in VSCode)
- Short updates at key moments; silent is not acceptable

## Testing
See the [/test](.claude/commands/test.md) skill for invocation, reporting, and authoring conventions.

## Preferences
- Save preferences to this CLAUDE.md file so they persist across sessions
- No pickle / opaque deserialization for shipped or loaded artifacts — prefer generated `.py` modules (security: `__reduce__` is an arbitrary-code-execution surface unacceptable in a healthcare-data library)
- Every production release tag (`vN.N.N`) needs a matching `gh release create --latest` after the PyPI upload step
- Keep unreachable safety-net `return` statements in walker `while True` loops — they document intent and protect against future control-flow changes
- Prefer small, single-concern PRs. Refactors with broad test impact (>20 sites) get split: introduce the new shape with the old shape kept as a compat wrapper first, then migrate callers / tests / delete the wrapper in follow-up PRs
- Before introducing a `Protocol`, abstract base, or new wrapper type, check whether existing inheritance / concrete types already provide the contract — usually they do, and a parallel abstraction just shadows the existing structure
- When migrating tests off a last-error-wins stub (e.g. `errh_null`) reveals extra errors the stub had been hiding, tighten the input fixture to exercise only the scenario named in the test rather than rebadging the assertion to match the full error list
- Use "error" terminology consistently (`err_handler`, `seg_error`, `ErrorItem`); reuse the `ErrorItem` / `EleError` / `SegError` hierarchy for new validator return types instead of inventing "Finding" / "Issue" / etc. names
- IK4/AK4 error codes follow the X12 spec: `"1"` = required data element missing, `"2"` = conditional required missing, etc. When historical pyx12 emission disagrees with the spec (e.g. missing required composite was emitting `"2"` but the spec says `"1"`), fix the validator
- Preserve Sphinx `:param` / `:type` / `:return` / `:rtype` blocks when refactoring methods that have them; drop only the entries for dropped parameters
- Releases go through `.github/workflows/release.yml` on `v*` tag push (PyPA trusted publishing, no local credentials). Don't run `twine upload` locally — it duplicates the workflow and needs `~/.pypirc`. Use the `/publish` skill, which drives the tag + watch + GitHub-release side
- RC validation loop: every RC must be exercised against production-like traffic (real X12 files via the downstream worker) before promoting to final. If a regression surfaces, file a fix PR and cut a new RC — never re-tag a published version
- Stale remote branch is deletable when (i) its content can't merge against current master AND (ii) its idea or bug fix is preserved elsewhere (architectural-review plan, already-fixed bug on master, etc.). Keep when it carries unique data not preserved elsewhere (e.g. XML map files for transactions not in master)
- For end-to-end regression tests of X12 input → output behavior, add a key to `pyx12/test/x12testdata.py` with `source` + `resAck` / `res997` / `resJson` and use the existing `_test_999` / `_test_997` / `_test_json` helpers. Reserve ad-hoc tempfile-based tests for paths the fixture pattern can't reach (e.g. the file-open encoding path that needs a real filename, not a `StringIO`)
- When sanitizing non-ASCII characters into X12 ASCII output (IK4-04 / AK4-04 / any AK/IK field), substitute `?` (0x3F). It's in the X12 basic charset and valid everywhere; use `pyx12.error_visitor.ascii_only(value)` rather than reinventing per-site
- When an issue body or PR description quantifies expected test impact ("N tests will fail"), run the change locally to verify the count is current before scoping the plan around it. Such claims drift after refactors — issue #107 claimed 47 failures, actual was 0 once the producer-side errors-as-data refactor reshaped the test surface
- After running `ruff check --fix` or `ruff format` against a wide selection, check `git diff --stat` before staging and revert files outside the current PR's scope. Auto-cleanup of unrelated files (dead-comment removal, import sorting) inflates PR diffs and obscures the focused change
- For non-obvious technical choices in a fix (e.g. encoding=latin-1 vs utf-8, sanitize at output vs at input), explain the tradeoff before committing. Surface the decision so the user can redirect; don't bake it into a fait-accompli commit
- The `pypi` GitHub environment's `wait_timer` is 5 minutes (set 2026-05-05; was 15). The `required_reviewers` rule still gates on azoner approval. Adjust via `gh api repos/azoner/pyx12/environments/pypi -X PUT -F wait_timer=N ...`