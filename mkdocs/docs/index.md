# Automatic Ripping Machine (A.R.M.) — Project Vision, Aims, and Objectives

Sources: [b3n.org/automatic\-ripping\-machine](https://b3n.org/automatic-ripping-machine/) (original project overview) and the [shitwolfymakes/automatic\-ripping\-machine, `integration/all-prs` branch](https://github.com/shitwolfymakes/automatic-ripping-machine/tree/integration/all-prs) (v3 rebuild — architecture docs, README, and roadmap)

This fork is mid\-rebuild: the original project ("v2") is a single\-container Flask app; the `integration/all-prs` branch is a **ground\-up rewrite ("v3")** that keeps the same core mission but splits it into a multi\-container, database\-backed system.

## Original Vision (v2)

Fully automate the digitization of optical media. Insert a disc, and the system detects it, identifies the media type, and autonomously performs the right action — no interaction beyond inserting the disc.

- **Video (DVD/Blu\-ray):** identified via the OMDb API, ripped with MakeMKV, transcoded with Handbrake, named and foldered for Plex/Emby.
- **Audio (CD):** ripped with abcde, tagged and art\-fetched from MusicBrainz.
- **Data discs:** backed up as raw ISO images.
- Headless and server\-oriented; supports multiple optical drives ripping in parallel; sends notifications via IFTTT, Pushbullet, Slack, Discord, and others.

## v3 Vision — Who It's For

**Single\-admin homelab users** — one person running ARM for their own household. Explicitly not a shared service, multi\-tenant platform, or commercial product. The user base skews toward **data\-hoarders**, who rip to preserve raw bits rather than just to feed a streaming app — this shapes defaults like keeping raw files forever and treating re\-transcode\-from\-raw as a first\-class operation.

## Problems v3 Is Built to Solve

1. **Resource isolation.** In v2, ripping, UI, and transcoding share one process tree, so heavy work in one area starves the others. v3 separates these into services so concurrent rips and a transcode don't degrade the UI.
2. **Batch\-rip resumability.** In v2, a power loss mid\-batch discards progress with no recovery path. v3 checkpoints finely enough that completed work survives a crash and unfinished work re\-queues automatically on restart.
3. **Sessions** (rip once, transcode many times with different presets) — v2 conflates ripping and transcoding into one irreversible pipeline; v3 separates them so a ripped disc can be re\-transcoded later without re\-ripping.

## Design Principles

1. **Bits first, metadata second, transcode third** — a rip succeeds once bits are safely on disk; metadata and transcoding are independent stages that can fail, retry, or re\-run without touching the raw.
2. **One service, one responsibility** — Ripper turns a disc into bytes; Backend owns state and talks to the internet; Transcode turns one raw file into one output; UI renders state and takes commands.
3. **Backend is the single internet boundary** — Ripper and Transcode containers never call external services (TMDB/OMDB/MusicBrainz/Apprise/webhooks); all external calls and credentials live in the Backend.
4. **Postgres is the source of truth; stdout is the source of logs** — durable state lives in Postgres; logs are structured JSON, no third persistence mechanism.
5. **Crash\-safe by default** — every long\-running operation is checkpointable; a stale "in\-progress" row with no live worker signals "re\-queue me."
6. **No sacred cows** — a greenfield rebuild; any assumption inherited from v2 is open for re\-decision from first principles.

## Explicit Non\-Goals (v3.0)

- No TrueNAS / iX Systems support.
- No v2 → v3 data migration (v3 starts from a clean schema).
- No multi\-tenancy or RBAC — single admin only.
- No Kubernetes/Helm — Docker Compose is the only supported deployment surface.
- No in\-backend transcoding — transcoding always runs in a dedicated ephemeral container.

## Success Criteria for v3.0

- Five discs queued across two drives complete without manual intervention, including a simulated power\-cut mid\-batch that resumes cleanly.
- A ripped disc can have a new session (transcode preset) applied from the UI months later without re\-ripping.
- A single PR can change a protocol payload and its two endpoints (ripper side \+ backend side) atomically.
- A bug\-reporter can download one log file scoped to one job ID and attach it to a GitHub issue.
- A fresh install on a new host reaches the login screen in under 5 minutes.

## Current Status / Roadmap

**Shipped in v3:**

- Service split into FastAPI backend, Vue UI, Postgres, one ripper container per drive, and an ephemeral per\-job transcoder, orchestrated with Docker Compose.
- Migrated from SQLite to Postgres with async SQLAlchemy/Alembic.
- Ripper and UI rewritten as new codebases with a pytest suite and a backend statement\-coverage policy.
- Sessions, rip presets, and transcode presets implemented and user\-editable in the UI (including music\-to\-FLAC/MP3, data copy, and ISO dump).
- One\-command install via `install.sh`, including TLS certs and per\-drive service blocks.
- Notifications via Apprise, configured from the UI.
- GPU transcoding (Intel QSV / AMD VAAPI / NVIDIA NVENC) via an opt\-in overlay.

**In progress / ahead:**

- Stabilizing the alpha toward a v3.0 release (signed images for every supported platform, CI\-built release tags).
- Ripping from an `.iso` source instead of a physical disc (designed, not yet built).
- TV\-series\-aware ripping (episode detection and naming) and further session ergonomics.
