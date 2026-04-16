# n8n Common Nodes Reference

## Core / Flow Control

### HTTP Request
```json
{
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "parameters": {
    "method": "GET",
    "url": "https://api.example.com/endpoint",
    "authentication": "none",
    "headers": {
      "parameters": [
        { "name": "Authorization", "value": "Bearer token" }
      ]
    },
    "queryParameters": {
      "parameters": [
        { "name": "key", "value": "value" }
      ]
    },
    "options": {
      "response": { "response": { "responseFormat": "json" } }
    }
  }
}
```

### Code Node
```json
{
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "parameters": {
    "jsCode": "// $input.all() returns all items\n// Return array of { json: {...} } objects\nconst items = $input.all();\nreturn items.map(item => ({\n  json: {\n    ...item.json,\n    processed: true\n  }\n}));"
  }
}
```

### Set (reshape/rename data)
```json
{
  "type": "n8n-nodes-base.set",
  "typeVersion": 3.4,
  "parameters": {
    "mode": "manual",
    "assignments": {
      "assignments": [
        {
          "id": "uuid",
          "name": "outputField",
          "value": "={{ $json.inputField }}",
          "type": "string"
        }
      ]
    },
    "options": {}
  }
}
```

### IF (branch on condition)
```json
{
  "type": "n8n-nodes-base.if",
  "typeVersion": 2,
  "parameters": {
    "conditions": {
      "options": { "caseSensitive": true, "leftValue": "", "typeValidation": "strict" },
      "conditions": [
        {
          "id": "uuid",
          "leftValue": "={{ $json.status }}",
          "rightValue": "active",
          "operator": { "type": "string", "operation": "equals" }
        }
      ],
      "combinator": "and"
    },
    "options": {}
  }
}
```
Outputs: index 0 = true branch, index 1 = false branch

### Switch (multi-branch)
```json
{
  "type": "n8n-nodes-base.switch",
  "typeVersion": 3,
  "parameters": {
    "mode": "rules",
    "rules": {
      "rules": [
        {
          "conditions": {
            "conditions": [
              {
                "leftValue": "={{ $json.type }}",
                "rightValue": "email",
                "operator": { "type": "string", "operation": "equals" }
              }
            ]
          },
          "renameOutput": true,
          "outputKey": "Email"
        }
      ]
    },
    "fallbackOutput": "none"
  }
}
```

### Merge
```json
{
  "type": "n8n-nodes-base.merge",
  "typeVersion": 3,
  "parameters": {
    "mode": "combine",
    "combineBy": "combineAll",
    "options": {}
  }
}
```

### Split In Batches
```json
{
  "type": "n8n-nodes-base.splitInBatches",
  "typeVersion": 3,
  "parameters": {
    "batchSize": 10,
    "options": {}
  }
}
```

### Wait
```json
{
  "type": "n8n-nodes-base.wait",
  "typeVersion": 1.1,
  "parameters": {
    "resume": "timeInterval",
    "unit": "seconds",
    "amount": 30
  }
}
```

---

## Integrations

### Slack
```json
{
  "type": "n8n-nodes-base.slack",
  "typeVersion": 2.2,
  "parameters": {
    "resource": "message",
    "operation": "post",
    "channel": {
      "__rl": true,
      "value": "#general",
      "mode": "name"
    },
    "text": "Hello from n8n!"
  },
  "credentials": {
    "slackApi": { "id": "cred-id", "name": "Slack" }
  }
}
```

### Gmail
```json
{
  "type": "n8n-nodes-base.gmail",
  "typeVersion": 2.1,
  "parameters": {
    "resource": "message",
    "operation": "send",
    "sendTo": "={{ $json.email }}",
    "subject": "Subject here",
    "message": "Body here",
    "options": {}
  },
  "credentials": {
    "gmailOAuth2": { "id": "cred-id", "name": "Gmail" }
  }
}
```

### Google Sheets
```json
{
  "type": "n8n-nodes-base.googleSheets",
  "typeVersion": 4.5,
  "parameters": {
    "resource": "sheet",
    "operation": "appendOrUpdate",
    "documentId": { "__rl": true, "value": "spreadsheet-id", "mode": "id" },
    "sheetName": { "__rl": true, "value": "Sheet1", "mode": "name" },
    "columns": {
      "mappingMode": "autoMapInputData",
      "value": {}
    }
  },
  "credentials": {
    "googleSheetsOAuth2Api": { "id": "cred-id", "name": "Google Sheets" }
  }
}
```

