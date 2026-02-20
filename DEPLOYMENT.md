# FrappeHR 15 — Deployment Guide (TrueNAS Scale)

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Phase 1 — Build the Custom Docker Image](#2-phase-1--build-the-custom-docker-image)
3. [Phase 2 — TrueNAS Dataset Setup](#3-phase-2--truenas-dataset-setup)
4. [Phase 3 — Configure Environment](#4-phase-3--configure-environment)
5. [Phase 4 — Deploy the Stack](#5-phase-4--deploy-the-stack)
6. [Phase 5 — Create the Frappe Site](#6-phase-5--create-the-frappe-site)
7. [Phase 6 — Reverse Proxy Configuration](#7-phase-6--reverse-proxy-configuration)
8. [Verification Checklist](#8-verification-checklist)
9. [Updating the Stack](#9-updating-the-stack)

---

## 1. Prerequisites

### TrueNAS Scale

- TrueNAS Scale 24.04 (Dragonfish) or newer (Docker-based app engine)
- At least one storage pool configured with free space for datasets
- SSH access to the TrueNAS shell

### Networking

- A domain or subdomain pointed at your TrueNAS IP (e.g. `hr.yourdomain.com`)
- An external reverse proxy managing HTTPS — Nginx Proxy Manager (NPM) is recommended
- Port `8080` (or your chosen `HTTP_PUBLISH_PORT`) reachable by the reverse proxy

### Build machine (any machine with Docker installed)

- Docker 24+ with BuildKit enabled
- Internet access to pull from GitHub and Docker Hub
- The build machine can be your local Mac, a Linux VM, or TrueNAS itself via SSH

---

## 2. Phase 1 — Build the Custom Docker Image

FrappeHR is not included in the base `frappe/erpnext` image. You must build a custom image
that bundles **frappe + erpnext + payments + hrms**.

### 2a. Clone the official frappe_docker repo

> __Important:__ Clone `frappe_docker` __outside__ this project directory. It is a build
> dependency, not part of this project. It is excluded by `.gitignore`.

```bash
# Clone alongside this project, not inside it
cd ~   # or wherever you keep projects
git clone https://github.com/frappe/frappe_docker
cd frappe_docker
```

### 2b. Encode apps.json

Reference the `apps.json` from this project — no need to copy it:

```bash
# macOS  (no -w flag needed)
export APPS_JSON_BASE64=$(base64 -i ~/Projects/hrms/apps.json)

# Linux / TrueNAS shell
export APPS_JSON_BASE64=$(base64 -w 0 ~/Projects/hrms/apps.json)
```

Verify it decoded correctly:

```bash
echo $APPS_JSON_BASE64 | base64 --decode
```

### 2c. Build the image

```bash
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-15 \
  --build-arg=APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --tag=custom-hrms:15 \
  --file=images/layered/Containerfile \
  --no-cache \
  .
```

> This pulls ~2 GB from GitHub and takes 15–40 minutes depending on network and CPU.

### 2d. Transfer the image to TrueNAS (if built on another machine)

**Option A — Save/load (no registry needed):**

```bash
# On build machine
docker save custom-hrms:15 | gzip > custom-hrms-15.tar.gz

# Copy to TrueNAS
scp custom-hrms-15.tar.gz root@<truenas-ip>:/mnt/<pool>/tmp/

# On TrueNAS (via SSH)
docker load < /mnt/<pool>/tmp/custom-hrms-15.tar.gz
```

**Option B — Push to a private registry (Docker Hub / GHCR):**

```bash
# Tag for your registry
docker tag custom-hrms:15 youruser/custom-hrms:15

# Push
docker push youruser/custom-hrms:15
```

Then set in `.env`:

```sh
CUSTOM_IMAGE=youruser/custom-hrms
PULL_POLICY=always
```

---

## 3. Phase 2 — TrueNAS Dataset Setup

### 3a. Recommended dataset layout

Create these three datasets under your chosen pool. Using separate datasets gives you
independent ZFS snapshot schedules for database vs. application files.

```ini
pool
└── apps
    └── hrms
        ├── mariadb      ← MariaDB InnoDB data files
        ├── redis        ← Redis queue persistence
        └── sites        ← Frappe sites (uploads, config, backups)
```

### 3b. Create datasets via TrueNAS UI

1. Go to **Storage → Create Dataset**
2. Create `apps/hrms/mariadb` with:
   - **Record Size**: 16K (matches InnoDB page size)
   - **Sync**: Standard
   - **Compression**: lz4

3. Create `apps/hrms/redis` with defaults
4. Create `apps/hrms/sites` with defaults

### 3c. Create the directories on the filesystem

SSH into TrueNAS and run:

```bash
mkdir -p /mnt/<pool>/apps/hrms/mariadb
mkdir -p /mnt/<pool>/apps/hrms/redis
mkdir -p /mnt/<pool>/apps/hrms/sites
```

Replace `<pool>` with your actual pool name (e.g. `tank`, `data`, `ssd`).

### 3d. Fix dataset ownership (required for TrueNAS bind mounts)

ZFS datasets are created owned by `root` by default. The containers run as
non-root users and will fail to write if the ownership is wrong:

| Directory | Container user | UID |
|-----------|---------------|-----|
| `sites/`  | `frappe`      | 1000 |
| `redis/`  | `redis`       | 999  |
| `mariadb/`| `mysql`       | 999  |

The `compose.yaml` configurator automatically fixes ownership of the `sites/`
directory on every startup, so that one takes care of itself.

For `mariadb/` and `redis/`, set ownership once before the first `docker compose up`:

```bash
# MariaDB data — owned by mysql (UID 999)
chown -R 999:999 /mnt/<pool>/apps/hrms/mariadb

# Redis data — owned by redis (UID 999)
chown -R 999:999 /mnt/<pool>/apps/hrms/redis
```

**Note:** If you prefer to manage all three manually (instead of letting
compose fix `sites/`), also run:

```bash
chown -R 1000:1000 /mnt/<pool>/apps/hrms/sites
```

---

## 4. Phase 3 — Configure Environment

### 4a. Copy the project files to TrueNAS

```bash
scp -r /path/to/hrms root@<truenas-ip>:/mnt/<pool>/apps/hrms/project/
```

Or clone from your version control system.

### 4b. Create `.env` from the template

```bash
cd /mnt/<pool>/apps/hrms/project
cp .env.example .env
```

### 4c. Edit `.env`

Open `.env` and set every value:

| Variable | Description | Example |
|----------|-------------|---------|
| `CUSTOM_IMAGE` | Image name you built/tagged | `custom-hrms` |
| `CUSTOM_TAG` | Image tag | `15` |
| `PULL_POLICY` | `missing` (local) or `always` (registry) | `missing` |
| `DB_ROOT_PASSWORD` | Strong MariaDB root password | `Str0ng!P@ssword` |
| `SITE_NAME` | Your domain — must match what the proxy forwards | `hr.yourdomain.com` |
| `FRAPPE_SITE_NAME_HEADER` | Usually `$$host` or hardcode your domain | `$$host` |
| `ADMIN_PASSWORD` | FrappeHR admin UI password | `Admin!2025` |
| `HTTP_PUBLISH_PORT` | Port exposed on TrueNAS host | `8080` |
| `UPSTREAM_REAL_IP_ADDRESS` | IP of your reverse proxy | `192.168.1.10` |
| `TRUENAS_DB_PATH` | Absolute path to mariadb dataset | `/mnt/tank/apps/hrms/mariadb` |
| `TRUENAS_REDIS_PATH` | Absolute path to redis dataset | `/mnt/tank/apps/hrms/redis` |
| `TRUENAS_SITES_PATH` | Absolute path to sites dataset | `/mnt/tank/apps/hrms/sites` |

> **Security**: Never commit `.env` to version control. The `.env.example` is safe to commit.

---

## 5. Phase 4 — Deploy the Stack

All commands run on TrueNAS via SSH from the project directory.

### 5a. Start the stack

```bash
docker compose -f compose.yaml -f compose.truenas.yaml up -d
```

### 5b. Watch startup progress

```bash
# Follow all logs
docker compose logs -f

# Watch just the configurator (exits when done)
docker compose logs -f configurator
```

The configurator container runs once, writes global Frappe config, then exits with code 0.
All other services wait for it before starting.

### 5c. Verify all services are running

```bash
docker compose ps
```

Expected output — all services should be `running` (configurator will show `exited 0`):

```md
NAME                 IMAGE              STATUS
hrms-backend-1       custom-hrms:15     running
hrms-db-1            mariadb:10.11      running (healthy)
hrms-frontend-1      custom-hrms:15     running
hrms-queue-long-1    custom-hrms:15     running
hrms-queue-short-1   custom-hrms:15     running
hrms-redis-cache-1   redis:7-alpine     running
hrms-redis-queue-1   redis:7-alpine     running
hrms-scheduler-1     custom-hrms:15     running
hrms-websocket-1     custom-hrms:15     running
hrms-configurator-1  custom-hrms:15     exited (0)
```

---

## 6. Phase 5 — Create the Frappe Site

This step is run **once** after the first deployment. It initialises the database and installs
ERPNext and HRMS on your site.

```bash
docker compose exec backend bench new-site hr.yourdomain.com \
  --db-root-password "$(grep DB_ROOT_PASSWORD .env | cut -d= -f2)" \
  --install-app erpnext \
  --install-app hrms \
  --admin-password "$(grep ADMIN_PASSWORD .env | cut -d= -f2)"
```

> Replace `hr.yourdomain.com` with the value of `SITE_NAME` in your `.env`.

This takes 5–15 minutes. It will:

1. Create the MariaDB database for your site
2. Run all Frappe framework migrations
3. Run all ERPNext migrations
4. Run all HRMS migrations
5. Set the admin password

### 6b. Post-install steps (required after bench new-site)

**Step 1 — Fix the MariaDB user grant (critical)**

Frappe creates the site's DB user locked to the backend container's current IP address.
Docker reassigns container IPs on every stack restart, so the grant breaks immediately.
Change the host to `%` (any Docker-network host) right after site creation:

```bash
# Get the DB username and password Frappe generated
docker compose exec backend \
  cat /home/frappe/frappe-bench/sites/hr.yourdomain.com/site_config.json
# Note the values of "db_name" (= DB user) and "db_password"

# Fix the grant — replace <db_user> and <db_password> from above
docker compose exec db \
  mysql -uroot -p"${DB_ROOT_PASSWORD}" -e "
RENAME USER '<db_user>'@'<current_ip>' TO '<db_user>'@'%';
ALTER USER '<db_user>'@'%' IDENTIFIED BY '<db_password>';
FLUSH PRIVILEGES;
"
```

Or run it non-interactively using shell substitution:

```bash
DB_USER=$(docker compose exec backend \
  python3 -c "import json,sys; d=json.load(open('/home/frappe/frappe-bench/sites/hr.yourdomain.com/site_config.json')); print(d['db_name'])")

DB_PASS=$(docker compose exec backend \
  python3 -c "import json,sys; d=json.load(open('/home/frappe/frappe-bench/sites/hr.yourdomain.com/site_config.json')); print(d['db_password'])")

CURRENT_IP=$(docker compose exec db \
  mysql -uroot -p"${DB_ROOT_PASSWORD}" -sNe \
  "SELECT Host FROM mysql.user WHERE User='${DB_USER}';")

docker compose exec db \
  mysql -uroot -p"${DB_ROOT_PASSWORD}" -e "
RENAME USER '${DB_USER}'@'${CURRENT_IP}' TO '${DB_USER}'@'%';
ALTER USER '${DB_USER}'@'%' IDENTIFIED BY '${DB_PASS}';
FLUSH PRIVILEGES;"

```

OR — Recreate the missing DB user manually

```bash
 docker compose exec db \
mysql -uroot -p"${DB_ROOT_PASSWORD}" -e "
CREATE USER '_1ac53aa32737700d'@'%' IDENTIFIED BY 'O5dknrg6FintNohG';
GRANT ALL PRIVILEGES ON \`_1ac53aa32737700d\`.* TO '_1ac53aa32737700d'@'%';
FLUSH PRIVILEGES;"
```

and — Also allow localhost login (Frappe needs this)
```bash
docker compose exec db \
mysql -uroot -p"${DB_ROOT_PASSWORD}" -e "
CREATE USER '_1ac53aa32737700d'@'localhost' IDENTIFIED BY 'O5dknrg6FintNohG';
GRANT ALL PRIVILEGES ON \`_1ac53aa32737700d\`.* TO '_1ac53aa32737700d'@'localhost';
FLUSH PRIVILEGES;"
```
**Step 2 — Enable the scheduler and set default site**

```bash
# Enable the Frappe scheduler for your site
docker compose exec backend bench --site hr.yourdomain.com enable-scheduler

# Set this site as the default (avoids workers guessing which site to serve)
docker compose exec backend bench use hr.yourdomain.com
```

Expected warnings you can safely ignore:

- `MariaDB version 10.11 > 10.8 not yet tested` — cosmetic; 10.11 is fully supported
- `rename_field: kra_title not found` — harmless data migration notice, no existing data
- `io_uring_queue_init() failed with EPERM` — macOS/Docker Desktop only; falls back to libaio

### 6c. Verify the site is accessible

```bash
# From TrueNAS, hit the frontend directly (replace 8080 with your HTTP_PUBLISH_PORT)
curl -I http://localhost:8080
# Expected: HTTP/1.1 200 OK (or 301 redirect to login)
```

---

## 7. Phase 6 — Reverse Proxy Configuration

### Nginx Proxy Manager (NPM)

1. In NPM, go to **Proxy Hosts → Add Proxy Host**
2. Fill in:
   - **Domain Names**: `hr.yourdomain.com`
   - **Scheme**: `http`
   - **Forward Hostname/IP**: `<truenas-ip>`
   - __Forward Port__: `8080` (your `HTTP_PUBLISH_PORT`)
   - **Websockets Support**: ✅ Enabled (required for real-time features)

3. On the **SSL** tab: request a Let's Encrypt certificate
4. Enable **Force SSL**

### Required proxy headers

NPM passes `X-Forwarded-For` and `X-Real-IP` automatically. For Frappe to log the real
client IP, set in `.env`:

```sh
UPSTREAM_REAL_IP_ADDRESS=<your-npm-container-or-host-ip>
UPSTREAM_REAL_IP_HEADER=X-Forwarded-For
```

Then restart the frontend:

```bash
docker compose restart frontend
```

---

## 8. Verification Checklist

- [ ] `docker compose ps` shows all services `running`
- [ ] `curl -I http://<truenas-ip>:8080` returns HTTP 200
- [ ] `https://hr.yourdomain.com` loads the Frappe login page
- [ ] Login with `Administrator` and your `ADMIN_PASSWORD` succeeds
- [ ] HR module appears in the app list
- [ ] Real-time notifications work (WebSocket connected — check browser DevTools → Network → WS)

---

## 9. Updating the Stack

### Update FrappeHR apps (patch releases within version-15)

```bash
# 1. Rebuild the image (same build command as Phase 1)
docker build ... --tag=custom-hrms:15 --no-cache .

# 2. If built on another machine, transfer/push the new image

# 3. Pull and restart
docker compose -f compose.yaml -f compose.truenas.yaml pull
docker compose -f compose.yaml -f compose.truenas.yaml up -d

# 4. Run migrations
docker compose exec backend bench --site hr.yourdomain.com migrate
```

### Update MariaDB or Redis

Edit the image tag in `compose.yaml`, then:

```bash
docker compose -f compose.yaml -f compose.truenas.yaml pull
docker compose -f compose.yaml -f compose.truenas.yaml up -d
```

> Always take a backup before any update. See `BACKUP.md`.
