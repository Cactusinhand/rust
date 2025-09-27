# git-filter-repo Architecture

This document explains the core flow and internal building blocks of `git-filter-repo` as implemented in the single-file script `git-filter-repo`.

## End-to-End Pipeline
- Parse CLI: `FilteringOptions.parse_args()` builds `args` and pre-processes complex flags (path rules, replacements, callbacks, state-branch).
- Orchestrate: `main()` chooses `RepoAnalyze.run(args)` or `RepoFilter(args).run()`.
- Export: spawn `git fast-export` with flags (`--show-original-ids`, `--reference-excluded-parents`, etc.) and stream its output.
- Filter: `FastExportParser` incrementally parses records into objects, invokes tweak callbacks in `RepoFilter` which apply rules and edits.
- Import: emit modified objects as fast-import commands to `git fast-import` (optionally also to a debug file), then reconcile refs.
- Finalize: write metadata (commit/ref maps, LFS), optionally save marks to a `--state-branch`, repack/cleanup, and print next steps.

## Major Components
- Data model objects
  - `Blob`, `Commit`, `Tag`, `Reset`, `FileChange`, `Progress`, `Checkpoint`.
  - Each implements `dump(file_)` which writes a fast-import stanza; `_GitElementWithId` supplies portable marks and skip/rename semantics.
- Parser: `FastExportParser`
  - Reads fast-export stream line-by-line, using compiled regexes for marks, parents (`from`/`merge`), users, quoted strings, and ref headers.
  - Builds typed objects and invokes registered callbacks; tracks exported vs imported refs and “latest” branch tips for implicit parent handling.
- Orchestrator: `RepoFilter`
  - Wires callbacks, prepares input/output processes, manages ancestry graphs, ID translation, path/content rules, and final ref updates.
  - Exposes `insert()` to programmatically inject objects into the stream.
- Utilities
  - `GitUtils`: wrappers for `rev-parse`, `show-ref`, `diff-tree`, `cat-file`, `count-objects`, etc.
  - `SubprocessWrapper`: normalizes bytes/str arguments (Windows and `PRETEND_UNICODE_ARGS`).

## Identity, Marks, and Graphs
- `_IDs` (global)
  - Assigns new marks for created objects; translates old→new marks; records transitive renames when inserting mid-branch.
  - Integrates with `skip()` to remap references when commits are pruned.
- `AncestryGraph`
  - Maintains a compact DAG keyed by fast-export IDs, with depth and parent lists; records original hashes (`commit.original_id`) and new ones.
  - Supports `is_ancestor()`, `map_to_hash()`, parent-hash lookups, and reverse maps for later hash translations and metadata generation.
- Short-hash rewriting
  - Commit messages are scanned for `[0-9a-f]{7,40}`; `_translate_commit_hash()` maps old prefixes to new hashes, disambiguating when possible.

## Filtering Lifecycle (per object)
- Blob: apply `--replace-text` (literal/regex) via `FileInfoValueHelper.apply_replace_text()` or `blob_callback`; track LFS pointers if enabled.
- Commit:
  - Message: `--replace-message` edits, then optional callback, then hash translation unless `--preserve-commit-hashes`.
  - Author/committer: apply mailmap, then optional `name_callback`/`email_callback`.
  - Files: `_filter_files()` maps and filters `FileChange`s (see below), enforces size/id stripping, de-duplicates collisions, and re-bases against new first-parent when necessary.
  - Parents: when empty commits are pruned, `_maybe_trim_extra_parents()` removes redundant parents without collapsing merges.
- Tag/Reset: apply tag renames (`--tag-rename`) and `refname_callback`; update imported refs set.

## Path Filtering Semantics
- Input: path rules from flags (`--path`, `--path-glob`, `--path-regex`, `--path-rename*`, `--to-subdirectory-filter`, `--use-base-name`, `--inclusive`).
- Mechanism: `newname()` computes new file names and inclusion; `_filter_files()` builds a new filename→`FileChange` map.
- Collisions: safe elision when a delete collides with a modify or when two modifies are identical; otherwise raise an error.
- Size/ID gates: drop changes exceeding `--max-blob-size` or listed in `--strip-blobs-with-ids`.

