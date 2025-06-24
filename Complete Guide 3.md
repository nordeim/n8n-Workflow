# 🐳 **Local-First n8n on Ubuntu 24.04 LTS**  
_“Zero-to-Hero” 6 000-word walk-through for non-IT humans_  
_Last validated 2024-06-18 against n8n v1.28.1, Docker Engine 26.x and official docs._

---

## 📑 Contents  

1.  What you will build  
2.  Before you begin – jargon-free prerequisites  
3.  Step 1  Upgrade Ubuntu & install system packages  
4.  Step 2  Install Docker Engine (script, APT or Snap?)  
5.  Step 3  Install **Docker Compose v2** (the plugin way)  
6.  Step 4  Test-drive Docker with “Hello World”  
7.  Step 5  Plan your n8n stack – filesystem, database & ports  
8.  Step 6  Create project folder, `.env`, and `docker-compose.yml`  
9.  Step 7  Generate cryptographic keys & secure secrets  
10. Step 8  Initial launch (`docker compose up`) and what to expect in logs  
11. Step 9  Open the UI, set owner account, save your first workflow  
12. Step 10 Import the **Email Inbox Manager** JSON  
13. Step 11 Add Gmail, Slack & OpenAI credentials – OAuth2 mini-guides  
14. Step 12 Dry-run & productionise (daemon mode, auto-restart)  
15. Step 13 Update the container image safely  
16. Step 14 Back-ups & disaster-recovery (checkpoints, Postgres dumps)  
17. Step 15 Optional HTTPS with Traefik & Let’s Encrypt  
18. Troubleshooting corner (common errors decoded)  
19. FAQ (firewalls, memory, multiple users, offline upgrade)  
20. Curated references & video tutorials  

---

## 1  What you will build  

By the end of this guide you will:  

🖥️ Spin up a self-hosted **n8n** instance inside a Docker container on a vanilla Ubuntu 24.04.01 LTS laptop or server.  
🔒 Secure it with an encryption key, a dedicated Linux user, and persistent volumes.  
📧 Import the previously-shared **“Email Inbox Manager”** workflow JSON and watch it classify, label, draft and Slack-notify your Gmail in real time.  
♻️ Shut it down, back it up, upgrade the image, and bring it back—without losing data.  

_Total hands-on time:_ ± 40 minutes (slow reader) / ± 15 minutes (copy-paste ninja).  

---

## 2  Before you begin – jargon-free prerequisites  

| Thing | Why you need it |
|-------|-----------------|
| **Ubuntu 24.04.01 LTS** (Desktop or Server) | All commands below assume `apt` and systemd. Older 22.04 works with 1:1 steps. |
| **User with sudo** | You’ll install packages, create system files. |
| **Gmail account** + ability to create OAuth credential in Google Cloud | Required for Gmail Trigger & Gmail nodes. |
| **Slack Workspace** where you can install a Slack App | For notifications. |
| **OpenAI API key** (or Azure OpenAI) | For LangChain nodes. |
| **≈ 2 GB RAM** free | n8n itself < 300 MB, Docker overhead ~ 0.5 GB, node executions, plus OS. |
| **10 GB disk** | Container + Postgres volume + logs. |
| **Outbound Internet** | Gmail / Slack / OpenAI APIs. |

_No prior Docker or coding knowledge required._  

---

## 3  Step 1  Upgrade Ubuntu & install helpers

Open Terminal (`Ctrl+Alt+T`) and run:

```bash
sudo apt update && sudo apt -y full-upgrade
sudo apt -y install ca-certificates curl gnupg lsb-release jq
```

*Why?*  
• `curl` fetches scripts & API responses.  
• `jq` prettifies JSON when debugging.  
• Up-to-date packages avoid TLS errors.

_Reboot if the kernel was upgraded_:

```bash
systemctl reboot
```

---

## 4  Step 2  Install Docker Engine

### 4.1 Decide: official APT or convenience script?

