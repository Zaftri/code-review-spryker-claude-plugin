---
description: Multi-pass thoughtful code review against Spryker, SOLID, DRY, YAGNI, KISS, TDA, SoC, performance, tests, contracts, dead-code and project rules
---

# Spryker Thoughtful Review

## Input
$ARGUMENTS

If `$ARGUMENTS` is empty, default to reviewing the **last commit** (`HEAD`) in **full** mode.
Otherwise parse `$ARGUMENTS` as `[--light] [<target>]` where:
- `<target>` is a git ref / range / commit SHA / MR number / ticket key (e.g. `ABC-123`)
- `--light` switches to a reduced agent set (see Step 3)

This command is **read-only**. It NEVER edits files or applies fixes — only produces a review document.

## Step 1: Scope summary + ticket grounding (do this first, then stop and confirm)

1. **Resolve the target:**
   - Bare commit/ref → `git show --stat <ref>`
   - Range `A..B` → `git diff --stat A..B`
   - GitLab MR `!1234` or `MR 1234` → `glab mr diff 1234` (use `glab`, not the MCP — see memory)
   - Ticket key (e.g. `ABC-123`) → infer commit by branch name; also fetch ticket via Atlassian MCP (`mcp__atlassian__getJiraIssue`) for acceptance criteria
2. **Ticket grounding (when a ticket key is detectable from `$ARGUMENTS` or the branch name):**
   - Fetch the ticket summary, description, and acceptance criteria.
   - Surface the **stated intent** alongside the diff stats — the review will compare implementation to intent, not just review code in a vacuum.
   - If no ticket is detectable, skip silently.
3. **Print a short scope brief to the user:**
   - Files changed, lines +/-
   - Module list, derived from changed paths:
     `git diff --name-only <range> -- '*.php' | grep -E '^src/(Pyz|Spryker)/' | sed -E 's|src/(Pyz\|Spryker)/(Zed\|Yves\|Client\|Shared\|Service)/([^/]+)/.*|\2/\3|' | sort -u`
   - Detected hotspots (using the canonical patterns in **Step 1.4** below)
4. **Ticket-vs-diff sanity check:** call out anything in the diff that doesn't seem to map to the ticket, AND any acceptance criteria that look unaddressed.
5. **Diff-size sanity guard:** if total LOC changed (`git diff --shortstat <range>`) exceeds **5,000**, warn the user that this is too large for a single thoughtful review and suggest reviewing per-module or per-commit. Do NOT block — just warn.
6. **Ask the user:** *full review (all applicable passes), hotspots-only, or a specific module / concern?* Wait for confirmation before launching agents.

## Step 1.4: Hotspot patterns (canonical, referenced by Pass C/D/E/F triggers)

A change is in a "hotspot" if any modified path matches one or more of these globs:

| Hotspot category | Glob / pattern | Triggers pass |
|---|---|---|
| **Migrations** | `src/**/Persistence/Propel/Schema/*.schema.xml`, `src/Orm/Propel/*/Migration_*/*.php` | C2 |
| **Post-deploy** | `config/post-deploy/*.yml` | C2 |
| **ACL / authz / multi-tenant** | files containing `Acl`, `Permission`, `Authorization`, `MerchantUser`, `Tenant` in the path | C1 |
| **Transfer XML** | `src/**/Transfer/*.transfer.xml` | F |
| **DependencyProvider / plugin registration** | `src/Pyz/{Zed,Yves,Client}/**/*DependencyProvider.php` | A (always); F if a plugin interface is added/removed |
| **Public API surface** | `*FacadeInterface.php`, `*PluginInterface.php`, `*Service*Interface.php`, `*ClientInterface.php` | F |
| **Performance / N+1 candidates** | `*Repository.php`, `*EntityManager.php`, `Communication/Console/*.php`, files with new `foreach` over query results | D |
| **Tests** | `tests/PyzTest/**/*.php`, `tests/PyzTest/**/*.feature` | E |
| **Frontend (heavy)** | `src/Pyz/**/Presentation/assets/{js,ts,scss}/**` over ~200 LOC changed | B' |

