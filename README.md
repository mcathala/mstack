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
| `to-action` | Turn an aligned understanding (typically from `/sync`) into a PRD as a parent GitHub issue plus task issues — each with priority (`high`/`medium`/`low`), blocked-by, sub-tasks, and relevant files. Kanban-board-shaped output. |
| `start-plan` | Implement the full PRD from `/to-action` by walking through every unblocked task issue in dependency order. Walks each task's sub-tasks one at a time, applying TDD where the work is deterministic, ticking the checkbox in the issue body as each sub-task completes. |
| `validate` | Run a local check against the iteration's declared metric and return pass/fail with a loop-back recommendation. |
| `diagnose` | Disciplined debug loop for stuck-red tests, runtime errors, and metric regressions. Build a feedback loop → reproduce → hypothesise → instrument → fix → regression-test. Invoked from `/start-plan` and `/validate` when something fails non-obviously. |
| `create-skill` | Scaffold a new agent skill with proper structure, progressive disclosure, and a strong description. |
| `dataviz` | Turn data into one honest, clear chart that makes a single point — and judge whether a chart is any good. Pick-the-chart matrix, encode/focus/land workflow, and a pass-before-ship rubric covering credibility and accessibility. |

**Workflow:**

```
/sync         → discover and align (relentless interview)
/to-action    → commit the plan (PRD issue + task issues with priority + sub-tasks)
/start-plan   → walk task issues, TDD per sub-task, tick checkboxes
/validate     → check the iteration against its metric
/ship         → commit, push, open PR

Sidecar: /diagnose — invoked from /start-plan or /validate when something
                     fails non-obviously (stuck-red test, mysterious metric
                     regression). Returns control to the caller after the fix.
```

**Commands** (user-invoked via `/` prefix):

| Command | Description |
|---------|-------------|
| `/ship` | Commit current changes as multiple logical commits, push to a feature branch, and open (or update) a PR targeting `staging`. Refuses to push to `master`/`staging`. |
| `/autoresearch` | Hypothesis-driven autonomous ML research loop. Reads `project.md`, diagnoses the current best's weakness, writes a falsifiable hypothesis, implements the smallest change, validates against anti-overfit gates, auto-commits each cycle on a session branch. Runs until you interrupt — then `/ship` packages the session into a PR. Never pushes, merges, or submits without approval. |
| `/yoloresearch` | Fully autonomous variant of `/autoresearch`. Same scientific spine, no per-cycle approval: the loop pushes after every commit, opens and refreshes a PR on every `keep`, and auto-submits to Kaggle within budget. Refuses only what would corrupt comparability (CV/seed/data drift) or overrun the submission ceiling. Use when you want to walk away. |
| `/handoff` | Compact the current session into a handoff doc at a timestamped temp path (`$TMPDIR/handoff-<YYYYMMDD-HHMMSS>.md`) — goal, status, active files, what didn't work, and a specific "Resume here" action. Reports the path so you can hand it to a fresh agent. Write-only: it does not read prior handoffs. |

## Planned work

The following pieces of mstack are **not yet shipped** — listed here so they don't get forgotten. The skills above intentionally don't reference them yet; once each piece lands, the relevant skill(s) will be updated to read/write to it.

### Planned persistent documents

mstack assumes a small set of repo-local documents that build up over time and are read/written by skills. Each one solves a different connective-tissue problem in the flow:

- **`CONTEXT.md`** — domain glossary; single source of truth for terminology. *Why:* keeps language consistent across humans and agents, reduces verbosity (one term replaces five paraphrases), and makes the codebase more navigable for the next agent picking up cold.
- **`EXPERIMENTS.md`** — running log of iterations and their metric impact (kept / killed / iterated). *Why:* research-flavored work needs cumulative memory across iterations. Without it, you re-try things you already killed and forget what the previous best variant was.
- **`design-tree.md`** — record of design decisions and the alternatives that were considered but not chosen, navigable as a tree. *Why:* ADRs capture *why* a decision was made; the design-tree captures *which alternatives were on the table*. Future reconsideration knows what was already weighed.
- **`docs/adr/NNNN-*.md`** — Architecture Decision Records, one file per hard-to-reverse, surprising-without-context decision. *Why:* prevents re-litigating settled decisions; the "why" survives the original conversation it was decided in.

### Planned skills

| Skill | Step | Notes |
|---|---|---|
| `propose` | step 1 (kick-off) | Frames the iteration: captures the idea, declares the metric (so `/validate` has a durable place to read it from), and for research iterations, captures the hypothesis + baseline + kill criteria. |
| `explore` | step 1a (optional) | EDA / analytics / codebase walk before `/sync`. Used when you can't yet articulate what to sync about — the output feeds the sync session. |

### Planned: embedded doc-update pattern

Doc-writing will be **embedded per skill**, not centralized in a separate `doc-update` skill. Trade-off: a format change touches every skill that writes the doc, but each skill stays self-contained — no inter-skill dependency to break.

Once the persistent docs above land, each skill will write to them inline as decisions / results crystallize:

| Skill | Writes / updates |
|---|---|
| `/sync` | `CONTEXT.md` (terms resolved during the interview); `design-tree.md` (when a branch resolves with a real alternative ruled out) |
| `/to-action` | (none — outputs flow into GitHub issues, not local docs) |
| `/start-plan` | `CONTEXT.md` (when implementation introduces or sharpens a domain term); ADR (when a load-bearing implementation decision crystallizes) |
| `/validate` | `EXPERIMENTS.md` (every run, pass or fail); ADR (when validation reveals a finding worth not re-litigating, e.g., "we tried gradient boosting twice — don't re-suggest") |
| `/diagnose` | `EXPERIMENTS.md` (for metric-regression diagnoses — what regressed, what caused it, the fix); `CONTEXT.md` (rarely, only if a term was sharpened during the investigation); ADR (when the root cause was a load-bearing architectural choice) |
| `/ship` | (none) |
| `/propose` (planned) | `EXPERIMENTS.md` (declares the iteration's metric, baseline, and kill criteria — `/validate` reads this back later) |

Conventions the embedded writes will follow:

- **Lazy creation** — never scaffold empty files; create the doc on first write.
- **Inline updates** — capture as decisions crystallize, don't batch at the end.
- **Short entries** — each entry should be readable in 30 seconds.

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
