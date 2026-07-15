# n8n Setup — Sales Invoice Normalizer

## Routine created

- **Name:** Sales Invoice Normalizer
- **Trigger ID:** `trig_01XvjqyuWwNKCK3BT5n87byd` (recreated twice already —
  see "Trigger history" below; if you ever recreate it again, update every
  `trig_...` reference in this file, re-grab a fresh token in n8n, **and
  re-add the Google Drive connector** — it does not carry over automatically)
- **Type:** poke-only (no schedule) — fires only when hit via its API
  endpoint, and spawns a brand-new Claude session on every fire (so
  concurrent reports never share state).
- **Connectors:** Google Drive (added via the Routine's Connectors tab in
  the Edit routine dialog — this is per-routine, not inherited from
  anything at the org level).
- **Prompt:** [`routine-prompt.md`](./routine-prompt.md) — edit that file and
  re-create/update the Routine if the normalization logic ever needs to
  change.

**No Google Sheets involved.** There is no Sheets connector in Anthropic's
connector registry, and the Google Drive connector's tools
(`create_file`, `download_file_content`, `get_file_metadata`,
`get_file_permissions`, `list_recent_files`, `read_file_content`,
`search_files`) don't support updating an existing file's contents either
way. So Claude never touches your Sheet. The Routine's only deliverable is
a JSON POST of the normalized rows to your n8n webhook — n8n (or whatever
consumes that webhook) does whatever it wants with them, sheets or
otherwise.

**Callback URL is fixed:** `https://blackswanmedia.app.n8n.cloud/webhook/claude-routine-callback`
— pass it as `callback_url` in every fire payload.

