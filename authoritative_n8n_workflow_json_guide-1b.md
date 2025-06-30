# n8n Workflow JSON Format: A Source-of-Truth Guide
# https://raw.githubusercontent.com/enescingoz/awesome-n8n-templates/refs/heads/main/Gmail_and_Email_Automation/Gmail%20AI%20Auto-Responder_%20Create%20Draft%20Replies%20to%20incoming%20emails.txt

> **Purpose:** This guide is an _authoritative_, end-to-end reference for crafting, reading, and debugging **any** n8n workflow in JSON. It uses a real-world Gmail + AI auto-responder workflow as the running example.

---

## Table of Contents

1. [Introduction](#introduction)  
2. [Top-Level Workflow Structure](#top-level-workflow-structure)  
   - id, name, active, versionId  
   - meta, pinData, settings  
   - tags  
3. [Nodes Array](#nodes-array)  
   - Core fields (`id`, `name`, `type`, `typeVersion`)  
   - `parameters` object  
   - `credentials` reference  
   - `position`  
4. [New & Advanced Node Types](#new--advanced-node-types)  
5. [Connections Object](#connections-object)  
   - `main` vs custom ports  
   - Mapping syntax  
6. [Detailed Example Breakdown](#detailed-example-breakdown)  
7. [Best Practices & Tips](#best-practices--tips)  
8. [Further Resources](#further-resources)  

---

## 1. Introduction

n8n stores every workflow as a single JSON object. Understanding this format lets you:

- Programmatically generate or update workflows  
- Version-control workflows in Git  
- Share templates without exporting/importing via UI  

We’ll dissect a _real_ workflow: **“Gmail AI auto-responder: create draft replies to incoming emails.”** This demo uses Gmail triggers, conditional logic, and n8n’s new LangChain/AI nodes to decide _if_ an email needs a reply, then draft it via OpenAI, and finally create a Gmail draft.

---

## 2. Top-Level Workflow Structure

Every workflow JSON lives at the root. Key fields:

```jsonc
{
  "id": "aOQANirVMuWrH0ZD",        // Unique workflow UUID
  "meta": {
    "instanceId": "…",            // n8n instance fingerprint
  },
  "name": "Gmail AI auto-responder: create draft replies to incoming emails",
  "tags": [],                      // Array of strings for categorization
  "active": true,                  // Whether workflow is enabled
  "versionId": "c4448c34-…",       // UI version identifier
  "pinData": {},                   // Pinning/annotation data (usually {})
  "settings": {
    "executionOrder": "v1"         // e.g. “v1” or “v2” execution engine
  },
  // nodes: [ … ], connections: { … } follow
}
```

- **id**: Workflow’s internal ID.  
- **meta.instanceId**: To de-duplicate across multiple n8n instances.  
- **name**: Human-readable.  
- **tags**: Free-form labels.  
- **active**: Set to `true` in production.  
- **versionId**: Incremented whenever the UI “Save” is clicked.  
- **pinData**: Reserved for UI annotations.  
- **settings.executionOrder**: Which internal engine to use (`v1` vs `v2`).

---

## 3. Nodes Array

`"nodes": [ … ]` holds each node’s definition. Every node includes:

| Field           | Purpose                                                           |
|-----------------|-------------------------------------------------------------------|
| `id`            | Unique node UUID                                                  |
| `name`          | Unique within workflow; used in expressions & connections         |
| `type`          | `<package>.<name>` e.g. `n8n-nodes-base.gmailTrigger`             |
| `typeVersion`   | Major version of the node                                         |
| `position`      | `[x, y]` coordinates for the UI layout                            |
| `parameters`    | Node-specific settings (see per-node docs)                        |
| `credentials`   | References to credential entries (no secrets in JSON)             |

### 3.1 Core Fields

```jsonc
{
  "id": "2a9ff08f-…",
  "name": "Gmail Trigger",
  "type": "n8n-nodes-base.gmailTrigger",
  "typeVersion": 1,
  "position": [-332.8, 566.1],
  "parameters": { /* … */ },
  "credentials": { /* … */ }
}
```

- **type**: Always matches the node’s `name` in the **n8n docs** under “Node Reference.”  
- **typeVersion**: When upgrading n8n, bump this if the node’s input/output shape changed.

### 3.2 Parameters

The `parameters` object differs per node:

- **Filters** (e.g. Gmail Trigger):  
  ```json
  "parameters": {
    "simple": false,
    "filters": { "q": "-from:me" },
    "pollTimes": { "item": [ { "mode": "everyMinute" } ] }
  }
  ```
- **If Node** conditions:
  ```json
  "parameters": {
    "conditions": {
      "combinator": "and",
      "conditions": [
        {
          "operator": { "type": "boolean", "operation": "true" },
          "leftValue": "={{ $json.needsReply }}",
          "rightValue": "true"
        }
      ]
    }
  }
  ```

> Tip: _Expressions_ use `={{ … }}` and full JavaScript. See § 5 below.

### 3.3 Credentials

Credentials reference existing n8n credentials by `id` and `name`—no secrets leak in JSON:

```json
"credentials": {
  "gmailOAuth2": {
    "id": "ofvBTX8A0aWfQb2O",
    "name": "Gmail account"
  }
}
```

---

## 4. New & Advanced Node Types

n8n v0.200+ ships AI/LLM nodes under the `@n8n/n8n-nodes-langchain` namespace:

- **chainLlm** ‑ generic LLM chain:  
  ```json
  "type": "@n8n/n8n-nodes-langchain.chainLlm",
  "parameters": { "prompt": "={{ … }}", "messages": { … } },
  "typeVersion": 1.3
  ```
- **lmChatOpenAi** ‑ OpenAI chat models (e.g. gpt-4, gpt-turbo):  
  ```json
  "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
  "parameters": { "model": "gpt-4o", "options": { "temperature": 0 } }
  ```
- **outputParserStructured** ‑ enforce JSON schema output:  
  ```json
  "type": "@n8n/n8n-nodes-langchain.outputParserStructured",
  "parameters": { "jsonSchema": "{ … }" }
  ```
- **Sticky Notes**: purely visual, no connections:
  ```json
  "type": "n8n-nodes-base.stickyNote",
  "parameters": { "content": "## …" }
  ```

---

## 5. Connections Object

The `"connections"` map defines _directed edges_ between nodes.

```jsonc
"connections": {
  "Gmail Trigger": {
    "main": [
      [
        { "node": "Assess if message needs a reply", "type": "main", "index": 0 }
      ]
    ]
  },
  "Assess if message needs a reply": {
    "main": [
      [
        { "node": "If Needs Reply", "type": "main", "index": 0 }
      ]
    ]
  },
  "If Needs Reply": {
    "main": [
      [
        { "node": "Generate email reply", "type": "main", "index": 0 }
      ]
    ]
  }
  // …and so on
}
```

- **Root key**: the **source** node’s `name`.  
- **Port**: `"main"` (primary) or custom (e.g. `"ai_outputParser"`, `"ai_languageModel"`).  
- **Array of arrays**: the outer array is outputs, each inner array is a list of targets.  
- **`index`**: port index (0-based for multi-output nodes).

---

## 6. Detailed Example Breakdown

Below is the fully-exported workflow JSON. Scroll through each section to see how top-level fields, nodes, and connections come together to implement:

> 1. **Trigger** on new Gmail messages not from you  
> 2. **Assess** via AI if a reply is needed  
> 3. **If yes**, generate reply draft via OpenAI  
> 4. **Create** a Gmail draft  

```json
{
  "id": "aOQANirVMuWrH0ZD",
  "meta": {
    "instanceId": "b78ce2d06ac74b90a581919cf44503cf07404c11eda5c3847597226683145618"
  },
  "name": "Gmail AI auto-responder: create draft replies to incoming emails",
  "tags": [],
  "active": true,
  "versionId": "c4448c34-1f75-4479-805e-20d8a69a7e00",
  "pinData": {},
  "settings": {
    "executionOrder": "v1"
  },
  "nodes": [
    {
      "id": "2a9ff08f-919a-41a8-980b-8c2bca3059e4",
      "name": "Gmail Trigger",
      "type": "n8n-nodes-base.gmailTrigger",
      "typeVersion": 1,
      "position": [-332.8, 566.1],
      "parameters": {
        "simple": false,
        "filters": { "q": "-from:me" },
        "pollTimes": { "item": [{ "mode": "everyMinute" }] }
      },
      "credentials": {
        "gmailOAuth2": { "id": "ofvBTX8A0aWfQb2O", "name": "Gmail account" }
      }
    },
    {
      "id": "86017ff4-9c57-4b2a-9cd9-f62571a05ffd",
      "name": "Assess if message needs a reply",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1.3,
      "position": [-92.8, 566.1],
      "parameters": {
        "prompt": "={{ `Subject: ${$json.subject}\\nMessage:\\n${$json.textAsHtml}` }}",
        "messages": {
          "messageValues": [
            {
              "message": "Your task is to assess if the message requires a response. Return in JSON format true if it does, false otherwise. Marketing emails don't require a response."
            }
          ]
        }
      }
    },
    {
      "id": "3ef14615-0045-404f-a21b-2c65a52f4be8",
      "name": "If Needs Reply",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [240, 560],
      "parameters": {
        "conditions": {
          "combinator": "and",
          "conditions": [
            {
              "operator": { "type": "boolean", "operation": "true" },
              "leftValue": "={{ $json.needsReply }}",
              "rightValue": "true"
            }
          ]
        }
      }
    },
    {
      "id": "36968dd5-8d51-4184-a05a-587b6c95aa82",
      "name": "JSON Parser",
      "type": "@n8n/n8n-nodes-langchain.outputParserStructured",
      "typeVersion": 1,
      "position": [100, 720],
      "parameters": {
        "jsonSchema": "{\n  \"type\":\"object\",\n  \"properties\":{ \"needsReply\":{ \"type\":\"boolean\" } },\n  \"required\":[\"needsReply\"]\n}"
      }
    },
    {
      "id": "cab1e7e5-93dc-4850-a471-e285cdbe2058",
      "name": "Generate email reply",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1.4,
      "position": [500, 520],
      "parameters": {
        "text": "={{ `Subject: ${$('Gmail Trigger').item.json.subject}\\nMessage: ${$('Gmail Trigger').item.json.textAsHtml}` }}",
        "messages": {
          "messageValues": [
            {
              "message": "You're a helpful assistant… Reply in same language…"
            }
          ]
        },
        "promptType": "define"
      }
    },
    {
      "id": "1a87c416-6b1c-4526-a2b6-20468c95ea0e",
      "name": "OpenAI Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1,
      "position": [480, 680],
      "parameters": {
        "model": "gpt-4-turbo",
        "options": {}
      },
      "credentials": {
        "openAiApi": { "id": "13ffkrNMlQMfvbZy", "name": "OpenAi account" }
      }
    },
    {
      "id": "84b4d516-252e-444e-b998-2d4aa0f89653",
      "name": "Gmail - Create Draft",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [900, 560],
      "parameters": {
        "message": "={{ $json.text.replace(/\\n/g, '<br />\\n') }}",
        "options": {
          "sendTo": "={{ $('Gmail Trigger').item.json.headers.from }}",
          "threadId": "={{ $('Gmail Trigger').item.json.threadId }}"
        },
        "subject": "={{ `Re: ${$('Gmail Trigger').item.json.headers.subject}` }}",
        "resource": "draft",
        "emailType": "html"
      },
      "credentials": {
        "gmailOAuth2": { "id": "ofvBTX8A0aWfQb2O", "name": "Gmail account" }
      }
    },
    // … plus several Sticky Note nodes for in-UI documentation …
  ],
  "connections": {
    "Gmail Trigger": {
      "main": [
        [
          { "node": "Assess if message needs a reply", "type": "main", "index": 0 }
        ]
      ]
    },
    "Assess if message needs a reply": {
      "main": [
        [
          { "node": "If Needs Reply", "type": "main", "index": 0 }
        ]
      ],
      "ai_outputParser": [
        [
          { "node": "JSON Parser", "type": "ai_outputParser", "index": 0 }
        ]
      ],
      "ai_languageModel": [
        [
          { "node": "JSON Parser", "type": "ai_languageModel", "index": 0 }
        ]
      ]
    },
    "If Needs Reply": {
      "main": [
        [
          { "node": "Generate email reply", "type": "main", "index": 0 }
        ]
      ]
    },
    "JSON Parser": {
      "ai_outputParser": [
        [
          { "node": "Assess if message needs a reply", "type": "ai_outputParser", "index": 0 }
        ]
      ]
    },
    "Generate email reply": {
      "main": [
        [
          { "node": "Gmail - Create Draft", "type": "main", "index": 0 }
        ]
      ]
    },
    "OpenAI Chat Model": {
      "ai_languageModel": [
        [
          { "node": "Generate email reply", "type": "ai_languageModel", "index": 0 }
        ]
      ]
    }
  }
}
```

---

## 7. Best Practices & Tips

- **Name Nodes Descriptively:** e.g. `Assess if message needs a reply` rather than `Node1`.  
- **Keep Credentials Out of JSON:** n8n stores secrets separately—never embed API keys.  
- **Use Expressions Sparingly:** Over-complex JS in `parameters` can obscure maintenance.  
- **Leverage Sticky Notes:** Document your intent in the canvas for team onboarding.  
- **Validate JSON:** Before import, run through a linter or CI pipeline.  
- **Version Control:** Check workflow JSON into Git; include `versionId` and `settings` for reproducibility.  

---

## 8. Further Resources

- Official Docs   https://docs.n8n.io  
- Node Reference  https://docs.n8n.io/nodes/  
- Community Forum https://community.n8n.io  
- GitHub Examples https://github.com/n8n-io/n8n  
- Workflow Templates https://n8n.io/templates  

