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
| **Deploy / runtime smoke** | `config/post-deploy/*.yml` added/modified, OR a console newly registered in `ConsoleDependencyProvider`, OR a `*DependencyProvider.php` diff that **removes/moves** an `add*` method (detect via `git diff <range> -- '*DependencyProvider.php' \| grep -E '^-.*function add'`) | G |

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

8. **Pre-flight: two diff-level questions** — before applying any pattern-match rule (rules 1–7), every Pass A / B / D / G agent MUST enumerate the answers to these two questions about the diff and include both enumerations near the top of its response. Findings that surface from these enumerations outrank later pattern-match findings.

   **Q1 — What invariants does this diff assert?**
   List every "must hold" property the diff introduces or relies on: uniqueness ("at most one X per Y"), totality ("every Y has at least one X"), default-presence ("every Y has exactly one X with `flag=true`"), referential ("every X.fk_y resolves"), monotonicity, conservation, **first-pick determinism** (any `findOne` / `reset()` / `[0]` over a multi-row source needs an ordering whose key is unique on the result set). For each invariant, enumerate every code path that **writes** the affected state. Then prove each writer establishes or preserves the invariant. Asymmetry across writers (e.g. update path sets the flag, create path doesn't) ⇒ Critical.

   **Q2 — What identifiers does this diff remove, rename, or relocate?**
   List every removed / renamed / moved DI key, method, transfer field, schema column, plugin registration, route, container key, or default-value contract (this includes *implicit* removal — e.g. a partial DTO whose `toArray()` emits absent fields as `null`, deleting them on the deserialiser side). For each, grep all consumers in scope plus parent classes / inherited factories. Any consumer not explicitly updated ⇒ Critical (broken contract).

   These two questions are framework-agnostic; Spryker-specific instances (DI container-layer separation, `AbstractTransfer::toArray()` null-emission, partial-unique-index workarounds, backfill-vs-runtime parity) are *examples* of the questions, not separate rules. If a Spryker-specific instance is the dominant risk in this diff, name it explicitly under the relevant question.

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
| G — Deploy & runtime smoke | ✅ if Deploy/runtime hotspot | ✅ if Deploy/runtime hotspot | per Step 1.4 (DP `add*` removed/moved, new console, new post-deploy YAML) |
| Validator | ✅ always | ✅ always | always |

**Worst-case agent count:**
- Full mode: 9 parallel + 1 validator = **10 agents** (G fires only when its narrow trigger matches; on most reviews 8 parallel)
- Light mode: 3 parallel (A + B + C1) + 1 validator = **4 agents** (+ G if its trigger fires)
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
  - **State-transition coverage**: for every new/modified Facade method, every new/modified Plugin, and every new column with semantic meaning (`is_default`, `is_disabled`, `fk_*`):
    1. List every code path that **writes** the column / triggers the state transition
    2. List every **invariant** the column carries (e.g. "every active user has exactly one `is_default=true` row")
    3. For each (write-path × invariant), assert a test exists that exercises the path AND asserts the invariant holds afterwards
    If no test exists ⇒ flag with sev=Major (untested state transition), even if the production code looks correct. Subsumes the older "every Critical/Major finding has a test" check.
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

### Pass G — Deploy & runtime smoke (only when Deploy/runtime hotspot triggers per Step 1.4)
- `subagent_type: backend-architect`
- Goal: walk every command in any added/modified `config/post-deploy/*.yml` AND every console newly registered in `ConsoleDependencyProvider`, AND every `add*` method removed/moved between `provideBusinessLayerDependencies` / `provideCommunicationLayerDependencies` / `provideClientDependencies`. For each:
  1. **Container-key trace.** Identify every container key the command's factory chain reads (grep up from `Console::execute` through `CommunicationFactory` + Bridges + parent factory). For each key, confirm the corresponding `add*Dependency` exists in the SAME layer's DependencyProvider AS OF THIS COMMIT — grep the diff for any `add*` removal/move that breaks the chain. **Spryker container keys are NOT shared across layers** (Business / Communication / Client containers are separate); a method moved from Communication to Business breaks every Communication-layer factory still resolving its key.
  2. **No-dev image safety.** Confirm the console's class file ships in the production image (i.e. NOT `require-dev` only, NOT under an `APPLICATION_ENV === 'development'` guard whose package isn't in `require`). Re-affirm the project CLAUDE.md `APPLICATION_ENV` gotcha.
  3. **Post-deploy task realism.** For each post-deploy task: confirm `execute_on` includes `prod` (and `pipeline` where relevant); confirm `timeout` is realistic for un-batched per-row work (default 900s often too tight); confirm task ordering vs migration apply.
- Output: per-command checklist with PASS / FAIL / UNKNOWN per step. Any FAIL ⇒ Critical (deploy-time crash or post-deploy data drift).

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
- **Cross-pass synthesis checks** (mechanical; raise NEW findings if matched):
  - **Runtime parity.** If Pass C2 found a backfill console / migration / post-deploy task that fixes a column or state, search Pass A/B findings for the matching runtime creation path. If no Pass A/B finding asserts the runtime path also sets it ⇒ raise a NEW Critical "one-shot fix without runtime parity" (e.g. `BackfillMerchantAclGroupFkMerchantConsole` exists, but runtime `AclEntityCreator::createAclEntitiesForMerchant` doesn't `setFkMerchant` — new merchants leak ACL).
  - **DI consistency.** If Pass A flagged an `add*` removal/move in any DependencyProvider, confirm Pass G ran AND its container-key trace passed. If Pass G didn't run (trigger missed it) ⇒ run the trace yourself: grep all consumers of the moved key; if any consumer is in a different layer's factory ⇒ raise NEW Critical "container-key resolution broken across layers".
  - **Invariant symmetry.** For every "missing X on creation path" finding (i.e. asymmetry surfaced by rule 8 Q1), check the symmetrical update / unassign / delete handlers in the same module. If any are also missing the X-handling ⇒ raise NEW Major per missing handler.
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
passes_run: [A, B, C1, C2, D, E, F, G, validator]
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
