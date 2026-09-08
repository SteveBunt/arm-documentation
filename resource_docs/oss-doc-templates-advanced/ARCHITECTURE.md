# Architecture

This document describes the high-level design of [Project Name] — how the
major pieces fit together and why key decisions were made. It's aimed at
contributors who need to understand the system before changing it, not at
end users (see the README and usage docs for that).

## Overview

A short paragraph describing what the system does end-to-end, and a diagram
if one helps.

```
[Client] -> [API layer] -> [Core logic] -> [Storage]
```

## Components

### Component A

What it's responsible for, its main entry points, and what it depends on.

### Component B

What it's responsible for, its main entry points, and what it depends on.

## Data flow

Describe how a typical request or operation moves through the system, from
input to output.

## Key design decisions

Document decisions that aren't obvious from the code, and the reasoning
behind them — this is often the most valuable part of the document for a
new contributor.

| Decision | Reasoning | Alternatives considered |
|---|---|---|
| e.g. Chose SQLite over Postgres for the default backend | Zero-config for new users | Postgres (rejected: adds a required dependency) |

## Extension points

Describe any plugin systems, interfaces, or hooks that contributors are
expected to extend rather than modify directly.

## Related documents

- [CONTRIBUTING.md](CONTRIBUTING.md) — how to work in this codebase
- `docs/rfcs/` — proposals for significant architectural changes
