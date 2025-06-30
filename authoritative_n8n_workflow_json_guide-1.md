# n8n Workflow JSON Guide

Table of Contents  
1. [Introduction](#introduction)  
2. [Overall JSON Structure](#overall-json-structure)  
3. [Nodes](#nodes)  
4. [Connections](#connections)  
5. [Credentials](#credentials)  
6. [Example: A Simple HTTP → Function Workflow](#example-simple-http-function-workflow)  
7. [Parameterizing & Expressions](#parameterizing--expressions)  
8. [Advanced Patterns](#advanced-patterns)  
9. [Versioning & Metadata](#versioning--metadata)  
10. [Common Pitfalls & Troubleshooting](#common-pitfalls--troubleshooting)  
11. [Best Practices](#best-practices)  
12. [Real-World Workflows](#real-world-workflows)  
13. [Further Resources](#further-resources)  

---

## 1. Introduction

n8n is a “fair-code” workflow automation tool. Under the hood, every workflow is a JSON object describing:

- **Nodes** (the tasks or actions)
- **Connections** (how data flows)
- **Credentials** (secure auth details)
- **Metadata** (name, version, active status)

Editing JSON directly lets you:
- Automate bulk creation or modification.
- Share templates.
- Version workflows in Git.

---

## 2. Overall JSON Structure

Every exported n8n workflow follows this high-level schema:
```json
{
  "name": "Workflow Name",
  "nodes": [
    /* Node definitions */
  ],
  "connections": {
    /* Graph edges */
  },
  "active": false,
  "settings": { /* Optional UI settings */ },
  "id": 1,                  /* Internal ID */
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-02T12:00:00.000Z",
  "tags": []
}
```

- **name**: Human-readable name.  
- **nodes**: Array of node objects (see §3).  
- **connections**: Dict mapping `nodeName.outputIndex[] → nextNode.inputIndex[]`.  
- **active**: Boolean: whether the workflow is enabled.  
- **settings**: Layout/UI hints (usually optional).  

---

## 3. Nodes

Each node has this basic schema:
```json
{
  "parameters": { /* Node-specific settings */ },
  "name": "MyNode",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 2,
  "position": [250, 300],
  "credentials": {
    "httpBasicAuth": {
      "id": "8",
      "name": "My Basic Creds"
    }
  }
}
```

Fields:

- **parameters**: A key/value object. Keys vary by node type.  
- **name**: Unique in the workflow.  
- **type**: `<package>.<nodeName>` (see n8n docs).  
- **typeVersion**: Node major version.  
- **position**: `[x, y]` coordinates in the editor.  
- **credentials**: (Optional) object of credential references.  

### Common Node Types

| Category      | Type                           | Example type key           |
|---------------|--------------------------------|----------------------------|
| Trigger       | Webhook                        | n8n-nodes-base.webhook    |
| HTTP          | HTTP Request                   | n8n-nodes-base.httpRequest|
| Logic/Control | If, Switch, Merge              | n8n-nodes-base.if         |
| Code          | Function, Function Item        | n8n-nodes-base.function   |
| Services      | Slack, GitHub, MySQL, AWS      | n8n-nodes-base.slack      |

---

## 4. Connections

Connections describe directed edges:

```json
"connections": {
  "StartNode": {
    "main": [
      [
        { "node": "NextNode", "type": "main", "index": 0 }
      ]
    ]
  }
}
```

- **Root key**: the node name sending data.  
- **main**: primary output port. Many nodes only have `main`.  
- **index**: port index (0-based).  
- **node**: receiving node’s `name`.  
- **type**: input port (`main` or others like `conditional`).  

To chain multiple nodes, just list multiple targets in the array.

---

## 5. Credentials

Credentials live outside the workflow JSON in the database/secrets, but workflows reference them:

```json
"credentials": {
  "httpHeaderAuth": {
    "id": "5",
    "name": "HeaderAuthCreds"
  }
}
```

- **id**: internal n8n-credential ID.  
- **name**: human label.  

You must first create credentials via the UI or API, then reference them here. No secrets leak in the workflow export.

---

## 6. Example: A Simple HTTP → Function Workflow

End-to-end example:

```json
{
  "name": "Example: HTTP to Function",
  "nodes": [
    {
      "parameters": {
        "httpMethod": "GET",
        "path": "hello"
      },
      "name": "Webhook1",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [250, 300]
    },
    {
      "parameters": {
        "functionCode": "return [{ json: { greeting: 'Hello, World!' } }];"
      },
      "name": "Function1",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [450, 300]
    },
    {
      "parameters": {
        "responseCode": 200,
        "responseMode": "lastNode",
        "options": {}
      },
      "name": "WebhookResponse1",
      "type": "n8n-nodes-base.webhook-response",
      "typeVersion": 1,
      "position": [650, 300]
    }
  ],
  "connections": {
    "Webhook1": {
      "main": [
        [
          { "node": "Function1", "type": "main", "index": 0 }
        ]
      ]
    },
    "Function1": {
      "main": [
        [
          { "node": "WebhookResponse1", "type": "main", "index": 0 }
        ]
      ]
    }
  },
  "active": false,
  "settings": {},
  "id": 10,
  "createdAt": "2024-01-01T00:00:00.000Z",
  "updatedAt": "2024-01-01T00:01:00.000Z",
  "tags": []
}
```

How it works:

1. **Webhook1** listens at `/webhook/hello`.  
2. **Function1** returns a JSON greeting.  
3. **WebhookResponse1** sends it back.

---

## 7. Parameterizing & Expressions

n8n supports expressions: `{{$node["PreviousNode"].json["fieldName"]}}`.  
Example using expressions in HTTP Request:

```json
"parameters": {
  "url": "https://api.example.com/users/{{$node[\"SetUserID\"].json[\"userId\"]}}",
  "options": {},
  "jsonParameters": false
}
```

- Wrap JS inside `{{ }}`.  
- Access previous node data via `$node["NodeName"].json`.  
- Use `$workflow` for globals or `$now` for dates.  

---

## 8. Advanced Patterns

1. **SplitInBatches**: process large arrays.  
   - Node type: `n8n-nodes-base.splitInBatches`.  
   - Parameters: `batchSize`.  

2. **If / Switch**: branch logic.  
   - `n8n-nodes-base.if` uses `.conditional` port for “false” path.  
   - Example connection object:

     ```json
     "connections": {
       "If1": {
         "main": [[{"node":"NextOnTrue","type":"main","index":0}]],
         "conditional": [[{"node":"NextOnFalse","type":"main","index":0}]]
       }
     }
     ```

3. **Error Workflows**  
   - Use the “Error Trigger” to catch and handle errors globally.

4. **Looping**  
   - Use a Function node to push/pop from an array, then route via SplitInBatches.

5. **Composite JSON**  
   - Build complex payloads in Set or Function nodes using expressions or JavaScript.

---

## 9. Versioning & Metadata

- Use `active: true` for production flows.  
- Maintain `id` and timestamps if syncing with an existing n8n instance.  
- Use the UI’s “Save as new version” to bump typeVersion where needed.

---

## 10. Common Pitfalls & Troubleshooting

• **Missing Connections** → Workflow shows “no data produced”.  
• **Mismatched Credentials** → Node errors: “No credentials found”.  
• **Invalid Expressions** → Check syntax: run with Debug mode.  
• **TypeVersion Mismatch** → Update node version when upgrading n8n.  
• **Position Collisions** → Overlapping nodes in the UI; adjust `position`.

---

## 11. Best Practices

1. **Name Nodes Clearly**  
   - E.g. “HTTP Fetch Users”, “Set Payload”—self-documenting.  
2. **Use Comments in Functions**  
   - Include reasoning for complex JS.  
3. **Group by Tags**  
   - Leverage `tags` to categorize high-level workflows.  
4. **Modularize**  
   - Use “sub-workflows” (via Execute Workflow node) for reusability.  
5. **Validate JSON**  
   - Before import, always run through a JSON linter.  

---

## 12. Real-World Workflows

### 12.1 Slack Notification on New GitHub Issue

- **Trigger**: Webhook  
- **Action**: GitHub Issue fetch → Slack Post  
- **Key JSON bits**:

```json
/* GitHub Node */
"type": "n8n-nodes-base.githubTrigger",
"parameters": { "events": ["issues"] }

/* Slack Node */
"type": "n8n-nodes-base.slack",
"parameters": {
  "resource": "message",
  "operation": "post",
  "text": "New issue: {{$json[\"issue\"][\"title\"]}}"
}
```

### 12.2 Daily Report to Email

- **Trigger**: Cron  
- **Fetch**: MySQL  
- **Format**: Function → HTML  
- **Send**: SMTP Email  

### 12.3 CSV → Airtable Import

- **Trigger**: S3 New File  
- **CSV Parse** → SplitInBatches → Airtable

---

## 13. Further Resources

- n8n Official Docs: https://docs.n8n.io  
- Community Forum: https://community.n8n.io  
- GitHub Examples: https://github.com/n8n-io/n8n  
- n8n Workflow Templates: https://n8n.io/templates

---

This guide should equip you with all the building blocks to author **any** n8n workflow in JSON—whether simple or highly complex. Bookmark it, refer back whenever you need to tweak parameters, create custom nodes, or debug connection mishaps. Happy automating! 🚀
