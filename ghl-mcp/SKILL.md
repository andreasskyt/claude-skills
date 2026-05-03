---
name: ghl-mcp
description: >
  Read and write to a GoHighLevel sub-account via the LeadConnector REST API.
  Use this skill whenever the user mentions GoHighLevel, GHL, HighLevel, LeadConnector, their CRM,
  contacts/pipelines/opportunities/tags/custom fields/calendars in GHL, or asks to clean up, audit,
  migrate, or sync the CRM. Also triggers on "ghl-mcp" or "/ghl-mcp" and on phrases like
  "check my CRM", "what's in my pipeline", "add a custom field in HighLevel", "delete this tag in GHL".
  Despite the name, there is NO MCP server — this skill calls the REST API directly with curl.
---

# GHL Skill

You have read/write access to a GoHighLevel sub-account via the LeadConnector REST API.

> **Naming note:** the trigger is `/ghl-mcp` for muscle memory, but **no MCP server is installed**. All
> calls go through direct HTTPS to `services.leadconnectorhq.com`. Do not waste a turn searching for
> `mcp__*ghl*` tools — they don't exist.

---

## Setup (do this once before using the skill)

1. **Create a Private Integration Token (PIT)** in GHL:
   `Settings → Business Profile → Private Integrations → Create New Integration`.
   Grant the scopes you need (contacts, opportunities, calendars, custom fields, tags, etc.).
   The token format is `pit-<uuid>` and is **scoped to a single sub-account (location)**.

2. **Export the PIT into your shell** so Claude Code can read it from any project. Add to `~/.zshrc`
   (or `~/.bashrc`, or a sourced `~/.config/secrets` file with `chmod 600`):

   ```bash
   export GHL_ACCESS_TOKEN="pit-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
   ```

3. **Find your `LOCATION_ID`** — it's the sub-account ID. In the GHL UI: `Settings → Business Profile`,
   or visible in the URL when inside a sub-account (`/v2/location/<LOCATION_ID>/...`).

