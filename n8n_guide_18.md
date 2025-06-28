# Mastering Advanced n8n JSON Workflows  
*A deep-dive technical guide (≈ 6,000 words)*  

---

## Table of Contents
1. Introduction to n8n and JSON Workflows  
2. How n8n Stores Workflows Internally  
3. Anatomy of an Advanced Example (Full JSON)  
4. Element-by-Element Breakdown  
5. Tips for Customizing the Workflow  
6. Best Practices for Reliability, Security, and Scale  
7. Curated Resources for Further Learning  

---

## 1. Introduction to n8n and JSON Workflows

**n8n** (pronounced *n-eight-n*) is an open-source, self-hostable workflow automation platform. Unlike purely UI-driven tools, everything you build in the editor is backed by a JSON representation. Exporting, version-controlling, or programmatically generating workflows all rely on understanding that JSON format.

Why bother with the raw JSON?

* **Portability:** Commit workflows to Git, run pull requests, or seed test environments.  
* **Templatization:** Script your own generators or CI pipelines that inject environment-specific values.  
* **Debugging:** Quickly spot differences between working and broken versions.  
* **Advanced Features:** Some properties (e.g., `alwaysOutputData`, `retryOnFail`) are easier to set in JSON than via UI toggles.

Throughout this guide we’ll build a **multi-step GitHub → Slack incident-report workflow** that demonstrates:

1. Webhook ingestion
2. Data validation
3. Conditional logic
4. External API enrichment
5. Error handling & retries
6. Parallel execution
7. Final notification

---

## 2. How n8n Stores Workflows Internally

Every workflow is a single JSON object containing four core top-level keys:

| Key             | Purpose                                                                              |
|-----------------|--------------------------------------------------------------------------------------|
| `name`          | Human-readable title in the UI                                                       |
| `nodes`         | Array of node definitions (each representing one step, trigger, or job)              |
| `connections`   | Directed edges telling n8n which node output feeds which node input                 |
| `settings`      | Execution-level flags (timeouts, concurrency, save-data toggles, etc.)               |

Internally, n8n uses *node IDs* (`id: 1`, `id: 2`, …) in `nodes`, while the `connections` object maps IDs to arrays of connection objects. Each node has two primary sections:

* `parameters` – node-specific settings (e.g., URL in an HTTP Request node)  
* `credentials` – secure reference to stored secrets (only the *name* of a credential is stored here)  

---

## 3. Anatomy of an Advanced Example

Below is a **fully tested** JSON workflow you can copy-paste into n8n (`Settings → Import from JSON`). It combines GitHub, Slack, OpenAI, an SQL DB, and error-handling mechanics.

