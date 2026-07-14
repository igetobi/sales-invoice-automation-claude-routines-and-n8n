# n8n Setup — Sales Invoice Normalizer

## Routine created

- **Name:** Sales Invoice Normalizer
- **Trigger ID:** `trig_014Y6nMJe5HTsrdW1pYQjtRn` (recreated once already —
  see "Trigger history" below; if you ever recreate it again, update every
  `trig_...` reference in this file and re-grab a fresh token in n8n)
- **Type:** poke-only (no schedule) — fires only when hit via its API
  endpoint, and spawns a brand-new Claude session on every fire (so
  concurrent reports never share state).
- **Prompt:** [`routine-prompt.md`](./routine-prompt.md) — edit that file and
  re-create/update the Routine if the normalization logic ever needs to
  change.

**`callback_url` is optional.** If you include it in the fire payload, the
Routine POSTs its result there. If you omit it (e.g. while testing without a
Webhook node set up yet), the Routine still fully processes the file — it
just prints the summary, the complete normalized rows, and the output
XLSX's path directly as its final chat message instead of POSTing anywhere.
You can read that off the session's transcript (the `claude_code_session_url`
from the fire response).

**Fire endpoint** (per the [Claude Platform API docs](https://platform.claude.com/docs/en/api/claude-code/routines-fire)):

```
POST https://api.anthropic.com/v1/claude_code/routines/trig_014Y6nMJe5HTsrdW1pYQjtRn/fire
```

Required headers (confirmed from this routine's own "Example request" panel
on its Call via API trigger):
- `Authorization: Bearer <token>` — get this token from the Routine's page in
  the Claude Code web app (claude.ai/code/routines) — this is per-routine and
  not something an MCP tool call can hand back.
- `anthropic-version: 2023-06-01`
- `anthropic-beta: experimental-cc-routine-2026-04-01` — required beta header;
  the endpoint is still in research preview so this may change.
- `Content-Type: application/json`

**Important — the body has exactly one field, `text`, and it is freeform,
unparsed text.** Per the docs: *"if you send JSON or another structured
payload, the routine receives it as a literal string."* There is no
multipart/file-upload support on this endpoint — a file can only get in by
being embedded as base64 inside that string, which the Routine's own prompt
then parses back out as JSON. (We tried skipping this with n8n's Form-Data
body type; it doesn't apply here since the endpoint has no binary/multipart
handling at all.)

**No idempotency key.** Each successful call spawns a brand-new session with
no dedup — if n8n's HTTP Request node retries on timeout/error, you get
multiple sessions processing the same file. Turn off automatic retries on
this node (Settings tab → "Retry On Fail") or dedupe on the n8n side.

Success response looks like:
```json
{"type": "routine_fire", "claude_code_session_id": "...", "claude_code_session_url": "..."}
```
That's just an "accepted" acknowledgement — the actual normalized result
arrives later via your Webhook node, not in this response.

## n8n workflow shape

```
On form submission (file upload)
        │
        ▼
Extract from File  (Operation: "Move File to Base64 String")   ← base64-encodes the file
        │
        ▼
HTTP Request  →  POST .../routines/trig_.../fire   ← calls the Routine
        │
        ▼
   (workflow ends here; the Routine calls back asynchronously, if callback_url was sent)

Webhook  (separate trigger, always-on — only needed once you add callback_url back in)
        │
        ▼
Google Sheets  (append/update rows from the callback payload)
```

The fire call and the callback are two separate n8n workflows/triggers,
because the Routine runs asynchronously — the HTTP Request node's response is
just "accepted", not the normalized result. The actual result arrives later
as a POST to your Webhook node (or, with no `callback_url`, as the Routine's
final chat message in its own session — see the "Trigger history" note
above).

### 1. "Extract from File" node

Insert this right after the file-upload trigger. Operation: **Move File to
Base64 String**, Input Binary Field: `file`, Destination Output Field:
`data` (confirmed working — its output is a `data` field containing the
base64 string).

### 2. HTTP Request node (fires the Routine)

- **Method:** POST
- **URL:** `https://api.anthropic.com/v1/claude_code/routines/trig_014Y6nMJe5HTsrdW1pYQjtRn/fire`
- **Authentication:** Header Auth credential — `Authorization: Bearer <token
  from the Routine's page>` (recommended over a raw header field, so the
  token isn't stored in plaintext in the visible workflow)
- **Send Headers:** ON (in addition to the Authentication credential above)
  — add these two headers:
  - `anthropic-version: 2023-06-01`
  - `anthropic-beta: experimental-cc-routine-2026-04-01`
- **Send Body:** ON, Body Content Type: **JSON** (this also sets
  `Content-Type: application/json` for you — no need to add it under Send
  Headers too)
- **Body** — switch the JSON field itself from "Fixed" to **"Expression"**
  mode (not just an inline `{{ }}` inside a Fixed JSON blob — that breaks on
  the unescaped quotes `JSON.stringify` produces) and use a double
  `JSON.stringify` so escaping is handled automatically at every level:

```
{{ JSON.stringify({ text: JSON.stringify({ file_base64: $json.data, file_name: $('On form submission').item.json.file.filename, mime_type: $('On form submission').item.json.file.mimetype, sheet_url: $('On form submission').item.json['google sheet url'] }) }) }}
```

`callback_url` is intentionally omitted above for testing — the Routine
will process the file and print its result inline in the session
transcript instead of POSTing anywhere (see the note at the top of this
file). Add it back into the inner object, pointed at your Webhook node's
production URL, once that's built:

```
{{ JSON.stringify({ text: JSON.stringify({ file_base64: $json.data, file_name: $('On form submission').item.json.file.filename, mime_type: $('On form submission').item.json.file.mimetype, sheet_url: $('On form submission').item.json['google sheet url'], callback_url: 'https://<your-n8n-host>/webhook/<callback-id>' }) }) }}
```

Reference the trigger node by name (`$('On form submission')...`) rather
than plain `$json` for `file_name`/`mime_type`/`sheet_url`, since the binary
conversion node's output doesn't carry those fields through. Also turn off
"Retry On Fail" in this node's Settings tab (see the idempotency note
above).

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

## Trigger history

The Routine was recreated once: the original (`trig_01Xq9YXAjxMfGfR1WaEXTVcJ`)
required `callback_url` on every fire and would refuse to process a file
without one. Since testing without a Webhook node set up yet is a normal
thing to want, the prompt was changed to make `callback_url` optional (see
`routine-prompt.md`'s Step 5, "Delivery A" vs "Delivery B") and the trigger
was deleted and recreated as `trig_014Y6nMJe5HTsrdW1pYQjtRn` — there's no
in-place "edit a routine's prompt" tool, so a prompt change means a new
trigger ID and a fresh token. All references in this file already point at
the current ID; if it's ever recreated again, update them here too.
