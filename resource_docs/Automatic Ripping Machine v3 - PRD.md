# Automatic Ripping Machine v3 — Product Requirements Document

Source material: [b3n.org/automatic\-ripping\-machine](https://b3n.org/automatic-ripping-machine/), the [shitwolfymakes/automatic\-ripping\-machine, `integration/all-prs` branch](https://github.com/shitwolfymakes/automatic-ripping-machine/tree/integration/all-prs), and its `docs/arch/` architecture set, `docs/plans/MASTER_IMPLEMENTATION_PLAN.md`, and `CHANGELOG.md`/`VERSION` (currently `3.0.0-rc3`).

This PRD covers the whole v3 rebuild as a single release, not one feature within it — v3 replaces the entire v2 application (a single\-container Flask app) with a multi\-container, database\-backed system, while keeping the same core promise: insert a disc, walk away, get a finished file.

## Problem Statement

ARM v2 works, but two structural problems in its design cause real user pain and block further growth. First, ripping, transcoding, and the web UI all share one process tree inside a single container, so heavy work in one area (a transcode, a second concurrent rip) starves every other area — the UI becomes unresponsive exactly when the user most wants to check progress. Second, v2 treats each rip as one atomic, non\-resumable operation: a user who queues five discs and loses power mid\-batch on disc \#3 has no way to recover completed work, and must restart from scratch. A closely related, adjacent pain point is that v2 conflates ripping and transcoding into a single irreversible pipeline, so a user who wants to re\-encode a disc with different settings later must re\-rip the physical disc rather than reuse the raw bits already captured. These problems compound for ARM's core audience — homelab data\-hoarders who run batches of discs unattended and expect the system to be as durable as the shelf of discs it's replacing.

## Goals

- **Resource isolation:** N concurrent rips and a transcode must not degrade the UI or each other. Achieved by splitting ripping, transcoding, backend state, and UI into separate containers, each with a single responsibility.
- **Crash\-safe batch processing:** A power event or crash mid\-batch must not discard completed work. Verified by an automated crash drill (five queued rips \+ simulated power cut mid\-batch resumes cleanly with no manual intervention).
- **Reusable rips (Sessions):** A user must be able to apply a new transcode preset to an already\-ripped disc months later, without re\-ripping. Verified by sessions and presets being user\-editable in the UI, decoupled from the original rip.
- **Fast, unattended onboarding:** A fresh install on a new host must reach the login screen in under 5 minutes, with no manual YAML editing.
- **Maintain feature parity with v2's core promise:** disc detection, type identification (video/audio/data), automated ripping (MakeMKV), transcoding (Handbrake/HandBrakeCLI), metadata lookup (TMDB/OMDB/MusicBrainz), multi\-drive parallel operation, and outbound notifications must all still work, at least as well as they do today.

## Non\-Goals

- **Multi\-tenancy or RBAC.** v3 targets a single homelab admin. Not a shared service or commercial product — out of scope because the target user and threat model don't call for it, and it would add auth/authorization complexity with no corresponding user.
- **v2 → v3 data migration.** v3 starts from a clean Postgres schema; v2 users opt in by pointing at a new database (and re\-ripping if they want the new data model's benefits), or stay on the pinned v2 tag. Out of scope because v2's SQLite schema doesn't map cleanly onto v3's job/session model, and building a migration path would delay the rebuild for a one\-time, one\-directional need.
- **TrueNAS / iX Systems, Unraid, Synology, QNAP, or other NAS\-appliance container GUIs.** Only Linux \+ Docker Compose ≥ v24/Compose v2 is supported. Out of scope because these platforms have their own container\-orchestration quirks that would multiply the support and testing surface for a single\-maintainer project; they may work by accident but are untested.
- **Kubernetes / Helm.** Docker Compose is the only supported deployment surface. Out of scope — massive operational overkill for a single\-host homelab appliance.
- **In\-backend transcoding.** Transcoding always runs in a dedicated ephemeral container, never inside the Backend process. Out of scope for the Backend's role — keeping transcode isolated is what delivers the resource\-isolation goal above.
- **Deeper observability (metrics/tracing).** v3.0 ships structured JSON logs only — no Prometheus, no OpenTelemetry. Deferred to a v3.1 backlog because it's disproportionate tooling for a 1\-4\-drive homelab scale; the typed `events` table covers ad hoc analysis in the meantime.

## User Stories

**Primary persona: the homelab operator (often a "data\-hoarder" who rips to preserve raw bits, not just to feed a streaming app).**

- As a homelab operator, I want to insert a disc and have it automatically detected, identified, ripped, and ejected, so that I don't have to babysit the process disc\-by\-disc.
- As a homelab operator running multiple optical drives, I want each drive to rip independently and in parallel, so a slow or problematic disc in one drive doesn't block the others.
- As a homelab operator, I want the raw rip retained by default (no auto\-prune), so that I can decide later — sometimes months later — how I want it transcoded, without touching the physical disc again.
- As a homelab operator, I want to apply a new transcode preset (a "session") to an already\-ripped disc from the UI, so that I can experiment with encoding settings or fix a bad transcode without re\-ripping.
- As a homelab operator, I want the system to recover automatically from a crash or power loss mid\-batch — resuming any in\-flight rip rather than losing it — so that an interrupted overnight batch doesn't cost me the whole night's work.
- As a homelab operator with a GPU, I want hardware\-accelerated transcoding (Intel QSV / AMD VAAPI / NVIDIA NVENC), so that transcodes finish faster without pegging my CPU.
- As a homelab operator, I want outbound notifications (via Apprise, to whatever service I already use) when a rip or transcode completes or fails, so I don't have to keep the UI open to know when a batch is done.
- As a homelab operator setting up ARM for the first time, I want a single install command that provisions TLS certs, the database, and per\-drive services, so that I can be running in minutes without hand\-editing compose files or YAML.
- As a homelab operator, I want to log in with a securely generated first\-boot password that forces a change, rather than a well\-known default, so my instance isn't trivially compromised on my LAN.

**Edge cases and error states:**

- As a homelab operator, when a disc can't be automatically identified against TMDB/OMDB/MusicBrainz, I want to be prompted to resolve its identity manually (or mark it a "skip" for a home movie), rather than have the rip silently fail or hang indefinitely.
- As a homelab operator, if a title's rip fails partway through a multi\-title disc, I want the system to track that failure per\-title so I can see and retry it, rather than losing the whole disc's progress.
- As a bug\-reporter, I want to download one log file scoped to a single job, so I can attach it to a GitHub issue without hunting across four separate service logs.

**Secondary persona: the contributor/maintainer.**

- As a contributor, I want v3 developed in isolation from v2 so that no file under active v2 maintenance is touched until cutover, so the rebuild doesn't destabilize the stable release contributors and users currently depend on.
- As a maintainer, I want a documented, checklist\-driven cutover process (not an ad hoc branch merge), so that retiring v2 in favor of v3 is a deliberate, reviewable, one\-time event.

## Requirements

### Must\-Have (P0) — required to replace v2

1. **Service isolation.** Ripping, transcoding, backend state, and UI run as separate containers (FastAPI backend, Vue UI, one ripper container per drive, ephemeral per\-job transcoder, Postgres).
   - *Acceptance:* N concurrent rips plus a transcode do not measurably degrade UI responsiveness; each service has exactly one reason to exist per `01-architecture.md`.
2. **Durable state in Postgres.** All durable state (jobs, tracks, sessions, config, users) lives in Postgres, not SQLite or in\-process memory.
   - *Acceptance:* Backend restart does not lose job/track state; Alembic migrations apply cleanly on a fresh DB.
3. **Real disc identification.** Video discs are identified via TMDB (primary) with OMDB fallback; CDs via MusicBrainz; identification failures are surfaced to the user rather than silently dropped.
   - *Acceptance:* A known disc's title/year populate automatically; an unknown disc lands in a `needs_user_input`/`awaiting_user_id` state rather than failing.
4. **Rip pipeline.** MakeMKV\-based ripping to `/raw`, per\-title track status tracking, disc ejection on completion.
   - *Acceptance:* A real BD, DVD, and audio CD each complete an end\-to\-end rip on contributor hardware.
5. **Sessions and presets (rip once, transcode many).** Rip and transcode are decoupled; a session (transcode preset) can be applied to a job at any time after the rip completes, including "months later."
   - *Acceptance:* A ripped job can have a new session applied from the UI without re\-ripping; built\-in rip/transcode presets are seeded on first boot.
6. **Ephemeral, GPU\-capable transcode containers.** Backend spawns one transcode container per job via the Docker socket; container exits on completion; GPU pass\-through (QSV/VAAPI/NVENC) is auto\-detected and optional.
   - *Acceptance:* A transcode job produces a file in `/media` in a Plex\-friendly path; GPU and CPU paths both route correctly through the dispatcher.
7. **Crash recovery.** Every long\-running operation is checkpointable; a stale in\-progress row with no live worker triggers automatic re\-queueing.
   - *Acceptance:* Five queued rips across two drives, with a simulated power cut mid\-batch, resume cleanly with no manual intervention (automated via `devtools/crash-drill.sh`).
8. **Auto\-session on rip complete.** A drive's default session auto\-applies when a rip finishes, if configured.
   - *Acceptance:* With `default_session_id` set and `auto_transcode_on_idle=true`, a completed rip auto\-queues its transcode without the user clicking "Apply session"; a collision guard skips auto\-apply against existing completed outputs.
9. **User auth.** JWT\-based login, argon2id password hashing, forced password change on first boot (no persistent default credentials).
   - *Acceptance:* First\-boot admin account requires a password change before any endpoint besides `/api/auth/*` is reachable.
10. **Notifications.** Apprise\-based outbound notifications configured from the UI, with no ARM\-side per\-service dictionary to maintain.
    - *Acceptance:* Pasting a valid Apprise URL into the UI results in notifications firing on typed events (e.g., `rip.completed`, `transcode.failed`).
11. **Per\-job log access.** Structured JSON logs per service, persisted to a shared volume, with a per\-job log zip endpoint for bug reports.
    - *Acceptance:* A user can download one zip scoped to a single `job_id` containing the relevant slice of every service's log.
12. **One\-command install.** `install.sh` provisions the full stack, including TLS certs (internal CA) and per\-drive service blocks, with no manual YAML editing.
    - *Acceptance:* A fresh host reaches the login screen in under 5 minutes after running the installer.
13. **CI and supply\-chain hygiene.** Green CI (lint, test, OpenAPI\-drift, build) on `main`, published images for the single supported target.
    - *Acceptance:* All CI jobs pass on `main`; the full stack runs end\-to\-end on a reference Linux \+ Docker Compose host.

### Nice\-to\-Have (P1) — improves the experience, not blocking

1. **Placeholder rips (deferred identity).** Opt\-in flow where an identification miss rips immediately to `/raw/<job_id>/` instead of blocking; queued sessions park in a `waiting_identify` state until the user resolves identity later. No files are ever renamed or moved as a side effect.
   - *Acceptance:* With `block_on_miss=false` or a `deferred_placeholder` rip preset, an unknown disc rips without blocking; the user resolves identity via the API/UI; queued transcodes fan out against the resolved title.
   - *Status:* designed (Phase 10 in the master plan), not yet built. Candidate fast\-follow after cutover.
2. **Generic\-title "skip" mode for home movies.** A form for discs that will never resolve against a metadata provider.
   - *Depends on:* placeholder rips above.

### Future Considerations (P2) — explicitly out of scope for v3.0

1. **ISO\-source ripping.** Ripping from a mounted `.iso` file instead of a physical disc, as a user\-facing feature. (An internal\-only test rig exists for CI/fixture purposes; it is not exposed to end users.) Designed in `docs/arch/10-iso-source-ripping.md` but not built as a feature.
2. **TV\-series\-aware ripping.** Episode detection and naming conventions for TV box sets, building on the sessions/presets foundation.
3. **Queue mechanism upgrade.** Replacing the current DB\-as\-queue pattern (`SELECT … FOR UPDATE SKIP LOCKED`) with Redis\+RQ or NATS, if a concrete pain point emerges (multi\-minute claim latency, retry explosion) at higher concurrency than the expected 1\-4 drives / 1\-4 concurrent transcodes.
4. **Deeper observability.** Prometheus metrics, OpenTelemetry tracing, log aggregation (Loki/ELK) — deferred to a v3.1 backlog.
5. **Community disc\-fingerprint database.** The `disc_fingerprint`/`aacs_disc_id` columns exist in the schema from Phase 0, but populating and using a shared community database of disc fingerprints is not designed or scheduled — the schema is forward\-compatible insurance, not a v3.0 deliverable.

## Success Metrics

ARM is self\-hosted homelab software with no telemetry or phone\-home — there is no usage\-analytics pipeline, so most signals here are qualitative or drawn from the project's own CI/test infrastructure and community channels (GitHub issues/discussions, Discord) rather than in\-product instrumentation.

**Leading indicators (days to weeks around cutover):**

- Readiness\-criteria checklist completion in `08-v2-isolation-and-cutover.md` — target: 100% (currently all criteria met except maintainer sign\-off).
- CI pass rate on `main` — target: 100% green across all jobs (lint ×3, test ×2, openapi\-drift, build ×4).
- Fresh\-install time\-to\-login\-screen — target: under 5 minutes, validated via the documented install path.
- Real\-disc smoke matrix — target: at least one successful end\-to\-end BD, DVD, and CD rip on contributor hardware (met as of the last recorded validation).
- Crash\-recovery drill pass rate — target: 100% pass on `devtools/crash-drill.sh`, run on every relevant PR.

**Lagging indicators (weeks to months post\-cutover):**

- Community adoption of the v3 image tag relative to v2 (Docker Hub pull trend, GitHub Discussions/issue volume referencing v3).
- Reduction in issues describing v2's known pain points — "UI unresponsive during a rip," "lost a batch after a crash/power loss" — as a share of new issues filed.
- Uptake of Sessions (qualitative: forum/Discord mentions of re\-transcoding without re\-ripping, a capability v2 never had).
- Rate of "reverted to v2" reports, which would signal an unresolved regression.

## Open Questions

- **Queue mechanism (OQ\-1).** Should DB\-as\-queue (`SELECT … FOR UPDATE SKIP LOCKED`) be upgraded to a dedicated broker? — *Engineering, non\-blocking.* Explicitly deferred until a concrete pain point emerges; not a v3.0 blocker.
- **Placeholder rips timing.** Should Phase 10 (placeholder rips) ship before or shortly after the v3.0 cutover, given it's designed but unbuilt and not a readiness\-criteria item? — *Stakeholder/product, non\-blocking* for cutover itself, but affects v3.0\-vs\-v3.1 scoping communication to users.
- **Community disc\-fingerprint database.** No design exists yet for how a shared fingerprint database would be populated, moderated, or queried — only the schema columns are reserved. — *Engineering, non\-blocking,* parked for v3.1 backlog.
- **Cutover sign\-off timing.** All technical readiness criteria are met; the only remaining gate is maintainer agreement that v3 is ready to replace v2 on `main`. — *Stakeholder, blocking* for Phase 16 (the cutover PR) specifically, though not for continued v3 use in the interim (v3 and v2 already coexist under the isolation plan).

## Timeline Considerations

- **Current state:** `VERSION` is `3.0.0-rc3` — the project is in release\-candidate territory, not early development. Phases 0 through 15.5 of the master implementation plan are shipped or functionally complete; only Phase 16 (the cutover PR) remains, and it is described in the plan as "mechanical... not a design phase."
- **No external hard deadline.** This is an open\-source, community\-maintained project with no contractual or regulatory date; the only true dependency is maintainer sign\-off (Phase 15 → Phase 16).
- **Phasing already executed sequentially** per the master plan (each phase depended on the state shipped in the previous one — e.g., Phase 16 depends on Phase 15's full exit criteria being met, which itself depended on Phases 3, 7, and 9). Any P1/P2 work above should be scoped as **v3.1**, phased in after cutover rather than reopening the v3.0 readiness gate.
- **Suggested phasing for P1/P2 backlog:** Placeholder rips (P1) is the most natural first v3.1 item since it's fully designed; ISO\-source ripping and TV\-series\-aware ripping can follow; the queue\-mechanism reassessment and deeper observability are explicitly "resolve only when pain emerges" items with no fixed timeline.
