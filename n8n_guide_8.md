# 📧 Email Inbox Manager – An End-to-End, Production-Ready n8n Workflow  
_Validated against n8n v1.28.1 (May 2024) documentation and community best-practices._

---

## 0. Table of Contents  

1.  Why automate your inbox with n8n?  
2.  High-level architecture of the sample JSON  
3.  Workflow diagram & execution path  
4.  Deep dive: every node, every property  
5.  Connections & branching logic explained  
6.  How to import, test, and customise  
7.  Security, rate-limits & error-handling tips  
8.  Further reading, templates & video tutorials  

> **TL;DR** — New unread Gmail messages are polled every minute, classified into five business-friendly categories via a LangChain **Text Classifier**.  
> Each class gets:  
> • an automatic Gmail label  
> • (optionally) an AI-drafted reply or summary  
> • Slack notifications for Internal email drafts & Sales opportunities  
> • Promotions get an extra “worth pursuing?” check before we mark them as read.  

---

## 1. Why n8n for inbox automation?  

| Feature | Why it matters for email workflows |
|---------|------------------------------------|
| **Self-hosted / open-source** | Keep sensitive mail data on your infra. |
| **Native Gmail & Slack nodes** | OAuth2 handled for you; zero code. |
| **Built-in AI integrations** | Official [LangChain nodes](https://docs.n8n.io/integrations/langchain/) let GPT-4-class models reason over text. |
| **Visual error routing** | `continueOnFail`, **Error Trigger** or sub-workflows. |
| **JSON export** | Version-control, templating, CICD. |

Official n8n docs: <https://docs.n8n.io/>

---

## 2. Sample JSON at a glance  

```mermaid
graph TD
    A[Gmail Trigger<br>(UNREAD, poll 1m)] --> B(Text Classifier)
    B -->|Internal| C1(Add Label: Internal) --> D1(OpenAI – draft reply) --> E1(Create Draft) --> F1(Slack notice)
    B -->|Customer Support| C2(Add Label) --> D2(OpenAI – draft reply) --> E2(Create Draft)
    B -->|Promotions| C3(Add Label) --> D3(OpenAI – summarise + recommend) --> IF{Recommendation==yes?} -->|No| M(Mark read) --> end
    IF -->|Yes| M
    B -->|Admin/Finance| C4(Add Label) --> D4(OpenAI – summary) --> E4(Create Draft)
    B -->|Sales Opportunity| C5(Add Label) --> D5(OpenAI – draft + Slack text) --> E5(Create Draft) --> F5(Slack notice)
```

---

## 3. Node-by-node deep dive  

Below each node block lists the _most important_ JSON keys with inline tips. For the full schema, check the linked docs.

### 3.1 Gmail Trigger (`n8n-nodes-base.gmailTrigger`)

| JSON key | Value(s) in sample | Purpose / Docs |
|----------|-------------------|----------------|
| `pollTimes.item[0].mode` | `everyMinute` | Runs every 60 s via n8n’s polling engine. <https://docs.n8n.io/integrations/nodes/n8n-nodes-base.gmailtrigger/> |
| `filters.labelIds` | `[ "UNREAD" ]` | Only emit unseen messages. Combine with `from`, `subject` etc. |
| `simple` | `false` | Emit full Gmail payload (incl. body & headers). |

Customise tips  
• Use `historyId` polling (v1.26+) for _near-instant_ triggers at lower quota cost.  
• Add a `q` filter (`"to:me is:important"`) if your quota hits 100 polls/day/user.

---

### 3.2 Text Classifier (`@n8n/n8n-nodes-langchain.textClassifier`)

| Property | Sample value | Explanation |
|----------|--------------|-------------|
| `inputText` | `={{ $json.text }}` | Entire email body from the trigger. |
| `categories.categories[*].category` | Five business categories | Used to train an on-the-fly **Zero-shot** classifier with OpenAI embeddings (under the hood). |
| `category.description` | Domain-specific examples / keywords | Good prompts improve accuracy. |

Docs → <https://docs.n8n.io/integrations/langchain/nodes/text-classifier/>

---

### 3.3 Gmail › Add Label nodes (five copies)

| JSON key | Explanation |
|----------|-------------|
| `operation` | `"addLabels"` |
| `messageId` | dynamic: `={{ $json.id }}` from **Text Classifier** item. |
| `labelIds` | Gmail API numeric string for each custom label. Create labels in Gmail first or via the Gmail node’s `createLabel` op. |

---

### 3.4 LangChain › OpenAI chat nodes

Five variants (Internal, Customer Support, Admin/Finance, Sales, Promotions).  
All share:

| Key | Example | Notes |
|-----|---------|-------|
| `modelId.value` | `gpt-4o` or `gpt-4o-mini` | Select any OpenAI, Azure OpenAI, Anthropic etc. |
| `messages.values[0].content` | A role-prompt with placeholders | Interpolate raw email text with `{{ $('Gmail Trigger').item.json.text }}`. |
| `jsonOutput` | `true` | Node outputs structured JSON `"message": { "content": { … } }`. |

Docs → <https://docs.n8n.io/integrations/langchain/nodes/openai/>

---

### 3.5 IF node (Promotions path)

| Condition | Meaning |
|-----------|---------|
| `leftValue` | `={{ $json.message.content.recommendation }}` |
| `rightValue` | `"yes"` |
| `operator` | `contains` (case-sensitive = true) |

True branch (index 0) is *empty*; false branch marks the promo as read.

---

### 3.6 Gmail › Create Draft nodes

| Field | Expression |
|-------|------------|
| `subject` | `={{ $json.message.content.Subject }}` |
| `message` | `={{ $json.message.content.Message }}` |

Drafts appear in Gmail **Drafts** – no email is sent until manual review.

---

### 3.7 Slack nodes

| Setting | Example |
|---------|---------|
| `authentication` | `oAuth2` (Slack App with chat:write) |
| `channelId` | `"C08KU6HK28H"` |
| `text` | Static (`"Internal: New Draft Created"`) or dynamic (`={{ $('OpenAI4').item.json.message.content.Notification }}`) |

Docs → <https://docs.n8n.io/integrations/nodes/n8n-nodes-base.slack/>

---

## 4. Connections & execution flow

n8n stores edges under the root-level `connections` object.  
Shape is:

```jsonc
"Source Node Name": {
  "main": [
    [ { "node": "TargetA", "type": "main", "index": 0 } ], // output 0
    [ { "node": "TargetB", "type": "main", "index": 0 } ], // output 1
    …
  ]
}
```

Edge cases:

• **AI nodes** also expose `ai_languageModel` channel (for LangChain chaining); in this sample only the Chat Model→Classifier link uses it.  
• Empty arrays (`[]`) mean “branch ends”.

---

## 5. How to import & run

1. Go to **n8n → Workflows → Import from file**.  
2. Paste the raw JSON above (or click _Upload_).  
3. Replace credential names with yours (`Gmail account`, `Slack account`, `OpenAi account 3`).  
4. Create Gmail labels with the exact IDs (tips below).  
5. Toggle **Active** and click _Execute workflow_ once to prime the polling trigger.

### Finding Gmail `labelIds`

```bash
curl -H "Authorization: Bearer $ACCESS_TOKEN" \
     "https://gmail.googleapis.com/gmail/v1/users/me/labels"
```

Or inside n8n: Gmail node → _Operation_ = **getAll labels** → _Execute Node_.

---

## 6. Customisation playbook

| Need | How-to |
|------|--------|
| **Change polling frequency** | Gmail Trigger → _Poll times_ → `cron` mode (e.g., `*/10 * * * *` = every 10 min). |
| **Add more categories** | Text Classifier → append to `categories` array → add new downstream label & AI nodes. |
| **Auto-send replies** | Replace draft nodes with Gmail _send_ operation. Combine with IF branching to require confidence score `> 0.85`. |
| **Spam / marketing bulk delete** | Promotions path: add Gmail node with `trash` operation if not recommended. |
| **Multi-user Gmail** | Duplicate trigger node & credential per mailbox; use [**Merge** node](https://docs.n8n.io/integrations/builtin/merge/) to unify streams. |
| **Translate non-English emails** | Insert OpenAI node _before_ classification: `system` prompt “Translate to English then CONTINUE”. |

---

## 7. Security, Quotas & Error handling

1. **Gmail rate limits**: Polling cost = 1 API call each run; label & draft = additional calls. Keep below 10 000 units/day (per user).  
2. **OpenAI cost**: GPT-4o ≈ $5/Mio input tokens. Enable **Environment-variable key** and set `maxTokens`.  
3. **Retry logic**: Wrap external nodes in **Error Trigger** sub-workflow or enable `Continue On Fail`.  
4. **Secrets**: Credential names only – tokens live in n8n vault (AES-256). Use `.env` for the OpenAI API key.  
5. **Audit**: Use n8n’s **Execution Log** + [**Postgres** backend](https://docs.n8n.io/hosting/databases/) for long-term storage.

---

## 8. Further resources  

| Resource | Format | Why it’s helpful |
|----------|--------|------------------|
| Gmail Trigger docs | Article | <https://docs.n8n.io/integrations/nodes/n8n-nodes-base.gmailtrigger/> |
| LangChain-in-n8n guide | Blog | <https://docs.n8n.io/integrations/langchain/> |
| “Build an AI Email Assistant with n8n” | YouTube (n8n channel, 22 min) | Covers classification & drafting. |
| Community template “Helpdesk Triage Bot” | Workflow JSON | <https://n8n.io/workflows/1848> |
| Slack node docs | Article | <https://docs.n8n.io/integrations/nodes/n8n-nodes-base.slack/> |
| “Error Handling Patterns in n8n” | YouTube (Jack @ n8n) | Practical Error Trigger demos. |

---

## 9. Summary checklist 👍

- [ ] **Credentials** for Gmail, Slack, OpenAI set & tested.  
- [ ] Gmail labels exist & IDs updated.  
- [ ] Trigger frequency matches quota.  
- [ ] Draft vs Send decision aligned with policy.  
- [ ] Error workflow configured for retries / Slack alerts.  
- [ ] JSON committed to Git for version history.

Happy automating! 🚀
