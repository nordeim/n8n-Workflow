Below is a comprehensive, practical deep-dive into n8n’s JSON workflow format.  
It covers:

1. A quick primer on n8n & why the JSON format matters.  
2. Anatomy of a workflow file (nodes, connections, credentials, settings, meta).  
3. A full-blown “real-world” workflow in JSON (Cron ➜ HubSpot API ➜ loop ➜ transform ➜ Google Sheets ➜ Slack notification ➜ error handling).  
4. Line-by-line explanations of every JSON property, plus customization tips.  
5. Curated references to official docs, community templates, blog posts & YouTube tutorials for deeper learning.

The guide is intentionally long and rich in detail; feel free to skim headings of interest.

---

## 1  What is n8n & why JSON?

n8n (“nodemation”) is an open-source, self-hostable alternative to Zapier, Make or Pipedream.  
Under the hood, each visual workflow is saved as **JSON**—either in your database or when you hit “Download” in the UI. That JSON can be:

• Version-controlled (Git).  
• Templated (parameterized, duplicated, shared).  
• Generated or patched programmatically.

Hence understanding the syntax unlocks power features such as CICD deployments or mass-editing nodes.

Official docs JSON reference →  
https://docs.n8n.io/hosting/data/indexing-your-workflows/#workflow-json

---

## 2  Anatomy of a workflow file

High-level structure (TypeScript interface trimmed for clarity):

```ts
{
  "name": string,
  "nodes": Array<INode>,          // Every action or trigger
  "connections": IConnections,    // Graph edges
  "settings": ISettings,          // Execution, timezone, error WF id …
  "active": boolean,              // isEnabled?
  "tags": string[],
  "id": number | string,
  "createdAt": string,
  "updatedAt": string
}
```

Key building blocks:

| Section       | Purpose | Customization highlights |
|---------------|---------|--------------------------|
| `nodes`       | The heart of the workflow—each item in the canvas. | `parameters`, `type`, `position`, `credentials`, `name` |
| `connections` | Defines *which output* of *which node* routes to *which input* of another node. | Multiple outputs, multiple targets, always array-of-arrays |
| `settings`    | Global options such as timezone, error workflow, exec mode. | `timezone`, `saveExecutionProgress`, `errorWorkflow` |
| `credentials` | Reference (by name) to secrets stored in n8n credential store. | Node-local, never in root |
| Meta fields   | `id`, `tags`, timestamps etc. Helpful for CI pipelines. | |



---

## 3  Full complex workflow JSON example

Scenario:  

1. **Cron** runs daily 09:00 UTC.  
2. **HTTP Request** pulls all HubSpot contacts changed in the last 24 h.  
3. **SplitInBatches** loops through them (100 at a time).  
4. **Set** maps + cleans fields.  
5. **Google Sheets** appends a row per contact.  
6. **IF** checks Lifecycle Stage; if it switched to *Customer*, fire Slack.  
7. Any error at any stage triggers an **Error Trigger** that posts to Slack & stores the run in a Postgres table.

Copy-paste ready JSON (shortened IDs for readability—n8n will auto-regenerate on import):