4. **Replace the placeholders** in the [Hardcoded constants](#hardcoded-constants) section below
   with your own values. If you manage multiple sub-accounts, see the
   [Multi-location](#multi-location-multiple-sub-accounts) section.

---

## Method summary (read this first)

- **Use `curl` for every API call.** Cloudflare's WAF in front of GHL **blocks Python's `urllib`**
  (User-Agent `Python-urllib/X.Y`) with **error 1010** ("Access denied based on browser signature").
  Same for any other non-browser-like UA. `curl/X.Y.Z` is allowed. Verified the hard way.
- **Always set `--max-time 20`** on every curl call. Past sessions have hit indefinite hangs in the
  background-task harness; the timeout makes failures loud and bounded.
- **Never silently swap clients.** If curl ever gets blocked or fails, **stop and report** — do not
  fall back to Python or another tool without asking the user first.
- **Preview before destructive ops.** List the IDs you're about to delete/update, then proceed.
- **Confirm scope on first failed write.** A 401 with `"not authorized for this scope"` is a real
  IAM rejection, not a transient. Don't retry blindly — report and ask.

---

## Credentials

The Private Integration Token (PIT) lives in the user's shell environment as `$GHL_ACCESS_TOKEN`,
exported from `~/.zshrc`. **Never read or print the token value.** Reference it as `$GHL_ACCESS_TOKEN`
in shell commands. If `$GHL_ACCESS_TOKEN` is unset (e.g. inside a non-login subshell), source it:

```bash
source ~/.zshrc 2>/dev/null
# or read just that one line:
TOKEN=$(grep -E '^export GHL_ACCESS_TOKEN=' ~/.zshrc | sed -E 's/.*="([^"]+)".*/\1/')
```

Token format is `pit-<uuid>`. PIT tokens are scoped to a **single location** (sub-account); they
**cannot** enumerate other locations or fetch user identity (`/users/me` returns 401).

---

## Hardcoded constants

Replace these with your own values before using the skill:

```bash
LOCATION_ID="YOUR_LOCATION_ID"          # GHL sub-account ID (see Setup above)
BASE="https://services.leadconnectorhq.com"
PIPELINE_MAIN="YOUR_PIPELINE_ID"        # primary pipeline; GET /opportunities/pipelines?locationId=$LOCATION_ID to find IDs
CALENDAR_PRIMARY="YOUR_CALENDAR_ID"     # primary calendar; GET /calendars/?locationId=$LOCATION_ID to find IDs
```

### Multi-location (multiple sub-accounts)

A single PIT is locked to one sub-account, so for multi-location setups create one PIT per
sub-account and store them as an array. Example:

```bash
# in ~/.zshrc or ~/.config/secrets
export GHL_TOKEN_ACME="pit-xxxxxxxx-xxxx-..."
export GHL_TOKEN_BETA="pit-yyyyyyyy-yyyy-..."

# in your skill calls, pick the one you need:
LOCATION_ID="acme_location_id"
GHL_ACCESS_TOKEN="$GHL_TOKEN_ACME"
```

Don't try to use one PIT across multiple locations — `/users/me` and
`/oauth/installedLocations` return 401 for PITs, so there's no way to enumerate or switch.

---

## Required headers

```
Authorization: Bearer $GHL_ACCESS_TOKEN
Version: 2021-07-28
Accept: application/json
Content-Type: application/json     # only on POST/PUT
```

The `Version` header is **mandatory** — omit it and you'll get 401s with cryptic messages.

---

## Canonical request pattern

```bash
curl -s --max-time 20 -o /tmp/ghl_resp -w "%{http_code}" \
  -H "Authorization: Bearer $GHL_ACCESS_TOKEN" \
  -H "Version: 2021-07-28" \
  -H "Accept: application/json" \
  "$BASE/locations/$LOCATION_ID/customFields"
```

Always capture status code via `-w "%{http_code}"` and body to a file via `-o`. Treat anything
outside `200/201/204` as a failure and surface the response body to the user.

For POST/PUT, write the JSON body to a file first (`/tmp/ghl_body.json`) and use
`--data-binary @/tmp/ghl_body.json`. Avoid inline `-d '{...}'` — quoting nightmares with `$`, `—`,
and non-ASCII characters that show up in option labels.

---

## Verified endpoints (PIT-safe)

| Method | Path | Purpose |
|---|---|---|
| GET | `/locations/{loc}` | Sub-account details |
| GET | `/locations/{loc}/customFields` | List all custom fields |
| POST | `/locations/{loc}/customFields` | Create custom field |
| PUT | `/locations/{loc}/customFields/{id}` | Update field (name/options/placeholder) |
| DELETE | `/locations/{loc}/customFields/{id}` | Delete field (irreversible) |
| GET | `/locations/{loc}/tags` | List all tags |
| POST | `/locations/{loc}/tags` | Create tag (`{"name":"foo"}`) |
| DELETE | `/locations/{loc}/tags/{id}` | Delete tag (irreversible) |
| GET | `/opportunities/pipelines?locationId={loc}` | List pipelines + stages |
| POST | `/opportunities/search?location_id={loc}&pipeline_id={pid}` | Search opportunities (note: snake_case query params) |
| POST | `/opportunities/` | Create opportunity |
| PUT | `/opportunities/{id}` | Update opportunity (move stage, change value) |
| DELETE | `/opportunities/{id}` | Delete opportunity |
| POST | `/contacts/search` | Search contacts (locationId in **body**, not query) |
| POST | `/contacts/` | Create contact |
| PUT | `/contacts/{id}` | Update contact |
| DELETE | `/contacts/{id}` | Delete contact |
| GET | `/calendars/?locationId={loc}` | List calendars |
| PUT | `/calendars/{id}` | Update calendar (name, hours, slug, etc.) |

---

## Known PIT limitations (do NOT attempt)

These returned 401 even with all GHL scopes enabled. They are gated to OAuth Marketplace apps and/or
plan tier — **not** fixable by adding scopes:

- **`PUT /opportunities/pipelines/{id}`** — pipeline structure / stage / win % editing.
  → Tell the user to do it manually in the GHL UI (and on lower plan tiers, the UI may not even
    expose win % — forecast math then has to live downstream).
- **`/forms/*`** — read or write calendar/booking forms.
  → Tell the user to edit form questions manually in the calendar's form builder.
- **`/users/me`, `/oauth/installedLocations`, `/oauth/userinfo`** — identity / location enumeration.
  → PITs are pre-scoped to one location; you already have the location ID hardcoded above.

---

## Critical body-shape gotchas

### Custom field `options` must be **plain string array**

```jsonc
// CORRECT
{ "name": "Market", "dataType": "SINGLE_OPTIONS", "model": "contact",
  "options": ["US", "Denmark", "Other"] }

// WRONG — returns 400 "v.trim is not a function"
{ "options": [{"key":"US","label":"US"}, ...] }
```

### `dataType` values (case-sensitive)

`TEXT`, `LARGE_TEXT`, `NUMERICAL`, `MONETORY` (yes, GHL misspelled "monetary" — use it as-is),
`SINGLE_OPTIONS`, `MULTIPLE_OPTIONS`, `RADIO`, `CHECKBOX`, `DATE`, `FILE_UPLOAD`, `PHONE`.

For `MONETORY`, GHL renders in the **location's currency** — there's no per-field currency override
via PIT. If you need a different currency, store as `NUMERICAL` and label it accordingly.

### Custom field create requires `model: "contact"`

Forgetting `"model": "contact"` silently creates an opportunity-scoped field (or fails depending on
account config). Always include it for contact fields.

### Custom field PUT preserves `picklistOptions` only if you send `options`

When renaming an option-based field, **always re-send the existing `options` array** in the PUT
body, otherwise the picklist may be cleared. Fetch the field first, copy `picklistOptions` →
`options` in the new body.

### Contacts search needs `locationId` in body, not query

```bash
curl ... -X POST "$BASE/contacts/search" \
  --data-binary "{\"locationId\":\"$LOCATION_ID\",\"pageLimit\":1}"
```
Total count is in `meta.total` (or top-level `total` depending on response shape).

### Opportunities search uses `location_id` (snake_case) in **query**

```bash
curl ... "$BASE/opportunities/search?location_id=$LOCATION_ID&pipeline_id=$PIPELINE_MAIN&limit=100"
```
Inconsistent naming between `/contacts/search` and `/opportunities/search` is a GHL quirk.

---

## Working patterns

### Preview → Confirm → Execute → Verify

For any destructive or bulk write:

1. **Read** current state (`GET /locations/.../customFields` etc.) into `/tmp/ghl_*.json`.
2. **Preview** to the user: list every ID + name + change in a table.
3. **Wait for "go"** before executing (unless the user pre-authorized a multi-phase plan).
4. **Loop with status capture** — print `[OK 200]` / `[ERR 4xx]` per item with the response body.
5. **Re-fetch and verify** the post-state, summarize what changed.

### Multi-phase batches

If the user asks for several phases (e.g. "delete junk fields, then rename, then create new"),
**always stop after each phase** and wait for explicit "go to Phase N+1". Long autonomous runs in GHL
are dangerous because every PUT/POST is a real CRM mutation visible to clients.

### Transient vs scope failures

| Symptom | Likely cause | Action |
|---|---|---|
| `401 "not authorized for this scope"` | PIT missing scope OR endpoint not PIT-eligible | Stop. Ask user to add scope, or flag as PIT limitation. |
| `403 Cloudflare error 1010` | Non-curl user-agent | Switch to curl (or stop if already on curl). |
| `400 "v.trim is not a function"` | Body has objects where strings expected (usually `options`) | Resend with strings. |
| `422 "X can't be undefined"` | Required field missing from body (often `locationId`) | Add it. |
| `429` | Rate limit | Sleep 1–2s, retry once. |
| Network timeout (curl `--max-time` hit) | GHL slow / regional issue | Retry once, then stop. |

---

## Reference: dataTypes seen in the wild

From a live production location:

- `TEXT` — single-line input
- `LARGE_TEXT` — textarea
- `NUMERICAL` — integer or decimal
- `MONETORY` — money (renders in location currency)
- `SINGLE_OPTIONS` — dropdown, one selection
- `MULTIPLE_OPTIONS` — dropdown, multi-select (returned as comma-joined string in some endpoints)
- `RADIO` — radio button group
- `CHECKBOX` — boolean

Custom field response shape includes `fieldKey` (lowercase snake-case auto-generated from name) —
this is what you reference in templates like `{{ contact.field_key }}`.

---

## Don't

- Don't print the PIT token value.
- Don't swap `curl` for any other HTTP client without asking.
- Don't `--data-binary @-` from a heredoc with literal `$` inside — interpolation will eat it.
  Always write the body to a file first.
- Don't assume a stage is "Lost" just because its `stageWinProbability` is 0 — there's no
  `isLost`/`isTerminal` field exposed via REST. Terminal flag is UI-only.
- Don't loop more than ~10 writes without pausing for user confirmation when the location has live
  contacts. Mistakes here are visible to real clients.
