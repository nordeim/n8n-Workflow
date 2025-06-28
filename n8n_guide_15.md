# The Complete Guide to n8n Workflow JSON Structure: A Deep Dive with Practical Examples
# https://raw.githubusercontent.com/enescingoz/awesome-n8n-templates/refs/heads/main/PDF_and_Document_Processing/Convert%20URL%20HTML%20to%20Markdown%20Format%20and%20Get%20Page%20Links.txt

## Table of Contents
1. Introduction to n8n JSON Workflows
2. Understanding the Example Workflow: Web Page to Markdown Converter
3. Core JSON Structure Elements
4. Node Types and Configurations
5. Connections and Workflow Flow
6. Metadata and Workflow Properties
7. Advanced Customization Techniques
8. Best Practices and Optimization
9. Resources and Documentation
10. Troubleshooting and Common Patterns

## 1. Introduction to n8n JSON Workflows

n8n is an extendable workflow automation tool that stores all workflows in JSON format. This guide provides a comprehensive understanding of n8n's JSON structure using a real-world example workflow that converts web pages to markdown format using the Firecrawl.dev API.

### Why Understanding JSON Structure Matters

- **Version Control**: Store and track workflow changes in Git
- **Portability**: Share workflows across different n8n instances
- **Programmatic Generation**: Create workflows through code
- **Backup and Recovery**: Export/import workflows easily
- **Debugging**: Understand workflow internals for troubleshooting

### Key Documentation Resources

- **Official n8n Documentation**: https://docs.n8n.io/
- **Workflow Templates**: https://n8n.io/workflows/
- **API Reference**: https://docs.n8n.io/api/
- **Community Forum**: https://community.n8n.io/

## 2. Understanding the Example Workflow: Web Page to Markdown Converter

The provided example workflow demonstrates a practical automation that:
- Accepts URLs from a data source
- Processes them in batches to respect API limits
- Converts HTML content to markdown using Firecrawl.dev
- Extracts links from web pages
- Outputs structured data to a destination

### Workflow Overview

```json
{
  "meta": {
    "instanceId": "6b6a2db47bdf8371d21090c511052883cc9a3f6af5d0d9d567c702d74a18820e"
  },
  "nodes": [...],
  "pinData": {},
  "connections": {...}
}
```

This structure contains four main sections:
- **meta**: Instance identification
- **nodes**: Array of workflow nodes
- **pinData**: Saved test data (empty in this example)
- **connections**: Node relationships and execution flow

## 3. Core JSON Structure Elements

### 3.1 Meta Object

The meta object contains instance-specific information:

```json
"meta": {
  "instanceId": "6b6a2db47bdf8371d21090c511052883cc9a3f6af5d0d9d567c702d74a18820e"
}
```

- **instanceId**: Unique identifier for the n8n instance that created this workflow
- Used for tracking workflow origin and compatibility

### 3.2 Nodes Array

Each node in the workflow has a consistent structure:

```json
{
  "id": "unique-identifier",
  "name": "Human-readable name",
  "type": "node-type-identifier",
  "position": [x, y],
  "parameters": {},
  "typeVersion": 1
}
```

#### Essential Node Properties:

**id**: UUID that uniquely identifies the node within the workflow. Generated automatically but can be customized.

**name**: Display name shown in the UI. Must be unique within the workflow.

**type**: Node type identifier following the pattern `n8n-nodes-base.nodeName`.

**position**: Array `[x, y]` defining the node's visual position on the canvas.

**parameters**: Configuration specific to the node type.

**typeVersion**: Version of the node implementation.

#### Optional Node Properties:

```json
{
  "notes": "Node-specific documentation",
  "notesInFlow": true,
  "credentials": {
    "credentialType": {
      "id": "credential-id",
      "name": "Credential Name"
    }
  },
  "disabled": false,
  "retryOnFail": true,
  "waitBetweenTries": 5000,
  "webhookId": "webhook-unique-id"
}
```

### 3.3 Connections Object

Defines the workflow execution flow:

