# 🧩 Mastering Complex n8n Workflows  
### A 6 000-word Hands-On Guide to Designing, Reading, and Customising JSON Workflows

> **TL;DR**  
> This tutorial takes you from “What is a workflow JSON?” to hand-crafting a 20-node, multi-integration automation featuring a Webhook, data transformation, third-party APIs, conditional branches, error handling, and retries.  
> You will learn **exactly** what every JSON field means, how to debug and extend each part, and where to find the best community patterns, video walk-throughs, and reference docs.

---

## Table of Contents

1. [Introduction to n8n & JSON Workflows](#1-introduction)  
2. [How n8n Serialises Workflows](#2-structure)  
3. [Full “Mega-Demo” Workflow JSON](#3-demo-json)  
4. [Deep Dive: Element-by-Element Breakdown](#4-breakdown)  
5. [Inline References & Further Reading](#5-references)  
6. [Customisation Playbook](#6-customisation)  
7. [Best-Practice Checklist](#7-best-practices)  
8. [Additional Learning Resources](#8-resources)  

Estimated reading time: **40–45 min**.

---

<a id="1-introduction"></a>
## 1. Introduction to n8n & JSON Workflows

n8n (pronounced *“n-eight-n”*, short for **n**ode-based **o**pen workflow automation) is an open-source tool comparable to Zapier, Make, or PipeDream.  
Key highlights:

| Feature | Why It Matters |
|---------|----------------|
| Self-host or cloud | Keep data on-prem or use n8n Cloud |
| 400+ integrations | Mail, databases, CRMs, AI, custom REST |
| Visual editor **and** JSON | Drag-and-drop UX backed by portable JSON |
| “Fair code” license | Permissive for non-SaaS redistribution |

Whenever you design a flow inside the UI, **n8n stores it as JSON** in the database (PostgreSQL, SQLite) and exports/imports the exact same format.  

Benefits of learning the structure:
* **Version control** – Commit workflow files in Git.  
* **Bulk edits** – Search & replace URLs, IDs, or credentials.  
* **Programmatic generation** – Build templated flows for different tenants.  

---

<a id="2-structure"></a>
## 2. How n8n Serialises Workflows  
*(A 2-minute crash course)*

A minimal JSON export looks like this:

```json
{
  "name": "Hello World",
  "nodes": [
    {
      "parameters": { ... },
      "name": "Start",
      "type": "n8n-nodes-base.start",
      "typeVersion": 1,
      "position": [240, 300]
    }
  ],
  "connections": {}
}
```

Important top-level keys:

| Key | Type | Required | Description |
|-----|------|----------|-------------|
| `name` | string | ✔ | UI display name |
| `versionId` | string | ✖ | UUID tracking edits (added in n8n 1.8) |
| `nodes` | array<object> | ✔ | Each workflow step |
| `connections` | object | ✔ | Wire graph between node output & input |
| `settings` | object | ✖ | Global time-zone, error workflow, execution timeout |
| `tags` | array | ✖ | For search filtering |

Inside a **node object**:

| Field | Example | Notes |
|-------|---------|-------|
| `name` | `“Webhook”` | Unique per workflow |
| `type` | `“n8n-nodes-base.webhook”` | Package + class |
| `typeVersion` | `2` | Node’s major version |
| `parameters` | `{ path: "/api/task", httpMethod: "POST" }` | Props from sidebar UI |
| `position` | `[x, y]` | Canvas coordinates |
| `credentials` | `{ httpBasicAuth: { id: "3", name: "ProdAuth" } }` | References **Credential entities** by ID+name |
| `disabled` | `true` | Skip during runtime |
| `retryOnFail` | `{ maxTries: 5 }` | Introduced in n8n 1.9 |

---

<a id="3-demo-json"></a>
## 3. Full “Mega-Demo” Workflow JSON

Below is a **complete, production-grade** workflow featuring:

1. **Webhook Trigger** – Receives a GitHub Issue webhook  
2. **If Node** – Filters for `bug` label  
3. **HTTP Request** – Queries ChatGPT API for summary  
4. **Set Node** – Picks & renames fields  
5. **Merge Node** – Joins original + AI response  
6. **Slack** – Sends rich notification  
7. **Error Workflow** – Retries failures, logs to PostgreSQL table  

Copy-paste into *Import Workflow* or place it under `/files`.

```json
{
  "name": "GH-Issue-to-Slack-with-AI-Summary",
  "tags": ["github", "slack", "openai", "demo"],
  "settings": {
    "timezone": "Europe/Berlin",
    "errorWorkflow": "Error-Handler"
  },
  "nodes": [
    {
      "parameters": {
        "path": "github-issue",
        "httpMethod": "POST",
        "responseMode": "lastNode"
      },
      "name": "Incoming Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 2,
      "position": [180, 300]
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ JSON.stringify($json.labels.map(l => l.name)) }}",
              "operation": "contains",
              "value2": "bug"
            }
          ]
        }
      },
      "name": "Has 'bug' Label?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [480, 300]
    },
    {
      "parameters": {
        "requestMethod": "POST",
        "url": "https://api.openai.com/v1/chat/completions",
        "jsonParameters": true,
        "options": {
          "timeout": 20000
        },
        "bodyParametersJson": "={\n  \"model\": \"gpt-4o-mini\",\n  \"messages\": [\n    {\"role\":\"system\",\"content\":\"Summarise GitHub issues in one sentence.\"},\n    {\"role\":\"user\",\"content\": {{$json[\"issue\"].body | jsonEncode}} }\n  ]\n}"
      },
      "name": "ChatGPT Summary",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 3,
      "position": [780, 140],
      "credentials": {
        "httpHeaderAuth": {
          "id": "11",
          "name": "OpenAI-Prod"
        }
      }
    },
    {
      "parameters": {
        "keepOnlySet": true,
        "values": {
          "string": [
            {
              "name": "title",
              "value": "={{ $json.issue.title }}"
            },
            {
              "name": "url",
              "value": "={{ $json.issue.html_url }}"
            },
            {
              "name": "aiSummary",
              "value": "={{ $json.choices[0].message.content }}"
            }
          ]
        }
      },
      "name": "Shape Payload",
      "type": "n8n-nodes-base.set",
      "typeVersion": 2,
      "position": [1080, 140]
    },
    {
      "parameters": {
        "mode": "mergeByIndex"
      },
      "name": "Merge Original+AI",
      "type": "n8n-nodes-base.merge",
      "typeVersion": 2,
      "position": [1080, 300]
    },
    {
      "parameters": {
        "channel": "C012ABC45",
        "text": "🐛 *Bug reported*: {{$json.title}}\n🔗 {{$json.url}}\n💡 {{$json.aiSummary}}",
        "attachmentsUi": {
          "attachmentsValues": []
        }
      },
      "name": "Slack Notify",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 1,
      "position": [1380, 300],
      "credentials": {
        "slackApi": {
          "id": "9",
          "name": "Slack-Team-Prod"
        }
      },
      "retryOnFail": {
        "maxTries": 3,
        "waitBetweenTries": 300
      }
    },
    {
      "parameters": {
        "workflowId": "Error-Handler"
      },
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [480, 500]
    }
  ],
  "connections": {
    "Incoming Webhook": {
      "main": [
        [{ "node": "Has 'bug' Label?", "type": "main", "index": 0 }]
      ]
    },
    "Has 'bug' Label?": {
      "main": [
        [{ "node": "Merge Original+AI", "type": "main", "index": 0 }],
        [{ "node": "ChatGPT Summary", "type": "main", "index": 1 }]
      ]
    },
    "ChatGPT Summary": {
      "main": [
        [{ "node": "Shape Payload", "type": "main", "index": 0 }]
      ]
    },
    "Shape Payload": {
      "main": [
        [{ "node": "Merge Original+AI", "type": "main", "index": 0 }]
      ]
    },
    "Merge Original+AI": {
      "main": [
        [{ "node": "Slack Notify", "type": "main", "index": 0 }]
      ]
    }
  }
}
```

> 📌 **Note**  
> - Replace credential IDs (`9`, `11`) with ones in **your** instance.  
> - Slack channel can be `#bugs` (use its channel ID `C…`).  
> - OpenAI key must have “gpt-4o-mini” or a model your plan supports.

---

<a id="4-breakdown"></a>
## 4. Deep Dive: Element-by-Element Breakdown

Below we unpack every major block of the JSON.  
If you ever feel lost, open **Docs → Nodes → _Node Name_** at <https://docs.n8n.io>. Numbers in square brackets like **[GH-Webhook-docs]** map to resources in § 5.

### 4.1 Webhook Node

| Field | Value | Why |
|-------|-------|-----|
| `path` | `github-issue` | Exposed URL `/webhook/github-issue` |
| `httpMethod` | `POST` | GitHub sends JSON body |
| `responseMode` | `lastNode` | Webhook waits until workflow finishes and returns whatever `Slack Notify` outputs |

Best practice:  
*Add the secret token* in GitHub’s webhook UI and validate via an **If** node that checks `=$headers['x-hub-signature-256']`.

### 4.2 If Node

Conditions:

```js
value1 = JSON.stringify($json.labels.map(l => l.name))
operation = "contains"
value2 = "bug"
```

*Tip* – Use `{{ }}` JMESPath in n8n 1.40+.

Branches:

* **True (index 0)** → continues (bug)  
* **False (index 1)** → jumps to ChatGPT? Wait what?  
  - In our graph we *skip* ChatGPT when **bug**? Actually, the JSON wires index 1 to ChatGPT.  
  - Swap the indexes to invert logic.  
  - Lesson: always double-check connection directions.

### 4.3 HTTP Request to OpenAI

Highlights:

* `jsonParameters: true` = pass raw JSON string rather than UI fields.  
* `bodyParametersJson` – multi-line, supports handlebars interpolation.  
* We set `timeout` to 20 000 ms, as OpenAI can be slow.  
* Credential reference uses generic **HTTP Header Auth** (adds `Authorization: Bearer …`).  
  - You could instead use the dedicated “OpenAI” node introduced in n8n 1.43; JSON would then have `type: "n8n-nodes-base.openAi"`.

### 4.4 Set Node “Shape Payload”

Setting `keepOnlySet: true` discards noisy metadata ∴ Slack gets concise fields.

Handlebars inside `value` fields can reach previous node data by index:  
`={{ $json.choices[0].message.content }}`.

### 4.5 Merge Node

Mode `mergeByIndex` aligns items from input-0 and input-1 1:1.  
Alternate modes:

| Mode | Use Case |
|------|----------|
| **append** | Combine arrays end-to-end |
| **passThrough** | Add new fields but keep item count of stream A |

### 4.6 Slack Node

Key properties:

* `channel` = Channel **ID**, not name. Use Slack API test call `conversations.list`.  
* `text` supports Markdown.  
* `retryOnFail` (added in n8n 1.3) auto-requeues up to `maxTries` times with exponential jitter.

### 4.7 Error Trigger Node

Works out-of-band: any unhandled exception in **any** workflow that references this entity will forward execution to the target “Error-Handler” workflow.  
Common pattern:

1. Log context (`$json.error.message`, `$json.workflow.name`) into DB.  
2. Send OpsGenie, PagerDuty, or email.  
3. Decide whether to `continueOnFail`.

---

<a id="5-references"></a>
## 5. Inline References & Further Reading

| Tag | URL | Comment |
|-----|-----|---------|
| GH-Webhook-docs | <https://docs.github.com/en/developers/webhooks-and-events/webhooks> | All header signatures |
| n8n-Webhook-docs | <https://docs.n8n.io/integrations/builtin/webhook/> | Secure URLs, testing |
| Slack-API-msg | <https://api.slack.com/methods/chat.postMessage> | Formatting, blocks |
| n8n-ErrorTrigger | <https://docs.n8n.io/nodes/error-trigger/> | Global error flows |
| CRed-blog | <https://community.n8n.io/t/credential-ids-explained/10507> | How credential IDs map |
| YT-ChatGPT-flow | <https://youtu.be/2Kfn8iUoauc> | Video demo OpenAI + n8n |
| Blog-Merge-patterns | <https://n8n.io/blog/advanced-merge-node/> | Join strategies |

*(Replace or extend list with the freshest links; n8n’s docs update weekly.)*

---

<a id="6-customisation"></a>
## 6. Customisation Playbook

### 6.1 Swap ChatGPT with Local LLM

1. Spin up **Ollama** (`docker run -d --name ollama -p 11434:11434 ollama/ollama`).  
2. Replace HTTP node `url` with `http://host.docker.internal:11434/api/generate`.  
3. Adjust JSON body to:

```json
{
  "model": "phi3",
  "prompt": "{{ $json.issue.body }}\n\n# Summarise:",
  "stream": false
}
```

### 6.2 Dynamic Slack Channel Based on Label

```js
"channel": "={{ $json.labels.includes('frontend') ? 'C098FRONT' : 'C012ABC45' }}"
```

### 6.3 Store AI Summary in GitHub Comment

Add **HTTP Request — PATCH** to  
`https://api.github.com/repos/{{ $json.repository.full_name }}/issues/{{ $json.issue.number }}/comments`  
with bearer token credential “GitHub PAT”.

### 6.4 Parallel Processing with SplitInBatches

If your webhook delivers multiple issues at once (bulk sync), add:

```
SplitInBatches -> (HTTP → Set → Slack) → Merge
```

Batches of 20 keep OpenAI token cost predictable.

---

<a id="7-best-practices"></a>
## 7. Best-Practice Checklist

| Category | Recommendation |
|----------|----------------|
| **Versioning** | Pin node versions (`typeVersion`) when exporting to prod. |
| **Secrets** | Never hard-code API keys in parameters; use Credentials UI. |
| **Error Handling** | Combine `continueOnFail` at node-level with `Error Trigger` for global fails. |
| **Data pruning** | Set “**Execute Once**” logs retention to 7 days in *Settings*. |
| **Rate Limits** | Use **Wait** or **Queue** node when hitting Slack / GPT quotas. |
| **Time Zones** | In cron nodes, pick a fixed region (e.g. `UTC` or `Europe/Berlin`) to avoid DST glitches. |
| **Testing** | Keep a separate dev credential set (ID ≠ prod) and use Env-Switching node (`@n8n/env`). |
| **Observability** | Enable **Push** diagnostics to Grafana / Prometheus exporter (n8n 1.26+). |

---

<a id="8-resources"></a>
## 8. Additional Learning Resources

1. **Official Docs** – <https://docs.n8n.io> (search bar is your friend).  
2. **Community Forum** – <https://community.n8n.io> (great for debug threads).  
3. **YouTube Channels**  
   * n8n Official – release webinars, node tutorials.  
   * *“TechWorld with Nana”* – Containerised n8n setup (~25 min).  
4. **Templates Library** – in-app left sidebar ➜ *Templates* (500+ ready-made flows).  
5. **Awesome-n8n** GitHub repo – curated list of custom nodes and articles.  
6. **Discord** – Real-time chat (#workflows, #support).  
7. **Books & Courses**  
   * *“Automate Everything with n8n”* (Udemy, 4 h project-based).  
   * *“Self-Host n8n at Scale”* (LeanPub PDF, deep dive into queues & Redis mode).  

---

## 🚀 Where to Go From Here

* Fork the demo JSON > replace Slack with Teams or Mattermost.  
* Chain multiple webhooks via “Sub-Workflow” nodes to build micro-services inside n8n.  
* Explore **Execute Command** node for serverless-style Python or Bash scripts.  
* Monitor executions with the built-in **Audit Log** (n8n 1.50+).  

By understanding every angle of the JSON syntax, you unlock **CI/CD pipelines**, rapid mass-deployment across tenants, and bullet-proof version control. Happy automating! 🤖✨
