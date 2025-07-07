# The Authoritative Guide to n8n Workflow JSON

This guide serves as the definitive, source-of-truth reference for creating, understanding, and manipulating n8n workflows directly in their JSON format. It synthesizes best practices, technical specifications, and real-world examples to empower developers to build robust, scalable, and maintainable automations.

## Table of Contents

1.  [**Introduction: Why Master n8n JSON?**](#1-introduction-why-master-n8n-json)
2.  [**Core Workflow Structure**](#2-core-workflow-structure)
3.  [**Detailed Breakdown of JSON Elements**](#3-detailed-breakdown-of-json-elements)
    *   3.1. Workflow Root Properties
    *   3.2. Node Properties Explained
    *   3.3. Connections Object Explained
    *   3.4. Workflow-Level Settings (`settings` object)
    *   3.5. Static Data and Pin Data
4.  [**Node Types and Their Configurations in Detail**](#4-node-types-and-their-configurations-in-detail)
    *   4.1. Trigger Nodes
    *   4.2. Data Transformation and Logic Nodes
    *   4.3. Integration and API Nodes
    *   4.4. AI and LangChain Nodes
5.  [**Dynamic Operation: The Expression System**](#5-dynamic-operation-the-expression-system)
6.  [**Working with Credentials**](#6-working-with-credentials)
7.  [**Advanced Workflow Patterns**](#7-advanced-workflow-patterns)
8.  [**Best Practices for Production-Ready Workflows**](#8-best-practices-for-production-ready-workflows)
9.  [**Troubleshooting and Debugging Techniques**](#9-troubleshooting-and-debugging-techniques)
10. [**Resources and Further Learning**](#10-resources-and-further-learning)
11. [**Conclusion and Next Steps**](#11-conclusion-and-next-steps)

---

## 1. Introduction: Why Master n8n JSON?

n8n is a powerful, open-source, and self-hostable workflow automation tool. While its visual editor is intuitive, the underlying JSON representation of every workflow is the key to unlocking advanced capabilities.

Mastering the JSON format is crucial for:
*   **Version Control**: Storing workflows in Git, enabling change tracking, pull requests, and reliable rollbacks.
*   **Portability & Reproducibility**: Easily sharing workflows and deploying them consistently across different environments (dev, staging, prod).
*   **Programmatic Creation & Modification**: Generating workflows dynamically, applying bulk updates, or building tools that manipulate workflows.
*   **Advanced Debugging**: Understanding the precise configuration and data flow to troubleshoot complex issues that may not be obvious in the UI.
*   **Unlocking Hidden Features**: Directly setting certain parameters (e.g., `retryOnFail`, `alwaysOutputData`) that may be less accessible through the UI.

This guide will provide you with the comprehensive knowledge to read, write, and manage any n8n workflow directly in JSON.

---

## 2. Core Workflow Structure

At its highest level, an n8n workflow is a JSON object with a few essential top-level keys.

```json
{
  "name": "My First Workflow",
  "nodes": [
    {
      "parameters": {},
      "id": "5f6b4d3d-4f1e-4a6c-8b2b-1d7f2e9c5a1a",
      "name": "Start",
      "type": "n8n-nodes-base.start",
      "typeVersion": 1,
      "position": [ 250, 300 ]
    }
  ],
  "connections": {},
  "active": false,
  "settings": {},
  "id": "1",
  "tags": []
}
```

*   **`name`**: The human-readable name of the workflow.
*   **`nodes`**: An array of node objects, each representing a single trigger, action, or logic step. This is the heart of the workflow.
*   **`connections`**: An object defining the directed graph of data flow between nodes.
*   **`active`**: A boolean flag indicating if the workflow is enabled to run automatically on triggers.
*   **`settings`**: An object for workflow-level configurations like timezone, error handling, and execution timeouts.
*   **`id`**: The unique identifier for the workflow within an n8n instance.
*   **`tags`**: An array of strings or objects used for organizing and filtering workflows.
*   Other metadata fields like `versionId`, `createdAt`, `updatedAt`, and `meta` are often present for versioning and instance tracking.

---

## 3. Detailed Breakdown of JSON Elements

### 3.1. Workflow Root Properties

Beyond the core keys, you may encounter:
*   `versionId`: A UUID that tracks the specific version of a saved workflow, crucial for concurrency control.
*   `meta`: Contains instance-specific information, such as the `instanceId` from which it was exported.
*   `pinData`: Stores data that has been "pinned" in the UI for testing and development.
*   `staticData`: A key-value store for data that needs to persist across executions of the same workflow, such as the state for an `executeOnce` node.

### 3.2. Node Properties Explained

Each object within the `nodes` array defines a single operational block on the canvas.

| Key | Example | Description & Importance |
| :--- | :--- | :--- |
| **`id`** | `"e49f8b4d-9e99-..."` | **Unique Identifier.** A UUID for the node, used internally and in the `connections` object. Must be unique within the workflow. |
| **`name`** | `"Validate & Clean Data"` | **Human-Readable Name.** Displayed in the UI. Must be unique for expression-based referencing (`$node["Node Name"]`). |
| **`type`** | `"n8n-nodes-base.httpRequest"` | **Node's Class.** Identifies the node's function. Follows the pattern `<package>.<nodeName>`. |
| **`typeVersion`** | `4.1` | **Node Version.** Locks the node's structure to a specific version of its constructor, ensuring backward compatibility. |
| **`position`** | `[460, 300]` | **UI Coordinates.** An `[x, y]` array for visual placement on the canvas. Purely cosmetic. |
| **`parameters`** | `{ "url": "...", "method": "POST" }` | **Configuration Heart.** An object containing all the node-specific settings you configure in the UI. Its structure is unique to each node `type`. |
| **`credentials`** | `{ "httpHeaderAuth": { "id": "..." } }` | **Security Reference.** References securely stored credentials by ID and name. **Actual secrets are never in this JSON.** |
| `disabled` | `true` | (Optional) If true, the node is skipped during execution. |
| `notes` | `"This node enriches lead data..."` | (Optional) Documentation that appears in the node's description. |
| `notesInFlow`| `true` | (Optional) If true, `notes` content is displayed on the canvas as a sticky note. |
| `retryOnFail` | `{ "maxTries": 3, "waitBetweenTries": 1000 }` | (Optional) Enables automatic retries on failure. `waitBetweenTries` is in milliseconds. |
| `continueOnFail` | `true` | (Optional) If true, the workflow continues even if this node fails, passing the error to the output. |
| `executeOnce`| `true` | (Optional) Ensures a node runs only the first time it's reached with a given set of input data items. |
| `webhookId` | `"lead-intake-webhook"` | (Optional) Specific to Webhook nodes, this is part of the external URL token. |

### 3.3. Connections Object Explained

The `connections` object defines the **directed acyclic graph (DAG)** of the workflow.

```json
"connections": {
  "Source Node Name": {
    "outputType": [
      [
        { "node": "Target Node Name", "type": "connectionType", "index": 0 }
      ]
    ]
  }
}
```

*   **First Level Key**: The `name` of the **source node**.
*   **Second Level Key**: The **output type**. The most common is `"main"`. Modern AI nodes introduce specialized types like `ai_languageModel`, `ai_memory`, and `ai_tool`.
*   **Third Level (Array of Arrays)**: The outer array represents different output "pins" of the source node. The inner array allows a single output pin to connect to multiple target nodes.
*   **Connection Object**:
    *   `"node"`: The `name` of the **target node**.
    *   `"type"`: The type of input on the target node, almost always `"main"`.
    *   `"index"`: The input pin index on the target node. `0` for the first input, `1` for the second (e.g., on a `Merge` node).

**Example: Branching from an IF node**
```json
"connections": {
  "Check Priority": {
    "main": [
      [ { "node": "High Priority Action", "type": "main", "index": 0 } ],
      [ { "node": "Standard Action", "type": "main", "index": 0 } ]
    ]
  }
}
```
Here, `main[0]` is the "true" path (output pin 0), and `main[1]` is the "false" path (output pin 1).

### 3.4. Workflow-Level Settings (`settings` object)

This object contains global configurations that affect the entire workflow.

| Key | Example | Description |
| :--- | :--- | :--- |
| **`timezone`** | `"America/New_York"` | Sets the default timezone for CRON triggers and date/time operations. **Highly recommended to set explicitly.** |
| **`errorWorkflow`** | `"workflow-id-of-error-handler"` | The `id` of a separate workflow to execute if an unhandled error occurs. |
| **`executionOrder`** | `"v1"` | Defines the algorithm for node execution sequence (e.g., "v0" legacy, "v1" improved). |
| **`saveDataSuccessExecution`** | `"all"` | Configures data saving on success (`"all"`, `"none"`, or `"lastNode"`). |
| **`saveDataErrorExecution`** | `"all"` | Configures data saving on error. |
| **`saveManualExecutions`** | `true` | Whether to persist data from manual test runs. |
| **`executionTimeout`** | `3600` | Sets the maximum execution time for the workflow in seconds. |
| **`callerPolicy`** | `"workflowsFromSameOwner"` | Controls which other workflows can trigger this one as a sub-workflow. |

### 3.5. Static Data and Pin Data

*   **`staticData`**: An object for runtime storage that persists across executions. For example, n8n internally stores the state for `executeOnce` nodes here. It is rarely edited manually.
*   **`pinData`**: Stores information about data pinned in the UI during development for testing. It's recommended to remove this before deploying to production to keep the workflow JSON clean.

---

## 4. Node Types and Their Configurations in Detail

### 4.1. Trigger Nodes

Triggers are the entry points that start a workflow.

*   **Webhook Trigger (`n8n-nodes-base.webhook`)**: Listens for incoming HTTP requests.
    *   `httpMethod`: `"POST"`, `"GET"`, etc.
    *   `path`: The custom URL path (e.g., `"customer-signup"`).
    *   `responseMode`: `"onReceived"` or `"lastNode"` or `"responseNode"`.
    *   `authentication`: Can be configured for basic auth, header auth, etc.
*   **Schedule Trigger (`n8n-nodes-base.cron`)**: Runs on a schedule.
    *   `mode`: `everyDay`, `everyX`, `custom`.
    *   `cronExpression`: A standard CRON string (e.g., `"0 9 * * 1-5"` for 9 AM on weekdays).
*   **Form Trigger (`n8n-nodes-base.formTrigger`)**: Creates a simple web form.
    *   `formFields.values[]`: An array defining form fields (`label`, `fieldType`, `requiredField`).
*   **Error Trigger (`n8n-nodes-base.errorTrigger`)**: Starts a workflow when another workflow fails. Must be present in the workflow designated as the `errorWorkflow` in settings.

### 4.2. Data Transformation and Logic Nodes

These nodes are the workhorses for manipulating data and controlling flow.

*   **Set (`n8n-nodes-base.set`)**: The primary tool for adding, modifying, or removing fields.
    *   `values`: Defines the fields to set, categorized by data type (`string`, `number`, `boolean`).
    *   `keepOnlySet`: If `true`, discards all properties except those explicitly set, excellent for cleaning up data.
*   **Code (`n8n-nodes-base.function`)**: Executes custom JavaScript for complex logic.
    *   Provides access to special variables: `$input`, `items`, `$json`, `$node`, `$workflow`, `$execution`, `$credentials`.
    *   Must return an array of n8n data items: `[{ json: {...}, binary: {...} }]`.
*   **IF (`n8n-nodes-base.if`)**: Implements conditional branching.
    *   `conditions`: An array of comparisons (`value1`, `operation`, `value2`).
    *   `combinator`: `"and"` or `"or"` logic for combining conditions.
    *   Outputs: `main[0]` for true, `main[1]` for false.
*   **Switch (`n8n-nodes-base.switch`)**: Routes execution to one of several branches based on an input value.
*   **Merge (`n8n-nodes-base.merge`)**: Combines data from multiple incoming branches.
    *   `mode`: `mergeByIndex` (aligns items 1:1), `append` (concatenates item lists), etc.
*   **Split In Batches (`n8n-nodes-base.splitInBatches`)**: Breaks a large array into smaller chunks for iterative processing, essential for handling API rate limits and managing memory.

### 4.3. Integration and API Nodes

*   **HTTP Request (`n8n-nodes-base.httpRequest`)**: The universal tool for interacting with any REST API.
    *   `authentication`: Supports various auth methods, including referencing stored credentials.
    *   `options.splitIntoItems`: If an API returns an array, this option turns each element into a separate n8n item for downstream processing.
*   **Service-Specific Nodes** (e.g., `n8n-nodes-base.slack`, `n8n-nodes-base.googleSheets`): These nodes provide a user-friendly interface for common operations on popular services, but under the hood, they are often wrappers around the service's API. Their `parameters` object will mirror the options available in the UI.

### 4.4. AI and LangChain Nodes

n8n's AI capabilities, powered by LangChain, are a composite of several specialized node types.

*   **Agent (`@n8n/n8n-nodes-langchain.agent`)**: The central coordinator. It receives a user prompt and decides which tools to use.
    *   `options.systemMessage`: The master prompt defining the agent's persona, goals, and rules.
*   **Language Model** (e.g., `@n8n/n8n-nodes-langchain.lmChatOpenAi`): Provides the reasoning engine (the "brain").
*   **Memory** (e.g., `@n8n/n8n-nodes-langchain.memoryBufferWindow`): Provides conversation history and context.
*   **Tool** (e.g., `n8n-nodes-base.httpRequestTool` or `@n8n/n8n-nodes-langchain.toolWorkflow`): An action the agent can perform.
    *   `toolDescription`: A natural language description of what the tool does, which the agent uses for decision-making.

---

## 5. Dynamic Operation: The Expression System

Expressions are what make n8n workflows dynamic. They are JavaScript-based snippets enclosed in `{{ }}`.

*   **Accessing Data**:
    *   `{{ $json.fieldName }}`: Access a field from the current item.
    *   `{{ $node["Node Name"].json.field }}`: Access data from a specific, previously executed node by its **unique name**.
    *   `{{ $items("Node Name", 0, 0).json.field }}`: More specific item access.
*   **Environment Variables**:
    *   `{{ $env.YOUR_VARIABLE_NAME }}`: Securely access environment variables. Essential for separating configuration from logic.
*   **Special Variables**:
    *   `{{ $now }}` / `{{ $today }}`: Current date/time objects (Luxon).
    *   `{{ $workflow.id }}` / `{{ $execution.id }}`: Metadata about the current run.
*   **The `$fromAI()` Expression**:
    *   A special placeholder used *only in tool nodes* for AI agents.
    *   It signals to the agent, "You need to determine the value for this parameter based on the conversation."
    *   Example: `{{ $fromAI('url', 'The full URL the user wants to navigate to', 'string') }}`.

---

## 6. Working with Credentials

Security is paramount. n8n's credential management system ensures that secrets are never stored directly in the workflow JSON.

*   **Creation**: Credentials (API keys, OAuth2 tokens, etc.) are created and encrypted in the n8n UI or via the API.
*   **Reference in JSON**: A node's `credentials` object only contains a reference to the credential's internal `id` and `name`.
    ```json
    "credentials": {
      "slackApi": {
        "id": "slack-bot-token-id",
        "name": "Production Slack Bot"
      }
    }
    ```
*   **Best Practice**: Use different credential sets for development, staging, and production. n8n Cloud's "Credential Overrides" feature simplifies this. Never hard-code tokens or keys in `parameters`.

---

## 7. Advanced Workflow Patterns

*   **Error Handling**: Combine node-level `continueOnFail` with a global `errorWorkflow` for comprehensive coverage. An error workflow should contain an `Error Trigger` node and can log details to a database, send a Slack/email alert, and even attempt recovery actions.
*   **Looping and Iteration**: The `SplitInBatches` node is the standard for processing array items one by one or in small batches. Its second output can be looped back to its input to create a `do-while` pattern.
*   **Modular Design (Sub-Workflows)**: Break down large, complex logic into smaller, reusable "sub-workflows". The main workflow can call these using the `Execute Workflow` node. This is the cornerstone of building scalable and maintainable automations.
*   **AI Agent Tooling**: A powerful pattern is to use an `Execute Workflow` node as a tool for an AI agent (via the `@n8n/n8n-nodes-langchain.toolWorkflow` node). This allows an agent to trigger complex, multi-step actions that are encapsulated in another workflow.
*   **Performance Optimization**:
    *   Use `Set` nodes with `keepOnlySet: true` to prune unnecessary data between steps, reducing memory footprint.
    *   For high-volume triggers, use a queueing system (like Redis, integrated into n8n's scaling options) to process executions asynchronously.
    *   For large binary files, configure `N8N_DEFAULT_BINARY_DATA_MODE=filesystem` to store them on disk instead of in the database, preventing memory bloat.

---

## 8. Best Practices for Production-Ready Workflows

1.  **Idempotency**: Design operations to be safely repeatable. Use unique transaction IDs. For database inserts, use `ON CONFLICT DO NOTHING` or `ON CONFLICT DO UPDATE`. Use the `executeOnce` node property for actions that must only happen once per input.
2.  **Naming Conventions**: Use clear, descriptive names for nodes and workflows. Prefix nodes by function (e.g., "API: Fetch Users", "Transform: Normalize Data").
3.  **Version Control**: Store all workflow JSON files in a Git repository. Document changes via commit messages.
4.  **Documentation as Code**: Use `Sticky Note` nodes on the canvas and the `notes` property within nodes to explain complex logic for future maintainers.
5.  **Environment Separation**: Maintain separate n8n instances (or use environment-specific credentials/variables) for development, staging, and production.
6.  **Fail Fast**: Validate all incoming data at the beginning of a workflow (e.g., in a `Code` node or with `IF` nodes) and throw an error immediately if the data is invalid.

---

## 9. Troubleshooting and Debugging Techniques

*   **Execution Log**: The first place to look. In the n8n UI, the "Executions" panel shows a log for each run, highlighting which node failed and the error message.
*   **Manual Execution & Pinned Data**: Run a workflow manually from the editor. You can "pin" the output data of any node to use it as a static input for subsequent test runs, allowing you to debug a specific part of the flow without re-running the beginning.
*   **"Chain of Thought" for AI Agents**: When debugging an agent, set the `returnIntermediateSteps` option to `true`. This will expose the agent's step-by-step reasoning in the output, showing which tools it chose and why.
*   **Dry-Run Technique**: Temporarily disable all nodes with side effects (e.g., sending emails, writing to a database) to safely inspect the data flow and logic.

---

## 10. Resources and Further Learning

### Official n8n Resources
*   **n8n Documentation Home**: [https://docs.n8n.io/](https://docs.n8n.io/)
*   **n8n Workflow Templates**: [https://n8n.io/workflows/](https://n8n.io/workflows/)
*   **n8n Blog**: [https://n8n.io/blog/](https://n8n.io/blog/)
*   **n8n YouTube Channel**: [https://www.youtube.com/@n8n-io](https://www.youtube.com/@n8n-io)

### Community Resources
*   **n8n Community Forum**: [https://community.n8n.io/](https://community.n8n.io/)
*   **n8n Discord Server**: [https://discord.gg/n8n](https://discord.gg/n8n)
*   **Awesome n8n (GitHub)**: A curated list of tools, resources, and custom nodes.

---

## 11. Conclusion and Next Steps

Mastering n8n's JSON format transforms you from a user into an architect of automation. You gain the ability to build with precision, collaborate effectively, and create systems that are robust, scalable, and maintainable.

**Key Takeaways**:
*   The JSON structure, centered around `nodes` and `connections`, is a direct representation of the visual graph.
*   AI agent workflows introduce specialized connection types (`ai_tool`, `ai_memory`, `ai_languageModel`) and the powerful `$fromAI()` expression.
*   Modular design using sub-workflows is essential for managing complexity and promoting reusability.
*   Security and reliability depend on proper credential management, error handling, and idempotent design patterns.

Your next step is to practice. Take an existing workflow from your n8n instance, export it, and study its JSON. Try making a small change directly in the file and re-importing it. By bridging the gap between the visual editor and the underlying code, you unlock the full potential of n8n.

---
https://drive.google.com/file/d/1-2z6hvI-aulXTTB5R-pE9xibu0ou6KI-/view?usp=sharing, https://drive.google.com/file/d/1-UX0QG1PuINPTj_sxdMipIZMJa0ZRpcX/view?usp=sharing, https://drive.google.com/file/d/10UoQY_65jWwoEQ3qtyjk9x_IqatLsKHT/view?usp=sharing, https://drive.google.com/file/d/11AtrmTXVfXv5OUutB70uYBRoe_iESJta/view?usp=sharing, https://drive.google.com/file/d/11VTwv9sJSGCg8hBiapB4nn7FXbi4KOdm/view?usp=sharing, https://drive.google.com/file/d/12Nm_hf9N1XYAbqaU3mn0MjkOTSxIyhsC/view?usp=sharing, https://drive.google.com/file/d/12jdT5n91-V4FscIkHRQsSKzdGhIC5_6s/view?usp=sharing, https://drive.google.com/file/d/13vQPf8iGRtXBOWirClyy-FXdUTV9CZhK/view?usp=sharing, https://drive.google.com/file/d/1508YzahapRjqOHIc68-TqpaMwUAfNfz1/view?usp=sharing, https://drive.google.com/file/d/16bzUSfbse0vraAURNkz_8jZvDHmMkY6_/view?usp=sharing, https://drive.google.com/file/d/18oYBXfhQsHy3ocHYX09xsKq0tzdp5CJj/view?usp=sharing, https://drive.google.com/file/d/1DybiqLzDYY7TFLxp0E6hfh15gVLTKBP7/view?usp=sharing, https://drive.google.com/file/d/1ESdCXlB8Av2WVDtQD9mgQjmJc0a1DKOg/view?usp=sharing, https://drive.google.com/file/d/1HglzeU42Wle-8zivgrvBoUp1IulJ48TK/view?usp=sharing, https://drive.google.com/file/d/1KuebPvQMP7CRsgMtyotgdkg9EX4BW5xn/view?usp=sharing, https://drive.google.com/file/d/1MvLKJghVzekTKvm79LJ7YkneGyhYJ7Vx/view?usp=sharing, https://aistudio.google.com/app/prompts?state=%7B%22ids%22:%5B%221OZbKtnjoNic-0hqRUYYB97y9dkWfhmNd%22%5D,%22action%22:%22open%22,%22userId%22:%22108686197475781557359%22,%22resourceKeys%22:%7B%7D%7D&usp=sharing, https://drive.google.com/file/d/1T14VCkyeWwoI6u85PgNxfT4tmriYMO9V/view?usp=sharing, https://drive.google.com/file/d/1TbYErXp76xrg0ysisli1IZKyvNr24OLr/view?usp=sharing, https://drive.google.com/file/d/1UfwHYlXj4nDwsao4gtJ9qupMr66xobpY/view?usp=sharing, https://drive.google.com/file/d/1UhznBMiRO_wHnFe0UbCqmxG-fTVC52TF/view?usp=sharing, https://drive.google.com/file/d/1Zi8sqNHSLWI6BEtkauI8HZd7yWFVDETM/view?usp=sharing, https://drive.google.com/file/d/1_P4hdL4hO4AVTvgURwhdSZuqQBph1WCm/view?usp=sharing, https://drive.google.com/file/d/1fRt7YW7ogpDOtZ7J4N1v2oEf7H08YJ9U/view?usp=sharing, https://drive.google.com/file/d/1fiZd4HA6uDF5REOFqorfx0mwHJ9x5Qvg/view?usp=sharing, https://drive.google.com/file/d/1jULOFRYubf2XhZpl42F8f_h6EzJGbZSk/view?usp=sharing, https://drive.google.com/file/d/1kMUZQN6AgHRuVreGSCJ4sUXKrlYoBO3y/view?usp=sharing, https://drive.google.com/file/d/1o8a0IyjCYN3nLJb69xNhhg4D75Oby2HN/view?usp=sharing, https://drive.google.com/file/d/1phItB5qTi2Nm2n9uCB1v822tK_Vp5FoM/view?usp=sharing, https://drive.google.com/file/d/1seFqQ82Qx1oR_peoDzq7gHq68AgLLy5S/view?usp=sharing, https://drive.google.com/file/d/1ud2vHYkSacwlHzvQbfNUoxVade4F0ZOh/view?usp=sharing, https://drive.google.com/file/d/1ws2d7smU132Y4ivn_C4EMeKOodslrXD6/view?usp=sharing, https://drive.google.com/file/d/1xnYgc6FOZW4QlglnMn1KgIVcoAoEAxaR/view?usp=sharing

