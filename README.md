# ImmichKeepR

> **Automated backup utility for [Immich](https://immich.app)** — backs up your photos, videos, and PostgreSQL database to Local storage, SMB (NAS), or SFTP destinations, managed entirely through a clean web UI.

![Docker](https://img.shields.io/badge/Docker-required-blue?logo=docker)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## Table of Contents

- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [First-Time Setup](#first-time-setup)
- [Features](#features)
- [docker-compose.yml Reference](#docker-composeyml-reference)
- [Environment Variables](#environment-variables)
- [Backup Scope](#backup-scope)
- [Scheduling](#scheduling)
- [Backup Verification](#backup-verification)
- [Retention Policy](#retention-policy)
- [Port Configuration](#port-configuration)
- [Troubleshooting](#troubleshooting)

---

## Requirements

- **Docker** running on the same host as Immich
- Read access to Immich's library folder (`UPLOAD_LOCATION` from Immich's `.env`)
- An **Immich API key** — generate one in Immich under Settings → API Keys
- Your **PostgreSQL container name** and password (default container name: `immich_postgres`)

---

## Quick Start

Create a `docker-compose.yml` file with the following contents (adjust volumes to match your setup):

```yaml
services:
  immichkeepr:
    image: ghcr.io/movioxsi-lab/immichkeepr:latest
    container_name: ImmichKeepR
    ports:
      - "1515:1515"
    volumes:
      - ./data:/app/data                           # SQLite — persistent config & history
      - /var/run/docker.sock:/var/run/docker.sock  # Required for pg_dump via docker exec
      - /usr/bin/docker:/usr/bin/docker:ro         # Docker binary
      - /:/host                                    # Host filesystem for library access
      # Optional: uncomment if using a local backup destination
      # - /mnt/backups:/backups
    environment:
      - TZ=America/Los_Angeles   # Set your timezone
      - LOG_LEVEL=INFO
    restart: unless-stopped
```

Then run:

```bash
docker compose up -d
```

Open **http://localhost:1515** and follow the [First-Time Setup](#first-time-setup) steps.

---

## First-Time Setup

1. Open **http://localhost:1515**
2. Go to **Settings** and fill in:
   - **Immich Host** — hostname or IP of your Immich server (default: `localhost`)
   - **Immich Port** — default `2283`
   - **Immich API Key** — generate in Immich → Settings → API Keys
   - **Library Path** — your `UPLOAD_LOCATION` from Immich's `.env` (e.g. `/mnt/media/immich`)
   - **PostgreSQL Container** — name of the Immich Postgres container (default: `immich_postgres`)
   - **PostgreSQL Password** — your database password
   - **Timezone** — for displayed timestamps
3. Click **Test Connection** to verify
4. Go to **Destinations** → add a backup target (Local, SMB, or SFTP)
5. Go to **Jobs** → create a job with a schedule and scope
6. Done — ImmichKeepR handles the rest

> **Tip:** Use an admin API key for full access to all users' statistics. A warning will appear in the UI if a non-admin key is detected.

---

## Features

| Feature | Details |
|---|---|
| **Incremental backup** | Only copies new or changed files (mtime + size comparison) |
| **Chunked copy with resume** | Interrupted transfers resume from where they left off |
| **Mirror mode** | Optionally propagates deletions to the destination |
| **Multiple destinations** | Local, SMB (NAS), and SFTP — can all run simultaneously |
| **Flexible backup scope** | Choose photos, encoded video, thumbnails, profile photos, and/or DB |
| **Database backup** | Compressed `pg_dump` via `docker exec` — no PostgreSQL port needed |
| **Scheduler** | Manual trigger, interval (every N hours), or full cron expression |
| **Dashboard** | Live source-vs-backup comparison, per-destination status and free space |
| **Backup verification** | Spot-checks random files by MD5 checksum |
| **Retention policy** | Auto-purges old DB dumps and run logs based on configurable limits |
| **Live backup log** | Real-time current file and progress, polled every 2 seconds |
| **Timezone support** | Configurable timezone for all timestamps |
| **Mobile friendly** | Full UI works on phones and tablets |

---

## docker-compose.yml Reference

| Volume | Required | Purpose |
|---|---|---|
| `./data:/app/data` | ✅ Yes | Persists SQLite database — config, jobs, and run history |
| `/var/run/docker.sock` | ✅ Yes | Allows running `docker exec` for `pg_dump` |
| `/usr/bin/docker:/usr/bin/docker:ro` | ✅ Yes | Docker CLI binary inside the container |
| `/:/host` | ✅ Yes | Lets the container read your Immich library from the host |
| `/mnt/backups:/backups` | ❌ Optional | Only needed if using a local backup destination |

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `TZ` | `America/Los_Angeles` | Container timezone (e.g. `Europe/London`, `Asia/Tokyo`) |
| `LOG_LEVEL` | `INFO` | Logging verbosity: `DEBUG`, `INFO`, `WARNING`, or `ERROR` |
| `DB_PATH` | `/app/data/immichkeepr.db` | SQLite database path — change only if you know why |

---

## Backup Scope

When creating a job you can choose what to include:

| Item | Default | Notes |
|---|---|---|
| Photos & Videos | ✅ ON | Original uploaded files |
| Encoded Videos | ✅ ON | Transcoded versions |
| Thumbnails | ❌ OFF | Regeneratable by Immich — safe to skip |
| Profile Photos | ✅ ON | Small, worth keeping |
| Database | ✅ ON | Compressed `pg_dump` |

Thumbnails are off by default because Immich can regenerate them — skipping them meaningfully reduces backup size and transfer time.

---

## Scheduling

Three modes are available per job:

| Mode | Example | Description |
|---|---|---|
| **Manual** | — | Only runs when you click Run Now |
| **Interval** | Every 6 hours | Runs every N hours |
| **Cron** | `0 2 * * *` | Full cron expression (e.g. daily at 2 AM) |

Cron expressions use standard 5-field format: `minute hour day month weekday`.

---

## Backup Verification

ImmichKeepR can spot-check a sample of backed-up files by comparing MD5 checksums between the source and destination. Trigger a verification from the **Jobs** page on any completed run. Useful for confirming integrity without rescanning the full library.

---

## Retention Policy

To prevent unbounded growth, ImmichKeepR automatically purges:

- **Database dumps** — keeps the last N dumps per destination
- **Run logs** — keeps logs from the last N days

Both limits are configurable in **Settings → Retention**. Purging runs automatically on a daily schedule.

---

## Port Configuration

The default port is **1515**. To change the external port, edit the `ports` line in your `docker-compose.yml`:

```yaml
ports:
  - "8080:1515"   # Access the UI at http://localhost:8080 instead
```

The internal container port is always `1515`.

---

## Troubleshooting

**UI shows "Immich not connected"**
- Verify the host, port, and API key in Settings.
- Ensure ImmichKeepR can reach Immich on the network (both should be on the same Docker host or network).
- Click **Test Connection** after saving.

**`pg_dump` fails**
- Confirm the PostgreSQL container name matches the running container: `docker ps`
- Verify `/var/run/docker.sock` is mounted.
- Double-check the PostgreSQL password.

**SMB connection fails**
- Test the share credentials from another machine first.
- Ensure the SMB share allows the configured username and password.

**SFTP connection fails**
- Verify the remote path exists and the user has write permission.
- Check that the host is reachable on the configured port.

**Files are being re-copied on every run**
- This happens when destination mtimes are not preserved. Check that the destination filesystem supports mtime. For SMB this depends on your server configuration.

**Container won't start**
- Check logs: `docker compose logs immichkeepr`
- Ensure the `./data` directory exists and is writable by Docker.