```json
{
  "name": "Daily HubSpot → Sheets sync with Slack alerts",
  "active": true,
  "versionId": "1.0.0",
  "nodes": [
    {
      "parameters": {
        "cronExpression": "0 9 * * *",
        "timezone": "UTC"
      },
      "name": "Cron Trigger",
      "type": "n8n-nodes-base.cron",
      "typeVersion": 1,
      "position": [200, 300]
    },
    {
      "parameters": {
        "authentication": "headerAuth",
        "url": "=https://api.hubapi.com/crm/v3/objects/contacts?limit=100&archived=false&properties=firstname,lastname,email,lifecyclestage,lastmodifieddate&filter=lastmodifieddate__gte={{$moment().subtract(1,'day').toISOString()}}",
        "headerParametersUi": {
          "parameter": [
            {
              "name": "Authorization",
              "value": "={{'Bearer ' + $credentials.hubspotApiKey}}"
            }
          ]
        },
        "responseFormat": "json",
        "options": {
          "batchSize": 100
        }
      },
      "name": "Fetch Contacts",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 2,
      "position": [450, 300],
      "credentials": {
        "httpHeaderAuth": {
          "id": "hubspot-api-token",
          "name": "HubSpot API Key"
        }
      }
    },
    {
      "parameters": {
        "batchSize": 100
      },
      "name": "SplitInBatches",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 1,
      "position": [700, 300]
    },
    {
      "parameters": {
        "keepOnlySet": true,
        "values": {
          "string": [
            { "name": "First Name", "value": "={{$json.properties.firstname}}" },
            { "name": "Last Name", "value": "={{$json.properties.lastname}}" },
            { "name": "Email", "value": "={{$json.properties.email}}" },
            { "name": "Lifecycle Stage", "value": "={{$json.properties.lifecyclestage}}" },
            { "name": "Last Modified", "value": "={{$json.properties.lastmodifieddate}}" }
          ]
        }
      },
      "name": "Map Fields",
      "type": "n8n-nodes-base.set",
      "typeVersion": 2,
      "position": [950, 300]
    },
    {
      "parameters": {
        "operation": "append",
        "sheetId": "1Uxxxxxxxxxxxxxxxxxxxxx",
        "range": "Contacts!A:E",
        "valueInputMode": "USER_ENTERED",
        "options": {}
      },
      "name": "Append to Google Sheet",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 3,
      "position": [1200, 300],
      "credentials": {
        "googleApi": {
          "id": "google-service-account",
          "name": "Google Service Account"
        }
      }
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{$json[\"Lifecycle Stage\"]}}",
              "operation": "equal",
              "value2": "customer"
            }
          ]
        }
      },
      "name": "Is Customer?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [1450, 300]
    },
    {
      "parameters": {
        "channel": "#sales-alerts",
        "text": "*New Customer* :tada:\n{{$json[\"First Name\"]}} {{$json[\"Last Name\"]}} ({{$json.Email}})",
        "as_user": true
      },
      "name": "Send Slack Message",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 1,
      "position": [1700, 200],
      "credentials": {
        "slackApi": {
          "id": "slack-bot-token",
          "name": "Slack Bot"
        }
      }
    },
    {
      "parameters": {
        "workflowId": "error-handler-wf-id"
      },
      "name": "Error Trigger",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [200, 600]
    }
  ],
  "connections": {
    "Cron Trigger": {
      "main": [
        [
          {
            "node": "Fetch Contacts",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Fetch Contacts": {
      "main": [
        [
          {
            "node": "SplitInBatches",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "SplitInBatches": {
      "main": [
        [
          {
            "node": "Map Fields",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "SplitInBatches",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Map Fields": {
      "main": [
        [
          {
            "node": "Append to Google Sheet",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Append to Google Sheet": {
      "main": [
        [
          {
            "node": "Is Customer?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Is Customer?": {
      "main": [
        [
          {
            "node": "Send Slack Message",
            "type": "main",
            "index": 0
          }
        ],
        [
          null
        ]
      ]
    }
  },
  "settings": {
    "executionOrder": "parallel",
    "errorWorkflow": "error-handler-wf-id",
    "timezone": "UTC"
  },
  "tags": ["hubspot", "google-sheets", "slack", "daily-sync"],
  "id": "hubspot-sheet-sync-main",
  "createdAt": "2023-11-01T10:00:00.000Z",
  "updatedAt": "2023-11-10T12:00:00.000Z"
}
```

---

## 4  Node-by-node, property-by-property breakdown

Below a table summarises the top-level keys you’ll meet inside every `nodes[X]` object:

