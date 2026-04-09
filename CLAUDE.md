# CLAUDE.md

## Chart Versioning

**Always increment the chart `version` in `Chart.yaml` when modifying any chart file** (templates, values, helpers).

Use [Semantic Versioning](https://semver.org/):
- `patch` (0.1.0 → 0.1.1): bug fixes, non-breaking template changes
- `minor` (0.1.0 → 0.2.0): new features, new values keys (backwards compatible)
- `major` (0.1.0 → 1.0.0): breaking changes to values structure or template behavior

The `release.yaml` GitHub Actions workflow packages and publishes charts to GitHub Pages on every push to `master`. A chart will only be re-published if its version has changed (`CR_SKIP_EXISTING: true`).
