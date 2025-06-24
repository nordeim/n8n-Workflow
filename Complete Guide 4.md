# Complete Guide: Deploying n8n with Docker on Ubuntu 24.04.01 LTS

## Table of Contents
1. [Introduction and Overview](#introduction)
2. [Prerequisites and System Requirements](#prerequisites)
3. [Installing Docker on Ubuntu 24.04.01](#installing-docker)
4. [Setting Up n8n with Docker Compose](#setting-up-n8n)
5. [Configuring n8n for Production Use](#configuring-n8n)
6. [Importing and Setting Up the Invoice Generator Workflow](#importing-workflow)
7. [Configuring Required Services and Credentials](#configuring-credentials)
8. [Testing and Troubleshooting](#testing)
9. [Security and Maintenance](#security)
10. [Conclusion and Next Steps](#conclusion)

## 1. Introduction and Overview {#introduction}

This guide will walk you through the complete process of setting up n8n, a powerful workflow automation platform, using Docker on Ubuntu 24.04.01 LTS. By the end of this guide, you'll have a fully functional n8n instance running the Invoice Generator workflow we explored earlier.

### What We'll Accomplish

1. **Install Docker**: Set up Docker and Docker Compose on Ubuntu
2. **Deploy n8n**: Create a production-ready n8n container
3. **Configure Storage**: Set up persistent data storage
4. **Import Workflow**: Load the Invoice Generator workflow
5. **Set Up Integrations**: Configure Telegram and OpenAI connections
6. **Secure the Installation**: Implement basic security measures

### Why Docker?

Docker provides several advantages for running n8n:
- **Isolation**: n8n runs in its own environment
- **Easy Updates**: Simple commands to update n8n
- **Portability**: Move your setup between servers easily
- **Resource Management**: Control CPU and memory usage
- **Quick Recovery**: Easy backup and restore

### Time Required

- Initial setup: 30-45 minutes
- Workflow configuration: 20-30 minutes
- Total time: 50-75 minutes

## 2. Prerequisites and System Requirements {#prerequisites}

### System Requirements

Before we begin, ensure your Ubuntu 24.04.01 system meets these requirements:

**Minimum Hardware:**
- CPU: 2 cores (4 cores recommended)
- RAM: 2GB minimum (4GB recommended)
- Storage: 10GB free space
- Network: Stable internet connection

**Software Requirements:**
- Ubuntu 24.04.01 LTS (fresh installation or updated system)
- Terminal access (local or SSH)
- sudo privileges (administrator access)

### Checking Your System

Open a terminal and run these commands to verify your system:

```bash
# Check Ubuntu version
lsb_release -a

# Check available memory
free -h

# Check available disk space
df -h

# Check CPU information
lscpu | grep -E '^CPU\(s\):'
```

### Required Accounts

You'll need accounts for:
1. **Telegram**: For the bot integration
2. **OpenAI**: For AI capabilities
3. **Docker Hub** (optional): For pulling images

### Network Requirements

Ensure these ports are available:
- **5678**: n8n web interface
- **5679**: n8n webhook port (optional)

Check if ports are in use:
```bash
sudo lsof -i :5678
sudo lsof -i :5679
```

If these commands return nothing, the ports are free.

## 3. Installing Docker on Ubuntu 24.04.01 {#installing-docker}

### Step 1: Update System Packages

First, update your system to ensure all packages are current:

```bash
# Update package index
sudo apt update

# Upgrade installed packages
sudo apt upgrade -y

# Install required packages
sudo apt install -y curl wget software-properties-common apt-transport-https ca-certificates gnupg lsb-release
```

### Step 2: Add Docker's Official Repository

Docker isn't included in Ubuntu's default repositories, so we'll add Docker's official repository:

```bash
# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Step 3: Install Docker Engine

Now install Docker:

```bash
# Update package index with Docker packages
sudo apt update

# Install Docker Engine, CLI, and containerd
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Step 4: Verify Docker Installation

Check that Docker is installed correctly:

```bash
# Check Docker version
docker --version

# Verify Docker is running
sudo systemctl status docker

# Test Docker with hello-world image
sudo docker run hello-world
```

You should see a message saying "Hello from Docker!"

### Step 5: Configure Docker for Non-Root Access (Optional but Recommended)

To run Docker commands without sudo:

```bash
# Add your user to the docker group
sudo usermod -aG docker $USER

# Apply the new group membership
newgrp docker

# Verify you can run docker without sudo
docker ps
```

**Note**: Log out and back in for changes to take full effect.

### Step 6: Install Docker Compose

Docker Compose is now included as a plugin, but let's verify:

```bash
# Check Docker Compose version
docker compose version
```

You should see version 2.x or higher.

## 4. Setting Up n8n with Docker Compose {#setting-up-n8n}

### Step 1: Create n8n Directory Structure

Let's create a organized directory structure for n8n:

```bash
# Create main n8n directory
mkdir -p ~/n8n-docker

# Navigate to the directory
cd ~/n8n-docker

# Create subdirectories
mkdir -p data workflows backups logs

# Create configuration directory
mkdir -p config
```

### Step 2: Create Docker Compose File

Create a `docker-compose.yml` file with a production-ready configuration:

```bash
# Create docker-compose.yml
nano docker-compose.yml
```

Add the following content:

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
      - N8N_PATH=/
      - N8N_PUSH_BACKEND=websocket
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=changethispassword
      - N8N_ENCRYPTION_KEY=your-encryption-key-here
      - WEBHOOK_URL=http://localhost:5678/
      - GENERIC_TIMEZONE=America/New_York
      - TZ=America/New_York
      - N8N_LOG_LEVEL=info
      - N8N_LOG_OUTPUT=console,file
      - N8N_LOG_FILE_LOCATION=/home/node/logs/n8n.log
      - DB_TYPE=sqlite
      - DB_SQLITE_DATABASE_FILE=/home/node/n8n/database.sqlite
      - N8N_USER_MANAGEMENT_DISABLED=false
      - N8N_VERSION_NOTIFICATIONS_ENABLED=true
      - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_PERSONALIZATION_ENABLED=false
      - N8N_DEFAULT_BINARY_DATA_MODE=filesystem
      - N8N_BINARY_DATA_STORAGE_PATH=/home/node/binaryData
      - N8N_EXTERNAL_STORAGE_S3_BUCKET_NAME=
      - N8N_METRICS=false
      - N8N_TEMPLATES_ENABLED=true
      - N8N_TEMPLATES_HOST=https://api.n8n.io/
      - EXECUTIONS_PROCESS=main
      - N8N_HIRING_BANNER_ENABLED=false
    volumes:
      - ./data:/home/node/.n8n
      - ./data/database:/home/node/n8n
      - ./data/binaryData:/home/node/binaryData
      - ./workflows:/home/node/workflows
      - ./backups:/home/node/backups
      - ./logs:/home/node/logs
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    networks:
      - n8n-network

networks:
  n8n-network:
    driver: bridge
```

Save the file (Ctrl+X, then Y, then Enter in nano).

### Step 3: Create Environment File

For better security, create an `.env` file for sensitive values:

```bash
# Create .env file
nano .env
```

Add the following content:

```env
# Basic Authentication
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your-secure-password-here

# Encryption Key (generate a random string)
N8N_ENCRYPTION_KEY=your-32-character-encryption-key

# Database
DB_TYPE=sqlite
DB_SQLITE_DATABASE_FILE=/home/node/n8n/database.sqlite

# Timezone (change to your timezone)
GENERIC_TIMEZONE=America/New_York
TZ=America/New_York

# n8n Configuration
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
NODE_ENV=production

# Webhook URL (update with your domain if using one)
WEBHOOK_URL=http://localhost:5678/
```

### Step 4: Generate Secure Values

Generate secure passwords and encryption key:

```bash
# Generate a secure password
openssl rand -base64 32

# Generate encryption key
openssl rand -hex 32
```

Update the `.env` file with these generated values.

### Step 5: Update Docker Compose to Use Environment File

Modify `docker-compose.yml` to use the environment file:

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    env_file:
      - .env
    volumes:
      - ./data:/home/node/.n8n
      - ./data/database:/home/node/n8n
      - ./data/binaryData:/home/node/binaryData
      - ./workflows:/home/node/workflows
      - ./backups:/home/node/backups
      - ./logs:/home/node/logs
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:5678/healthz"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    networks:
      - n8n-network

networks:
  n8n-network:
    driver: bridge
```

### Step 6: Set Correct Permissions

Ensure proper permissions for n8n directories:

```bash
# Set ownership (n8n runs as user 1000)
sudo chown -R 1000:1000 ~/n8n-docker/data
sudo chown -R 1000:1000 ~/n8n-docker/logs
sudo chown -R 1000:1000 ~/n8n-docker/workflows
sudo chown -R 1000:1000 ~/n8n-docker/backups

# Set permissions
chmod -R 755 ~/n8n-docker
```

## 5. Starting and Configuring n8n {#configuring-n8n}

### Step 1: Start n8n Container

Now let's start n8n:

```bash
# Navigate to n8n directory
cd ~/n8n-docker

# Start n8n in detached mode
docker compose up -d

# Check if container is running
docker compose ps

# View logs
docker compose logs -f n8n
```

You should see n8n starting up. Wait for the message "n8n ready on http://localhost:5678"

### Step 2: Access n8n Web Interface

1. Open a web browser
2. Navigate to: `http://localhost:5678`
3. You'll see a login prompt

### Step 3: Initial Setup

1. **First Login**:
   - Username: `admin` (or what you set in .env)
   - Password: Your password from .env file

2. **Create Owner Account**:
   - Email: Your email address
   - First Name: Your first name
   - Last Name: Your last name
   - Password: Choose a strong password

3. **Complete Setup**:
   - Skip the survey (optional)
   - You'll be redirected to the main dashboard

### Step 4: Configure n8n Settings

1. Click on your profile icon (top right)
2. Select "Settings"
3. Configure these important settings:

**General Settings:**
- Timezone: Verify it matches your location
- Save execution progress: Enable for debugging
- Save manual executions: Enable

**Security Settings:**
- Two-factor authentication: Consider enabling
- Session timeout: Set according to your needs

### Step 5: Test n8n Installation

Create a simple test workflow:

1. Click "New Workflow"
2. Add a "Manual Trigger" node
3. Add a "Set" node
4. Connect them
5. In the Set node, add:
   - Name: `test`
   - Value: `Hello from n8n!`
6. Click "Execute Workflow"
7. You should see the output

## 6. Importing and Setting Up the Invoice Generator Workflow {#importing-workflow}

### Step 1: Prepare the Workflow JSON

Create a file for the workflow:

```bash
# Navigate to workflows directory
cd ~/n8n-docker/workflows

# Create workflow file
nano invoice-generator.json
```

Copy the entire Invoice Generator workflow JSON from our previous guide into this file.

### Step 2: Import the Workflow

1. In n8n web interface, click "Workflows" in the sidebar
2. Click the "..." menu (three dots) in the top right
3. Select "Import from File"
4. Choose the `invoice-generator.json` file
5. Click "Import"

### Step 3: Understanding the Workflow Components

The imported workflow contains:
- **Telegram Trigger**: Receives messages
- **AI Agent**: Processes requests
- **OpenAI Chat Model**: Provides AI capabilities
- **Window Buffer Memory**: Maintains conversation context
- **Document/Image Processing Tools**: Handle file processing
- **Telegram Response**: Sends replies

### Step 4: Workflow Configuration Overview

Before the workflow can function, we need to:
1. Set up Telegram Bot credentials
2. Configure OpenAI API access
3. Create sub-workflows for document/image processing
4. Test each component

## 7. Configuring Required Services and Credentials {#configuring-credentials}

### Step 1: Setting Up Telegram Bot

**Create a Telegram Bot:**

1. Open Telegram and search for "@BotFather"
2. Start a conversation and send `/newbot`
3. Choose a name for your bot (e.g., "Invoice Processor")
4. Choose a username (must end in 'bot', e.g., "invoice_processor_bot")
5. Save the API token provided

**Add Telegram Credentials in n8n:**

1. In n8n, go to "Credentials" (left sidebar)
2. Click "Add Credential"
3. Search for "Telegram"
4. Select "Telegram API"
5. Enter:
   - Name: "Telegram Bot API"
   - Access Token: Your bot token from BotFather
6. Click "Create"

**Configure Webhook:**

```bash
# Get your bot updates
curl https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates

# Set webhook (replace with your actual URL if not localhost)
curl https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=http://localhost:5678/webhook/telegram-trigger-webhook-id
```

### Step 2: Setting Up OpenAI API

**Get OpenAI API Key:**

1. Go to [https://platform.openai.com/](https://platform.openai.com/)
2. Sign up or log in
3. Navigate to API Keys section
4. Create a new API key
5. Save the key securely

**Add OpenAI Credentials in n8n:**

1. In n8n, go to "Credentials"
2. Click "Add Credential"
3. Search for "OpenAI"
4. Select "OpenAI API"
5. Enter:
   - Name: "OpenAI API"
   - API Key: Your OpenAI API key
6. Click "Create"

### Step 3: Creating Sub-Workflows

The Invoice Generator uses two sub-workflows. Let's create simple versions:

**Document Processing Sub-Workflow:**

1. Create a new workflow
2. Name it "Document Processing"
3. Add these nodes:
   - Workflow Trigger (start node)
   - Code node for processing
   - Respond to Webhook node

**Simple Document Processing Code:**
```javascript
// In the Code node
const input = $input.first();
const documentId = input.json.id;
const message = input.json.message;

// Simulate document processing
return [{
  json: {
    success: true,
    documentId: documentId,
    extractedText: "Sample invoice data extracted",
    message: message,
    processedAt: new Date().toISOString()
  }
}];
```

4. Save the workflow
5. Note the workflow ID from the URL

**Image Processing Sub-Workflow:**

Follow similar steps for image processing workflow.

### Step 4: Update Main Workflow References

1. Open the Invoice Generator workflow
2. Click on "Document processing" node
3. Update the workflow ID to match your created sub-workflow
4. Repeat for "Image processing" node
5. Save the workflow

### Step 5: Testing Credentials

Test each credential:

1. **Telegram**: Send a message to your bot
2. **OpenAI**: Execute a simple prompt in the workflow
3. **Sub-workflows**: Test with sample data

## 8. Testing and Troubleshooting {#testing}

### Step 1: Initial Workflow Test

1. Activate the workflow (toggle switch in workflow list)
2. Send a test message to your Telegram bot
3. Monitor the execution in n8n

### Step 2: Common Issues and Solutions

**Issue 1: Telegram Webhook Not Receiving Messages**

```bash
# Check webhook status
curl https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getWebhookInfo

# If webhook URL is wrong, reset it
curl https://api.telegram.org/bot<YOUR_BOT_TOKEN>/deleteWebhook
curl https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<CORRECT_URL>
```

**Issue 2: OpenAI Errors**

Common causes:
- Invalid API key
- Exceeded quota
- Model not available

Solution:
- Verify API key in OpenAI dashboard
- Check usage limits
- Use available models (gpt-3.5-turbo if gpt-4 not available)

**Issue 3: Workflow Execution Errors**

Debug steps:
1. Click on the execution in the Executions list
2. Check each node's output
3. Look for red error indicators
4. Read error messages carefully

**Issue 4: Permission Errors**

```bash
# Fix permission issues
sudo chown -R 1000:1000 ~/n8n-docker/data
docker compose restart n8n
```

### Step 3: Monitoring Logs

```bash
# View real-time logs
docker compose logs -f n8n

# View last 100 lines
docker compose logs --tail=100 n8n

# Check specific log file
cat ~/n8n-docker/logs/n8n.log
```

### Step 4: Performance Monitoring

```bash
# Check container resource usage
docker stats n8n

# Check container details
docker inspect n8n
```

## 9. Security and Maintenance {#security}

### Step 1: Securing n8n Installation

**1. Use HTTPS with Nginx Reverse Proxy:**

Install Nginx:
```bash
sudo apt update
sudo apt install nginx
```

Create Nginx configuration:
```bash
sudo nano /etc/nginx/sites-available/n8n
```

Add configuration:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

**2. Install SSL Certificate with Certbot:**

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

**3. Configure Firewall:**

```bash
# Install UFW if not installed
sudo apt install ufw

# Allow SSH (important!)
sudo ufw allow 22/tcp

# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw enable
```

### Step 2: Backup Strategy

**Create Backup Script:**

```bash
nano ~/n8n-docker/backup.sh
```

Add content:
```bash
#!/bin/bash
# n8n Backup Script

# Variables
BACKUP_DIR="/home/$USER/n8n-docker/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="n8n_backup_$DATE"

# Create backup directory
mkdir -p "$BACKUP_DIR/$BACKUP_NAME"

# Stop n8n container
cd /home/$USER/n8n-docker
docker compose stop n8n

# Copy data
cp -r data "$BACKUP_DIR/$BACKUP_NAME/"
cp docker-compose.yml "$BACKUP_DIR/$BACKUP_NAME/"
cp .env "$BACKUP_DIR/$BACKUP_NAME/"

# Create tar archive
cd "$BACKUP_DIR"
tar -czf "$BACKUP_NAME.tar.gz" "$BACKUP_NAME"
rm -rf "$BACKUP_NAME"

# Start n8n container
cd /home/$USER/n8n-docker
docker compose start n8n

# Keep only last 7 backups
cd "$BACKUP_DIR"
ls -t *.tar.gz | tail -n +8 | xargs -r rm

echo "Backup completed: $BACKUP_NAME.tar.gz"
```

Make executable:
```bash
chmod +x ~/n8n-docker/backup.sh
```

**Schedule Automatic Backups:**

```bash
# Edit crontab
crontab -e

# Add daily backup at 2 AM
0 2 * * * /home/your-username/n8n-docker/backup.sh >> /home/your-username/n8n-docker/logs/backup.log 2>&1
```

### Step 3: Updating n8n

**Update Process:**

```bash
# Navigate to n8n directory
cd ~/n8n-docker

# Pull latest image
docker compose pull

# Backup before updating
./backup.sh

# Restart with new image
docker compose up -d

# Check logs
docker compose logs -f n8n
```

### Step 4: Monitoring and Alerts

**Basic Health Check Script:**

```bash
nano ~/n8n-docker/health-check.sh
```

Add content:
```bash
#!/bin/bash
# n8n Health Check

URL="http://localhost:5678/healthz"
EXPECTED="ok"

# Check health endpoint
RESPONSE=$(curl -s "$URL")

if [ "$RESPONSE" != "$EXPECTED" ]; then
    echo "n8n health check failed at $(date)"
    # Add notification method here (email, telegram, etc.)
    
    # Attempt restart
    cd /home/$USER/n8n-docker
    docker compose restart n8n
fi
```

Make executable and schedule:
```bash
chmod +x ~/n8n-docker/health-check.sh

# Add to crontab (every 5 minutes)
*/5 * * * * /home/your-username/n8n-docker/health-check.sh
```

## 10. Conclusion and Next Steps {#conclusion}

### What You've Accomplished

Congratulations! You've successfully:

1. ✅ Installed Docker on Ubuntu 24.04.01
2. ✅ Deployed n8n in a production-ready configuration
3. ✅ Set up persistent storage and logging
4. ✅ Imported a complex AI-powered workflow
5. ✅ Configured external service integrations
6. ✅ Implemented security measures
7. ✅ Created backup and monitoring systems

### Your n8n Installation Details

- **Access URL**: http://localhost:5678 (or https://your-domain.com)
- **Data Location**: ~/n8n-docker/data
- **Backups**: ~/n8n-docker/backups
- **Logs**: ~/n8n-docker/logs
- **Configuration**: ~/n8n-docker/docker-compose.yml

### Next Steps

**1. Enhance Security:**
- Implement IP whitelisting
- Set up VPN access for remote management
- Enable audit logging
- Configure rate limiting

**2. Expand Workflows:**
- Create more sub-workflows
- Build workflow templates
- Implement error handling workflows
- Set up monitoring workflows

**3. Optimize Performance:**
- Configure Redis for caching
- Set up PostgreSQL for better performance
- Implement load balancing for high availability

**4. Learn More:**
- Explore n8n's advanced features
- Join n8n community forums
- Watch tutorial videos
- Read documentation for new nodes

### Useful Commands Reference

```bash
# Container Management
docker compose up -d          # Start n8n
docker compose down          # Stop n8n
docker compose restart n8n   # Restart n8n
docker compose logs -f n8n   # View logs

# Backup and Restore
./backup.sh                  # Create backup
tar -xzf backup.tar.gz      # Extract backup

# Updates
docker compose pull         # Download latest version
docker compose up -d        # Apply update

# Troubleshooting
docker compose ps           # Check container status
docker stats n8n           # Monitor resources
docker exec -it n8n sh     # Access container shell
```

### Getting Help

**Resources:**
- Official Documentation: https://docs.n8n.io/
- Community Forum: https://community.n8n.io/
- GitHub Issues: https://github.com/n8n-io/n8n/issues
- Discord Server: https://discord.gg/n8n

**Common Support Topics:**
- Workflow debugging
- Performance optimization
- Integration issues
- Upgrade procedures

### Final Tips

1. **Regular Maintenance**: Schedule weekly checks of logs and performance
2. **Stay Updated**: Follow n8n's blog for new features and security updates
3. **Document Changes**: Keep notes of any customizations
4. **Test First**: Always test workflows in development before production
5. **Monitor Usage**: Keep track of API limits for external services

### Closing Thoughts

You now have a robust n8n installation capable of running complex AI-powered workflows. The Invoice Generator workflow demonstrates just a fraction of what's possible with n8n. As you become more comfortable with the platform, you'll discover countless ways to automate tasks and integrate services.

Remember that automation is a journey, not a destination. Start small, iterate often, and gradually build more complex workflows as your confidence grows. The n8n community is incredibly helpful, so don't hesitate to ask questions and share your creations.

Happy automating!