| Key             | Required | Example (from above) | Meaning / Customization notes |
|-----------------|----------|----------------------|--------------------------------|
| `name`          | ✔︎       | `"Cron Trigger"`     | Human-friendly. Use unique names for easier expressions, e.g. `{{$node["Cron Trigger"].json}}`. |
| `type`          | ✔︎       | `n8n-nodes-base.cron`| Identifies the internal class; determines which `parameters` schema is expected. |
| `typeVersion`   | ✔︎       | `2`                 | Node-version (changes on upgrades). Export + import across instances → keep versions aligned. |
| `position`      | ✔︎       | `[200,300]`          | XY co-ordinates in canvas. Purely cosmetic. |
| `parameters`    | ✔︎       | `{ "cronExpression": … }` | Each node type defines its own sub-schema (see docs). |
| `credentials`   | ◼︎ (if needed) | `{ "slackApi": { "id":"…","name":"…" } }` | References saved credentials **by name**; ID is internal. No secrets are stored in workflow JSON. |
| `notes`         | optional | `"Fetch only contacts modified last 24h"` | Comments shown in UI. |
| `disabled`      | optional | `true`               | Node will be skipped. |

Below we zoom in on special/interesting parameters per node:

### Cron Trigger

```json
{
  "cronExpression": "0 9 * * *", // 09:00 daily
  "timezone": "UTC",
  "offset": 0                     // (ms) delay after schedule before emitting
}
```
– Accepts either human CRON syntax or UI-assisted fields (`minutes`, `hours`, …).  
Docs → https://docs.n8n.io/integrations/builtin/trigger-nodes/cron/

### HTTP Request (HubSpot)

Important customization levers:

| Param              | Example | Why/How |
|--------------------|---------|---------|
| `authentication`   | `"headerAuth"` | Could also be `predefinedCredentialType` for built-in OAuth2 flows. |
| `url`              | Expression using `{{$moment()}}` to build dynamic query. |
| `headerParametersUi.parameter[]` | Sets `Authorization` header but *does not* store secret in workflow (we concatenate `Bearer ` + credential variable). |
| `responseFormat`   | `"json"` or `"string"` / `"file"` |
| `options.batchSize`| For pagination; n8n auto-loops “Fetch Next” when the next page URL is returned as `next` etc. |

Docs → https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.httpRequest/

### SplitInBatches

Keeps memory usage down on large arrays:

```json
{
  "batchSize": 100
}
```

Loop behaviour:  
• Output 0 → next batch item  
• Output 1 → if you connect **back into itself**, creates a while-loop (see connection above).  
Docs → https://docs.n8n.io/integrations/builtin/utility-nodes/#split-in-batches

### Set (Map Fields)

Key flag: `"keepOnlySet": true` removes every property except what you explicitly map—handy before exporting to Sheets.  
Each value is an expression; you can transform with `$jmesPath()` or native JS.

### Google Sheets

Operations: `append`, `update`, `lookup`, `delete`.  
When using Service Account credentials, set spreadsheet sharing to that SA email.  
`valueInputMode` `"USER_ENTERED"` respects on-sheet formatting & formulas; `"RAW"` bypasses formatting.

Docs → https://docs.n8n.io/integrations/n8n-nodes-base.googleSheets/

### IF

`conditions.string[]` (there are also `boolean`, `number`, `date`) – every array element is ANDed within each group; groups across types are ANDed as well.  
By default: Output 0 = “true” branch, Output 1 = “false”.

Docs → https://docs.n8n.io/integrations/builtin/logic-nodes/if/

### Slack

`channel` can be channel ID or name (`#general`).  
`text` field supports Slack mrkdwn.  
Send as bot user vs legacy token user determined by credentials.

Docs → https://docs.n8n.io/integrations/n8n-nodes-base.slack/

### Error Trigger

When any node errors and no local “Continue On Fail” is set, n8n jumps to the referenced workflow id.

Good pattern: Keep an “Error Handler” WF that receives execution data via JSON, stores it in Postgres & pings a Slack channel.

Docs → https://docs.n8n.io/integrations/builtin/trigger-nodes/error-trigger/

---

## 5  Understanding `connections`

`connections` is a dictionary keyed by *source node name*.  
Each key’s `main` array contains **one array per output index**.