```json
"connections": {
  "Source Node Name": {
    "main": [
      [
        {
          "node": "Target Node Name",
          "type": "main",
          "index": 0
        }
      ]
    ]
  }
}
```

Connection structure explanation:
- First level: Source node name
- Second level: Output type (usually "main")
- Third level: Array of output pins (supports multiple outputs)
- Fourth level: Array of connections per output
- Connection object: Target node, connection type, and input index

## 4. Node Types and Configurations

### 4.1 Manual Trigger Node

```json
{
  "id": "f4570aad-db25-4dcd-8589-b1c8335935de",
  "name": "When clicking 'Test workflow'",
  "type": "n8n-nodes-base.manualTrigger",
  "position": [-180, 3800],
  "parameters": {},
  "typeVersion": 1
}
```

- **Purpose**: Starts workflow execution manually
- **Parameters**: Usually empty for manual triggers
- **Use Case**: Testing and on-demand execution

### 4.2 Set Node (Data Transformation)

```json
{
  "id": "221a75eb-0bc8-4747-9ec1-1879c46d9163",
  "name": "Example fields from data source",
  "type": "n8n-nodes-base.set",
  "position": [200, 3800],
  "parameters": {
    "options": {},
    "assignments": {
      "assignments": [
        {
          "id": "cc2c6af0-68d3-49eb-85fe-3288d2ed0f6b",
          "name": "Page",
          "type": "array",
          "value": "[\"https://www.automake.io/\", \"https://www.n8n.io/\"]"
        }
      ]
    },
    "includeOtherFields": true
  },
  "typeVersion": 3.4
}
```

Key parameters:
- **assignments**: Define new or modify existing fields
- **includeOtherFields**: Whether to keep fields not explicitly set
- **options**: Additional configuration options

### 4.3 HTTP Request Node

```json
{
  "id": "71c5a0d4-540e-4766-ae99-bdc427019dac",
  "name": "Retrieve Page Markdown and Links",
  "type": "n8n-nodes-base.httpRequest",
  "position": [960, 3820],
  "parameters": {
    "url": "https://api.firecrawl.dev/v1/scrape",
    "method": "POST",
    "options": {},
    "jsonBody": "={\n  \"url\": \"{{ $json.Page }}\",\n  \"formats\" : [\"markdown\", \"links\"]\n}",
    "sendBody": true,
    "sendHeaders": true,
    "specifyBody": "json",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "headerParameters": {
      "parameters": [
        {
          "name": "Content-Type",
          "value": "application/json"
        }
      ]
    }
  },
  "credentials": {
    "httpHeaderAuth": {
      "id": "nbamiF1MDku2NNz7",
      "name": "Firecrawl Bearer"
    }
  },
  "retryOnFail": true,
  "typeVersion": 4.2,
  "waitBetweenTries": 5000
}
```

Important configuration elements:
- **url**: Target API endpoint
- **method**: HTTP method (GET, POST, etc.)
- **jsonBody**: Request body with n8n expressions
- **authentication**: Type of authentication
- **credentials**: Reference to stored credentials
- **retryOnFail**: Automatic retry on failure
- **waitBetweenTries**: Delay between retry attempts

### 4.4 Wait Node

```json
{
  "id": "bd481559-85f2-4865-8d85-e50e72369f26",
  "name": "Wait",
  "type": "n8n-nodes-base.wait",
  "position": [940, 3620],
  "webhookId": "f10708f0-38c6-4c75-b635-37222d5b183a",
  "parameters": {
    "amount": 45
  },
  "typeVersion": 1.1
}
```

- **amount**: Wait time in seconds
- **webhookId**: Used for webhook-resume functionality

### 4.5 Split In Batches Node

```json
{
  "id": "43557ab1-4e52-4598-83a9-e39d5afc6de7",
  "name": "10 at a time",
  "type": "n8n-nodes-base.splitInBatches",
  "position": [740, 3800],
  "parameters": {
    "options": {},
    "batchSize": 10
  },
  "typeVersion": 3
}
```

