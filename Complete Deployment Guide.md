# Complete Guide: Deploying n8n Workflow Automation with Docker on Ubuntu 24.04.01

## Table of Contents
1. [Introduction and Overview](#introduction-and-overview)
2. [Prerequisites and System Requirements](#prerequisites-and-system-requirements)
3. [Installing Docker on Ubuntu 24.04.01](#installing-docker-on-ubuntu-24-04-01)
4. [Setting up n8n with Docker Compose](#setting-up-n8n-with-docker-compose)
5. [Configuring Environment Variables](#configuring-environment-variables)
6. [Setting up External Services](#setting-up-external-services)
7. [Importing and Configuring the Workflow](#importing-and-configuring-the-workflow)
8. [Setting up Credentials and Integrations](#setting-up-credentials-and-integrations)
9. [Testing the Workflow](#testing-the-workflow)
10. [Security Hardening](#security-hardening)
11. [Maintenance and Updates](#maintenance-and-updates)
12. [Troubleshooting Common Issues](#troubleshooting-common-issues)
13. [Advanced Configuration](#advanced-configuration)
14. [Conclusion](#conclusion)

## Introduction and Overview

This comprehensive guide will walk you through the complete process of setting up n8n (a powerful workflow automation tool) in a Docker container on Ubuntu 24.04.01. By the end of this guide, you'll have a fully functional n8n instance running locally that can execute complex workflows, including the advanced lead processing pipeline we created earlier.

n8n (pronounced "n-eight-n") is an open-source workflow automation tool that allows you to connect different services and automate business processes. Running it in Docker provides several advantages:

- **Isolation**: Your n8n instance runs in its own container, separate from your host system
- **Portability**: Easy to move between different machines or cloud providers
- **Consistency**: Same environment across development, testing, and production
- **Easy Updates**: Simple container replacement for updates
- **Resource Management**: Control over CPU and memory allocation

### What We'll Accomplish

By following this guide, you will:
- Install Docker and Docker Compose on Ubuntu 24.04.01
- Deploy n8n using Docker Compose with persistent data storage
- Configure a PostgreSQL database for workflow execution data
- Set up environment variables and security configurations
- Import and configure the advanced lead processing workflow
- Configure credentials for external service integrations
- Test the complete workflow end-to-end
- Implement security best practices
- Learn maintenance and troubleshooting procedures

### Architecture Overview

Our final setup will include:
- **n8n Container**: Main workflow automation engine
- **PostgreSQL Container**: Database for execution history and workflow data
- **Redis Container**: For queue management and caching (optional but recommended)
- **Reverse Proxy**: Nginx for SSL termination and security
- **Volume Mounts**: Persistent storage for data and configurations

## Prerequisites and System Requirements

### Hardware Requirements

**Minimum Requirements:**
- CPU: 2 cores (2.0 GHz or higher)
- RAM: 4 GB
- Storage: 20 GB available disk space
- Network: Internet connection for downloading containers and accessing external APIs

**Recommended Requirements:**
- CPU: 4 cores (2.5 GHz or higher)
- RAM: 8 GB or more
- Storage: 50 GB SSD storage
- Network: Stable broadband connection

### Software Prerequisites

Before starting, ensure you have:
- Fresh Ubuntu 24.04.01 LTS installation
- Root or sudo access to the system
- Basic familiarity with command line interface
- Text editor (we'll use nano, but vim or any editor works)

### Network Requirements

Your system should have:
- Internet connectivity for downloading Docker images
- Port 5678 available for n8n web interface
- Port 80 and 443 available if setting up reverse proxy
- Outbound HTTPS (443) access for external API calls

### Account Prerequisites

For the complete workflow functionality, you'll need accounts with:
- HubSpot (for CRM integration)
- Slack (for notifications)
- Clearbit (for lead enrichment) - optional but recommended

## Installing Docker on Ubuntu 24.04.01

Docker is the containerization platform that will run our n8n instance. Follow these steps to install Docker and Docker Compose on your Ubuntu system.

### Step 1: Update System Packages

First, update your package index and install prerequisites:

```bash
# Update package index
sudo apt update

# Install packages to allow apt to use a repository over HTTPS
sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    software-properties-common
```

### Step 2: Add Docker's Official GPG Key

```bash
# Create directory for keyrings
sudo mkdir -p /etc/apt/keyrings

# Download and add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set appropriate permissions
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Step 3: Add Docker Repository

```bash
# Add Docker repository to apt sources
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 4: Install Docker Engine

```bash
# Update package index with Docker packages
sudo apt update

# Install Docker Engine, CLI, and containerd
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify Docker installation
sudo docker --version
sudo docker compose version
```

### Step 5: Configure Docker for Non-Root User

```bash
# Add your user to the docker group
sudo usermod -aG docker $USER

# Apply group membership (logout/login or use newgrp)
newgrp docker

# Test Docker without sudo
docker run hello-world
```

If the hello-world container runs successfully, Docker is properly installed.

### Step 6: Enable Docker to Start on Boot

```bash
# Enable Docker service to start automatically
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# Verify services are running
sudo systemctl status docker
```

## Setting up n8n with Docker Compose

Now we'll create a complete Docker Compose setup for n8n with PostgreSQL database and Redis for optimal performance.

### Step 1: Create Project Directory Structure

```bash
# Create main project directory
mkdir -p ~/n8n-deployment
cd ~/n8n-deployment

# Create subdirectories for organization
mkdir -p {data,config,logs,backups}
mkdir -p data/{n8n,postgres,redis}

# Set proper permissions
chmod 755 data
chmod 755 data/{n8n,postgres,redis}
```

### Step 2: Create Docker Compose Configuration

Create the main Docker Compose file:

```bash
nano docker-compose.yml
```

Add the following comprehensive configuration:

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-n8n_secure_password_2024}
      POSTGRES_NON_ROOT_USER: n8n
      POSTGRES_NON_ROOT_PASSWORD: ${POSTGRES_PASSWORD:-n8n_secure_password_2024}
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
      - ./config/postgres-init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U n8n -d n8n"]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 5s
    networks:
      - n8n-network

  # Redis for Queue Management
  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-redis_secure_password_2024}
    volumes:
      - ./data/redis:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      retries: 3
      start_period: 30s
      timeout: 5s
    networks:
      - n8n-network

  # n8n Workflow Automation
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n-app
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      # Database Configuration
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD:-n8n_secure_password_2024}
      
      # Redis Configuration
      QUEUE_BULL_REDIS_HOST: redis
      QUEUE_BULL_REDIS_PORT: 6379
      QUEUE_BULL_REDIS_PASSWORD: ${REDIS_PASSWORD:-redis_secure_password_2024}
      
      # n8n Configuration
      N8N_HOST: ${N8N_HOST:-localhost}
      N8N_PORT: 5678
      N8N_PROTOCOL: ${N8N_PROTOCOL:-http}
      WEBHOOK_URL: ${WEBHOOK_URL:-http://localhost:5678}
      
      # Security
      N8N_BASIC_AUTH_ACTIVE: true
      N8N_BASIC_AUTH_USER: ${N8N_USER:-admin}
      N8N_BASIC_AUTH_PASSWORD: ${N8N_PASSWORD:-admin_secure_password_2024}
      
      # Execution Settings
      EXECUTIONS_PROCESS: main
      EXECUTIONS_DATA_SAVE_ON_ERROR: all
      EXECUTIONS_DATA_SAVE_ON_SUCCESS: all
      EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS: true
      EXECUTIONS_DATA_PRUNE: true
      EXECUTIONS_DATA_MAX_AGE: 336
      
      # Generic OAuth
      N8N_CONFIG_FILES: /home/node/.n8n/config
      
      # Timezone
      GENERIC_TIMEZONE: ${TIMEZONE:-UTC}
      TZ: ${TIMEZONE:-UTC}
      
      # Logging
      N8N_LOG_LEVEL: info
      N8N_LOG_OUTPUT: console,file
      N8N_LOG_FILE_LOCATION: /home/node/.n8n/logs/
      
      # Security Headers
      N8N_SECURE_COOKIE: false
      N8N_COOKIES_SECURE: false
      
    ports:
      - "${N8N_PORT:-5678}:5678"
    volumes:
      - ./data/n8n:/home/node/.n8n
      - ./config:/home/node/.n8n/config:ro
      - ./logs:/home/node/.n8n/logs
    networks:
      - n8n-network
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --tries=1 --spider http://localhost:5678/healthz || exit 1"]
      interval: 30s
      retries: 3
      start_period: 60s
      timeout: 10s

networks:
  n8n-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  postgres_data:
  redis_data:
  n8n_data:
```

### Step 3: Create Environment Variables File

Create a `.env` file for configuration:

```bash
nano .env
```

Add your environment-specific configurations:

```bash
# Database Configuration
POSTGRES_PASSWORD=your_very_secure_postgres_password_here
REDIS_PASSWORD=your_very_secure_redis_password_here

# n8n Configuration
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678
N8N_USER=admin
N8N_PASSWORD=your_very_secure_n8n_password_here

# Timezone (change to your timezone)
TIMEZONE=America/New_York

# Optional: Custom domain if using reverse proxy
# N8N_HOST=your-domain.com
# N8N_PROTOCOL=https
# WEBHOOK_URL=https://your-domain.com
```

### Step 4: Create PostgreSQL Initialization Script

Create a database initialization script:

```bash
nano config/postgres-init.sql
```

Add the following SQL:

```sql
-- Create additional database for lead storage
CREATE DATABASE leads_db;

-- Grant permissions
GRANT ALL PRIVILEGES ON DATABASE leads_db TO n8n;

-- Connect to leads database and create table
\c leads_db;

CREATE TABLE IF NOT EXISTS leads (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    company VARCHAR(255),
    phone VARCHAR(50),
    source VARCHAR(100),
    lead_score INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    enriched_data JSONB,
    status VARCHAR(50) DEFAULT 'new'
);

-- Create index for better performance
CREATE INDEX idx_leads_email ON leads(email);
CREATE INDEX idx_leads_score ON leads(lead_score);
CREATE INDEX idx_leads_created_at ON leads(created_at);

-- Grant permissions on the table
GRANT ALL PRIVILEGES ON TABLE leads TO n8n;
GRANT USAGE, SELECT ON SEQUENCE leads_id_seq TO n8n;
```

### Step 5: Set File Permissions

```bash
# Set appropriate permissions
chmod 600 .env
chmod 755 config/
chmod 644 config/postgres-init.sql
chmod 755 data/
```

## Configuring Environment Variables

### Step 1: Create n8n Configuration Directory

```bash
mkdir -p config/n8n
nano config/n8n/config.json
```

Add n8n-specific configuration:

```json
{
  "database": {
    "type": "postgresdb",
    "postgresdb": {
      "host": "postgres",
      "port": 5432,
      "database": "n8n",
      "user": "n8n",
      "password": "${POSTGRES_PASSWORD}"
    }
  },
  "credentials": {
    "overwrite": {
      "data": "keep"
    }
  },
  "nodes": {
    "exclude": [],
    "errorTriggerType": "n8n-nodes-base.errorTrigger"
  },
  "workflows": {
    "defaultName": "My Workflow"
  },
  "executions": {
    "saveDataOnError": "all",
    "saveDataOnSuccess": "all",
    "saveDataManualExecutions": true,
    "pruneData": true,
    "pruneDataMaxAge": 336
  }
}
```

### Step 2: Create Logging Configuration

```bash
mkdir -p logs
touch logs/n8n.log
chmod 644 logs/n8n.log
```

## Setting up External Services

### Step 1: Start the Services

```bash
# Start all services
docker compose up -d

# Check if all services are running
docker compose ps

# View logs to ensure everything started correctly
docker compose logs -f
```

You should see output indicating all services are healthy:

```
n8n-postgres    | PostgreSQL init process complete; ready for start up
n8n-redis       | Ready to accept connections
n8n-app         | n8n ready on 0.0.0.0, port 5678
```

### Step 2: Verify Database Connection

```bash
# Connect to PostgreSQL to verify setup
docker compose exec postgres psql -U n8n -d n8n

# Inside PostgreSQL shell, verify the leads database
\l
\c leads_db
\dt
\q
```

### Step 3: Verify Redis Connection

```bash
# Test Redis connection
docker compose exec redis redis-cli -a redis_secure_password_2024 ping
```

You should receive a "PONG" response.

### Step 4: Access n8n Web Interface

Open your web browser and navigate to:
```
http://localhost:5678
```

You'll be prompted for basic authentication:
- Username: admin
- Password: your_very_secure_n8n_password_here

## Importing and Configuring the Workflow

### Step 1: Access n8n Interface

Once logged in to n8n, you'll see the main workflow editor interface.

### Step 2: Import the Lead Processing Workflow

1. Click on "Workflows" in the left sidebar
2. Click the "+" button to create a new workflow
3. Click on the workflow settings (gear icon)
4. Select "Import from JSON"
5. Paste the complete JSON workflow from our earlier example

### Step 3: Save the Workflow

1. Click "Save" button
2. Name it "Advanced Lead Processing Pipeline"
3. Add tags: "lead-generation", "crm", "automation"

### Step 4: Configure Node Positions

The workflow should display with all nodes properly positioned. If any nodes appear disconnected:

1. Select each node and verify connections
2. Reconnect any loose connections by dragging from output to input ports
3. Save the workflow after making corrections

## Setting up Credentials and Integrations

### Step 1: Configure HubSpot Credentials

1. In n8n, go to "Credentials" in the left sidebar
2. Click "Create Credential"
3. Search for "HubSpot"
4. Click "HubSpot API"
5. Fill in the required fields:
   - **Access Token**: Your HubSpot private app access token
   - **Name**: "HubSpot API"

To obtain HubSpot credentials:
1. Log into your HubSpot account
2. Go to Settings → Integrations → Private Apps
3. Create a new private app
4. Grant required scopes: contacts (read/write)
5. Copy the access token

### Step 2: Configure Slack Credentials

1. Create new credential → "Slack API"
2. Fill in the fields:
   - **Access Token**: Your Slack Bot Token
   - **Name**: "Slack Bot Token"

To obtain Slack credentials:
1. Go to https://api.slack.com/apps
2. Create a new app
3. Go to "OAuth & Permissions"
4. Add Bot Token Scopes: chat:write, chat:write.public
5. Install app to workspace
6. Copy Bot User OAuth Token

### Step 3: Configure Clearbit Credentials (Optional)

1. Create new credential → "HTTP API Key"
2. Fill in the fields:
   - **API Key**: Your Clearbit secret key
   - **Name**: "Clearbit API"

### Step 4: Configure PostgreSQL Credentials

1. Create new credential → "Postgres"
2. Fill in the fields:
   - **Host**: postgres
   - **Database**: leads_db
   - **User**: n8n
   - **Password**: your_very_secure_postgres_password_here
   - **Port**: 5432
   - **Name**: "Lead Database"

### Step 5: Update Workflow Nodes with Credentials

Go back to your workflow and update each node that requires credentials:

1. **HubSpot nodes**: Select "HubSpot API" credential
2. **Slack node**: Select "Slack Bot Token" credential
3. **PostgreSQL node**: Select "Lead Database" credential
4. **Clearbit HTTP Request node**: Select "Clearbit API" credential

Save the workflow after updating all credentials.

## Testing the Workflow

### Step 1: Activate the Workflow

1. In the workflow editor, click the "Inactive" toggle to make it "Active"
2. The workflow should now be listening for webhook requests

### Step 2: Get the Webhook URL

1. Click on the "Lead Intake Webhook" node
2. Copy the webhook URL (should be something like: http://localhost:5678/webhook/lead-intake-webhook)

### Step 3: Test with Sample Data

Create a test script to send sample data:

```bash
nano test-webhook.sh
```

Add the following content:

```bash
#!/bin/bash

# Test webhook with sample lead data
curl -X POST http://localhost:5678/webhook/lead-intake \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john.doe@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "company": "Example Corp",
    "phone": "+1-555-123-4567",
    "source": "website"
  }'
```

Make it executable and run:

```bash
chmod +x test-webhook.sh
./test-webhook.sh
```

### Step 4: Monitor Execution

1. In n8n, go to "Executions" in the left sidebar
2. You should see your test execution
3. Click on it to view the execution details
4. Check each node's output to verify data flow

### Step 5: Verify Database Storage

```bash
# Check if lead was stored in database
docker compose exec postgres psql -U n8n -d leads_db -c "SELECT * FROM leads;"
```

### Step 6: Check Logs

```bash
# View n8n logs
docker compose logs n8n

# View all service logs
docker compose logs
```

## Security Hardening

### Step 1: Use Strong Passwords

Update your `.env` file with strong, unique passwords:

```bash
nano .env
```

Generate strong passwords:

```bash
# Generate random passwords
openssl rand -base64 32  # For PostgreSQL
openssl rand -base64 32  # For Redis
openssl rand -base64 32  # For n8n admin
```

### Step 2: Configure Firewall

```bash
# Install UFW if not already installed
sudo apt install ufw

# Configure firewall rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 5678/tcp  # n8n web interface
sudo ufw enable
```

### Step 3: Set up Reverse Proxy with SSL (Optional but Recommended)

Create an Nginx configuration for SSL termination:

```bash
nano config/nginx.conf
```

```nginx
server {
    listen 80;
    server_name your-domain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;

    ssl_certificate /etc/ssl/certs/your-domain.crt;
    ssl_certificate_key /etc/ssl/private/your-domain.key;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Step 4: Regular Security Updates

Create an update script:

```bash
nano update-containers.sh
```

```bash
#!/bin/bash

echo "Updating n8n containers..."
cd ~/n8n-deployment

# Pull latest images
docker compose pull

# Restart services with new images
docker compose up -d

# Clean up old images
docker image prune -f

echo "Update complete!"
```

Make it executable:

```bash
chmod +x update-containers.sh
```

## Maintenance and Updates

### Step 1: Create Backup Script

```bash
nano backup-n8n.sh
```

```bash
#!/bin/bash

BACKUP_DIR="$HOME/n8n-deployment/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="n8n_backup_$DATE.tar.gz"

echo "Creating backup: $BACKUP_FILE"

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Stop containers for consistent backup
cd ~/n8n-deployment
docker compose stop

# Create compressed backup
tar -czf "$BACKUP_DIR/$BACKUP_FILE" \
    data/ \
    config/ \
    docker-compose.yml \
    .env

# Restart containers
docker compose up -d

# Keep only last 7 backups
cd "$BACKUP_DIR"
ls -t n8n_backup_*.tar.gz | tail -n +8 | xargs -r rm

echo "Backup completed: $BACKUP_DIR/$BACKUP_FILE"
```

Make it executable:

```bash
chmod +x backup-n8n.sh
```

### Step 2: Set up Automated Backups

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /home/$USER/n8n-deployment/backup-n8n.sh >> /home/$USER/n8n-deployment/logs/backup.log 2>&1
```

### Step 3: Create Restore Script

```bash
nano restore-n8n.sh
```

```bash
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Usage: $0 <backup_file>"
    echo "Available backups:"
    ls -la backups/n8n_backup_*.tar.gz
    exit 1
fi

BACKUP_FILE="$1"

if [ ! -f "$BACKUP_FILE" ]; then
    echo "Backup file not found: $BACKUP_FILE"
    exit 1
fi

echo "Restoring from: $BACKUP_FILE"

# Stop containers
docker compose down

# Remove existing data (backup current state first)
mv data data_old_$(date +%Y%m%d_%H%M%S)
mv config config_old_$(date +%Y%m%d_%H%M%S)

# Extract backup
tar -xzf "$BACKUP_FILE"

# Start containers
docker compose up -d

echo "Restore completed!"
```

Make it executable:

```bash
chmod +x restore-n8n.sh
```

### Step 4: Monitor System Resources

Create a monitoring script:

```bash
nano monitor-n8n.sh
```

```bash
#!/bin/bash

echo "=== n8n System Status ==="
echo "Date: $(date)"
echo

echo "=== Container Status ==="
docker compose ps

echo
echo "=== Resource Usage ==="
docker stats --no-stream

echo
echo "=== Disk Usage ==="
du -sh data/*

echo
echo "=== Recent Logs (last 10 lines) ==="
docker compose logs --tail=10
```

Make it executable:

```bash
chmod +x monitor-n8n.sh
```

## Troubleshooting Common Issues

### Issue 1: Containers Won't Start

**Symptoms**: Services fail to start or keep restarting

**Solutions**:

```bash
# Check container logs
docker compose logs <service_name>

# Check port conflicts
sudo netstat -tulpn | grep :5678

# Verify environment variables
docker compose config

# Reset and rebuild
docker compose down
docker compose up --build -d
```

### Issue 2: Database Connection Errors

**Symptoms**: n8n can't connect to PostgreSQL

**Solutions**:

```bash
# Check PostgreSQL status
docker compose exec postgres pg_isready -U n8n

# Verify credentials
docker compose exec postgres psql -U n8n -d n8n

# Check network connectivity
docker compose exec n8n ping postgres

# Reset database
docker compose down
sudo rm -rf data/postgres/*
docker compose up -d
```

### Issue 3: Webhook Not Receiving Data

**Symptoms**: Webhook triggers don't execute

**Solutions**:

1. Verify workflow is active
2. Check webhook URL is correct
3. Test with curl:

```bash
curl -X POST http://localhost:5678/webhook/test \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

4. Check n8n logs for errors:

```bash
docker compose logs n8n | grep -i error
```

### Issue 4: High Memory Usage

**Symptoms**: System becomes slow or containers get killed

**Solutions**:

1. Limit container memory in docker-compose.yml:

```yaml
services:
  n8n:
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
```

2. Configure execution settings to reduce memory usage:

```bash
# In .env file
EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=24
```

### Issue 5: SSL/TLS Certificate Issues

**Symptoms**: HTTPS not working or certificate errors

**Solutions**:

1. Use Let's Encrypt for free SSL certificates:

```bash
# Install certbot
sudo apt install certbot

# Get certificate
sudo certbot certonly --standalone -d your-domain.com

# Update nginx configuration with certificate paths
```

2. Configure automatic renewal:

```bash
# Add to crontab
0 2 * * 0 certbot renew --quiet && docker compose restart nginx
```

## Advanced Configuration

### Step 1: Multi-Environment Setup

Create separate configurations for development, staging, and production:

```bash
# Development environment
cp .env .env.dev

# Staging environment  
cp .env .env.staging

# Production environment
cp .env .env.prod
```

Create environment-specific compose files:

```bash
# docker-compose.prod.yml
version: '3.8'

services:
  n8n:
    environment:
      N8N_LOG_LEVEL: warn
      EXECUTIONS_DATA_SAVE_ON_ERROR: none
      EXECUTIONS_DATA_SAVE_ON_SUCCESS: none
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '2'
```

Use with:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml --env-file .env.prod up -d
```

### Step 2: Load Balancing and High Availability

For production environments, set up multiple n8n instances:

```yaml
# docker-compose.ha.yml
version: '3.8'

services:
  n8n-worker-1:
    image: n8nio/n8n:latest
    environment:
      N8N_WORKER_ID: worker-1
      EXECUTIONS_PROCESS: queue
    depends_on:
      - postgres
      - redis

  n8n-worker-2:
    image: n8nio/n8n:latest
    environment:
      N8N_WORKER_ID: worker-2
      EXECUTIONS_PROCESS: queue
    depends_on:
      - postgres
      - redis

  nginx-lb:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx-lb.conf:/etc/nginx/nginx.conf
    depends_on:
      - n8n
      - n8n-worker-1
      - n8n-worker-2
```

### Step 3: Monitoring and Alerting

Set up monitoring with Prometheus and Grafana:

```yaml
# Add to docker-compose.yml
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - ./data/grafana:/var/lib/grafana
```

Create Prometheus configuration:

```bash
nano config/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'n8n'
    static_configs:
      - targets: ['n8n:5678']
    metrics_path: /metrics
```

### Step 4: Custom Node Development

If you need custom functionality, create custom nodes:

```bash
# Create custom nodes directory
mkdir -p custom-nodes

# Initialize npm project
cd custom-nodes
npm init -y

# Install n8n node development tools
npm install --save-dev @types/node n8n-workflow n8n-core

# Create custom node structure
mkdir -p nodes/CustomNode
```

## Conclusion

Congratulations! You have successfully deployed a complete n8n workflow automation system using Docker on Ubuntu 24.04.01. Your setup includes:

### What You've Accomplished

1. **Full Docker Environment**: A robust containerized setup with n8n, PostgreSQL, and Redis
2. **Advanced Workflow**: Imported and configured a complex lead processing pipeline
3. **Security Implementation**: Basic authentication, firewall configuration, and secure credentials management
4. **Database Integration**: PostgreSQL setup for persistent data storage
5. **Monitoring and Maintenance**: Backup scripts, update procedures, and troubleshooting guides
6. **Production Readiness**: SSL configuration, resource limits, and high availability options

### Key Benefits of Your Setup

- **Scalability**: Easy to scale up with additional worker containers
- **Reliability**: Persistent data storage and automatic container restarts
- **Security**: Isolated containers with secure credential management
- **Maintainability**: Automated backups and straightforward update procedures
- **Flexibility**: Easy to modify and extend workflows

### Next Steps for Enhancement

1. **Custom Integrations**: Develop custom nodes for specific business needs
2. **Advanced Monitoring**: Implement comprehensive monitoring with Grafana dashboards
3. **Multi-Environment**: Set up development, staging, and production environments
4. **Performance Optimization**: Fine-tune database queries and container resources
5. **Workflow Library**: Build a collection of reusable workflow templates

### Best Practices to Remember

- **Regular Backups**: Run the backup script daily and test restore procedures
- **Security Updates**: Keep containers updated with the latest security patches
- **Resource Monitoring**: Monitor CPU, memory, and disk usage regularly
- **Log Management**: Implement log rotation to prevent disk space issues
- **Documentation**: Maintain documentation of custom workflows and configurations

### Community Resources

- **n8n Community Forum**: Get help and share workflows at https://community.n8n.io/
- **GitHub Repository**: Contribute to the project at https://github.com/n8n-io/n8n
- **Documentation**: Official docs at https://docs.n8n.io/
- **YouTube Channel**: Video tutorials at https://www.youtube.com/c/n8nio

### Support and Troubleshooting

If you encounter issues:

1. Check the troubleshooting section in this guide
2. Review container logs using `docker compose logs`
3. Consult the n8n community forum
4. Check GitHub issues for similar problems
5. Use the monitoring script to identify resource bottlenecks

Your n8n installation is now ready to handle complex business automation workflows. The lead processing pipeline you've deployed demonstrates advanced features like data enrichment, conditional routing, database integration, and external API calls. You can use this as a foundation to build additional workflows for your specific business needs.

Remember to regularly maintain your installation, keep backups current, and stay updated with the latest n8n releases for new features and security improvements. Happy automating!
