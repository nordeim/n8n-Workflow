# Step-by-Step Guide: Deploying n8n with Docker on Ubuntu 24.04.01 and Running a Custom Workflow

---

## Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Step 1: Preparing Your Ubuntu Server](#step-1-preparing-your-ubuntu-server)
4. [Step 2: Installing Docker](#step-2-installing-docker)
5. [Step 3: Installing Docker Compose](#step-3-installing-docker-compose)
6. [Step 4: Pulling the n8n Docker Image](#step-4-pulling-the-n8n-docker-image)
7. [Step 5: Setting Up Persistent Storage](#step-5-setting-up-persistent-storage)
8. [Step 6: Creating an n8n Docker Compose File](#step-6-creating-an-n8n-docker-compose-file)
9. [Step 7: Configuring Environment Variables](#step-7-configuring-environment-variables)
10. [Step 8: Starting the n8n Container](#step-8-starting-the-n8n-container)
11. [Step 9: Accessing the n8n Web Interface](#step-9-accessing-the-n8n-web-interface)
12. [Step 10: Importing Your Workflow JSON](#step-10-importing-your-workflow-json)
13. [Step 11: Connecting Credentials and APIs](#step-11-connecting-credentials-and-apis)
14. [Step 12: Testing and Running the Workflow](#step-12-testing-and-running-the-workflow)
15. [Step 13: (Optional) Securing Your n8n Instance](#step-13-optional-securing-your-n8n-instance)
16. [Step 14: (Optional) Setting Up a Reverse Proxy (NGINX)](#step-14-optional-setting-up-a-reverse-proxy-nginx)
17. [Step 15: (Optional) Enabling HTTPS](#step-15-optional-enabling-https)
18. [Step 16: Managing and Updating Your n8n Instance](#step-16-managing-and-updating-your-n8n-instance)
19. [Troubleshooting Common Issues](#troubleshooting-common-issues)
20. [Resources and Further Reading](#resources-and-further-reading)

---

## Introduction

This guide will walk you step-by-step through the process of running [n8n](https://n8n.io/)—a powerful, open source workflow automation tool—inside a Docker container on Ubuntu 24.04.01 LTS. Following this guide, you will:

- Deploy a fully functional n8n instance on your local or cloud Ubuntu server
- Use Docker and Docker Compose for easy setup and management
- Import and run any n8n workflow JSON, including advanced AI and vector database automations like RAG, Pinecone, and OpenAI
- Learn best practices for persistence, security, and production-readiness

This tutorial is designed to be accessible for users new to Docker or Linux server administration. All commands are copy-paste ready and explained in plain language.

---

## Prerequisites

- **A running Ubuntu 24.04.01 LTS server** (physical, VM, or cloud)
- **A user account with sudo privileges**
- **An internet connection**
- **A computer with a web browser** (for accessing the n8n UI)
- **(Optional) n8n workflow JSON file** (your custom automation)

> **No programming or advanced IT knowledge required!**

---

## Step 1: Preparing Your Ubuntu Server

Before installing anything, make sure your server is up-to-date.

```bash
sudo apt update && sudo apt upgrade -y
```

This updates your package list and installs the latest security patches.

> **Tip:** If you're on a cloud VM, connect via SSH. Example:
> ```bash
> ssh your-user@your-server-ip
> ```

---

## Step 2: Installing Docker

Docker makes it easy to run applications in containers. We'll install the official Docker package.

### 2.1 Remove Old Versions (if any)

```bash
sudo apt remove docker docker-engine docker.io containerd runc
```

### 2.2 Install Required Dependencies

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

### 2.3 Add Docker’s Official GPG Key

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### 2.4 Add Docker Repository

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2.5 Install Docker Engine

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 2.6 Verify Docker Installation

```bash
sudo docker run hello-world
```

You should see a message: “Hello from Docker!”

> **Reference:**  
> [Install Docker Engine on Ubuntu (Official Docs)](https://docs.docker.com/engine/install/ubuntu/)

---

## Step 3: Installing Docker Compose

Docker Compose lets you define and run multi-container apps using a simple YAML file.  
**For Ubuntu 24.04, Docker Compose is included as a plugin, but you can install the standalone tool if preferred.**

### 3.1 Check Docker Compose Version

```bash
docker compose version
```
or
```bash
docker-compose version
```

If you see a version (e.g., `Docker Compose version v2.29.0`), you’re ready.

### 3.2 (Optional) Install Standalone Compose

```bash
sudo apt install -y docker-compose
```

---

## Step 4: Pulling the n8n Docker Image

We'll use the official [n8n Docker image](https://hub.docker.com/r/n8nio/n8n).

```bash
sudo docker pull n8nio/n8n
```

This downloads the latest image.

> **Tip:** You can check for the latest tags at [Docker Hub](https://hub.docker.com/r/n8nio/n8n/tags).

---

## Step 5: Setting Up Persistent Storage

n8n stores data (workflows, credentials, executions) locally. **Persisting this data is critical**—otherwise, upgrades or container restarts will erase your work.

### 5.1 Create Persistent Folders

```bash
sudo mkdir -p /home/$USER/n8n/.n8n
sudo chown -R $USER:$USER /home/$USER/n8n
```

- `/home/$USER/n8n/.n8n/` will store all n8n data.

---

## Step 6: Creating an n8n Docker Compose File

Docker Compose makes managing n8n easy and repeatable.

### 6.1 Create `docker-compose.yml` File

Navigate to your n8n directory:

```bash
cd /home/$USER/n8n
```

Create the file:

```bash
nano docker-compose.yml
```

Paste the following (basic example):

```yaml
version: "3.8"
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    volumes:
      - ./.n8n:/home/node/.n8n
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=strong_password_here
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_EDITOR_BASE_URL=http://localhost:5678
      - WEBHOOK_URL=http://localhost:5678
      # Add more environment variables as needed, e.g., for credentials
```

- Change `N8N_BASIC_AUTH_USER` and `N8N_BASIC_AUTH_PASSWORD` to your chosen username and password.
- For remote access, replace `localhost` with your server’s hostname or public IP.

**Save and exit (Ctrl+O, Enter, Ctrl+X).**

> **Reference:**  
> [n8n Docker Compose Reference](https://docs.n8n.io/hosting/docker/#using-docker-compose)

---

## Step 7: Configuring Environment Variables

Environment variables let you securely configure n8n and pass secrets.

- **Common variables:**  
  - `N8N_BASIC_AUTH_ACTIVE` (enable basic auth)
  - `N8N_ENCRYPTION_KEY` (encrypt credentials—generate a strong random string)
  - `N8N_HOST`, `N8N_PORT`, `WEBHOOK_URL`
  - (Optional) API keys, DB connection strings, etc.

**Example:**

```yaml
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=chooseAStrongPassword
      - N8N_ENCRYPTION_KEY=supersecretkey1234567890
```

> **Tip:** For sensitive variables, use a `.env` file.  
> [n8n Docs: Configuration via environment variables](https://docs.n8n.io/hosting/environment-variables/)

---

## Step 8: Starting the n8n Container

From your `/home/$USER/n8n` directory, launch n8n:

```bash
docker compose up -d
```

- The `-d` flag runs the container in the background (“detached” mode).

**Check that n8n is running:**

```bash
docker compose ps
```

You should see the `n8n` service running.

**To see logs:**

```bash
docker compose logs -f
```

---

## Step 9: Accessing the n8n Web Interface

- Open your browser and visit:  
  `http://<your-server-ip>:5678`
- If you’re on the same machine, use:  
  `http://localhost:5678`

**Log in** using the username and password you set in the Compose file.

> **If you can't connect:**  
> - Make sure port 5678 is open (check firewall)
> - Run `docker compose logs` for errors
> - For cloud VMs, add a security rule for TCP port 5678

---

## Step 10: Importing Your Workflow JSON

You can now import any n8n workflow (including advanced AI and vector workflows):

### 10.1 Get Your Workflow JSON

- Save your workflow JSON file (e.g., `myworkflow.json`) on your computer.

### 10.2 Import via the n8n UI

1. **Open the n8n Editor (web UI)**
2. Click the menu button (“hamburger” ☰) at the top-left
3. Choose **Workflows > Import from File**
4. Select your `myworkflow.json` and upload
5. The workflow will appear on your canvas

**Alternatively**, you can copy the JSON and use **Import from Clipboard**.

> **Reference:**  
> [n8n Docs: Importing Workflows](https://docs.n8n.io/workflows/import-export/)

---

## Step 11: Connecting Credentials and APIs

Your workflow may require API keys (e.g., OpenAI, Pinecone, Google Drive).

### 11.1 Add API Credentials

1. In the n8n UI, click the **Credentials** icon (key symbol) on the left
2. Click **Create New**
3. Select the service (e.g., OpenAI, Google Drive, Pinecone)
4. Enter your credentials (API key, OAuth, etc.)
5. Save

**Important:**  
- Match the credential names in your workflow JSON to those in n8n.
- You can edit nodes to select the correct credential if needed.

### 11.2 (Optional) Use Environment Variables for Sensitive Data

- Many credentials can reference environment variables for secrets.

---

## Step 12: Testing and Running the Workflow

### 12.1 Manual Trigger

- If your workflow uses a **Manual Trigger**, click “Execute Workflow” in the UI.

### 12.2 Webhook or Chat Trigger

- For webhooks, copy the webhook URL from the **Webhook** node and test it (e.g., with [Postman](https://www.postman.com/) or `curl`).

### 12.3 Monitor and Debug

- Use the **Execution List** (left sidebar) to view past runs, errors, and outputs.
- Click any node to inspect its data.

### 12.4 Activate for Automation

- Click the “Active” toggle at the top of the workflow to enable automatic triggers.

---

## Step 13: (Optional) Securing Your n8n Instance

**Production n8n should always be secured.**

- Use strong `N8N_BASIC_AUTH_PASSWORD`
- Set `N8N_ENCRYPTION_KEY` for credential encryption
- Restrict access with firewall rules (allow only specific IPs)
- Never expose n8n directly to the internet without authentication

> **Reference:**  
> [n8n Security Best Practices](https://docs.n8n.io/security/best-practices/)

---

## Step 14: (Optional) Setting Up a Reverse Proxy (NGINX)

To run n8n on standard HTTPS (port 443) or behind a domain, use a reverse proxy.

### 14.1 Install NGINX

```bash
sudo apt install -y nginx
```

### 14.2 Configure a Reverse Proxy

Edit `/etc/nginx/sites-available/n8n`:

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the config:

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

> **Reference:**  
> [n8n Docs: Reverse Proxy](https://docs.n8n.io/hosting/reverse-proxy/)

---

## Step 15: (Optional) Enabling HTTPS

Use Let’s Encrypt for free SSL certificates.

### 15.1 Install Certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
```

### 15.2 Obtain and Install Certificate

```bash
sudo certbot --nginx -d your-domain.com
```

- Follow prompts to enable HTTPS.

---

## Step 16: Managing and Updating Your n8n Instance

### 16.1 Stopping and Starting n8n

```bash
docker compose stop   # Stop the container
docker compose start  # Start the container
```

### 16.2 Restarting n8n

```bash
docker compose restart
```

### 16.3 Updating n8n

1. Pull the latest image:
   ```bash
   docker pull n8nio/n8n:latest
   ```
2. Recreate the container:
   ```bash
   docker compose down
   docker compose up -d
   ```

**Your data is preserved in `/home/$USER/n8n/.n8n`.**

---

## Troubleshooting Common Issues

### 1. **Port 5678 is not accessible**

- Check server firewall:  
  `sudo ufw allow 5678/tcp`
- For cloud VMs, add a rule for port 5678

### 2. **Web UI doesn’t load**

- Run `docker compose logs` to check for errors
- Ensure Docker and Docker Compose are installed and running

### 3. **Workflow import fails**

- Ensure your JSON is valid and not truncated
- Try importing via clipboard if file import fails

### 4. **Credentials not working**

- Double-check your API keys and OAuth flows
- Make sure credential names match those referenced in workflow nodes

### 5. **n8n container keeps restarting**

- Check logs for environment variable errors or permission issues on `.n8n` folder

---

## Resources and Further Reading

- [Official n8n Docker Docs](https://docs.n8n.io/hosting/docker/)
- [n8n Environment Variables](https://docs.n8n.io/hosting/environment-variables/)
- [n8n Community Forum](https://community.n8n.io/)
- [n8n Example Workflows](https://n8n.io/workflows)
- [n8n Security Guide](https://docs.n8n.io/security/)
- [Docker Documentation](https://docs.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [Let’s Encrypt (Certbot)](https://certbot.eff.org/)

---

## Final Notes

You now have a robust, production-ready n8n setup running in Docker on Ubuntu 24.04.01. You can:

- Import and run any n8n workflow JSON
- Add integrations (APIs, LLMs, vector DBs, etc.)
- Secure and manage your automations with industry best practices

**If you get stuck, the [n8n community](https://community.n8n.io/) is friendly and helpful!**

---

**Congratulations on automating your world with n8n!**