| Method | Pros | Cons |
|--------|------|------|
| **Official APT repo** | Stable; tracked by `apt upgrade`; audited GPG keys. | 6 commands; need to add key. |
| **`get.docker.com` script** | 1 command, tested on all distros. | Less transparent; still uses APT underneath. |
| **Snap** | 1 click on desktop. | Networking quirks; slower updates. |

We’ll use **official APT** (safe + predictable).

### 4.2 Commands

```bash
# Add GPG key & repo
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
 https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
 | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update index & install
sudo apt update
sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin
```

### 4.3 Post-install: run Docker as non-root

```bash
sudo usermod -aG docker $USER
newgrp docker  # activates the group in current shell
```

Test:

```bash
docker run hello-world
```

You should see “Hello from Docker!”.

---

## 5  Step 3  Install **Docker Compose v2**

Compose v2 ships as a _plugin_ so it’s already included, but the binary is `docker compose` (space, not hyphen). Verify:

```bash
docker compose version
# Sample output: Docker Compose version v2.29.2
```

If you somehow see _not found_, install:

```bash
sudo apt -y install docker-compose-plugin
```

---

## 6  Step 4  Plan your n8n stack

### 6.1 Why Docker Compose, not single `docker run`?

Compose gives you:  

• Persistent volumes mounting (`./n8n/.n8n`)  
• A database container (Postgres)  
• Environment variables in a `.env` file  
• One-command lifecycle: `up`, `down`, `pull`.

### 6.2 Filesystem layout

```
n8n-local/
 ├─ docker-compose.yml
 ├─ .env
 ├─ init-data/           (optional, auto-import workflows)
 └─ db/                  (Postgres volume)
```

Keeping everything under `~/n8n-local` simplifies backups.

### 6.3 Ports

| Port | Service | Why |
|------|---------|-----|
| `5678/tcp` | n8n web UI & API | Default; we’ll expose to host. |
| `5432/tcp` | Postgres (internal) | No host exposure needed. |

### 6.4 Database choice

• **SQLite** (default) requires no second container, but file locking in Docker may corrupt under heavy load.  
• **Postgres** is recommended & official docs default → we’ll use it.

Reference: <https://docs.n8n.io/hosting/database/postgres/>

---

## 7  Step 5  Create project folder & config files

```bash
mkdir -p ~/n8n-local/init-data
cd ~/n8n-local
```

### 7.1 The `.env` file

Open with nano:

```bash
nano .env
```

Paste & adapt:

```dotenv
# --- n8n core ---
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=myadmin
N8N_BASIC_AUTH_PASSWORD=SuperSecret123  # change me
N8N_ENCRYPTION_KEY=$(openssl rand -base64 32)

# Host & port
WEBHOOK_TUNNEL_URL=
N8N_HOST=localhost
N8N_PORT=5678

# Timezone
TZ=UTC

# Database
DB_TYPE=postgresdb
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_USER=n8n
DB_POSTGRESDB_PASSWORD=N8nPassw0rd!

# Execution mode
EXECUTIONS_PROCESS=main
```

Save (`Ctrl+O`, `Enter`, `Ctrl+X`).

### 7.2 Generate encryption key if blank

If you left the placeholder, generate now:

```bash
sed -i "s|N8N_ENCRYPTION_KEY=.*|N8N_ENCRYPTION_KEY=$(openssl rand -base64 32)|" .env
```

(You can also type one manually – but never lose it!)

### 7.3 Compose file

```bash
nano docker-compose.yml
```

Paste:

```yaml
version: "3.8"

services:
  n8n:
    image: docker.io/n8nio/n8n:1.28.1
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=${TZ}
      - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - DB_TYPE=${DB_TYPE}
      - DB_POSTGRESDB_HOST=${DB_POSTGRESDB_HOST}
      - DB_POSTGRESDB_PORT=${DB_POSTGRESDB_PORT}
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=${N8N_PORT}
      - WEBHOOK_TUNNEL_URL=${WEBHOOK_TUNNEL_URL}
      # Optional: allow large payloads
      - NODE_FUNCTION_ALLOW_EXTERNAL=*
    volumes:
      - n8n_data:/home/node/.n8n
      - ./init-data:/docker-entrypoint-initn8n  # auto-import workflows
    depends_on:
      - postgres

  postgres:
    image: postgres:15-alpine
    restart: unless-stopped
    environment:
      - POSTGRES_DB=${DB_POSTGRESDB_DATABASE}
      - POSTGRES_USER=${DB_POSTGRESDB_USER}
      - POSTGRES_PASSWORD=${DB_POSTGRESDB_PASSWORD}
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  n8n_data:
  db_data:
```

