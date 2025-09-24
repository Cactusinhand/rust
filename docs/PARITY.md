Parity With Python git-filter-repo (and Safety Notes)
====================================================

Purpose
-------

- Track parity status with the Python implementation and document decisions.
- Capture safety measures and known differences, so contributors have a single, evolving reference.

What’s Implemented (Parity or Equivalent)
----------------------------------------

- Streaming pipeline and artifacts
  - `git fast-export` → in-process filters → `git fast-import`, with debug copies at `.git/filter-repo/fast-export.{original,filtered}`.
  - `fast-import` runs with `-c core.ignorecase=false` and exports marks to `.git/filter-repo/target-marks`.

- Path selection & rewriting
  - Include by `--path` prefix, `--path-glob` (`*`, `?`, `**`), and `--path-regex`.
  - `--invert-paths`, `--path-rename OLD:NEW`, helpers `--subdirectory-filter` and `--to-subdirectory-filter`.

- Blob filtering & redaction
  - `--replace-text FILE` supports mixed literal and `regex:` rules in one file.
  - `--max-blob-size BYTES` to drop large blobs and remove referencing paths.
  - `--strip-blobs-with-ids FILE` to drop listed 40-hex blob IDs.

- Commits, tags, and refs
  - `--replace-message FILE` (literal) and automatic rewriting of short/long commit hashes in messages using `commit-map`.
  - Tag/branch prefix renames: `--tag-rename`, `--branch-rename`; annotated tags are deduped, lightweight tags buffered and flushed correctly.
  - Non-merge empty commits are pruned via `alias` to first parent; merges are preserved with parent de-duplication.
  - Atomic ref updates via `git update-ref --stdin`, writing `ref-map` and `commit-map` (pruned commits map to all-zeros).

- Modes & behavior
  - `--dry-run` (no ref updates/reset/gc) with full debug artifacts.
  - `--partial` (skip remote migration/removal), `--sensitive` (optionally fetch all refs; keep `origin`), `--no-fetch`.
  - Optional preflight `--enforce-sanity` and pre-rewrite `--backup` (bundle of selected refs).
  - Optional cleanup `--cleanup [none|standard|aggressive]` → reflog expire + `git gc --prune=now`.

- Analyze mode
  - `--analyze` (human) and `--analyze --analyze-json` (machine) for repository metrics and warnings.

- Windows compatibility
  - Path-byte sanitization for reserved characters and trailing dot/space trimming, C-style quoting, and case-sensitive import.

What’s Missing or Different (vs Python)
--------------------------------------

- Identity rewriting
  - Not yet implemented: mailmap-based identity rewriting.

- Replace-refs & incremental filtering
  - Not yet implemented: `--replace-refs` strategies, `--state-branch`, cross-run “already ran” state, and stash (`refs/stash`) rewriting.

- Sensitive-data specific extras
  - Not yet implemented: LFS orphaning checks, SDR metadata like `first-changed-commits`/`changed-refs`, and post-rewrite “Next steps” guidance.

- Merge pruning strategy
  - Current behavior preserves merges (with parent de-duplication); full “degenerate merge pruning with ancestry guarantees” not yet implemented.

- CLI differences
  - Not yet implemented: `--paths-from-file`, `--use-base-name`, and regex-based path rename matching.
  - `--replace-message` supports literal rules; `regex:` for messages and `--preserve-commit-hashes` toggle are planned.

Safety Measures (Current)
-------------------------

- Preflight (`--enforce-sanity`) blocks risky runs unless `--force`:
  - Freshly packed or empty repos; exactly one `origin` remote or no remotes; clean/staged/unstaged/untracked checks; single worktree.
  - Note: we currently do not block on case-insensitive or Unicode-normalization ref collisions, stash existence, or reflog cardinality.

- Sensitive & partial modes
  - `--sensitive` can fetch all refs to ensure coverage (unless `--no-fetch`), and does not remove `origin` after run; `--partial` skips remote migration/removal entirely.

- Recovery & auditability
  - `--backup` (bundle) before rewriting; `.git/filter-repo/fast-export.*`, `commit-map`, `ref-map`, and optional `report.txt` facilitate auditing.

- Atomic & conservative updates
  - Refs are updated/deleted via a single `git update-ref --stdin` batch after the new refs are known to exist; HEAD is adjusted safely.

Documentation Changes (2025-09-20)
----------------------------------

- Fixed a README typo: `.git/filer-repo/target-marks` → `.git/filter-repo/target-marks`.
- Added this parity & safety notes document to centralize design rationale and gaps.

Non-goals
---------

- Callback framework (filename/refname/blob/commit/tag/reset/message/name/email): not planned for this project. We prefer explicit CLI flags and focused features instead of embedding a general callback API layer.

Next Steps (Roadmap Excerpts)
-----------------------------

- Mailmap-based identity rewriting.
- `--replace-refs …`, `--state-branch`, and cross-run state/stash handling.
- LFS orphaning checks and SDR metadata/reporting.
- Merge pruning for degenerate merges with ancestry guarantees.
- CLI: `--paths-from-file`, `--use-base-name`, regex-based path rename; human-readable sizes; `regex:` in `--replace-message`; `--preserve-commit-hashes`.

Notes & Process
---------------

- Keep this file updated as features land or decisions change. Prefer concise bullets with links to code when helpful.
