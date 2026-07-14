# n8n Setup — Sales Invoice Normalizer

## Routine created

- **Name:** Sales Invoice Normalizer
- **Trigger ID:** `trig_01Xq9YXAjxMfGfR1WaEXTVcJ`
- **Type:** poke-only (no schedule) — fires only when hit via its API
  endpoint, and spawns a brand-new Claude session on every fire (so
  concurrent reports never share state).
- **Prompt:** [`routine-prompt.md`](./routine-prompt.md) — edit that file and
  re-create/update the Routine if the normalization logic ever needs to
  change.

**To get the fire endpoint + bearer token**: open the Routines list in the
Claude Code web app (claude.ai/code) and find "Sales Invoice Normalizer". Its
detail page shows the HTTPS `.../v1/claude_code/routines/{routine_id}/fire`
URL and bearer token n8n needs — those aren't something an MCP tool call can
hand back, so grab them from the UI.

## n8n workflow shape

```
On form submission (file upload)
        │
        ▼
Move Binary Data  (Mode: "Binary to JSON")   ← base64-encodes the file
        │
        ▼
HTTP Request  →  POST .../routines/trig_.../fire   ← calls the Routine
        │
        ▼
   (workflow ends here; the Routine calls back asynchronously)

Webhook  (separate trigger, always-on)
        │
        ▼
Google Sheets  (append/update rows from the callback payload)
```

The fire call and the callback are two separate n8n workflows/triggers,
because the Routine runs asynchronously — the HTTP Request node's response is
just "accepted", not the normalized result. The actual result arrives later
as a POST to your Webhook node.

### 1. "Move Binary Data" node

Insert this right after the file-upload trigger. Mode: **Binary to JSON**.
This turns the binary `file` property into JSON fields you can reference in
expressions: `{{$json.data}}` (base64 string), `{{$json.fileName}}`,
`{{$json.mimeType}}`.

### 2. HTTP Request node (fires the Routine)

- **Method:** POST
- **URL:** the fire URL from the Routine's page (ends in `/fire`)
- **Authentication:** Header Auth — `Authorization: Bearer <token from the
  Routine's page>`
- **Send Body:** ON, Body Content Type: **JSON**
- **Body:**

```json
{
  "text": "{{ JSON.stringify({ file_base64: $json.data, file_name: $json.fileName, mime_type: $json.mimeType, sheet_url: $('On form submission').item.json.sheet_url, callback_url: 'https://<your-n8n-host>/webhook/<callback-id>' }) }}"
}
```

Adjust `$('On form submission').item.json.sheet_url` to wherever the Google
Sheet URL actually lives in your form/trigger data. Replace the
`callback_url` value with your real Webhook node's production URL (see
below) — it can be hardcoded here since it doesn't change between runs.

### 3. Webhook node (receives the normalized result)

Create a separate workflow (or a separate trigger in this one) with a
**Webhook** node, method POST. Its production URL is what you put in
`callback_url` above. The Routine POSTs this body to it:

```json
{
  "status": "ok",
  "sheet_url": "...",
  "file_name": "...",
  "normalized_rows": [
    { "full_name": "...", "phone_number": "...", "lead_source": "...", "contract_date": "...", "contract_total": "..." }
  ],
  "summary": { "rows_normalized": 0, "rows_skipped": 0, "missing_fields": [] },
  "output_xlsx_base64": "..."
}
```

(`status: "error"` with a `message` field instead, if something went wrong
parsing the input or processing the file.)

### 4. Google Sheets node

Once the Google Sheet's tab/column layout is decided, add a **Google
Sheets** node after the Webhook node:
- **Operation:** Append (or Append or Update)
- **Input:** `{{$json.normalized_rows}}` (split into items with a preceding
  "Split Out" node, since it's an array)
- Map each of the 5 fields (`full_name`, `phone_number`, `lead_source`,
  `contract_date`, `contract_total`) to the matching sheet column.

This last step is intentionally left unwired until the actual Google Sheet
(and its column layout) is shared.

## Why this shape

- The Routine never touches Google Sheets credentials — n8n owns that,
  since it already has a mature Sheets node.
- Files stay inline (base64) rather than round-tripping through Drive,
  since real sales reports here are tens of KB, well within a JSON body.
- A fresh Claude session per fire means concurrent files never interfere
  with each other's working directory or context.
