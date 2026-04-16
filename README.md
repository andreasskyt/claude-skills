# Claude Code Skills

A collection of skills for [Claude Code](https://claude.ai/claude-code) that add integrations with popular tools and services.

## Skills included

| Skill | Description |
|---|---|
| **find-skills** | Discover and install new Claude Code skills |
| **frontend-design** | Create distinctive, production-grade frontend interfaces with high design quality |
| **gmail** | Read emails from Gmail (read-only) |
| **google-calendar** | Read and create Google Calendar events |
| **google-sheets** | Read, audit, and edit Google Sheets spreadsheets |
| **n8n** | Build, read, debug, and execute n8n automation workflows |
| **notion-api** | Full Notion API interaction — pages, databases, blocks, comments |
| **obsidian-bases** | Create and edit Obsidian Bases (.base files) with views, filters, and formulas |
| **obsidian-cli** | Interact with Obsidian vaults via the CLI — notes, tasks, properties, plugin dev |
| **obsidian-markdown** | Obsidian Flavored Markdown — wikilinks, embeds, callouts, properties |
| **skill-creator** | Create, modify, and benchmark Claude Code skills |
| **web-design-guidelines** | Review UI code for Web Interface Guidelines compliance |

## Installation

Copy any skill folder into your Claude Code skills directory:

```bash
# Copy a single skill
cp -R gmail ~/.claude/skills/

# Copy all skills
cp -R */ ~/.claude/skills/
```

Restart Claude Code to pick up the new skills.

## Configuration

Some skills require credentials via environment variables. Set these in your shell profile or `.env`:

| Skill | Required env vars |
|---|---|
| **gmail** | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REFRESH_TOKEN` |
| **google-calendar** | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REFRESH_TOKEN` |
| **google-sheets** | Google auth (service account, gcloud CLI, or API key — see skill docs) |
| **n8n** | Create `~/.claude/skills/n8n/.credentials` with your `N8N_BASE_URL` and `N8N_API_KEY` (see `.credentials.example`) |
| **notion-api** | `NOTION_API_TOKEN` |
| **obsidian-cli / obsidian-bases / obsidian-markdown** | Obsidian desktop app running with CLI enabled |
| **frontend-design / find-skills / skill-creator / web-design-guidelines** | None |

## License

Each skill may have its own license terms. Check individual skill folders for details.