- **batchSize**: Number of items per batch
- **options**: Additional processing options

### 4.6 Limit Node

```json
{
  "id": "77311c67-f50f-427a-87fd-b29b1f542bbc",
  "name": "40 items at a time",
  "type": "n8n-nodes-base.limit",
  "position": [580, 3800],
  "parameters": {
    "maxItems": 40
  },
  "typeVersion": 1
}
```

- **maxItems**: Maximum number of items to pass through

### 4.7 Split Out Node

```json
{
  "id": "aac948e6-ac86-4cea-be84-f27919d6d936",
  "name": "Split out page URLs",
  "type": "n8n-nodes-base.splitOut",
  "position": [380, 3800],
  "parameters": {
    "options": {},
    "fieldToSplitOut": "Page"
  },
  "typeVersion": 1
}
```

- **fieldToSplitOut**: Field containing array to split into individual items

### 4.8 NoOp Node (Placeholder)

```json
{
  "id": "71c0f975-c0f9-47ae-a245-f852387ad461",
  "name": "Connect to your own data source",
  "type": "n8n-nodes-base.noOp",
  "position": [1380, 3820],
  "parameters": {},
  "typeVersion": 1
}
```

- Used as placeholder for future implementation
- Passes data through without modification

### 4.9 Sticky Note Node

```json
{
  "id": "9ebbd993-9194-40b1-a98e-352eb3a3f9eb",
  "name": "Sticky Note28",
  "type": "n8n-nodes-base.stickyNote",
  "position": [-50.797941767307435, 3729.028866440868],
  "parameters": {
    "color": 7,
    "width": 574.7594700148138,
    "height": 248.90718753310907,
    "content": "**Firecrawl.dev retrieves markdown inc. title, description, links & content. First define the URLs you'd like to scrape**\n"
  },
  "typeVersion": 1
}
```

Sticky note parameters:
- **color**: Visual color (0-7)
- **width/height**: Dimensions in pixels
- **content**: Markdown-formatted text

## 5. Connections and Workflow Flow

### 5.1 Understanding Connection Structure

The example workflow's connections demonstrate various patterns:

```json
"connections": {
  "When clicking 'Test workflow'": {
    "main": [
      [
        {
          "node": "Get urls from own data source",
          "type": "main",
          "index": 0
        }
      ]
    ]
  },
  "10 at a time": {
    "main": [
      null,
      [
        {
          "node": "Retrieve Page Markdown and Links",
          "type": "main",
          "index": 0
        }
      ]
    ]
  }
}
```

### 5.2 Connection Patterns

#### Linear Connection
```json
"Node A": {
  "main": [[{"node": "Node B", "type": "main", "index": 0}]]
}
```

#### Multiple Outputs (Split In Batches)
```json
"Split Node": {
  "main": [
    null,  // First output (iteration complete)
    [{"node": "Process Items", "type": "main", "index": 0}]  // Second output
  ]
}
```

#### Multiple Targets
```json
"Source Node": {
  "main": [
    [
      {"node": "Target 1", "type": "main", "index": 0},
      {"node": "Target 2", "type": "main", "index": 0}
    ]
  ]
}
```

### 5.3 Workflow Execution Flow

The example workflow follows this execution pattern:

1. **Manual Trigger** → **Data Source**
2. **Data Source** → **Set Example Data**
3. **Example Data** → **Split URLs**
4. **Split URLs** → **Limit (40 items)**
5. **Limit** → **Split In Batches (10)**
6. **Batches** → **Wait** → **HTTP Request**
7. **HTTP Request** → **Extract Data**
8. **Extract Data** → **Output Destination**
9. **Loop back** through batches until complete

## 6. Metadata and Workflow Properties

### 6.1 Expression Syntax

n8n uses a powerful expression syntax for dynamic values:

```javascript
// Basic field access
{{ $json.fieldName }}

// Nested field access
{{ $json.data.metadata.title }}

// Array access
{{ $json.links[0].url }}

// Using JavaScript
{{ $json.Page.slice(0, 100) }}

// Conditional expressions
{{ $json.priority === 'high' ? 1 : 0 }}

// Node references
{{ $node["Previous Node"].json.field }}

// Workflow variables
{{ $workflow.id }}
{{ $execution.id }}
{{ $now }}
```

