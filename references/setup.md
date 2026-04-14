# First-Time Setup

Each user of this skill runs their own copy of the SWAY Sales Pipeline template and needs three pieces of configuration: an API key, an organization access ID (in some plans), and their board ID. This guide walks through getting each one and storing them safely.

## Prerequisites

Before starting, make sure your machine has:

- **Python 3.10 or newer.** Check with `python3 --version`. Install from https://www.python.org/downloads/ if missing. macOS ships with a system Python but it's often outdated — the python.org installer is the cleanest path.
- **`certifi`** (Python package — CA root certificates). Install with `pip3 install certifi` or `pip3 install --break-system-packages certifi` on newer macOS setups. Without it, API calls fail with `SSL: CERTIFICATE_VERIFY_FAILED` on macOS.
  - If you installed Python from python.org and still hit SSL errors, run the bundled `Applications/Python 3.x/Install Certificates.command` once.
- **A Kanban Zone account** with API access (typically an Enterprise-tier plan — see step 1 below).
- **A Kanban Zone board built from the SWAY Sales Pipeline template.** If your board's columns have been heavily renamed or restructured, update `references/pipeline-structure.md` to match, or the skill's stage-name-to-columnId resolution will fail.
- **A code editor or terminal** capable of editing your shell profile (`~/.zshrc`, `~/.bashrc`) or creating files under `~/.config/`.

### Optional (for the browser-fallback workflows)

- **A Chromium-based browser** (Chrome, Comet, Edge, Brave, Arc) for checklist and attachment operations (the two features the API doesn't expose).
- **The Chrome MCP extension** installed and connected to that browser. Not required for API-only use — comments, card creation, moves, and updates all go through the API.

### No other Python dependencies

The script uses only the Python standard library plus `certifi`. No `requests`, no `httpx`, no virtualenv needed.

## 1. Generate your Kanban Zone API key

1. Log in to Kanban Zone at https://kanbanzone.io/
2. Go to **Organization Settings → Integrations → API Key**
   (Direct link: https://kanbanzone.io/settings/integrations)
3. Click **Generate API Key**
4. Copy the key immediately — some key-types are only shown once

> **Note:** The API may require an Enterprise-tier plan, depending on your subscription. If you don't see the API Key option in settings, your plan may not include it.

## 2. Find your SWAY board ID

1. Open your SWAY Sales Pipeline board in Kanban Zone
2. Look at the URL: `https://kanbanzone.io/b/{BOARD_ID}`
3. Copy the `{BOARD_ID}` portion (a short alphanumeric string, e.g. `aBcD1234`)

## 3. Store your credentials

Two options — pick one.

### Option A: Environment variables (recommended for ad-hoc use)

Add to your shell profile (`~/.zshrc`, `~/.bashrc`, or similar):

```bash
export KZ_API_KEY="your-api-key-here"
export KZ_BOARD_ID="your-board-id-here"
```

Reload the shell (`source ~/.zshrc`) and the skill's helper script will pick them up automatically.

### Option B: Config file (recommended for daily use)

Create `~/.config/sway-crm/config.json`:

```json
{
  "api_key": "your-api-key-here",
  "board_id": "your-board-id-here"
}
```

Set file permissions so only you can read it:

```bash
chmod 600 ~/.config/sway-crm/config.json
```

The helper script looks here if env vars aren't set.

## 4. Verify

From the skill's directory, run:

```bash
python scripts/sway.py verify
```

Expected output: something like

```
✅ API reachable
✅ Board "SWAY Sales Pipeline" resolved
   Columns discovered: 10
   Swimlanes: Leads, Customers
   Labels: Warm Lead, Cold Lead, ...
```

If you see errors, check that:
- The API key is correctly copied (no trailing whitespace)
- The board ID matches the URL exactly
- Your Kanban Zone plan includes API access

## 5. (Optional) Cache board metadata

Running `verify` also caches the board's column name → columnId mapping to `~/.config/sway-crm/board-cache.json`. This is what the skill uses to translate "Leads to Qualify" (a human-friendly name) into the columnId the API expects. Refresh the cache anytime with:

```bash
python scripts/sway.py refresh-cache
```

Refresh if you (or a teammate) rename a column or add a new one to the board.

## Troubleshooting

- **`SSL: CERTIFICATE_VERIFY_FAILED`** — Python can't find root CA certificates. Run `pip3 install --break-system-packages certifi`, or (on python.org installs) run `Install Certificates.command` from your Python install folder.
- **`python: command not found`** — Try `python3` instead. On macOS, `python` may be missing even when `python3` is installed.
- **HTTP 200 with body `"Bad Request"` or `"Unauthorized"`** — The API key isn't being sent as expected. The helper base64-encodes it for you; if you pre-encoded it, set `KZ_KEY_PREENCODED=1` in your environment.
- **401 Unauthorized** — API key is wrong or expired. Regenerate in Kanban Zone settings.
- **404 Board not found** — Board ID is wrong, or your API key doesn't have access to that board.
- **403 Forbidden** — Your plan may not include API access. Check with Kanban Zone support.
- **Columns don't match the template** — Your board's column names have drifted from the canonical SWAY template. Either rename them back, or update `references/pipeline-structure.md` to match your local customizations.

## Security notes

- The API key is a secret. Don't commit it to version control, don't paste it into shared docs, don't include it in screenshots.
- The skill's helper script never prints the key in output.
- If you suspect your key has leaked, revoke it immediately in Kanban Zone settings and generate a new one.
