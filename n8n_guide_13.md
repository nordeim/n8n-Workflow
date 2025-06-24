# 1. Introduction to n8n & JSON Workflows

n8n (pronounced “n-eight-n”) is an open-source, node-based workflow automation tool. Instead of writing glue code, you visually connect “nodes” that can:

- Trigger on events (HTTP requests, CRON schedules, Webhooks, etc.)  
- Process or transform data (Set, Function, Code, etc.)  
- Call external APIs (HTTP Request, Twitter, Slack, etc.)  
- Branch, loop, and handle errors  

Under the hood, every workflow you build in the Editor is stored as a JSON document. Understanding this JSON schema lets you:

1. Version-control workflows  
2. Share or import/export workflows  
3. Auto-generate or modify workflows programmatically  

---

# 2. Anatomy of an n8n Workflow JSON

Every workflow JSON has these top-level keys:

- **nodes** (array): each node’s configuration  
- **connections** (object): how node outputs connect to other nodes  
- **active** (boolean): is the workflow enabled?  
- **settings** (object): workflow-level toggles (e.g. timezone, retry settings)  
- **credentials** (object): references to stored credentials  

Minimal skeleton:
```json
{
  "name": "My Workflow",
  "active": false,
  "nodes": [ /* … */ ],
  "connections": { /* … */ },
  "settings": { /* … */ },
  "credentials": { /* … */ }
}
```

---

# 3. A Complex, Real-World Example Workflow

Below is an example that:
1. Exposes an HTTP webhook  
2. Validates and transforms incoming JSON  
3. Conditionally calls one of two external APIs  
4. Loops over returned items to post to Slack  
5. Implements error-handling via a Catch node  

```jsonc
{
  "name": "Order Processing & Notification",
  "active": true,
  "nodes": [
    {
      "parameters": {
        "httpMethod": "POST",
        "path": "webhook/order",
        "options": {}
      },
      "name": "Webhook Trigger",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "values": {
          "string": [
            { "name": "orderId", "value": "={{$json[\"id\"]}}" },
            { "name": "items",   "value": "={{JSON.stringify($json.items)}}" }
          ]
        },
        "options": {}
      },
      "name": "Set Order Data",
      "type": "n8n-nodes-base.set",
      "typeVersion": 1,
      "position": [450, 300]
    },
    {
      "parameters": {
        "requestMethod": "GET",
        "url": "https://api.example.com/inventory/{{$node[\"Set Order Data\"].json[\"orderId\"]}}",
        "jsonParameters": true
      },
      "name": "Check Inventory",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 1,
      "position": [650, 200]
    },
    {
      "parameters": {
        "conditions": {
          "boolean": [
            {
              "value1": "={{$json[\"available\"]}}",
              "value2": true
            }
          ]
        }
      },
      "name": "If In Stock",
      "type": "n8n-nodes-base.if",
      "typeVersion": 1,
      "position": [850, 200]
    },
    {
      "parameters": {
        "requestMethod": "POST",
        "url": "https://api.shipping.com/create",
        "jsonParameters": true,
        "options": {},
        "bodyParametersJson": "={{{ name: $json[\"orderId\"], items: JSON.parse($json[\"items\"]) }}}"
      },
      "name": "Create Shipment",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 1,
      "position": [1050, 120]
    },
    {
      "parameters": {
        "functionCode": "items = JSON.parse($node[\"Set Order Data\"].json.items);\nreturn items.map(item => ({ json: item }));"
      },
      "name": "Split Items",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [850, 400]
    },
    {
      "parameters": {
        "text": "New order {{$node[\"Set Order Data\"].json.orderId}}: Item {{$json.name}} (Qty {{$json.qty}})"
      },
      "name": "Slack Notify",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 1,
      "position": [1050, 400],
      "credentials": {
        "slackApi": "Slack Dev Token"
      }
    },
    {
      "parameters": {
        "errorMessage": "Workflow Error!"
      },
      "name": "Catch Error",
      "type": "n8n-nodes-base.errorTrigger",
      "typeVersion": 1,
      "position": [650, 520]
    }
  ],
  "connections": {
    "Webhook Trigger": { "main": [[ { "node": "Set Order Data", "type": "main", "index": 0} ]] },
    "Set Order Data":  { "main": [[ { "node": "Check Inventory",   "type": "main", "index": 0} ]] },
    "Check Inventory": { "main": [[                                   
        { "node": "If In Stock", "type": "main", "index": 0 }
      ],[
        { "node": "Catch Error","type": "main","index": 0 }
    ]]},
    "If In Stock": {
      "main": [
        [ { "node": "Create Shipment", "type": "main", "index": 0 } ],    // true
        [ { "node": "Catch Error",    "type": "main", "index": 0 } ]     // false
      ]
    },
    "Create Shipment": {
      "main": [[ { "node": "Split Items", "type": "main", "index": 0 } ]]
    },
    "Split Items": {
      "main": [[ { "node": "Slack Notify", "type": "main", "index": 0 } ]]
    },
    "Catch Error": {
      "main": [[ /* optionally route to a notification node */ ]]
    }
  },
  "settings": {
    "timezone": "America/New_York"
  },
  "credentials": {
    "slackApi": { "id": "1", "name": "Slack Dev Token" }
  }
}
```