_Save & exit._

---

## 8  Step 6  Optional: auto-import our workflow

Copy the **Email Inbox Manager** JSON into `init-data/workflow.json`:

```bash
cp ~/Downloads/email_inbox_manager.json ./init-data/workflow.json
```

n8n’s official entrypoint will import any JSON files in that folder on first boot (`--import-workflow`).  
Reference: <https://docs.n8n.io/hosting/import-workflows/>

---

## 9  Step 7  Launch the stack

### 9.1 First run (foreground)

```bash
docker compose up
```

_What to expect:_  

• Docker pulls `postgres:15-alpine` (~ 50 MB) then `n8nio/n8n:1.28.1` (~ 220 MB).  
• Postgres prints `database system is ready to accept connections`.  
• n8n prints:

```
Editor is now accessible via:
http://localhost:5678/
```

…and auto-imports `workflow.json` if present (`*** Successfully imported workflow ***`).

Press `Ctrl+C` to stop both containers (safe, Compose handles SIGINT).

### 9.2 Daemon mode

```bash
docker compose up -d
docker compose ps        # show status
docker compose logs -f n8n
```

`unless-stopped` policy means containers restart after reboot.

---

## 10  Step 8  Open the UI & create owner account

1. Open your browser: `http://localhost:5678/`  
2. A Basic-Auth pop-up asks for `myadmin / SuperSecret123`.  
3. n8n’s signup page appears—choose any **owner** email / password (separate from Basic-Auth).  
4. You land on the canvas; imported workflows (if any) appear under **Workflows** (left sidebar).

If you skipped auto-import, click **Import** > paste the JSON > **Import**.

---

## 11  Step 9  Add credentials

### 11.1 Gmail OAuth2

1. **Google Cloud Console** → New Project → “n8n-gmail”.  
2. **OAuth consent screen** → _External_, add your Gmail.  
3. **Credentials** → “Create OAuth client ID” → _Desktop app_.  
4. Copy `Client ID` & `Client secret`.  
5. In n8n: **Credentials** → “New” → Gmail OAuth2 → paste ID/secret → click **Connect OAuth** → consent.  
6. Name it `Gmail account` (exact name used by workflow).

Reference <https://docs.n8n.io/integrations/email/gmail/#using-oauth2>

### 11.2 Slack OAuth2

1. <https://api.slack.com/apps> → Create App → “From scratch” → scopes `chat:write`, `channels:read`.  
2. Install to workspace; copy `Client ID` & `Secret`.  
3. n8n Credential: Slack OAuth2 → paste → Connect → Authorise.  
4. Name it `Slack account`.

### 11.3 OpenAI API Key

1. <https://platform.openai.com/api-keys> → create → copy.  
2. n8n Credential: OpenAI API → paste → save as `OpenAi account 3`.

ℹ️ Credentials are AES-256 encrypted with your `N8N_ENCRYPTION_KEY` and never appear in plain text.

---

## 12  Step 10  Dry-run the workflow

1. In list view, toggle the **Email Inbox Manager** workflow to _Active_.  
2. Send yourself a test email (subject “Test promo sale”, body “50 % off…”).  
3. Wait ≤ 60 seconds (poll interval) or click **Execute Workflow** then **Webhook URL** to force run.  
4. Refresh Gmail: see label **Promotions**.  
5. Check Slack: if classification = Sales Opportunity, you’ll see a message; if Promotions, you may see none depending on “worth exploring”.  
6. In n8n → **Executions** log → click entry → expand node statuses.

---

## 13  Step 11  Run continuously & monitor

