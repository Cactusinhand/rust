filter-repo-rs Scope and Priorities
===================================

Purpose
-------

- Make explicit what we do and don’t do, to keep the tool focused and usable.
- Capture trade‑offs so roadmap and PR reviews have a stable reference.

High‑Value (Prioritized) Features
---------------------------------

- Sensitive data scrubbing (files + messages)
  - `--replace-text` across history (bytes/regex), combinable with path filters/renames/size thresholds; with `--sensitive` to cover all refs.
- Historical repository slimming
  - `--max-blob-size`, `--strip-blobs-with-ids` to drop large objects and remove referencing paths; write a report for verification.
- Path restructuring (monorepo split, root move, bulk renames)
  - `--subdirectory-filter`, `--to-subdirectory-filter`, `--path-rename` plus `--path/--path-glob/--path-regex/--invert-paths`.
- Auto‑rewrite of old commit hashes in messages
  - Use `commit-map` to rewrite short/long hashes, avoiding broken references.
- Consistent tag/branch renames
  - Handle annotated/lightweight tags in order, dedupe properly, and emit `ref-map` to reduce manual mistakes.
- Empty commit pruning while preserving merges
  - Alias non‑merge empty commits to first parent; keep merges and de‑duplicate parents.
- Atomic updates and audit artifacts
  - Batch `git update-ref --stdin`; always write `commit-map` and `ref-map` for migration parity.
- Sensitive mode across all refs
  - Optionally fetch every ref namespace (not just branches/tags), reducing missed leaks; keep `origin` if needed.
- Verifiable dry‑runs
  - `--dry-run` saves both original and filtered fast‑export streams for human/script diffs.
- Windows path compatibility across history
  - Sanitize reserved characters and trailing dot/space; C‑style quoting.
- Analysis (human/JSON)
  - Footprint, Top objects, hot directories, longest paths, duplicate blobs, parent counts, etc.

Why Raw Git Makes This Hard
---------------------------

- Realistically requires combining `fast-export/import`, `rev-list`, `cat-file`, shell, and multiple traversals; error‑prone and slow. `filter-branch` is deprecated.
- No native support for “auto‑rewrite hashes in messages”, producing `commit-map`/`ref-map`, coordinated annotated/lightweight tag handling, or Windows path mass fixes.
- Covering “all refs” (beyond branches/tags) and keeping updates consistent is high‑risk and tedious by hand.

Common Pain Points → Mapped Capabilities
----------------------------------------

- Secret/token leakage: `--replace-text` (incl. regex) + `--sensitive` + reports/dry‑run/maps.
- Monorepo split/path reshaping: subdirectory extraction, root move, bulk renames with consistent results.
- Repo slimming: threshold or allowlist delete of large objects, with samples and counts for stakeholder sign‑off.
- History cleanup: batch tag/branch renames, empty‑commit pruning, preserved merges, sane HEAD.

Low‑Priority / Non‑Core (both ecosystems)
-----------------------------------------

- Callback framework (filename/refname/blob/commit/tag/reset/message/name/email)
  - Explicit non‑goal here; cover common needs via clear CLI flags.
- Incremental/replace‑refs stack
  - `--state-branch`, “already ran” state, stash rewriting, multiple `--replace-refs` strategies.
- LFS orphan checks and SDR extras
  - Orphan detection, `first-changed-commits`/`changed-refs`, long “next steps” docs.
- Fine‑grained encoding/hash toggles
  - `--preserve-commit-hashes`, `--preserve-commit-encoding`, etc.
- Convenience path/rename extensions
  - `--use-base-name`, regex‑based rename rules.
- Rare flags/inputs
  - `--stdin`, `--date-order`, `--no-quotepath`, `--no-mark-tags`, `--no-gc`, etc.
- Rare preflight blockers
  - Case‑insensitive/Unicode‑normalization ref collisions, stash presence, reflog cardinality.

Python‑specific Items We Intentionally Don’t Match (for now)
------------------------------------------------------------

- Complex `--replace-refs` variants and cross‑run state management: high cognitive/ops cost, misfire risk.
- Fast‑export literal passthrough and extreme input tolerance: low coverage, large complexity.
- Exhaustive “suboptimal issues” reports: better left to auditing scripts/tools.

Defer / Not Implementing Now
----------------------------

- Regex in `--replace-message` (commit/tag messages): literal + hash rewrite covers the bulk; regex carries mis‑replace risk.
- `--paths-from-file`: convenience, lower priority than correctness/consistency.
- Windows path policy variants (sanitize/skip/error): default sanitize works; modes add surface area.
- Mailmap identity rewriting: valuable, but not MVP; revisit with clear demand.

Boundaries (Converged Scope)
----------------------------

- Keep core:
  - Path filters/renames (prefix/subdir), blob redaction (incl. regex for files), large‑object removal,
    tag/branch prefix renames, empty‑commit pruning (preserve merges), message hash rewrite, atomic ref updates,
    dry‑run comparability, sensitive mode (all refs), Windows compatibility, analysis.
- Explicit non‑goals:
  - Callback framework, incremental/state‑branch, advanced replace‑refs strategies, LFS orphan/SDR extras,
    encoding/hash preservation toggles, regex path renames, stdin pipeline, excessive preflight micro‑flags.
- Re‑evaluate only with clear repeated demand:
  - `--paths-from-file`, message regex, mailmap identity rewriting.

Maintenance
-----------

- This document is the living “trade‑off ledger”. Update it with new/removed features, priority shifts, and context.
- Related docs:
  - PARITY.md (parity/safety notes vs Python)
  - STATUS.md (current status, limitations, MVP)

