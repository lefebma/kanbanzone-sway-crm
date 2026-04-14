# Kanban Zone SWAY Sales Pipeline CRM — Claude skill

A Claude Agent SDK / Claude Code skill that drives the **SWAY Sales Pipeline** template on [Kanban Zone](https://kanbanzone.io/) as a lightweight CRM. API-first against the Kanban Zone Public API v1.3, with a small browser fallback for the two features the API doesn't expose (checklists and attachments).

## What it does

Covers the four core CRM motions on any SWAY-template board, plus comment logging:

- **Read the pipeline** — `list-cards`, grouped by column, with "days idle" for stall detection
- **Create lead cards** — single (`create-card`) or bulk from CSV (`bulk-create`)
- **Move cards between stages** — `move-card`, with stage-name to columnId resolution
- **Update card fields** — title, description, label, owner, due date, priority, blocked state, custom fields, links
- **Log comments** — `comment`, for the standing hygiene practice of noting what triggered each move

Everything runs through a single Python CLI (`scripts/sway.py`) that wraps the API. The script has no runtime dependencies outside the Python standard library and `certifi`.

## Installation

### As a Claude skill

1. Clone this repo, or download the packaged `.skill` file from Releases.
2. Drop the folder into your skills directory (or load the `.skill` in Claude).
3. Follow `references/setup.md` to generate your Kanban Zone API key and configure.

### As a standalone CLI

```bash
git clone https://github.com/lefebma/kanbanzone-sway-crm.git
cd kanbanzone-sway-crm
pip3 install certifi
python3 scripts/sway.py verify
```

See `references/setup.md` for full first-time setup (API key, board ID, config).

## Portability

The skill is designed so any user who builds from the [SWAY Sales Pipeline template](https://kanbanzone.com/) can use it by providing their own API key and board ID. Nothing is hardcoded to a specific instance.

If your board's column names have drifted from the canonical SWAY template, update `references/pipeline-structure.md` to match.

## Quick start

```bash
# Read state
python3 scripts/sway.py list-cards

# Find a card
python3 scripts/sway.py find --title "Aryan"

# Add a lead
python3 scripts/sway.py create-card \
  --stage "Leads to Qualify" \
  --title "Jane Doe" \
  --description "Head of Ops at Acme" \
  --label "Warm Lead"

# Move and comment
python3 scripts/sway.py move-card --id 42 --stage "First Contact Made"
python3 scripts/sway.py comment --id 42 --text "Sent intro DM"
```

## Repository layout

```
SKILL.md                          Skill entry point for Claude
README.md                         This file
LICENSE                           MIT
scripts/
  sway.py                         CLI wrapper for the Kanban Zone API
references/
  setup.md                        Prerequisites + first-time setup
  api-reference.md                Kanban Zone Public API v1.3 reference
  pipeline-structure.md           SWAY template column / swimlane structure
  card-fields.md                  Card field conventions
  browser-automation-tips.md      Chrome MCP patterns for checklists + attachments
```

## Contributing

Issues and PRs welcome. Things that would be particularly useful:

- Support for more Kanban Zone templates beyond SWAY
- A web-hook listener mode (so the skill can react to moves in real time)
- Tests against a mock Kanban Zone API

## License

MIT — see `LICENSE`.

## Credits

Built as a working example of a Claude skill that wraps a real vendor API with a narrow, opinionated CLI. Kanban Zone and the SWAY template are third-party products; this project is not affiliated with either.