### 6.2 Credential References

Credentials are referenced by ID and name:

```json
"credentials": {
  "httpHeaderAuth": {
    "id": "nbamiF1MDku2NNz7",
    "name": "Firecrawl Bearer"
  }
}
```

Important notes:
- Actual credential values are never exposed in JSON
- Credentials are stored encrypted in the n8n database
- IDs must match existing credentials in the instance

## 7. Advanced Customization Techniques

### 7.1 Dynamic Parameter Values

Use expressions for dynamic configuration:

```json
{
  "parameters": {
    "url": "={{ $env.API_ENDPOINT }}/scrape",
    "batchSize": "={{ $env.BATCH_SIZE || 10 }}",
    "timeout": "={{ $json.priority === 'high' ? 5000 : 30000 }}"
  }
}
```

### 7.2 Error Handling Patterns

Implement robust error handling:

```json
{
  "retryOnFail": true,
  "maxTries": 3,
  "waitBetweenTries": 5000,
  "continueOnFail": false,
  "alwaysOutputData": true
}
```

### 7.3 Workflow Templates

Create reusable workflow patterns:

```javascript
function createBatchProcessor(config) {
  return {
    nodes: [
      {
        id: generateId(),
        name: `Limit to ${config.limit}`,
        type: "n8n-nodes-base.limit",
        parameters: { maxItems: config.limit },
        position: config.positions.limit
      },
      {
        id: generateId(),
        name: `Batch by ${config.batchSize}`,
        type: "n8n-nodes-base.splitInBatches",
        parameters: { batchSize: config.batchSize },
        position: config.positions.batch
      }
    ]
  };
}
```

### 7.4 Custom Functions in Code Nodes

```json
{
  "type": "n8n-nodes-base.code",
  "parameters": {
    "language": "javaScript",
    "code": "// Custom processing logic\nconst items = $input.all();\n\nreturn items.map(item => {\n  return {\n    json: {\n      ...item.json,\n      processed: true,\n      timestamp: new Date().toISOString()\n    }\n  };\n});"
  }
}
```

## 8. Best Practices and Optimization

### 8.1 Performance Optimization

1. **Batch Processing**
   - Use appropriate batch sizes based on API limits
   - Implement pagination for large datasets
   - Monitor memory usage with system resources

2. **Rate Limiting**
   - Respect API rate limits with Wait nodes
   - Implement exponential backoff for retries
   - Use queue systems for high-volume processing

3. **Resource Management**
   - Limit concurrent executions
   - Use streaming for large files
   - Clear unnecessary data between nodes

### 8.2 Workflow Organization

1. **Naming Conventions**
   ```
   - Triggers: "When [event]"
   - Actions: "[Verb] [Object]"
   - Conditions: "If [condition]"
   - Transformations: "[Transform type] Data"
   ```

2. **Visual Organization**
   - Use consistent positioning
   - Group related nodes
   - Add sticky notes for documentation
   - Use colors meaningfully

3. **Modular Design**
   - Create sub-workflows for complex logic
   - Use webhook nodes for inter-workflow communication
   - Implement standard error handling patterns

### 8.3 Security Best Practices

1. **Credential Management**
   - Never hardcode credentials
   - Use environment variables
   - Implement least-privilege access
   - Rotate credentials regularly

2. **Input Validation**
   ```json
   {
     "type": "n8n-nodes-base.if",
     "parameters": {
       "conditions": {
         "string": [
           {
             "value1": "={{ $json.url }}",
             "operation": "regex",
             "value2": "^https?://[\\w\\-._]+\\.[\\w\\-._]+.*$"
           }
         ]
       }
     }
   }
   ```

3. **Output Sanitization**
   - Escape special characters
   - Validate data types
   - Implement size limits

## 9. Resources and Documentation

### 9.1 Official Resources