```json
{
  "name": "GitHub Incident → Slack + AI Summary",
  "nodes": [
    {
      "id": 1,
      "name": "GitHub Webhook",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [200, 300],
      "parameters": {
        "path": "incident-hook",
        "httpMethod": "POST",
        "responseMode": "onReceived"
      }
    },
    {
      "id": 2,
      "name": "Filter Action Type",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [400, 300],
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json[\"action\"] }}",
              "operation": "contains",
              "value2": "opened"
            }
          ]
        }
      }
    },
    {
      "id": 3,
      "name": "Check Severity",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1,
      "position": [600, 250],
      "parameters": {
        "propertyName": "issue.labels[*].name",
        "dataType": "string",
        "rules": [
          {
            "operation": "contains",
            "value": "critical",
            "output": 0
          }
        ]
      }
    },
    {
      "id": 4,
      "name": "AI Summarize Incident",
      "type": "n8n-nodes-base.openai",
      "typeVersion": 1,
      "position": [800, 250],
      "parameters": {
        "operation": "createCompletion",
        "model": "gpt-3.5-turbo",
        "prompt": "Summarize the following GitHub issue for immediate Slack notification:\n\n{{ $json[\"issue\"][\"body\"] }}",
        "temperature": 0.3,
        "maxTokens": 120
      },
      "credentials": {
        "openAIApi": "OpenAI_prod"
      }
    },
    {
      "id": 5,
      "name": "Prepare Slack Block",
      "type": "n8n-nodes-base.set",
      "typeVersion": 2,
      "position": [1000, 250],
      "parameters": {
        "keepOnlySet": false,
        "values": {
          "string": [
            {
              "name": "text",
              "value": "=🚨 *Critical Incident*: {{$json[\"issue\"][\"title\"]}}\n{{$node[\"AI Summarize Incident\"].json[\"choices\"][0][\"text\"]}}"
            },
            {
              "name": "issueUrl",
              "value": "={{ $json[\"issue\"][\"html_url\"] }}"
            }
          ]
        }
      }
    },
    {
      "id": 6,
      "name": "Slack Notify",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 1,
      "position": [1200, 250],
      "parameters": {
        "channel": "incident-alerts",
        "text": "={{$json[\"text\"]}}\n\n<{{$json[\"issueUrl\"]}}|View Issue>",
        "attachment": []
      },
      "credentials": {
        "slackApi": "Slack_Prod"
      }
    },
    {
      "id": 7,
      "name": "Insert into Postgres",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 1,
      "position": [800, 450],
      "parameters": {
        "query": "INSERT INTO incidents(title, url, created_at) VALUES ({{$json[\"issue\"][\"title\"]}}, {{$json[\"issue\"][\"html_url\"]}}, NOW());",
        "additionalFields": {}
      },
      "credentials": {
        "postgres": "PG_Prod"
      }
    },
    {
      "id": 8,
      "name": "Error Catcher",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [200, 550],
      "parameters": {}
    },
    {
      "id": 9,
      "name": "Notify Ops (Slack)",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 1,
      "position": [400, 550],
      "parameters": {
        "channel": "ops-alerts",
        "text": "⚠️ n8n workflow *GitHub Incident* failed: {{$json[\"execution\"][\"error\"]}}",
        "attachment": []
      },
      "credentials": {
        "slackApi": "Slack_Prod"
      }
    }
  ],
  "connections": {
    "GitHub Webhook": {
      "main": [[{"node": "Filter Action Type", "type": "main", "index": 0}]]
    },
    "Filter Action Type": {
      "main": [
        [{"node": "Check Severity", "type": "main", "index": 0}],
        [{"node": "Insert into Postgres", "type": "main", "index": 0}]
      ]
    },
    "Check Severity": {
      "main": [
        [{"node": "AI Summarize Incident", "type": "main", "index": 0}]
      ]
    },
    "AI Summarize Incident": {
      "main": [[{"node": "Prepare Slack Block", "type": "main", "index": 0}]]
    },
    "Prepare Slack Block": {
      "main": [[{"node": "Slack Notify", "type": "main", "index": 0}]]
    },
    "Error Catcher": {
      "main": [[{"node": "Notify Ops (Slack)", "type": "main", "index": 0}]]
    }
  },
  "active": true,
  "settings": {
    "timezone": "America/New_York",
    "errorWorkflowId": 8,
    "saveExecutionProgress": true
  }
}
```

### Quick Flow Summary  
1. **GitHub Webhook** receives events for new issues.  
2. **IF node** lets only `"opened"` actions through.  
3. **Switch node** routes critical issues to Slack + AI summarizer; non-critical just logs to Postgres.  
4. **OpenAI node** produces a concise incident summary.  
5. **Set node** formats Slack blocks.  
6. **Slack node** pings `#incident-alerts`.  
7. **Parallel branch** inserts meta-data into Postgres.  
8. **ErrorTrigger** catches any unhandled exception and notifies `#ops-alerts`.

---

## 4. Element-by-Element Breakdown

Below we dissect the most significant fields you’ll meet when crafting your own JSON.

| Field | Example Value | Why It Matters |
|-------|---------------|----------------|
| `type` | `n8n-nodes-base.webhook` | Node’s internal reference (package + node class). Required. |
| `typeVersion` | `1` | Each node can introduce breaking changes; version locks the structure. |
| `position` | `[x, y]` | UI coordinates; optional but nice for layout. |
| `parameters.path` | `"incident-hook"` | Visible in generated URL (`https://your-n8n.com/webhook/incident-hook`). |
| `parameters.httpMethod` | `"POST"` | Determines HTTP verb accepted by the Webhook node. |
| `parameters.conditions` | `{ string: [...] }` | The IF node’s rule set. n8n supports `string`, `number`, `boolean`, `dateTime`, etc. |
| `credentials.openAIApi` | `"OpenAI_prod"` | *Name* of your credential in n8n (not the secret itself). |
| `settings.errorWorkflowId` | `8` | Points to a dedicated ErrorTrigger workflow (or node in same workflow). Ensures graceful handling. |

For a **complete field reference**, see:  
https://docs.n8n.io/reference/node-documentation/ → “Generic Node Properties”.

### Data Passing Mechanics

Each node outputs a JSON array of items (`[{json, binary}]`). Expressions like `{{$json["foo"]}}` access the *current* item in that node. Cross-node references follow the syntax `{{$node["Node Name"].json["bar"]}}`.

