# Repository Guidelines

## Project Structure & Module Organization
- `git-filter-repo`: primary Python entrypoint; installed as `git filter-repo`.
- `Documentation/`: user manual, FAQs, and migration guides.
- `contrib/filter-repo-demos/`: example scripts built on the library.
- `t/`: bash test suite (`t[0-9]*.sh` plus helpers and runners).
- `pyproject.toml`: packaging (setuptools + setuptools_scm).
- `Makefile`: install, test, docs, and release helpers.

## Build, Test, and Development Commands
- Run tool locally: `python3 ./git-filter-repo -h` (within a Git repo).
- Quick tests: `bash t/run_tests`.
- Coverage + HTML report: `make test` or `bash t/run_coverage` (outputs summary and `t/report/`).
- Install (Unix-like): `make install` (copies script into Git exec path and links module).
- Docs (from git clone): `make snag_docs` to fetch prebuilt manpage/html.

Prereqs: Git >= 2.36, Python >= 3.6, bash, and `coverage3` for coverage reporting.

## Coding Style & Naming Conventions
- Python style: follow PEP 8 where practical; use two-space indentation (project standard).
- Encoding: prefer bytes for paths, commit messages, and content; avoid implicit unicode.
- No new runtime dependencies without discussion; keep the single-file script model.
- Keep functions small and focused; descriptive names; avoid large refactors unrelated to the change.

## Testing Guidelines
- Add/modify tests under `t/` following existing patterns (e.g., `t939X-*.sh`).
- Maintain 100% line coverage for `git-filter-repo` core; contrib/tests may be excluded.
- Run `make test` before submitting; include tests for new flags, callbacks, or behaviors.

## Commit & Pull Request Guidelines
- Follow Git’s SubmittingPatches conventions (subject, rationale, incremental commits).
- Subjects: start with `filter-repo: ` and a concise imperative summary.
- PRs: include what/why, minimal repro or example command, and link issues. Small, focused changes are preferred.
- For larger series, consider logical commits per step; avoid noisy formatting-only diffs.

## Agent-Specific Instructions
- Do not reformat untouched code; preserve two-space indents and bytes-centric APIs.
- Place new tests in `t/`; update docs under `Documentation/` when adding flags or behavior.
- Validate locally with `bash t/run_tests` and `make test` before proposing changes.