## Stream Setup and Flags
- Fast-export (`_setup_input()`):
  - Always: `--show-original-ids`, `--signed-tags=strip`, `--tag-of-filtered-object=rewrite`, `--fake-missing-tagger`, `--reference-excluded-parents`.
  - Optional: `--no-data` when blobs are not needed; `--date-order`; `--use-done-feature` for end-of-stream coordination.
  - Marks: `--import-marks/--export-marks` to a local file when `--state-branch` is used.
- Fast-import (`_setup_output()`):
  - Always: `--force --quiet`, `-c core.ignorecase=false`; `--date-format=raw-permissive` to accept non-strict dates.
  - Marks mirroring when `--state-branch` is enabled; support dual-writer for debug streams.
- Debug/dry-run: original and filtered streams are saved under `.git/filter-repo/fast-export.{original,filtered}`.

## Refs, Replace-Refs, and HEAD
- Exported/imported refs are tracked by the parser; after import, `_ref_update()` deletes refs not re-imported (excluding non-refs), updates `HEAD`, and applies `refs/replace/*` policy based on `--replace-refs` (`delete-no-add`, `delete-and-add`, `update-or-add`, `update-and-add`).
- Origin migration: `_migrate_origin_to_heads()` can rewrite `refs/remotes/origin/*` to `refs/heads/*` and removes `origin` (unless SDR/no-fetch), preventing accidental pushes.
- Stash: `_read_stash()` and `_write_stash()` preserve and rewrite reflog entries for `refs/stash`.

## State, Metadata, and Re-runs
- Results dir: `.git/filter-repo/` via `results_tmp_dir()` stores maps, streams, and reports.
- Commit map: old→new commit-hash mapping; accumulated across runs via `_compute_metadata()` to support multi-pass rewrites.
- Ref map: per-ref old/new hashes; includes deleted refs via a sentinel `000...0` value (see code for `deleted_hash`).
- Marks branch: `--state-branch=<name>` commits `source-marks` and `target-marks` blobs to `refs/heads/<name>` using `commit-tree`/`update-ref`, enabling incremental re-runs.

## LFS Handling
- Detection: enabled when previous run recorded LFS, or when `.gitattributes` contains `filter=lfs`.
- Tracking: `LFSObjectTracker` scans small blobs (<1024 bytes) for pointer format (`version`, `oid ...`) using `cat-file --batch-command` via `FileInfoValueHelper`.
- Reporting: compares source vs target LFS object sets; records orphaned objects in metadata and annotates SDR completion hints.

## Performance and Robustness
- Bytes-first API: filenames, messages, and contents handled as bytes; quoting minimized (`PathQuoting.enquote/dequote`).
- Skipping blobs: `--no-data` avoids emitting blob bodies when not needed (e.g., path-only rewrites), while sizes may be precomputed via `cat-file --batch-all-objects`.
- Progress: `ProgressWriter` throttles output; debug output gated behind `--debug`.
- Error handling: consistent `SystemExit` messages for invalid inputs or collisions; safeguards for evil merges when diffs cannot be recomputed.

## Callback Contract
- RepoFilter constructor accepts optional callbacks: `filename`, `message`, `name`, `email`, `refname`, `blob`, `commit`, `tag`, `reset`, `done`, `file_info`.
- `--*-callback` flags can also provide inline Python; `_handle_arg_callbacks()` wraps user code with safe globals via `public_globals`.
- Execution order for commits: replace-message → message_callback → short-hash translation → mailmap → name/email callbacks → path filtering.
- `done_callback` runs after final commands, allowing post-processing of maps/reports.

## Typical Execution Walkthrough
1) `RepoFilter.run()` performs sanity checks; may fetch all refs for SDR, migrates origin branches, and reads stash.
2) `_setup_input()` launches fast-export and sets up backup/`--no-data`/marks as needed.
3) `_setup_output()` launches fast-import (or sets debug file handles) and prepares dual-writing if `--debug`.
4) `FastExportParser.run()` reads: for each `blob/commit/tag/reset`, `RepoFilter` tweaks then inserts into stream; `_IDs` and graphs are updated.
5) On `done`, `_final_commands()` handles ref updates, metadata emission, LFS reports, marks saving, and stash writing.
6) Close streams; optionally repack and clean; print elapsed time and SDR guidance with suggested `git push` commands.

