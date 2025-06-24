# Complete Guide: Deploying n8n Human-in-the-Loop AI Workflow Automation with Docker on Ubuntu 24.04.01

## Table of Contents
1. [Introduction and Workflow Overview](#introduction-and-workflow-overview)
2. [Prerequisites and System Requirements](#prerequisites-and-system-requirements)
3. [Installing Docker on Ubuntu 24.04.01](#installing-docker-on-ubuntu-24-04-01)
4. [Setting up n8n with Docker Compose](#setting-up-n8n-with-docker-compose)
5. [Configuring Environment Variables](#configuring-environment-variables)
6. [Setting up External Service Accounts](#setting-up-external-service-accounts)
7. [Importing and Configuring the HITL Workflow](#importing-and-configuring-the-hitl-workflow)
8. [Setting up Credentials and API Integrations](#setting-up-credentials-and-api-integrations)
9. [Testing the Complete Workflow](#testing-the-complete-workflow)
10. [Security and Production Hardening](#security-and-production-hardening)
11. [Monitoring and Maintenance](#monitoring-and-maintenance)
12. [Troubleshooting Common Issues](#troubleshooting-common-issues)
13. [Advanced Configuration and Scaling](#advanced-configuration-and-scaling)
14. [Conclusion and Next Steps](#conclusion-and-next-steps)

## Introduction and Workflow Overview

This comprehensive guide will walk you through deploying a sophisticated Human-in-the-Loop (HITL) AI workflow automation system using n8n on Ubuntu 24.04.01. The workflow we'll implement demonstrates cutting-edge AI automation with human oversight, perfect for content creation, social media management, and AI-assisted decision making.

### What is Human-in-the-Loop (HITL)?

Human-in-the-Loop refers to AI systems that incorporate human judgment and feedback into automated processes. This approach combines the efficiency of automation with human intelligence and oversight, ensuring quality control and appropriate decision-making in critical workflows.

### Our HITL Workflow Features

The workflow we'll deploy includes two sophisticated automation flows:

**Flow 1: Simple Approval Workflow**
- Telegram-triggered content generation
- AI-powered X/Twitter post creation with web research
- Human yes/no approval via Telegram
- Automatic posting upon approval

**Flow 2: Advanced Feedback Workflow**  
- Telegram-triggered content generation
- AI-powered post creation with Tavily web search
- Human text-based feedback collection
- AI-powered content revision based on feedback
- Intelligent feedback classification (approved/needs revision)
- Iterative improvement loop

### Technologies and Integrations

Our deployment will integrate:
- **n8n**: Core workflow automation platform
- **Telegram**: User interface and approval mechanism
- **OpenRouter**: AI model gateway (GPT-4.1, Gemini 2.0 Flash)
- **Tavily**: Real-time web search and research
- **X/Twitter**: Social media posting platform
- **PostgreSQL**: Persistent data storage
- **Redis**: Queue management and caching
- **Docker**: Containerization and deployment

### Key Benefits

- **Quality Control**: Human oversight ensures content quality
- **Efficiency**: AI handles research and initial content creation
- **Flexibility**: Easy modification of approval processes
- **Scalability**: Handles multiple concurrent workflows
- **Transparency**: Complete audit trail of decisions and revisions

## Prerequisites and System Requirements

### Hardware Requirements

**Minimum System Specifications:**
- CPU: 4 cores (2.5 GHz or higher)
- RAM: 8 GB
- Storage: 50 GB available disk space (SSD recommended)
- Network: Stable broadband internet connection

**Recommended System Specifications:**
- CPU: 8 cores (3.0 GHz or higher)
- RAM: 16 GB or more
- Storage: 100 GB SSD storage
- Network: High-speed broadband with unlimited data

### Software Prerequisites

**Operating System:**
- Ubuntu 24.04.01 LTS (fresh installation recommended)
- Root or sudo access
- Updated package repositories

**Required Tools:**
- Terminal/Command line access
- Text editor (nano, vim, or VS Code)
- Web browser for accessing web interfaces

### Network Requirements

**Port Availability:**
- Port 5678: n8n web interface
- Port 5432: PostgreSQL database
- Port 6379: Redis cache
- Port 80/443: Optional reverse proxy

**Internet Access:**
- Outbound HTTPS (443) for API calls
- Webhook access for Telegram integration
- Docker Hub access for container images

### Service Account Prerequisites

Before starting, you'll need accounts with:

1. **Telegram**: Bot token for workflow triggers
2. **OpenRouter**: API key for AI language models
3. **Tavily**: API key for web search functionality
4. **X/Twitter**: API credentials for posting (optional)

### Knowledge Prerequisites

**Basic Understanding Of:**
- Linux command line operations
- Basic networking concepts
- API key management
- Docker containers (helpful but not required)

## Installing Docker on Ubuntu 24.04.01

Docker provides the containerization foundation for our n8n deployment. This section covers the complete Docker installation process.

### Step 1: System Preparation

Update your Ubuntu system and install prerequisites:

```bash
# Update package index
sudo apt update && sudo apt upgrade -y

# Install required packages for Docker installation
sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    software-properties-common \
    wget \
    unzip
```

### Step 2: Install Docker Engine

Add Docker's official repository and install Docker:

```bash
# Create directory for Docker's GPG key
sudo mkdir -p /etc/apt/keyrings

# Download and add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Set appropriate permissions
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update package index
sudo apt update

# Install Docker Engine and Docker Compose
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Step 3: Configure Docker for Current User

```bash
# Add current user to docker group
sudo usermod -aG docker $USER

# Apply group membership immediately
newgrp docker

# Test Docker installation
docker run hello-world
```

If successful, you should see a "Hello from Docker!" message.

### Step 4: Enable Docker System Services

```bash
# Enable Docker services to start on boot
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# Verify services are running
sudo systemctl status docker
sudo systemctl status containerd
```

### Step 5: Verify Installation

```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker compose version

# Verify Docker can pull images
docker pull alpine:latest
docker image ls
```

## Setting up n8n with Docker Compose

Now we'll create a comprehensive Docker Compose setup optimized for our HITL workflow requirements.

### Step 1: Create Project Structure

```bash
# Create main project directory
mkdir -p ~/n8n-hitl-deployment
cd ~/n8n-hitl-deployment

# Create organized directory structure
mkdir -p {data,config,logs,backups,scripts}
mkdir -p data/{n8n,postgres,redis}
mkdir -p config/{n8n,postgres,nginx}

# Set appropriate permissions
chmod 755 data data/{n8n,postgres,redis}
chmod 755 config config/{n8n,postgres,nginx}
```

### Step 2: Create Docker Compose Configuration

Create the comprehensive Docker Compose file:

```bash
nano docker-compose.yml
```

Add the following configuration:

```yaml
version: '3.8'

services:
  # PostgreSQL Database for n8n data persistence
  postgres:
    image: postgres:15-alpine
    container_name: n8n-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: n8n
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - ./data/postgres:/var/lib/postgresql/data
      - ./config/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
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

  # Redis for queue management and session storage
  redis:
    image: redis:7-alpine
    container_name: n8n-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
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

  # n8n Workflow Automation Platform
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
      DB_POSTGRESDB_PASSWORD: ${POSTGRES_PASSWORD}
      
      # Redis Configuration
      QUEUE_BULL_REDIS_HOST: redis
      QUEUE_BULL_REDIS_PORT: 6379
      QUEUE_BULL_REDIS_PASSWORD: ${REDIS_PASSWORD}
      
      # n8n Core Configuration
      N8N_HOST: ${N8N_HOST:-localhost}
      N8N_PORT: 5678
      N8N_PROTOCOL: ${N8N_PROTOCOL:-http}
      WEBHOOK_URL: ${WEBHOOK_URL:-http://localhost:5678}
      
      # Authentication & Security
      N8N_BASIC_AUTH_ACTIVE: true
      N8N_BASIC_AUTH_USER: ${N8N_USER}
      N8N_BASIC_AUTH_PASSWORD: ${N8N_PASSWORD}
      
      # Execution Configuration
      EXECUTIONS_PROCESS: main
      EXECUTIONS_DATA_SAVE_ON_ERROR: all
      EXECUTIONS_DATA_SAVE_ON_SUCCESS: all
      EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS: true
      EXECUTIONS_DATA_PRUNE: true
      EXECUTIONS_DATA_MAX_AGE: 168
      
      # AI and LangChain Configuration
      N8N_AI_ENABLED: true
      N8N_LANGCHAIN_ENABLED: true
      
      # Webhook Configuration
      N8N_PAYLOAD_SIZE_MAX: 16
      
      # Timezone and Localization
      GENERIC_TIMEZONE: ${TIMEZONE:-UTC}
      TZ: ${TIMEZONE:-UTC}
      
      # Logging Configuration
      N8N_LOG_LEVEL: info
      N8N_LOG_OUTPUT: console,file
      N8N_LOG_FILE_LOCATION: /home/node/.n8n/logs/
      
      # Security Settings
      N8N_SECURE_COOKIE: false
      N8N_COOKIES_SECURE: false
      
      # Performance Settings
      N8N_METRICS: true
      N8N_DIAGNOSTICS_ENABLED: false
      
    ports:
      - "${N8N_PORT:-5678}:5678"
    volumes:
      - ./data/n8n:/home/node/.n8n
      - ./config/n8n:/home/node/.n8n/config:ro
      - ./logs:/home/node/.n8n/logs
    networks:
      - n8n-network
    healthcheck:
      test: ["CMD-SHELL", "wget --quiet --tries=1 --spider http://localhost:5678/healthz || exit 1"]
      interval: 30s
      retries: 3
      start_period: 60s
      timeout: 10s

  # Optional: Nginx reverse proxy for production
  nginx:
    image: nginx:alpine
    container_name: n8n-nginx
    restart: unless-stopped
    depends_on:
      - n8n
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./config/nginx/ssl:/etc/nginx/ssl:ro
      - ./logs/nginx:/var/log/nginx
    networks:
      - n8n-network
    profiles:
      - production

networks:
  n8n-network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: 172.25.0.0/16

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  n8n_data:
    driver: local
```

### Step 3: Create Environment Configuration

Create a comprehensive environment variables file:

```bash
nano .env
```

Add the following configuration (customize values as needed):

```bash
# =============================================================================
# n8n HITL Workflow Environment Configuration
# =============================================================================

# Database Configuration
POSTGRES_PASSWORD=your_ultra_secure_postgres_password_2024

# Redis Configuration  
REDIS_PASSWORD=your_ultra_secure_redis_password_2024

# n8n Core Configuration
N8N_HOST=localhost
N8N_PORT=5678
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678
N8N_USER=admin
N8N_PASSWORD=your_ultra_secure_n8n_admin_password_2024

# Timezone Configuration (adjust to your local timezone)
TIMEZONE=America/New_York

# Optional: Production Configuration
# Uncomment and configure for production deployment
# N8N_HOST=your-domain.com
# N8N_PROTOCOL=https
# WEBHOOK_URL=https://your-domain.com

# =============================================================================
# API Keys and External Service Configuration
# =============================================================================
# These will be configured through the n8n interface, but can be set here
# for environment-specific deployments

# OpenRouter API Configuration
# OPENROUTER_API_KEY=your_openrouter_api_key_here

# Tavily Search API Configuration
# TAVILY_API_KEY=your_tavily_api_key_here

# Telegram Bot Configuration
# TELEGRAM_BOT_TOKEN=your_telegram_bot_token_here

# X/Twitter API Configuration (optional)
# TWITTER_API_KEY=your_twitter_api_key_here
# TWITTER_API_SECRET=your_twitter_api_secret_here
# TWITTER_ACCESS_TOKEN=your_twitter_access_token_here
# TWITTER_ACCESS_TOKEN_SECRET=your_twitter_access_token_secret_here

# =============================================================================
# Advanced Configuration
# =============================================================================

# Security Configuration
N8N_ENCRYPTION_KEY=your_encryption_key_for_credentials_here

# Performance Configuration
NODE_OPTIONS=--max_old_space_size=4096
```

### Step 4: Create Database Initialization Script

```bash
nano config/postgres/init.sql
```

Add the following SQL initialization script:

```sql
-- n8n HITL Workflow Database Initialization Script
-- This script sets up additional tables and configurations for the HITL workflow

-- Ensure proper encoding and collation
ALTER DATABASE n8n SET timezone TO 'UTC';

-- Create extension for UUID generation
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Create additional schema for workflow-specific data
CREATE SCHEMA IF NOT EXISTS hitl_data;

-- Grant permissions to n8n user
GRANT ALL PRIVILEGES ON SCHEMA hitl_data TO n8n;

-- Create workflow execution tracking table
CREATE TABLE IF NOT EXISTS hitl_data.workflow_executions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    workflow_name VARCHAR(255) NOT NULL,
    execution_id VARCHAR(255) UNIQUE NOT NULL,
    user_id VARCHAR(255),
    status VARCHAR(50) DEFAULT 'running',
    input_data JSONB,
    output_data JSONB,
    human_feedback TEXT,
    approval_status VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP
);

-- Create telegram interactions tracking table
CREATE TABLE IF NOT EXISTS hitl_data.telegram_interactions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    chat_id VARCHAR(255) NOT NULL,
    user_id VARCHAR(255),
    message_id VARCHAR(255),
    workflow_execution_id UUID REFERENCES hitl_data.workflow_executions(id),
    interaction_type VARCHAR(100), -- 'trigger', 'approval', 'feedback'
    message_text TEXT,
    response_text TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create content generation tracking table
CREATE TABLE IF NOT EXISTS hitl_data.generated_content (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    workflow_execution_id UUID REFERENCES hitl_data.workflow_executions(id),
    content_type VARCHAR(100), -- 'twitter_post', 'blog_article', etc.
    original_content TEXT,
    revised_content TEXT,
    revision_count INTEGER DEFAULT 0,
    search_query TEXT,
    search_results JSONB,
    ai_model_used VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes for better performance
CREATE INDEX idx_workflow_executions_status ON hitl_data.workflow_executions(status);
CREATE INDEX idx_workflow_executions_created_at ON hitl_data.workflow_executions(created_at);
CREATE INDEX idx_telegram_interactions_chat_id ON hitl_data.telegram_interactions(chat_id);
CREATE INDEX idx_telegram_interactions_created_at ON hitl_data.telegram_interactions(created_at);
CREATE INDEX idx_generated_content_workflow_id ON hitl_data.generated_content(workflow_execution_id);

-- Grant all permissions on tables to n8n user
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA hitl_data TO n8n;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA hitl_data TO n8n;

-- Set up automatic timestamp updates
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ language 'plpgsql';

-- Apply update triggers
CREATE TRIGGER update_workflow_executions_updated_at 
    BEFORE UPDATE ON hitl_data.workflow_executions 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_generated_content_updated_at 
    BEFORE UPDATE ON hitl_data.generated_content 
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();
```

### Step 5: Set File Permissions and Security

```bash
# Secure environment file
chmod 600 .env

# Set appropriate permissions for config files
chmod 644 config/postgres/init.sql
chmod 755 config/n8n config/postgres config/nginx

# Create log directories with proper permissions
mkdir -p logs/{n8n,nginx}
chmod 755 logs logs/{n8n,nginx}
```

## Configuring Environment Variables

### Step 1: Generate Secure Passwords

Generate strong, unique passwords for all services:

```bash
# Install password generation tool if not available
sudo apt install -y pwgen

# Generate secure passwords
echo "PostgreSQL Password: $(pwgen -s 32 1)"
echo "Redis Password: $(pwgen -s 32 1)" 
echo "n8n Admin Password: $(pwgen -s 24 1)"
echo "Encryption Key: $(pwgen -s 64 1)"
```

### Step 2: Update Environment File

```bash
nano .env
```

Replace the placeholder passwords with the generated secure passwords.

### Step 3: Create n8n Configuration File

```bash
nano config/n8n/config.json
```

Add advanced n8n configuration:

```json
{
  "database": {
    "type": "postgresdb",
    "postgresdb": {
      "host": "postgres",
      "port": 5432,
      "database": "n8n",
      "user": "n8n",
      "ssl": false
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
    "defaultName": "HITL Workflow",
    "onboardingFlowDisabled": false,
    "callerPolicyDefaultOption": "workflowsFromSameOwner"
  },
  "executions": {
    "saveDataOnError": "all",
    "saveDataOnSuccess": "all",
    "saveDataManualExecutions": true,
    "pruneData": true,
    "pruneDataMaxAge": 168,
    "pruneDataTimeout": 3600
  },
  "endpoints": {
    "webhooks": {
      "waitingWebhooks": true,
      "disableProductionWebhooksOnMainProcess": true
    }
  },
  "ai": {
    "enabled": true
  },
  "langchain": {
    "enabled": true
  }
}
```

## Setting up External Service Accounts

Before importing our workflow, we need to set up accounts and obtain API keys for all external services.

### Step 1: Telegram Bot Setup

**Create a Telegram Bot:**

1. Open Telegram and search for `@BotFather`
2. Start a conversation and send `/newbot`
3. Follow the prompts to create your bot:
   - Choose a name for your bot (e.g., "HITL Workflow Bot")
   - Choose a username ending in 'bot' (e.g., "hitl_workflow_bot")
4. Save the bot token provided by BotFather
5. Send `/setprivacy` to BotFather, select your bot, and choose "Disable" to allow the bot to see all messages

**Get Your Chat ID:**

1. Start a conversation with your new bot
2. Send any message to the bot
3. Visit: `https://api.telegram.org/bot{YOUR_BOT_TOKEN}/getUpdates`
4. Look for the "chat" object and note the "id" value

### Step 2: OpenRouter API Setup

**Create OpenRouter Account:**

1. Visit https://openrouter.ai/
2. Sign up for an account
3. Navigate to the API Keys section
4. Create a new API key
5. Note down your API key
6. Add credits to your account (minimum $5 recommended)

**Supported Models:**
- GPT-4.1 (OpenAI)
- Gemini 2.0 Flash (Google)
- Claude 3.5 Sonnet (Anthropic)
- And many others

### Step 3: Tavily Search API Setup

**Create Tavily Account:**

1. Visit https://tavily.com/
2. Sign up for an account
3. Navigate to the API section
4. Generate an API key
5. Note the free tier limits (typically 1000 searches/month)

### Step 4: X/Twitter API Setup (Optional)

**Create Twitter Developer Account:**

1. Visit https://developer.twitter.com/
2. Apply for a developer account
3. Create a new project/app
4. Generate API keys:
   - API Key
   - API Secret Key
   - Access Token
   - Access Token Secret
5. Ensure your app has write permissions

### Step 5: Document Your Credentials

Create a secure credentials file:

```bash
nano credentials.txt
```

```text
=== HITL Workflow Credentials ===

Telegram Bot:
- Bot Token: YOUR_TELEGRAM_BOT_TOKEN
- Chat ID: YOUR_CHAT_ID

OpenRouter:
- API Key: YOUR_OPENROUTER_API_KEY

Tavily:
- API Key: YOUR_TAVILY_API_KEY

Twitter (Optional):
- API Key: YOUR_TWITTER_API_KEY
- API Secret: YOUR_TWITTER_API_SECRET
- Access Token: YOUR_TWITTER_ACCESS_TOKEN
- Access Token Secret: YOUR_TWITTER_ACCESS_TOKEN_SECRET

Generated Passwords:
- PostgreSQL: [password from previous step]
- Redis: [password from previous step]
- n8n Admin: [password from previous step]
```

Secure the credentials file:

```bash
chmod 600 credentials.txt
```

## Importing and Configuring the HITL Workflow

### Step 1: Start the n8n Environment

Launch all services using Docker Compose:

```bash
# Start all services
docker compose up -d

# Monitor startup logs
docker compose logs -f

# Check service health
docker compose ps
```

Wait for all services to show as "healthy" or "running".

### Step 2: Access n8n Web Interface

1. Open your web browser
2. Navigate to `http://localhost:5678`
3. Enter your admin credentials:
   - Username: admin
   - Password: [your_n8n_admin_password]

### Step 3: Import the HITL Workflow

1. In the n8n interface, click "Add Workflow"
2. Click on the workflow settings (three dots in top right)
3. Select "Import from JSON"
4. Paste the complete HITL workflow JSON:

```json
{
  "name": "HITL Example Flows",
  "nodes": [
    {
      "parameters": {
        "updates": [
          "message"
        ],
        "additionalFields": {}
      },
      "type": "n8n-nodes-base.telegramTrigger",
      "typeVersion": 1.2,
      "position": [
        40,
        -140
      ],
      "id": "b5514931-b2e4-4820-9d28-b4ea566e2708",
      "name": "Telegram Trigger",
      "webhookId": "f9d2a169-fd6e-40f1-82f5-36afa714030d",
      "credentials": {
        "telegramApi": {
          "id": "9jQWan3cOz3tE62s",
          "name": "Telegram account 2"
        }
      }
    },
    {
      "parameters": {
        "toolDescription": "Use this tool to search the web. ",
        "method": "POST",
        "url": "https://api.tavily.com/search",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "{\n  \"query\": \"{searchTerm}\",\n  \"topic\": \"general\",\n  \"search_depth\": \"basic\",\n  \"chunks_per_source\": 3,\n  \"max_results\": 1,\n  \"time_range\": null,\n  \"days\": 7,\n  \"include_answer\": true,\n  \"include_raw_content\": false,\n  \"include_images\": false,\n  \"include_image_descriptions\": false,\n  \"include_domains\": [],\n  \"exclude_domains\": []\n}",
        "placeholderDefinitions": {
          "values": [
            {
              "name": "searchTerm",
              "description": "What the user is searching for. "
            }
          ]
        }
      },
      "type": "@n8n/n8n-nodes-langchain.toolHttpRequest",
      "typeVersion": 1.1,
      "position": [
        340,
        140
      ],
      "id": "7c6f31d8-41ad-48ab-a3b9-e057521edb8f",
      "name": "Tavily",
      "credentials": {
        "httpHeaderAuth": {
          "id": "1Gs5ooRQh4ZYMIK6",
          "name": "Tavily Credential"
        }
      }
    },
    {
      "parameters": {
        "operation": "sendAndWait",
        "chatId": "={{ $('Telegram Trigger').item.json.message.chat.id }}",
        "message": "=Good to go?\n\n{{ $json.output }}",
        "approvalOptions": {
          "values": {
            "approvalType": "double"
          }
        },
        "options": {
          "appendAttribution": false
        }
      },
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        580,
        -140
      ],
      "id": "566a19cb-be81-4302-af6a-6cf15412197e",
      "name": "Submit Approval",
      "webhookId": "da32f9b9-1b07-4e0f-9b5a-55a8c9ab9d59",
      "credentials": {
        "telegramApi": {
          "id": "9jQWan3cOz3tE62s",
          "name": "Telegram account 2"
        }
      }
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "938f9bd1-c97f-4aff-a17c-b5e83f83d639",
              "leftValue": "={{ $json.data.approved }}",
              "rightValue": true,
              "operator": {
                "type": "boolean",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        780,
        -140
      ],
      "id": "e0a7cbc4-223c-40be-8d79-b16b192f3e5f",
      "name": "If"
    },
    {
      "parameters": {
        "text": "={{ $('X_Post Agent').item.json.output }}",
        "additionalFields": {}
      },
      "type": "n8n-nodes-base.twitter",
      "typeVersion": 2,
      "position": [
        1040,
        -240
      ],
      "id": "9009d479-c91c-4e31-9857-dd42045af558",
      "name": "X",
      "credentials": {
        "twitterOAuth2Api": {
          "id": "rFbIQ4T5OtF02UIE",
          "name": "X account"
        }
      }
    },
    {
      "parameters": {
        "model": "openai/gpt-4.1",
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter",
      "typeVersion": 1,
      "position": [
        320,
        960
      ],
      "id": "56582801-61f0-4a9e-bfa0-bc9b4d2ffacb",
      "name": "GPT 4.1",
      "credentials": {
        "openRouterApi": {
          "id": "fpo6OUh9TcHg29jk",
          "name": "OpenRouter account"
        }
      }
    },
    {
      "parameters": {
        "model": "openai/gpt-4.1",
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter",
      "typeVersion": 1,
      "position": [
        60,
        140
      ],
      "id": "5f678d37-fdca-4c78-84a3-2af37b7306fa",
      "name": "GPT_4.1",
      "credentials": {
        "openRouterApi": {
          "id": "fpo6OUh9TcHg29jk",
          "name": "OpenRouter account"
        }
      }
    },
    {
      "parameters": {
        "updates": [
          "message"
        ],
        "additionalFields": {}
      },
      "type": "n8n-nodes-base.telegramTrigger",
      "typeVersion": 1.2,
      "position": [
        0,
        540
      ],
      "id": "d70fa860-edbf-4404-b5b4-7cd2b3d19ed3",
      "name": "Telegram_Trigger",
      "webhookId": "f9d2a169-fd6e-40f1-82f5-36afa714030d",
      "credentials": {
        "telegramApi": {
          "id": "9jQWan3cOz3tE62s",
          "name": "Telegram account 2"
        }
      }
    },
    {
      "parameters": {
        "operation": "sendAndWait",
        "chatId": "={{ $('Telegram_Trigger').item.json.message.chat.id }}",
        "message": "=Good to go?\n\n{{ $json.post }}",
        "responseType": "freeText",
        "options": {
          "appendAttribution": false
        }
      },
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        720,
        540
      ],
      "id": "2a987587-5633-405c-843a-ac2baf044af9",
      "name": "Request Feedback",
      "webhookId": "da32f9b9-1b07-4e0f-9b5a-55a8c9ab9d59",
      "credentials": {
        "telegramApi": {
          "id": "9jQWan3cOz3tE62s",
          "name": "Telegram account 2"
        }
      }
    },
    {
      "parameters": {
        "inputText": "={{ $json.data.text }}",
        "categories": {
          "categories": [
            {
              "category": "approved",
              "description": "=The post has been reviewed and accepted as-is. The human explicitly or implicitly expresses approval, indicating that no changes are needed. \n\nExample phrases include:\n\n\"Looks good.\"\n\"Go ahead and send it.\"\n\"This works for me.\"\n\"Approved.\"\n\"No changes needed.\"\n"
            },
            {
              "category": "declined",
              "description": "=The post has been reviewed, but the human requests modifications before it is sent like tweaks, removing parts, rewording, etc. This could include suggested edits, rewording, or major revisions. \n\nExample phrases include:\n\n\"Can we tweak this part?\"\n\"make it shorter\"\n\"change the source to 'American Kennel Club'\"\n\"more emojis\""
            }
          ]
        },
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.textClassifier",
      "typeVersion": 1,
      "position": [
        920,
        540
      ],
      "id": "70b9be21-fe2e-4436-b65b-18e8630f080b",
      "name": "Text Classifier"
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=Here is the post to revise: {{ $('Set Post').item.json.post }}\n\nHere is the human feedback: {{ $json.data.text }}",
        "options": {
          "systemMessage": "=# Overview\nYou are an expert twitter writer. Your job is to take an incoming post and revise it based on the feedback the human submitted.\n"
        }
      },
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        1320,
        680
      ],
      "id": "b9d7bccd-c306-4cb9-bbdd-104e54da46a2",
      "name": "Revision Agent"
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "454eb351-9781-48c2-b279-d5341210a1a1",
              "name": "post",
              "value": "={{ $json.output }}",
              "type": "string"
            }
          ]
        },
        "options": {}
      },
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [
        520,
        540
      ],
      "id": "306d0760-6955-4eb0-bef8-e31cc36e4ab1",
      "name": "Set Post"
    },
    {
      "parameters": {
        "toolDescription": "Use this tool to search the web. ",
        "method": "POST",
        "url": "https://api.tavily.com/search",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "{\n  \"query\": \"{searchTerm}\",\n  \"topic\": \"general\",\n  \"search_depth\": \"basic\",\n  \"chunks_per_source\": 3,\n  \"max_results\": 1,\n  \"time_range\": null,\n  \"days\": 7,\n  \"include_answer\": true,\n  \"include_raw_content\": false,\n  \"include_images\": false,\n  \"include_image_descriptions\": false,\n  \"include_domains\": [],\n  \"exclude_domains\": []\n}",
        "placeholderDefinitions": {
          "values": [
            {
              "name": "searchTerm",
              "description": "What the user is searching for. "
            }
          ]
        }
      },
      "type": "@n8n/n8n-nodes-langchain.toolHttpRequest",
      "typeVersion": 1.1,
      "position": [
        40,
        960
      ],
      "id": "e5d17efd-2fc4-4bf6-bada-e17f6b1e914a",
      "name": "Tavily Search",
      "credentials": {
        "httpHeaderAuth": {
          "id": "1Gs5ooRQh4ZYMIK6",
          "name": "Tavily Credential"
        }
      }
    },
    {
      "parameters": {
        "model": "google/gemini-2.0-flash-001",
        "options": {}
      },
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenRouter",
      "typeVersion": 1,
      "position": [
        480,
        960
      ],
      "id": "165151ea-4cfb-4bb3-ae73-eb41a4aa3bab",
      "name": "2.0 Flash",
      "credentials": {
        "openRouterApi": {
          "id": "fpo6OUh9TcHg29jk",
          "name": "OpenRouter account"
        }
      }
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "={{ $json.message.text }}",
        "options": {
          "systemMessage": "=# Overview  \nYou are an AI agent responsible for creating Twitter (X) posts based on user requests.  \n\n## Instructions  \n1. Always use the Tavily Search tool to find accurate, current information about the topic.  \n2. Write an informative, engaging tweet (up to 280 characters).  \n3. Include a brief reference to the source (e.g., \"via TechCrunch\", \"according to The Verge\") directly in the tweet.  \n4. Do not output anything except the final tweet.  \n\n## Tool\n- Tavily Search: Use this for real-time web search. Must be used every time before creating the post.\n\n## Example\n1) Input: \"Create a tweet about NASA's latest discovery.\"  \n2) Action: Search the web using Tavily Search.\n3) Output: \"NASA just found signs of ancient riverbeds on Mars—suggesting the Red Planet may have once been home to life. Huge leap in space exploration 🌌 (via NASA) #Mars #SpaceNews\"  \n\n## Final Notes  \n- Avoid clickbait or speculation—stick to factual and sourced content.  \n- Use hashtags or emojis only to enhance visibility.  \n- Keep language friendly, concise, and informative.  \n- Never use an \"@\" symbol when referencing the source.\n"
        }
      },
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.9,
      "position": [
        180,
        540
      ],
      "id": "616275e1-888e-42c1-9c67-f1f002c8ec0d",
      "name": "X Post Agent"
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "={{ $json.message.text }}",
        "options": {
          "systemMessage": "=# Overview  \nYou are an AI agent responsible for creating Twitter (X) posts based on user requests.  \n\n## Instructions  \n1. Always use the Tavily tool to find accurate, current information about the topic.  \n2. Write an informative, engaging tweet (up to 280 characters).  \n3. Include a brief reference to the source (e.g., \"via TechCrunch\", \"according to The Verge\") directly in the tweet.  \n4. Do not output anything except the final tweet.  \n\n## Tool\n- Tavily: Use this for real-time web search. Must be used every time before creating the post.\n\n## Example\n1) Input: \"Create a tweet about NASA's latest discovery.\"  \n2) Action: Search the web using Tavily.\n3) Output: \"NASA just found signs of ancient riverbeds on Mars—suggesting the Red Planet may have once been home to life. Huge leap in space exploration 🌌 (via NASA) #Mars #SpaceNews\"  \n\n## Final Notes  \n- Avoid clickbait or speculation—stick to factual and sourced content.  \n- Use hashtags or emojis only when relevant to enhance visibility.  \n- Keep language friendly, concise, and informative.  \n- Never use an \"@\" symbol when referencing the source.\n"
        }
      },
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.9,
      "position": [
        220,
        -140
      ],
      "id": "8d27919d-6515-4ccf-90d8-62c119010e71",
      "name": "X_Post Agent"
    },
    {
      "parameters": {
        "text": "={{ $('Set Post').item.json.post }}",
        "additionalFields": {}
      },
      "type": "n8n-nodes-base.twitter",
      "typeVersion": 2,
      "position": [
        1320,
        420
      ],
      "id": "8fc4db08-a0ce-4d39-ad80-73d6ad6abd53",
      "name": "X Post",
      "credentials": {
        "twitterOAuth2Api": {
          "id": "rFbIQ4T5OtF02UIE",
          "name": "X account"
        }
      }
    },
    {
      "parameters": {
        "chatId": "={{ $('Telegram Trigger').item.json.message.chat.id }}",
        "text": "=Post was denied. Please submit another request.",
        "additionalFields": {
          "appendAttribution": false
        }
      },
      "type": "n8n-nodes-base.telegram",
      "typeVersion": 1.2,
      "position": [
        1040,
        -40
      ],
      "id": "cf521176-0fe1-46a6-aa0a-45b2b82276a4",
      "name": "Denial message",
      "webhookId": "09d73d82-d0bf-416b-98bc-fc9a5c1b0c13",
      "credentials": {
        "telegramApi": {
          "id": "9jQWan3cOz3tE62s",
          "name": "Telegram account 2"
        }
      }
    }
  ],
  "pinData": {},
  "connections": {
    "Telegram Trigger": {
      "main": [
        [
          {
            "node": "X_Post Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Tavily": {
      "ai_tool": [
        [
          {
            "node": "X_Post Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    },
    "Submit Approval": {
      "main": [
        [
          {
            "node": "If",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "If": {
      "main": [
        [
          {
            "node": "X",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Denial message",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "GPT 4.1": {
      "ai_languageModel": [
        [
          {
            "node": "X Post Agent",
            "type": "ai_languageModel",
            "index": 0
          },
          {
            "node": "Revision Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "GPT_4.1": {
      "ai_languageModel": [
        [
          {
            "node": "X_Post Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Telegram_Trigger": {
      "main": [
        [
          {
            "node": "X Post Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Request Feedback": {
      "main": [
        [
          {
            "node": "Text Classifier",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Text Classifier": {
      "main": [
        [
          {
            "node": "X Post",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Revision Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Revision Agent": {
      "main": [
        [
          {
            "node": "Set Post",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Set Post": {
      "main": [
        [
          {
            "node": "Request Feedback",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Tavily Search": {
      "ai_tool": [
        [
          {
            "node": "X Post Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    },
    "2.0 Flash": {
      "ai_languageModel": [
        [
          {
            "node": "Text Classifier",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "X Post Agent": {
      "main": [
        [
          {
            "node": "Set Post",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "X_Post Agent": {
      "main": [
        [
          {
            "node": "Submit Approval",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "versionId": "5e53159c-b916-41e2-94e4-00107b5ad8d1",
  "meta": {
    "templateCredsSetupCompleted": true,
    "instanceId": "95e5a8c2e51c83e33b232ea792bbe3f063c094c33d9806a5565cb31759e1ad39"
  },
  "id": "VRi93B3lBaLzMSmf",
  "tags": []
}
```

5. Click "Import"
6. Save the workflow with the name "HITL Example Flows"

### Step 4: Understanding the Workflow Structure

The imported workflow contains two main flows:

**Top Flow - Simple Approval:**
- Telegram Trigger → X Post Agent → Submit Approval → If → X/Denial Message

**Bottom Flow - Advanced Feedback:**
- Telegram Trigger → X Post Agent → Set Post → Request Feedback → Text Classifier → X Post/Revision Agent

## Setting up Credentials and API Integrations

Now we'll configure all the necessary credentials for the workflow to function.

### Step 1: Configure Telegram API Credentials

1. In n8n, go to "Credentials" in the left sidebar
2. Click "Create Credential"
3. Search for and select "Telegram API"
4. Fill in the required fields:
   - **Access Token**: Your Telegram bot token from BotFather
   - **Name**: "Telegram account 2" (to match the workflow)
5. Test the credential and save

### Step 2: Configure OpenRouter API Credentials

1. Create a new credential
2. Search for and select "OpenRouter API"
3. Fill in the fields:
   - **API Key**: Your OpenRouter API key
   - **Name**: "OpenRouter account" (to match the workflow)
4. Test and save the credential

### Step 3: Configure Tavily Search Credentials

1. Create a new credential
2. Search for and select "HTTP Header Auth"
3. Fill in the fields:
   - **Name**: "Tavily Credential" (to match the workflow)
   - **Header Name**: "Authorization"
   - **Header Value**: `Bearer YOUR_TAVILY_API_KEY`
4. Save the credential

### Step 4: Configure Twitter/X API Credentials (Optional)

1. Create a new credential
2. Search for and select "Twitter OAuth2 API"
3. Fill in the OAuth2 fields:
   - **Client ID**: Your Twitter API Key
   - **Client Secret**: Your Twitter API Secret Key
   - **Name**: "X account" (to match the workflow)
4. Complete the OAuth2 flow
5. Save the credential

### Step 5: Update Workflow Nodes with Credentials

Go back to your workflow and verify each node has the correct credentials:

1. **Telegram Trigger nodes**: Should reference "Telegram account 2"
2. **OpenRouter language model nodes**: Should reference "OpenRouter account"
3. **Tavily tool nodes**: Should reference "Tavily Credential"
4. **Twitter nodes**: Should reference "X account"

### Step 6: Test Credential Connections

Test each credential by executing individual nodes:

1. Click on a node that uses credentials
2. Click "Test" or "Execute Node"
3. Verify the connection is successful
4. Repeat for all credential-dependent nodes

## Testing the Complete Workflow

### Step 1: Activate the Workflow

1. In the workflow editor, find the workflow activation toggle
2. Switch it from "Inactive" to "Active"
3. Verify both Telegram trigger nodes are now listening

### Step 2: Test Simple Approval Flow

1. Open Telegram and start a chat with your bot
2. Send a message like: "Create a tweet about the latest AI developments"
3. The workflow should:
   - Receive your message
   - Use Tavily to search for current AI news
   - Generate a tweet using GPT-4.1
   - Send you the tweet for approval via Telegram
4. Respond with "Yes" or "No" to approve or deny
5. If approved, the tweet should be posted to X/Twitter (if configured)

### Step 3: Test Advanced Feedback Flow

1. Send another message to your Telegram bot
2. The bot should generate a tweet and ask for feedback
3. Provide detailed feedback like: "Make it shorter and add more emojis"
4. The workflow should:
   - Classify your feedback as requiring revision
   - Use the Revision Agent to modify the content
   - Present the revised version for feedback
   - Continue the loop until you approve

### Step 4: Monitor Workflow Execution

1. In n8n, go to "Executions" in the left sidebar
2. Watch your test executions in real-time
3. Click on any execution to see detailed node-by-node results
4. Verify data flows correctly between nodes

### Step 5: Verify Database Logging

Check that execution data is being stored:

```bash
# Connect to PostgreSQL
docker compose exec postgres psql -U n8n -d n8n

# Check workflow executions
\c n8n
SELECT * FROM execution_entity ORDER BY "startedAt" DESC LIMIT 5;

# Check custom tracking tables
\c n8n
\dn
SELECT * FROM hitl_data.workflow_executions LIMIT 5;

\q
```

### Step 6: Test Error Handling

1. Temporarily disable one of the API credentials
2. Try running the workflow
3. Verify error messages are clear and helpful
4. Re-enable the credential and test again

## Security and Production Hardening

### Step 1: Implement SSL/TLS with Nginx

Create an Nginx configuration for production:

```bash
nano config/nginx/nginx.conf
```

```nginx
events {
    worker_connections 1024;
}

http {
    upstream n8n {
        server n8n:5678;
    }

    server {
        listen 80;
        server_name your-domain.com;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name your-domain.com;

        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;

        location / {
            proxy_pass http://n8n;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # WebSocket support for real-time features
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }

        # Security headers
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        add_header X-XSS-Protection "1; mode=block";
        add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
    }
}
```

### Step 2: Set up Let's Encrypt SSL Certificates

```bash
# Install Certbot
sudo apt install -y certbot

# Obtain SSL certificate
sudo certbot certonly --standalone -d your-domain.com

# Copy certificates to nginx config
sudo cp /etc/letsencrypt/live/your-domain.com/fullchain.pem config/nginx/ssl/cert.pem
sudo cp /etc/letsencrypt/live/your-domain.com/privkey.pem config/nginx/ssl/key.pem

# Set proper permissions
chmod 644 config/nginx/ssl/cert.pem
chmod 600 config/nginx/ssl/key.pem
```

### Step 3: Configure Firewall Rules

```bash
# Configure UFW firewall
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH
sudo ufw allow ssh

# Allow HTTP and HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable firewall
sudo ufw --force enable

# Check status
sudo ufw status verbose
```

### Step 4: Implement Resource Limits

Update the Docker Compose file with resource constraints:

```bash
nano docker-compose.yml
```

Add to each service:

```yaml
services:
  n8n:
    # ... existing configuration ...
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '2'
        reservations:
          memory: 2G
          cpus: '1'
    ulimits:
      nofile:
        soft: 65536
        hard: 65536
```

### Step 5: Set up Log Rotation

```bash
# Create logrotate configuration
sudo nano /etc/logrotate.d/n8n-docker
```

```
/home/*/n8n-hitl-deployment/logs/*.log {
    daily
    missingok
    rotate 14
    compress
    delaycompress
    notifempty
    create 644 root root
    postrotate
        docker compose -f /home/*/n8n-hitl-deployment/docker-compose.yml restart n8n
    endscript
}
```

### Step 6: Implement Monitoring Scripts

Create a health check script:

```bash
nano scripts/health-check.sh
```

```bash
#!/bin/bash

echo "=== n8n HITL Workflow Health Check ==="
echo "Timestamp: $(date)"
echo

# Check Docker services
echo "Docker Services Status:"
docker compose ps

echo
echo "Service Health Checks:"

# Check n8n health
if curl -f -s http://localhost:5678/healthz > /dev/null; then
    echo "✅ n8n: Healthy"
else
    echo "❌ n8n: Unhealthy"
fi

# Check PostgreSQL
if docker compose exec -T postgres pg_isready -U n8n > /dev/null; then
    echo "✅ PostgreSQL: Healthy"
else
    echo "❌ PostgreSQL: Unhealthy"
fi

# Check Redis
if docker compose exec -T redis redis-cli ping > /dev/null; then
    echo "✅ Redis: Healthy"
else
    echo "❌ Redis: Unhealthy"
fi

echo
echo "Resource Usage:"
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"

echo
echo "Disk Usage:"
du -sh data/*
```

Make it executable:

```bash
chmod +x scripts/health-check.sh
```

## Monitoring and Maintenance

### Step 1: Set up Automated Backups

Create a comprehensive backup script:

```bash
nano scripts/backup.sh
```

```bash
#!/bin/bash

BACKUP_DIR="$HOME/n8n-hitl-deployment/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="n8n_hitl_backup_$DATE"

echo "Starting backup: $BACKUP_NAME"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Stop containers for consistent backup
cd ~/n8n-hitl-deployment
echo "Stopping containers..."
docker compose stop

# Create database dump
echo "Backing up PostgreSQL database..."
docker compose run --rm postgres pg_dump -U n8n -h postgres n8n > "$BACKUP_DIR/${BACKUP_NAME}_db.sql"

# Create compressed backup of all data
echo "Creating compressed backup..."
tar -czf "$BACKUP_DIR/${BACKUP_NAME}.tar.gz" \
    --exclude=backups \
    --exclude=logs/*.log \
    data/ config/ docker-compose.yml .env scripts/

# Restart containers
echo "Restarting containers..."
docker compose up -d

# Cleanup old backups (keep last 7 days)
echo "Cleaning up old backups..."
find "$BACKUP_DIR" -name "n8n_hitl_backup_*.tar.gz" -mtime +7 -delete
find "$BACKUP_DIR" -name "n8n_hitl_backup_*_db.sql" -mtime +7 -delete

echo "Backup completed: $BACKUP_DIR/${BACKUP_NAME}.tar.gz"

# Log backup completion
echo "$(date): Backup completed successfully" >> logs/backup.log
```

Make it executable:

```bash
chmod +x scripts/backup.sh
```

### Step 2: Schedule Automated Tasks

```bash
# Edit crontab
crontab -e

# Add automated tasks
# Daily backup at 2 AM
0 2 * * * /home/$USER/n8n-hitl-deployment/scripts/backup.sh >> /home/$USER/n8n-hitl-deployment/logs/backup.log 2>&1

# Health check every 15 minutes
*/15 * * * * /home/$USER/n8n-hitl-deployment/scripts/health-check.sh >> /home/$USER/n8n-hitl-deployment/logs/health.log 2>&1

# Update containers weekly (Sunday at 3 AM)
0 3 * * 0 /home/$USER/n8n-hitl-deployment/scripts/update.sh >> /home/$USER/n8n-hitl-deployment/logs/update.log 2>&1
```

### Step 3: Create Update Script

```bash
nano scripts/update.sh
```

```bash
#!/bin/bash

echo "Starting n8n HITL system update: $(date)"

cd ~/n8n-hitl-deployment

# Create pre-update backup
echo "Creating pre-update backup..."
./scripts/backup.sh

# Pull latest images
echo "Pulling latest Docker images..."
docker compose pull

# Restart with new images
echo "Restarting services with updated images..."
docker compose up -d

# Cleanup old images
echo "Cleaning up old Docker images..."
docker image prune -f

# Wait for services to start
echo "Waiting for services to start..."
sleep 30

# Run health check
echo "Running post-update health check..."
./scripts/health-check.sh

echo "Update completed: $(date)"
```

Make it executable:

```bash
chmod +x scripts/update.sh
```

### Step 4: Set up Monitoring Dashboard

Create a simple monitoring dashboard:

```bash
nano scripts/dashboard.sh
```

```bash
#!/bin/bash

clear
echo "=================================================="
echo "n8n HITL Workflow Monitoring Dashboard"
echo "=================================================="
echo "Last Updated: $(date)"
echo

# System Information
echo "📊 SYSTEM OVERVIEW"
echo "------------------"
echo "Uptime: $(uptime -p)"
echo "Load Average: $(uptime | awk -F'load average:' '{print $2}')"
echo "Memory Usage: $(free -h | awk 'NR==2{printf "%.1f%%", $3*100/$2}')"
echo "Disk Usage: $(df -h / | awk 'NR==2{print $5}')"
echo

# Docker Services
echo "🐳 DOCKER SERVICES"
echo "------------------"
docker compose ps --format "table {{.Service}}\t{{.Status}}\t{{.Ports}}"
echo

# Recent Workflow Executions
echo "⚡ RECENT EXECUTIONS"
echo "------------------"
if docker compose exec -T postgres psql -U n8n -d n8n -c "SELECT id, mode, status, \"startedAt\" FROM execution_entity ORDER BY \"startedAt\" DESC LIMIT 5;" 2>/dev/null; then
    echo "Database query successful"
else
    echo "Unable to fetch execution data"
fi
echo

# Log Summary
echo "📋 LOG SUMMARY (Last 24h)"
echo "-------------------------"
echo "Total Executions: $(grep -c "Execution" logs/n8n/*.log 2>/dev/null || echo "N/A")"
echo "Errors: $(grep -c "ERROR" logs/n8n/*.log 2>/dev/null || echo "N/A")"
echo "Warnings: $(grep -c "WARN" logs/n8n/*.log 2>/dev/null || echo "N/A")"
echo

echo "Press Ctrl+C to exit"
sleep 30
exec $0
```

Make it executable:

```bash
chmod +x scripts/dashboard.sh
```

### Step 5: Configure Alerting

Create an alert script:

```bash
nano scripts/alert.sh
```

```bash
#!/bin/bash

# Configuration
ALERT_EMAIL="your-email@example.com"
WEBHOOK_URL="your-slack-webhook-url"

# Check if n8n is responding
if ! curl -f -s http://localhost:5678/healthz > /dev/null; then
    MESSAGE="🚨 ALERT: n8n service is not responding on $(hostname) at $(date)"
    
    # Send email alert (requires mailutils)
    if command -v mail > /dev/null; then
        echo "$MESSAGE" | mail -s "n8n Service Alert" "$ALERT_EMAIL"
    fi
    
    # Send Slack alert (if webhook configured)
    if [ ! -z "$WEBHOOK_URL" ]; then
        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"$MESSAGE\"}" \
            "$WEBHOOK_URL"
    fi
    
    # Log alert
    echo "$(date): $MESSAGE" >> logs/alerts.log
fi

# Check disk space
DISK_USAGE=$(df / | awk 'NR==2{print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -gt 80 ]; then
    MESSAGE="⚠️ WARNING: Disk usage is ${DISK_USAGE}% on $(hostname)"
    echo "$(date): $MESSAGE" >> logs/alerts.log
fi

# Check memory usage
MEMORY_USAGE=$(free | awk 'NR==2{printf "%.0f", $3*100/$2}')
if [ "$MEMORY_USAGE" -gt 90 ]; then
    MESSAGE="⚠️ WARNING: Memory usage is ${MEMORY_USAGE}% on $(hostname)"
    echo "$(date): $MESSAGE" >> logs/alerts.log
fi
```

Make it executable:

```bash
chmod +x scripts/alert.sh
```

Add to crontab for regular monitoring:

```bash
# Check every 5 minutes
*/5 * * * * /home/$USER/n8n-hitl-deployment/scripts/alert.sh
```

## Troubleshooting Common Issues

### Issue 1: Telegram Bot Not Responding

**Symptoms:**
- Messages sent to bot don't trigger workflow
- Telegram trigger nodes show as inactive

**Diagnosis:**
```bash
# Check workflow status
docker compose logs n8n | grep -i telegram

# Test bot token
curl -X GET "https://api.telegram.org/bot{YOUR_BOT_TOKEN}/getMe"

# Check webhook registration
curl -X GET "https://api.telegram.org/bot{YOUR_BOT_TOKEN}/getWebhookInfo"
```

**Solutions:**
1. Verify bot token in credentials
2. Ensure workflow is active
3. Check bot permissions with BotFather
4. Restart n8n container: `docker compose restart n8n`

### Issue 2: OpenRouter API Errors

**Symptoms:**
- AI nodes fail with authentication errors
- "Insufficient credits" messages

**Diagnosis:**
```bash
# Check OpenRouter API status
curl -H "Authorization: Bearer YOUR_API_KEY" \
     "https://openrouter.ai/api/v1/models"

# Check account balance
curl -H "Authorization: Bearer YOUR_API_KEY" \
     "https://openrouter.ai/api/v1/auth/key"
```

**Solutions:**
1. Verify API key in credentials
2. Check account balance and add credits
3. Ensure correct model names in workflow nodes
4. Check rate limits and usage quotas

### Issue 3: Tavily Search Failures

**Symptoms:**
- Web search tools fail
- "API key invalid" errors

**Diagnosis:**
```bash
# Test Tavily API directly
curl -X POST "https://api.tavily.com/search" \
     -H "Authorization: Bearer YOUR_TAVILY_API_KEY" \
     -H "Content-Type: application/json" \
     -d '{"query": "test search"}'
```

**Solutions:**
1. Verify API key format (Bearer token)
2. Check monthly usage limits
3. Ensure correct endpoint URL
4. Update credential configuration

### Issue 4: Database Connection Issues

**Symptoms:**
- n8n fails to start
- "Database connection failed" errors

**Diagnosis:**
```bash
# Check PostgreSQL status
docker compose logs postgres

# Test database connection
docker compose exec postgres psql -U n8n -d n8n -c "SELECT version();"

# Check database disk space
docker compose exec postgres df -h
```

**Solutions:**
1. Verify database credentials in .env file
2. Check PostgreSQL container health
3. Ensure sufficient disk space
4. Reset database if corrupted:
   ```bash
   docker compose down
   sudo rm -rf data/postgres/*
   docker compose up -d
   ```

### Issue 5: High Memory Usage

**Symptoms:**
- System becomes slow
- Containers getting killed
- Out of memory errors

**Diagnosis:**
```bash
# Check memory usage
docker stats --no-stream

# Check system memory
free -h

# Check for memory leaks
docker compose exec n8n ps aux --sort=-%mem | head -10
```

**Solutions:**
1. Add memory limits to docker-compose.yml
2. Increase system RAM
3. Configure execution data pruning:
   ```bash
   # In .env file
   EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
   EXECUTIONS_DATA_MAX_AGE=24
   ```
4. Restart containers periodically

### Issue 6: Workflow Execution Failures

**Symptoms:**
- Workflows start but don't complete
- Random node failures
- Timeout errors

**Diagnosis:**
```bash
# Check execution logs
docker compose logs n8n | grep -i error

# Check workflow execution history in n8n interface
# Look for specific error messages in failed executions

# Check system resources during execution
watch docker stats
```

**Solutions:**
1. Increase execution timeout in workflow settings
2. Add error handling nodes to workflow
3. Implement retry logic for external API calls
4. Check and fix credential configurations
5. Monitor and resolve resource constraints

## Advanced Configuration and Scaling

### Step 1: Multi-Instance Setup for High Availability

For production environments with high loads, set up multiple n8n instances:

```bash
nano docker-compose.ha.yml
```

```yaml
version: '3.8'

services:
  # Load Balancer
  nginx-lb:
    image: nginx:alpine
    container_name: n8n-load-balancer
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./config/nginx/load-balancer.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - n8n-main
      - n8n-worker-1
      - n8n-worker-2
    networks:
      - n8n-network

  # Main n8n instance (handles webhooks)
  n8n-main:
    extends:
      file: docker-compose.yml
      service: n8n
    container_name: n8n-main
    environment:
      N8N_WORKER_ID: main
      EXECUTIONS_PROCESS: main
    ports: []

  # Worker instances (handle queue processing)
  n8n-worker-1:
    extends:
      file: docker-compose.yml
      service: n8n
    container_name: n8n-worker-1
    environment:
      N8N_WORKER_ID: worker-1
      EXECUTIONS_PROCESS: queue
    ports: []

  n8n-worker-2:
    extends:
      file: docker-compose.yml
      service: n8n
    container_name: n8n-worker-2
    environment:
      N8N_WORKER_ID: worker-2
      EXECUTIONS_PROCESS: queue
    ports: []
```

### Step 2: Configure Load Balancer

```bash
nano config/nginx/load-balancer.conf
```

```nginx
events {
    worker_connections 1024;
}

http {
    upstream n8n_backend {
        least_conn;
        server n8n-main:5678 weight=3;
        server n8n-worker-1:5678 weight=1;
        server n8n-worker-2:5678 weight=1;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://n8n_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Health check
            health_check uri=/healthz;
        }
    }
}
```

### Step 3: Environment-Specific Configurations

Create different configurations for various environments:

```bash
# Development environment
cp .env .env.development
nano .env.development
```

```bash
# Development-specific settings
N8N_LOG_LEVEL=debug
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
EXECUTIONS_DATA_SAVE_ON_ERROR=all
N8N_DIAGNOSTICS_ENABLED=true
```

```bash
# Production environment
cp .env .env.production
nano .env.production
```

```bash
# Production-specific settings
N8N_LOG_LEVEL=warn
EXECUTIONS_DATA_SAVE_ON_SUCCESS=none
EXECUTIONS_DATA_SAVE_ON_ERROR=all
N8N_DIAGNOSTICS_ENABLED=false
N8N_SECURE_COOKIE=true
N8N_COOKIES_SECURE=true
```

### Step 4: Custom Node Development

If you need custom functionality, create custom nodes:

```bash
# Create custom nodes directory
mkdir -p custom-nodes/n8n-nodes-hitl-extensions
cd custom-nodes/n8n-nodes-hitl-extensions

# Initialize npm package
npm init -y

# Install development dependencies
npm install --save-dev @types/node n8n-workflow n8n-core typescript

# Create node structure
mkdir -p nodes credentials

# Create a custom HITL approval node
nano nodes/HitlApproval.node.ts
```

```typescript
import {
    IExecuteFunctions,
    INodeExecutionData,
    INodeType,
    INodeTypeDescription,
} from 'n8n-workflow';

export class HitlApproval implements INodeType {
    description: INodeTypeDescription = {
        displayName: 'HITL Approval',
        name: 'hitlApproval',
        group: ['trigger'],
        version: 1,
        description: 'Human-in-the-loop approval node with custom logic',
        defaults: {
            name: 'HITL Approval',
            color: '#772244',
        },
        inputs: ['main'],
        outputs: ['main'],
        properties: [
            {
                displayName: 'Approval Message',
                name: 'approvalMessage',
                type: 'string',
                default: 'Please review and approve:',
                description: 'Message to display for approval',
            },
        ],
    };

    async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
        const items = this.getInputData();
        const returnData: INodeExecutionData[] = [];

        for (let i = 0; i < items.length; i++) {
            const approvalMessage = this.getNodeParameter('approvalMessage', i) as string;
            
            // Custom approval logic here
            const newItem: INodeExecutionData = {
                json: {
                    ...items[i].json,
                    approvalMessage,
                    timestamp: new Date().toISOString(),
                },
            };

            returnData.push(newItem);
        }

        return [returnData];
    }
}
```

### Step 5: Performance Optimization

Optimize the deployment for better performance:

```bash
nano docker-compose.optimized.yml
```

```yaml
version: '3.8'

services:
  n8n:
    # ... existing configuration ...
    environment:
      # Performance optimizations
      NODE_OPTIONS: "--max_old_space_size=8192 --max-semi-space-size=128"
      UV_THREADPOOL_SIZE: 128
      
      # Execution optimizations
      EXECUTIONS_DATA_SAVE_MANUAL_EXECUTIONS: false
      EXECUTIONS_DATA_PRUNE: true
      EXECUTIONS_DATA_MAX_AGE: 72
      
      # Queue optimizations
      QUEUE_BULL_REDIS_DB: 0
      QUEUE_BULL_SETTINGS_STALLEDINTERVAL: 30
      QUEUE_BULL_SETTINGS_MAXSTALLEDCOUNT: 1

  postgres:
    # PostgreSQL optimizations
    command: >
      postgres
      -c shared_buffers=256MB
      -c effective_cache_size=1GB
      -c maintenance_work_mem=64MB
      -c checkpoint_completion_target=0.9
      -c wal_buffers=16MB
      -c default_statistics_target=100
      -c random_page_cost=1.1
      -c effective_io_concurrency=200

  redis:
    # Redis optimizations
    command: >
      redis-server
      --appendonly yes
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --tcp-keepalive 60
```

## Conclusion and Next Steps

Congratulations! You have successfully deployed a sophisticated Human-in-the-Loop AI workflow automation system using n8n on Ubuntu 24.04.01. Your deployment includes:

### What You've Accomplished

**Complete Infrastructure:**
- Containerized n8n deployment with PostgreSQL and Redis
- Secure credential management and API integrations
- Human-in-the-loop approval workflows with Telegram integration
- AI-powered content generation with web search capabilities
- Advanced feedback processing and content revision loops

**Production-Ready Features:**
- SSL/TLS encryption and security hardening
- Automated backups and monitoring
- Health checks and alerting systems
- Resource optimization and scaling capabilities
- Comprehensive logging and troubleshooting tools

**AI Integration Capabilities:**
- OpenRouter API integration for multiple AI models
- Tavily search for real-time web information
- Text classification for feedback processing
- Intelligent content revision based on human feedback

### Key Benefits Achieved

1. **Human Oversight**: AI automation with human quality control
2. **Flexibility**: Easy modification of approval processes and content generation
3. **Scalability**: Ready for production loads with load balancing capabilities
4. **Reliability**: Robust error handling and recovery mechanisms
5. **Security**: Enterprise-grade security implementations
6. **Maintainability**: Comprehensive monitoring and automated maintenance

### Next Steps for Enhancement

**Immediate Improvements:**
1. **Custom Workflows**: Create workflows for your specific business needs
2. **Additional Integrations**: Add more service integrations (Discord, Slack, Email)
3. **Advanced AI Models**: Experiment with different AI models through OpenRouter
4. **Multi-Language Support**: Extend workflows for international content

**Advanced Development:**
1. **Custom Nodes**: Develop organization-specific n8n nodes
2. **API Integrations**: Connect to internal systems and databases
3. **Workflow Templates**: Build a library of reusable workflow components
4. **Analytics Dashboard**: Implement advanced workflow analytics

**Scaling and Optimization:**
1. **Multi-Region Deployment**: Deploy across multiple geographic regions
2. **Kubernetes Migration**: Move to Kubernetes for enterprise scaling
3. **Advanced Monitoring**: Implement Prometheus, Grafana, and ELK stack
4. **Performance Tuning**: Optimize for specific workload patterns

### Best Practices to Remember

**Security:**
- Regularly update all containers and dependencies
- Rotate API keys and passwords periodically
- Monitor access logs and unusual activities
- Implement proper backup encryption

**Performance:**
- Monitor resource usage and scale accordingly
- Optimize database queries and indexing
- Implement proper caching strategies
- Regular performance testing and optimization

**Maintenance:**
- Keep comprehensive documentation updated
- Test backup and recovery procedures regularly
- Monitor execution logs for trends and issues
- Plan for capacity growth and scaling needs

### Community and Support Resources

**Official Resources:**
- **n8n Documentation**: https://docs.n8n.io/
- **n8n Community Forum**: https://community.n8n.io/
- **GitHub Repository**: https://github.com/n8n-io/n8n
- **YouTube Channel**: https://www.youtube.com/c/n8nio

**API Documentation:**
- **OpenRouter**: https://openrouter.ai/docs
- **Tavily**: https://docs.tavily.com/
- **Telegram Bot API**: https://core.telegram.org/bots/api
- **Twitter API**: https://developer.twitter.com/en/docs

**Learning Resources:**
- **Docker Documentation**: https://docs.docker.com/
- **PostgreSQL Documentation**: https://www.postgresql.org/docs/
- **Nginx Documentation**: https://nginx.org/en/docs/

### Final Notes

Your n8n HITL deployment represents a cutting-edge approach to AI automation that maintains human oversight and quality control. The workflows you've implemented demonstrate the power of combining AI efficiency with human intelligence and judgment.

The system is designed to grow with your needs, whether you're processing dozens or thousands of content pieces daily. The monitoring, backup, and scaling systems ensure reliable operation as your automation requirements expand.

Remember to:
- **Monitor regularly**: Keep an eye on system health and performance
- **Test thoroughly**: Always test workflows before putting them into production
- **Document changes**: Maintain clear documentation of customizations and configurations
- **Stay updated**: Keep up with n8n releases and security updates
- **Engage with the community**: Share your experiences and learn from others

Your deployment is now ready to handle sophisticated AI automation workflows with human oversight, providing the perfect balance of efficiency and quality control for your organization's needs. Happy automating!
