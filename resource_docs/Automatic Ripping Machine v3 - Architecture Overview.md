# ARM v3 — Architecture Overview

A one\-page orientation to `docs/arch/` on the [`integration/all-prs`](https://github.com/shitwolfymakes/automatic-ripping-machine/tree/integration/all-prs) branch. For per\-document detail, see the companion Architecture Summary; for full field\-level and rationale\-level depth, see the Detailed Architecture Reference.

## What v3 Is

ARM v3 is a ground\-up rebuild of the Automatic Ripping Machine — a self\-hosted system that detects an inserted optical disc, identifies it, rips it, and transcodes it into a media library, with no manual intervention beyond inserting the disc. v2 did all of this inside a single container; v3 splits the same job across five purpose\-built services coordinated by one backend, specifically to fix two problems v2 couldn't: heavy work in one area (a rip, a transcode) starving everything else, and a crashed or power\-interrupted batch having no way to recover.

## The Shape of the System

Five services, one job each: a **Vue UI** (stateless, renders state and takes commands), a **FastAPI Backend** (owns all durable state, is the only service allowed to talk to the internet, and spawns the other workers), one **Ripper container per optical drive** (long\-running, bound to one `/dev/sr*`, never talks to the internet), an **ephemeral Transcode container per job** (spawned on demand, exits when done), and **Postgres** (the single source of truth for everything). Rippers and transcoders coordinate only through the Backend — never directly with each other — over two transports: REST for anything that must durably land, WebSocket for live progress that can simply be superseded by the next tick.

## The Decisions That Define the Architecture

- **A rip crashes back to the start, not mid\-point.** An interrupted rip restarts entirely — every track re\-rips, nothing resumes mid\-title — because MakeMKV can't resume anyway and a "done" row in the DB isn't guaranteed durable on disk. Transcoding, by contrast, uses real per\-task claims and checkpointing, because a multi\-hour transcode batch losing everything is a much worse trade.
- **Rip once, transcode many times (Sessions).** Rip strategy, transcode settings, and output\-path convention are three separate, recomposable tables instead of one bundled v2\-style concept — so a disc ripped six months ago can be re\-transcoded today with no re\-rip.
- **The Backend is the only service that touches the internet.** Rippers and transcoders are simpler and more secure for it — no API keys, no outbound firewall holes, credentials living in exactly one place.
- **TLS everywhere, LAN\-only, never internet\-exposed.** Every hop — including Postgres itself — runs behind a per\-install internal CA. Remote access is meant to go through WireGuard/Tailscale, not port\-forwarding.
- **File ownership never gets fixed by force.** v3 will not recursively `chown` a user\-mounted volume — a direct reaction to real v2 bugs that clobbered user\-owned Plex libraries. A startup ownership mismatch fails loudly instead.
- **Filenames are honest, not guessed.** ARM never invents an episode number or a season it wasn't told — mechanical, greppable defaults that the user renames once they know more, rather than a confident wrong guess.
- **Docker Compose on Linux, full stop.** No Kubernetes, no NAS\-appliance GUIs, no TrueNAS — one supported deployment target, matching the single\-admin homelab user this is built for.

## What's Explicitly Out of Scope

Multi\-tenancy or RBAC (one admin, one household), a v2\-to\-v3 data migration (start clean or stay on v2), in\-backend transcoding (always a dedicated ephemeral container), and — for now — anything beyond structured\-log observability (no Prometheus, no tracing).

## Where the Project Stands

`VERSION` is `3.0.0-rc3`. Every phase of the build plan through crash recovery, real disc\-identification, sessions, GPU transcoding, and real\-disc validation on contributor hardware is shipped. The one thing left before the "cutover" — the single PR that retires v2 from `main` — is maintainer sign\-off; every technical readiness criterion is already met.

## Document Index

| Doc | One\-line orientation |
| --- | --- |
| **00** — Vision, goals, principles | Why rebuild instead of refactor, who it's for, six design principles |
| **01** — System architecture overview | The five\-service topology and the disc\-to\-file data flow |
| **02** — Job lifecycle & crash recovery | State machines for jobs/tracks/sessions, and why crashes restart rips from scratch |
| **03** — Protocol: REST \+ WebSocket | The full wire contract between every service pair |
| **04** — Data model | Every core Postgres table and why it's shaped the way it is |
| **05** — Cross\-cutting concerns | Auth, TLS, secrets, logging, code quality, notifications |
| **06** — Deployment | Install layout, build chain, file ownership, backup/upgrade |
| **07** — Open questions | The one thing still genuinely undecided (the queue mechanism) |
| **08** — v2 isolation & cutover | How v3 was built without touching v2, and how the switchover happens |
| **09** — Testing philosophy | Zero\-infrastructure testing, coverage policy, what's deliberately untested |
| **10** — ISO\-source ripping | A proposed (not\-yet\-built) feature: rip from a file instead of a disc |

## Further Reading

- **Architecture Summary** — a condensed, paragraph\-per\-document walkthrough.
- **Architecture Reference (Detailed)** — the full field\-level and rationale\-level reference.
