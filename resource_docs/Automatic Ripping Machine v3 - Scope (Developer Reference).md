# ARM v3 Scope — Developer Reference

Source: `docs/arch/00-vision.md`, `docs/arch/06-deployment.md`, `docs/arch/07-open-questions.md`, `docs/plans/MASTER_IMPLEMENTATION_PLAN.md`, and `VERSION` (`3.0.0-rc3`) on [`integration/all-prs`](https://github.com/shitwolfymakes/automatic-ripping-machine/tree/integration/all-prs).

## TL;DR

v3 is a ground\-up rebuild of ARM as five single\-responsibility containers instead of one monolith. Core promise unchanged: insert a disc, walk away, get a finished file. Scope splits into four layers below — what it does, how it's built, what platform it runs on, and what's deliberately excluded.

## Functional Scope

| Capability | Detail |
| --- | --- |
| Disc detection | `ioctl(CDROM_DRIVE_STATUS)` poll, 2s interval — no udev dependency |
| Video identify | TMDB (primary) → OMDB (fallback) |
| Audio identify | MusicBrainz Disc ID |
| Ripping | MakeMKV (video), `abcde` (audio CD), raw ISO dump (data discs) |
| Track selection | Per\-title, via `rip_presets.track_selection` |
| Transcoding | HandBrake, in an ephemeral per\-job container |
| GPU acceleration | Intel QSV / AMD VAAPI / NVIDIA NVENC, auto\-detected, opt\-in overlay |
| Multi\-drive | One long\-running ripper container per drive, fully parallel |
| Sessions | Named (rip preset, transcode preset, output template) bundle — rip once, transcode many times later |
| Crash recovery | Interrupted rips restart from scratch; transcode tasks resume from last checkpoint |
| Notifications | Outbound via Apprise, pass\-through URLs, no ARM\-side service dictionary |
| Auth | JWT, argon2id, forced password change on first login |
| Progress | Live per\-job progress over WebSocket |
| Debugging | Per\-job log zip, scoped to one `job_id` |
| Install | One\-command installer — TLS certs, secrets, per\-drive containers |

## Architectural Scope

```
UI (Vue/nginx) ──▶ Backend (FastAPI) ──▶ Ripper × N (one per drive)
                        │        └────▶ Transcode (ephemeral, per job)
                        ▼
                    Postgres
```

- **Five single\-responsibility services.** No service does "a little of the other guy's job."
- **Backend is the only service that talks to the internet.** Ripper and Transcode never make outbound calls.
- **Two transports only.** REST for anything that must durably land (state transitions); WebSocket for progress that a dropped tick doesn't hurt.
- **TLS on every hop**, including Postgres, via a per\-install internal CA. LAN\-only by design — never internet\-exposed.
- **PUID/PGID file ownership**, with a hard rule: **never recursively `chown` a user\-mounted volume.** A startup ownership mismatch fails loudly instead of "fixing" it.

These aren't just implementation choices — they define what's in scope. Anything that would need a second internet\-facing service, a plaintext hop, or forcible ownership repair is out of bounds by design, not backlog.

## Platform Scope

**Supported (one target):**

```
Linux + Docker Engine ≥ 24 + Compose v2 ≥ 2.20
```

Distro\-agnostic — Ubuntu, Debian, Fedora, Arch all fine.

**Explicitly NOT supported** (dropped or never a goal, not just untested):

- Unraid / Synology DSM / QNAP / other NAS\-appliance container GUIs — *dropped 2026\-06\-05*
- TrueNAS / iX Systems
- Kubernetes / Helm
- Docker Desktop on macOS/Windows **for ripping** — internal SATA drives can't cross the VM boundary (UI/transcoder\-as\-frontend still works over SMB/WSL2)
- Podman — may work by accident, not tested

## Explicit Non\-Goals

Not "not yet" — deliberately decided against for v3.0:

- ❌ **Multi\-tenancy / RBAC** — single admin, single household by design
- ❌ **v2 → v3 data migration** — opt into a fresh DB, or stay on the pinned v2 tag
- ❌ **In\-backend transcoding** — always a dedicated ephemeral container
- ❌ **Metrics / tracing** (Prometheus, OpenTelemetry) — structured logs only; the `events` table is the closest thing to metrics
- ❌ **Queue broker** (Redis, NATS) — DB\-as\-queue (`SELECT … FOR UPDATE SKIP LOCKED`) is deliberately good enough at 1–4 drives / 1–4 concurrent transcodes; revisit only if a concrete pain point emerges (tracked as OQ\-1 in `docs/arch/07-open-questions.md`)

## Shipped vs. Deferred (as of `3.0.0-rc3`)

**✅ Shipped and validated end\-to\-end** — everything in [Functional Scope](#functional-scope) above, including real BD/DVD/CD rips on contributor hardware and an automated crash\-recovery drill (`devtools/crash-drill.sh`).

**🚧 Explicitly deferred (not a v3.0 non\-goal — candidate v3.1 backlog):**

| Item | Status |
| --- | --- |
| Placeholder rips (rip immediately on identify\-miss, resolve identity later) | Fully designed, not built |
| ISO\-source ripping as a real feature | Internal test hook only (`ARM_MANUAL_TRIGGER_ISO`) |
| TV\-series\-aware ripping (episode detection/naming) | Not designed |
| Community disc\-fingerprint DB | Schema columns reserved (`disc_fingerprint`, `aacs_disc_id`), nothing writes to them yet |

**🔴 The one actual blocker to release:** maintainer sign\-off. Every technical readiness criterion in the cutover checklist is already met.
