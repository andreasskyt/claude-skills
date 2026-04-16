# n8n Workflow JSON Structure

## Top-Level Schema

```json
{
  "name": "My Workflow",
  "active": false,
  "nodes": [...],
  "connections": {...},
  "settings": {
    "executionOrder": "v1",
    "saveManualExecutions": true,
    "callerPolicy": "workflowsFromSameOwner",
    "errorWorkflow": ""
  },
  "tags": []
}
```

Always set `"active": false` when creating — never activate via API.

---

## Node Schema

```json
{
  "id": "uuid-string",
  "name": "Human Readable Name",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4,
  "position": [250, 300],
  "parameters": {
    // node-specific params
  },
  "credentials": {
    "credentialTypeName": {
      "id": "credential-id",
      "name": "My Credential"
    }
  }
}
```

- `id` — generate a UUID (use `uuidgen` or any uuid v4)
- `name` — must be unique within the workflow; used as connection keys
- `type` — full node type string including namespace
- `typeVersion` — use the latest version for each node type (see nodes.md)
- `position` — canvas coordinates [x, y]

---

## Connections Schema

Connections map source node outputs to target node inputs:

```json
{
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
}
```

- The outer key is the **source node name** (must match exactly)
- `"main"` is the output type (always `"main"` for standard flow; `"ai_tool"` etc. for AI nodes)
- The array of arrays: outer = output branches, inner = connections from that branch
- `index` = which input on the target node (usually 0)

### Branching Example (IF node with two outputs)

```json
"connections": {
  "Check Condition": {
    "main": [
      [{ "node": "True Path", "type": "main", "index": 0 }],
      [{ "node": "False Path", "type": "main", "index": 0 }]
    ]
  }
}
```

---

## Trigger Node Patterns

### Manual Trigger
```json
{
  "name": "Manual Trigger",
  "type": "n8n-nodes-base.manualTrigger",
  "typeVersion": 1,
  "position": [0, 0],
  "parameters": {}
}
```

### Webhook Trigger
```json
{
  "name": "Webhook",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [0, 0],
  "parameters": {
    "httpMethod": "POST",
    "path": "my-webhook-path",
    "responseMode": "onReceived",
    "responseData": "allEntries"
  }
}
```

### Schedule Trigger
```json
{
  "name": "Schedule",
  "type": "n8n-nodes-base.scheduleTrigger",
  "typeVersion": 1.2,
  "position": [0, 0],
  "parameters": {
    "rule": {
      "interval": [
        {
          "field": "cronExpression",
          "expression": "0 9 * * 1-5"
        }
      ]
    }
  }
}
```

---

## Data Model

n8n passes data between nodes as **items** — an array of objects:

```json
[
  { "json": { "field": "value", "count": 42 } },
  { "json": { "field": "other", "count": 7 } }
]
```

- Each item has a `json` key containing the data
- Nodes process all items in the array
- Use `{{ $json.fieldName }}` in expressions to reference current item data
- Use `{{ $node["Node Name"].json.fieldName }}` to reference another node's output
- Use `{{ $items("Node Name") }}` to get all items from a specific node

---

## Common Expressions

| Expression | Returns |
|------------|---------|
| `{{ $json.fieldName }}` | Field from current item |
| `{{ $json["field-with-dashes"] }}` | Field with special chars |
| `{{ $node["HTTP Request"].json.id }}` | Field from named node |
| `{{ $now.toISO() }}` | Current timestamp ISO string |
| `{{ $today.format("YYYY-MM-DD") }}` | Today's date formatted |
| `{{ $runIndex }}` | Current loop iteration index |
| `{{ $items().length }}` | Number of items in current batch |
| `{{ JSON.stringify($json) }}` | Stringify current item |