---

# 4. Breakdown of Every JSON Element

### 4.1 Workflow Metadata
- **name**: Human-friendly title.  
- **active**: `true` or `false` toggles execution.  
- **settings.timezone**: Applies to CRON triggers & date nodes.  

### 4.2 Nodes Array  
Each node object includes:
- **type**: `{package}-{nodeName}` (e.g. `n8n-nodes-base.httpRequest`)  
- **typeVersion**: Version of the node definition  
- **name**: Unique label in the Editor  
- **position**: `[x, y]` in UI canvas  
- **parameters**: Node-specific configuration (see below)  
- **credentials** *(optional)*: References a named credential  

#### 4.2.1 Webhook Trigger  
- `httpMethod`: GET/POST/PUT…  
- `path`: appended to your n8n base URL (e.g. `/webhook/order`)  
- `options`: header definitions, response codes  

Official Docs:  
https://docs.n8n.io/nodes/n8n-nodes-base.webhook/

#### 4.2.2 Set Node  
- `values`: New fields you want to set on the data object  
- Supports types `string`, `number`, `boolean`, `json`, `collection`  
- Use expressions with `={{…}}` to pull from `$json`  

Docs:  
https://docs.n8n.io/nodes/n8n-nodes-base.set/

#### 4.2.3 HTTP Request  
- `requestMethod`: GET/POST/etc  
- `url`: can include expressions: `={{…}}`  
- `jsonParameters`: true → allows raw JSON in “Body Parameters JSON”  
- `bodyParametersJson`: free-form JSON  

Docs:  
https://docs.n8n.io/nodes/n8n-nodes-base.httpRequest/

#### 4.2.4 If Node  
- `conditions`: group by `boolean`, `number`, `string`, `multiOptions`  
- Defines two output branches: `main[0]` (true), `main[1]` (false)  

Docs:  
https://docs.n8n.io/nodes/n8n-nodes-base.if/

#### 4.2.5 Function / Code Nodes  
- **Function**: JavaScript inlined, returns array of `{ json, binary }` items  
- Good for loops, transformations, splitting  

Docs:  
https://docs.n8n.io/nodes/n8n-nodes-base.function/

#### 4.2.6 Slack Node  
- Requires Slack API credentials  
- `text`, `channel`, `attachments`, etc.  

Docs & Setup:  
https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/

#### 4.2.7 Error Trigger  
- Runs on any node error  
- Useful to catch and notify admins  

Docs:  
https://docs.n8n.io/nodes/n8n-nodes-base.errorTrigger/

---

# 5. Connections Object

Connections define directed edges between nodes. Format:

```json
"connections": {
  "<Node Name>": {
    "main": [
      [ { "node": "<Target Node>", "type": "main", "index": 0 } ],
      [ /* second output branch for IF, etc. */ ]
    ]
  }
}
```

- Each top-level array corresponds to an output port  
- `index` is used when a node has multiple outputs  

---

# 6. Credential References

The `credentials` section at the bottom references stored credentials by name/id:

```json
"credentials": {
  "slackApi": { "id": "1", "name": "Slack Dev Token" }
}
```

Inside a node:

```json
"credentials": {
  "slackApi": "Slack Dev Token"
}
```

Credentials must be created in n8n’s UI (Settings → Credentials).

---

# 7. Tips to Customize & Extend

1. **Parameterize Paths & URLs**  
   Use workflow or node parameters (e.g. `$workflow.name`) in URLs.  
2. **Use Environment Variables**  
   Reference via `$env['VAR_NAME']` in expressions to avoid hard-coding secrets.  
3. **Modularize with Sub-Workflows**  
   Call other workflows via “Execute Workflow” node.  
4. **Error Handling**  
   Pair every critical node with its own Catch node → send Slack or email.  
5. **Batch Processing**  
   Use “SplitInBatches” to avoid rate-limit issues on API calls.  
6. **Reusability**  
   Export and store JSON snippets of frequently used node chains.  

---

# 8. Official References & Further Learning

1. **n8n Docs**  
   https://docs.n8n.io/  
2. **Community Workflows & Templates**  
   https://n8n.io/workflows  
3. **n8n YouTube Channel**  
   https://www.youtube.com/c/n8n-io  
4. **Blog Tutorials**  
   - FreeCodeCamp: “Automate Everything with n8n”  
   - Medium: “Advanced n8n Workflow Patterns”  
5. **GitHub Examples**  
   https://github.com/n8n-io/n8n/tree/master/examples  

---

# 9. Best Practices

- **Descriptive Node Names**: eases maintenance.  
- **Comment Blocks**: use the “Comment” node to annotate large workflows.  
- **Version Control**: store your JSON in Git.  
- **Testing Environments**: mirror prod and sandbox API keys.  
- **Monitoring**: enable “Save Execution” for critical workflows.  

---

# 10. Conclusion

You now have:

- A full, complex workflow in JSON  
- In-depth explanation of every element  
- Pointers to official docs, templates, and videos  
- Tips for customization, error handling, and best practices  

Next steps:

1. Import this JSON into your n8n (Menu → Import from File).  
2. Replace credential names with your own.  
3. Test each branch via the Execution Preview.  
4. Iterate: add more nodes (e.g., database writes, notifications).  

Happy automating! 🚀
