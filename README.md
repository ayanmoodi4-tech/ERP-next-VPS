# ERPNext v16 Production Setup Guide (Ubuntu 24.04)

## ✅ 1. System Update

```bash
sudo apt update && sudo apt upgrade -y
```

---

## ✅ 2. Install Required Packages

```bash
sudo apt install -y git python3-dev python3-setuptools python3-pip python3-distutils \
build-essential libmysqlclient-dev redis-server curl nginx supervisor \
pkg-config libffi-dev libssl-dev wkhtmltopdf
```

---

## ✅ 3. Install Node.js (v24+ REQUIRED)

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
sudo apt install -y nodejs
node -v
```

---

## ✅ 4. Install Yarn

```bash
sudo npm install -g yarn
yarn -v
```

---

## ✅ 5. Install Python 3.14 (Manual Build)

```bash
cd /usr/src
sudo wget https://www.python.org/ftp/python/3.14.0/Python-3.14.0.tgz
sudo tar xzf Python-3.14.0.tgz
cd Python-3.14.0

sudo ./configure --enable-optimizations
sudo make -j$(nproc)
sudo make altinstall

python3.14 --version
```

---

## ✅ 6. Install Bench (via pipx)

```bash
pip3 install --user pipx
pipx ensurepath

# reload shell
exec $SHELL

pipx install frappe-bench
bench --version
```

---

## ✅ 7. Setup MariaDB

```bash
sudo apt install mariadb-server -y
sudo mysql_secure_installation
```

### Configure charset:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add under `[mysqld]`:

```ini
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
```

Restart:

```bash
sudo systemctl restart mariadb
```

Verify:

```bash
mysql -u root -p -e "SHOW VARIABLES LIKE 'character_set_server';"
```

---

## ✅ 8. Create Bench (IMPORTANT: use Python 3.14)

```bash
cd ~
bench init --frappe-branch version-16 --python python3.14 frappe-bench
cd frappe-bench
```

---

## ✅ 9. Create Site

```bash
bench new-site site1.local
```

---

## ✅ 10. Install ERPNext

```bash
bench get-app erpnext --branch version-16
bench --site site1.local install-app erpnext
```

---

## ✅ 11. Build Assets (IMPORTANT)

```bash
bench build
bench clear-cache
bench clear-website-cache
```

---

## ✅ 12. Enable Scheduler

```bash
bench --site site1.local enable-scheduler
```

---

## ✅ 13. Production Setup

```bash
sudo /home/erpuser/.local/bin/bench setup production erpuser
```

---

## ✅ 14. Fix Supervisor (if needed)

```bash
bench setup supervisor
sudo cp config/supervisor.conf /etc/supervisor/conf.d/frappe-bench.conf

sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart all
```

---

## ✅ 15. Setup Nginx

```bash
bench setup nginx
sudo systemctl restart nginx
```

---

## ✅ 16. Set Default Site

```bash
bench use site1.local
bench set-config -g serve_default_site on
sudo supervisorctl restart all
```

---

## ✅ 17. Fix Redis Warning (Recommended)

```bash
sudo nano /etc/sysctl.conf
```

Add:

```ini
vm.overcommit_memory = 1
```

Apply:

```bash
sudo sysctl -p
```

---

## ✅ 18. Access ERPNext

Open browser:

```
http://YOUR_SERVER_IP
```

---

# 🔧 Troubleshooting

## ❌ CSS/JS 404 (unstyled UI)

```bash
bench build
bench clear-cache
```

---

## ❌ Supervisor shows BACKOFF

```bash
bench setup supervisor
sudo supervisorctl restart all
```

---

## ❌ Redis connection error

```bash
bench setup redis
bench restart
```

---

## ❌ Bench not found with sudo

```bash
sudo /home/erpuser/.local/bin/bench setup production erpuser
```

---

## ❌ Check services

```bash
sudo supervisorctl status
sudo systemctl status nginx
```

---

# ✅ Final Stack

* Python 3.14
* Node.js 24+
* MariaDB
* Redis
* Bench
* ERPNext v16
* Nginx (production)
* Supervisor (process manager)

---

# 🎯 Done

You now have a full **production-ready ERPNext setup**.

---