**Fire endpoint** (per the [Claude Platform API docs](https://platform.claude.com/docs/en/api/claude-code/routines-fire)):

```
POST https://api.anthropic.com/v1/claude_code/routines/trig_01XvjqyuWwNKCK3BT5n87byd/fire
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
payload, the routine receives it as a literal string."* The Routine's own
prompt parses that string back out as JSON.

**No idempotency key.** Each successful call spawns a brand-new session with
no dedup — if n8n's HTTP Request node retries on timeout/error, you get
multiple sessions processing the same file. Turn off automatic retries on
this node (Settings tab → "Retry On Fail") or dedupe on the n8n side.

Success response looks like:
```json
{"type": "routine_fire", "claude_code_session_id": "...", "claude_code_session_url": "..."}
```
That's just an "accepted" acknowledgement — the actual normalized result
arrives later via your Webhook node.

## n8n workflow shape

```
On form submission (file upload)
        │
        ▼
Google Drive — Upload   ← uploads the file to Drive, returns a file ID/URL
        │
        ▼
HTTP Request  →  POST .../routines/trig_.../fire   ← calls the Routine
        │
        ▼
   (workflow ends here; the Routine calls back asynchronously)

Webhook (claude-routine-callback)  ← separate trigger, always-on
        │
        ▼
   (whatever you do with normalized_rows — Sheets, DB, etc. — your call downstream)
```

The fire call and the callback are two separate n8n workflows/triggers,
because the Routine runs asynchronously — the HTTP Request node's response
is just "accepted", not the normalized result. The actual result arrives
later as a POST to the callback webhook.

### 1. Google Drive node (uploads the file)

Insert this right after the file-upload trigger, using n8n's own Google
Drive node (separate from Anthropic's connector — this is n8n uploading on
your Drive account's behalf). Operation: Upload. This replaces the earlier
base64/"Extract from File" approach entirely — the file's actual bytes
never need to touch the fire request body, which both shrinks the request
and avoids the multi-minute hang we saw when Claude had to reproduce a
~62,000-character base64 blob as a literal tool-call argument to decode it.

The upload node's output (confirmed field names from a real test): `id`
(bare Drive file ID), `name`, `mimeType`, `webContentLink` (a
`uc?id=...&export=download`-style URL), `webViewLink`. Use `id` directly for
`file_url_or_id` — simplest, no URL parsing needed; `webContentLink` also
works since the Routine extracts the file ID out of a full URL itself.

Whether to leave the uploaded file **private** (default) or share it
"Anyone with the link" is your call — the Routine's preferred path
(fetching via its own Drive connector, authenticated) works either way,
so there's no need to make it public. Sharing publicly is only relevant
as a fallback path if the connector-based fetch ever fails.

### 2. HTTP Request node (fires the Routine)

- **Method:** POST
- **URL:** `https://api.anthropic.com/v1/claude_code/routines/trig_01XvjqyuWwNKCK3BT5n87byd/fire`
- **Authentication:** Header Auth credential — `Authorization: Bearer <token
  from the Routine's page>` (recommended over a raw header field, so the
  token isn't stored in plaintext in the visible workflow)
- **Send Headers:** ON (in addition to the Authentication credential above)
  — add these two headers:
  - `anthropic-version: 2023-06-01`
  - `anthropic-beta: experimental-cc-routine-2026-04-01`
- **Send Body:** ON, Body Content Type: **JSON**
- **Body** — JSON field switched from "Fixed" to **"Expression"** mode, using
  a double `JSON.stringify` so escaping is handled automatically at every
  level:

```
{{ JSON.stringify({ text: JSON.stringify({ file_url_or_id: $json.id, file_name: $json.name, mime_type: $json.mimeType, callback_url: 'https://blackswanmedia.app.n8n.cloud/webhook/claude-routine-callback' }) }) }}
```

This assumes the HTTP Request node sits directly after the Google Drive
Upload node, so `$json` refers to its output (`id`/`name`/`mimeType`) —
no need to reference the upload node by name unless another node sits
between them. Also turn off "Retry On Fail" in this node's Settings tab
(see the idempotency note above).

### 3. Webhook node (receives the normalized result)

A **Webhook** node at path `/claude-routine-callback` (matching the fixed
`callback_url` above), method POST. The Routine POSTs this body to it:

```json
{
  "status": "ok",
  "file_name": "...",
  "normalized_rows": [
    { "full_name": "...", "phone_number": "...", "lead_source": "...", "contract_date": "...", "contract_total": "..." }
  ],
  "summary": { "rows_normalized": 0, "rows_skipped": 0, "missing_fields": [] }
}
```

(`status: "error"` with a `message` field instead, if something went wrong
parsing the input or processing the file.) Only the normalized rows are
included — no original-report data, no file attached. What happens after
this webhook (Sheets, a database, anything else) is entirely up to whatever
you build downstream of it — out of scope for the Routine itself.

## Why this shape

- The Routine never touches Google Sheets — there's no Sheets connector to
  add, and Drive's tools don't support updating an existing file anyway.
  Whatever consumes the callback webhook owns that.
- Files move through Drive rather than inline base64, since reproducing a
  large base64 blob as a literal tool-call argument was the likely cause of
  a run hanging for 20+ minutes on a 46KB file.
- A fresh Claude session per fire means concurrent files never interfere
  with each other's working directory or context.

## Trigger history

The Routine has been recreated twice — there's no in-place "edit a
routine's prompt" tool, so any prompt change means deleting and recreating
the trigger, which means a new trigger ID, a fresh token, and re-adding any
connectors:

1. **`trig_01Xq9YXAjxMfGfR1WaEXTVcJ`** (original) → required `callback_url`
   on every fire and refused to process without one.
2. **`trig_014Y6nMJe5HTsrdW1pYQjtRn`** → made `callback_url` optional. This
   run hung for 20+ minutes after only writing an intermediate base64 text
   file, never completing — root cause suspected to be the size of the
   inline base64 payload.
3. **`trig_01XvjqyuWwNKCK3BT5n87byd`** (current) → switched file transport
   to Google Drive (this document's current state) and dropped Google
   Sheets/`sheet_url` entirely in favor of a pure JSON callback.