## Notable Data Structures and Globals
- `_SKIPPED_COMMITS`: set of pruned commit IDs (by original/new mark) used to prune ancestry and translate parents.
- `BLOB_HASH_TO_NEW_ID` / `BLOB_NEW_ID_TO_HASH`: correlates blob hashes with assigned marks for consistent `FileChange` emission.
- `deleted_hash`: constant `000...0` sentinel for removed commits/refs.

## Limitations and Trade-offs
- Recomputing merge diffs without a repo checkout is non-trivial; the tool assumes non-evil merges unless both parents and fast-export piping permit a safe recomputation.
- Unicode: by design, most paths/messages are bytes; conversions are explicit to avoid mangling non-UTF8 histories.
- Long-running rewrites rely on stable `git` plumbing behavior; unusual repository states may require additional flags.

This architecture centers on a reliable streaming transform with explicit identity management and graph-aware pruning, enabling safe, scalable history rewrites while remaining a single-file tool that composes with core Git.

## ASCII Streaming Pipeline

```
User/CLI                  RepoFilter                 git fast-export         FastExportParser            git fast-import
   |                         |                               |                      |                              |
   |  parse args             |                               |                      |                              |
   |------------------------>| FilteringOptions.parse_args()  |                      |                              |
   |  run()                  |                               |                      |                              |
   |------------------------>| _run_sanity_checks()           |                      |                              |
   |                         | _migrate_origin_to_heads()     |                      |                              |
   |                         | _setup_input()                 | spawn                |                              |
   |                         |------------------------------->| fast-export          |                              |
   |                         |  (may wrap with                | --stdout-----------> | _advance_currentline()       |
   |                         |   InputFileBackup for debug)   |                      |  parse -> build objects       |
   |                         |                                |                      |  (Blob/Commit/Tag/Reset)      |
   |                         |                                |                      |  invoke RepoFilter._tweak_*   |
   |                         | _setup_output()                |                      |  (path/content/mailmap/etc.) |
   |                         |  (DualFileWriter if --debug)   |                      |--------------dump()---------> |
   |                         |                                |                      |                              |  --stdin
   |                         |                                |                      |                              | (create objects/refs)
   |                         | <--- _import_pipes.stdout -----+                      |                              |
   |                         |  (new commit hashes)                                  |                              |
   |                         |  _flush_renames() updates graphs & maps               |                              |
   |                         |                                                     done                             |
   |                         |---------------------------------------------- RepoFilter._final_commands() ---------->|
   |                         |  - update/delete refs (update-ref)                                                     |
   |                         |  - apply replace-refs policy                                                           |
   |                         |  - write commit-map / ref-map / LFS reports                                            |
   |                         |  - save marks to --state-branch (commit-tree/update-ref)                               |
   |                         |  - rewrite stash reflog (if present)                                                   |
   |                         |  - optional repack/cleanup                                                              |
   |                         v                                                                                         v
Artifacts in .git/filter-repo/:
  - fast-export.original (debug/dry-run)    - fast-export.filtered (debug/dry-run)
  - commit-map, ref-map, original_lfs_objects, orphaned_lfs_objects
  - source-marks / target-marks (also committed to refs/heads/<state-branch>)
```

## Git Plumbing Commands

Streaming (core pipeline)
- `git fast-export [<refs>] --show-original-ids --signed-tags=strip --tag-of-filtered-object=rewrite --fake-missing-tagger --reference-excluded-parents [--no-data] [--use-done-feature] [--date-order] [--import-marks=… --export-marks=…]`
  - Streams source history as objects; options ensure stable identity, safe tag handling, ancestry fidelity, and efficiency when blobs aren’t needed.
  - Code: `git-filter-repo:_setup_input()` (approx L4325–L4369)
- `git fast-import --force --quiet [--date-format=raw-permissive] [--import-marks=… --export-marks=…]`
  - Applies rewritten history to target; permissive date parsing and marks persistence enable incremental reruns.
  - Code: `git-filter-repo:_setup_output()` (approx L4373–L4465)