- **n8n Documentation**: https://docs.n8n.io/
- **Node Reference**: https://docs.n8n.io/integrations/
- **API Documentation**: https://docs.n8n.io/api/
- **Expression Reference**: https://docs.n8n.io/code-examples/expressions/

### 9.2 Community Resources

- **Community Forum**: https://community.n8n.io/
- **Discord Server**: Real-time community support
- **GitHub Repository**: https://github.com/n8n-io/n8n
- **Workflow Templates**: https://n8n.io/workflows/

### 9.3 Learning Resources

**YouTube Channels**:
- n8n Official: Tutorials and feature updates
- Automation tutorials by community members
- Integration-specific guides

**Blog Posts**:
- n8n Blog: https://n8n.io/blog/
- Community tutorials and use cases
- Integration announcements

**Courses and Training**:
- Official n8n tutorials
- Community-created courses
- Video walkthroughs

### 9.4 Useful Tools

1. **JSON Validators**
   - JSONLint for syntax validation
   - JSON Schema validators
   - n8n's built-in import validation

2. **Version Control**
   ```bash
   # Export workflow
   n8n export:workflow --id=<workflow-id> --output=workflow.json
   
   # Import workflow
   n8n import:workflow --input=workflow.json
   ```

3. **Workflow Generators**
   - Template generators
   - Code-based workflow creation
   - CI/CD integration tools

## 10. Troubleshooting and Common Patterns

### 10.1 Common Issues and Solutions

1. **Missing Connections**
   ```json
   // Ensure all nodes are connected
   "connections": {
     "Orphaned Node": {
       "main": [[]]  // Empty connection
     }
   }
   ```

2. **Expression Errors**
   - Check for typos in field names
   - Verify data structure matches expressions
   - Use optional chaining: `{{ $json.field?.subfield }}`

3. **Credential Issues**
   - Verify credential IDs match
   - Check credential permissions
   - Test credentials independently

### 10.2 Debugging Techniques

1. **Pin Data**
   ```json
   "pinData": {
     "Node Name": [
       {
         "json": {
           "test": "data"
         }
       }
     ]
   }
   ```

2. **Execution Logs**
   - Enable verbose logging
   - Check execution history
   - Use console.log in Code nodes

3. **Step-by-Step Testing**
   - Test nodes individually
   - Use manual triggers for debugging
   - Verify data at each step

### 10.3 Common Workflow Patterns

1. **Error Recovery Pattern**
   ```json
   {
     "nodes": [
       {
         "name": "Try Operation",
         "type": "n8n-nodes-base.httpRequest",
         "continueOnFail": true
       },
       {
         "name": "Check Success",
         "type": "n8n-nodes-base.if",
         "parameters": {
           "conditions": {
             "boolean": [
               {
                 "value1": "={{ $json.error }}",
                 "operation": "isEmpty"
               }
             ]
           }
         }
       }
     ]
   }
   ```

2. **Batch Processing Pattern**
   - Split large datasets
   - Process in parallel
   - Aggregate results

3. **Webhook Response Pattern**
   - Immediate acknowledgment
   - Async processing
   - Status updates via callbacks

## Conclusion

Understanding n8n's JSON workflow structure enables powerful automation capabilities. This guide has covered:

- Complete JSON structure breakdown
- Node types and configurations
- Connection patterns and workflow flow
- Best practices for optimization and security
- Resources for continued learning

The example workflow demonstrates practical implementation of these concepts, showing how to:
- Handle API rate limits
- Process data in batches
- Transform web content
- Implement error handling

### Next Steps

1. **Practice**: Import and modify the example workflow
2. **Experiment**: Create custom workflows using these patterns
3. **Share**: Contribute to the n8n community
4. **Learn**: Explore advanced features and integrations

Remember that n8n's flexibility allows for creative solutions to complex automation challenges. Start with simple workflows and gradually increase complexity as you become more comfortable with the JSON structure and expression syntax.

For the latest updates and features, always refer to the official n8n documentation at https://docs.n8n.io/.
