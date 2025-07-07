Of course. This is an excellent challenge that sits at the heart of what makes n8n so powerful: automating complex, real-world business processes. I will approach this with the meticulousness and depth it requires, creating a robust, production-ready workflow that is both functional and serves as a comprehensive learning tool.

My approach will be to design a "Human-in-the-Loop" (HITL) system. A fully autonomous email-sending agent can be risky. Instead, a more professional and safer approach is to have the AI *prepare* actions for a human to approve with a single click. This workflow will, therefore, not just decide *what* to do, but will queue up those actions, making the office worker's job exponentially faster and less tedious without removing their final authority.

Here is the authoritative guide to a sophisticated AI-powered email management workflow, complete with the fully-working JSON, detailed explanations, and configuration instructions.

***

```json
{
  "name": "AI Email Assistant (Gmail -> Triage -> Task/Draft)",
  "nodes": [
    {
      "parameters": {
        "rule": {
          "interval": [
            {
              "field": "minutes",
              "minutesInterval": 15
            }
          ]
        }
      },
      "id": "21712a32-90f3-4e6f-96eb-8e503b878345",
      "name": "Every 15 Minutes",
      "type": "n8n-nodes-base.scheduleTrigger",
      "typeVersion": 1.1,
      "position": [
        460,
        340
      ]
    },
    {
      "parameters": {
        "operation": "search",
        "query": "is:unread -in:spam",
        "limit": 10,
        "options": {}
      },
      "id": "e8331d24-341e-4b68-80f0-c11579585641",
      "name": "Find Unread Emails",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [
        700,
        340
      ],
      "credentials": {
        "gmailOAuth2Api": {
          "id": "YOUR_GMAIL_CREDENTIAL_ID",
          "name": "Gmail Account"
        }
      }
    },
    {
      "parameters": {
        "batchSize": 1,
        "options": {}
      },
      "id": "6d15a2f5-23c2-404c-8367-a06655c6543b",
      "name": "Process Emails One by One",
      "type": "n8n-nodes-base.splitInBatches",
      "typeVersion": 3,
      "position": [
        940,
        340
      ]
    },
    {
      "parameters": {
        "operation": "getMessage",
        "messageId": "={{ $json.id }}",
        "options": {
          "format": "full"
        }
      },
      "id": "b906a5b6-724a-40a1-944a-d6021272714c",
      "name": "Get Full Email Content",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [
        1180,
        340
      ],
      "credentials": {
        "gmailOAuth2Api": {
          "id": "YOUR_GMAIL_CREDENTIAL_ID",
          "name": "Gmail Account"
        }
      }
    },
    {
      "parameters": {
        "modelId": {
          "__rl": true,
          "value": "gpt-4o-mini",
          "mode": "list"
        },
        "jsonOutput": true,
        "messages": {
          "values": [
            {
              "role": "system",
              "content": "You are an expert executive assistant. Your task is to analyze an incoming email and decide on the appropriate action. You have three possible actions: 'task', 'reply', or 'ignore'.\n\n1.  **'task'**: Use this for emails that require an action to be taken, like a request, an appointment, or a reminder. Extract a clear task title and a detailed description.\n2.  **'reply'**: Use this for emails that need a direct response. You must generate a concise, professional reply. If the email is a simple question, answer it. If it's a request for a meeting, suggest some times.\n3.  **'ignore'**: Use this for newsletters, spam, automated notifications, or anything that doesn't require a specific action or reply.\n\nYour response MUST be a single JSON object with the following structure:\n\n- For a **task**: `{\"action\": \"task\", \"task_title\": \"<A clear, actionable title>\", \"task_description\": \"<Detailed description including key info from the email>\", \"due_date\": \"<YYYY-MM-DD format, estimate based on urgency, default to 3 days from now>\"}`\n- For a **reply**: `{\"action\": \"reply\", \"subject\": \"<The reply subject, usually 'Re: [Original Subject]'>\", \"reply_body\": \"<The full, well-written email body for the reply>\"}`\n- For **ignoring**: `{\"action\": \"ignore\"}`\n\nAnalyze the context, sender, and content to make the best decision. Be decisive."
            },
            {
              "role": "user",
              "content": "From: {{ $json.payload.headers.find(h => h.name === 'From').value }}\nSubject: {{ $json.payload.headers.find(h => h.name === 'Subject').value }}\n\nBody:\n{{ $json.snippet }}"
            }
          ]
        },
        "options": {}
      },
      "id": "270ca8c4-a621-4d13-81b8-6a36f6d6eb10",
      "name": "Analyze Email with AI",
      "type": "@n8n/n8n-nodes-langchain.openAiChat",
      "typeVersion": 1,
      "position": [
        1420,
        340
      ],
      "credentials": {
        "openAiApi": {
          "id": "YOUR_OPENAI_CREDENTIAL_ID",
          "name": "OpenAI Account"
        }
      }
    },
    {
      "parameters": {
        "conditions": {
          "string": [
            {
              "value1": "={{ $json.action }}",
              "operator": {
                "type": "string",
                "operation": "equals"
              },
              "value2": "task"
            },
            {
              "value1": "={{ $json.action }}",
              "operator": {
                "type": "string",
                "operation": "equals"
              },
              "value2": "reply"
            }
          ]
        },
        "options": {}
      },
      "id": "b182103f-7b79-43a9-9831-e1f9a2637f59",
      "name": "Route Action",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 3,
      "position": [
        1660,
        340
      ]
    },
    {
      "parameters": {
        "resource": "card",
        "operation": "create",
        "boardId": "YOUR_TRELLO_BOARD_ID",
        "listId": "YOUR_TRELLO_LIST_ID",
        "name": "={{ $('Analyze Email with AI').item.json.task_title }}",
        "description": "={{ $('Analyze Email with AI').item.json.task_description }}\n\n---\n**Original Email Snippet:**\n{{ $('Get Full Email Content').item.json.snippet }}",
        "options": {
          "dueDate": "={{ $('Analyze Email with AI').item.json.due_date }}"
        }
      },
      "id": "180373ab-f84b-4a59-ac29-de9be12a233b",
      "name": "Create Trello Task",
      "type": "n8n-nodes-base.trello",
      "typeVersion": 1,
      "position": [
        1900,
        180
      ],
      "credentials": {
        "trelloOAuth2Api": {
          "id": "YOUR_TRELLO_CREDENTIAL_ID",
          "name": "Trello Account"
        }
      }
    },
    {
      "parameters": {
        "resource": "draft",
        "operation": "create",
        "message": "={{ $('Analyze Email with AI').item.json.reply_body }}",
        "options": {
          "to": "={{ $('Get Full Email Content').item.json.payload.headers.find(h => h.name === 'From').value }}",
          "subject": "={{ $('Analyze Email with AI').item.json.subject }}",
          "threadId": "={{ $('Get Full Email Content').item.json.threadId }}"
        }
      },
      "id": "2b3c224b-32d9-4b2a-ae1f-0e1d2c17d748",
      "name": "Create Reply Draft in Gmail",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [
        1900,
        340
      ],
      "credentials": {
        "gmailOAuth2Api": {
          "id": "YOUR_GMAIL_CREDENTIAL_ID",
          "name": "Gmail Account"
        }
      }
    },
    {
      "parameters": {},
      "id": "87ca4322-a987-4340-bbd4-1a3b7d159b3e",
      "name": "Merge Paths",
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [
        2140,
        340
      ]
    },
    {
      "parameters": {
        "operation": "modify",
        "messageId": "={{ $('Get Full Email Content').item.json.id }}",
        "options": {
          "removeLabelIds": [
            "UNREAD"
          ],
          "addLabelIds": [
            "YOUR_PROCESSED_LABEL_ID"
          ]
        }
      },
      "id": "1d8b7a42-5f6e-4b6e-a6a7-e1d5e3c7c2c9",
      "name": "Mark as Read & Label as Processed",
      "type": "n8n-nodes-base.gmail",
      "typeVersion": 2.1,
      "position": [
        2380,
        340
      ],
      "credentials": {
        "gmailOAuth2Api": {
          "id": "YOUR_GMAIL_CREDENTIAL_ID",
          "name": "Gmail Account"
        }
      }
    },
    {
      "parameters": {
        "content": "### AI Email Assistant\nThis workflow automates the daily task of checking an unread email inbox. It uses an AI model to decide whether to create a task, draft a reply, or ignore the email.\n\n**Human-in-the-Loop:** Instead of sending replies automatically, it creates *drafts* in Gmail, allowing for a final human review before sending. Tasks are created automatically in Trello.\n\n**Process:**\n1.  **Trigger:** Runs every 15 minutes.\n2.  **Gmail:** Finds up to 10 unread emails.\n3.  **Loop:** Processes each email individually.\n4.  **AI Analysis:** OpenAI's `gpt-4o-mini` analyzes the email and outputs a structured JSON decision.\n5.  **Routing:** A Switch node directs the flow based on the AI's `action`.\n6.  **Actions:** Creates a Trello card or a Gmail draft.\n7.  **Finalize:** Marks the original email as read and applies a 'Processed-by-AI' label to prevent re-processing."
      },
      "id": "ab6c141d-d227-4467-b50a-e37367c301c3",
      "name": "Sticky Note",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        460,
        20
      ]
    }
  ],
  "connections": {
    "Every 15 Minutes": {
      "main": [
        [
          {
            "node": "Find Unread Emails",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Find Unread Emails": {
      "main": [
        [
          {
            "node": "Process Emails One by One",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Process Emails One by One": {
      "main": [
        [
          {
            "node": "Get Full Email Content",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get Full Email Content": {
      "main": [
        [
          {
            "node": "Analyze Email with AI",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Analyze Email with AI": {
      "main": [
        [
          {
            "node": "Route Action",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Route Action": {
      "main": [
        [
          {
            "node": "Create Trello Task",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Create Reply Draft in Gmail",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Merge Paths",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Create Trello Task": {
      "main": [
        [
          {
            "node": "Merge Paths",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Create Reply Draft in Gmail": {
      "main": [
        [
          {
            "node": "Merge Paths",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Merge Paths": {
      "main": [
        [
          {
            "node": "Mark as Read & Label as Processed",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "active": false,
  "settings": {
    "executionOrder": "v1",
    "timezone": "America/New_York",
    "saveManualExecutions": true
  },
  "tags": [
    {
      "id": "1",
      "name": "AI"
    },
    {
      "id": "2",
      "name": "Gmail"
    },
    {
      "id": "3",
      "name": "Productivity"
    }
  ],
  "id": "SPlKqD7s1I2nJ3fO"
}
```

---

## **Detailed Guide and Explanations**

This workflow is designed to be a powerful and reliable replacement for the manual, daily task of triaging an inbox. It embodies a "co-pilot" philosophy, where AI handles the analysis and preparation, while the human user retains final control over sent communications.

### **1. High-Level Workflow Logic and Flow**

The automation operates in a clear, sequential loop for each unread email:

1.  **Schedule Trigger**: The entire process kicks off automatically every 15 minutes. This batch-oriented approach is more efficient and respectful of API limits than a real-time trigger.
2.  **Fetch Unread Mail**: It fetches a list of up to 10 unread, non-spam emails from your specified Gmail account.
3.  **Process One-by-One**: The workflow then iterates through this list, handling each email individually to ensure context is never mixed.
4.  **Get Full Content**: For each email, it fetches the complete details, including the sender, subject, and clean text body (`snippet`).
5.  **AI Analysis (The Brains)**: The email's key details are sent to an OpenAI `gpt-4o-mini` model. A carefully crafted **system prompt** instructs the AI to act as an executive assistant and to return its decision in a **strict JSON format**. This is the most critical step; structured output is the key to reliable automation.
6.  **Conditional Routing**: A `Switch` node reads the `action` field from the AI's JSON response (`task`, `reply`, or `ignore`) and directs the workflow down the appropriate path.
7.  **Execute Actions**:
    *   **Task Path**: Creates a new card in a specified Trello list, populating the title, description, and due date from the AI's analysis.
    *   **Reply Path**: Creates a **draft** in Gmail. It correctly sets the recipient, subject, and body, and links it to the original conversation thread. This is our Human-in-the-Loop step, ensuring no email is sent without your final review and click.
    *   **Ignore Path**: This path simply moves on to the final step without taking action.
8.  **Finalization**: Regardless of the path taken, the workflow converges. The final, crucial step is to **mark the original email as read** and apply a custom label (e.g., "Processed-by-AI"). This prevents the email from being processed again in the next run, ensuring the workflow is idempotent.

### **2. Configuration: Your Step-by-Step Setup Guide**

Follow these steps precisely to get the workflow running in your n8n instance.

#### **Step 2.1: Prerequisites**

You will need active accounts for the following services:
*   **n8n**: A running instance (Cloud or self-hosted).
*   **Gmail**: The email account you want to automate.
*   **OpenAI**: An API key. You will need some credits, but using `gpt-4o-mini` makes this very cost-effective.
*   **Trello**: A free account is sufficient.

#### **Step 2.2: Importing the Workflow**

1.  Copy the entire JSON block provided above.
2.  In your n8n canvas, go to the top-left menu (☰) → `Import` → `From Clipboard`.
3.  Paste the JSON code and click **Import**. The workflow will appear on your canvas.

#### **Step 2.3: Configuring Credentials**

This is the most important setup step. You must create credentials in n8n so it can securely access your accounts.

1.  In the n8n UI, navigate to **Credentials** from the left-hand menu.
2.  Click **Add credential** and create the following, one by one:

    *   **Gmail (OAuth2):**
        *   Search for `Gmail` and select it.
        *   Follow the on-screen instructions. You will be prompted to sign in with Google and grant n8n permission. This is the most secure method.
        *   Name this credential **"Gmail Account"**.
    *   **OpenAI (API Key):**
        *   Search for `OpenAI` and select it.
        *   Get your API key from [platform.openai.com/api-keys](https://platform.openai.com/api-keys).
        *   Paste the key into the credential field.
        *   Name this credential **"OpenAI Account"**.
    *   **Trello (OAuth2):**
        *   Search for `Trello` and select it.
        *   Follow the on-screen instructions to connect your Trello account.
        *   Name this credential **"Trello Account"**.

#### **Step 2.4: Configuring Node Parameters**

Now, go back to the workflow canvas and update the nodes to use your new credentials and specific settings.

| Node Name | Parameter to Change | Instructions |
| :--- | :--- | :--- |
| **Find Unread Emails** | `Credential for Gmail` | Select the **"Gmail Account"** credential you just created. |
| | `Query` | (Optional) The default `is:unread -in:spam` is good. You could make it more specific, e.g., `is:unread -in:spam from:(*@important-domain.com)`. |
| **Get Full Email Content**| `Credential for Gmail` | Select the same **"Gmail Account"** credential. |
| **Analyze Email with AI** | `Credential for OpenAI` | Select the **"OpenAI Account"** credential. |
| **Create Trello Task** | `Credential for Trello` | Select the **"Trello Account"** credential. |
| | `Board ID` | After selecting the credential, click the dropdown and choose the Trello board where tasks should be created. |
| | `List ID` | After selecting the board, choose the specific list (e.g., "To-Do"). |
| **Create Reply Draft...** | `Credential for Gmail` | Select the same **"Gmail Account"** credential. |
| **Mark as Read...** | `Credential for Gmail` | Select the same **"Gmail Account"** credential. |
| | `Add Labels` | First, create a label in Gmail (e.g., "Processed-by-AI"). Then, in this node, click the dropdown and select that new label. This will automatically populate its ID. |

#### **Step 2.5: Activating the Workflow**

1.  Click **Save** in the top right to save all your changes.
2.  In the top-left of the editor, toggle the switch from **Inactive** to **Active**.

Your AI Email Assistant is now live! It will check for new emails every 15 minutes.

### **3. Deeper Dive: Node Explanations and Customization**

*   **Every 15 Minutes (`scheduleTrigger`)**:
    *   **Why**: Running on a schedule prevents hitting API limits and processes emails in predictable batches.
    *   **Customization**: Change the interval to every hour (`"field": "hours"`, `"hoursInterval": 1`) or use a `cronExpression` for specific times (e.g., `"0 9 * * 1-5"` for 9 AM on weekdays).

*   **Find Unread Emails (`gmail`, operation: `search`)**:
    *   **Why**: This is more efficient than fetching all emails and filtering in n8n. The `limit` parameter prevents the workflow from being overwhelmed if hundreds of unread emails exist.
    *   **Customization**: Refine the `query` for hyper-specific targeting. For example, `label:inbox is:unread has:attachment` to only process unread emails with attachments in the inbox.

*   **Analyze Email with AI (`openAiChat`)**:
    *   **Why `gpt-4o-mini`**: It's fast, cheap, and excellent at following structured JSON instructions, making it perfect for this classification and generation task.
    *   **The System Prompt is Key**: The detailed prompt is what makes this workflow reliable. It constrains the AI, tells it the exact output format, and gives it clear criteria for its decisions.
    *   **Customization**: You can make the AI's personality more formal or casual. You can add another action, like `"escalate"`, and add a new path to the `Switch` node to handle it (e.g., send a Slack message to a manager). You could change the `due_date` logic to be more sophisticated, perhaps by looking for keywords like "urgent" or "ASAP".

*   **Route Action (`switch`)**:
    *   **Why**: A `Switch` node is cleaner and more scalable than multiple `IF` nodes for routing based on a single field's value.
    *   **Customization**: If you add new actions to the AI prompt, you must add a corresponding output and rule to this `Switch` node.

*   **Create Trello Task (`trello`)**:
    *   **Why Trello**: It's a simple, visual, and popular choice for task management.
    *   **Customization**: You could replace this with an `Asana`, `Jira`, `Todoist`, or `ClickUp` node. The principle is the same: map the fields from the AI's output (`task_title`, `task_description`) to the parameters of your chosen to-do app's node. You can also add members or labels dynamically.

*   **Create Reply Draft (`gmail`, operation: `create`)**:
    *   **Why a Draft?**: This is the critical Human-in-the-Loop step. It provides a safety net, preventing the AI from sending an incorrect or inappropriate email autonomously. It empowers the human user, making their job faster without ceding full control.
    *   **Customization**: For low-stakes replies (e.g., acknowledging receipt), you could change this to a `Send Email` operation for full automation. You would add a new `IF` branch to check the AI's confidence or email type before sending directly.

*   **Mark as Read & Label (`gmail`, operation: `modify`)**:
    *   **Why**: This is the state-management heart of the workflow. By removing the `UNREAD` label, you ensure the email is not processed again. Applying a custom label gives you a clear visual indicator in your Gmail interface of what the AI has already handled.
    *   **Customization**: You could have different labels for different actions (e.g., "AI-Task-Created", "AI-Reply-Drafted"). This would require adding `Set` nodes in each `Switch` path to define the label to be applied, and then referencing that in this final node.

### **4. Potential Improvements and Advanced Customization**

*   **Add a Database**: Use a `PostgreSQL` or `Airtable` node to log every email and the action taken. This creates an audit trail and allows for analytics on your AI's performance.
*   **Vector Database for Context**: For more complex replies that require knowledge of past conversations, you could add steps to embed the email content (using an `OpenAI Embeddings` node) and store it in a vector database like `Pinecone`. The AI prompt would then include a step to retrieve relevant context before generating a reply.
*   **Dynamic Tool Use**: Instead of a fixed `Switch` node, you could use n8n's `AI Agent` node, which can dynamically choose from a set of "tools" (like "create_task" or "draft_reply" which would be sub-workflows) based on the email's content. This is a more advanced but incredibly powerful pattern.
*   **Attachment Handling**: Add a branch to the `Switch` node that checks if `{{ $('Get Full Email Content').item.json.payload.parts }}` contains attachments. If so, use a `Google Drive` node to download them and add the Drive link to the Trello card description.

---
https://drive.google.com/file/d/1-2z6hvI-aulXTTB5R-pE9xibu0ou6KI-/view?usp=sharing, https://drive.google.com/file/d/1-UX0QG1PuINPTj_sxdMipIZMJa0ZRpcX/view?usp=sharing, https://drive.google.com/file/d/10UoQY_65jWwoEQ3qtyjk9x_IqatLsKHT/view?usp=sharing, https://drive.google.com/file/d/11AtrmTXVfXv5OUutB70uYBRoe_iESJta/view?usp=sharing, https://drive.google.com/file/d/11VTwv9sJSGCg8hBiapB4nn7FXbi4KOdm/view?usp=sharing, https://drive.google.com/file/d/12Nm_hf9N1XYAbqaU3mn0MjkOTSxIyhsC/view?usp=sharing, https://drive.google.com/file/d/12jdT5n91-V4FscIkHRQsSKzdGhIC5_6s/view?usp=sharing, https://drive.google.com/file/d/13vQPf8iGRtXBOWirClyy-FXdUTV9CZhK/view?usp=sharing, https://drive.google.com/file/d/1508YzahapRjqOHIc68-TqpaMwUAfNfz1/view?usp=sharing, https://drive.google.com/file/d/16bzUSfbse0vraAURNkz_8jZvDHmMkY6_/view?usp=sharing, https://drive.google.com/file/d/18oYBXfhQsHy3ocHYX09xsKq0tzdp5CJj/view?usp=sharing, https://drive.google.com/file/d/1DybiqLzDYY7TFLxp0E6hfh15gVLTKBP7/view?usp=sharing, https://drive.google.com/file/d/1ESdCXlB8Av2WVDtQD9mgQjmJc0a1DKOg/view?usp=sharing, https://drive.google.com/file/d/1HglzeU42Wle-8zivgrvBoUp1IulJ48TK/view?usp=sharing, https://drive.google.com/file/d/1KuebPvQMP7CRsgMtyotgdkg9EX4BW5xn/view?usp=sharing, https://drive.google.com/file/d/1MvLKJghVzekTKvm79LJ7YkneGyhYJ7Vx/view?usp=sharing, https://aistudio.google.com/app/prompts?state=%7B%22ids%22:%5B%221OZbKtnjoNic-0hqRUYYB97y9dkWfhmNd%22%5D,%22action%22:%22open%22,%22userId%22:%22108686197475781557359%22,%22resourceKeys%22:%7B%7D%7D&usp=sharing, https://drive.google.com/file/d/1T14VCkyeWwoI6u85PgNxfT4tmriYMO9V/view?usp=sharing, https://drive.google.com/file/d/1TbYErXp76xrg0ysisli1IZKyvNr24OLr/view?usp=sharing, https://drive.google.com/file/d/1UfwHYlXj4nDwsao4gtJ9qupMr66xobpY/view?usp=sharing, https://drive.google.com/file/d/1UhznBMiRO_wHnFe0UbCqmxG-fTVC52TF/view?usp=sharing, https://drive.google.com/file/d/1Zi8sqNHSLWI6BEtkauI8HZd7yWFVDETM/view?usp=sharing, https://drive.google.com/file/d/1_P4hdL4hO4AVTvgURwhdSZuqQBph1WCm/view?usp=sharing, https://drive.google.com/file/d/1fRt7YW7ogpDOtZ7J4N1v2oEf7H08YJ9U/view?usp=sharing, https://drive.google.com/file/d/1fiZd4HA6uDF5REOFqorfx0mwHJ9x5Qvg/view?usp=sharing, https://drive.google.com/file/d/1jULOFRYubf2XhZpl42F8f_h6EzJGbZSk/view?usp=sharing, https://drive.google.com/file/d/1kMUZQN6AgHRuVreGSCJ4sUXKrlYoBO3y/view?usp=sharing, https://drive.google.com/file/d/1o8a0IyjCYN3nLJb69xNhhg4D75Oby2HN/view?usp=sharing, https://drive.google.com/file/d/1phItB5qTi2Nm2n9uCB1v822tK_Vp5FoM/view?usp=sharing, https://drive.google.com/file/d/1seFqQ82Qx1oR_peoDzq7gHq68AgLLy5S/view?usp=sharing, https://drive.google.com/file/d/1ud2vHYkSacwlHzvQbfNUoxVade4F0ZOh/view?usp=sharing, https://drive.google.com/file/d/1ws2d7smU132Y4ivn_C4EMeKOodslrXD6/view?usp=sharing, https://drive.google.com/file/d/1xnYgc6FOZW4QlglnMn1KgIVcoAoEAxaR/view?usp=sharing

