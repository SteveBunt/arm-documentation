# Migrating from vX to vY

This guide covers the breaking changes introduced in vY.0.0 and how to
update your code. See the [CHANGELOG](../../CHANGELOG.md) for the full list
of changes in this release.

## Who needs to migrate

Describe who's affected — e.g. "anyone calling `oldFunction()` directly,"
or "anyone using the v1 config file format."

## Breaking changes

### [Change 1 — e.g. `oldFunction()` renamed to `newFunction()`]

**Before:**
```
oldFunction(arg)
```

**After:**
```
newFunction(arg)
```

**Why:** Brief reasoning, or a link to the RFC/issue that proposed it.

### [Change 2]

**Before / After / Why**, same pattern as above.

## Automated migration

If a codemod, migration script, or CLI flag can do some of this
automatically, document it here:

```bash
npx project-name-migrate v1-to-v2
```

## Need help?

If you get stuck migrating, see [SUPPORT.md](../../SUPPORT.md).
