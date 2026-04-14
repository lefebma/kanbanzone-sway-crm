# Kanban Zone Card — Field Reference

This file maps card fields to (a) their API names in `PUT /card/{id}` and `POST /card`, and (b) their UI locations. Use this when translating a user request into an API payload or a UI action.

## API-writable fields (use `scripts/sway.py update-card` or `create-card`)

| API field | UI location | Type / format | Notes |
|-----------|------------|---------------|-------|
| `title` | Title row, top of card | string | Usually `{First Last}` for leads |
| `description` | Description field below title | string (supports newlines + URLs) | See formatting convention below |
| `columnId` | — (changed via move) | string | Use column name → columnId mapping from board cache |
| `label` | Right sidebar, colored tag | string (label name) | Must match a label defined on the board |
| `owner` | Right sidebar, Owner | email | User must exist in the org |
| `watchers` | Right sidebar, Watchers | array of emails | |
| `dueAt` | Right sidebar, Dates (end of Planned Dates) | ISO-8601 timestamp | Use `2026-04-20T00:00:00Z` format |
| `priority` | Right sidebar, often near label | string | `low` / `normal` / `high` / `critical` (board-dependent) |
| `size` | Card metrics | string | Optional; board-dependent |
| `blocked` | Right sidebar, Block | boolean | Set true with `blockedBy` + `blockedReason` |
| `blockedBy` | Right sidebar, Block | string | Who/what is blocking |
| `blockedReason` | Right sidebar, Block | string | Short explanation |
| `customFields` | Right sidebar, Custom Fields | array of `{name, value}` | Only if the board has custom fields defined |
| `links` | Body, Links section | `{add: [], remove: []}` | Each item is `{number, type}` |

## Browser-only fields (use Chrome MCP per `references/browser-automation-tips.md`)

| Field | UI location | Notes |
|-------|------------|-------|
| **Comments** | Body, Comments section | Not exposed in v1.3 API |
| **Checklists / tasks** | Body, Tasks section | Includes checklist name + items + checked state |
| **Attachments** | Right sidebar, Attachments | File uploads/downloads |

## Description formatting convention (SWAY template)

Keep new lead cards parseable at a glance. Recommended pattern:

```
{Role} at {Company} | {City, Region}

{Source + date}. LinkedIn: {url}
{Optional: opening hook or relevant context}
```

Example:
```
Senior Operations Manager / Resource Manager at CSI Consulting Inc. | Toronto, Ontario, Canada

Warm lead - accepted LinkedIn connection invite on 10/04/2026. LinkedIn: https://www.linkedin.com/in/imtiaz-atcha-4000ba2/
```

## URL patterns

- Board: `https://kanbanzone.io/b/{BOARD_ID}`
- Specific card: `https://kanbanzone.io/b/{BOARD_ID}/c/{card_number}-{slug}`
  - Slug is URL-safe card title (hyphenated); optional
  - API's `GET /card?board=...&number=...` accepts the numeric card number

## Read-only / metric fields (returned by `GET /card` and `GET /cards`)

These appear in API responses but cannot be set directly:

- `id` / `number` — the visible card ID
- `plannedStart` / `plannedEnd` — date range
- `actualStart` / `actualEnd` — auto-populated when card enters/exits active columns
- `timeInColumn` — duration since entering current column
- `createdAt`, `updatedAt`
- `history` — audit trail of field changes

Use these for pipeline analysis and aging detection.
