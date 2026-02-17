# FrappeHR 15 — Backup & Restore Guide

## Overview

Three independent backup layers are recommended. Use all three for full protection:

| Layer | Tool | What it covers | Recovery speed |
|-------|------|----------------|----------------|
| **1 — Frappe bench backup** | `bench backup` | DB dump + uploaded files | Fast (app-level) |
| **2 — ZFS snapshots** | TrueNAS UI | All dataset data at point-in-time | Very fast (block-level) |
| **3 — ZFS replication** | TrueNAS UI | Snapshots copied to remote pool/server | Fast after initial sync |

Layer 1 is application-aware and produces portable, restorable `.sql.gz` + file archives.
Layers 2 and 3 are filesystem-level and protect against hardware failure or accidental deletion.

---

## Table of Contents

1. [Layer 1 — Frappe bench backup](#1-layer-1--frappe-bench-backup)
2. [Layer 2 — ZFS Snapshots (TrueNAS)](#2-layer-2--zfs-snapshots-truenas)
3. [Layer 3 — ZFS Replication (off-site)](#3-layer-3--zfs-replication-off-site)
4. [Manual MariaDB dump](#4-manual-mariadb-dump)
5. [Restore Procedures](#5-restore-procedures)
6. [Backup Verification](#6-backup-verification)
7. [Recommended Schedule](#7-recommended-schedule)

---

## 1. Layer 1 — Frappe bench backup

### What `bench backup` produces

Frappe stores backups inside the container volume at:
```
sites/<sitename>/private/backups/
```
Which maps to your TrueNAS dataset at:
```
/mnt/<pool>/apps/hrms/sites/<sitename>/private/backups/
```

Each backup run creates three files:
```
20250218_120000-<sitename>-database.sql.gz    ← compressed DB dump
20250218_120000-<sitename>-private-files.tar  ← private uploads
20250218_120000-<sitename>-files.tar          ← public uploads
```

### Run a manual backup

```bash
cd /mnt/<pool>/apps/hrms/project

docker compose exec backend \
  bench --site hr.yourdomain.com backup --with-files
```

### Automate daily backups

Frappe's **scheduler** service automatically runs `bench backup` for all sites every day
as long as the `scheduler` container is running. You can verify it in the Frappe UI:

1. Log in as Administrator
2. Go to **Settings → System Settings → Backups**
3. Confirm **Take Backup Every** is set (default: 1 day)
4. Set **Number of Backups to Keep** (default: 3 — increase to 7 or more)

### Copy backups off the container volume

The backup files are already on the `sites` dataset at:
```
/mnt/<pool>/apps/hrms/sites/hr.yourdomain.com/private/backups/
```

You can sync them to another location with `rsync`:

```bash
# Example: sync to a NAS share or external backup server
rsync -avz --delete \
  /mnt/<pool>/apps/hrms/sites/hr.yourdomain.com/private/backups/ \
  backup-server:/backups/hrms/
```

Add this to a TrueNAS **Cron Job** (System → Advanced → Cron Jobs) to run after the
Frappe daily backup window (e.g. daily at 03:00).

---

## 2. Layer 2 — ZFS Snapshots (TrueNAS)

ZFS snapshots are instant and space-efficient (copy-on-write). They protect all three
datasets — database, redis, and sites — independently.

### 2a. Create snapshot tasks via TrueNAS UI

1. Go to **Data Protection → Periodic Snapshot Tasks → Add**
2. Create one task per dataset:

**MariaDB dataset** (`apps/hrms/mariadb`):
- Recursive: No
- Schedule: Every hour (`0 * * * *`)
- Keep: 24 hourly, 7 daily, 4 weekly

**Sites dataset** (`apps/hrms/sites`):
- Recursive: No
- Schedule: Every 4 hours (`0 */4 * * *`)
- Keep: 7 daily, 4 weekly, 3 monthly

**Redis dataset** (`apps/hrms/redis`):
- Recursive: No
- Schedule: Daily (`0 2 * * *`)
- Keep: 7 daily

### 2b. Manually create a snapshot before maintenance

```bash
# SSH into TrueNAS
zfs snapshot tank/apps/hrms/mariadb@pre-update-$(date +%Y%m%d)
zfs snapshot tank/apps/hrms/sites@pre-update-$(date +%Y%m%d)
```

### 2c. List existing snapshots

```bash
zfs list -t snapshot -o name,creation,used tank/apps/hrms
```

---

## 3. Layer 3 — ZFS Replication (off-site)

Replication sends snapshots to a remote TrueNAS instance or another pool on the same machine.

### Via TrueNAS UI

1. Go to **Data Protection → Replication Tasks → Add**
2. Source datasets: `apps/hrms/mariadb`, `apps/hrms/sites`
3. Destination: remote TrueNAS SSH endpoint or local second pool
4. Replication schedule: Daily (after snapshot tasks run)
5. Enable **Encryption** for remote replication

> For cloud replication, TrueNAS supports Backblaze B2, Amazon S3, and others
> via **Cloud Sync Tasks** (Data Protection → Cloud Sync Tasks).

---

## 4. Manual MariaDB dump

Use this when you need a database dump independent of Frappe bench, or for migrating
to a different server.

```bash
cd /mnt/<pool>/apps/hrms/project

# Full dump of the site database
docker compose exec db \
  mysqldump \
    --user=root \
    --password="$(grep DB_ROOT_PASSWORD .env | cut -d= -f2)" \
    --single-transaction \
    --routines \
    --triggers \
    hr_yourdomain_com \
  | gzip > /mnt/<pool>/apps/hrms/manual-backup-$(date +%Y%m%d_%H%M%S).sql.gz
```

> The database name is your site name with dots replaced by underscores.
> `hr.yourdomain.com` → `hr_yourdomain_com`

List databases to confirm the exact name:
```bash
docker compose exec db \
  mysql -uroot -p"$(grep DB_ROOT_PASSWORD .env | cut -d= -f2)" \
  -e "SHOW DATABASES;"
```

---

## 5. Restore Procedures

### 5a. Restore from Frappe bench backup

Use this when you have a bench backup archive (`.sql.gz` + `.tar` files).

```bash
cd /mnt/<pool>/apps/hrms/project

# List available backups
ls /mnt/<pool>/apps/hrms/sites/hr.yourdomain.com/private/backups/

# Restore — specify the database file; Frappe finds the matching file archives
docker compose exec backend \
  bench --site hr.yourdomain.com restore \
  /home/frappe/frappe-bench/sites/hr.yourdomain.com/private/backups/20250218_120000-hr.yourdomain.com-database.sql.gz \
  --with-public-files \
  /home/frappe/frappe-bench/sites/hr.yourdomain.com/private/backups/20250218_120000-hr.yourdomain.com-files.tar \
  --with-private-files \
  /home/frappe/frappe-bench/sites/hr.yourdomain.com/private/backups/20250218_120000-hr.yourdomain.com-private-files.tar

# Run migrations after restore
docker compose exec backend \
  bench --site hr.yourdomain.com migrate
```

### 5b. Restore from ZFS snapshot (same machine)

Use this for quick rollback after a bad update or accidental deletion.

```bash
# Stop the stack first
docker compose -f compose.yaml -f compose.truenas.yaml down

# Roll back the MariaDB dataset to the snapshot
zfs rollback tank/apps/hrms/mariadb@pre-update-20250218

# Roll back the sites dataset
zfs rollback tank/apps/hrms/sites@pre-update-20250218

# Restart the stack
docker compose -f compose.yaml -f compose.truenas.yaml up -d
```

### 5c. Full disaster recovery (new TrueNAS machine)

1. **Restore datasets from replication** (TrueNAS UI → Replication → Run manually)
   or from a ZFS send/receive stream:
   ```bash
   # On the source (or from a backup file)
   zfs send tank/apps/hrms/mariadb@snapshot | zfs receive newtank/apps/hrms/mariadb
   zfs send tank/apps/hrms/sites@snapshot   | zfs receive newtank/apps/hrms/sites
   ```

2. **Transfer the image** to the new server (see Deployment Guide Phase 1d)

3. **Copy the project files** (compose files + `.env`) to the new server

4. **Update `.env`** with the new server's dataset paths

5. **Start the stack**:
   ```bash
   docker compose -f compose.yaml -f compose.truenas.yaml up -d
   ```

6. **Verify** the site is accessible and data is intact

---

## 6. Backup Verification

A backup you have never tested is not a backup. Test monthly:

### Test bench backup restore

```bash
# 1. Spin up a test instance (use named volumes, not TrueNAS datasets)
docker compose up -d

# 2. Create a test site
docker compose exec backend bench new-site test.local \
  --db-root-password <password> \
  --install-app erpnext --install-app hrms \
  --admin-password <password>

# 3. Restore a production backup into the test site
docker compose exec backend \
  bench --site test.local restore \
  /home/frappe/frappe-bench/sites/hr.yourdomain.com/private/backups/<latest-backup>.sql.gz

# 4. Verify data looks correct, then tear down
docker compose down -v
```

### Verify ZFS snapshot integrity

```bash
# Clone a dataset from a snapshot without affecting production
zfs clone tank/apps/hrms/mariadb@daily-2025-02-18 tank/hrms-test/mariadb

# Start a test MariaDB pointing at the clone
# ... verify the data ...

# Clean up
zfs destroy tank/hrms-test/mariadb
```

---

## 7. Recommended Schedule

| Task | Frequency | Tool |
|------|-----------|------|
| Frappe bench backup (DB + files) | Every 6 hours | Frappe Scheduler (automatic) |
| Rsync bench backups to NAS/remote | Daily at 03:30 | TrueNAS Cron Job |
| ZFS snapshot — mariadb dataset | Hourly | TrueNAS Periodic Snapshot |
| ZFS snapshot — sites dataset | Every 4 hours | TrueNAS Periodic Snapshot |
| ZFS replication to remote | Daily at 04:00 | TrueNAS Replication Task |
| Manual backup before any update | Before each update | Manual |
| Restore test | Monthly | Manual |
