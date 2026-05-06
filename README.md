# spryker-review

A Claude Code plugin for thoughtful, multi-pass code review on Spryker projects.

**Repo:** https://github.com/Zaftri/code-review-spryker-claude-plugin

## Quick install (teammate paste)

In any Claude Code session:

```
/plugin marketplace add https://github.com/Zaftri/code-review-spryker-claude-plugin.git
/plugin install spryker-review@code-review-spryker-claude-plugin
/plugin list
```

Then run `/spryker-review:review` in any Spryker project.

## What it gives you

- **`/spryker-review:review`** — a slash command that runs a multi-pass code review (architecture, code quality, security, performance, tests, public API, dead code, deploy safety) and saves a structured synthesis to `reviews/`.
- **`spryker-conventions` skill** — a knowledge skill that auto-loads when you're reviewing or writing Spryker code, covering layer rules, the Bridge class-vs-interface distinction at Pyz, factory conventions, module public/private API, transfers, translations, and Propel ordering.

The command is **read-only**. It produces a review document — it never edits, fixes, commits, or pushes.

## Requirements

- Claude Code CLI installed
- A Spryker project (Pyz layout assumed: `src/Pyz/{Zed,Yves,Client,Shared,Service}/...`)
- Optional but recommended in the project being reviewed:
  - `glab` CLI for GitLab MR resolution
  - Atlassian MCP configured for ticket grounding (Jira lookup)
  - `context7` MCP for live Spryker docs lookup
  - A `.claude/CLAUDE.md` with project-specific rules (deploy gotchas, test patterns, etc.) — the command picks it up automatically

If any of these are missing the command degrades gracefully (see Step 1.6 in the command file).

## Install

### Option A — Local path (recommended for first use / dev)

From inside the project where you want the command available:

```bash
claude --plugin-dir /path/to/spryker-review-plugin
```

Or, in a Claude Code session:

```
/plugin marketplace add /path/to/spryker-review-plugin
/plugin install spryker-review@spryker-review
```

### Option B — From the published GitLab repo (recommended for teammates)

```
/plugin marketplace add https://github.com/Zaftri/code-review-spryker-claude-plugin.git
/plugin install spryker-review@code-review-spryker-claude-plugin
```

If the repo is private, make sure you can `git clone` it from your terminal first — Claude Code uses the same git credentials.

### Verify

```
/plugin list
```

You should see `spryker-review` listed. The command appears in the slash menu as `/spryker-review:review`. The skill auto-activates based on context.

## Usage

### Quick start

```
/spryker-review:review
```

Reviews the **last commit (HEAD)** in **full mode**.

### Common invocations

| Scenario | Invocation | Why |
|---|---|---|
| Quick local sanity check on what you just committed | `/spryker-review:review --light` | ~4 agents, skips perf/test/contract/migration passes; fast feedback |
| Pre-merge review of a major feature branch | `/spryker-review:review` | Full mode, all applicable passes (~8 agents max) |
| Review a specific GitLab MR | `/spryker-review:review MR 1234` | Resolves via `glab` |
| Review a ticket whose branch is checked out | `/spryker-review:review ABC-123` | Adds Jira-based ticket grounding (acceptance criteria comparison) |
| Big diff (>5,000 LOC) | Same as full — the command will warn at Step 1.5 | Single review on huge diffs is unfocused; consider splitting |
| Hotspots-only check | `/spryker-review:review` then answer "hotspots-only" at Step 1 prompt | ~2-3 agents; runs only the hotspots that match |

### What happens

1. **Step 1 — Scope** — the command resolves the diff target, fetches the ticket if detectable, prints scope brief, identifies hotspots, and asks "full / light / hotspots-only / module-focused?".
2. **Step 3 — Parallel passes** — up to 8 review agents run concurrently (architectural, code quality, frontend, security, deploy/data, performance, tests, API/contract).
3. **Step 4 — Validator pass** — an independent agent re-reads cited files and decides CONFIRMED / DISPUTED / INCONCLUSIVE per finding, plus flags new issues spotted while validating.
4. **Step 5 — Synthesis** — saved to `reviews/<TICKET>-spryker-review.md` (or `reviews/<short-sha>-spryker-review.md` if no ticket). Starts with a YAML frontmatter (`spryker_review_version`, severity counts, validator status, verdict).
5. **Step 6 — Exit criteria checklist** when you ask "is the review done?".

### Modes

- **Full** (default): A + B + B' (frontend) + C1 + C2 + D + E + F + validator. Worst case ≈9 agents.
- **`--light`**: A + B + C1 (only if hotspot) + validator. ≈4 agents.
- **Hotspots-only** (chosen at Step 1 prompt): only the hotspot passes that match. 2–3 agents.

## What the command will NOT do

- Edit any source file
- Apply fixes (even if you ask — it will redirect you to `/implement` or a manual edit session)
- Run `/verify` / `/test` on your behalf
- Commit, push, or comment on PRs/MRs

## Output schema (parseable)

The synthesis file starts with versioned YAML frontmatter so CI gates / dashboards / PR-bots can parse it:

```yaml
---
spryker_review_version: 1
ticket: ABC-123
commit: ef0d08c75
mode: full
verdict: block_merge          # block_merge | proceed_with_caveats | clean
counts_by_severity:
  critical: 7
  blocker: 3
  major: 25
  minor: 22
  nit: 8
validator_status:
  confirmed: 11
  disputed: 1
  inconclusive: 5
generated_at: 2026-05-06T14:32:00Z
---
```

## Updating

```
/plugin uninstall spryker-review@code-review-spryker-claude-plugin
/plugin install spryker-review@code-review-spryker-claude-plugin
```

Or, for development with the repo cloned locally, `/reload-plugins` picks up edits without re-install.

## Disable / uninstall

```
/plugin disable spryker-review@code-review-spryker-claude-plugin     # temporary
/plugin uninstall spryker-review@code-review-spryker-claude-plugin   # permanent
```

## Project-specific tuning

The command reads `./.claude/CLAUDE.md` from the project being reviewed. Put project-specific rules there — the agents will automatically inline them as the highest-priority rule sheet. Examples that work well:
- Deploy / post-deploy gotchas
- Test patterns (static cache contamination, helper builds)
- Custom code style decisions
- Module ownership / responsibility map

If `./.claude/CLAUDE.md` is missing, the review proceeds with Spryker conventions only and notes "no project rule sheet found" in the synthesis.

## License

MIT