## Step 1.6: Degradation / failure modes

Tools this command relies on may be unavailable. Document the degradation per tool; do NOT abort the review:

| Tool | Used for | If unavailable |
|---|---|---|
| `mcp__context7__query-docs` | Spryker convention lookup | Fall back to `WebFetch` against `docs.spryker.com`; flag in synthesis: "context7 unavailable — Spryker rule citations not verified" |
| `glab` CLI | Resolving GitLab MR scope | Fall back to git only (`git diff` for the local branch); skip MR-specific scope |
| `mcp__atlassian__getJiraIssue` | Ticket grounding (Step 1.2) | Skip ticket fetch silently; reviews still proceed without intent comparison |
| `/verify` skill (PHPStan/Psalm) | Cross-checking dead-code findings | Agents flag dead code from grep alone; mark such findings "(unverified by static analysis)" |
| `reviews/` directory writable | Persisting synthesis (Step 5) | If write fails (e.g. `.gitignore` plus permission denied), output the synthesis inline and warn the user |

## Step 1.5: Out-of-scope (do not waste tokens reviewing)

These are explicitly **not reviewed** by any pass:
- Vendor code under `vendor/`
- Generated transfer objects (the `.transfer.xml` IS reviewed; the generated PHP is not)
- Generated Propel entities (`src/Orm/Propel/*/Base/`, `*EntityTransfer.php`)
- Generated migration SQL bodies (the schema XML and migration `up`/`down` ordering ARE reviewed; SQL syntax inside is not)
- Translation CSV *content* spelling (existence, completeness across locales, and code references ARE reviewed — see rule 7)
- Fixture data shape (`data/import/fixtures/`)

## Step 2: Rule sheet (always loaded into every review agent)

Every dispatched agent MUST receive these as ground rules, in priority order:

1. **Project rules** — the relevant sections of `./.claude/CLAUDE.md` (path relative to the project root in which this command is invoked):
   - Code Style (visibility, line length, native types on constants, magic numbers, interface scope)
   - Static Analysis & Testing requirements
   - Propel command order
   - Deployment / post-deploy gotchas
   - If the file is missing, agents should note "no project rule sheet found" in their output and continue with Spryker conventions only.
2. **Spryker canonical conventions** — agent must call `mcp__context7__query-docs`
   with `libraryId: "/spryker/spryker-docs"` whenever it hits an unfamiliar pattern
   (facade/factory/plugin/dependency-provider/module API/DI). Do NOT WebFetch unless context7 fails.
   Suggested queries:
   - "architectural conventions layers Zed Yves Client Shared Service module API"
   - "dependency provider facade plugin stack conventions"
   - "module public vs private API rules"
   - "factory createX vs getX conventions visibility"
   - **Bridges at Pyz**: bridge **classes** are Core-only; at Pyz use direct `$container->getLocator()->y()->facade()` injection. Bridge **interfaces** in `Pyz/Zed/X/Dependency/Facade/` ARE recommended as factory return-type hints (see saved memory `spryker_bridges_pyz.md`). Do NOT flag direct foreign-facade imports as Blockers.
3. **SOLID / DRY / YAGNI** — flag specifically:
   - SRP violations (classes doing persistence + business + presentation)
   - OCP via plugin extension points where Spryker expects them
   - LSP / interface conformance for Pyz overrides of Spryker classes
   - ISP — interfaces only for Facade / Plugin / Service / Client (per project CLAUDE.md)
   - DIP — depend on facade interfaces, not concrete classes; cross-module via facade only
   - DRY — duplicated logic, especially copy-paste between Zed and Yves (≥3 lines of meaningful duplication)
   - YAGNI — speculative abstractions, unused config, dead branches, premature plugin stacks (see also rule 7 for concrete dead-code detection)

