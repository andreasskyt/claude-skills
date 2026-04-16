---
name: n8n
description: >
  Interact with the user's self-hosted n8n automation instance to build, read, debug, and execute workflows.
  Use this skill whenever the user mentions n8n, wants to automate something, asks about their workflows,
  wants to connect two apps together, build a pipeline, trigger something automatically, debug a failing
  workflow, inspect an execution, or asks what automations they have running. Also use when the user says
  things like "set up automation", "make this run automatically", "connect X to Y", or "when X happens do Y".
---

# n8n Skill

You have full access to the user's self-hosted n8n instance. You can build workflows, read and inspect
existing ones, debug executions, and run workflows — but you must never activate or deactivate workflows,
and you must always ask for explicit permission before executing anything.

## Credentials

Read credentials from: `~/.claude/skills/n8n/.credentials`

```
N8N_BASE_URL=<your n8n instance URL, e.g. https://n8n.example.com>
N8N_API_KEY=<your n8n API key>
```

All API requests use:
```
X-N8N-API-KEY: <N8N_API_KEY>
Content-Type: application/json
```

---

## Rules — Read These First

1. **Never activate or deactivate workflows.** Do not call `/activate` or `/deactivate` endpoints. Ever.
2. **Always ask before executing.** Before calling any run/execute endpoint, stop and ask: "Should I run this now?" Wait for explicit yes.
3. **Read before you write.** When modifying an existing workflow, always fetch it first to understand its current state.
4. **Confirm destructive changes.** Deleting or overwriting a workflow requires user confirmation.
5. **Prefer dry-run explanation.** When building a workflow, explain what it will do before creating it.

---

## API Reference

See `references/api.md` for the full endpoint list.
See `references/workflow-structure.md` for workflow JSON schema and node anatomy.
See `references/nodes.md` for common node types, parameters, and Code node runtime details.
See `references/debugging.md` for execution data navigation, common errors, and step-by-step debug process.

---

## Core Workflows

### Listing & Reading

```bash
# List all workflows
curl -H "Authorization: Bearer $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/workflows"

# Get a specific workflow (replace :id)
curl -H "Authorization: Bearer $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/workflows/:id"

# List recent executions
curl -H "Authorization: Bearer $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/executions?limit=20"

# Get execution detail (includes input/output data for each node)
curl -H "Authorization: Bearer $N8N_API_KEY" \
  "$N8N_BASE_URL/api/v1/executions/:id"
```

### Building a Workflow

1. Read `references/workflow-structure.md` to understand the JSON schema
2. Consult `references/nodes.md` for the nodes you need
3. Build the workflow JSON
4. Explain to the user what the workflow does and how it's structured
5. Ask for confirmation before creating

```bash
# Create a new workflow
curl -X POST \
  -H "X-N8N-API-KEY: $N8N_API_KEY" \
  -H "Content-Type: application/json" \
  -d '<workflow_json>' \
  "$N8N_BASE_URL/api/v1/workflows"
```

### Editing an Existing Workflow — The Mandatory Pattern

**Always follow this exact pattern. Skipping any step causes failures.**

```bash
# Step 1: Always fetch fresh (never edit a stale copy)
curl -s "$N8N_BASE_URL/api/v1/workflows/{id}" \
  -H "X-N8N-API-KEY: $N8N_API_KEY" > /tmp/wf_latest.json

# Step 2: Edit what you need (e.g. update a Code node)
jq '(.nodes[] | select(.name == "My Node") | .parameters.jsCode) = "...new code..."' \
  /tmp/wf_latest.json > /tmp/wf_preflight.json

# Step 3: Strip to ONLY PUT-safe fields — this is critical
jq '{
  name: .name,
  nodes: .nodes,
  connections: .connections,
  settings: {
    executionOrder: .settings.executionOrder,
    callerPolicy: .settings.callerPolicy
  }
}' /tmp/wf_preflight.json > /tmp/wf_deploy.json

# Step 4: Deploy
curl -s -X PUT "$N8N_BASE_URL/api/v1/workflows/{id}" \
  -H "X-N8N-API-KEY: $N8N_API_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/wf_deploy.json | jq '{id, name, updatedAt}'
```

**PUT rejects with 400 if you include ANY of these fields:** `id`, `active`, `createdAt`, `updatedAt`, `versionId`, `tags`, `meta`, `pinData`, `staticData`. In `settings`, only `executionOrder` and `callerPolicy` are safe — everything else (`binaryMode`, `saveManualExecutions`, `saveExecutionProgress`, `availableInMCP`) causes rejection. Always strip before PUT.

### Debugging

When a workflow is failing:
1. Fetch the workflow to understand its structure
2. Fetch the most recent failed execution: `GET /api/v1/executions?workflowId=:id&status=error`
3. Inspect each node's input/output data in the execution response
4. Identify where the data broke down — wrong field name, unexpected format, missing credential
5. Propose a fix and explain why it failed

### Executing a Workflow

**Always ask for permission first.** Then:

```bash
# Trigger a manual execution
curl -X POST \
  -H "Authorization: Bearer $N8N_API_KEY" \
  -H "Content-Type: application/json" \
  "$N8N_BASE_URL/api/v1/workflows/:id/run"
```

After execution, fetch the execution result and summarise what happened at each node.

---

## How to Build Good Workflows

- **Start with the trigger.** Every workflow needs exactly one trigger node (Manual, Webhook, Schedule, etc.)
- **One thing per node.** Nodes should do one clear thing. Use Set nodes to reshape data, IF nodes to branch.
- **Name nodes clearly.** Use descriptive names like "Get Slack Message" not "HTTP Request 3"
- **Handle errors.** Add error branches or use the Error Trigger workflow pattern for critical automations
- **Test with real data.** Explain to the user what test data to use before executing

---

## Positioning Nodes

n8n uses a canvas with x/y coordinates. Use these rough rules for readable layouts:
- Trigger starts at `[0, 0]`
- Each subsequent node moves `+250` on x
- Branches go `+150` or `-150` on y
- Keep connected nodes close; avoid crossing lines