Example:

```json
"SplitInBatches": {
  "main": [
    [ { "node": "Map Fields", "type": "main", "index": 0 } ],
    [ { "node": "SplitInBatches", "type": "main", "index": 0 } ]
  ]
}
```

• `main[0]` → output 0 (the default “next item”)  
• `main[1]` → output 1 (loop-back for “continue”)

Other channels:

| Channel name   | Meaning |
|----------------|---------|
| `main`         | Standard data path |
| `timeout`      | Where to route when node has timeout (HTTP Request supports) |
| `error`        | Node-local error path (rare, mostly not used) |

---

## 6  Workflow-level `settings`

Common flags:

| Key                    | Values / Default | Notes |
|------------------------|------------------|-------|
| `timezone`             | `"America/New_York"` etc. | Overrides server default. |
| `executionOrder`       | `"parallel"` or `"serial"` | Serial = step-by-step inside same workflow run. |
| `saveExecutionProgress`| `true/false` | Saves after each node; heavier DB write load but lets you resume after crash. |
| `errorWorkflow`        | `"xyz"` | Fallback workflow id (see Error Trigger). |

---

## 7  Customisation & scaling tips

1. **Parameterize with environment variables**  
   In expressions you can access `{{$env.MY_VAR}}`. Use for sheet IDs, Slack channels across dev/prod.

2. **Use [Expressions → Global Functions](https://docs.n8n.io/code/expressions/functions/globals/)**  
   e.g. `{{$moment().tz('America/Los_Angeles').format()}}`.

3. **Modularise**  
   When flows balloon, split helper logic into *Sub-Workflows* and call via **Execute Workflow** node.

4. **Version-control**  
   Store JSON exports in Git. Commit each change, diff easily.

5. **Unit test via CLI**  
   `n8n execute --id <workflowId> --input='[{"foo":"bar"}]' --output=json`.

6. **Error isolation**  
   For nodes like Sheets where API limits may stall, set `continueOnFail: true` (checkbox in UI) then handle failures downstream.

7. **Bulk credentials rotation**  
   Because the JSON only references credential **names** you can rotate actual token secrets centrally without touching workflows.

8. **UI comment blocks**  
   The **Annotation** node is zero-logic but helps team onboarding.

---

## 8  Learning resources & templates

| Resource | What you’ll find | Link |
|----------|------------------|------|
| Official Docs | Node references, expression grammar, self-hosting guides. | https://docs.n8n.io/ |
| n8n Workflows Gallery | 400+ community-shared templates (search HubSpot, Sheets). | https://n8n.io/workflows |
| n8n YouTube | Tutorials incl. *“Mastering SplitInBatches”* & *“Error Handling Patterns”*. | https://www.youtube.com/@n8n_io |
| @ivov’s Blog | Deep dives on looping & cursor-based pagination. | https://medium.com/tag/n8n |
| Alex-P-Tech channel | Hands-on builds (Google Sheets automations). | https://www.youtube.com/c/AlexPTech |
| Community Forum | Ask Qs, paste JSON, get peer fixes. | https://community.n8n.io |
| Awesome-n8n GitHub | Curated list of libs, boilerplates, Docker tips. | https://github.com/jaeseopark/awesome-n8n |

Template sites worth scraping for ideas:  
• https://flows.n8n.cloud (unofficial)  
• https://automatesthings.com/tag/n8n/

---

## 9  Putting it all together: Checklist

☑️ Map every real-world step to a node.  
☑️ Test each node standalone (blue “play” in UI).  
☑️ Use expressions not hard-coded IDs for things like dates.  
☑️ Save + commit downloaded JSON to Git *before* major edits.  
☑️ Build an “Error Trigger” workflow once; reuse across dozens of WFs.  
☑️ Periodically upgrade n8n and run **“Update all nodes to latest typeVersion”** from the UI.

With these practices—and the fully annotated example above—you now have a production-grade template for daily data syncs, alerting and robust failure handling.

Happy automating! 🚀
