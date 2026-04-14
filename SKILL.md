---
name: kanbanzone-sway-crm
description: Use this skill for any CRM work on a SWAY Sales Pipeline board in Kanban Zone. Trigger on requests like "add [name] to the pipeline", "move [name] to [stage]", "what's in my pipeline", "pipeline review", "log a note on [card]", "update the card for [name]", "who's in First Contact Made", "mark [name] as Won/Lost", or any mention of SWAY, the sales pipeline, or Kanban Zone cards. Also trigger when the user references SWAY template stage names (Leads to Qualify, First Contact Made, Interview Scheduled, Value Sent, Strategy Call, Proposal Sent, Client Won, Active Client, Follow-up After 3/6 Months). The skill works API-first against the Kanban Zone Public API v1.3 — including comments — with a browser fallback only for checklists and attachments. It's designed to be portable: any user who builds from the SWAY template and provides their own API key + board ID can use it.
---

# Kanban Zone — SWAY Sales Pipeline CRM

This skill drives the CRM workflow on any **SWAY Sales Pipeline** board in Kanban Zone. It covers the four core motions: **read the pipeline, create lead cards, move cards through stages, and update card fields**, plus **log comments** to the card activity log. It uses the Kanban Zone Public API v1.3 as the primary access path, falling back to browser automation only for checklists and attachments (the two features the API doesn't expose).

## Portability by design

The SWAY Sales Pipeline is a template — multiple users run their own copy. This skill assumes that:

- The **column structure** is the template's canonical structure (documented in `references/pipeline-structure.md`). All SWAY template users share the same columns.
- The **board ID** and **API key** are per-user. A first-time user must complete the setup flow (see `references/setup.md`) to configure their own credentials.
- No Marc-specific or org-specific data is hardcoded in the skill.

On first invocation for a given user, check whether configuration exists. If not, walk them through setup before attempting any action.

## When this skill applies

Trigger whenever the user references:

- The board itself ("SWAY", "the pipeline", "my pipeline", "Kanban Zone")
- Any stage name from the template (see `references/pipeline-structure.md`)
- A named prospect or client in a CRM context ("what's happening with Tyler?", "move Aryan forward")
- Pipeline-level questions ("who's stuck?", "where's my WIP?", "pull a pipeline review")

If a request sounds like generic kanban work with no CRM context, ask before assuming — this skill is intentionally narrow to the SWAY sales workflow.

## Configuration

The skill reads config from (in priority order):

1. Environment variables: `KZ_API_KEY`, `KZ_BOARD_ID`
2. A config file at `~/.config/sway-crm/config.json` with the same keys

If neither is set, hand off to the setup flow in `references/setup.md` — don't prompt for credentials inline in the conversation unless the user explicitly insists.

## Access strategy: API first, browser where needed

The Kanban Zone Public API (base URL `https://integrations.kanbanzone.io/v1/`) covers:

- Reading boards and cards (list and detail)
- Creating cards (single and bulk, via `POST /cards`)
- Updating card fields: title, description, label, owner, dueAt, priority, blocked state, custom fields, links
- Moving cards between columns (`POST /cards/{id}/move`)
- Adding comments to a card's activity log (`POST /cards/{id}/comments`)

Auth is base64-encoded API key sent as HTTP Basic. See `references/api-reference.md` for endpoint details, envelope patterns, and payload shapes. Use `scripts/sway.py` for the actual HTTP calls — it's a thin CLI that wraps the endpoints and handles the CardItem/BoardItem/ColumnItem envelope unwrapping. Always prefer calling this script over writing inline HTTP code.

The API does **not** cover:

- **Checklists / tasks** (Kanban Zone's to-do items inside a card)
- **Attachments** (file uploads)

For those two cases, fall back to browser automation via the Chrome MCP. See `references/browser-automation-tips.md` for patterns.

## Core workflows

### 1. Read pipeline state

When the user asks "what's in my pipeline", "pipeline review", "who's stuck", "what should I chase this week":

1. Call `scripts/sway.py list-cards` to pull all cards via the API. This returns title, column, label, owner, dueAt, and a `daysSinceActivity` computed from `lastActionAt` (the API has no `timeInColumn` field — this approximation is close enough for stall detection).
2. Summarize column-by-column using the canonical column list from `references/pipeline-structure.md`. Group output the way the board is laid out (Leads swimlane first, then Customers).
3. **Flag aging cards proactively.** Anything with >7 days idle in First Contact Made, Interview Scheduled, Value Sent, or Strategy Call is a stall signal worth naming. Anything in Proposal Sent > 5 days idle is worth naming. Keep it brief; don't lecture.
4. If the user asks for a written review, invoke the `sales:pipeline-review` skill as a companion after you've captured the state.

### 2. Create a new lead card

When the user says "add [name] to the pipeline", "new lead [name]", "pull [person] from Apollo into SWAY":

1. Gather the minimum required info: **name** (card title) and **stage** (usually Leads to Qualify). Confirm stage if ambiguous.
2. Gather expected enrichment: role, company, location, source, LinkedIn URL, opening hook. Format the description using the convention in `references/card-fields.md`.
3. Create the card by calling `scripts/sway.py create-card --stage "Leads to Qualify" --title "..." --description "..." --label "Warm Lead"`. The script resolves stage name → columnId using the cached board metadata.
4. Report back the card's ID and URL so the user can verify.

For **bulk creation** (≥5 cards from Apollo, LinkedIn Sales Navigator export, event attendee list), use `scripts/sway.py bulk-create --from leads.csv` with a CSV containing `title,description,stage,label,owner` columns.

### 3. Move a card between stages

When the user says "move [name] to [stage]", "Aryan accepted the call — update his card", "mark Perry as lost":

1. Translate the user's shorthand to the canonical column name using `references/pipeline-structure.md`. Common translations: "call booked" → "Invited for Strategy Call", "won" → "Client Won (Contract + Invoice)", "lost" → "Client Lost" sub-row.
2. Find the card ID. If unknown, call `scripts/sway.py find --title "Aryan Sharma"` to search by title (exact or substring).
3. Move it: `scripts/sway.py move-card --id {id} --stage "Invited for Strategy Call"`.
4. **Log the reason as a comment** via `scripts/sway.py comment --id {num} --text "Moved to Proposal Sent — proposal emailed 14/04/2026"`. This uses `POST /cards/{id}/comments` and keeps the card's History useful. Standing hygiene practice for the SWAY workflow — do it on every move.

### 4. Update card details

API-supported updates (use `scripts/sway.py update-card`):

- **Title / description:** `--title "..."`, `--description "..."`
- **Label:** `--label "Warm Lead"` (must match a label defined on the board)
- **Owner:** `--owner user@example.com`
- **Due date:** `--due 2026-04-20`
- **Priority:** `--priority high` (low/normal/high/critical depending on board config)
- **Block:** `--block "Waiting on CFO sign-off"` / `--unblock`
- **Custom fields:** `--custom-field "Deal Value=12000"` (repeatable)

Comments are API-supported — use `scripts/sway.py comment --id {num} --text "..."`.

Browser-only updates (use the Chrome MCP per `references/browser-automation-tips.md`):

- **Checklist item add/toggle:** always via browser
- **Attachment upload:** always via browser

## Reference files

- `references/setup.md` — First-time user setup: generating API key, finding board ID, writing config.
- `references/pipeline-structure.md` — SWAY template column list, swimlane semantics, labels, movement conventions.
- `references/card-fields.md` — Card field layout (API field names + UI equivalents), description formatting convention.
- `references/api-reference.md` — Full Kanban Zone Public API v1.3 endpoint documentation, with payload shapes.
- `references/browser-automation-tips.md` — Chrome MCP patterns for the three browser-only operations (comments, checklists, attachments).

## Things to avoid

- **Don't drag a lead card straight into the Customers swimlane.** That swimlane is for post-close clients only. A won deal moves to **Client Won (Contract + Invoice)** in the Leads row first, and is then moved to **Active Client** in the Customers row — two separate moves.
- **Don't delete cards.** If the user wants to remove one, ask whether they want it archived instead. Hard delete loses history and is prohibited per safety rules.
- **Don't bulk-move without confirming.** Even obvious-looking batches ("move everyone in Proposal Sent to Client Lost after 30 days") should confirm before running.
- **Don't store, log, or echo API keys in output.** The script reads keys from env vars or the config file and never prints them.

## Why this skill exists

SWAY is a high-touch, low-volume outbound template — a handful of warm leads at a time, each meaningfully nurtured. The board is how a practitioner keeps that discipline without a heavy CRM. This skill is the thin layer that lets Claude participate in the workflow directly — triage a week of activity, log a call, add a prospect from an Apollo list, move deals as they progress. Keep the touch light and the records clean.