4. **KISS — Keep It Simple, Stupid** — flag complexity that doesn't pay rent:
   - Cyclomatic complexity: nested ifs / switches deeper than 3 levels, methods over ~30 lines
   - Clever one-liners (chained ternaries, regex bombs, `array_*` chains) where a loop reads better
   - Premature performance tricks (micro-optimizations, manual caching) without measurement
   - Over-engineered class hierarchies — abstract class + interface + 1 implementation
   - Configuration knobs / feature flags for things that have one real value
   - Reflection, dynamic dispatch, or magic methods used where a direct call exists
   - Sign of pain: a method needs a comment to be understandable → simplify the code, don't add the comment

5. **TDA — Tell, Don't Ask** — flag feature-envy and exposed internals:
   - Long getter chains (`$obj->getA()->getB()->getC()->doX()`) — Law of Demeter violation
   - Callers inspecting an object's state via getters and then deciding what method to call on it — move the decision *into* the object
   - Public getters/setters added "just in case"
   - Procedural code on rich domain objects: anaemic models with external "managers"
   - In Spryker: outside callers should never `getEntity()` then mutate; call a Facade method
   - Caveat: pure data **Transfers** in Spryker are deliberately anaemic — TDA does NOT apply to DTOs

6. **SoC — Separation of Concerns** — flag mixed responsibilities at layer/file/method level:
   - Layer-level (Spryker): business logic in Communication; Persistence calls outside `Persistence/`; presentation inlined in PHP business code; validation split half-form half-facade
   - Method-level: methods doing N things ("does X *and* logs *and* sends an event")
   - File-level: helpers, constants, and DTOs in one giant class
   - I/O mixed with pure logic: extract pure functions to make logic testable
   - Cross-cutting concerns (logging/auth/transactions) leaking into business code

7. **Dead code & unused artifacts** — concrete sniff-tests; flag with file:line and severity:
   - **Major**: unused public Facade method (added to `*FacadeInterface.php`, no caller outside tests — premature contract surface)
   - **Major**: unused private/protected method (no caller in repo)
   - **Major**: unused method parameter (declared, never referenced in body) — often signals interface drift
   - **Major**: unused class property (declared but never read OR never written)
   - **Major**: unused transfer field (`*.transfer.xml` declares it; no `getX/setX` usage — silent drop in `fromArray`)
   - **Major**: unused schema column (`*.schema.xml` declares it; no repository/entity-manager reads/writes it — migration without consumer)
   - **Major**: translation key referenced in code/twig but missing from CSV (causes runtime fallback to raw key — visible UX bug)
   - **Minor**: unused constant
   - **Minor**: unused `use` import
   - **Minor**: translation key added in one locale CSV but missing from sibling locales (`de_CH`/`en_GB`/`fr_CH`/`it_CH` must stay in lockstep)
   - **Minor**: translation key added but not referenced in any twig/PHP (orphaned)
   - **Minor**: unused config key (`*Config.php::getX()` with no caller)
   - **Minor**: unused plugin in a stack (registered in DependencyProvider but the extension point is never invoked, OR the stack is built but never consumed)
   - **Minor**: dead control-flow branches (catch for never-thrown exception, `if ($flag === true)` where `$flag` is hardcoded)
   - **Nit**: bare `// TODO` / `// FIXME` without ticket reference (acceptable: `// TODO: ABC-123 remove after deployed`)

   **Detection tactics for the agent:**
   - Cross-check against PHPStan/Psalm output via the `/verify` skill — tooling is authoritative; don't double-flag
   - For Facade public surface: `grep -rn '::methodName\|->methodName' src/ tests/` — flag if only callers are within the module's own Business/ or only tests
   - For translations:
     ```bash
     # Keys added in this commit
     git diff <range> -- 'data/translation/Zed/*.csv' | grep '^+[^+]' | cut -d, -f1 | sort -u
     # Cross-reference with twig + PHP usage
     grep -rE "trans\(|\.trans|\| trans" src/Pyz/Zed/<Module>/ templates/
     ```
   - For transfer fields: match each new XML `<property name="X">` against `getX`/`setX` in PHP
   - For schema columns: match each new `<column name="X">` against `filterByX`, `getX`, `setX` in repositories/entity managers

## Step 3: Dispatch review passes (parallel Agent calls)

