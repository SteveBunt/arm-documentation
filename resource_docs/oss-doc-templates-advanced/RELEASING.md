# Release Process

This document describes how maintainers cut a new release of
[Project Name]. It's for maintainers with publish access, not general
contributors.

## Versioning

This project follows [Semantic Versioning](https://semver.org/):
`MAJOR.MINOR.PATCH`.

- **MAJOR** — breaking changes
- **MINOR** — backwards-compatible new features
- **PATCH** — backwards-compatible bug fixes

See [VERSIONING.md](VERSIONING.md) for what counts as breaking.

## Release checklist

1. Make sure `main` is green (all CI checks passing).
2. Update [CHANGELOG.md](CHANGELOG.md): move `[Unreleased]` items under a
   new version heading with today's date.
3. Bump the version number in [wherever it lives — e.g. `package.json`,
   `pyproject.toml`, `Cargo.toml`].
4. Commit: `git commit -m "chore: release vX.Y.Z"`.
5. Tag: `git tag -a vX.Y.Z -m "vX.Y.Z"`.
6. Push: `git push origin main --tags`.
7. Publish to the package registry:
   ```bash
   # e.g. npm publish / twine upload dist/* / cargo publish
   ```
8. Create a GitHub Release from the tag, pasting in the relevant
   CHANGELOG section.
9. Announce the release (community channels, mailing list, etc. — see
   [SUPPORT.md](SUPPORT.md) for where those are).

## Pre-releases

Describe your process for alpha/beta/rc releases, if any, and how they're
tagged (e.g. `v1.2.0-beta.1`).

## Rolling back a bad release

Describe how to yank/deprecate a broken release on your package registry,
and how to communicate the issue to users.