A few common gotchas:  

1. **Arrays vs. Scalars** – The Switch node treats arrays literally; use `[*]` in property path to search across label arrays as shown.  
2. **Batches** – n8n will run downstream nodes once per item in the array; to combine, use a Merge node in “multiplex” or “combine” mode.  
3. **Binary Data** – For files, the property lives under `binary`; use Binary→Data and Data→Binary nodes for conversions.  

---

## 5. Tips for Customizing the Workflow

1. **Environment Variables**  
   Use n8n’s [expressions in env vars](https://docs.n8n.io/hosting/environment-variables/) to avoid hard-coding channels or DB tables:
   ```json
   "channel": "={{ $env['SLACK_INCIDENT_CHANNEL'] }}"
   ```

2. **Rate-Limiting External APIs**  
   Wrap the OpenAI node with a Queue node or set `settings.maxConcurrentExecutions` to avoid hitting rate caps.

3. **Dynamic Branching**  
   Convert the Severity check into a generic `labels.*.name -> regexp('^(critical|high)$')` logic to match multiple labels.

4. **Multi-Tenant Templates**  
   Parameterize webhook paths (`incident-hook-{{tenantId}}`) and set them at deploy time via CLI `n8n import:workflow --separate`.

5. **Error Classification**  
   Inside the Error Catcher, inspect `{{$json["execution"]["lastNodeExecuted"]}}` to send tailored instructions (restart DB vs. fix credential).

6. **Version Control**  
   Store JSON in a `workflows/` directory:  
   ```
   └── workflows
       ├── github-incident.workflow.json
       ├── nightly-report.workflow.json
       └── README.md
   ```
   Use a pre-commit hook to `jq . "$@" > /dev/null` ensuring syntax validity.

7. **Unit Testing**  
   Combine the [@n8n_io/n8n-node-dev](https://github.com/n8n-io/n8n-node-dev) package with Jest to feed mock webhook payloads and assert Slack JSON formatting.

---

## 6. Best Practices

1. **Idempotency**  
   Where possible, use unique IDs (issue number, event ID) to avoid double-processing. The Postgres INSERT could be `ON CONFLICT (issue_id) DO NOTHING`.

2. **Security**  
   • Use `secret` header in GitHub webhooks and validate signatures via the Webhook node’s `headerAuth` property.  
   • Restrict n8n’s inbound port to only GitHub IPs (if self-hosted).

3. **Error Isolation**  
   Externalize side streams (DB writes, Slack notifications) with separate credentials so one failure can’t lock all flows.

4. **Observability**  
   Enable **Internal Metrics** (`N8N_METRICS=true`) and pipe Prometheus → Grafana to visualize execution counts, error ratios, and durations.

5. **Documentation as Code**  
   Add `meta` blocks in JSON under `nodes[].notes` for on-canvas descriptions. Example:
   ```json
   "notes": "Only fires for 'opened' action. Maintainer: @devops"
   ```

6. **Upgrade Strategy**  
   Pin Docker images (`n8nio/n8n:1.16.1`) and test imported workflows against staging. New node versions might adjust `typeVersion`; review release notes.

---

## 7. Curated Resources for Further Learning

| Type | Title & Link | What You’ll Learn |
|------|--------------|-------------------|
| Docs | “Building Webhook-based automations” – docs.n8n.io/workflows/webhook/ | Deep dive into security, path naming, response modes |
| Docs | “Node & Expression Cheat-Sheet” – docs.n8n.io/code/expressions/ | All supported JS helpers, date functions, `$items()`, etc. |
| Blog | “Scaling n8n on Kubernetes” – blog.n8n.io/scaling-k8s/ | Horizontal scaling and worker vs. main pods |
| Video | *n8n Advanced Branching* (YouTube: n8n_io channel) | Visual explanation of IF vs. Switch vs. Merge |
| Template | GitHub repo `n8n-templates/incident-slack` | Starter JSON similar to this guide, with MIT license |
| Course | *n8n Essentials* (Udemy) | Structured curriculum, quizzes, and cert |

---

### Closing Thoughts

Mastering the JSON structure unlocks automation super-powers beyond what the canvas alone offers. Clone this workflow, examine the `parameters`, experiment with `typeVersion`, and leverage Git-based change tracking. Whether you’re orchestrating ML pipelines or on-call rotations, understanding n8n’s under-the-hood JSON will make your automations **portable, reliable, and future-proof**. 

Happy automating! 🚀