Use the **Agent** tool. Send all independent passes in a SINGLE message with multiple tool calls.

### Mode: full (default) vs light

| Pass | Full mode | Light mode (`--light`) | Trigger (per Step 1.4) |
|---|---|---|---|
| A — Architectural | ✅ always | ✅ always | always |
| B — Code quality + dead code | ✅ always | ✅ always | always |
| B' — Frontend (heavy) | ✅ if frontend hotspot | ❌ skipped | `Presentation/assets/**` over ~200 LOC |
| C1 — Security | ✅ if ACL/authz hotspot | ✅ if ACL/authz hotspot | ACL / authz / multi-tenant pattern |
| C2 — Backend-architect | ✅ if migration/post-deploy hotspot | ❌ skipped | migrations or post-deploy |
| D — Performance / N+1 | ✅ if perf hotspot | ❌ skipped | Repository/EntityManager/console loops |
| E — Test quality | ✅ if test hotspot or hotspots flagged | ❌ skipped | tests changed OR Pass A/C flagged untested code |
| F — API/contract | ✅ if API surface hotspot | ❌ skipped | Facade/Plugin/Transfer XML |
| Validator | ✅ always | ✅ always | always |

**Worst-case agent count:**
- Full mode: 8 parallel + 1 validator = **9 agents**
- Light mode: 3 parallel (A + B + C1) + 1 validator = **4 agents**
- Hotspots-only (user picks at Step 1): 1–2 parallel + validator = **2–3 agents**

### Agent prompt requirements

