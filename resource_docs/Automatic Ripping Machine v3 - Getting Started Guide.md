# Getting Started with Automatic Ripping Machine v3

Source material: `arm_wiki/Getting-Started.md`, `docs/arch/06-deployment.md`, `CONTRIBUTING.md`, `devtools/README.md`, and `VERSION` (currently `3.0.0-rc3`) from the [`integration/all-prs`](https://github.com/shitwolfymakes/automatic-ripping-machine/tree/integration/all-prs) branch.

## Before You Start: a Note on Where the Project Is

v3 is at **release\-candidate stage (`3.0.0-rc3`)**, not a finished, published release yet — this guide reflects that. Two things follow from it:

1. **Published Docker images may not exist yet for every service tag.** `docker compose pull` can 404 during this alpha/rc window. Until v3.0 is officially published, build the images yourself from a checkout rather than pulling them — that path is documented below and is what actually works today.
2. **`install.sh` (the one\-line curl installer) is currently behind the app's own architecture.** It predates the current drive\-enrollment model and still tries to write one Compose service block per detected drive; the current backend instead lets you *enroll* drives from the UI after the stack is already running. `install.sh` is scheduled for a rewrite before the general release. Until then, the project's own contributor tooling (`devtools/setup-dev.sh`) is the accurate path, and it's what this guide uses.

Once v3.0 is officially published, the one\-line installer becomes the intended path for ordinary users — that flow is summarized at the end of this guide so you know what to expect.

## Hardware

- **CPU:** anything reasonably modern transcodes comfortably (6th\-gen\-or\-newer Intel Core/Xeon, or a modern AMD Ryzen/Threadripper/Epyc). A GPU is optional and only speeds up transcoding.
- **Memory:** roughly 1 GB for SD content, 2–8 GB for HD (720p/1080p), 6–16 GB\+ for 4K transcodes, on top of whatever else the stack uses.
- **Optical drive(s):** one or more, each exposed to the host as `/dev/sr*`. ARM runs one ripper container per drive, in parallel.
- **Storage:** ripping is disk\-hungry — budget roughly 10–20 GB free per in\-flight Blu\-ray for intermediate raw files, plus space for the finished media library. Audio CDs are well under 1 GB each.
- **Windows/macOS:** Docker Desktop cannot pass an internal SATA optical drive into its Linux VM, so you cannot rip from Windows or macOS. You can still run the UI \+ transcoder as a library\-management frontend over SMB/WSL2.

## Prerequisites

| Tool | Minimum | Notes |
| --- | --- | --- |
| Docker Engine | 24 | [https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/) |
| `docker compose` | v2 plugin | `docker compose version` must work |
| `openssl` | 1\.1.1 | Used to generate the internal CA |
| `bash` | 4 | Scripts use bash\-4 features |
| `uv` | latest | Python workspace/dependency manager — [https://astral.sh/uv](https://astral.sh/uv) |
| Node / `npm` | per `services/ui/.nvmrc` | Needed to build the UI locally |

Your user must be able to reach the Docker daemon:

```bash
sudo usermod -aG docker "$USER" && newgrp docker
```

You do **not** need to be in the host's optical group yourself — the ripper container gets drive access automatically via `group_add`.

**Supported target:** any Linux host running Docker Engine ≥ 24 and Compose v2 ≥ 2.20 (Ubuntu, Debian, Fedora, Arch, …). This is the one and only supported target for v3.0. Unraid, Synology, QNAP, other NAS\-appliance GUIs, TrueNAS, and Kubernetes are explicitly not supported.

## Step 1: Clone and Bootstrap

```bash
git clone https://github.com/shitwolfymakes/automatic-ripping-machine.git
cd automatic-ripping-machine
git checkout integration/all-prs   # or whichever branch/tag you're targeting

bash devtools/setup-dev.sh
```

`setup-dev.sh` is idempotent — safe to re\-run. In order, it:

1. Checks that `uv`, `docker`, `docker compose`, and `openssl` are available.
2. Runs `uv sync` to create a `.venv/` with all workspace members.
3. Generates the internal CA and certs under `certs/` (delegating to `install.sh --certs-only`) if they don't already exist.
4. Creates `.env` from `.env.example` if missing — filling in a random `POSTGRES_PASSWORD` and `ARM_SERVICE_TOKEN`, and detecting `PUID`/`PGID`/`CDROM_GID` from your host. An existing `.env` is left untouched.
5. Creates `docker-compose.yml` from the template. **No drive enumeration happens here** — you enroll drives from the UI once the stack is up (Step 5).
6. Writes a host\-wide udev rule (`KERNEL=="sr[0-9]*"`, `UDISKS_AUTO=0`) so the host's desktop auto\-mounter doesn't grab your optical drives out from under the ripper.

## Step 2: Start the Stack

```bash
docker compose up -d --build
```

`--build` matters right now: since published images aren't guaranteed to exist yet at this rc stage, this builds every service image locally from the checkout instead of trying to pull.

Check that everything came up healthy:

```bash
docker compose ps
docker compose logs -f arm-backend
```

## Step 3: First Login

On first boot, the backend waits for Postgres, runs its database migrations, and seeds an `admin` account with a **randomly generated password**, written once to the backend's boot log and to `/logs/first-boot.log`\:

```bash
docker compose logs arm-backend | grep -i "admin password"
# or
docker exec armv3-backend cat /logs/first-boot.log
```

Open **`https://localhost:8081`** (or `https://<host-ip>:8081` from another device on your LAN), log in as `admin` with that generated password, and you'll be forced to set a new password immediately — every other endpoint stays locked (HTTP 403) until you do.

## Step 4: Trust the Certificate (Recommended)

The stack serves HTTPS using its own internal CA, so your browser will show a certificate warning on first visit. You can click through it, or trust the CA once per device to make it go away for good:

```bash
bash devtools/trust-ca.sh
```

This installs `certs/arm-ca.crt` into your Linux trust store (and, under WSL, the Windows Current User Root store too, so Chrome/Edge on Windows trust it as well). It's idempotent and safe to re\-run. You can also trust it manually on any device by importing `certs/arm-ca.crt` as a trusted root CA — this is a one\-time action per device; per\-service leaf certs regenerate on reruns but are all signed by the same CA, so you never have to re\-import.

## Step 5: Enroll Your Drive(s)

Unlike v2 (and unlike v3's own not\-yet\-rewritten `install.sh`), drives are not wired into `docker-compose.yml` automatically. Instead:

1. On boot, the backend scans `/sys/class/block/sr*` and `/dev/disk/by-id` and lists every optical drive it finds on the **Drives** page as *detected*.
2. Enroll a drive from that page. This makes the backend create a durable `arm-ripper-<serial>` container for it over the Docker socket — labeled, TLS\-secured, and configured with the right device access. It survives unplug/replug and drive renumbering.
3. Enrolled drives show up as ripper containers outside the `docker compose` project (`docker compose down` won't stop them). Manage them with:

```bash
bash devtools/ripper-containers.sh list
bash devtools/ripper-containers.sh stop
bash devtools/ripper-containers.sh remove
```

## Step 6: Your First Rip

1. **Add metadata API keys (recommended).** In the UI, open **Settings** and add a [TMDb](https://www.themoviedb.org/settings/api) and/or [OMDb](https://www.omdbapi.com/apikey.aspx) API key so ARM can name your video discs. Audio CDs use MusicBrainz, which needs no key.
2. **Insert a disc.** The ripper polls its drive every couple of seconds — no udev events required — and a new job appears on the dashboard within a few seconds of the drive spinning up.
3. **Watch it work.** ARM identifies the disc, rips it (MakeMKV for video, abcde for audio CDs), and streams live progress to the browser. Video then transcodes with HandBrake into your media library.
4. **Eject.** ARM ejects automatically when the rip finishes.

If a disc can't be identified and `block_on_miss` is on (the default), ARM pauses and asks you to confirm or search for the title before ripping rather than guessing or failing silently.

## Once v3.0 Ships: the One\-Command Installer

This is the path the project intends for ordinary end users once `install.sh` catches up to the current architecture and images are published. It won't require a git checkout at all:

```bash
curl -fsSL https://raw.githubusercontent.com/automatic-ripping-machine/automatic-ripping-machine/main/install.sh | bash
```

It installs everything under `~/arm/` by default (override with `--prefix <path>`), generates the internal CA and per\-service certs, seeds `.env` with generated secrets, writes `docker-compose.yml`, and prints the exact next commands (`cd ~/arm && docker compose up -d`). It's idempotent — safe to re\-run any time you attach a new drive or upgrade across a major version, and it preserves your existing `.env` secrets and CA. `install.sh --start` runs `docker compose up -d` automatically instead of just printing the command.

## Troubleshooting Quick Hits

- **Disc won't eject after a rip (desktop hosts only):** your desktop environment's auto\-mounter grabbed the drive. `setup-dev.sh` (and the real installer) write a udev rule to prevent this — confirm it's active with `cat /etc/udev/rules.d/99-arm-no-automount.rules` and reload with `sudo udevadm control --reload-rules && sudo udevadm trigger`.
- **A disc "looks unidentifiable" / zero titles found:** MakeMKV needs both the block device (`/dev/srN`) and its paired SCSI\-generic node (`/dev/sgM`) — the ripper container's entrypoint handles this pairing itself now, so this should be rare, but if you hit it, check `ls /sys/class/block/sr0/device/scsi_generic/` to find the real pairing.
- **v2 is still running on the same host:** stop it before ripping with v3. The kernel allows two containers to map the same `/dev/sr0`, but MakeMKV won't cope with two processes touching the same disc at once.
- **File ownership looks wrong on `/raw` or `/media`\:** v3 never recursively `chown`s a user\-mounted volume (a deliberate change from v2, which had real data\-loss bugs doing this). If ownership is wrong at container startup, it fails with a clear diagnostic instead of silently rewriting your files — check that `PUID`/`PGID` in `.env` match the owner of your host directories.

## Next Steps

- Configure GPU\-accelerated transcoding (Intel QSV / AMD VAAPI / NVIDIA NVENC) if your hardware supports it.
- Explore **Sessions** — apply a different transcode preset to an already\-ripped disc without re\-ripping it.
- Set up Apprise notifications so you don't have to keep the UI open to know when a batch finishes.
- If something looks broken, check `docs/contributors/real-disc-smoke.md` and the architecture docs under `docs/arch/` before filing an issue — several rough edges (like `install.sh` being behind) are already known and tracked.
