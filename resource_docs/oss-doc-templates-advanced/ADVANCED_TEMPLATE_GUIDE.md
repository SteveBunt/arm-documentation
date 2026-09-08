# Advanced Template Guide

This bundle covers the optional/advanced tier of documentation — the docs
you add as a project grows in size, contributors, or formality. Unlike the
first (core) bundle, you should pick and choose from these rather than add
them all at once; each note below says when a file earns its place.

## Governance and process

- **MAINTAINERS.md** — add once you have more than one maintainer.
- **RELEASING.md** — add once releases follow a repeatable process worth
  writing down.
- **VERSIONING.md** — add once you're making promises about backwards
  compatibility (most useful once you hit v1.0).
- **DEPRECATION.md** — add once the project is stable enough that removing
  things needs a notice period.
- **docs/rfcs/** (`README.md` + `0000-template.md`) — add once proposing a
  big change without writing it down first starts causing friction or
  re-litigated decisions.

## Technical and reference

- **ARCHITECTURE.md** — add as soon as a new contributor can't reasonably
  infer the system's shape from the code alone.
- **FAQ.md** — add once the same few questions keep showing up in issues
  or chat.
- **GLOSSARY.md** — add once the project has its own vocabulary that isn't
  obvious to newcomers.
- **docs/migrations/TEMPLATE.md** — copy this once per breaking release
  (e.g. `docs/migrations/v1-to-v2.md`) rather than keeping it generic.
- **TESTING.md** — add once "how do I run/write tests" becomes a common
  question from new contributors.

## Community and growth

- **ADOPTERS.md** — add once you have real production users willing to be
  named; it's a credibility signal for prospective adopters.
- **TRANSLATING.md** — add only if you're actually accepting translations;
  otherwise it invites requests you can't support yet.

## Legal, compliance, and identity

- **AUTHORS** — optional if your Git host's contributor graph already
  covers this; some projects keep it anyway for non-code contributors.
- **NOTICE** — only required if your license mandates it (Apache 2.0 does;
  MIT and BSD don't). Delete it otherwise.
- **TRADEMARK.md** — add once the project name/logo has real recognition
  worth protecting. Get this reviewed by a lawyer before relying on it.
- **CITATION.cff** — add if the project is used in academic or research
  contexts and you want a standard, machine-readable citation format
  (GitHub renders a "Cite this repository" button when this file exists).
- **ACCESSIBILITY.md** — add if the project has a user interface (web app,
  desktop app, docs site) — not typically needed for a CLI tool or library
  with no UI.

## Notes

- All bracketed placeholders (`[Project Name]`, `[INSERT CONTACT EMAIL]`,
  etc.) need editing before these are useful.
- `docs/rfcs/` and `docs/migrations/` are folders, not single files — drop
  them into a `docs/` directory at your repo root, or wherever your
  project already keeps documentation.
- None of this is legal advice, particularly TRADEMARK.md and NOTICE —
  have a lawyer review anything with real legal weight before publishing it.
