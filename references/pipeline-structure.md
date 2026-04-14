# SWAY Sales Pipeline — Template Structure

This describes the canonical column and swimlane layout of the **SWAY Sales Pipeline** template in Kanban Zone. Every user who instantiates this template starts with this structure. Individual users may rename columns or add custom ones — always confirm against a live `refresh-cache` / `verify` run if unsure.

## Swimlanes

The board has two horizontal swimlanes representing two different lifecycles:

1. **Leads** — pre-sale. Prospects move left-to-right as they're qualified, engaged, and closed. Cards enter in Leads to Qualify and exit in Client Won or Client Lost.
2. **Customers** — post-sale. Once a deal is won, the card is moved down to the Customers swimlane and starts the retention/reconnect loop.

**Why two swimlanes:** Leads is a fast-moving sales funnel; Customers is a slow-moving nurture loop. They deserve different WIP limits and different cadences of attention.

## Parent groups (column families)

The template ships with six parent groups, each of which may contain one or more sub-row columns. These are the values you'll see in `parentTitle` on a `GET /board/{board}?includeColumns=true` response:

| Parent | Contains |
|--------|----------|
| **Leads** | The first two stages of the funnel (Leads to Qualify, First Contact Made) and their waiting/closed variants |
| **Call** | Interview and Strategy Call stages, plus their Waiting sub-rows |
| **Value** | Value Sent After Interview stage and its Waiting sub-row |
| **Proposal** | Proposal Sent stage and its Waiting sub-row |
| **Outcome** | Closing sub-rows: Client Won, Client Lost, Closed – Not Now |
| **Customers** | Post-sale lifecycle: Active Client, Follow-up After 3 / 6 / 12 Months |

Duplicate column titles (e.g. multiple "Waiting" sub-rows) are disambiguated by parent. The helper script's `resolve_column` accepts either `"Title"` when unique or `"Parent / Title"` when ambiguous, and the error message lists available options.

## Columns — Leads swimlane (left to right)

Each column has a description visible in Kanban Zone reminding you what the stage means. Shorthand matches what people tend to say conversationally.

| # | Parent | Canonical column name | Common shorthand | What it means | Exit action |
|---|--------|-----------------------|------------------|--------------|-------------|
| 1 | Leads | **Leads to Qualify** | "to qualify", "new lead", "inbox" | New contacts from DMs, events, webinars, Apollo pulls | Review profile; decide if ICP fit |
| 2 | Leads | **First Contact Made** | "contacted", "reached out", "sent a DM" | Initial message/reply exchanged | Invite to 45-min interview; set follow-up reminder |
| 3 | Call | **Interview Scheduled / Held** | "interview booked", "discovery call" | Discovery interview booked or completed | Take notes on pain points |
| 4 | Value | **Value Sent After Interview** | "value sent", "sent the doc", "sent the recap" | You sent personalized value (PDF, post, ideas) | Email recap + resources; optional LinkedIn post |
| 5 | Call | **Invited for Strategy Call** | "strategy call", "call booked" | You invited them to see how you can help | Share solution overview; book proposal discussion |
| 6 | Proposal | **Proposal Sent** | "proposal out", "sent the proposal" | Proposal delivered, awaiting decision | Add 3–5 day follow-up reminder; track outcome |
| 7a | Outcome | **Client Won (Contract + Invoice)** | "won", "closed", "signed" | Client confirmed; onboarding begins | Send contract + invoice; welcome email; then move card down to Customers swimlane |
| 7b | Outcome | **Client Lost** | "lost", "passed", "no" | Deal did not close | Log reason in comment; optionally move to 3-month reconnect later |
| 7c | Outcome | **Closed – Not Now** | "not now", "parked", "deferred" | Fit, but wrong timing | Schedule a future-quarter reconnect; optionally move back to Leads later |

### Waiting sub-rows

Several parents (Call, Value, Proposal) have a **Waiting** sub-row under the main stage. The main row is active work; Waiting is "I sent the thing, I'm waiting on them". Moving a card from the main row into a Waiting sub-row is a distinct action from moving it rightward. When disambiguation is needed, address these as `Call / Waiting`, `Value / Waiting`, `Proposal / Waiting`.

In the API, each sub-row is its own `columnId` and its own `title` with `parent` / `parentTitle` pointing back to the family — `GET /board/{board}?includeColumns=true` returns them all in one list.

## Columns — Customers swimlane

| # | Parent | Canonical column name | What it means |
|---|--------|-----------------------|---------------|
| 1 | Customers | **Active Client (In Service)** | Currently delivering. Weekly checkpoints; track results + testimonial |
| 2 | Customers | **Follow-up After 3 Months** | Auto-move clients here after 3 months. Ask for progress update; share next offer |
| 3 | Customers | **Follow-up After 6 Months** | Longer-term reconnect. Share updates; offer a review call |

Additional follow-up columns may exist further right (12 Months, etc.) in some instantiations. Run `scripts/sway.py refresh-cache` to see the full list for your board.

## Labels (card tags)

Common labels in the SWAY template:

- **Warm Lead** — source gave signal of interest (referral, inbound, warm LinkedIn connection)
- **Cold Lead** — outbound with no prior relationship (Apollo, LinkedIn Sales Navigator)
- **Hot Lead** — late-stage, high-intent

Additional labels may exist or be added per user. `GET /board/{board}` returns the full list of labels defined on the board.

Apply a lead-temperature label on card creation when source/temperature is known from context:

- Referral or inbound → Warm Lead
- Apollo/LinkedIn outbound, no prior relationship → Cold Lead
- Active negotiation, verbal commit → Hot Lead

## Card fields visible in the UI vs. API

When viewing the board in the browser, every card shows: colored top band, card `number`, title, assignee avatar, date range, checklist progress bar, and an on-screen "Time in Column" counter.

**The API does *not* return a `timeInColumn` field.** For stall detection, compute days-since from `lastActionAt` on the CardItem (the helper's `list-cards` does this and displays "(Nd idle)"). This is close enough to the UI counter for flagging stalled cards.

## Movement conventions

- **Always log a comment** when moving a card, briefly naming what triggered the move. This keeps the History tab useful. Comments are API-accessible — use `scripts/sway.py comment --id <N> --text "..."`.
- **Leads move forward or backward but never skip stages.** If a card jumps from Interview to Proposal without passing through Value/Call, flag it — it's probably a mis-drop.
- **Customers swimlane is a destination, not a workspace.** Don't create new cards there; only move existing Won cards down.
- **Lost is not a trash can.** A Client Lost or Closed – Not Now card may be reactivated months later via a Follow-up card or a fresh Leads to Qualify entry — don't archive aggressively.
