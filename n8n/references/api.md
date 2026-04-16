# n8n API Reference

Base URL: `$N8N_BASE_URL/api/v1`
Auth header: `X-N8N-API-KEY: <N8N_API_KEY>`

---

## Workflows

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/workflows` | List all workflows |
| GET | `/workflows/:id` | Get a specific workflow |
| POST | `/workflows` | Create a new workflow |
| PUT | `/workflows/:id` | Update a workflow |
| DELETE | `/workflows/:id` | Delete a workflow |
| ~~POST~~ | ~~`/workflows/:id/activate`~~ | ❌ BLOCKED — do not use |
| ~~POST~~ | ~~`/workflows/:id/deactivate`~~ | ❌ BLOCKED — do not use |

### GET /workflows query params
- `active=true|false` — filter by active status
- `tags=tag1,tag2` — filter by tags
- `name=string` — filter by name (partial match)
- `limit=int` — max results (default 10, max 250)
- `cursor=string` — pagination cursor

### POST /workflows/:id/run
Execute a workflow manually. **Always ask user permission before calling this.**

```json
{
  "startNodes": ["Node Name"],   // optional — start from specific node
  "runData": {}                  // optional — inject test data
}
```

---

## Executions

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/executions` | List executions |
| GET | `/executions/:id` | Get execution detail (full node I/O data) |
| DELETE | `/executions/:id` | Delete an execution record |

### GET /executions query params
- `workflowId=string` — filter by workflow
- `status=success|error|waiting` — filter by status
- `limit=int` — max results
- `includeData=true` — include full node input/output (expensive, use for debugging)

---

## Credentials

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/credentials` | List credential names (no secrets exposed) |
| GET | `/credentials/:id` | Get credential metadata |
| GET | `/credentials/schema/:credentialType` | Get schema for a credential type |

Use credential IDs when building workflows that need auth (Slack, Google, etc.).

---

## Tags

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/tags` | List all tags |
| POST | `/tags` | Create a tag |

---

## Useful Patterns

### Find a workflow by name
```bash
curl -G "$N8N_BASE_URL/api/v1/workflows" \
  -H "Authorization: Bearer $N8N_API_KEY" \
  --data-urlencode "name=My Workflow"
```

### Get the last 5 failed executions for a workflow
```bash
curl -G "$N8N_BASE_URL/api/v1/executions" \
  -H "Authorization: Bearer $N8N_API_KEY" \
  --data-urlencode "workflowId=123" \
  --data-urlencode "status=error" \
  --data-urlencode "limit=5" \
  --data-urlencode "includeData=true"
```

### Create then inspect
Always fetch the created workflow after POST to confirm it was saved correctly:
```bash
RESPONSE=$(curl -X POST ...)
WORKFLOW_ID=$(echo $RESPONSE | jq -r '.id')
curl "$N8N_BASE_URL/api/v1/workflows/$WORKFLOW_ID" -H "Authorization: Bearer $N8N_API_KEY"
```
