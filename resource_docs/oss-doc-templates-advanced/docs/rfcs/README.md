# RFC Process

Significant changes to [Project Name] — breaking changes, new
dependencies, major architectural shifts, or anything that's hard to
undo — go through a lightweight RFC ("request for comments") process
before implementation begins.

## When you need an RFC

- Anything that changes a public API in a breaking way
- New dependencies with a meaningful footprint (binary size, license,
  maintenance burden)
- Architectural changes affecting more than one component
- Anything a maintainer asks to see written up before it's built

Small features, bug fixes, and anything easily reversible don't need one —
just open a pull request.

## Process

1. Copy `0000-template.md` to `NNNN-short-title.md` (use the next
   available number, or `0000-` if you don't know it yet — a maintainer
   will assign the real number on merge).
2. Fill in the template and open a pull request adding it to this folder.
3. The RFC is discussed in the PR. Expect at least [N days] of open
   comment period for anyone to weigh in.
4. A maintainer marks the RFC **accepted**, **rejected**, or **needs
   revision** and merges (or closes) the PR accordingly.
5. Once accepted, implementation happens in follow-up PRs that reference
   the RFC.

## Status values

- `draft` — under discussion
- `accepted` — approved, not yet (fully) implemented
- `implemented` — shipped
- `rejected` — not moving forward, kept for historical reference
- `superseded` — replaced by a later RFC (link it)
