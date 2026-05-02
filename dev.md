# ERPNext Production Admin Guide
### For: fameenterprises.in | Solo Admin | Beginner

---

## 📋 Table of Contents
1. [Daily Checks](#1-daily-checks)
2. [Backups](#2-backups)
3. [Updates & Upgrades](#3-updates--upgrades)
4. [User Management](#4-user-management)
5. [Customizations](#5-customizations)
6. [Performance Monitoring](#6-performance-monitoring)
7. [Troubleshooting](#7-troubleshooting)
8. [Emergency Commands](#8-emergency-commands)

---

## 1. Daily Checks

Quick health check every morning (takes 2 minutes):

```bash
# Check all services are running
sudo supervisorctl status

# Check NGINX
sudo systemctl status nginx

# Check disk space (keep above 20% free)
df -h

# Check memory
free -h
```

Everything should show `RUNNING`. If anything is down, see Section 7.

---

## 2. Backups

### 2a. Manual Backup (run anytime)

```bash
cd ~/frappe-bench

# Backup database + all files
bench --site site1.local backup --with-files
```

Backups are saved to:
```
~/frappe-bench/sites/site1.local/private/backups/
```

### 2b. Automated Daily Backup (set this up NOW)

```bash
# Open cron editor
crontab -e
```

Add this line (runs backup every day at 2 AM):
```
0 2 * * * cd /home/erpuser/frappe-bench && /home/erpuser/.local/bin/bench --site site1.local backup --with-files >> /home/erpuser/backup.log 2>&1
```

Save and exit.

### 2c. Keep Only Last 7 Backups (save disk space)

```bash
# Add this to crontab as well (runs daily at 3 AM)
0 3 * * * find /home/erpuser/frappe-bench/sites/site1.local/private/backups/ -mtime +7 -delete
```

### 2d. Copy Backups Off-Server (IMPORTANT)

Never rely on backups stored on the same server. Options:

**Option A — Copy to your local machine (manual):**
```bash
# Run this on YOUR local machine, not the VPS
scp erpuser@31.97.229.226:~/frappe-bench/sites/site1.local/private/backups/*.gz ./
```

**Option B — Google Drive / Rclone (automated):**
```bash
# Install rclone
sudo apt install rclone

# Configure (follow interactive setup)
rclone config

# Add to crontab after backup runs
30 2 * * * rclone copy /home/erpuser/frappe-bench/sites/site1.local/private/backups/ gdrive:erpnext-backups/
```

### 2e. How to Restore a Backup

```bash
cd ~/frappe-bench

# Restore database
bench --site site1.local restore /path/to/backup.sql.gz

# Restore with files
bench --site site1.local restore /path/to/backup.sql.gz \
  --with-public-files /path/to/public-files.tar \
  --with-private-files /path/to/private-files.tar

# Run migrations after restore
bench --site site1.local migrate

# Restart everything
bench restart
```

---

## 3. Updates & Upgrades

### 3a. Minor Updates (bug fixes — do monthly)

```bash
cd ~/frappe-bench

# ALWAYS backup before updating
bench --site site1.local backup --with-files

# Pull latest updates
bench update --pull

# Run database migrations
bench --site site1.local migrate

# Restart services
bench restart
```

### 3b. Before ANY Update — Checklist

- [ ] Take backup (`bench --site site1.local backup --with-files`)
- [ ] Note current versions (`bench version`)
- [ ] Do it during off-hours (nights/weekends)
- [ ] Test on staging first if possible

### 3c. Check Current Versions

```bash
cd ~/frappe-bench
bench version
```

### 3d. Clear Cache After Updates

```bash
bench --site site1.local clear-cache
bench --site site1.local clear-website-cache
sudo systemctl restart nginx
```

---

## 4. User Management

### 4a. Add a New User (via UI)

1. Login to https://fameenterprises.in
2. Go to **Settings → User List → New**
3. Fill in email, name
4. Assign **Roles** (e.g. "Accounts User", "Sales User")
5. User receives email invite

### 4b. Reset a User's Password (via command line)

```bash
cd ~/frappe-bench
bench --site site1.local set-admin-password newpassword123
```

### 4c. Disable a User

1. Go to **Settings → User List**
2. Open the user
3. Uncheck **Enabled**

### 4d. Important Roles to Know

| Role | Access |
|------|--------|
| System Manager | Full admin access |
| Accounts Manager | All accounting |
| Accounts User | Limited accounting |
| Sales Manager | All sales |
| Sales User | Limited sales |
| Purchase Manager | All purchases |

---

## 5. Customizations

### 5a. Simple Customizations (No Coding — via UI)

**Custom Fields** — Add fields to any form:
1. Go to **Settings → Customize Form**
2. Select the form (e.g. "Sales Invoice")
3. Click **Add Row** to add a custom field
4. Save

**Custom Print Formats** — Change how invoices/documents look:
1. Go to **Settings → Print Format → New**
2. Select DocType, design your template
3. Set as default in the form

**Naming Series** — Change document numbering:
1. Go to **Settings → Naming Series**
2. Modify prefixes (e.g. `INV-2024-`)

### 5b. Workflow Automation (No Coding)

Set up approval workflows:
1. Go to **Settings → Workflow → New**
2. Define states (Draft → Pending Approval → Approved)
3. Assign roles to each transition

### 5c. Custom App Development (Requires Coding)

For bigger customizations, create a custom app:

```bash
cd ~/frappe-bench

# Create new app
bench new-app my_custom_app

# Install on site
bench --site site1.local install-app my_custom_app
```

Custom app folder: `~/frappe-bench/apps/my_custom_app/`

Key files:
```
my_custom_app/
├── hooks.py          # App hooks and events
├── modules.txt       # List of modules
└── my_module/
    └── doctype/      # Your custom doctypes
```

### 5d. Rebuild Assets After Customization

```bash
cd ~/frappe-bench
bench build
bench --site site1.local clear-cache
```

---

## 6. Performance Monitoring

### 6a. Check Server Resources

```bash
# CPU and memory live view
top

# Disk usage
df -h

# Check which process uses most memory
ps aux --sort=-%mem | head -10
```

### 6b. Check ERPNext Logs

```bash
# Frappe/Gunicorn errors
tail -50 ~/frappe-bench/logs/frappe.log

# Worker errors
tail -50 ~/frappe-bench/logs/worker.log

# NGINX errors
sudo tail -50 /var/log/nginx/error.log

# Live log watching
tail -f ~/frappe-bench/logs/frappe.log
```

### 6c. Check Slow Queries (via UI)

1. Go to **Settings → Error Log** — shows application errors
2. Go to **Settings → Scheduled Job Log** — shows background job status
3. Go to **Settings → Activity Log** — shows user activity

### 6d. Restart Services When Slow

```bash
cd ~/frappe-bench

# Restart all bench processes
bench restart

# Or restart individually
sudo supervisorctl restart frappe-bench-frappe:
sudo supervisorctl restart frappe-bench-redis-cache:
sudo supervisorctl restart frappe-bench-worker:
```

### 6e. MariaDB Performance Check

```bash
# Connect to database
mysql -u root -p

# Check running queries
SHOW PROCESSLIST;

# Exit
exit
```

### 6f. Set Up Simple Uptime Monitoring (Free)

Sign up at https://uptimerobot.com (free tier):
1. Add monitor → HTTP(s)
2. URL: `https://fameenterprises.in`
3. Get email alerts if site goes down

---

## 7. Troubleshooting

### Site shows 502 Bad Gateway

```bash
cd ~/frappe-bench
sudo supervisorctl status        # Check if gunicorn is running
bench restart                    # Restart all processes
sudo systemctl restart nginx     # Restart nginx
```

### Assets not loading (CSS/JS broken)

```bash
cd ~/frappe-bench
bench build                                    # Rebuild assets
bench --site site1.local clear-cache          # Clear cache
sudo chmod -R o+rx ~/frappe-bench/sites/assets  # Fix permissions
sudo systemctl restart nginx
```

### Scheduler not running (emails not sending, reports not generating)

```bash
cd ~/frappe-bench

# Check scheduler status
bench --site site1.local scheduler status

# Enable if disabled
bench --site site1.local enable-scheduler

# Check worker logs
tail -50 ~/frappe-bench/logs/worker.log
```

### Database connection errors

```bash
# Restart MariaDB
sudo systemctl restart mariadb

# Check MariaDB status
sudo systemctl status mariadb
```

### Redis errors

```bash
sudo supervisorctl restart frappe-bench-redis-cache:
sudo supervisorctl restart frappe-bench-redis-queue:
sudo supervisorctl restart frappe-bench-redis-socketio:
```

### SSL Certificate expired

```bash
# Renew manually
sudo certbot renew

# Reload nginx
sudo systemctl reload nginx
```

SSL auto-renews every 60 days automatically. Verify with:
```bash
sudo certbot renew --dry-run
```

---

## 8. Emergency Commands

### Complete restart of everything

```bash
cd ~/frappe-bench
sudo systemctl restart mariadb
sudo pkill redis-server
sudo supervisorctl restart all
sudo systemctl restart nginx
bench restart
```

### Put site in maintenance mode (during updates)

```bash
cd ~/frappe-bench
bench --site site1.local set-maintenance-mode on
# ... do your work ...
bench --site site1.local set-maintenance-mode off
```

### Check what's eating disk space

```bash
du -sh ~/frappe-bench/sites/site1.local/private/backups/
du -sh ~/frappe-bench/sites/assets/
du -sh /var/log/
```

### Full system status check

```bash
echo "=== Supervisor ===" && sudo supervisorctl status
echo "=== NGINX ===" && sudo systemctl status nginx --no-pager
echo "=== MariaDB ===" && sudo systemctl status mariadb --no-pager
echo "=== Disk ===" && df -h /
echo "=== Memory ===" && free -h
```

---

## 📅 Maintenance Schedule (Recommended)

| Frequency | Task |
|-----------|------|
| Daily | Auto-backup runs at 2 AM (set up in Section 2b) |
| Weekly | Check error logs, check disk space |
| Monthly | Apply minor updates (Section 3a), verify backup restore works |
| Every 3 months | Review user access, clean old backups |
| Before any update | Full backup + note current versions |

---

## 🔗 Useful Links

- ERPNext Docs: https://docs.erpnext.com
- Frappe Forum: https://discuss.frappe.io
- ERPNext GitHub: https://github.com/frappe/erpnext
- Your Site: https://fameenterprises.in
- Certbot Renewal: `sudo certbot renew --dry-run`

---

*Generated for fameenterprises.in — ERPNext v16 on Ubuntu VPS*
