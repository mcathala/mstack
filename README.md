# mstack

Claude Code plugin marketplace for workflow-oriented agent skills and commands.

A Claude Code marketplace/plugin wrapper so the skills install as a single plugin.

## Available Plugins

### mstack

Workflow skills and commands for shipping with agents.

**Skills** (model-invoked — Claude uses them when the task matches):

| Skill | Description |
|-------|-------------|
| `sync` | Interview the user relentlessly about a plan until each branch of the design tree is resolved. |
| `to-action` | Turn an aligned understanding (typically from `/sync`) into a PRD as a parent GitHub issue plus vertical-slice child issues an AI can pick up cold. |
| `create-skill` | Scaffold a new agent skill with proper structure, progressive disclosure, and a strong description. |

**Workflow:**

```
/sync         → discover and align (relentless interview)
/to-action    → commit the plan (PRD issue + child issues)
hand to AI    → child issues, one at a time
```

**Commands** (user-invoked via `/` prefix):

| Command | Description |
|---------|-------------|
| `/ship` | Commit current changes as multiple logical commits, push to a feature branch, and open (or update) a PR targeting `staging`. Refuses to push to `master`/`staging`. |

## Setup (one-time)

### 1. Register the marketplace

Add mstack to your `~/.claude/plugins/known_marketplaces.json`:

```json
{
  "mstack": {
    "source": {
      "source": "github",
      "repo": "<your-github>/mstack"
    },
    "installLocation": "/Users/<your-username>/.claude/plugins/marketplaces/mstack",
    "lastUpdated": "2026-04-24T00:00:00.000Z"
  }
}
```

Clone it into the install location:

```bash
git clone https://github.com/<your-github>/mstack.git ~/.claude/plugins/marketplaces/mstack
```

### 2. Install the plugin

```bash
claude plugin install mstack@mstack
```

### 3. Enable auto-updates (optional)

Plugins are cached locally and won't update automatically. Add a `SessionStart` hook to your `~/.claude/settings.json`:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "claude plugin install mstack@mstack 2>/dev/null",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

## Alternative install paths

- **Manual copy** — drop any `skills/<name>/` directory into `.claude/skills/` (project scope) or `~/.claude/skills/` (user scope).

## Adding a New Plugin

1. Create a directory under `plugins/` (e.g. `plugins/mstack-extras/`)
2. Add `.claude-plugin/plugin.json`
3. Add `commands/`, `agents/`, and/or `skills/` directories as needed
4. Add the plugin entry to `.claude-plugin/marketplace.json`
5. Commit and push

### Plugin structure

```
plugins/my-plugin/
├── .claude-plugin/
│   └── plugin.json      # Plugin metadata (required)
├── commands/             # Slash commands (optional)
├── agents/               # Agent definitions (optional)
└── skills/               # Skill definitions (optional)
```

## License

MIT — see [LICENSE](LICENSE).
