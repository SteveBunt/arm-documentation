# Deprecation Policy

This document describes how [Project Name] deprecates and removes
features.

## Process

1. **Announce** — the feature is marked deprecated in the code (e.g. a
   compiler/runtime warning, a `@deprecated` annotation, or a docs notice)
   and in the [CHANGELOG](CHANGELOG.md).
2. **Notice period** — the feature remains available, deprecated, for at
   least [N releases / N months] before removal. This gives downstream
   users time to migrate.
3. **Migration path** — a deprecation announcement is accompanied by
   guidance on what to use instead, and a migration guide for anything
   non-trivial (see `docs/migrations/`).
4. **Removal** — the feature is removed in the next MAJOR release, per
   [VERSIONING.md](VERSIONING.md), and the removal is called out clearly
   in that release's CHANGELOG entry.

## Exceptions

Security-critical issues may require a faster deprecation/removal
timeline than the standard notice period. This will always be called out
explicitly and paired with clear migration guidance.

## Currently deprecated

| Feature | Deprecated in | Removal planned | Replacement |
|---|---|---|---|
| [feature name] | vX.Y.0 | vN.0.0 | [replacement] |
