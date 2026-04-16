---
name: google-sheets
description: Read, audit, and edit Google Sheets spreadsheets — including multiple sheets within document. Use this skill whenever the user wants to: open or inspect a Google Sheet, fix broken formulas or errors (#REF!, #DIV/0!, #VALUE!, #N/A, circular refs), add or rewrite formulas in specific cells, build out spreadsheet architecture from a business description, understand what a sheet is calculating, rename or restructure sheets, update ranges of data, batch-edit formulas across a workbook, or translate a real-world problem (budgets, trackers, dashboards, P&Ls, pipelines, KPI sheets) into Google Sheets structure and formulas. Trigger this skill proactively any time the user mentions a spreadsheet ID, a Sheets URL, "my sheet", "my workbook", "tab", "formula error", or anything that sounds like Google Sheets work.
---

# Google Sheets Skill

Full read/write access to Google Sheets documents via the Sheets API v4. Covers: reading structure + values + formulas, writing/rewriting formulas, fixing errors, batch updates, and translating business logic into sheet architecture.

---

## Step 1: Credentials Check

Before doing anything, verify auth:

```bash
echo $GOOGLE_SHEETS_API_KEY       # API key (read-only, public sheets only)
echo $GOOGLE_SERVICE_ACCOUNT_JSON # Path to service account JSON key file
echo $GCLOUD_ACCESS_TOKEN         # Short-lived OAuth token from gcloud
```

**Three auth paths — pick the one that applies:**

### A) Service Account (recommended for automation)
The user has a `service_account.json` from Google Cloud Console. Set the path:
```bash
export GOOGLE_SERVICE_ACCOUNT_JSON="/path/to/service_account.json"
```
Get an access token:
```bash
export ACCESS_TOKEN=$(python3 -c "
from google.oauth2 import service_account
import google.auth.transport.requests
creds = service_account.Credentials.from_service_account_file(
    '$GOOGLE_SERVICE_ACCOUNT_JSON',
    scopes=['https://www.googleapis.com/auth/spreadsheets']
)
creds.refresh(google.auth.transport.requests.Request())
print(creds.token)
")
```
The sheet must be shared with the service account email (from the JSON file, field `client_email`).

### B) gcloud CLI (easiest for personal use)
```bash
export ACCESS_TOKEN=$(gcloud auth print-access-token)
```
Requires `gcloud auth login` done previously. Token expires after ~1 hour; rerun if needed.

### C) API Key (read-only, public sheets only)
```bash
export SHEETS_API_KEY="your_api_key"
```
Only works if the sheet is publicly accessible.

**If none of these work, stop and ask the user:**
> To access your Google Sheet I need credentials. The easiest option:
> 1. Run `gcloud auth login` in your terminal, then `! gcloud auth print-access-token` here
> 2. Or share your service account JSON path
>
> Also share the Spreadsheet ID — it's the long string in your sheet URL:
> `https://docs.google.com/spreadsheets/d/**SPREADSHEET_ID**/edit`

---

## Step 2: Get the Spreadsheet ID

Extract from a URL or use directly. The ID is the string between `/d/` and `/edit`:
```
https://docs.google.com/spreadsheets/d/YOUR_SPREADSHEET_ID/edit
                                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

---

## Step 3: Discover the Document Structure

Always do this first — never assume sheet names or structure.

```bash
SPREADSHEET_ID="your_id_here"

# Get all sheet names and their IDs
curl -s "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}?fields=sheets.properties" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for s in data.get('sheets', []):
    p = s['properties']
    print(f\"Sheet: '{p['title']}' | ID: {p['sheetId']} | Rows: {p.get('gridProperties',{}).get('rowCount','?')} | Cols: {p.get('gridProperties',{}).get('columnCount','?')}\")
"
```

This gives you the tab names (e.g., "Summary", "Raw Data", "Jan 2024"). Note them — you'll use the exact title string in range references like `'Sheet Name'!A1:Z100`.

---

## Step 4: Read Values and Formulas

### Read displayed values (what you see on screen)
```bash
SHEET_NAME="Sheet1"
RANGE="${SHEET_NAME}!A1:Z200"

curl -s "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values/${RANGE}?valueRenderOption=FORMATTED_VALUE&dateTimeRenderOption=FORMATTED_STRING" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### Read underlying formulas (critical for auditing)
```bash
curl -s "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values/${RANGE}?valueRenderOption=FORMULA" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

**Always read formulas before editing** — you need to see what's there before changing it.

### Read multiple sheets at once (batch)
```bash
curl -s "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values:batchGet?\
ranges=Sheet1!A1:Z200&ranges=Sheet2!A1:Z200&ranges=Summary!A1:Z50\
&valueRenderOption=FORMULA" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### Parse and display the data
```python
import json, subprocess

result = subprocess.run([
    'curl', '-s',
    f'https://sheets.googleapis.com/v4/spreadsheets/{SPREADSHEET_ID}/values/{RANGE}?valueRenderOption=FORMULA',
    '-H', f'Authorization: Bearer {ACCESS_TOKEN}'
], capture_output=True, text=True)

data = json.loads(result.stdout)
rows = data.get('values', [])
for i, row in enumerate(rows, 1):
    for j, cell in enumerate(row):
        if cell:
            col_letter = chr(64 + j + 1) if j < 26 else f"A{chr(64 + j - 25)}"
            print(f"  {col_letter}{i}: {cell}")
```

---

## Step 5: Write Values and Formulas

Always use `valueInputOption=USER_ENTERED` so Google parses formulas, dates, and numbers correctly.

### Write a single range
```bash
RANGE_TO_WRITE="Sheet1!B2"

curl -s -X PUT \
  "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values/${RANGE_TO_WRITE}?valueInputOption=USER_ENTERED" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "range": "Sheet1!B2",
    "majorDimension": "ROWS",
    "values": [["=SUM(A2:A100)"]]
  }'
```

### Write multiple ranges in one call (prefer this for bulk edits)
```bash
curl -s -X POST \
  "https://sheets.googleapis.com/v4/spreadsheets/${SPREADSHEET_ID}/values:batchUpdate" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "valueInputOption": "USER_ENTERED",
    "data": [
      {"range": "Sheet1!B2", "values": [["=SUM(A2:A100)"]]},
      {"range": "Sheet1!C2", "values": [["=AVERAGE(A2:A100)"]]},
      {"range": "Summary!B5", "values": [["=Sheet1!B2 / Sheet1!B3"]]}
    ]
  }'
```

### Write a block of values (multiple rows/columns)
```bash
# values is an array of rows; each row is an array of cells
-d '{
  "valueInputOption": "USER_ENTERED",
  "data": [{
    "range": "Sheet1!A1:C3",
    "values": [
      ["Month", "Revenue", "Expenses"],
      ["Jan", "=B2-C2", "5000"],
      ["Feb", "=B3-C3", "6000"]
    ]
  }]
}'
```

---

## Step 6: Identify and Fix Formula Errors

### Common error types and what causes them

| Error | Meaning | Common fix |
|---|---|---|
| `#REF!` | Cell reference is broken (row/col deleted, out of range) | Fix the range reference; check if source sheet was renamed |
| `#DIV/0!` | Dividing by zero or empty cell | Wrap with `=IFERROR(A1/B1, 0)` or `=IF(B1=0, "", A1/B1)` |
| `#VALUE!` | Wrong data type (text in numeric formula) | Check source cells; use `=IFERROR(...)` or `VALUE()` to coerce |
| `#N/A` | VLOOKUP/MATCH/INDEX didn't find the value | Use `=IFERROR(VLOOKUP(...), "Not found")` |
| `#NAME?` | Typo in formula name | Check spelling; some functions differ in Sheets vs Excel |
| `#NUM!` | Invalid number (e.g., `SQRT(-1)`) | Add guard condition |
| `#NULL!` | Intersection of two non-overlapping ranges | Check range syntax |
| Circular ref | Formula refers back to itself | Restructure the formula flow; identify the loop |

### Audit pass — find all errors across a sheet

```python
import json, subprocess

def find_errors(spreadsheet_id, sheet_name, access_token):
    range_ref = f"'{sheet_name}'!A1:ZZ1000"
    r = subprocess.run([
        'curl', '-s',
        f'https://sheets.googleapis.com/v4/spreadsheets/{spreadsheet_id}/values/{range_ref}?valueRenderOption=FORMATTED_VALUE',
        '-H', f'Authorization: Bearer {access_token}'
    ], capture_output=True, text=True)
    
    data = json.loads(r.stdout)
    errors = []
    for i, row in enumerate(data.get('values', []), 1):
        for j, cell in enumerate(row):
            if str(cell).startswith('#') and cell.strip('#') in ['REF!','DIV/0!','VALUE!','N/A','NAME?','NUM!','NULL!']:
                col = col_num_to_letter(j + 1)
                errors.append(f"{col}{i}: {cell}")
    return errors

def col_num_to_letter(n):
    result = ""
    while n > 0:
        n, remainder = divmod(n - 1, 26)
        result = chr(65 + remainder) + result
    return result
```

**Always read the formula in the error cell before deciding how to fix it** — the fix depends on intent.

---

## Step 7: Translating Business Problems to Sheet Architecture

When the user describes what they want to track or calculate (not a specific formula), use this framework:

### Architecture principles

1. **Separate raw data from calculations.** Put source data on its own sheet (e.g., "Data" or "Raw"). Build summary/dashboard sheets that reference it via `=` or `QUERY()`.

2. **Name your sheets clearly.** Use: `Data`, `Summary`, `Dashboard`, `Config` (for constants like tax rates), `[Month] YYYY` for time-series sheets.

3. **Use named ranges for constants.** If a tax rate or exchange rate appears in multiple formulas, put it in a `Config` sheet and reference it as `=Config!B2` or use named ranges.

4. **Build formulas that tolerate new data.** Use full-column references (`A:A`) or structured table references rather than hardcoded row numbers when the data will grow.

5. **One formula per pattern, copied down.** Write one formula, verify it, then replicate it down the column with drag or `arrayformula`.

### Common patterns

**Running total:**
```
=SUM($B$2:B2)
```

**Dynamic lookup across sheets:**
```
=IFERROR(VLOOKUP(A2, 'Raw Data'!$A:$D, 3, FALSE), "—")
```

**Conditional aggregation:**
```
=SUMIF('Data'!C:C, "Category A", 'Data'!D:D)
=COUNTIFS('Data'!B:B, ">="&DATE(2024,1,1), 'Data'!B:B, "<="&DATE(2024,12,31))
```

**Cross-sheet reference:**
```
='Sheet Name'!B5
```

**Dynamic headers / dates:**
```
=EOMONTH(A1, 0)   # Last day of month
=TEXT(A1, "MMM YYYY")
```

**Percentage of total:**
```
=B2/SUM($B$2:$B$100)
```

**ARRAYFORMULA (fills a whole column automatically):**
```
=ARRAYFORMULA(IF(A2:A<>"", B2:B * C2:C, ""))
```

### Translation process

When given a business problem, always:
1. **Clarify the inputs** — what raw data exists or will be entered?
2. **Identify the outputs** — what does the user need to see/report?
3. **Design the sheet map** — which sheets hold what, and how they connect
4. **Write the formulas** — starting from raw data, building toward outputs
5. **Document what you built** — report cell-by-cell what was placed where and why

---

## Step 8: Reporting Format

After every operation, report clearly:

```
## Google Sheets Summary

**Spreadsheet:** [name or ID]
**Sheets inspected:** Sheet1, Sheet2, Summary

### Changes made
| Cell | Before | After | Reason |
|------|--------|-------|--------|
| Summary!B5 | #REF! | =Sheet1!B2/Sheet1!B3 | Reference was broken after Sheet1 was renamed |
| Data!C2:C50 | (empty) | =A2*B2 | Added revenue formula (price × units) |

### Errors fixed: 3
### Formulas added: 12
### Sheets modified: 2

### Notes
- Sheet "Jan 2024" had 3 #DIV/0! errors in column G — wrapped with IFERROR
- Summary tab was referencing "Sheet1" but tab was renamed to "Data" — updated all refs
```

Always report what you changed, where, and why. Never silently edit.

---

## Common Python Helper (use when curl gets verbose)

```python
#!/usr/bin/env python3
import subprocess, json, os, sys

ACCESS_TOKEN = os.environ.get('ACCESS_TOKEN') or subprocess.check_output(
    ['gcloud', 'auth', 'print-access-token'], text=True).strip()

SPREADSHEET_ID = sys.argv[1] if len(sys.argv) > 1 else input("Spreadsheet ID: ")
BASE = f"https://sheets.googleapis.com/v4/spreadsheets/{SPREADSHEET_ID}"
HEADERS = ['-H', f'Authorization: Bearer {ACCESS_TOKEN}', '-H', 'Content-Type: application/json']

def api_get(path, params=""):
    r = subprocess.run(['curl', '-s', f"{BASE}{path}{params}"] + HEADERS, capture_output=True, text=True)
    return json.loads(r.stdout)

def api_post(path, body):
    r = subprocess.run(['curl', '-s', '-X', 'POST', f"{BASE}{path}"] + HEADERS + ['-d', json.dumps(body)], capture_output=True, text=True)
    return json.loads(r.stdout)

def api_put(path, body):
    r = subprocess.run(['curl', '-s', '-X', 'PUT', f"{BASE}{path}"] + HEADERS + ['-d', json.dumps(body)], capture_output=True, text=True)
    return json.loads(r.stdout)

# Get sheet names
meta = api_get("?fields=sheets.properties")
sheets = [(s['properties']['title'], s['properties']['sheetId']) for s in meta.get('sheets', [])]
print("Sheets:", [name for name, _ in sheets])

# Read formulas from first sheet
name, _ = sheets[0]
data = api_get(f"/values/'{name}'!A1:Z500", "?valueRenderOption=FORMULA")
rows = data.get('values', [])
```

---

## Dependency Notes

- `curl` — always available
- `python3` — available on macOS/Linux
- `google-auth` Python package — needed for service account auth: `pip install google-auth`
- `gcloud` CLI — needed for path B auth: install from cloud.google.com/sdk

For service account path, also install:
```bash
pip install google-auth google-auth-httplib2 google-api-python-client
```
