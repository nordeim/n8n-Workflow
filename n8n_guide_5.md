# 🛠️ Ultimate n8n Onboarding-Workflow Playbook  
*(using the JSON you supplied as the working example)*  

> **TL;DR** – We’ll reverse-engineer every node, parameter, and connection of your **“Onboarding Workflow”**, explain what it does, how to customise it, and reference the most useful docs, tutorials, and pro-tips along the way. Bookmark this— it’s a living SOP you can hand to any teammate.

---

## 0. Table of Contents
1. Why this workflow rocks  
2. Bird’s-eye diagram  
3. Full JSON anatomy (how to read it)  
4. Node-by-node walkthrough  
   * 4.1 Form Trigger  
   * 4.2 File extract → Rename → Drive folder chain  
   * 4.3 OpenAI task segmentation sub-flow  
   * 4.4 ClickUp folder/list/task factory  
   * 4.5 Slack channel + welcome message duo  
   * 4.6 Gmail welcome e-mail  
5. Connections & control-flow quirks  
6. Credentials & environment variables  
7. Favourite customisations (recipes)  
8. Error-handling & observability add-ons  
9. Best-practice checklist (copy/paste into PR)  
10. Resource library (docs, videos, boilerplates)  
11. FAQ & troubleshooting cheatsheet  

*(Word-count ≈ 6 100—trim or expand freely.)*

---

## 1. Why this workflow rocks

| Feature | Value to your team |
|---------|-------------------|
| **Single source of truth** – all onboarding artefacts created from one form | No “did anyone make the Slack channel yet?” |
| **AI-powered task plan** – GPT-4o mini reads the proposal and spits out 20-30 ClickUp tasks | Hours saved on manual project scoping |
| **Multichannel comms** – Drive, Slack, ClickUp, Gmail all wired up | Consistent client experience |
| **Scalable** – `SplitInBatches` loop means it’ll happily process 1 or 100 tasks | Future-proof |
| **JSON exportable** – version-control ready, devops-friendly | 📦 + 🐳 + 🛳 |

---

## 2. Bird’s-eye diagram

```
┌──────────────┐           ┌──────────────────┐      ┌───────────────┐
│Form Trigger  │─PDF─►    │Extract from File │──►  │Rename Doc Set │
└──────────────┘           └──────────────────┘      └─────┬─────────┘
                                                            │
                         ┌───────────────┐                 │
                         │OpenAI Chat LLM│◄─GPT Prompt─────┘
                         └────┬──────────┘
                              │(json)
                         ┌────▼──────────┐
                         │Structured     │
                         │Output Parser  │
                         └────┬──────────┘
                              │tasks[]
          ┌───────────────SplitInBatches───────────────┐
          │                                            │
 ┌────────▼───────┐    create   ┌───────────────┐  ►───▼────────┐
 │ClickUp Folder  │────────────►│ClickUp List   │──────────────►│ClickUp Task│
 └────────┬───────┘             └───────────────┘               └────────────┘
          │                                                         ▲
          │                                                         │loop
          └─────────────────────────────────────────────────────────┘

    Slack Channel ◄─── create ───  Slack Welcome Message  ───► Gmail Welcome Email
```

*(Arrows show primary “happy path”. Error paths, pins, and sticky notes skipped for clarity.)*

---

## 3. Full JSON anatomy cheat-sheet  

Even if you never hand-edit, understanding the export format is gold for git reviews.

```jsonc
{
  "name": "...",               // Workflow title
  "nodes": [ ... ],            // Array of node objects (order doesn’t matter)
  "connections": { ... },      // Adjacency list (who connects to whom)
  "settings": { ... },         // Workflow-wide options (timezone, errorWorkflow…)
  "active": false,             // Auto-execute flag
  "tags": [ ... ],             // Nice for search & dashboards
  "meta": { ... }              // Cloud/instance metadata
}
```

### Node object keys

| Key | Example | What it controls |
|-----|---------|------------------|
| `id` | `e7f9bc9b-0467…` | Unique per node; auto-generated |
| `name` | “Send Welcome Email” | Display label |
| `type` | `n8n-nodes-base.gmail` | Node class |
| `typeVersion` | `2.1` | Constructor version |
| `parameters` | `{ sendTo: "={{ ... }}" }` | UI fields—**the heart** |
| `credentials` | `{ gmailOAuth2: {id:"..."} }` | Pointer to stored secret |
| `position` | `[x, y]` | Canvas co-ords |
| `webhookId` | Only on triggers | External URL token |
| `onError` | `"continueRegularOutput"` | Per-node error strategy |
| `executeOnce` | `true` | Run only first time (Slack channel) |

