# Deploying Hyperion: GitHub Actions CI/CD, Docker on Hostinger, and an EC2 Solar Sensor

*By Nikhil Roy — May 2026*

---

Hyperion is a MERN + TailwindCSS v4 solar panel dashboard I built for home consumers to monitor live device output, energy history, and maintenance status. This post documents everything I did to take it from a local dev environment to a fully live deployment: containerising the app, setting up a CI/CD pipeline through GitHub Actions, deploying to a Hostinger VPS with Nginx Proxy Manager, and running a Python sensor agent on AWS EC2 that simulates (and will eventually read real) solar hardware. I'm writing this both as a public record and as personal reference notes — so every file, every fix, every mistake is in here.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Phase 1 — EC2 Sensor Agent](#2-phase-1--ec2-sensor-agent)
3. [Phase 2 — Dockerising the App](#3-phase-2--dockerising-the-app)
4. [Phase 3 — First CI/CD Attempt: SSH from GitHub Actions](#4-phase-3--first-cicd-attempt-ssh-from-github-actions)
5. [Phase 4 — Nginx Proxy Manager + Self-Hosted Runner](#5-phase-4--nginx-proxy-manager--self-hosted-runner)
6. [Bugs Hit in Production](#6-bugs-hit-in-production)
   - [Bug 1: Docker DNS Caches Stale Container IPs](#bug-1-docker-dns-caches-stale-container-ips)
   - [Bug 2: Double `/api/` Path After DNS Fix](#bug-2-double-api-path-after-dns-fix)
   - [Bug 3: Ingest Secret Auth Failing (Header vs Body)](#bug-3-ingest-secret-auth-failing-header-vs-body)
7. [Sensor Simulation Upgrade](#7-sensor-simulation-upgrade)
8. [Final State — All Key Files](#8-final-state--all-key-files)
9. [Quick-Reference Checklist](#9-quick-reference-checklist)

---

## 1. Architecture Overview

```
GitHub (main branch)
    │
    │  push triggers GitHub Actions
    ▼
Self-Hosted Runner (on Hostinger VPS)
    │  runs /opt/hyperion/deploy/deploy.sh
    │  git pull + docker compose up -d --build
    ▼
Hostinger VPS (Ubuntu)
    ├── Nginx Proxy Manager (Docker)  ← handles TLS + routing
    │       │
    │       │  proxies to client container on web-network
    │       ▼
    ├── client container  (nginx:alpine, serves React SPA)
    │       │  /api/* proxied to server:5000
    │       ▼
    ├── server container  (node:20-alpine, Express API)
    │       │  reads/writes Redis
    │       ▼
    └── redis container   (redis:7-alpine, named volume)

AWS EC2 (Amazon Linux 2023)
    └── sensor container  (python:3.12-slim)
            │  POST /api/ingest every 30s
            │  x-ingest-secret header
            ▼
        https://hyperion.nikhilroy.com/api/ingest
```

The two-stage telemetry pipeline in the backend:

- `POST /api/ingest` — secret-key authenticated (no JWT), writes to Redis accumulators: live state (`telemetry:current` HSET, 90s TTL), rolling readings list, running watt sum, peak tracker.
- `GET /api/telemetry/live` — reads `telemetry:current` from Redis.
- `node-cron` job at 00:01 daily — aggregates Redis today keys into MongoDB `TelemetryDaily`, then DELs the keys.

---

## 2. Phase 1 — EC2 Sensor Agent

The sensor lives on a separate AWS EC2 instance entirely. In a real product this would be a Raspberry Pi or embedded Linux device sitting next to the solar inverter, but for this project EC2 gives us a stable always-on Linux box we can SSH into and control. The sensor's only job is to read a wattage value and POST it to the backend every 30 seconds.

### The Python Ingest Script — `sensor/ingest.py`

```python
#!/usr/bin/env python3
import os, time, csv, math, random, datetime, requests

API_URL        = os.environ["HYPERION_API_URL"]
INGEST_SECRET  = os.environ["HYPERION_INGEST_SECRET"]
DEVICE_ID      = os.environ["HYPERION_DEVICE_ID"]
CSV_PATH       = os.environ.get("HYPERION_CSV_PATH", "/var/log/hyperion/readings.csv")
PEAK_WATTS     = float(os.environ.get("HYPERION_PEAK_WATTS", "3500"))
SUNRISE_HOUR   = float(os.environ.get("HYPERION_SUNRISE_HOUR", "6"))
SUNSET_HOUR    = float(os.environ.get("HYPERION_SUNSET_HOUR", "18"))
INTERVAL       = 30  # seconds

_cloud_factor  = 1.0
_cloud_ticks   = 0

def read_sensor() -> float:
    global _cloud_factor, _cloud_ticks

    hour = datetime.datetime.now(datetime.timezone.utc).hour + \
           datetime.datetime.now(datetime.timezone.utc).minute / 60.0

    if hour < SUNRISE_HOUR or hour > SUNSET_HOUR:
        return 0.0

    progress = (hour - SUNRISE_HOUR) / (SUNSET_HOUR - SUNRISE_HOUR)
    base = math.sin(math.pi * progress) * PEAK_WATTS

    if _cloud_ticks <= 0:
        if random.random() < 0.08:
            _cloud_factor = random.uniform(0.2, 0.7)
            _cloud_ticks  = random.randint(2, 6)
        else:
            _cloud_factor = 1.0
    else:
        _cloud_ticks -= 1

    noise = random.uniform(-0.03, 0.03)
    return max(0.0, round(base * _cloud_factor * (1 + noise), 1))

def main():
    os.makedirs(os.path.dirname(CSV_PATH), exist_ok=True)
    while True:
        now = datetime.datetime.now(datetime.timezone.utc).isoformat().replace("+00:00", "Z")
        watts = read_sensor()
        payload = {"deviceId": DEVICE_ID, "outputW": watts, "timestamp": now, "secret": INGEST_SECRET}
        try:
            r = requests.post(API_URL, json=payload,
                              headers={"x-ingest-secret": INGEST_SECRET}, timeout=10)
            r.raise_for_status()
            print(f"[{now}] OK — {watts}W")
        except Exception as e:
            print(f"[{now}] POST failed: {e}")
        with open(CSV_PATH, "a", newline="") as f:
            csv.writer(f).writerow([now, watts])
        time.sleep(INTERVAL)

if __name__ == "__main__":
    main()
```

Key design decisions:

- Everything is configured via environment variables — no hardcoding.
- The CSV at `/var/log/hyperion/readings.csv` is a local backup that persists even if the API is unreachable.
- The secret is sent in **both** the `x-ingest-secret` header and the `secret` body field — this turned out to be important later (see Bug 3).
- `datetime.timezone.utc` instead of `datetime.utcnow()` — the latter is deprecated in Python 3.12.

### Sensor Dockerfile — `sensor/Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY ingest.py ./
VOLUME ["/var/log/hyperion"]
ENV PYTHONUNBUFFERED=1
CMD ["python", "ingest.py"]
```

`PYTHONUNBUFFERED=1` is critical — without it, Python buffers stdout and `docker logs` shows nothing until the buffer fills. I missed this initially and spent time wondering why `journalctl` was silent.

### Sensor Docker Compose — `sensor/docker-compose.yml`

```yaml
services:
  sensor:
    build: .
    restart: unless-stopped
    env_file: sensor.env
    volumes:
      - sensor_logs:/var/log/hyperion

volumes:
  sensor_logs:
```

No ports exposed — the sensor only makes outbound HTTPS calls.

### Sensor Environment — `sensor/sensor.env.example`

```bash
# On Amazon Linux: sudo usermod -aG docker ec2-user  (then log out and back in)
HYPERION_API_URL=https://yourdomain.com/api/ingest
HYPERION_INGEST_SECRET=<value from server .env>
HYPERION_DEVICE_ID=<deviceId from MongoDB>
# Optional — defaults shown
# HYPERION_CSV_PATH=/var/log/hyperion/readings.csv
# HYPERION_PEAK_WATTS=3500
# HYPERION_SUNRISE_HOUR=6
# HYPERION_SUNSET_HOUR=18
```

Copy this to `sensor.env` and fill in real values. The `DEVICE_ID` comes from the `Device` document in MongoDB — you need to insert a device record first via the API or directly in Mongo.

### Systemd Service — `sensor/hyperion-sensor.service`

```ini
[Unit]
Description=Hyperion Solar Sensor Ingest
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=ec2-user
WorkingDirectory=/opt/hyperion/sensor
ExecStart=/usr/bin/docker compose up
ExecStop=/usr/bin/docker compose down
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
LogRateLimitIntervalSec=30
LogRateLimitBurst=5

[Install]
WantedBy=multi-user.target
```

### EC2 Setup Steps (Amazon Linux 2023)

```bash
# 1. Install Docker
sudo yum update -y
sudo yum install -y docker
sudo systemctl enable docker --now

# 2. Allow ec2-user to run Docker without sudo
sudo usermod -aG docker ec2-user
# Log out and back in for this to take effect

# 3. Install Docker Compose plugin
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# 4. Clone / copy sensor files to the instance
sudo mkdir -p /opt/hyperion/sensor
# scp sensor/* ec2-user@<ip>:/opt/hyperion/sensor/
# or git clone the repo and copy the sensor/ folder

# 5. Create the real sensor.env (DO NOT commit this)
cp /opt/hyperion/sensor/sensor.env.example /opt/hyperion/sensor/sensor.env
nano /opt/hyperion/sensor/sensor.env

# 6. Install and start the systemd service
sudo cp /opt/hyperion/sensor/hyperion-sensor.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable hyperion-sensor --now

# 7. Check it is running
sudo systemctl status hyperion-sensor
journalctl -u hyperion-sensor -f
```

The `LogRateLimitIntervalSec=30` and `LogRateLimitBurst=5` in the unit file are important — without them, journald throttles the sensor's 30-second log output and you get gaps in `journalctl -f`.

---

## 3. Phase 2 — Dockerising the App

With the sensor in place I needed to containerise the main app so it could run consistently on the Hostinger VPS.

### Server Dockerfile — `server/Dockerfile`

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 5000
CMD ["node", "index.js"]
```

`--omit=dev` keeps the image small — no devDependencies in production. Port 5000 is never exposed to the host; only other containers on the `internal` network reach it.

### Client Dockerfile — `client/Dockerfile`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

Multi-stage build: the first stage compiles the Vite/React app, the second stage is a tiny `nginx:alpine` image that serves only the compiled static files. The node_modules and build toolchain never make it into the final image.

### Nginx Config Inside the Client Container — `client/nginx.conf`

```nginx
server {
  listen 80;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }

  location /api/ {
    resolver 127.0.0.11 valid=30s;
    set $upstream http://server:5000;
    proxy_pass $upstream;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

This file went through three revisions (see Bugs 1 and 2). The `resolver 127.0.0.11` and `set $upstream` variable are the result of two separate fixes — both are necessary.

### Docker Compose — `docker-compose.yml`

```yaml
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - internal

  server:
    build: ./server
    restart: unless-stopped
    env_file: ./server/.env
    environment:
      REDIS_URL: redis://redis:6379
    depends_on:
      - redis
    networks:
      - internal

  client:
    build: ./client
    restart: unless-stopped
    depends_on:
      - server
    networks:
      - internal
      - web-network

networks:
  internal:
    driver: bridge
  web-network:
    external: true

volumes:
  redis_data:
```

Key points:

- **Two networks**: `internal` (private, bridge) for server↔redis and server↔client communication; `web-network` (external, must be pre-created) for Nginx Proxy Manager to reach the client container.
- `server` and `redis` are on `internal` only — no exposed ports. They are unreachable from outside Docker.
- `redis_data` is a named volume so data survives `docker compose down`.
- `web-network` is created once manually on the host: `docker network create web-network`.

### `.dockerignore`

```
node_modules
*/node_modules
.env
*/.env
client/dist
*.log
.git
HANDOFF.md
CLAUDE.md
```

Without this, `COPY . .` in each Dockerfile would send `node_modules` (hundreds of MB) and `.env` files into the build context — slow and a security risk.

---

## 4. Phase 3 — First CI/CD Attempt: SSH from GitHub Actions

My first CI/CD plan was straightforward: push to `main` → GitHub Actions runner SSHes into Hostinger → runs deploy script.

### GitHub Actions Workflow (v1) — `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Hostinger
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.HOSTINGER_HOST }}
          username: ${{ secrets.HOSTINGER_USER }}
          key: ${{ secrets.HOSTINGER_SSH_KEY }}
          script: bash /opt/hyperion/deploy/deploy.sh
```

The three secrets — `HOSTINGER_HOST`, `HOSTINGER_USER`, `HOSTINGER_SSH_KEY` — were added to the GitHub repository under Settings → Secrets and Variables → Actions.

### Original Deploy Script (v1) — `deploy/deploy.sh`

```bash
#!/bin/bash
set -e
cd /opt/hyperion

git pull origin main

# Restart server + Redis containers
docker compose pull redis || true
docker compose up -d --build server redis

# Build React client and copy to nginx root
cd client
npm ci
npm run build
sudo cp -r dist/* /var/www/hyperion/
sudo nginx -s reload
```

This version builds the React client natively on the host and copies files to `/var/www/hyperion` — the host nginx serves them statically. The server container runs behind the host nginx on port 5000.

### The Original Nginx Config — `deploy/nginx-hostinger.conf`

```nginx
server {
  listen 80;
  server_name yourdomain.com www.yourdomain.com;

  root /var/www/hyperion;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }

  location /api/ {
    proxy_pass http://127.0.0.1:5000/api/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

This was to be placed in `/etc/nginx/sites-available/hyperion` and symlinked to `sites-enabled/`.

### The Problem: Hostinger Blocks Inbound SSH from GitHub

This approach never fully worked. GitHub Actions runners run from a pool of IP ranges that Hostinger's firewall (or network policy) blocks for inbound SSH. The `appleboy/ssh-action` step would timeout waiting to connect. Even after whitelisting IPs in the Hostinger panel, the range is too large and dynamic to be reliable.

The fix was to flip the connection direction entirely.

---

## 5. Phase 4 — Nginx Proxy Manager + Self-Hosted Runner

I made two changes simultaneously: switched to Nginx Proxy Manager for the reverse proxy, and switched to a self-hosted GitHub Actions runner.

### Why Nginx Proxy Manager (NPM)

NPM is an open-source reverse proxy with a web UI. It runs in Docker and handles:
- Domain routing (subdomain → container)
- Let's Encrypt TLS certificate provisioning and renewal
- No manual nginx config editing needed

NPM is part of the `web-network` Docker network. Any container on that network is routable by hostname. This is the standard pattern for Docker-on-VPS setups.

**Setup on Hostinger (one-time):**

```bash
# Create the external Docker network all app stacks share
docker network create web-network

# Run Nginx Proxy Manager (typically via its own compose file in /opt/npm)
# NPM then routes based on domain names to containers on web-network
```

In NPM's UI, I created a Proxy Host:
- Domain: `hyperion.nikhilroy.com`
- Scheme: `http`
- Forward Hostname: `hyperion-client-1` (Docker container name) or just `client` (service name on web-network)
- Forward Port: `80`
- SSL: Let's Encrypt (enable Force SSL)

### Adapting Docker Compose for NPM

The old compose had server exposed on port 5000 to the host and the host nginx serving the static files. With NPM:

- `client` joins `web-network` so NPM can reach it.
- `server` and `redis` are on `internal` only — no ports exposed.
- The client container's nginx proxies `/api/` calls internally to `server:5000`.
- NPM routes external HTTPS → `client:80`.

The `docker-compose.yml` shown in Phase 2 is the final state.

### Simplified Deploy Script — `deploy/deploy.sh`

```bash
#!/bin/bash
set -e
cd /opt/hyperion

git config --global --add safe.directory /opt/hyperion
git pull origin main

docker compose up -d --build
```

One command. Docker Compose handles rebuilding changed images and restarting only affected services.

### Self-Hosted GitHub Actions Runner

Instead of GitHub SSHing into the VPS (inbound, blocked), I installed the GitHub Actions runner directly on the Hostinger VPS. The runner is a process that polls GitHub for jobs — outbound only. No firewall changes needed.

**Setup on Hostinger:**

1. Go to GitHub repo → Settings → Actions → Runners → New self-hosted runner.
2. Choose Linux, copy the installation commands.
3. On the VPS:

```bash
# Create a runner user or use existing
mkdir -p /opt/github-runner && cd /opt/github-runner

# Download and extract (use the exact URL from GitHub — it changes with versions)
curl -o actions-runner-linux-x64.tar.gz -L https://github.com/actions/runner/releases/download/v2.x.x/actions-runner-linux-x64-2.x.x.tar.gz
tar xzf actions-runner-linux-x64.tar.gz

# Configure (token from GitHub UI — expires after ~1hr)
./config.sh --url https://github.com/phy-nikhilroy/hyperion --token <TOKEN>

# Install as a systemd service so it starts on boot
sudo ./svc.sh install
sudo ./svc.sh start
```

The runner shows as "Idle" in GitHub → Settings → Actions → Runners when healthy.

### Final GitHub Actions Workflow — `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Deploy
        run: bash /opt/hyperion/deploy/deploy.sh
```

The entire workflow is now three meaningful lines. `runs-on: self-hosted` routes the job to the runner on the VPS. No checkout step needed because deploy.sh does a `git pull` directly on the already-cloned repo at `/opt/hyperion`.

**Initial clone on VPS (one-time):**

```bash
sudo mkdir -p /opt/hyperion
sudo chown $USER:$USER /opt/hyperion
git clone https://github.com/phy-nikhilroy/hyperion.git /opt/hyperion
cp /opt/hyperion/server/.env.example /opt/hyperion/server/.env
nano /opt/hyperion/server/.env  # fill in real values
```

---

## 6. Bugs Hit in Production

### Bug 1: Docker DNS Caches Stale Container IPs

**Symptom:** After restarting the `server` container (e.g. during a deploy), API calls from the React app returned `502 Bad Gateway`. The client container's nginx was still routing to the old container IP.

**Root cause:** By default nginx resolves DNS names at startup and caches them forever. `server` is a Docker DNS name that resolves to the container's internal IP. When the container is recreated (new IP), nginx still has the old one cached in memory.

**The fix:** In `client/nginx.conf`, use Docker's embedded DNS resolver (`127.0.0.11`) and force nginx to re-resolve by assigning the upstream to a variable:

```nginx
location /api/ {
  resolver 127.0.0.11 valid=30s;
  set $upstream http://server:5000;
  proxy_pass $upstream;
  ...
}
```

The key insight: nginx only uses the `resolver` directive and re-resolves periodically when the upstream is set via a **variable** (`set $upstream`). If you write `proxy_pass http://server:5000` directly (a literal), nginx ignores the resolver for that directive. The `valid=30s` means it re-resolves at most every 30 seconds. This also drops the now-obsolete `version: '3'` field from `docker-compose.yml` (a warning since Compose v2).

**Commit:** `fix(docker): use Docker embedded DNS resolver in nginx and drop obsolete version` (f759ae5)

---

### Bug 2: Double `/api/` Path After DNS Fix

**Symptom:** After the DNS fix above, all API requests returned 404. Server logs showed requests arriving at `/api//api/route`.

**Root cause:** This is a subtle nginx behaviour with variables in `proxy_pass`.

When I first introduced the variable approach, I wrote:

```nginx
# WRONG
proxy_pass $upstream/api/;
```

I thought: "the location block strips `/api/` from the URI before passing it, so I need to put `/api/` back in proxy_pass." This is true when `proxy_pass` contains a **literal URI** — nginx strips the location prefix and appends the remainder to the proxy URI. But with a **variable**, nginx does not strip the location prefix — it passes the full request URI as-is. So with `proxy_pass $upstream/api/`, a request for `/api/health` becomes `/api//api/health` upstream.

**The fix:** Remove the `/api/` suffix from the variable proxy_pass entirely:

```nginx
# CORRECT
proxy_pass $upstream;
```

With `proxy_pass $upstream`, the variable form, nginx passes the original URI `/api/health` untouched → server receives `/api/health` ✓

**Commit:** `fix(nginx): remove duplicate /api/ path in proxy_pass variable` (1ea93ea)

---

### Bug 3: Ingest Secret Auth Failing (Header vs Body)

**Symptom:** The EC2 sensor was POSTing every 30 seconds but the server was returning `401 Unauthorized`. The sensor logs showed repeated `POST failed: 401`.

**Root cause:** The sensor sends the ingest secret in two places:
```python
headers={"x-ingest-secret": INGEST_SECRET}  # in the header
payload = {..., "secret": INGEST_SECRET}      # also in the body
```

The original server controller only read from the request body:

```javascript
// ORIGINAL — broken
const { deviceId, outputW, timestamp, secret } = req.body

if (secret !== process.env.INGEST_SECRET)
  return res.status(401).json({ message: 'Unauthorized' })
```

The first version of the sensor (before the simulation upgrade) only sent the secret in the **header** — not the body. So `req.body.secret` was `undefined`, the comparison failed, and every request got a 401.

**The fix:** Read from either the header or the body:

```javascript
// FIXED
const secret = req.headers['x-ingest-secret'] || req.body.secret
const { deviceId, outputW, timestamp } = req.body
```

This makes the endpoint tolerant of both patterns — useful if you later have a hardware device that can only set headers, or only send JSON body.

**Commit:** `fix(ingest): accept secret from header or body` (7828cfc)

---

### Bug 4: Git Ownership Check Fails on Self-Hosted Runner

**Symptom:** After switching to the self-hosted runner, the first deploy failed immediately with:

```
fatal: detected dubious ownership in repository at '/opt/hyperion'
```

**Root cause:** Git 2.35.2+ introduced a security check: if the `.git` directory is owned by a different user than the process running git, it refuses to operate. The repo at `/opt/hyperion` was owned by my user, but the GitHub Actions runner service runs as a different system user.

**The fix:** Add a `safe.directory` exception in `deploy.sh`:

```bash
git config --global --add safe.directory /opt/hyperion
git pull origin main
```

This is idempotent — safe to run on every deploy.

**Commit:** `fix(ci): add safe.directory for self-hosted runner git ownership check` (1f666eb)

---

## 7. Sensor Simulation Upgrade

When the sensor was first created, `read_sensor()` returned a hardcoded `0.0`:

```python
def read_sensor() -> float:
    """Replace with actual hardware read."""
    return 0.0
```

This meant the dashboard showed a flat zero for all output — not useful for development or demos. I upgraded it to a physics-based simulation that produces realistic-looking solar data:

**Sine curve solar arc:** Output follows a sine wave between configurable sunrise and sunset hours, peaking at solar noon. A 3500W panel at noon reads ~3500W; at 8am it reads ~1750W.

```python
progress = (hour - SUNRISE_HOUR) / (SUNSET_HOUR - SUNRISE_HOUR)
base = math.sin(math.pi * progress) * PEAK_WATTS
```

**Persistent cloud events:** Every tick there's an 8% chance a cloud begins. Clouds reduce output by 30–80% and persist for 2–6 readings (1–3 minutes), which produces realistic dips rather than single-point noise spikes.

```python
if _cloud_ticks <= 0:
    if random.random() < 0.08:
        _cloud_factor = random.uniform(0.2, 0.7)
        _cloud_ticks  = random.randint(2, 6)
    else:
        _cloud_factor = 1.0
else:
    _cloud_ticks -= 1
```

**Sensor noise:** ±3% random noise on every reading, simulating real-world measurement variation.

All parameters are configurable via environment variables: `HYPERION_PEAK_WATTS`, `HYPERION_SUNRISE_HOUR`, `HYPERION_SUNSET_HOUR`.

Other fixes in the same commit:
- Fixed `datetime.utcnow()` deprecation (Python 3.12+) → `datetime.now(datetime.timezone.utc)`
- Added `PYTHONUNBUFFERED=1` to Dockerfile so logs appear in real time in `docker logs` / `journalctl`
- Removed `--build` from the systemd `ExecStart` — the service should start a pre-built image, not rebuild on every restart

**Commit:** `feat(sensor): realistic solar simulation with day/night cycle and cloud variation` (5998aa0)

---

## 8. Final State — All Key Files

### `sensor/ingest.py`

Full file shown in Phase 1.

### `sensor/Dockerfile`

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY ingest.py ./
VOLUME ["/var/log/hyperion"]
ENV PYTHONUNBUFFERED=1
CMD ["python", "ingest.py"]
```

### `sensor/docker-compose.yml`

```yaml
services:
  sensor:
    build: .
    restart: unless-stopped
    env_file: sensor.env
    volumes:
      - sensor_logs:/var/log/hyperion

volumes:
  sensor_logs:
```

### `sensor/hyperion-sensor.service`

```ini
[Unit]
Description=Hyperion Solar Sensor Ingest
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=ec2-user
WorkingDirectory=/opt/hyperion/sensor
ExecStart=/usr/bin/docker compose up
ExecStop=/usr/bin/docker compose down
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal
LogRateLimitIntervalSec=30
LogRateLimitBurst=5

[Install]
WantedBy=multi-user.target
```

### `sensor/requirements.txt`

```
requests
```

### `server/Dockerfile`

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 5000
CMD ["node", "index.js"]
```

### `client/Dockerfile`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

### `client/nginx.conf`

```nginx
server {
  listen 80;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ /index.html;
  }

  location /api/ {
    resolver 127.0.0.11 valid=30s;
    set $upstream http://server:5000;
    proxy_pass $upstream;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}
```

### `docker-compose.yml`

```yaml
services:
  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data
    networks:
      - internal

  server:
    build: ./server
    restart: unless-stopped
    env_file: ./server/.env
    environment:
      REDIS_URL: redis://redis:6379
    depends_on:
      - redis
    networks:
      - internal

  client:
    build: ./client
    restart: unless-stopped
    depends_on:
      - server
    networks:
      - internal
      - web-network

networks:
  internal:
    driver: bridge
  web-network:
    external: true

volumes:
  redis_data:
```

### `deploy/deploy.sh`

```bash
#!/bin/bash
set -e
cd /opt/hyperion

git config --global --add safe.directory /opt/hyperion
git pull origin main

docker compose up -d --build
```

### `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Deploy
        run: bash /opt/hyperion/deploy/deploy.sh
```

### `.dockerignore`

```
node_modules
*/node_modules
.env
*/.env
client/dist
*.log
.git
```

---

## 9. Quick-Reference Checklist

Use this when setting up a new VPS or reprovisioning the EC2.

### Hostinger VPS — One-Time Setup

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# log out and back in

# Create Docker networks
docker network create web-network

# Set up Nginx Proxy Manager (separate compose stack in /opt/npm)
# Add Proxy Host in NPM UI: hyperion.nikhilroy.com → client:80, SSL via Let's Encrypt

# Clone repo
sudo mkdir -p /opt/hyperion
sudo chown $USER:$USER /opt/hyperion
git clone https://github.com/phy-nikhilroy/hyperion.git /opt/hyperion
cp /opt/hyperion/server/.env.example /opt/hyperion/server/.env
# Edit /opt/hyperion/server/.env with real MONGO_URI, JWT_SECRET, INGEST_SECRET, etc.

# First deploy (before runner is set up)
cd /opt/hyperion && docker compose up -d --build

# Install GitHub Actions self-hosted runner (follow GitHub UI instructions)
# Runner should appear as Idle in repo Settings → Actions → Runners
```

### AWS EC2 — One-Time Setup

```bash
sudo yum update -y && sudo yum install -y docker
sudo systemctl enable docker --now
sudo usermod -aG docker ec2-user
# log out and back in

# Docker Compose plugin
sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 \
  -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

# Sensor files
sudo mkdir -p /opt/hyperion/sensor
# Copy sensor/* to /opt/hyperion/sensor/
cp /opt/hyperion/sensor/sensor.env.example /opt/hyperion/sensor/sensor.env
# Edit sensor.env with real values

# Systemd service
sudo cp /opt/hyperion/sensor/hyperion-sensor.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable hyperion-sensor --now
sudo systemctl status hyperion-sensor
```

### Ongoing Deploys

Push to `main` → GitHub Actions triggers → self-hosted runner on Hostinger runs `deploy.sh` → `docker compose up -d --build` rebuilds changed images and restarts services.

### Useful Debug Commands

```bash
# On Hostinger VPS
docker compose -f /opt/hyperion/docker-compose.yml ps
docker compose -f /opt/hyperion/docker-compose.yml logs -f server
docker compose -f /opt/hyperion/docker-compose.yml logs -f client

# On EC2
sudo systemctl status hyperion-sensor
journalctl -u hyperion-sensor -f --no-pager

# Check GitHub Actions runner status on Hostinger
sudo systemctl status actions.runner.*
```

---

*All commits referenced are on the `main` branch of [github.com/phy-nikhilroy/hyperion](https://github.com/phy-nikhilroy/hyperion). The sensor runs on a `t2.micro` EC2 in `us-east-1`. The app is live at [hyperion.nikhilroy.com](https://hyperion.nikhilroy.com).*
