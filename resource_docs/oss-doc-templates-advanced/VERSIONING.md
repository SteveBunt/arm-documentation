# Versioning Policy

[Project Name] follows [Semantic Versioning 2.0.0](https://semver.org/):
given a version number `MAJOR.MINOR.PATCH`, we increment the:

- **MAJOR** version when we make incompatible/breaking changes
- **MINOR** version when we add functionality in a backwards-compatible way
- **PATCH** version when we make backwards-compatible bug fixes

## What counts as a breaking change

List the things your project treats as breaking, for example:

- Removing or renaming a public function, class, or CLI flag
- Changing a function's default behavior
- Dropping support for a previously supported language/runtime version
- Changing the format of persisted data without a migration path

## What does not count as breaking

- Adding new, optional functionality
- Performance improvements that don't change behavior
- Fixing a bug that only "worked" due to undocumented behavior (use
  judgment and call this out clearly in the CHANGELOG)
- Changes to code marked experimental/unstable (define your own convention
  for marking these, e.g. an `unstable_` prefix or an explicit notice in
  the docs)

## Supported versions

See [SECURITY.md](SECURITY.md) for which versions currently receive
security patches, and describe your general support window here — e.g.
"the latest major version and the one prior receive bug fixes."

## Pre-1.0 versions

If the project is pre-1.0, note here whether MINOR bumps may still include
breaking changes, per SemVer's pre-1.0 convention.