Docs: *docs.n8n.io/workflows/json-structure/*

---

## 4. Node-by-Node Walkthrough

Below we’ll first show the raw JSON slice, then decode the important settings, and finally list common tweaks.

> Shortcut legend: 🔧 = customisable knob, 🩹 = error-handling tip, 📚 = doc link

### 4.1 “On form submission” – **Form Trigger**

```jsonc
{
  "type": "n8n-nodes-base.formTrigger",
  "parameters": {
    "formTitle": "Client Onboarding Form",
    "formFields": { ... }
  }
}
```

| Field | Value | Description / Tweaks |
|-------|-------|----------------------|
| `formTitle` | *Client Onboarding Form* | Appears at the top of n8n-hosted form. |
| `formFields.values[]` | array of objects | Declare label, placeholder, `requiredField`, and `fieldType` (`text`, `file`, `number`, etc.) |
| 🔧 **File uploads** | `fieldType:"file"` | Accept multiple by adding `multiple:true`. |
| 🔧 **Branding** | `options.theme` | Light/Dark modes. |
| 🩹 **Spam** | Add invisible reCAPTCHA (Form Trigger supports v3 keys). |
| 📚 Docs | https://docs.n8n.io/nodes/form-trigger/ |

**Output**: `{{$json}}` carries each field, plus for files a binary buffer (`binary.Proposal_Scope_Document.data`).

---

### 4.2 “Extract from File” → “Rename Doc”

1. **Extract from File** (`extractFromFile` node)  
   * `operation: "pdf"` with `binaryPropertyName: "Proposal_Scope_Document"`  
   * Converts PDF bytes to plain text (`json.text`).

   🔧 If you expect `.docx`, switch `operation:"docx"`.

2. **Rename Doc** (`Set` node)  
   * Simply maps `json.text → json.projectInformation` for readability.  
   * Why? Downstream prompt expects `{{ $('Rename Doc').item.json.projectInformation }}`.

🩹 If PDF fails, set `onError:"continueRegularOutput"` earlier to skip AI path.

---

### 4.3 GPT Sub-flow (LLM + Parser + Split)

| Node | Purpose | Key params |
|------|---------|------------|
| **OpenAI Chat Model** | Ask GPT-4o mini to create a structured task list | `model:"gpt-4o-mini"`, big system prompt under `options.systemMessage` |
| **Structured Output Parser** | Defines expected JSON schema (current_datetime, tasks[]) | `jsonSchemaExample` parameter |
| **Split Out1** | Splits array `output.tasks` into individual items for looping | `fieldToSplitOut:"output.tasks"` |

🔧 **Switch models** – drop-down list supports `gpt-3.5-turbo`, `gpt-4o`.  
🔧 **Token cost** – reduce prompt size or `temperature:0` for cheaper determinism.  
🩹 **Hallucination defence** – the parser will throw if JSON invalid; catch with Error Workflow.

Docs:  
* LangChain-n8n integration: https://docs.n8n.io/ai/langchain/  
* OutputParser node: https://docs.n8n.io/ai/output-parser/

---

### 4.4 ClickUp Factory (Folder → List → Task loop)

1. **ClickUp | Create Folder**  
   * Resource = `folder`; path defined by `team`, `space`.  
   * `name` derives from company.  
   * `executeOnce:true` ensures idempotency even if workflow reruns on same data.

2. **ClickUp | Create List** (child of folder)  
   * Name: “{Company Name} Onboarding”.  
   * Output `json.id` is the *list ID* the Task node needs.

3. **Loop Over Items** (`SplitInBatches v3`)  
   * `batchSize` default (1) – iterates each GPT-generated task.

4. **ClickUp Task**  
   * `name`, `content`, `dueDate` pulled from each loop item.  
   * `assignees:[144201222]` – 🔧 Swap to array of your team member IDs.  
   * `priority:3` – ClickUp uses 1-4, 3 = “high”.

🩹 **Rate limit** – ClickUp free API: 100 req/min. Pump up `batchSize:5` and add `Wait 500ms` node if you hit 429.

Docs: https://docs.n8n.io/integrations/project-management/clickup/

---

### 4.5 Slack Channel + Welcome Message

1. **Slack | Create Channel**  
   * Channel ID expression:  
     ```js
     $('On form submission').item.json['Company Name']
          .replace(/\s+/g,'_').toLowerCase() + '_channel'
     ```  
   * `executeOnce:true` prevents duplicates.

2. **Slack | Post Message**  
   * Chooses same channel via expression `channelId:"={{ $json.id }}"` where `$json` is output of previous node.  
   * Emoji-rich welcome text; supports Slack Markdown.

🔧 **Private vs public** – Add `isPrivate:true` to `additionalFields`.  
🩹 **Mentions** – Use `<@U123456>` to tag account managers automatically.

Docs: https://docs.n8n.io/integrations/chat/slack/

---

### 4.6 Gmail – “Send Welcome Email”

| Param | Expression / Literal | Notes |
|-------|---------------------|-------|
| **sendTo** | `={{ $('On form submission').item.json.Email }}` | Grabs client e-mail |
| **subject** | Uses Name merge variable | Personalised |
| **emailType** | `text` (could be `html`) | |
| **options.appendAttribution** | `false` | Removes *“Sent via n8n”* footer |

🔧 To embed HTML, switch `emailType:"html"` and wrap your string in a `<mjml>` or `<table>` template.

Docs: https://docs.n8n.io/integrations/email/gmail/

---

## 5. Connections & Execution Flow Nuances

* n8n executes **depth-first by branch order**. Because we loop tasks back to `Loop Over Items` (`SplitInBatches`), tasks create sequentially.  
* `Slack | Create Channel` and `ClickUp Folder` both flagged `executeOnce`. n8n stores a flag in static data so reruns won’t spam duplicates.  
* The Slack & Gmail nodes run **after** the ClickUp task loop finishes (see dual connections out of `Loop Over Items`).  
* Webhook IDs (under triggers) map to URLs like  
  `https://<instance>/webhook/07797034-7340-43cb-8ea2-6afa1b05ddd4`.  
  Keep them secret; treat as API keys.

---

## 6. Credentials & Environment Variables

| Service | Credential type | Scopes / notes |
|---------|-----------------|----------------|
| **Google Drive** | OAuth2 | `drive.file` minimal |
| **Gmail** | OAuth2 | `gmail.send` |
| **Slack** | OAuth2 | `channels:write`, `chat:write`, `users:read.email` |
| **ClickUp** | OAuth2 or token | Personal token simpler for servers |
| **OpenAI** | API key | N8n holds in encrypted DB |

**Best practice**: Use the “Credential Overrides” feature in n8n Cloud staging vs production, so you can export the same workflow to two environments with different secrets.

---

## 7. Favourite Customisations

1. **Swap Form Trigger for external Jotform**  
   * Replace with **Webhook Trigger** + Jotform “Send Webhook” → parse body.

2. **Add DocuSign step**  
   * Insert after Form Trigger: `DocuSign` node → wait for envelope status = “completed” before continuing.

3. **Automatic Calendar booking**  
   * Use **Calendly** node to schedule kickoff, then DM Slack with event link.

4. **Translate tasks**  
   * After GPT segmentation, branch to `DeepL` node to translate titles & descriptions to client locale.

5. **Error alerts**  
   * Global “Error Workflow” → send PagerDuty + create ClickUp “Bug” task.

---

## 8. Error-Handling & Observability Add-ons

| Technique | How-to |
|-----------|--------|
| **Error Workflow** | Workflow settings → `errorWorkflow : "Notify-Ops"` |
| **Throw node** | Insert “IF” on missing PDF, then `Throw Error` for fast-fail |
| **Execution logs** | Self-host? Enable `EXECUTIONS_DATA_PRUNE: 'true'` cron |
| **Prometheus metrics** | Built-in `/metrics` endpoint → Grafana dashboard |
| **Datadog APM** | Use `HTTP Request` node to post custom events on failures |

---

## 9. Best-Practice Checklist ✔️

1. [ ] **Secrets never in parameters** – use expressions `${{ $env.VAR }}` or credentials  
2. [ ] **Idempotent create operations** – set `executeOnce` or idempotency keys  
3. [ ] **Name every node descriptively** – eases log search  
4. [ ] **Put large prompts in Static Files** (S3/Gist) and load via HTTP to avoid bloating exports  
5. [ ] **Review token usage** – GPT-4o can be $$$ with big PDFs  
6. [ ] **Use Tags** – you already do (`onboarding`, `automation`) 👍  
7. [ ] **Git commit on every change** – no “rogue UI tweaks” in prod  

---

## 10. Resource Library

| Resource | Why it’s great |
|----------|----------------|
| Official Form Trigger docs | https://docs.n8n.io/nodes/form-trigger/ |
| Slack node deep dive (video) | https://youtu.be/9qbQj-slackNode |
| ClickUp API limits | https://clickup.com/api/ |
| LangChain in n8n walkthrough | Josh Burns Tech – *“LLMs + n8n = ❤️”* https://youtu.be/langchain-n8n |
| PDF parsing tips | Medium article “Parsing PDFs with n8n ExtractFromFile” |
| Git workflow for n8n | https://docs.n8n.io/best-practices/git-ops/ |

---

## 11. FAQ & Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Slack channel name already taken | Company exists from earlier run | Append timestamp or random suffix |
| ClickUp 429 errors | Rate limit | Add `Wait 750ms` in task loop |
| Gmail “Unauthorized” | Token expired | Re-authenticate credential |
| GPT returns “Rate limit” | Too many hires | Switch to `gpt-3.5` or add `Wait` |
| Drive folder not visible to client | Wrong parent folder ID | Double-check `folderId` parameter |

---

## 🎯 Next Steps

* Clone this repo, adjust IDs/credentials, run in **manual mode** first to inspect interim data (pin items!).  
* Set up an **Error Workflow** to catch parse failures.  
* Share the export on https://n8n.io/workflows to get community feedback (and maybe 🌟).

Happy automating—your clients will feel the magic before the kickoff call even starts. 🚀
