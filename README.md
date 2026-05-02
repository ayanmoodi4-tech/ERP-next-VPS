# ERPNext v16 — Complete VPS Deployment Guide
> **Server:** Hostinger VPS · 1 vCPU · 4GB RAM · 50GB NVMe · Ubuntu 24.04  
> **Goal:** Production deployment with Bench (not Docker)  
> **ERPNext Version:** v16 (stable since Jan 12, 2026)

---

## Table of Contents
1. [Why Bench over Docker](#1-why-bench-over-docker)
2. [v16 Requirements — What Changed](#2-v16-requirements--what-changed)
3. [Phase 1 — Initial Server Setup](#3-phase-1--initial-server-setup)
4. [Phase 2 — Security Hardening](#4-phase-2--security-hardening)
5. [Phase 3 — Install Prerequisites](#5-phase-3--install-prerequisites)
6. [Phase 4 — Configure MariaDB](#6-phase-4--configure-mariadb)
7. [Phase 5 — Install ERPNext with Bench](#7-phase-5--install-erpnext-with-bench)
8. [Phase 6 — Domain & DNS Setup](#8-phase-6--domain--dns-setup)
9. [Phase 7 — Production Setup (Nginx + SSL)](#9-phase-7--production-setup-nginx--ssl)
10. [Phase 8 — Performance Tuning for 4GB RAM](#10-phase-8--performance-tuning-for-4gb-ram)
11. [Phase 9 — Automated Backups](#11-phase-9--automated-backups)
12. [Multiple Benches — Dev & Staging](#12-multiple-benches--dev--staging)
13. [Day-to-Day Commands](#13-day-to-day-commands)
14. [Post-Login Checklist](#14-post-login-checklist)
15. [Common Errors & Fixes](#15-common-errors--fixes)

---

## 1. Why Bench over Docker

| Factor | Bench ✅ | Docker ❌ |
|---|---|---|
| RAM usage | ~1.5–2GB | ~2.5–3.5GB (containers stack up) |
| 1 vCPU performance | Good | Poor (container orchestration overhead) |
| Production stability | Battle-tested | Needs more RAM to breathe |
| SSL/Nginx | Native, simple | Traefik/Nginx proxy layers = complexity |
| Multiple sites | Easy, native | Requires compose file changes |
| Debug/logs | Direct access | Requires `docker exec` |

> **Bottom line:** With 4GB RAM and 1 vCPU, Docker would leave you ~500MB free for ERPNext itself. Bench on bare Ubuntu is the correct choice.

---

## 2. v16 Requirements — What Changed

| Component | v15 | v16 (what you need) |
|---|---|---|
| Python | 3.10 / 3.11 | **3.12+ minimum (3.14 recommended)** |
| Node.js | 18.x | **24.x** |
| Frappe branch flag | `version-15` | `version-16` |
| Performance | Baseline | ~2x faster |
| UI | Old desk | Redesigned, modern |
| Stable since | — | **January 12, 2026** |

> ⚠️ Python 3.10/3.11 will NOT work with v16. Do not skip the Python upgrade step.

---

## 3. Phase 1 — Initial Server Setup

### Connect as root
```bash
ssh root@YOUR_SERVER_IP
```

### Create a non-root user (never run ERPNext as root)
```bash
adduser erpuser
usermod -aG sudo erpuser
```

### Basic firewall setup
```bash
apt update && apt upgrade -y

ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable
```

### Switch to your new user
```bash
su - erpuser
```

### Phase 1 Checklist
- [ ] SSH'd into server as root
- [ ] Created `erpuser` with sudo privileges
- [ ] UFW enabled with SSH, 80, 443 open
- [ ] Switched to `erpuser`

---

## 4. Phase 2 — Security Hardening

> Do this **before** anything else. Getting locked out is easy if done in wrong order.

### Step 1 — Set up SSH key on your LOCAL machine first
```bash
# Run this on YOUR laptop/PC, not the server
ssh-keygen -t ed25519 -C "erpnext-vps"
ssh-copy-id erpuser@YOUR_VPS_IP

# Test it works before proceeding
ssh erpuser@YOUR_VPS_IP
```

### Step 2 — Harden SSH config (on the server)
```bash
sudo nano /etc/ssh/sshd_config
```

Change/add these lines:
```
PermitRootLogin no
PasswordAuthentication no
Port 2222
```

Update UFW for new SSH port, then restart SSH:
```bash
sudo ufw allow 2222
sudo ufw delete allow OpenSSH
sudo systemctl restart sshd
```

> ⚠️ Open a **second SSH session** to verify login still works before closing the first one.

### Step 3 — Install Fail2ban (brute force protection)
```bash
sudo apt install -y fail2ban

sudo nano /etc/fail2ban/jail.local
```

Paste this:
```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5

[sshd]
enabled = true
port = 2222
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Step 4 — Enable automatic security updates
```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
# Choose "Yes" when prompted
```

### Step 5 — Verify MariaDB is localhost-only (after install)
```bash
sudo ss -tlnp | grep mysql
# Must show 127.0.0.1:3306 — NOT 0.0.0.0:3306
```

### Security Checklist
- [ ] SSH key copied to server and tested
- [ ] Root SSH login disabled (`PermitRootLogin no`)
- [ ] Password authentication disabled
- [ ] SSH port changed to 2222
- [ ] UFW updated for new SSH port
- [ ] Fail2ban installed and running
- [ ] Unattended security updates enabled
- [ ] MariaDB confirmed localhost-only (check after Phase 4)

### Final UFW rules should look like this:
```
2222/tcp   ALLOW   (SSH)
80/tcp     ALLOW   (HTTP)
443/tcp    ALLOW   (HTTPS)
Everything else → DENY
```

---

## 5. Phase 3 — Install Prerequisites

### System packages
```bash
sudo apt install -y \
  git curl wget nano \
  software-properties-common \
  mariadb-server mariadb-client \
  redis-server \
  xvfb libfontconfig wkhtmltopdf \
  nginx supervisor \
  pkg-config libmariadb-dev gcc build-essential libssl-dev cron
```

### Python 3.12 (v16 minimum requirement)
```bash
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install -y python3.12 python3.12-dev python3.12-venv python3.12-distutils python3-pip python3-setuptools
```

Verify:
```bash
python3.12 --version
# Expected: Python 3.12.x
```

### Node.js 24 (v16 requirement)
```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
node -v   # Expected: v24.x
```

### Yarn
```bash
sudo npm install -g yarn
yarn --version
```

### Frappe Bench CLI
```bash
sudo pip3 install frappe-bench
bench --version
```

### Prerequisites Checklist
- [ ] System packages installed
- [ ] Python 3.12+ installed and verified
- [ ] Node.js 24.x installed and verified
- [ ] Yarn installed
- [ ] `frappe-bench` CLI installed

---

## 6. Phase 4 — Configure MariaDB

### Run secure installation
```bash
sudo mysql_secure_installation
```
Answer all prompts:
- Set strong root password → **Yes**
- Remove anonymous users → **Yes**
- Disallow remote root login → **Yes**
- Remove test database → **Yes**
- Reload privilege tables → **Yes**

### Configure character set for Frappe
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add under `[mysqld]`:
```ini
[mysqld]
character-set-client-handshake = FALSE
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

[mysql]
default-character-set = utf8mb4
```

```bash
sudo systemctl restart mariadb
sudo systemctl enable mariadb
```

### Verify it's running
```bash
sudo systemctl status mariadb
```

### Phase 4 Checklist
- [ ] `mysql_secure_installation` completed
- [ ] utf8mb4 character set configured
- [ ] MariaDB restarted and enabled
- [ ] MariaDB confirmed localhost-only (`sudo ss -tlnp | grep mysql`)

---

## 7. Phase 5 — Install ERPNext with Bench

> This is the longest phase. Steps 3–5 take 15–30 minutes combined.

### Step 1 — Initialize Frappe Bench with v16
```bash
cd ~
bench init --frappe-branch version-16 --python python3.12 frappe-bench
cd frappe-bench
```

### Step 2 — Get ERPNext app
```bash
bench get-app --branch version-16 erpnext
```

### Step 3 — Create your site
```bash
# Replace yourdomain.com, MARIADB_ROOT_PASS, and ADMIN_PASSWORD
bench new-site yourdomain.com \
  --db-root-password YOUR_MARIADB_ROOT_PASS \
  --admin-password YOUR_ADMIN_PASSWORD
```

### Step 4 — Install ERPNext on the site
```bash
bench --site yourdomain.com install-app erpnext
```

### Step 5 — Verify installation
```bash
bench --site yourdomain.com list-apps
# Should show: frappe, erpnext
```

### Phase 5 Checklist
- [ ] Bench initialized with `version-16` branch and `python3.12`
- [ ] ERPNext app downloaded
- [ ] Site created with strong admin password
- [ ] ERPNext installed on site
- [ ] `list-apps` shows both frappe and erpnext

---

## 8. Phase 6 — Domain & DNS Setup

### On Hostinger DNS panel

Go to **Hostinger → Domains → DNS Zone** and add:

| Type | Name | Value | TTL |
|---|---|---|---|
| A | @ | YOUR_VPS_IP | 300 |
| A | www | YOUR_VPS_IP | 300 |

Save and wait **5–30 minutes** for propagation.

### Verify DNS has propagated
```bash
ping yourdomain.com
# Should resolve to your VPS IP

# Or use:
nslookup yourdomain.com
```

### Phase 6 Checklist
- [ ] A record added for `@` → VPS IP
- [ ] A record added for `www` → VPS IP
- [ ] DNS propagation confirmed (`ping yourdomain.com` resolves correctly)

---

## 9. Phase 7 — Production Setup (Nginx + SSL)

### Step 1 — Enable production mode
```bash
cd ~/frappe-bench

# Add site to hosts
bench --site yourdomain.com add-to-hosts

# Set up Supervisor + Nginx configs (run as sudo)
sudo bench setup production erpuser
```

### Step 2 — Verify services are running
```bash
sudo supervisorctl status
# All processes should show RUNNING

sudo systemctl status nginx
# Should show active (running)
```

### Step 3 — SSL with Let's Encrypt
```bash
sudo apt install -y certbot python3-certbot-nginx

sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Follow the prompts. Certbot will:
- Verify domain ownership
- Issue the certificate
- Auto-configure Nginx for HTTPS
- Set up auto-renewal via cron

### Step 4 — Final site configuration
```bash
bench --site yourdomain.com enable-scheduler
bench --site yourdomain.com set-maintenance-mode off
```

### Step 5 — Test auto-renewal
```bash
sudo certbot renew --dry-run
# Should say "Congratulations, all renewals succeeded"
```

### Phase 7 Checklist
- [ ] Production mode enabled with Supervisor + Nginx
- [ ] All Supervisor processes showing RUNNING
- [ ] Nginx active and serving the site
- [ ] SSL certificate issued via certbot
- [ ] Auto-renewal dry-run successful
- [ ] Scheduler enabled
- [ ] Maintenance mode off
- [ ] Site loads at `https://yourdomain.com` ✅

---

## 10. Phase 8 — Performance Tuning for 4GB RAM

> **Critical step.** Default settings assume more RAM. Skip this and you'll hit OOM errors.

### Step 1 — Add a 2GB swap file
```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

# Verify swap is active
free -h
```

### Step 2 — Reduce Supervisor worker count
```bash
nano ~/frappe-bench/config/supervisor.conf
```

Find lines with `numprocs` for web workers and reduce:
```ini
numprocs=1    ; reduce from 2 to 1
```

### Step 3 — Limit Redis memory
```bash
sudo nano /etc/redis/redis.conf
```

Add at the bottom:
```
maxmemory 256mb
maxmemory-policy allkeys-lru
```

```bash
sudo systemctl restart redis
sudo supervisorctl reload
```

### Step 4 — Verify memory after tuning
```bash
free -h
# Aim for at least 1GB free during idle
```

### Phase 8 Checklist
- [ ] 2GB swap file created and active
- [ ] Supervisor workers reduced to `numprocs=1`
- [ ] Redis limited to 256MB
- [ ] `free -h` shows reasonable free memory at idle

---

## 11. Phase 9 — Automated Backups

### Set backup limit
```bash
bench --site yourdomain.com set-config backup_limit 5
```

### Add nightly backup cron job
```bash
crontab -e
```

Add this line (backs up every night at 2 AM):
```
0 2 * * * cd /home/erpuser/frappe-bench && bench --site yourdomain.com backup --with-files >> /home/erpuser/backup.log 2>&1
```

### Off-site backup (strongly recommended)
In ERPNext → **System Settings → Backups**, configure S3-compatible storage:
- AWS S3 (free tier: 5GB)
- Cloudflare R2 (free tier: 10GB — best option)
- Backblaze B2 (cheapest paid option)

### Phase 9 Checklist
- [ ] Backup limit set to 5
- [ ] Nightly cron job added
- [ ] Off-site backup (S3/R2) configured in ERPNext settings
- [ ] First manual backup tested: `bench --site yourdomain.com backup --with-files`

---

## 12. Multiple Benches — Dev & Staging

### Yes — you can run multiple benches ✅

Each bench is fully independent with its own virtualenv, apps, and sites. This is one of the main advantages of Bench over Docker.

```
/home/erpuser/
├── frappe-bench/          ← Production (always running via Supervisor)
└── frappe-bench-dev/      ← Dev/Staging (start manually when needed)
```

### Create a second bench for staging/dev
```bash
cd ~
bench init --frappe-branch version-16 --python python3.12 frappe-bench-dev
cd frappe-bench-dev
bench get-app --branch version-16 erpnext
bench new-site staging.yourdomain.com \
  --db-root-password YOUR_MARIADB_ROOT_PASS \
  --admin-password STAGING_ADMIN_PASS
bench --site staging.yourdomain.com install-app erpnext
```

### Start dev bench (foreground, with live logs)
```bash
cd ~/frappe-bench-dev
bench start
# Ctrl+C to stop
```

### ⚠️ RAM Warning
Running both benches simultaneously is tight on 4GB. Best practice:
- Production bench → always running via Supervisor
- Dev bench → start only when actively working, stop when done

```bash
# Stop dev bench
cd ~/frappe-bench-dev && bench stop

# Start it again
bench start
```

### Use dev bench to test before updating production
```bash
# Test update on dev first
cd ~/frappe-bench-dev
bench update --reset

# If all good, update production
cd ~/frappe-bench
bench update --reset
```

---

## 13. Day-to-Day Commands

### Bench management
```bash
# Restart all services
cd ~/frappe-bench
bench restart

# Check status of all processes
sudo supervisorctl status

# View live web log
tail -f ~/frappe-bench/logs/web.log

# View live worker log
tail -f ~/frappe-bench/logs/worker.log

# View scheduler log
tail -f ~/frappe-bench/logs/schedule.log
```

### Site management
```bash
# Manual backup
bench --site yourdomain.com backup --with-files

# List installed apps
bench --site yourdomain.com list-apps

# Health check
bench --site yourdomain.com doctor

# Run migrations (after update)
bench --site yourdomain.com migrate

# Clear cache
bench --site yourdomain.com clear-cache
```

### Updating ERPNext
```bash
# Always backup FIRST
bench --site yourdomain.com backup --with-files

# Then update
bench update --reset

# If something breaks, restore from backup
bench --site yourdomain.com restore /path/to/backup.sql.gz
```

### Install additional Frappe apps
```bash
# Example: HR module
bench get-app --branch version-16 hrms
bench --site yourdomain.com install-app hrms

# Example: Print Designer
bench get-app print_designer
bench --site yourdomain.com install-app print_designer
```

---

## 14. Post-Login Checklist

After visiting `https://yourdomain.com` and logging in as Administrator:

### First-time setup
- [ ] Complete the **Setup Wizard** (company name, currency, timezone, industry)
- [ ] Set timezone in **System Settings**
- [ ] Configure **Email (SMTP)** — use Mailgun, Brevo, or Resend (all have free tiers)
- [ ] Set up **Roles & Users** before going live
- [ ] Configure **Backups to S3** in System Settings

### Recommended free SMTP options
| Provider | Free tier | Notes |
|---|---|---|
| Brevo | 300 emails/day | Best free option |
| Mailgun | 100 emails/day | Easy setup |
| Resend | 3,000 emails/month | Developer-friendly |

### Modules to enable based on your business
Go to **Settings → Module Profile** and enable only what you need. Fewer active modules = faster performance on your VPS.

---

## 15. Common Errors & Fixes

### `bench init` fails with Python error
```bash
# Make sure you're specifying Python 3.12 explicitly
bench init --frappe-branch version-16 --python python3.12 frappe-bench
```

### Supervisor processes not starting
```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart all
```

### Nginx 502 Bad Gateway
```bash
# Check if gunicorn is running
sudo supervisorctl status frappe-bench-web:frappe-bench-web-0

# Check logs
tail -f ~/frappe-bench/logs/web.error.log
```

### Site not loading after certbot
```bash
sudo nginx -t           # Check config for syntax errors
sudo systemctl reload nginx
```

### MariaDB connection refused
```bash
sudo systemctl status mariadb
sudo systemctl restart mariadb

# Check if character set was applied
mysql -u root -p -e "SHOW VARIABLES LIKE 'character_set_server';"
# Should show utf8mb4
```

### Out of memory / OOM kills
```bash
free -h                  # Check memory
swapon --show            # Verify swap is active

# If swap is missing, re-add it:
sudo swapon /swapfile
```

### certbot fails — domain not resolving
```bash
# DNS hasn't propagated yet. Wait and retry:
nslookup yourdomain.com    # Check if it resolves to your IP
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

---

## Architecture Overview

```
Internet
   │
   ▼
Nginx (80/443) ──── SSL via Let's Encrypt
   │
   ▼
Gunicorn (Python WSGI) ── Port 8000 (internal only)
   │
   ├── MariaDB (database) ── localhost:3306
   ├── Redis (cache + queue) ── localhost:6379
   └── Background Workers ── managed by Supervisor
          │
          └── Scheduler (cron jobs, emails, reports)
```

All managed by **Supervisor** (process manager), which auto-restarts crashed processes and starts everything on server reboot.

---

## Quick Reference Card

```
VPS User:        erpuser
Bench location:  /home/erpuser/frappe-bench
Logs:            /home/erpuser/frappe-bench/logs/
Site config:     /home/erpuser/frappe-bench/sites/yourdomain.com/site_config.json
Nginx config:    /etc/nginx/conf.d/frappe-bench.conf
Supervisor conf: /etc/supervisor/conf.d/frappe-bench.conf
Backups:         /home/erpuser/frappe-bench/sites/yourdomain.com/private/backups/
SSH Port:        2222
```

---

*Guide covers ERPNext v16 (stable Jan 2026) · Frappe Framework v16 · Ubuntu 24.04 LTS*
