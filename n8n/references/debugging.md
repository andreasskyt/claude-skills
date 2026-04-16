# n8n Debugging Reference

## Reading Execution Data

```bash
# Get execution summary (fast)
curl -s "$N8N_BASE_URL/api/v1/executions/{id}" \
  -H "X-N8N-API-KEY: $N8N_API_KEY" | jq '{id, status, startedAt, stoppedAt}'

# Get full execution with all node I/O data (can be very large)
curl -s "$N8N_BASE_URL/api/v1/executions/{id}?includeData=true" \
  -H "X-N8N-API-KEY: $N8N_API_KEY" > /tmp/exec_{id}.json

# Always check size before reading into context
wc -c /tmp/exec_{id}.json
```

**Size warning:** Executions with loops over many items + large payloads (e.g. transcripts) can be 50–500 MB. Always save to /tmp and use jq to extract specific nodes rather than loading the whole file.

---

## Navigating Execution Data

```bash
# List all node names that ran
cat /tmp/exec_{id}.json | jq '.data.resultData.runData | keys[]'

# Find all nodes that errored
cat /tmp/exec_{id}.json | jq '
  .data.resultData.runData | to_entries[]
  | select(.value[0].error != null)
  | { node: .key, error: .value[0].error.message }
'

# Get output data from a specific node (first iteration)
cat /tmp/exec_{id}.json | jq '.data.resultData.runData["Node Name"][0].data.main[0][0].json'

# Get output from a specific loop iteration (index N)
cat /tmp/exec_{id}.json | jq '.data.resultData.runData["Node Name"][N].data.main[0][0].json'

# Get error detail from a node
cat /tmp/exec_{id}.json | jq '.data.resultData.runData["Node Name"][0].error'
```

### Data path breakdown
```
.data.resultData.runData
  ["Node Name"]          ← array of run objects (one per loop iteration)
    [iterationIndex]
      .executionStatus   ← "success" | "error"
      .error             ← present if errored: { message, stack }
      .data.main
        [outputIndex]    ← 0 = first output, 1 = second (IF/Switch branches)
          [itemIndex]    ← 0 = first item
            .json        ← the actual data
```

**Critical:** Workflow-level `status: "success"` does NOT mean all nodes succeeded. n8n can continue past errored nodes. Always check per-node `.error` when debugging.

---

## Finding Recent Failures

```bash
# Last 5 failed executions for a workflow
curl -s -G "$N8N_BASE_URL/api/v1/executions" \
  -H "X-N8N-API-KEY: $N8N_API_KEY" \
  --data-urlencode "workflowId=WORKFLOW_ID" \
  --data-urlencode "status=error" \
  --data-urlencode "limit=5" \
  | jq '[.data[] | {id, startedAt, stoppedAt}]'
```

---

## Common Errors & Fixes

### PUT 400 "must NOT have additional properties"
You included a field that n8n rejects. Strip to only:
```json
{ "name", "nodes", "connections", "settings": { "executionOrder", "callerPolicy" } }
```

### 401 "X-N8N-API-KEY header required"
You're using `Authorization: Bearer` instead of `X-N8N-API-KEY`. Fix the header.

### 401 "unauthorized"
API key is wrong or expired. Regenerate at `{N8N_BASE_URL}/settings/api`.

### Anthropic API 400 inside a Code node
Could be any of:
1. **Out of credits** — 400 with "credit balance too low". Check console.anthropic.com → Plans & Billing
2. **Invalid model name** — model was deprecated/renamed. Verify against current Anthropic docs. Current valid: `claude-sonnet-4-20250514`, `claude-opus-4-20250514`, `claude-haiku-4-5-20251001`
3. **Malformed request body** — use `json: false` + `JSON.stringify(body)` for Anthropic API calls

### Anthropic 429 inside a Code node
Rate limited. Use exponential backoff (see nodes.md for pattern). Start at 10s delay, double each retry, max 4 retries.

### GHL (GoHighLevel) Contacts — wrong endpoint
- ❌ `/contacts/search` → 400 (treats "search" as an ID)
- ❌ `/contacts/?email=foo@bar.com` → 422
- ✅ `/contacts/?locationId={id}&query={email or name}` — works for both

### Execution data is massive
For workflows that loop over many items with large payloads, `includeData=true` returns huge responses. Strategy:
1. Save to `/tmp/exec_{id}.json`
2. Use `jq` to extract only the node you care about
3. Never try to read the whole file into context

### Loop node — wrong input access
Inside a loop body, each iteration passes one item. Use:
- ✅ `$input.first().json`
- ❌ `$items('Previous Node')` — only returns current iteration data, not all items

---

## Debugging Workflow — Step by Step

1. **Get the workflow ID** from `GET /api/v1/workflows` (filter by name)
2. **Get last failed execution**: `GET /api/v1/executions?workflowId={id}&status=error&limit=1`
3. **Fetch full execution**: `GET /api/v1/executions/{execId}?includeData=true` → save to /tmp
4. **Find errored node**: `jq '.data.resultData.runData | to_entries[] | select(.value[0].error != null)'`
5. **Inspect that node's input** (the node just before it) to see what data it received
6. **Read the Code node source** from the live workflow: `jq '.nodes[] | select(.name == "Node Name") | .parameters.jsCode'`
7. **Identify the mismatch** between expected and actual input data
8. **Fix → deploy** using the fetch → strip → PUT pattern