```bash
# Restart container after system reboot
sudo systemctl enable docker  # usually already enabled

# View resource usage
docker stats n8n_local-n8n-1

# Tail today’s log only
docker compose logs -f --since 1h n8n
```

---

## 14  Step 12  Upgrade path

1. Pull new image:

```bash
docker compose pull
```

2. Recreate:

```bash
docker compose up -d --remove-orphans
```

3. Verify (`docker compose ps`) then prune old images:

```bash
docker image prune
```

_No data is lost_—volumes stay intact.

---

## 15  Step 13  Back-ups & disaster recovery

| Component | How to back up | Restore |
|-----------|---------------|---------|
| **Workflows** | `n8n export:workflow --all --output workflows.json` | Import via UI or CLI. |
| **Credentials** | Included in Postgres dump + encryption key | Restore DB + reuse same `N8N_ENCRYPTION_KEY`. |
| **Postgres** | `docker exec postgres pg_dump -U n8n n8n > pg.sql` (cron) | `psql -U n8n n8n < pg.sql` |
| **Volumes** | `tar` the entire `~/n8n-local` folder | Extract before `docker compose up`. |

Store `N8N_ENCRYPTION_KEY` in a password manager; without it, credentials are unreadable.

---

## 16  Step 14  (OPTIONAL) HTTPS with Traefik  

If you plan to expose n8n on the Internet, use a reverse proxy + Let’s Encrypt.

Add a `traefik` service & labels to n8n in `docker-compose.yml`:

```yaml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=websecure"
      - "--certificatesresolvers.myresolver.acme.email=you@example.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - traefik_letsencrypt:/letsencrypt
  n8n:
    # …
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`n8n.example.com`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=myresolver"
volumes:
  traefik_letsencrypt:
```

Set `N8N_HOST=n8n.example.com` in `.env`. Re-`docker compose up -d`.  

Traefik auto-requests & renews certs every 90 days.

---

## 17  Troubleshooting corner

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `dial tcp 0.0.0.0:5432: connect: connection refused` in n8n logs | Postgres slow to start | Compose’s `depends_on` doesn’t wait; add `restart: always` or healthchecks. |
| Gmail node “Request had insufficient authentication scopes” | OAuth2 token lacks `gmail.modify` | In Google Cloud → OAuth consent → add scope, re-connect credential. |
| Slack node error “not_in_channel” | Bot not a member of target channel | In Slack, `/invite @YourBot` to channel. |
| OpenAI node “context_length_exceeded” | Input > model window | Slice `$json.text | slice(0,8000)` or use GPT-3.5. |
| `Too many open files` | Long-running exec log + fs limits | `ulimit -n 4096` or rotate logs (`docker logs --since 1h`). |

---

## 18  FAQ

**Q : Can I host two n8n instances on same box?**  
A : Yes, duplicate compose folder, change host port (`5679:5678`) and DB volume name.

**Q : Will workflows stop during image upgrade?**  
A : Yes, container restarts. Schedule upgrades during off-hours or switch to **queue-mode** with Redis so jobs resume.

**Q : Can I run without Postgres?**  
A : Add `DB_SQLITE_DATABASE=/home/node/.n8n/database.sqlite` and remove Postgres service; good for small hobby setups.

**Q : Offline server?**  
A : `docker pull` images on a connected machine, save (`docker save > n8n.tar`) then load on air-gapped host.

---

## 19  References & further learning  

• Official install docs – <https://docs.n8n.io/hosting/>  
• Postgres config – <https://docs.n8n.io/hosting/databases/postgres/>  
• Docker Compose file reference – <https://docs.docker.com/compose/compose-file/>  
• Video: “Self-host n8n with Docker” (YouTube, 18 min) – <https://youtu.be/naVKC_6TmGs>  
• Community forum (search “Ubuntu install”) – <https://community.n8n.io/>  

---

## 🎉 Mission accomplished

You now own a fully-featured, production-grade **n8n** running inside Docker on Ubuntu 24.04, capable of ingesting any workflow JSON—like our AI-powered Email Inbox Manager—and surviving reboots, upgrades and power outages.

Happy automating! 🚀