Refs, marks, and state
- `git show-ref` and `git -C <repo> show-ref <branch>`: enumerate refs and check for state-branch existence.
  - Code: `git-filter-repo:GitUtils.get_refs()` (L1675); `_load_marks_file()` (L4222, L4246)
- `git -C <repo> show <branch>:<path>`: read previous marks blobs from the state branch.
  - Code: `_load_marks_file()` (approx L4233–L4243)
- `git -C <repo> hash-object -w <file>` → `git -C <repo> mktree` → `git -C <repo> commit-tree -m … <tree)` → `git -C <repo> update-ref <branch> <commit>`
  - Stores `source-marks`/`target-marks` as a commit on `refs/heads/<state-branch>`.
  - Code: `_save_marks_files()` (approx L4256–L4277)
- `git update-ref --no-deref --stdin`
  - Atomically delete/update refs (including `HEAD`) and apply `refs/replace/*` policy post-import.
  - Code: `_migrate_origin_to_heads()` (L4409); `_ref_update()` (L4487)

Content, sizes, and diffs
- `git cat-file --batch-all-objects --batch-check=%(objectname) %(objecttype) %(objectsize) %(objectsize:disk)`
  - Precompute blob sizes to enforce `--max-blob-size` and skip heavy blobs when possible.
  - Code: `GitUtils.get_blob_sizes()` (approx L1698–L1736)
  - Rust rewrite: `BlobSizeTracker` streams batch output, caching only blobs above `--max-blob-size` for filtering and reporting.
  - Code: `filter-repo-rs::stream::BlobSizeTracker` (approx L150–L260)
- `git cat-file --batch-command` (lines: `contents <oid>`, `info <oid>`)
  - On-demand content/size for filtering and LFS pointer detection via a persistent subprocess.
  - Code: `FileInfoValueHelper` (approx L2932–L2970)
- `git diff-tree -r <parent> <commit>`
  - Recompute file changes when parentage shifts due to pruning or path filtering.
  - Code: `GitUtils.get_file_changes()` (approx L1719–L1755); `RepoAnalyze` pipeline (L2577–L2640)
- `git rev-list --count [--all|<args>]`
  - Commit count for reporting/guardrails.
  - Code: `GitUtils.get_commit_count()` (approx L1615–L1648)
- `git rev-list --objects --all`
  - Enumerate objects for LFS scanning.
  - Code: `LFSObjectTracker.find_all_lfs_objects_in_repo()` (approx L3053–L3075)

Sanity and guardrails (pre-run)
- `git rev-parse --is-bare-repository` / `git rev-parse --git-dir`: repository layout discovery.
  - Code: `GitUtils.is_repository_bare()` (L1659–L1663); `GitUtils.determine_git_dir()` (L1665–L1672)
- `git diff --staged --quiet` and `git diff --quiet`: ensure a clean working tree.
  - Code: `_run_sanity_checks()` (approx L3497–L3509)
- `git ls-files -o --exclude-standard --directory`: detect untracked files efficiently and honor ignore rules.
  - Code: `_run_sanity_checks()` (approx L3500–L3511)
- `git worktree list`: prevent multi-worktree surprises.
  - Code: `_run_sanity_checks()` (approx L3524–L3529)

Cleanup and hygiene (post-run)
- `git reset --hard [--quiet]`: sync working tree to rewritten refs (non-bare).
  - Code: `RepoFilter.cleanup()` (approx L3530–L3556)
- `git reflog expire --expire=now --all` and `git gc [--quiet] --prune=now`: repack and prune unreachable objects.
  - Code: `RepoFilter.cleanup()` (approx L3538–L3556)
- `git fetch -q --prune --update-head-ok --refmap "" origin +refs/*:refs/*` (for sensitive-data removal)
  - Force-fetch all refs so every remote reference gets rewritten.
  - Code: `_migrate_origin_to_heads()` (approx L4455–L4471)
- `git remote rm origin`: avoid accidental pushes to the original remote after rewrite.
  - Code: `_migrate_origin_to_heads()` (L4474)
