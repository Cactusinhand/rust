# Changelog

## Unreleased

### CLI convergence and configuration
- Documented how local `.filter-repo-rs.toml` files interact with CLI flags and clarified that explicit arguments take precedence when migrating existing automation.
- Highlighted the FRRS_DEBUG/`--debug-mode` toggle as the single entry point for debug- and cleanup-related options as part of the convergence plan.

### Test coverage and safe defaults
- Added regression tests covering cleanup combinations, configuration precedence, FRRS_DEBUG help output, and the default fast-export safety flags (`--reencode=yes`, `core.quotepath=false`, `--mark-tags`).

### Deprecation roadmap
- Reiterated the staged removal of legacy cleanup syntaxes and deprecated analysis threshold flags, pointing users to configuration-based replacements.
