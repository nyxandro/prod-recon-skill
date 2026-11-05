# Changelog

The current version is `metadata.version` in `prod-recon/SKILL.md`. Every version is tagged
`vX.Y.Z` in git and published as a GitHub release with its own `.skill` archive. Versions are
numbered MAJOR.MINOR.PATCH.

- **PATCH** (1.0.1) is a fix or clearer wording. The skill behaves the same.
- **MINOR** (1.1.0) is a new or changed rule, for example another source to read.
- **MAJOR** (2.0.0) changes the read-only contract or the shape of the report.

## [1.0.0] - 2026-11-06

The first numbered version. The skill itself is unchanged since `ec55d9e`. This release adds the
version number to `SKILL.md`.

## Before 1.0.0

- **2026-10-24.** The skill installs as a Claude Code plugin from the
  [nyxandro/skills](https://github.com/nyxandro/skills) catalog (`ec55d9e`).
- **2026-09-13.** First version. Read-only reconnaissance of a running deployment (`ff25f63`).
