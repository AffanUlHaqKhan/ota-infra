# Server storage: SD now, NVMe later

The OTA server runs from an SD card for the moment. That is fine for a bench
fleet, but SD cards fail by wearing out under writes, not by filling up — so the
work is in reducing writes and in making the eventual move to NVMe a
non-event.

## Everything lives in one directory

```
${OTA_DATA_ROOT:-/srv/ota}/
  repo/     OSTree objects — what devices pull
  db/       ota.db (SQLite)
```

`deploy/compose.yml` bind-mounts that path rather than using Docker named
volumes. Named volumes live inside `/var/lib/docker/volumes`, so moving them to
new storage means relocating Docker's whole data root or hand-copying volumes.
With a bind mount, migration is a copy and a variable.

## Measured sizes

| Thing | Size |
|---|---|
| OSTree repo, several commits (5596 objects) | **127 MB** |
| One uncompressed rootfs | 973 MB |
| `wic.gz` (flashing artifact, not stored server-side) | 137 MB |

The repo is content-addressed, so unchanged files are shared between commits.
Growth per release is tens of MB unless the kernel or Qt changes. A year of
releases is single-digit GB. **Capacity is not the constraint.**

## Why SQLite rather than postgres

The server uses stdlib `sqlite3`. On an SD card that is a feature, not a
compromise: one file, no server process, and far less write amplification than
postgres, whose WAL plus full-page writes are close to the worst case for
flash. It is also one less container on an 8GB Pi.

The limit is a single writer. At one device — or a few hundred — that is
nowhere near binding. Revisit when concurrent writes actually contend.

## Reducing wear while on SD

Worth doing, roughly in order of payoff:

```bash
# 1. Don't write access times on every read.
#    In /etc/fstab, add `noatime` to the root mount.

# 2. Cap the systemd journal, or keep it in RAM entirely.
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=volatile\nRuntimeMaxUse=64M\n' \
  | sudo tee /etc/systemd/journald.conf.d/volatile.conf
sudo systemctl restart systemd-journald

# 3. Put the Docker build cache somewhere disposable, or prune it regularly.
docker builder prune -f
```

Do **not** disable SQLite's WAL or set `synchronous=OFF` to save writes. That
trades durability for wear, and a corrupted database is a worse day than a worn
card.

## Back up the two things that matter

The repo is regenerable from a Yocto build; the database is not.

```bash
# Database — SQLite's own consistent backup, safe while the server is running.
sqlite3 "$OTA_DATA_ROOT/db/ota.db" ".backup '/path/to/backup/ota-$(date +%F).db'"

# Repo — plain files, content-addressed, safe to rsync incrementally.
rsync -a "$OTA_DATA_ROOT/repo/" /path/to/backup/repo/
```

Losing the database loses rollout history and device grouping; devices keep
running and re-register on their next heartbeat. Losing the repo means devices
cannot update until a build is republished. Neither is fatal — unlike the root
signing key, which is the one thing with no recovery path (`DECISIONS.md` D5).

## Migrating to NVMe later

With the bind-mount layout, the whole move is:

```bash
docker compose -f deploy/compose.yml down
sudo rsync -aHAX --info=progress2 /srv/ota/ /mnt/nvme/ota/
export OTA_DATA_ROOT=/mnt/nvme/ota          # or mount the NVMe at /srv/ota
docker compose -f deploy/compose.yml up -d
curl -s localhost:8000/api/v1/fleet | head   # devices still there?
```

`-H` preserves hard links, which matters: OSTree checkouts share objects by
hard link, and copying without it can multiply the repo's size.

Keep the SD card untouched until the NVMe copy is verified — it is a free
rollback.

## Watching for the card giving up

SD cards report no SMART data, so there is no health counter to read. What you
get instead is filesystem errors appearing in the journal, usually as read-only
remounts:

```bash
journalctl -k | grep -iE "ext4.*(error|remount|readonly)|mmcblk.*error"
```

Treat the first such line as the card telling you it is done. Have a backup
before that day, not after.
