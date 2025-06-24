# Complete Guide: Deploying n8n with Docker on Ubuntu 24.04.01 for AI-Powered Email Automation

## Table of Contents
1. [Introduction and Overview](#introduction)
2. [Prerequisites and System Requirements](#prerequisites)
3. [Ubuntu System Preparation](#ubuntu-preparation)
4. [Docker Installation and Configuration](#docker-installation)
5. [n8n Docker Container Setup](#n8n-setup)
6. [Initial n8n Configuration](#initial-configuration)
7. [Workflow Import and Setup](#workflow-import)
8. [Credential Configuration](#credential-configuration)
9. [AI Services Integration](#ai-integration)
10. [Email Service Configuration](#email-configuration)
11. [Testing and Validation](#testing)
12. [Security and Access Control](#security)
13. [Backup and Data Management](#backup)
14. [Monitoring and Maintenance](#monitoring)
15. [Troubleshooting Guide](#troubleshooting)
16. [Advanced Configuration](#advanced-configuration)

## Introduction and Overview {#introduction}

This comprehensive guide will walk you through the complete process of setting up n8n (pronounced "n-eight-n") in a Docker container on Ubuntu 24.04.01, specifically configured to run the AI-powered Outlook Inbox Manager workflow. By the end of this guide, you'll have a fully functional automation system that can intelligently process emails using artificial intelligence.

### What You'll Accomplish

By following this guide, you will:
- Install and configure Docker on Ubuntu 24.04.01
- Deploy n8n in a secure Docker container
- Import and configure the AI-powered email management workflow
- Set up credentials for Microsoft Outlook, OpenAI, Google Gemini, and Telegram
- Configure persistent data storage and backups
- Implement security best practices
- Create a production-ready automation system

### Why Docker?

Docker provides several advantages for n8n deployment:
- **Isolation**: Keeps n8n and its dependencies separate from your host system
- **Portability**: Easy to move between different servers
- **Scalability**: Simple to scale up or replicate
- **Security**: Contained environment with controlled access
- **Maintenance**: Easy updates and rollbacks
- **Consistency**: Same environment across development and production

### Architecture Overview

```
Internet → Ubuntu Host → Docker Container → n8n Application
                     ↓
              Persistent Volumes (Data Storage)
                     ↓
            External Services (Outlook, OpenAI, etc.)
```

## Prerequisites and System Requirements {#prerequisites}

### Hardware Requirements

**Minimum Requirements:**
- CPU: 2 cores (2.0 GHz or higher)
- RAM: 4 GB (8 GB recommended for AI workloads)
- Storage: 20 GB free space (SSD recommended)
- Network: Stable internet connection with at least 10 Mbps

**Recommended Requirements:**
- CPU: 4 cores (2.5 GHz or higher)
- RAM: 8 GB or more
- Storage: 50 GB SSD
- Network: High-speed internet connection

### Software Requirements

- Ubuntu 24.04.01 LTS (Server or Desktop)
- Root or sudo access
- Basic command line familiarity
- Web browser for n8n interface access

### Service Account Requirements

You'll need accounts for the following services:
- **Microsoft 365/Outlook**: For email processing
- **OpenAI**: For AI-powered text processing (API key required)
- **Google Cloud Platform**: For Gemini AI model access
- **Telegram**: For notifications (bot token required)

### Network Requirements

- Port 5678: n8n web interface (will be mapped to host)
- Outbound HTTPS (443): For API calls to external services
- Outbound HTTP (80): For webhook callbacks (if needed)

## Ubuntu System Preparation {#ubuntu-preparation}

### Step 1: System Update and Upgrade

First, ensure your Ubuntu system is up to date:

```bash
# Update package lists
sudo apt update

# Upgrade installed packages
sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl wget git nano htop unzip software-properties-common apt-transport-https ca-certificates gnupg lsb-release
```

### Step 2: Create Dedicated User (Recommended)

For security purposes, create a dedicated user for running n8n:

```bash
# Create n8n user
sudo useradd -m -s /bin/bash n8nuser

# Add user to sudo group (if needed for maintenance)
sudo usermod -aG sudo n8nuser

# Set password for the user
sudo passwd n8nuser

# Switch to the new user
su - n8nuser
```

### Step 3: Configure Firewall

Set up UFW (Uncomplicated Firewall) for basic security:

```bash
# Enable UFW
sudo ufw enable

# Allow SSH (adjust port if you use non-standard SSH port)
sudo ufw allow ssh

# Allow n8n web interface port
sudo ufw allow 5678

# Check firewall status
sudo ufw status
```

### Step 4: Create Directory Structure

Create directories for n8n data and configuration:

```bash
# Create n8n directories
mkdir -p ~/n8n-docker/{data,config,backups,logs}

# Create environment file
touch ~/n8n-docker/.env

# Set proper permissions
chmod 700 ~/n8n-docker
chmod 600 ~/n8n-docker/.env
```

### Step 5: System Resource Configuration

Optimize system resources for Docker and n8n:

```bash
# Increase file descriptor limits
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# Configure swap (if not already configured)
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

## Docker Installation and Configuration {#docker-installation}

### Step 1: Remove Old Docker Versions

Remove any existing Docker installations:

```bash
# Remove old Docker packages
sudo apt remove -y docker docker-engine docker.io containerd runc
```

### Step 2: Add Docker Repository

Add the official Docker repository:

```bash
# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package lists
sudo apt update
```

### Step 3: Install Docker

Install Docker Engine and related components:

```bash
# Install Docker Engine
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Verify Docker installation
sudo docker --version
sudo docker compose version
```

### Step 4: Configure Docker for Non-Root User

Add your user to the Docker group:

```bash
# Add user to docker group
sudo usermod -aG docker $USER

# Apply group changes (logout and login, or use newgrp)
newgrp docker

# Test Docker without sudo
docker run hello-world
```

### Step 5: Configure Docker Daemon

Create Docker daemon configuration for optimal performance:

```bash
# Create Docker daemon configuration
sudo mkdir -p /etc/docker

# Create daemon.json configuration
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "dns": ["8.8.8.8", "8.8.4.4"],
  "default-address-pools": [
    {
      "base": "172.17.0.0/16",
      "size": 24
    }
  ]
}
EOF

# Restart Docker service
sudo systemctl restart docker

# Enable Docker to start on boot
sudo systemctl enable docker
```

### Step 6: Verify Docker Installation

Confirm Docker is working correctly:

```bash
# Check Docker service status
sudo systemctl status docker

# Run Docker info command
docker info

# Test Docker functionality
docker run --rm alpine echo "Docker is working!"
```

## n8n Docker Container Setup {#n8n-setup}

### Step 1: Create Docker Compose Configuration

Navigate to your n8n directory and create the Docker Compose file:

```bash
cd ~/n8n-docker

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=${WEBHOOK_URL}
      - GENERIC_TIMEZONE=${TIMEZONE}
      - N8N_METRICS=true
      - N8N_LOG_LEVEL=info
      - N8N_LOG_OUTPUT=console,file
      - N8N_LOG_FILE_LOCATION=/home/node/.n8n/logs/
      - DB_TYPE=sqlite
      - DB_SQLITE_DATABASE=/home/node/.n8n/database.sqlite
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_USER_FOLDER=/home/node/.n8n
      - N8N_DISABLE_UI=false
      - N8N_PERSONALIZATION_ENABLED=false
      - EXECUTIONS_PROCESS=main
      - EXECUTIONS_DATA_SAVE_ON_ERROR=all  
      - EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
      - EXECUTIONS_DATA_MAX_AGE=168
    volumes:
      - ./data:/home/node/.n8n
      - ./logs:/home/node/.n8n/logs
    networks:
      - n8n-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  n8n-network:
    driver: bridge

volumes:
  n8n-data:
    driver: local
EOF
```

### Step 2: Create Environment Configuration

Create the environment variables file:

```bash
# Generate a secure encryption key
ENCRYPTION_KEY=$(openssl rand -hex 32)

# Create .env file with configuration
cat > .env << EOF
# Basic Authentication
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=$(openssl rand -base64 32 | tr -d "=+/" | cut -c1-20)

# Host Configuration
N8N_HOST=localhost
WEBHOOK_URL=http://localhost:5678/

# Timezone
TIMEZONE=America/New_York

# Security
N8N_ENCRYPTION_KEY=${ENCRYPTION_KEY}

# Performance
N8N_DEFAULT_BINARY_DATA_MODE=filesystem
N8N_BINARY_DATA_TTL=24

# Features
N8N_DISABLE_PRODUCTION_MAIN_PROCESS=false
N8N_SKIP_WEBHOOK_DEREGISTRATION_SHUTDOWN=true
EOF

# Display generated credentials
echo "Generated n8n credentials:"
echo "Username: admin"
echo "Password: $(grep N8N_BASIC_AUTH_PASSWORD .env | cut -d'=' -f2)"
echo ""
echo "Please save these credentials securely!"
```

### Step 3: Set Directory Permissions

Ensure proper permissions for Docker volumes:

```bash
# Set ownership for n8n data directory
sudo chown -R 1000:1000 ./data ./logs

# Set proper permissions
chmod -R 755 ./data
chmod -R 755 ./logs
```

### Step 4: Deploy n8n Container

Start the n8n container:

```bash
# Pull the latest n8n image
docker compose pull

# Start n8n in detached mode
docker compose up -d

# Check container status
docker compose ps

# View container logs
docker compose logs -f n8n
```

### Step 5: Verify n8n Deployment

Check if n8n is running correctly:

```bash
# Check if n8n is responding
curl -I http://localhost:5678

# Check container health
docker compose exec n8n ps aux

# View detailed logs
docker compose logs --tail 50 n8n
```

## Initial n8n Configuration {#initial-configuration}

### Step 1: Access n8n Web Interface

1. Open your web browser and navigate to: `http://your-server-ip:5678`
2. You'll see a basic authentication prompt
3. Enter the credentials from your `.env` file:
   - Username: `admin`
   - Password: (the generated password from Step 2 above)

### Step 2: Initial Setup Wizard

Complete the n8n setup wizard:

1. **Welcome Screen**: Click "Let's get started"
2. **Usage Statistics**: Choose your preference (recommended: opt out for privacy)
3. **Setup Complete**: You should now see the n8n dashboard

### Step 3: Configure Basic Settings

Navigate to Settings (gear icon) and configure:

**General Settings:**
- Timezone: Set to your local timezone
- Date Format: Choose your preferred format
- First Day of Week: Set according to your preference

**Security Settings:**
- Enable two-factor authentication (if available)
- Review password policy settings

**Execution Settings:**
- Execution Timeout: 3600 seconds (1 hour)
- Max Execution Time: 3600 seconds
- Save Data on Success: Yes
- Save Data on Error: Yes

### Step 4: Create Initial Workflow

Create a simple test workflow to verify functionality:

1. Click "Add workflow" or the "+" button
2. Add a "Manual Trigger" node
3. Add a "Set" node and connect it to the trigger
4. Configure the Set node to add some test data
5. Save the workflow as "Test Workflow"
6. Execute the workflow to verify it works

## Workflow Import and Setup {#workflow-import}

### Step 1: Prepare Workflow JSON

Create the workflow JSON file on your server:

```bash
# Create workflows directory
mkdir -p ~/n8n-docker/workflows

# Create the Outlook Inbox Manager workflow file
cat > ~/n8n-docker/workflows/outlook-inbox-manager.json << 'EOF'
{
  "name": "Outlook Inbox Manager",
  "nodes": [
    {
      "parameters": {
        "pollTimes": {
          "item": [
            {
              "mode": "everyMinute"
            }
          ]
        },
        "output": "raw",
        "filters": {},
        "options": {}
      },
      "type": "n8n-nodes-base.microsoftOutlookTrigger",
      "typeVersion": 1,
      "position": [
        -60,
        -100
      ],
      "id": "b2d39e20-53a6-478d-b64f-61867594aa99",
      "name": "Microsoft Outlook Trigger",
      "credentials": {
        "microsoftOutlookOAuth2Api": {
          "id": "outlook-credentials",
          "name": "Outlook Credentials"
        }
      }
    },
    {
      "parameters": {
        "modelId": {
          "__rl": true,
          "value": "gpt-4o-mini",
          "mode": "list",
          "cachedResultName": "GPT-4O-MINI"
        },
        "messages": {
          "values": [
            {
              "content": "=Here is an incoming email:  {{ $json.body.content }}"
            },
            {
              "content": "Take the incoming email and clean it up so it is more readable. Get rid of the HTML tagging, but don't get rid of any of the email content. Don't include a subject.\n",
              "role": "system"
            }
          ]
        },
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.openAi",
      "typeVersion": 1.8,
      "position": [
        120,
        -100
      ],
      "id": "1cd1a5ea-ec93-4da0-a4a1-ffee379410cd",
      "name": "Clean Email",
      "credentials": {
        "openAiApi": {
          "id": "openai-credentials",
          "name": "OpenAI Credentials"
        }
      }
    },
    {
      "parameters": {
        "inputText": "={{ $json.message.content }}",
        "categories": {
          "categories": [
            {
              "category": "High Priority",
              "description": "Emails that require immediate attention, often involving urgent issues, escalations, system failures, or critical business matters.\n\nCommon Words/Phrases:\nUrgent, ASAP, Immediate action required, Critical issue, Escalation, Emergency, System outage, High priority, Downtime, Affected customers, Please respond quickly, Need resolution, Time-sensitive."
            },
            {
              "category": "Billing",
              "description": "Emails related to payments, invoices, subscriptions, financial transactions, or account balances. These emails often contain due dates, payment instructions, or financial statements.\n\nCommon Words/Phrases:\nInvoice, Billing statement, Payment due, Past due, Outstanding balance, Subscription renewal, Payment confirmation, Charge notice, Overdue notice, Auto-renewal, Finance department, ACH transfer, Bank details."
            },
            {
              "category": "Promotion",
              "description": "Emails related to marketing campaigns, sales offers, discounts, partnership opportunities, or advertisements. These emails are often sent in bulk and contain promotional language.\n\nCommon Words/Phrases:\nSpecial offer, Limited-time deal, Exclusive discount, Save big, Flash sale, Promo code, Get 20% off, Earn rewards, Best deals, New product launch, Marketing campaign, Subscription offer, Early access, Upgrade now, Act fast."
            }
          ]
        },
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.textClassifier",
      "typeVersion": 1,
      "position": [
        480,
        -100
      ],
      "id": "9ab40f44-fc29-4197-9ba4-ba95ee96ba23",
      "name": "Text Classifier"
    }
  ],
  "connections": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  }
}
EOF
```

### Step 2: Import Workflow via Web Interface

1. **Access n8n Interface**: Open http://your-server-ip:5678
2. **Create New Workflow**: Click "Add workflow"
3. **Import Method**: Click the three dots menu (⋯) and select "Import from file"
4. **Upload JSON**: Select the workflow JSON file you created
5. **Verify Import**: Check that all nodes are imported correctly

### Step 3: Alternative CLI Import Method

You can also import workflows using the n8n CLI within the container:

```bash
# Copy workflow file to container
docker compose cp ~/n8n-docker/workflows/outlook-inbox-manager.json n8n:/tmp/

# Execute import command in container
docker compose exec n8n n8n import:workflow --input=/tmp/outlook-inbox-manager.json

# Verify import
docker compose exec n8n n8n list:workflow
```

### Step 4: Workflow Validation

After importing, validate the workflow:

1. **Check Node Connections**: Ensure all nodes are properly connected
2. **Verify Node Configuration**: Check that each node has the correct parameters
3. **Review Credentials**: Note which credentials need to be configured
4. **Test Node Compatibility**: Ensure all node types are available in your n8n version

## Credential Configuration {#credential-configuration}

### Step 1: Microsoft Outlook OAuth2 Setup

**Create Azure App Registration:**

1. **Access Azure Portal**: Go to https://portal.azure.com
2. **Navigate to App Registrations**: Azure Active Directory → App registrations
3. **Create New Registration**:
   - Name: "n8n Outlook Integration"
   - Supported account types: "Accounts in this organizational directory only"
   - Redirect URI: `http://your-server-ip:5678/rest/oauth2-credential/callback`

4. **Configure API Permissions**:
   - Microsoft Graph → Delegated permissions
   - Add: `Mail.Read`, `Mail.ReadWrite`, `Mail.Send`
   - Grant admin consent

5. **Create Client Secret**:
   - Go to "Certificates & secrets"
   - Create new client secret
   - Copy the secret value (you won't see it again)

**Configure in n8n:**

1. **Access Credentials**: In n8n, go to Settings → Credentials
2. **Add New Credential**: Click "Add Credential"
3. **Select Type**: Choose "Microsoft Outlook OAuth2 API"
4. **Configure Settings**:
   - Client ID: (from Azure app registration)
   - Client Secret: (from Azure app registration)
   - Scope: `https://graph.microsoft.com/Mail.Read https://graph.microsoft.com/Mail.ReadWrite https://graph.microsoft.com/Mail.Send offline_access`

5. **Test Connection**: Click "Connect my account" and authorize
6. **Save Credential**: Name it "Outlook Credentials"

### Step 2: OpenAI API Configuration

**Obtain OpenAI API Key:**

1. **Access OpenAI Platform**: Go to https://platform.openai.com
2. **Navigate to API Keys**: Go to API → API Keys
3. **Create New Key**: Click "Create new secret key"
4. **Copy Key**: Store the key securely (you won't see it again)

**Configure in n8n:**

1. **Add New Credential**: In n8n Credentials, click "Add Credential"
2. **Select Type**: Choose "OpenAI API"
3. **Configure Settings**:
   - API Key: (paste your OpenAI API key)
   - Organization ID: (optional, if you have one)
4. **Test Connection**: Verify the connection works
5. **Save Credential**: Name it "OpenAI Credentials"

### Step 3: Google Gemini API Configuration

**Setup Google Cloud Project:**

1. **Access Google Cloud Console**: Go to https://console.cloud.google.com
2. **Create Project**: Create a new project or select existing
3. **Enable APIs**: Enable the "Generative Language API"
4. **Create API Key**:
   - Go to APIs & Credentials → Credentials
   - Create credentials → API Key
   - Copy the API key

**Configure in n8n:**

1. **Add New Credential**: Choose "Google PaLM API"
2. **Configure Settings**:
   - API Key: (paste your Google API key)
3. **Test Connection**: Verify it works
4. **Save Credential**: Name it "Google Gemini Credentials"

### Step 4: Telegram Bot Configuration

**Create Telegram Bot:**

1. **Message BotFather**: Open Telegram and message @BotFather
2. **Create Bot**: Send `/newbot` command
3. **Set Bot Name**: Follow prompts to name your bot
4. **Copy Token**: Save the bot token provided by BotFather
5. **Get Chat ID**: 
   - Add your bot to a chat or message it directly
   - Visit: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
   - Find your chat ID in the response

**Configure in n8n:**

1. **Add New Credential**: Choose "Telegram API"
2. **Configure Settings**:
   - Access Token: (your bot token)
   - Chat ID: (your chat/user ID)
3. **Test Connection**: Send a test message
4. **Save Credential**: Name it "Telegram Credentials"

### Step 5: Credential Security Best Practices

**Backup Credentials:**
```bash
# Create credentials backup
docker compose exec n8n n8n export:credentials --output=/tmp/credentials-backup.json --decrypt

# Copy backup to host
docker compose cp n8n:/tmp/credentials-backup.json ~/n8n-docker/backups/

# Secure the backup file
chmod 600 ~/n8n-docker/backups/credentials-backup.json
```

**Rotate Credentials Regularly:**
- Set up calendar reminders to rotate API keys monthly
- Monitor credential usage in respective platforms
- Update credentials in n8n when rotated

## AI Services Integration {#ai-integration}

### Step 1: Configure OpenAI Integration

**Model Selection and Optimization:**

1. **Access Workflow**: Open the Outlook Inbox Manager workflow
2. **Configure Clean Email Node**:
   - Model: `gpt-4o-mini` (cost-effective for text cleaning)
   - Temperature: 0.1 (for consistent results)
   - Max Tokens: 1000 (sufficient for email cleaning)

3. **Optimize Prompts**:
```json
{
  "system": "You are an email processing assistant. Clean up HTML emails to make them readable while preserving all content and context.",
  "user": "Clean this email content: {{ $json.body.content }}"
}
```

**Token Usage Monitoring:**
```bash
# Create monitoring script
cat > ~/n8n-docker/scripts/monitor-openai-usage.sh << 'EOF'
#!/bin/bash
# Monitor OpenAI API usage
curl -H "Authorization: Bearer $OPENAI_API_KEY" \
     "https://api.openai.com/v1/usage?date=$(date +%Y-%m-%d)" | jq .
EOF

chmod +x ~/n8n-docker/scripts/monitor-openai-usage.sh
```

### Step 2: Configure Google Gemini Integration

**Model Configuration:**

1. **Access Text Classifier Node**
2. **Configure Gemini Model**:
   - Model: `models/gemini-2.0-flash`
   - Temperature: 0.3 (for consistent classification)
   - Safety Settings: Configure appropriate content filtering

3. **Optimize Classification Categories**:
   - Ensure category descriptions are detailed and specific
   - Include relevant keywords and phrases
   - Test classification accuracy with sample emails

### Step 3: AI Agent Configuration

**Billing Agent Setup:**

1. **Configure System Message**:
```markdown
# Billing Assistant Instructions

You are a professional billing assistant for [Company Name]. Your role is to:

1. **Acknowledge** customer inquiries promptly and professionally
2. **Analyze** billing-related questions and concerns
3. **Provide** accurate information about charges, payments, and policies
4. **Create** draft responses for complex issues requiring human review
5. **Escalate** urgent matters appropriately

## Response Guidelines:
- Use professional, empathetic tone
- Include relevant account information when available
- Provide clear next steps
- Reference company policies when applicable
- Sign all emails as "[Your Name], Billing Department"

## Tools Available:
- Create Draft: Use for responses requiring human review
- Email Send: Use for standard automated responses
```

2. **Configure Tool Integration**:
   - Link "Create Draft" tool to Outlook
   - Set appropriate permissions and scopes

**Promotion Agent Setup:**

1. **Configure Decline Templates**:
```markdown
# Promotional Email Decline Assistant

You politely decline all promotional offers while maintaining professional relationships.

## Standard Response Template:
"Thank you for thinking of us with this opportunity. At this time, we're not pursuing new [promotional offers/partnerships/etc.]. 

Please remove us from future promotional communications.

Best regards,
[Name]"

## Guidelines:
- Always be polite and professional
- Keep responses brief
- Request removal from future communications
- Use consistent messaging
```

### Step 4: Performance Optimization

**Resource Management:**
```bash
# Monitor AI API usage
cat > ~/n8n-docker/scripts/monitor-ai-performance.sh << 'EOF'
#!/bin/bash
echo "=== n8n Container Stats ==="
docker stats n8n --no-stream

echo "=== Workflow Execution Stats ==="
docker compose exec n8n n8n list:execution --limit=10

echo "=== Memory Usage ==="
docker compose exec n8n cat /proc/meminfo | grep Mem
EOF

chmod +x ~/n8n-docker/scripts/monitor-ai-performance.sh
```

## Email Service Configuration {#email-configuration}

### Step 1: Microsoft Outlook Advanced Configuration

**Folder Management:**

1. **Create Email Folders**:
   - High Priority
   - Billing Inquiries  
   - Promotions
   - Processed

2. **Configure Folder IDs**:
```bash
# Get Outlook folder information
docker compose exec n8n node -e "
const { MSGraphApi } = require('n8n-nodes-base/dist/nodes/Microsoft/Outlook/GenericFunctions');
// Script to list folders and their IDs
"
```

3. **Update Workflow Nodes**:
   - Configure folder destinations in move operations
   - Set appropriate folder IDs for each category

**Email Processing Rules:**

1. **Configure Polling Frequency**:
```json
{
  "pollTimes": {
    "item": [
      {
        "mode": "everyMinute"  // Adjust based on email volume
      }
    ]
  }
}
```

2. **Set Email Filters**:
```json
{
  "filters": {
    "includeAttachments": false,
    "folder": "Inbox",
    "receivedAfter": "{{ $now.minus({hours: 24}).toISO() }}"
  }
}
```

### Step 2: Email Template Configuration

**Create Response Templates:**

```bash
# Create email templates directory
mkdir -p ~/n8n-docker/templates

# High priority notification template
cat > ~/n8n-docker/templates/high-priority-notification.md << 'EOF'
# High Priority Email Alert

**From:** {{ sender.name }} ({{ sender.email }})
**Subject:** {{ subject }}
**Received:** {{ received_date }}
**Priority Score:** {{ priority_score }}

**Preview:**
{{ email_preview }}

**Action Required:** Manual review and response needed.
EOF

# Billing response template
cat > ~/n8n-docker/templates/billing-response.md << 'EOF'
Subject: Re: {{ original_subject }}

Dear {{ customer_name }},

Thank you for contacting our billing department. We have received your inquiry regarding {{ inquiry_type }}.

{{ ai_generated_response }}

If you have any additional questions, please don't hesitate to contact us.

Best regards,
{{ agent_name }}
Billing Department
{{ company_name }}
EOF
```

### Step 3: Notification Configuration

**Telegram Notifications:**

1. **Configure Message Templates**:
```javascript
// High Priority Alert
const highPriorityMessage = `
🚨 HIGH PRIORITY EMAIL

From: ${sender.name}
Subject: ${subject}
Time: ${formatDate(new Date())}

Action: Moved to High Priority folder
Status: Requires immediate attention
`;

// Billing Notification
const billingMessage = `
💰 BILLING INQUIRY

From: ${sender.name}
Time: ${formatDate(new Date())}
Status: Draft response created
Action: Review draft in Outlook
`;
```

2. **Configure Notification Scheduling**:
   - Business hours only for non-urgent items
   - Immediate for high-priority emails
   - Digest mode for promotional emails

### Step 4: Error Handling and Logging

**Email Processing Error Handling:**

```json
{
  "errorHandling": {
    "retryAttempts": 3,
    "retryDelay": 30000,
    "fallbackAction": "move_to_error_folder",
    "notificationOnError": true
  }
}
```

**Logging Configuration:**

```bash
# Create logging script
cat > ~/n8n-docker/scripts/email-processing-log.sh << 'EOF'
#!/bin/bash
# Email processing statistics

echo "=== Email Processing Stats (Last 24h) ==="
docker compose logs n8n --since 24h | grep -E "(email|outlook|processed)" | wc -l

echo "=== Error Count ==="
docker compose logs n8n --since 24h | grep -i error | wc -l

echo "=== Recent Activity ==="
docker compose logs n8n --tail 20 | grep -E "(Outlook|Email)"
EOF

chmod +x ~/n8n-docker/scripts/email-processing-log.sh
```

## Testing and Validation {#testing}

### Step 1: Component Testing

**Test Email Connectivity:**

```bash
# Test Outlook connection
docker compose exec n8n node -e "
console.log('Testing Outlook connectivity...');
// Test script to verify Outlook API access
"
```

**Test AI Services:**

1. **OpenAI Test**:
   - Create simple workflow with OpenAI node
   - Send test prompt
   - Verify response quality and speed

2. **Gemini Test**:
   - Test text classification with sample emails
   - Verify category accuracy
   - Check response time

**Test Telegram Notifications:**

```bash
# Send test notification
curl -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
     -H "Content-Type: application/json" \
     -d '{
       "chat_id": "'${CHAT_ID}'",
       "text": "n8n Email Automation Test - System is operational"
     }'
```

### Step 2: Integration Testing

**End-to-End Workflow Test:**

1. **Prepare Test Email**:
   - Send test email to monitored account
   - Include keywords for each category (High Priority, Billing, Promotion)

2. **Monitor Workflow Execution**:
```bash
# Watch workflow execution in real-time
docker compose logs -f n8n | grep -E "(execution|workflow|email)"
```

3. **Verify Results**:
   - Check email folder movement
   - Verify AI classification accuracy
   - Confirm notifications received
   - Review generated responses

### Step 3: Load Testing

**Email Volume Testing:**

```bash
# Create load test script
cat > ~/n8n-docker/scripts/load-test.sh << 'EOF'
#!/bin/bash
# Send multiple test emails to simulate load

for i in {1..10}; do
  echo "Sending test email $i"
  # Use your preferred email sending method
  sleep 30  # Wait 30 seconds between emails
done
EOF

chmod +x ~/n8n-docker/scripts/load-test.sh
```

**Performance Monitoring:**

```bash
# Monitor system resources during testing
watch -n 5 'docker stats n8n --no-stream'
```

### Step 4: Validation Checklist

**Pre-Production Checklist:**

- [ ] All credentials configured and tested
- [ ] Workflow imports successfully
- [ ] Email connectivity verified
- [ ] AI services responding correctly
- [ ] Notifications working
- [ ] Folder organization functional
- [ ] Error handling tested
- [ ] Backup procedures verified
- [ ] Security settings configured
- [ ] Performance benchmarks established

**Test Documentation:**

```bash
# Create test results documentation
cat > ~/n8n-docker/test-results.md << 'EOF'
# n8n Email Automation Test Results

## Test Date: $(date)

### Component Tests
- [ ] Outlook Connection: PASS/FAIL
- [ ] OpenAI Integration: PASS/FAIL
- [ ] Gemini Classification: PASS/FAIL
- [ ] Telegram Notifications: PASS/FAIL

### Integration Tests
- [ ] End-to-End Email Processing: PASS/FAIL
- [ ] Category Classification Accuracy: X%
- [ ] Response Generation Quality: PASS/FAIL
- [ ] Notification Delivery: PASS/FAIL

### Performance Tests
- [ ] Average Processing Time: X seconds
- [ ] Memory Usage: X MB
- [ ] CPU Usage: X%
- [ ] Error Rate: X%

### Notes
Add any observations or issues discovered during testing.
EOF
```

## Security and Access Control {#security}

### Step 1: Network Security

**Firewall Configuration:**

```bash
# Configure UFW for n8n
sudo ufw allow from trusted.ip.address to any port 5678
sudo ufw deny 5678  # Block all other access to n8n port

# Allow only necessary outbound connections
sudo ufw allow out 443  # HTTPS for API calls
sudo ufw allow out 80   # HTTP if needed
sudo ufw allow out 587  # SMTP if using email
```

**Reverse Proxy Setup (Recommended):**

```bash
# Install nginx
sudo apt install -y nginx

# Create nginx configuration for n8n
sudo tee /etc/nginx/sites-available/n8n << 'EOF'
server {
    listen 80;
    server_name your-domain.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-domain.com;
    
    # SSL Configuration (use Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;
    
    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    
    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
EOF

# Enable site
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Step 2: SSL/TLS Configuration

**Install Let's Encrypt:**

```bash
# Install Certbot
sudo apt install -y snapd
sudo snap install --classic certbot

# Create symbolic link
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# Obtain SSL certificate
sudo certbot --nginx -d your-domain.com

# Set up automatic renewal
sudo crontab -e
# Add: 0 12 * * * /usr/bin/certbot renew --quiet
```

### Step 3: Authentication and Authorization

**Enhanced Authentication:**

```bash
# Update docker-compose.yml with stronger authentication
cat >> ~/n8n-docker/docker-compose.yml << 'EOF'
    environment:
      # ... existing environment variables ...
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_JWT_AUTH_ACTIVE=true
      - N8N_JWT_AUTH_HEADER=authorization
      - N8N_SECURE_COOKIE=true
      - N8N_SESSION_TIMEOUT=3600
EOF
```

**User Management:**

```bash
# Create user management script
cat > ~/n8n-docker/scripts/manage-users.sh << 'EOF'
#!/bin/bash
# n8n user management

case $1 in
  "create")
    docker compose exec n8n n8n user:create --email="$2" --firstName="$3" --lastName="$4" --password="$5"
    ;;
  "list")
    docker compose exec n8n n8n user:list
    ;;
  "delete")
    docker compose exec n8n n8n user:delete --email="$2"
    ;;
  *)
    echo "Usage: $0 {create|list|delete} [email] [firstName] [lastName] [password]"
    ;;
esac
EOF

chmod +x ~/n8n-docker/scripts/manage-users.sh
```

### Step 4: Data Protection

**Encrypt Sensitive Data:**

```bash
# Create encryption key backup
echo "Encryption Key: $(grep N8N_ENCRYPTION_KEY ~/.env | cut -d'=' -f2)" > ~/n8n-docker/encryption-key-backup.txt
chmod 600 ~/n8n-docker/encryption-key-backup.txt

# Encrypt backup file
gpg -c ~/n8n-docker/encryption-key-backup.txt
rm ~/n8n-docker/encryption-key-backup.txt
```

**Secure File Permissions:**

```bash
# Set secure permissions
chmod -R 750 ~/n8n-docker/
chmod 600 ~/n8n-docker/.env
chmod -R 700 ~/n8n-docker/data/
chmod -R 700 ~/n8n-docker/backups/
```

## Backup and Data Management {#backup}

### Step 1: Automated Backup System

**Create Backup Script:**

```bash
# Create comprehensive backup script
cat > ~/n8n-docker/scripts/backup-n8n.sh << 'EOF'
#!/bin/bash
set -e

# Configuration
BACKUP_DIR="/home/$USER/n8n-docker/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="n8n_backup_$DATE"

# Create backup directory
mkdir -p "$BACKUP_DIR/$BACKUP_NAME"

echo "Starting n8n backup at $(date)"

# Stop n8n container for consistent backup
echo "Stopping n8n container..."
docker compose -f ~/n8n-docker/docker-compose.yml stop n8n

# Backup database and files
echo "Backing up data directory..."
cp -r ~/n8n-docker/data "$BACKUP_DIR/$BACKUP_NAME/"

# Backup configuration files
echo "Backing up configuration..."
cp ~/n8n-docker/.env "$BACKUP_DIR/$BACKUP_NAME/"
cp ~/n8n-docker/docker-compose.yml "$BACKUP_DIR/$BACKUP_NAME/"

# Export workflows
echo "Exporting workflows..."
docker compose -f ~/n8n-docker/docker-compose.yml start n8n
sleep 30  # Wait for n8n to start

docker compose -f ~/n8n-docker/docker-compose.yml exec -T n8n n8n export:workflow --all --output="/tmp/workflows_backup.json"
docker compose -f ~/n8n-docker/docker-compose.yml cp n8n:/tmp/workflows_backup.json "$BACKUP_DIR/$BACKUP_NAME/"

# Export credentials (encrypted)
echo "Exporting credentials..."
docker compose -f ~/n8n-docker/docker-compose.yml exec -T n8n n8n export:credentials --all --output="/tmp/credentials_backup.json"
docker compose -f ~/n8n-docker/docker-compose.yml cp n8n:/tmp/credentials_backup.json "$BACKUP_DIR/$BACKUP_NAME/"

# Create archive
echo "Creating backup archive..."
tar -czf "$BACKUP_DIR/$BACKUP_NAME.tar.gz" -C "$BACKUP_DIR" "$BACKUP_NAME"

# Clean up uncompressed backup
rm -rf "$BACKUP_DIR/$BACKUP_NAME"

# Remove old backups (keep last 7 days)
find "$BACKUP_DIR" -name "n8n_backup_*.tar.gz" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/$BACKUP_NAME.tar.gz"
echo "Backup size: $(du -h "$BACKUP_DIR/$BACKUP_NAME.tar.gz" | cut -f1)"
EOF

chmod +x ~/n8n-docker/scripts/backup-n8n.sh
```

### Step 2: Backup Scheduling

**Setup Cron Job:**

```bash
# Add to crontab
crontab -e

# Add the following line for daily backups at 2 AM
0 2 * * * /home/$USER/n8n-docker/scripts/backup-n8n.sh >> /home/$USER/n8n-docker/logs/backup.log 2>&1
```

### Step 3: Restore Procedures

**Create Restore Script:**

```bash
cat > ~/n8n-docker/scripts/restore-n8n.sh << 'EOF'
#!/bin/bash
set -e

if [ -z "$1" ]; then
    echo "Usage: $0 <backup_file.tar.gz>"
    echo "Available backups:"
    ls -la ~/n8n-docker/backups/n8n_backup_*.tar.gz
    exit 1
fi

BACKUP_FILE="$1"
RESTORE_DIR="/tmp/n8n_restore_$(date +%Y%m%d_%H%M%S)"

echo "Starting restore from $BACKUP_FILE"

# Stop n8n
docker compose -f ~/n8n-docker/docker-compose.yml down

# Extract backup
mkdir -p "$RESTORE_DIR"
tar -xzf "$BACKUP_FILE" -C "$RESTORE_DIR"

# Find extracted directory
EXTRACTED_DIR=$(find "$RESTORE_DIR" -maxdepth 1 -type d -name "n8n_backup_*" | head -1)

if [ -z "$EXTRACTED_DIR" ]; then
    echo "Error: Could not find extracted backup directory"
    exit 1
fi

# Backup current data (just in case)
mv ~/n8n-docker/data ~/n8n-docker/data.backup.$(date +%Y%m%d_%H%M%S)

# Restore data
cp -r "$EXTRACTED_DIR/data" ~/n8n-docker/

# Restore configuration (ask for confirmation)
echo "Do you want to restore configuration files? (y/N)"
read -r response
if [[ "$response" =~ ^[Yy]$ ]]; then
    cp "$EXTRACTED_DIR/.env" ~/n8n-docker/
    cp "$EXTRACTED_DIR/docker-compose.yml" ~/n8n-docker/
fi

# Set permissions
sudo chown -R 1000:1000 ~/n8n-docker/data
chmod -R 755 ~/n8n-docker/data

# Start n8n
docker compose -f ~/n8n-docker/docker-compose.yml up -d

echo "Restore completed. Please verify your workflows and credentials."
echo "Previous data backed up to: ~/n8n-docker/data.backup.*"

# Clean up
rm -rf "$RESTORE_DIR"
EOF

chmod +x ~/n8n-docker/scripts/restore-n8n.sh
```

### Step 4: Backup Verification

**Create Backup Test Script:**

```bash
cat > ~/n8n-docker/scripts/test-backup.sh << 'EOF'
#!/bin/bash
# Test backup integrity

LATEST_BACKUP=$(ls -t ~/n8n-docker/backups/n8n_backup_*.tar.gz | head -1)

if [ -z "$LATEST_BACKUP" ]; then
    echo "No backups found!"
    exit 1
fi

echo "Testing backup: $LATEST_BACKUP"

# Test archive integrity
if tar -tzf "$LATEST_BACKUP" > /dev/null 2>&1; then
    echo "✓ Archive integrity: PASS"
else
    echo "✗ Archive integrity: FAIL"
    exit 1
fi

# Check backup contents
TEMP_DIR="/tmp/backup_test_$(date +%s)"
mkdir -p "$TEMP_DIR"
tar -xzf "$LATEST_BACKUP" -C "$TEMP_DIR"

EXTRACTED_DIR=$(find "$TEMP_DIR" -maxdepth 1 -type d -name "n8n_backup_*" | head -1)

# Check for required files
FILES_TO_CHECK=("data" ".env" "docker-compose.yml" "workflows_backup.json" "credentials_backup.json")

for file in "${FILES_TO_CHECK[@]}"; do
    if [ -e "$EXTRACTED_DIR/$file" ]; then
        echo "✓ $file: PRESENT"
    else
        echo "✗ $file: MISSING"
    fi
done

# Clean up
rm -rf "$TEMP_DIR"

echo "Backup test completed."
EOF

chmod +x ~/n8n-docker/scripts/test-backup.sh
```

## Monitoring and Maintenance {#monitoring}

### Step 1: System Monitoring

**Create Monitoring Dashboard:**

```bash
# Install monitoring tools
sudo apt install -y htop iotop nethogs

# Create system monitoring script
cat > ~/n8n-docker/scripts/monitor-system.sh << 'EOF'
#!/bin/bash
# Comprehensive system monitoring for n8n

echo "=== n8n System Monitoring Report ==="
echo "Generated: $(date)"
echo ""

echo "=== Container Status ==="
docker compose -f ~/n8n-docker/docker-compose.yml ps

echo ""
echo "=== Container Resource Usage ==="
docker stats n8n --no-stream

echo ""
echo "=== Disk Usage ==="
df -h ~/n8n-docker/

echo ""
echo "=== Memory Usage ==="
free -h

echo ""
echo "=== CPU Usage ==="
uptime

echo ""
echo "=== Network Connections ==="
ss -tuln | grep :5678

echo ""
echo "=== Recent Logs (Last 20 lines) ==="
docker compose -f ~/n8n-docker/docker-compose.yml logs --tail 20 n8n

echo ""
echo "=== Workflow Execution Status ==="
docker compose -f ~/n8n-docker/docker-compose.yml exec -T n8n n8n list:execution --limit=5
EOF

chmod +x ~/n8n-docker/scripts/monitor-system.sh
```

### Step 2: Application Monitoring

**Workflow Performance Monitoring:**

```bash
cat > ~/n8n-docker/scripts/monitor-workflows.sh << 'EOF'
#!/bin/bash
# Monitor n8n workflow performance

echo "=== Workflow Performance Report ==="
echo "Generated: $(date)"
echo ""

# Get execution statistics
echo "=== Execution Statistics ==="
docker compose -f ~/n8n-docker/docker-compose.yml exec -T n8n node -e "
const fs = require('fs');
const path = require('path');
const dbPath = '/home/node/.n8n/database.sqlite';

if (fs.existsSync(dbPath)) {
    console.log('Database size:', (fs.statSync(dbPath).size / 1024 / 1024).toFixed(2), 'MB');
} else {
    console.log('Database not found');
}
"

# Check for errors in logs
echo ""
echo "=== Recent Errors ==="
docker compose -f ~/n8n-docker/docker-compose.yml logs --since 24h n8n | grep -i error | tail -10

# Check execution times
echo ""
echo "=== Average Response Times ==="
docker compose -f ~/n8n-docker/docker-compose.yml logs --since 1h n8n | grep -E "execution.*finished" | tail -5
EOF

chmod +x ~/n8n-docker/scripts/monitor-workflows.sh
```

### Step 3: Automated Alerts

**Setup Alert System:**

```bash
cat > ~/n8n-docker/scripts/alert-system.sh << 'EOF'
#!/bin/bash
# n8n Alert System

# Configuration
TELEGRAM_BOT_TOKEN="YOUR_BOT_TOKEN"
TELEGRAM_CHAT_ID="YOUR_CHAT_ID"
LOG_FILE="/home/$USER/n8n-docker/logs/alerts.log"

# Function to send Telegram alert
send_alert() {
    local message="$1"
    curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/sendMessage" \
         -d "chat_id=$TELEGRAM_CHAT_ID" \
         -d "text=🚨 n8n Alert: $message" \
         -d "parse_mode=HTML" > /dev/null
    
    echo "$(date): ALERT - $message" >> "$LOG_FILE"
}

# Check if n8n container is running
if ! docker compose -f ~/n8n-docker/docker-compose.yml ps | grep -q "Up"; then
    send_alert "n8n container is not running!"
fi

# Check disk space
DISK_USAGE=$(df ~/n8n-docker/ | awk 'NR==2 {gsub(/%/,""); print $5}')
if [ "$DISK_USAGE" -gt 85 ]; then
    send_alert "Disk usage is high: ${DISK_USAGE}%"
fi

# Check for recent errors
ERROR_COUNT=$(docker compose -f ~/n8n-docker/docker-compose.yml logs --since 1h n8n 2>/dev/null | grep -i error | wc -l)
if [ "$ERROR_COUNT" -gt 5 ]; then
    send_alert "High error rate detected: $ERROR_COUNT errors in the last hour"
fi

# Check memory usage
MEMORY_USAGE=$(docker stats n8n --no-stream --format "{{.MemPerc}}" | sed 's/%//')
if (( $(echo "$MEMORY_USAGE > 90" | bc -l) )); then
    send_alert "High memory usage: ${MEMORY_USAGE}%"
fi
EOF

chmod +x ~/n8n-docker/scripts/alert-system.sh

# Add to crontab for regular monitoring
(crontab -l 2>/dev/null; echo "*/15 * * * * /home/$USER/n8n-docker/scripts/alert-system.sh") | crontab -
```

### Step 4: Log Management

**Setup Log Rotation:**

```bash
# Create logrotate configuration
sudo tee /etc/logrotate.d/n8n << 'EOF'
/home/*/n8n-docker/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    copytruncate
    su root root
}
EOF

# Create log analysis script
cat > ~/n8n-docker/scripts/analyze-logs.sh << 'EOF'
#!/bin/bash
# Analyze n8n logs for patterns and issues

echo "=== n8n Log Analysis Report ==="
echo "Generated: $(date)"
echo ""

# Get recent activity summary
echo "=== Activity Summary (Last 24h) ==="
docker compose -f ~/n8n-docker/docker-compose.yml logs --since 24h n8n | \
  grep -E "(workflow|execution|email)" | \
  awk '{print $1 " " $2}' | \
  sort | uniq -c | sort -nr | head -10

echo ""
echo "=== Error Analysis ==="
docker compose -f ~/n8n-docker/docker-compose.yml logs --since 24h n8n | \
  grep -i error | \
  awk '{print $NF}' | \
  sort | uniq -c | sort -nr

echo ""
echo "=== Performance Metrics ==="
echo "Container uptime:"
docker inspect n8n --format='{{.State.StartedAt}}' | xargs -I {} date -d {} '+%Y-%m-%d %H:%M:%S'

echo ""
echo "Recent execution times:"
docker compose -f ~/n8n-docker/docker-compose.yml logs --since 1h n8n | \
  grep -E "execution.*finished" | tail -5
EOF

chmod +x ~/n8n-docker/scripts/analyze-logs.sh
```

## Troubleshooting Guide {#troubleshooting}

### Step 1: Common Issues and Solutions

**Container Won't Start:**

```bash
# Check Docker service
sudo systemctl status docker

# Check compose file syntax
docker compose -f ~/n8n-docker/docker-compose.yml config

# Check logs for startup errors
docker compose -f ~/n8n-docker/docker-compose.yml logs n8n

# Common fixes:
# 1. Check port availability
sudo netstat -tulpn | grep :5678

# 2. Check file permissions
ls -la ~/n8n-docker/data/
sudo chown -R 1000:1000 ~/n8n-docker/data/

# 3. Check environment variables
cat ~/n8n-docker/.env
```

**Cannot Access Web Interface:**

```bash
# Check if container is running
docker compose -f ~/n8n-docker/docker-compose.yml ps

# Check port binding
docker port n8n

# Test local connection
curl -I http://localhost:5678

# Check firewall
sudo ufw status

# Check nginx (if using reverse proxy)
sudo nginx -t
sudo systemctl status nginx
```

**Credential Issues:**

```bash
# Check credential storage
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n ls -la /home/node/.n8n/

# Test API connections
# (Use appropriate test commands for each service)

# Reset credentials if needed
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n n8n reset --all
```

### Step 2: Performance Issues

**High Memory Usage:**

```bash
# Monitor memory usage
docker stats n8n --no-stream

# Check for memory leaks
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n node -e "
console.log('Memory usage:', process.memoryUsage());
setTimeout(() => console.log('Memory after 30s:', process.memoryUsage()), 30000);
"

# Restart container to clear memory
docker compose -f ~/n8n-docker/docker-compose.yml restart n8n
```

**Slow Workflow Execution:**

```bash
# Check system resources
htop

# Monitor I/O
iotop

# Check database size
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n du -sh /home/node/.n8n/database.sqlite

# Optimize database (if needed)
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n sqlite3 /home/node/.n8n/database.sqlite "VACUUM;"
```

### Step 3: Network Issues

**API Connection Problems:**

```bash
# Test external connectivity
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n curl -I https://api.openai.com
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n curl -I https://graph.microsoft.com

# Check DNS resolution
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n nslookup api.openai.com

# Test from host system
curl -I https://api.openai.com
curl -I https://graph.microsoft.com
```

**Webhook Issues:**

```bash
# Check webhook URLs
docker compose -f ~/n8n-docker/docker-compose.yml logs n8n | grep webhook

# Test webhook endpoint
curl -X POST http://localhost:5678/webhook-test/your-webhook-id \
     -H "Content-Type: application/json" \
     -d '{"test": "data"}'
```

### Step 4: Data Issues

**Database Corruption:**

```bash
# Check database integrity
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n sqlite3 /home/node/.n8n/database.sqlite "PRAGMA integrity_check;"

# Backup current database
docker compose -f ~/n8n-docker/docker-compose.yml cp n8n:/home/node/.n8n/database.sqlite ~/n8n-docker/backups/database-backup-$(date +%Y%m%d).sqlite

# Restore from backup if needed
# (Use restore script created earlier)
```

**Lost Workflows:**

```bash
# Check if workflows exist in database
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n n8n list:workflow

# Import from backup
docker compose -f ~/n8n-docker/docker-compose.yml exec n8n n8n import:workflow --input=/path/to/backup.json

# Manually recreate if necessary
```

### Step 5: Emergency Procedures

**Complete System Recovery:**

```bash
cat > ~/n8n-docker/scripts/emergency-recovery.sh << 'EOF'
#!/bin/bash
# Emergency recovery procedure for n8n

echo "Starting emergency recovery procedure..."

# 1. Stop all services
docker compose -f ~/n8n-docker/docker-compose.yml down

# 2. Create emergency backup
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p ~/n8n-docker/emergency-backup-$DATE
cp -r ~/n8n-docker/data ~/n8n-docker/emergency-backup-$DATE/
cp ~/n8n-docker/.env ~/n8n-docker/emergency-backup-$DATE/

# 3. Check system resources
df -h
free -h

# 4. Clean up if needed
docker system prune -f

# 5. Restart with fresh container
docker compose -f ~/n8n-docker/docker-compose.yml pull
docker compose -f ~/n8n-docker/docker-compose.yml up -d

# 6. Wait for startup
sleep 60

# 7. Verify functionality
if curl -f -s http://localhost:5678/healthz > /dev/null; then
    echo "✓ n8n is responding"
else
    echo "✗ n8n is not responding - manual intervention required"
fi

echo "Emergency recovery procedure completed."
EOF

chmod +x ~/n8n-docker/scripts/emergency-recovery.sh
```

## Advanced Configuration {#advanced-configuration}

### Step 1: Custom Node Development

**Setting Up Development Environment:**

```bash
# Create custom nodes directory
mkdir -p ~/n8n-docker/custom-nodes

# Create package.json for custom nodes
cat > ~/n8n-docker/custom-nodes/package.json << 'EOF'
{
  "name": "n8n-custom-nodes",
  "version": "1.0.0",
  "description": "Custom nodes for n8n",
  "main": "index.js",
  "dependencies": {},
  "n8n": {
    "nodes": [
      "nodes/**/*.node.js"
    ]
  }
}
EOF

# Update docker-compose.yml to include custom nodes
cat >> ~/n8n-docker/docker-compose.yml << 'EOF'
    volumes:
      - ./data:/home/node/.n8n
      - ./custom-nodes:/home/node/.n8n/custom
EOF
```

### Step 2: Performance Optimization

**Advanced Environment Configuration:**

```bash
# Update .env with performance optimizations
cat >> ~/n8n-docker/.env << 'EOF'

# Performance Optimizations
NODE_OPTIONS=--max-old-space-size=4096
N8N_BINARY_DATA_TTL=1440
N8N_DEFAULT_BINARY_DATA_MODE=filesystem
EXECUTIONS_PROCESS=main
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=336
N8N_CONCURRENCY_PRODUCTION_LIMIT=10

# Queue Configuration
QUEUE_BULL_REDIS_HOST=redis
QUEUE_BULL_REDIS_PORT=6379
QUEUE_BULL_REDIS_DB=0

# Metrics
N8N_METRICS=true
N8N_METRICS_PREFIX=n8n_
EOF
```

**Redis Integration for Scaling:**

```bash
# Add Redis to docker-compose.yml
cat >> ~/n8n-docker/docker-compose.yml << 'EOF'
  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    restart: unless-stopped
    volumes:
      - redis-data:/data
    networks:
      - n8n-network
    command: redis-server --appendonly yes

volumes:
  redis-data:
    driver: local
EOF
```

### Step 3: Multi-Instance Setup

**Load Balancer Configuration:**

```bash
# Create nginx load balancer config
sudo tee /etc/nginx/sites-available/n8n-lb << 'EOF'
upstream n8n_backend {
    server localhost:5678;
    server localhost:5679;
    server localhost:5680;
}

server {
    listen 80;
    server_name your-domain.com;
    
    location / {
        proxy_pass http://n8n_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
EOF
```

### Step 4: Advanced Monitoring

**Prometheus Integration:**

```bash
# Add Prometheus to docker-compose.yml
cat >> ~/n8n-docker/docker-compose.yml << 'EOF'
  prometheus:
    image: prom/prometheus:latest
    container_name: n8n-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - n8n-network

  grafana:
    image: grafana/grafana:latest
    container_name: n8n-grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - n8n-network

volumes:
  grafana-data:
    driver: local
EOF

# Create Prometheus configuration
mkdir -p ~/n8n-docker/monitoring
cat > ~/n8n-docker/monitoring/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'n8n'
    static_configs:
      - targets: ['n8n:5678']
EOF
```

This comprehensive guide provides everything needed to successfully deploy and run n8n with the AI-powered Outlook Inbox Manager workflow on Ubuntu 24.04.01. The step-by-step instructions cover installation, configuration, security, monitoring, and troubleshooting to ensure a robust, production-ready automation system.

The guide emphasizes security best practices, automated backups, comprehensive monitoring, and provides troubleshooting procedures for common issues. Following this guide will result in a fully functional, secure, and maintainable n8n deployment capable of intelligent email processing using multiple AI services.
