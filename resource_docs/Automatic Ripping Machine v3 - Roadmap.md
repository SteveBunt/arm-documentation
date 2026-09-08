# Automatic Ripping Machine v3 — Roadmap

Format: Now / Next / Later (chosen over a dated timeline because this is a volunteer\-maintained open\-source project with no external deadlines — precise dates would be false precision). Built from `docs/plans/MASTER_IMPLEMENTATION_PLAN.md`, `docs/arch/07-open-questions.md`, the project's existing wiki roadmap (`arm_wiki/Status-Roadmap.md`), and the Future\-Considerations backlog from the v3 PRD.

## Status Overview

**3 in Now, 3 in Next, 5 in Later.** Phases 0–15.5 of the v3 rebuild are shipped or functionally complete — the project is effectively feature\-complete for its v3.0 target (`VERSION` \= `3.0.0-rc3`). Nothing is currently at risk on a hard date, because there are no hard dates; the one thing genuinely blocking release is a human decision, not engineering work.

## Now

Work that is either actively gating the v3.0 release or already in flight.

| Item | Description | Status | Target | Owner | Dependencies |
| --- | --- | --- | --- | --- | --- |
| Maintainer sign\-off | Final "v3 is ready to replace v2" agreement — the last unchecked box in the cutover readiness criteria. | **At risk (stalled — no date set)** | As soon as maintainers convene | ARM maintainers | None technical — every other readiness criterion is met |
| Phase 16 — Cutover PR | Mechanical PR that moves v3 to repo root and retires v2 from `main`, per `08-v2-isolation-and-cutover.md`. | **Blocked** | Immediately after sign\-off | ARM maintainers / core contributors | Maintainer sign\-off (above); Phase 15 (done) |
| Documentation polish (Track A) | User\-facing README rewrite, per\-service README.md files, ADRs for resolved open questions. | **On track** | Lands with cutover | ARM maintainers | None |

## Next

Designed and scoped, good candidates for the first v3.1 push once cutover lands. Not yet started; no committed dates.

| Item | Description | Status | Target | Owner | Dependencies |
| --- | --- | --- | --- | --- | --- |
| Placeholder rips (deferred identity) | Opt\-in flow: an identify miss rips immediately instead of blocking; session applications wait in `waiting_identify` until the user resolves identity later. Fully designed (Phase 10 in the master plan). | **Not started** | v3.1, first push | Unassigned | Phase 7 (transcode fan\-out, done), Phase 6 (session applications, done), Phase 3 (rip pipeline, done) — all satisfied |
| Generic "skip" mode for home movies | A form for discs that will never resolve against a metadata provider (e.g., home videos), so they don't get stuck waiting on an identity that will never come. | **Not started** | v3.1, alongside placeholder rips | Unassigned | Placeholder rips (above) |
| ADRs for resolved architecture decisions | Formal decision records for the open questions already resolved during the rebuild, so the "why" isn't only recoverable from git history. | **Not started** | v3.1 | ARM maintainers | None |

## Later

Directional. Real, but scope and timing are intentionally loose — several are explicitly "resolve only when a concrete pain point emerges," not scheduled work.

| Item | Description | Status | Owner | Dependencies |
| --- | --- | --- | --- | --- |
| ISO\-source ripping (user feature) | Rip from a mounted `.iso` file instead of a physical disc, exposed as a real user\-facing feature rather than the internal CI fixture rig that already exists. Design doc exists (`docs/arch/10-iso-source-ripping.md`). | Designed only | Unassigned | None blocking — ready to scope into a phase when picked up |
| TV\-series\-aware ripping | Episode detection and naming conventions for TV box sets, building on the sessions/presets foundation. | Directional, not designed | Unassigned | Sessions/presets (done) |
| Queue mechanism upgrade | Replace DB\-as\-queue (`SELECT … FOR UPDATE SKIP LOCKED`) with Redis\+RQ or NATS if/when concurrency needs outgrow 1–4 drives / 1–4 concurrent transcodes. | Explicitly deferred (OQ\-1) | Unassigned | Triggered by a concrete pain point, not a date |
| Deeper observability | Prometheus metrics, OpenTelemetry tracing, log aggregation beyond the current structured\-JSON\-to\-`/logs` approach. | Explicitly out of v3.0/v3.1 scope, backlog only | Unassigned | None |
| Community disc\-fingerprint database | Populate and use the `disc_fingerprint`/`aacs_disc_id` columns (schema exists since Phase 0) via a shared community lookup. | Schema\-only placeholder | Unassigned | None designed yet — needs its own spec |

## Risks and Dependencies

- **The whole roadmap currently hangs on one unscheduled human decision.** Maintainer sign\-off has no target date, and every "Next" and "Later" item is implicitly waiting behind it (v3.1 work is unlikely to get planning attention before v3.0 actually ships). This is the single biggest risk on the roadmap — worth naming explicitly rather than leaving as an invisible blocker.
- **No named owners on any Next/Later item.** As a volunteer open\-source project, "unassigned" isn't unusual, but it means none of these have real committed timelines yet — they're backlog, not a plan, until someone picks them up.
- **Platform\-scope narrowing could generate support load post\-cutover.** Unraid/Synology/NAS\-appliance support was explicitly dropped for v3.0 (2026\-06\-05). Users on those platforms who upgrade expecting parity may file issues the project has already decided are out of scope — worth having the "explicitly not supported" messaging ready at cutover, not just in the arch docs.
- **Contributor\-hardware dependency.** v3.0's real\-disc validation (BD/DVD/CD) depended on contributors testing against their own drives. Later items like TV\-series\-aware ripping will need the same kind of volunteer hardware/time, which isn't something a roadmap date can guarantee.

## Changes This Update

This is the first Now/Next/Later view of the project. It reorganizes two existing sources that were tracking roadmap information separately: the wiki's shipped\-vs\-in\-progress list (`arm_wiki/Status-Roadmap.md`) and the phase\-by\-phase master implementation plan. It also surfaces, for the first time in roadmap form, the Future\-Considerations backlog from the v3 PRD (placeholder rips, ISO\-source ripping, TV\-series\-aware ripping, the queue\-mechanism question, observability, and the community fingerprint database) as explicit Next/Later items rather than footnotes.
