# mstack

Claude Code plugin marketplace for workflow-oriented agent skills and commands.

A Claude Code marketplace/plugin wrapper so the skills install as a single plugin.

## Available Plugins

### mstack

Workflow skills and commands for shipping with agents.

**Skills** (model-invoked — Claude uses them when the task matches — unless marked user-invoked):

| Skill | Description |
|-------|-------------|
| `sync` | User-invoked: `/mstack:sync <brief>`. Interview until we share one understanding of the request. Detects the work type (project, feature, fix, refactor, explore), reads linked briefs, issues and docs, starts from your approach or proposes a plan if you gave none, builds the design tree, and each round asks every open choice that can be decided now, with 2 to 3 options and a recommendation. Writes no files. |
| `spec` | User-invoked: `/mstack:spec`. Turn the decisions from `/sync` into a one-page spec at `docs/specs/<n>-<slug>.md` — summary (problem, solution, done when), original request, decisions, eval and guardrails, tasks. A new step gets its own spec linked in "Related". No interview, no code. Ends by asking you to review and say go. |
| `validate` | Run a local check against the iteration's declared metric and return pass/fail with a loop-back recommendation. |
| `diagnose` | Disciplined debug loop for stuck-red tests, runtime errors, and metric regressions. Build a feedback loop → reproduce → hypothesise → instrument → fix → regression-test. Invoked during implementation or from `/validate` when something fails non-obviously. |
| `create-skill` | Scaffold a new agent skill with proper structure, progressive disclosure, and a strong description. |
| `dataviz` | Turn data into one honest, clear chart that makes a single point — and judge whether a chart is any good. Pick-the-chart matrix, encode/focus/land workflow, and a pass-before-ship rubric covering credibility and accessibility. |

**Workflow:**

```
/sync         → interview: work type, design tree, decisions
/spec         → write the spec and task list, then wait for go
build         → all spec tasks in order, after one go
/code-review  → (planned) review the diff against the spec
/ship         → commit, push, open PR, sync base once you've merged it

Sidecars: /diagnose — when stuck (stuck-red test, runtime error, mysterious
                      metric regression). Returns control after the fix.
          /validate — when the work has a metric: check it locally
                      before /ship.

Chores (dependencies, CI, config): no sync, no spec — just plan then go.
```

**Commands** (user-invoked via `/` prefix):

| Command | Description |
|---------|-------------|
| `/ship` | Commit current changes as multiple logical commits, push to a feature branch, and open (or update) a PR targeting `staging`. Then waits while you merge on GitHub — answer and it checks out the base branch, pulls, and offers to delete the merged branch. Also detects an already-merged branch on entry and syncs before shipping. Refuses to push to `master`/`staging`, never merges a PR itself. |
| `/autoresearch` | Hypothesis-driven autonomous ML research loop. Reads `project.md`, diagnoses the current best's weakness, writes a falsifiable hypothesis, implements the smallest change, validates against anti-overfit gates, auto-commits each cycle on a session branch. Runs until you interrupt — then `/ship` packages the session into a PR. Never pushes, merges, or submits without approval. |
| `/yoloresearch` | Fully autonomous variant of `/autoresearch`. Same scientific spine, no per-cycle approval: the loop pushes after every commit, opens and refreshes a PR on every `keep`, and auto-submits to Kaggle within budget. Refuses only what would corrupt comparability (CV/seed/data drift) or overrun the submission ceiling. Use when you want to walk away. |
| `/handoff` | Compact the current session into a handoff doc at a timestamped temp path (`$TMPDIR/handoff-<YYYYMMDD-HHMMSS>.md`) — goal, status, active files, what didn't work, and a specific "Resume here" action. Reports the path so you can hand it to a fresh agent. Write-only: it does not read prior handoffs. |

**Files the skills write:** only `/spec`, to `docs/specs/<n>-<slug>.md`. The other skills write no docs.

## Planned work

The following pieces of mstack are **not yet shipped** — listed here so they don't get forgotten.

### Planned skills

| Skill | Step | Notes |
|---|---|---|
| `code-review` | after build | Fresh-context subagent, reviews the diff against the spec. |

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