### Notion
```json
{
  "type": "n8n-nodes-base.notion",
  "typeVersion": 2.2,
  "parameters": {
    "resource": "page",
    "operation": "create",
    "databaseId": { "__rl": true, "value": "database-id", "mode": "id" },
    "title": "={{ $json.title }}",
    "propertiesUi": {
      "propertyValues": []
    }
  },
  "credentials": {
    "notionApi": { "id": "cred-id", "name": "Notion" }
  }
}
```

### Airtable
```json
{
  "type": "n8n-nodes-base.airtable",
  "typeVersion": 2.1,
  "parameters": {
    "resource": "record",
    "operation": "create",
    "base": { "__rl": true, "value": "base-id", "mode": "id" },
    "table": { "__rl": true, "value": "table-id", "mode": "id" },
    "fieldsUi": {
      "fieldValues": [
        { "fieldId": "Name", "fieldValue": "={{ $json.name }}" }
      ]
    }
  },
  "credentials": {
    "airtableTokenApi": { "id": "cred-id", "name": "Airtable" }
  }
}
```

### OpenAI
```json
{
  "type": "@n8n/n8n-nodes-langchain.openAi",
  "typeVersion": 1.8,
  "parameters": {
    "resource": "text",
    "operation": "message",
    "modelId": { "__rl": true, "value": "gpt-4o", "mode": "list" },
    "messages": {
      "values": [
        { "role": "user", "content": "={{ $json.prompt }}" }
      ]
    }
  },
  "credentials": {
    "openAiApi": { "id": "cred-id", "name": "OpenAI" }
  }
}
```

---

## Code Node Runtime — Critical Details

The Code node runs JavaScript (ES2020+). These are the real rules learned from production:

### HTTP requests inside Code nodes
**Do not use `fetch` or `axios`.** Use `this.helpers.request`:

```javascript
// json: true = auto-serializes body AND auto-parses response
const result = await this.helpers.request({
  method: 'POST',
  uri: 'https://api.example.com/endpoint',
  headers: { 'Authorization': 'Bearer token' },
  body: { key: 'value' },
  json: true
});

// json: false + JSON.stringify = sends raw string body (required for Anthropic API)
const result = await this.helpers.request({
  method: 'POST',
  uri: 'https://api.anthropic.com/v1/messages',
  headers: {
    'x-api-key': apiKey,
    'anthropic-version': '2023-06-01',
    'content-type': 'application/json'
  },
  body: JSON.stringify({ model: 'claude-sonnet-4-20250514', max_tokens: 1024, messages }),
  json: false
});
const parsed = JSON.parse(result);
```

### Input/output
```javascript
// Single item
const item = $input.first().json;

// All items
const items = $input.all();

// Must return array of items
return [{ json: { result: 'value' } }];

// Or multiple items
return items.map(item => ({ json: { ...item.json, processed: true } }));
```

### Inside a loop node
Each iteration passes one item — use `$input.first().json`, not `$items(...)`.
`$items('Node Name')` inside a loop only returns the current iteration's data.

### Exponential backoff retry pattern (use for external API calls)
```javascript
const MAX_RETRIES = 4;
let delay = 10000;
for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
  try {
    const response = await this.helpers.request({ method: 'POST', uri: '...', json: true });
    // success — break out
    return [{ json: response }];
  } catch (err) {
    if (attempt === MAX_RETRIES) throw err;
    await new Promise(r => setTimeout(r, delay));
    delay *= 2;
  }
}
```

---

## Loop Node — Connection Structure

Loop nodes have two outputs:
- `main[0]` = **done** (fires after all items exhausted)
- `main[1]` = **loop body** (fires for each item)

```json
"connections": {
  "Loop Over Items": {
    "main": [
      [{ "node": "Post-Loop Node", "type": "main", "index": 0 }],
      [{ "node": "Process Each Item", "type": "main", "index": 0 }]
    ]
  }
}
```

---

## Tips

- Always check the credential ID by calling `GET /api/v1/credentials` before wiring credentials
- `typeVersion` matters — wrong version can break parameter parsing
- The `__rl` pattern (resource locator) is n8n's way of resolving IDs vs names — always include it for resource references
- Use the Code node for anything complex that doesn't have a native node
- Workflow-level `status: "success"` does NOT mean all nodes succeeded — individual nodes can error while the workflow reports success. Always check per-node errors when debugging