Each agent prompt MUST:
- State the goal and the specific files in scope (don't make them re-discover).
- Include the Step 2 rule sheet inline.
- Demand findings as `severity | file:line | primary rule | what's wrong | suggested fix`. List secondary rules in parentheses if relevant (e.g. `SoC (also: KISS, SRP)`); pick **one primary rule** per finding to avoid triple-counting.
- For security findings: include a Given/When/Then attack scenario.
- **Signal budget — strict:**
  - Only include findings that would change code or shipping decisions. Drop pure preference, drop "smell-only" with no concrete fix.
  - Cap each pass at **30 substantive findings**. If you would exceed 30, drop the lowest-severity items and note "N additional Nit-level items omitted for signal" at the end.
  - Severity discipline: Critical = blocks merge; Major = fix before merge or open follow-up; Minor = fix opportunistically; Nit = preference. If unsure, drop a tier.
- Cap response length (~400-600 words per agent).
- **Forbid edits** — review only. Agents must NOT use Write/Edit tools.

### Pass A — Architectural / Spryker conformance
- `subagent_type: general-purpose`
- Scope: layer boundaries, facade-only cross-module access (per Bridge nuance in rule 2), factory conventions (`createX` protected/private, `getX` for cached deps), dependency provider correctness (constants, late-bound closures, plugin stacks), transfer object usage vs raw arrays, persistence isolated to `Persistence/`, no business logic in plugins/controllers, module public vs private API, Propel schema changes paired with migrations.
- Must use context7 `/spryker/spryker-docs` to verify any rule before citing it.

### Pass B — Code quality (SOLID / DRY / YAGNI / KISS / TDA / SoC / dead code / project style)
- `subagent_type: refactoring-expert` (or general-purpose)
- Scope:
  - **SOLID / DRY / YAGNI / KISS / TDA / SoC** per Step 2 rules 3–6
  - **Dead code & unused artifacts** per Step 2 rule 7 — explicitly run translation cross-reference and Facade-caller grep
  - **Project style** — visibility, native types on constants, magic numbers, comments restating code, line length > 120, interface scope
- For frontend changes < ~200 LOC, also includes JS/TS/SCSS smells (magic numbers, missing `*.constant.ts`, inline HTML in PHP). Heavier frontend goes to Pass B'.

### Pass B' — Frontend (only when frontend hotspot per Step 1.4)
- `subagent_type: frontend-architect`
- Scope:
  - **Magic numbers / strings**: must live in `*.constant.ts` (or equivalent constants module per CLAUDE.md)
  - **Selectors / DOM coupling**: avoid duplicated CSS class strings across modules; centralise
  - **Inline HTML in PHP**: PHP table classes / forms emitting `<div>...` should delegate to twig partials
  - **Twig changes**: i18n keys (cross-check with Pass B's translation rule), accessibility (alt text, aria-labels, button vs anchor for state changes)
  - **Bundle / asset entries**: check `*.entry.js` registrations match what's actually loaded
  - **Reusability**: components copy-pasted between merchant-portal / backoffice / yves
  - **State management**: shared mutable state on the window/global scope; event listener cleanup on widget destroy

### Pass C — Risk hotspots (only when Step 1 found applicable hotspots)
- `subagent_type: security-engineer` for ACL / multi-user / authz / IDOR / CSRF / session-trust diffs
- `subagent_type: backend-architect` for migration / post-deploy / data-integrity / FK-cascade / backfill-console diffs

### Pass D — Performance & N+1 (when diff touches `*Repository.php`, `*EntityManager.php`, business loops, console commands, or new query paths)
- `subagent_type: performance-engineer`
- Scope:
  - **N+1 queries**: queries inside `foreach`, `find()` calls in loops, `getX()` accessor chains that hit DB lazily, `findOneBy*` per row
  - **Missing indexes**: new FK without covering index, new WHERE/ORDER BY column without index
  - **Hydration cost**: full-table `find()` without `findEach`/batching/`select()` projection
  - **Eager vs lazy loading**: missing `joinWith`/`useXQuery` where multi-table access is obvious
  - **Caching**: cache invalidation on writes, missing cache on hot reads, stale cache contracts
  - **Blocking I/O in request paths**: HTTP calls, file reads, large queries in controller flow without timeouts
  - **Idempotency** for write operations re-runnable safely
  - Output should include estimated impact (e.g. "1+N queries per merchant — at 1000 merchants ≈ 1001 queries").

### Pass E — Test quality & coverage (always run if any tests added/modified, or if Pass A/C flagged untested hotspots)
- `subagent_type: quality-engineer`
- Scope:
  - **Real assertions**: no `assertTrue(true)`, no assertion-free tests; per project CLAUDE.md
  - **Hotspot coverage**: every Critical/Major finding from other passes — does a test exist that would have caught it?
  - **Pyramid balance**: too many slow Acceptance tests vs missing Unit tests
  - **Edge cases**: nulls, empty collections, boundary values, error paths, concurrent writes (where relevant)
  - **Snapshot/contract tests** for new public Facade methods
  - **Static cache contamination** patterns (per CLAUDE.md "Test Splitting" section): `_before()` clearing, `StaticCacheHelper` usage where shared static state exists
  - Open the actual test files; spot-check assertions; don't trust file presence alone.

### Pass F — Public API / contract changes (only when diff touches `*FacadeInterface.php`, `*PluginInterface.php`, `*.transfer.xml`, or GLUE API resources)
- `subagent_type: backend-architect`
- Scope:
  - **Removed methods** from a Facade interface — breaking change; flag Critical
  - **Renamed methods** without deprecation alias — breaking
  - **Signature changes** (param removed, type narrowed, return type changed)
  - **Removed transfer fields** — silent breakage in consumers calling `getX()`
  - **Renamed transfer fields** — same
  - **Removed plugin extension points** — breaks downstream Pyz / customer plugins
  - **GLUE API**: removed/renamed resources, response shape changes
  - For every breaking change: list affected callers (`grep -rn 'methodName' vendor/ src/ tests/`) and propose a deprecation path.

## Step 4: Validator pass (always run; serial after Step 3 completes)

Spawn ONE more agent (`subagent_type: general-purpose`) that:
- Receives the synthesized punch list (not the raw agent outputs).
- Re-reads each cited file at the cited line and decides **CONFIRMED / DISPUTED / INCONCLUSIVE** for every Critical and Major item; spot-checks ~30% of Minor.
- Specifically watches for failure modes:
  - Auth findings where a route-level ACL exists in `config/Zed/acl.rules.php` or similar
  - Bridge findings where the import IS already a Pyz Bridge interface
  - Communication-layer ORM findings where access actually goes through `getQueryContainer()`
  - Generated-constant references (`SpyXEntityTransfer::CONST`) the reviewer couldn't open
  - Dead-code findings where the caller is in a non-PHP file (twig, JS) the reviewer didn't grep
- Returns adjustments: severity changes, false-positive list, plus any **new issues** spotted while validating.

The validator's output is folded into the final synthesis. Disputed findings are kept but flagged.

## Step 5: Synthesize and persist

Consolidate findings into a single punch list grouped by severity (Critical / Blocker / Major / Minor / Nit), deduplicated across passes, each item with `file:line` and the violated rule. Include:
- Validator status per Critical/Major (Confirmed / Disputed / Inconclusive)
- Highlights worth keeping (good patterns; brief)
- Open questions for the implementer
- Skipped / not reviewed
- Recommended next steps

**Always save the synthesis to a file:**
```
reviews/<TICKET>-spryker-review.md     # if a ticket key is detectable
reviews/<short-sha>-spryker-review.md  # otherwise
```
Create the `reviews/` directory if it doesn't exist. Show the user the punch list summary inline AND the file path.

**Synthesis file MUST start with YAML frontmatter** so downstream tooling (CI gate, dashboards, PR-bot) can parse it:

```yaml
---
spryker_review_version: 1
ticket: ABC-123                       # or null
commit: ef0d08c75
branch: abc-123-feature-branch
mode: full                            # full | light | hotspots-only
passes_run: [A, B, C1, C2, D, E, F, validator]
verdict: block_merge                  # block_merge | proceed_with_caveats | clean
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

The YAML schema is versioned via `spryker_review_version`. Bump only on breaking schema changes.

## Step 6: Exit criteria (review-complete check)

The review is "complete" when **all** of:
1. All Critical/Blocker items have an explicit decision: accepted with rationale, deferred to a follow-up ticket, or handed to the implementer to fix
2. All Major items are at minimum triaged (decision made, even if "deferred")
3. Validator pass has run and adjustments are applied
4. Synthesis file is saved

Note: actually applying fixes, running `/verify`, and running `/test` are **the implementer's responsibility, not this command's**. Show the user a checklist when they ask "is the review done?"

## Boundaries

**This command WILL:**
- Read code and configs
- Spawn read-only review agents
- Produce a synthesis file at `reviews/<ticket-or-sha>-spryker-review.md`
- Suggest fixes in writing within that file

**This command WILL NOT:**
- Edit any source file
- Apply fixes (even if the user says "fix the blockers" — direct them to `/implement` or a manual edit session instead)
- Run `/verify` or `/test` on the user's behalf
- Commit, push, or comment on PRs/MRs

If the user wants fixes applied after the review, they invoke `/implement` (or an explicit edit session) separately, using the synthesis file as input.

## Examples — when to use which mode

| Scenario | Invocation | Why |
|---|---|---|
| Iterative local edit; quick sanity check on what you just committed | `/spryker-review --light` | Skips perf/test/contract/migration passes; ~4 agents; fast feedback |
| Pre-merge review of a major feature branch (like ABC-123) | `/spryker-review:review` (full mode, default) | All applicable passes including perf, tests, API contracts |
| Reviewing a specific MR by number | `/spryker-review MR 1234` | Resolves via `glab`; otherwise same as full |
| Reviewing a ticket whose branch you have checked out | `/spryker-review ABC-123` | Adds ticket grounding (acceptance criteria comparison) |
| Quick auth-only check on an ACL diff | `/spryker-review:review` then answer "hotspots-only" at Step 1.6 | Runs only Pass C1; ~2-3 agents |
| Big diff (>5,000 LOC) | Same as full, but Step 1.5 will warn — consider splitting per module first | Single review on huge diffs produces unfocused output |
| Pure refactor (no schema, no auth, no perf concern) | `/spryker-review --light` | Full mode would spin up D/E/F passes that have nothing to do |
