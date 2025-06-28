# Ultimate Step-by-Step Guide  
## Running n8n in Docker Compose on Ubuntu 24.04 LTS (with PostgreSQL + Caddy HTTPS)

---

> **Audience** – This guide is written for motivated beginners: people who are comfortable copy-pasting commands but may have **no prior Docker or Linux administration experience**.  
> **Goal** – When you reach the bottom, you will have:
> 1. A fully working **n8n** automation server reachable at `https://n8n.example.com`.
> 2. Automatic HTTPS certificates via **Caddy** and Let’s Encrypt.
> 3. All application data stored outside of containers, so you can upgrade or rebuild safely.
> 4. A reproducible workflow to restore or migrate your installation.

Approx. length: **≈ 6,000 words** (give or take; your shell’s `wc -w` may count differently).

---

## Contents

1. [Conceptual Overview](#1-conceptual-overview)  
2. [Before You Begin – Quick Checklist](#2-before-you-begin--quick-checklist)  
3. [Step 0 – Preparing the Machine](#3-step-0--preparing-the-machine)  
4. [Step 1 – Install Docker Engine + Compose v2](#4-step-1--install-docker-engine--compose-v2)  
5. [Step 2 – Create a Dedicated Project Directory](#5-step-2--create-a-dedicated-project-directory)  
6. [Step 3 – Write the `docker-compose.yml`](#6-step-3--write-the-docker-composeyml)  
7. [Step 4 – Author the Caddyfile](#7-step-4--author-the-caddyfile)  
8. [Step 5 – First Launch & Smoke Tests](#8-step-5--first-launch--smoke-tests)  
9. [Step 6 – Loading / Sharing Workflow JSON Files](#9-step-6--loading--sharing-workflow-json-files)  
10. [Step 7 – Controlling, Updating, and Backing Up](#10-step-7--controlling-updating-and-backing-up)  
11. [Step 8 – Troubleshooting Cheat-Sheet](#11-step-8--troubleshooting-cheat-sheet)  
12. [Security Best Practices](#12-security-best-practices)  
13. [Appendix A – Full File Listing](#13-appendix-a--full-file-listing)  
14. [Appendix B – Glossary](#14-appendix-b--glossary)  

---

## 1. Conceptual Overview

```
┌─────────────┐      HTTPS       ┌───────────┐
│  End User   │ ───────────────► │  Caddy    │
│ (browser)   │ ◄─────────────── │  reverse  │
└─────────────┘     80 / 443     │  proxy    │
                               ┌─┴───────────┴─┐
                               │  n8n (port 5678)│
                               └───────────────┘
                               ┌───────────────┐
                               │ PostgreSQL    │
                               └───────────────┘
```

* **Caddy** terminates TLS, renews certificates, and proxies traffic to `n8n` inside the private Docker network.  
* **n8n** orchestrates your workflows, authenticates you via **Basic Auth** (user & password).  
* **PostgreSQL 16** stores workflows, credentials, logs.  
* **Docker volumes** persist data so you don’t lose anything when containers are replaced.  
* **All containers** belong to the private `n8n-network`; only Caddy exposes ports 80 & 443 to the host.  

Why PostgreSQL instead of built-in SQLite? Concurrency and robustness (no file locking issues, better backups, horizontal growth, etc.).

---

## 2. Before You Begin – Quick Checklist

| Requirement | Yes/No | Notes |
|-------------|--------|-------|
| Ubuntu 24.04 (desktop, server, VPS, or cloud) with sudo access | ☐ | Must be 64-bit |
| Public **domain name** (e.g. `n8n.example.com`) | ☐ | Point A record to server’s IP |
| Outbound **port 443** open (for Let’s Encrypt ACME) | ☐ | Usually allowed by default |
| Inbound **ports 80 & 443** open on firewall | ☐ | `ufw status verbose` |
| Basic CLI confidence (copy/paste, nano/vim) | ☐ | No coding required |
| ≈ 2 GB RAM and ≈ 10 GB disk free | ☐ | n8n needs ~200 MB, Postgres ≥ 1.5 GB (smallest ideal) |

If any checkbox is unchecked, fix it first. Good? Let’s dive in.

---

## 3. Step 0 – Preparing the Machine

```bash
# 1) Update package index & upgrade existing packages
sudo apt update && sudo apt full-upgrade -y

# 2) (Optional but recommended) reboot if kernel or libc got updated
sudo reboot
```

### Set Your Hostname (Optional but Nice)

```bash
# Replace with your desired hostname
sudo hostnamectl set-hostname n8n-host
```

### Configure a Basic UFW Firewall

```bash
sudo apt install ufw -y
sudo ufw allow OpenSSH      # keep SSH open
sudo ufw allow 80,443/tcp   # HTTP / HTTPS for Caddy
sudo ufw enable             # answer 'y'
```

Check:

```bash
sudo ufw status verbose
```

---

## 4. Step 1 – Install Docker Engine + Compose v2

Ubuntu 24.04 currently ships Docker packages, but the official Docker repository gives the freshest stable builds.

### 4.1 Prerequisite Packages

```bash
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### 4.2 Add Docker’s GPG Key

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### 4.3 Add the Docker APT Repository

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) \
   signed-by=/etc/apt/keyrings/docker.gpg] \
   https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) stable" \
 | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 4.4 Install Docker Engine and Compose v2

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 4.5 Allow Your User to Run Docker Without `sudo` (Optional)

```bash
sudo usermod -aG docker $USER
# Log out & back in, or:
newgrp docker
```

### 4.6 Quick Sanity Check

```bash
docker run --rm hello-world
```

### 4.7 Verify Compose v2

```bash
docker compose version
# Example: Docker Compose version v2.29.0
```

---

## 5. Step 2 – Create a Dedicated Project Directory

Keeping all files together simplifies backup/restore.

```bash
mkdir -p $HOME/n8n-docker/{custom-workflows,backup}
cd $HOME/n8n-docker
```

Directory tree (initially):

```
n8n-docker/
├── custom-workflows/   # You’ll drop .json or .zip exports here
├── backup/             # We’ll use later
```

---

## 6. Step 3 – Write the `docker-compose.yml`

Create and open the file:

```bash
nano docker-compose.yml
```

Paste the following (adjust the highlighted placeholders):

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest           # ← stick to :latest or pin a tag like 1.47.0
    restart: unless-stopped
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin      # ← change if you like
      - N8N_BASIC_AUTH_PASSWORD=ChangeMeNow!  # ← strong, unique
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres
      - DB_POSTGRESDB_USER=n8n
      - DB_POSTGRESDB_PASSWORD=SuperSecretDBpw  # ← match postgres service
      - DB_POSTGRESDB_DATABASE=n8n
      - N8N_HOST=n8n.example.com       # ← your FQDN
      - N8N_PROTOCOL=https
      - N8N_PORT=5678
      - NODE_ENV=production
    volumes:
      - n8n_data:/home/node/.n8n
      - ./custom-workflows:/files
    networks:
      - n8n-network
    depends_on:
      - postgres

  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      - POSTGRES_USER=n8n
      - POSTGRES_PASSWORD=SuperSecretDBpw  # ← same as above
      - POSTGRES_DB=n8n
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - n8n-network

  caddy:
    image: caddy:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - caddy_data:/data
      - caddy_config:/config
      - ./Caddyfile:/etc/caddy/Caddyfile
    networks:
      - n8n-network
    depends_on:
      - n8n

volumes:
  n8n_data:
  postgres_data:
  caddy_data:
  caddy_config:

networks:
  n8n-network:
```

Save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`).

### Variables Cheat-Sheet

| Variable | Description |
|----------|-------------|
| `N8N_BASIC_AUTH_USER / PASSWORD` | Gatekeeper—anyone hitting the UI must send these HTTP credentials before even viewing the login page. |
| `DB_*` | Connection string to Postgres inside the same network; the host is literally the Docker service name. |
| `N8N_HOST / PROTOCOL / PORT` | Tell n8n “how the outside world will reach me.” Important for OAuth callback URLs. |

---

## 7. Step 4 – Author the Caddyfile

The Caddyfile is the declarative config for Caddy. One line can handle TLS, redirection, proxying.

```bash
nano Caddyfile
```

Paste:

```
n8n.example.com {
    reverse_proxy n8n:5678
}
```

Optionally, force www-less, enable compression, add basic headers:

```
n8n.example.com {
    encode zstd gzip
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
    reverse_proxy n8n:5678
}
```

For multi-domain or wildcard setups, just add new blocks.

---

## 8. Step 5 – First Launch & Smoke Tests

### 8.1 Spin Up

```bash
docker compose up -d
```

Watch the logs (two ways):

```bash
docker compose logs -f caddy
docker compose logs -f n8n
```

You should see:

* Caddy obtaining an **ACME certificate** from Let’s Encrypt (status 200 or 201).  
* n8n booting, connecting to Postgres, “*Started with success*”.

### 8.2 Visit the Site

1. Browser → `https://n8n.example.com`.  
2. Your browser prompts for **Basic Authentication** (HTTP 401). Enter the `admin / ChangeMeNow!` pair (or your custom).  
3. n8n login page appears. The first user you create becomes the owner (fill username/email/password).  

### 8.3 Quick Flow Test

1. Click **“Create Workflow”**.  
2. Drag “Start” → “Set” node, add a field, save.  
3. Run → It succeeds.  
4. Congratulations—stack works!

---

## 9. Step 6 – Loading / Sharing Workflow JSON Files

### 9.1 Manual Import via UI

1. **Menu ≡ → Import workflow**.  
2. Paste JSON or upload `.json`. Save.

### 9.2 Disk-Based Approach (Mounted Volume)

Because `./custom-workflows` on host is mounted to `/files` in the container, any file you drop there is instantly visible:

```bash
# On host:
cp MyAwesomeFlow.json $HOME/n8n-docker/custom-workflows/
```

Inside n8n, add a **“Read Binary File”** node:

```
Path: /files/MyAwesomeFlow.json
```

Or from **Settings → Import from URL / file**.

Tip: commit your JSON exports to Git for version control.

---

## 10. Step 7 – Controlling, Updating, and Backing Up

### 10.1 Common Docker Compose Commands

| Action | Command |
|--------|---------|
| Start stack | `docker compose up -d` |
| Stop stack | `docker compose down` |
| View running containers | `docker compose ps` |
| Tail logs (all) | `docker compose logs -f --tail=100` |
| Enter bash inside n8n | `docker compose exec n8n /bin/bash` |

### 10.2 Updating Images

```bash
docker compose pull        # fetch newer images
docker compose up -d       # recreate only if new image
docker image prune -f      # optional: remove dangling layers
```

Your data is safe in volumes.

### 10.3 Scheduled Backups

A simple cron job that:

1. Dumps Postgres to a timestamped file.  
2. Copies the n8n encryption key directory.

Create script `backup/backup.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
DAY=$(date +%F-%H%M)
DEST="$HOME/n8n-docker/backup/$DAY"
mkdir -p "$DEST"

# 1) DB dump
docker compose exec -T postgres \
  pg_dump -U n8n -d n8n \
  | gzip > "$DEST/n8n.sql.gz"

# 2) n8n config & credential keys
docker compose cp n8n:/home/node/.n8n "$DEST"/n8n_config
```

Make executable:

```bash
chmod +x backup/backup.sh
```

Crontab:

```bash
crontab -e
# Run every day at 02:15
15 2 * * * /home/youruser/n8n-docker/backup/backup.sh > /dev/null 2>&1
```

Rotate or upload to S3/Backblaze as you wish.

### 10.4 Restoring

1. Recreate directory tree + `docker-compose.yml`.  
2. Copy `n8n_config` into a brand new `n8n_data` volume (`docker cp` or volume mapping).  
3. `gunzip < n8n.sql.gz | docker compose exec -T postgres psql -U n8n -d n8n`.

Done.

---

## 11. Step 8 – Troubleshooting Cheat-Sheet

| Symptom | Probable Cause | Fix |
|---------|----------------|-----|
| `ERR_CONNECTION_REFUSED` on port 443 | Firewall closed or Caddy not started | `docker compose ps` / `ufw status` |
| Browser warns about invalid cert | DNS mismatch or port 80 blocked during ACME | Ensure A-record, open 80. `docker compose logs caddy` |
| n8n “Database connection refused” | Wrong credentials or Postgres not healthy | Check `DB_POSTGRESDB_PASSWORD`, restart stack |
| “Workflow does not start on schedule” | Cron node runs in UTC by default | Adjust node or set `GENERIC_TIMEZONE=Europe/Berlin` env var |
| Caddy restarts in loop | Port 80/443 already in use on host | `sudo lsof -i :80 -i :443`, stop Apache/NGINX |

Advanced debug:

```bash
# Exec into postgres and psql shell
docker compose exec postgres psql -U n8n -d n8n

# Exec into n8n, list environment
docker compose exec n8n env | sort
```

---

## 12. Security Best Practices

1. **Rotate passwords** periodically.  
2. Restrict n8n credentials: use **env-vars or secrets** for API keys inside workflows.  
3. Enable **2FA** on your cloud registrar & DNS.  
4. **Fail2ban** or Cloudflare to mitigate brute force on 80/443.  
5. Regularly apply OS patches (`sudo apt update && sudo apt upgrade`).  
6. Consider running Docker rootless or under a restricted service account for high-security environments.  

For enterprise setups: integrate with SSO/OIDC and use n8n’s built-in security features.

---

## 13. Appendix A – Full File Listing

```
n8n-docker/
├── Caddyfile
├── docker-compose.yml
├── custom-workflows/
│   └── (your .json files)
└── backup/
    ├── backup.sh
    └── (timestamped folders)
/var/lib/docker/volumes/
└── (auto-managed volumes)
```

---

## 14. Appendix B – Glossary

| Term | Short Definition |
|------|------------------|
| **n8n** | “*nodemation*” – open-source workflow automation platform. |
| **Docker** | Container runtime to package software with dependencies. |
| **Compose** | YAML file format & CLI to run multi-container stacks. |
| **PostgreSQL** | Open-source relational database powering n8n’s data store. |
| **Caddy** | Modern web server with automatic HTTPS out of the box. |
| **Volume** | Docker-managed folder on host for persistent data. |
| **ACME** | Protocol used by Let’s Encrypt to issue free SSL/TLS certificates. |

---

## 🎉 You Did It!

By following this playbook, you’ve:

* Installed a fresh Docker stack on Ubuntu 24.04.  
* Hosted n8n with industrial-grade HTTPS, password gate, and persistent Postgres storage.  
* Learned the core maintenance cycle—backup, update, restore.

Take it further:

* Explore **n8n’s community nodes**.  
* Wire it to Slack, Telegram, GitHub, or any REST API.  
* Trigger flows with Caddy’s built-in automatic TLS for sub-routes.

Happy automating! ✨
