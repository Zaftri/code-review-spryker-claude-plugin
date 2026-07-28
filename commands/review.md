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
   - Detected hotspots (using the canonical patterns in **Step 1A** below)
4. **Ticket-vs-diff sanity check:** call out anything in the diff that doesn't seem to map to the ticket, AND any acceptance criteria that look unaddressed.
5. **Diff-size sanity guard:** if total LOC changed (`git diff --shortstat <range>`) exceeds **5,000**, warn the user that this is too large for a single thoughtful review and suggest reviewing per-module or per-commit. Do NOT block — just warn.
6. **Ask the user:** *full review (all applicable passes), hotspots-only, or a specific module / concern?* Wait for confirmation before launching agents.

> **Note on numbering:** the lettered sections below (`Step 1A`–`Step 1D`) are reference material for Step 1. They are lettered, not numbered, so that cross-references to them never collide with Step 1's own numbered list items 1–6 above.

## Step 1A: Hotspot patterns (canonical, referenced by Pass C/D/E/F triggers)

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
| **Shared read-request builder** | a `*Mapper` / criteria / request-DTO builder (e.g. `mapToTorskRequest`, `*CriteriaBuilder`, `*RequestBuilder`) that gains a field, default, or eager-join (`set*`, `->fromArray`, a new default-injection guard) **and is called by ≥2 consumers** — typically a list/fetch path AND a count/exists path. Detect: `grep -rn '->methodName('  src/` for the changed builder method and list every caller | D |
| **Tests** | `tests/PyzTest/**/*.php`, `tests/PyzTest/**/*.feature` | E |
| **Feature-flag removal** | a `src/Pyz/Shared/FeatureFlag/*` diff with `^-` on a flag constant, a `DEFAULT_VALUE_MAP` entry, or a `ZED_MENUS_BY_FEATURE_FLAG` entry (detect: `git diff <range> -- 'src/Pyz/Shared/FeatureFlag/*' \| grep -E '^-.*(const\|=>)'`) | B + E (always — the source deletion is the easy half; see 9o) |
| **Frontend (heavy)** | `src/Pyz/**/Presentation/assets/{js,ts,scss}/**` where `git diff --numstat <range> -- 'src/Pyz/**/Presentation/assets/**' | awk '{a+=$1+$2} END {print a}'` >= 200 | B' |
| **Deploy / runtime smoke** | `config/post-deploy/*.yml` added/modified, OR a console newly registered in `ConsoleDependencyProvider`, OR a `*DependencyProvider.php` diff that **removes/moves** an `add*` method (detect via `git diff <range> -- '*DependencyProvider.php' \| grep -E '^-.*function add'`) | G |

## Step 1B: Out-of-scope (do not waste tokens reviewing)

These are explicitly **not reviewed** by any pass:
- Vendor code under `vendor/`
- Generated transfer objects (the `.transfer.xml` IS reviewed; the generated PHP is not)
- Generated Propel entities (`src/Orm/Propel/*/Base/`, `*EntityTransfer.php`)
- Generated migration SQL bodies (the schema XML and migration `up`/`down` ordering ARE reviewed; SQL syntax inside is not)
- Translation CSV *content* spelling (existence, completeness across locales, and code references ARE reviewed — see rule 7)
- Generated locale bundles (`src/Pyz/Zed/**/Presentation/Components/app/locale/**/*.json`) — build artifacts mirroring the translation CSVs. Not reviewed, and critically **must never be counted as a translation-key usage site** (see rule 7's detection note)
- Fixture data shape (`data/import/fixtures/`)

## Step 1C: Degradation / failure modes

Tools this command relies on may be unavailable, and dispatched agents can fail. Document the degradation; do NOT abort the review:

| Tool / component | Used for | If unavailable or failed |
|---|---|---|
| `mcp__context7__query-docs` | Spryker convention lookup | Fall back to `WebFetch` against `docs.spryker.com`; flag in synthesis: "context7 unavailable — Spryker rule citations not verified" |
| `glab` CLI | Resolving GitLab MR scope | Fall back to git only (`git diff` for the local branch); skip MR-specific scope |
| `mcp__atlassian__getJiraIssue` | Ticket grounding (Step 1, item 2) | Skip ticket fetch silently; reviews still proceed without intent comparison |
| `/verify` skill (PHPStan/Psalm) | Cross-checking dead-code findings | Agents flag dead code from grep alone; mark such findings "(unverified by static analysis)" |
| `reviews/` directory writable | Persisting synthesis (Step 5) | If write fails (e.g. `.gitignore` plus permission denied), output the synthesis inline and warn the user |
| **A dispatched review agent (Pass A–G or validator)** | One review dimension | **A pass that returns empty, errors, or is skipped mid-run is NOT the same as a pass that found nothing** — that ambiguity is the exact false-negative shape rule 9h flags in test skip-guards, and this command must not commit it either. Record the pass in `passes_degraded` (Step 5 frontmatter), state it in the "Skipped / not reviewed" section with the reason, and NEVER let a degraded pass contribute to a `clean` verdict. If the **validator** fails, the verdict is capped at `proceed_with_caveats` and every Critical/Major stays `INCONCLUSIVE`. Retry a failed pass once before recording it as degraded. |

## Step 1D: Severity tiers (canonical — referenced by Steps 3, 4, 5, 6)

Five tiers, highest first. Every finding carries exactly one. **No other severity words are permitted** anywhere in a pass output or the synthesis.

| Tier | Meaning | Gate |
|---|---|---|
| **Critical** | Correctness, security, or data-integrity defect. Ships a bug, breaks a contract, corrupts or leaks data. | Blocks merge |
| **Blocker** | Not a correctness defect, but violates a hard project law — a rule stated in `./.claude/CLAUDE.md` — or leaves a CLAUDE.md gate undischarged (e.g. the testing requirement). The code may work perfectly and still be a Blocker. | Blocks merge |
| **Major** | Real defect or convention violation with contained impact. Fix before merge, or open a tracked follow-up with an explicit decision. | Triage required |
| **Minor** | Genuine improvement with a concrete fix; no functional or process consequence if deferred. | Opportunistic |
| **Nit** | Preference or polish. Author may decline without justification. | None |

**Critical and Blocker are siblings, not a ladder** — they block merge for different reasons (correctness vs. project law) and are counted separately in the frontmatter. "At minimum a Blocker" for a CLAUDE.md violation means *Blocker or Critical*, never Major or below: a CLAUDE.md violation that ALSO causes a correctness defect is Critical. When unsure between the two, ask "would this be wrong even if CLAUDE.md didn't exist?" — yes ⇒ Critical, no ⇒ Blocker.

Both Critical and Blocker are exempt from the Step 3 signal-budget cap and from the "drop lowest-severity" rule. `verdict: clean` requires zero Critical **and** zero Blocker **and** zero degraded passes.

## Step 2: Rule sheet (always loaded into every review agent)

Every dispatched agent MUST receive these as ground rules, in priority order:

1. **Project rules** — the relevant sections of `./.claude/CLAUDE.md` (path relative to the project root in which this command is invoked):
   - Code Style (visibility, line length, native types on constants, magic numbers, interface scope)
   - Static Analysis & Testing requirements
   - Propel command order
   - Deployment / post-deploy gotchas
   - If the file is missing, agents should note "no project rule sheet found" in their output and continue with Spryker conventions only.
   - **CLAUDE.md rules are CRUCIAL and authoritative.** Anything stated in `./.claude/CLAUDE.md` is a hard project law, not a preference. A violation of a CLAUDE.md rule is **at minimum a Blocker** — meaning Blocker or Critical per the Step 1D tier definitions, never Major or below — is **exempt from the Step 3 signal-budget cap and the "drop lowest-severity" rule**, and the **validator MUST NOT downgrade or overturn it as "team taste"** — cite the exact CLAUDE.md line. Conversely, do NOT invent stricter rules than CLAUDE.md states and then confidently DEFEND code with them: when the review's conclusion (e.g. "no interface needed here", "this comment should stay") merely *matches* CLAUDE.md but a human reviewer might plausibly request the opposite, record it as an **open question for the implementer**, not a settled Confirmation. State the rule, cite the line, and flag the tension — never assert the code is unimprovable.
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
   - **Minor**: unreachable defensive guard — a newly-added null/empty check on a value that cannot hold that state at this point (e.g. `if ($x->getId() === null)` where every producer of `$x` sets the id, or a `?: null` after an `=== []` guard that already narrowed the type). Ask "can this condition actually occur?" and name the producer(s) that prove it can't. **A passing test that exercises the branch does NOT prove the branch is reachable in production** — hand-built fixtures can construct impossible states, so test coverage must never be cited as evidence that a defensive branch is necessary.
   - **Nit**: bare `// TODO` / `// FIXME` without ticket reference (acceptable: `// TODO: ABC-123 remove after deployed`)

   **Detection tactics for the agent:**
   - Cross-check against PHPStan/Psalm output via the `/verify` skill — tooling is authoritative; don't double-flag
   - For Facade public surface: `grep -rn '::methodName\|->methodName' src/ tests/` — flag if only callers are within the module's own Business/ or only tests
   - For translations:
     ```bash
     # Keys added OR removed-from-use in this commit
     git diff <range> -- 'data/translation/Zed/*.csv' | grep '^+[^+]' | cut -d, -f1 | sort -u
     # Cross-reference with REAL source usage — restrict by extension, never grep src/ blindly:
     grep -rn "<key>" src/ --include='*.twig' --include='*.php' --include='*.ts' --include='*.html'
     ```
     **Never count compiled locale bundles as usage.** Merchant-Portal modules ship generated
     JSON translation bundles (`src/Pyz/Zed/**/Presentation/Components/app/locale/**/*.json`)
     that mirror the CSV wholesale. An unrestricted `grep -r <key> src/` therefore hits the
     bundle for *every* key, including orphaned ones — the cross-reference silently confirms
     itself and **no dead translation is ever found in any MP module**. A key with **0 real-source
     refs but N bundle hits IS orphaned**. Verify by contrast: a genuinely-used sibling key shows
     at least one non-JSON hit (e.g. `foo.table.title` → 1 twig ref, while `foo.label` → 0).
     The same caution applies to any other generated mirror of a source-of-truth file.
   - For transfer fields: match each new XML `<property name="X">` against `getX`/`setX` in PHP
   - For schema columns: match each new `<column name="X">` against `filterByX`, `getX`, `setX` in repositories/entity managers

8. **Pre-flight: diff-level questions** — before applying any pattern-match rule (rules 1–7), every Pass A / B / B' / D / G agent MUST enumerate the answers to the questions below that apply to its scope, and include those enumerations near the top of its response. Findings that surface from these enumerations outrank later pattern-match findings.

   **Q1 — What invariants does this diff assert?**
   List every "must hold" property the diff introduces or relies on: uniqueness ("at most one X per Y"), totality ("every Y has at least one X"), default-presence ("every Y has exactly one X with `flag=true`"), referential ("every X.fk_y resolves"), monotonicity, conservation, **first-pick determinism** (any `findOne` / `reset()` / `[0]` over a multi-row source needs an ordering whose key is unique on the result set), **shared-state reset symmetry** (a `BehaviorSubject`/`Subject`/observable mutated by one event handler must be reset by every handler that logically clears it — a manual-clear path that clears a *sibling* signal but leaves the shared subject holding its last value replays that stale value into any later-mounting subscriber). For each invariant, enumerate every code path that **writes** the affected state. Then prove each writer establishes or preserves the invariant. Asymmetry across writers (e.g. update path sets the flag, create path doesn't) ⇒ Critical. **Default-injection guards:** for any default supplied through a falsy check (`if (!$x)`, `empty($x)`, `$x ?: default`, or `$x ?? default` where `$x` can legitimately be `0` / `'0'` / `''` / `false`), enumerate the falsy boundary explicitly — does a legitimate zero/empty/false value get silently overwritten with the default? A `!$transfer->getMaxNum...()` guard that rewrites an intentional `0` / `'0'` to `'1'` (recall PHP treats the string `'0'` as falsy) is a **Major correctness bug**, not a Nit. State whether callers can supply the falsy value and whether that value is semantically meaningful.

   **Q2 — What identifiers does this diff remove, rename, or relocate?**
   List every removed / renamed / moved DI key, method, transfer field, schema column, plugin registration, route, container key, or default-value contract (this includes *implicit* removal — e.g. a partial DTO whose `toArray()` emits absent fields as `null`, deleting them on the deserialiser side). For each, grep all consumers in scope plus parent classes / inherited factories. Any consumer not explicitly updated ⇒ Critical (broken contract).

   **Q3 — What value does this diff render in more than one place?**
   List every value the diff transforms for output where another code path already transforms the *same source value* for a different sink (table cell ↔ export row, list ↔ detail, UI ↔ API response, read model ↔ write model, two formatters of one quantity). For each, enumerate the full transform the existing sink applies — lookup/mapping, formatting, rounding, flag-gated remap, escaping, fallback default — and prove the new sink applies the same set, or diverges for a stated reason. Silent divergence ⇒ the new output contradicts what the user already sees ⇒ Major (Critical if the two are shown side by side, or one is claimed to mirror the other). A new sink copied from an *older* revision of the existing one is the usual origin — diff the branches, not just the method names.

   **Q4 — Where does data cross a format boundary?**
   List every point where a value crosses into a different format, language, or process: URL / query string, HTML, CSV / spreadsheet cell, SQL, shell, JSON, regex, log line, filename / path. For each, confirm the value is encoded with the encoder for *that exact target* — not a looser superset (e.g. whole-URL vs single-component encoders) — and that any leading character the sink interprets is neutralised (spreadsheet formula triggers, control / format chars, path traversal). A missing or over-broad encoder ⇒ corruption (wrong data, silently) or injection. Severity by sink and source trust.

   **Q5 — What does this diff acquire that must later be released?**
   List every long-lived resource the diff opens: subscription / observable, event or DOM listener, timer / interval, file or network handle, DB cursor, lock, registered callback. For each, prove a symmetric release exists on the owner's teardown path (destroy / unsubscribe / close / `finally`) and that the release runs on *every* exit, including error paths. An acquire with no matching release ⇒ leak (Major); released only on the happy path ⇒ Major.

   **Q6 — What in this diff can fail or run unboundedly, and how is that observed?**
   For every operation that can fail or stream an unbounded result: (a) confirm it is launched through a channel that can *observe* the outcome — a fire-and-forget trigger that structurally cannot read an error (navigation-style download, detached request) hides failures; (b) confirm user-visible progress / done / error state is bound to the *actual* result, not a fixed timer or optimistic assumption; (c) for streamed or paged output, confirm incremental flushing, a timeout budget covering *every* layer (app + proxy / web server), and that all validation able to fail runs *before* the first byte is committed — once headers / BOM are sent, a mid-stream guard cannot surface. Every guard or error branch the diff adds here must have a test that triggers it (feeds Pass E).

   These questions are framework-agnostic; concrete instances are *examples*, not separate rules — Q1: DI container-layer separation, `AbstractTransfer::toArray()` null-emission, partial-unique-index workarounds, backfill-vs-runtime parity; Q2: removed DI keys / transfer fields / plugin registrations; Q3: table-cell vs export-cell parity under a feature flag; Q4: `encodeURIComponent` for query params, CSV formula-injection neutralisation; Q5: Observable / `takeUntilDestroyed` teardown, event-listener cleanup on widget destroy; Q6: `StreamedResponse` flush + proxy timeout budget, HTTP-status-aware download. If a concrete instance is the dominant risk in this diff, name it explicitly under the relevant question.

9. **Spryker idiom & convention checks** — deterministic, grep-detectable conventions that human Spryker reviewers reliably flag and authors reliably fix. These are usually Minor/Nit but are **exempt from the Step 3 signal-budget cap and the "drop lowest-severity" rule** (see Step 3) — report *every* instance, because they are cheap to detect and cheap to fix, and missing them is exactly how a review loses credibility vs. a human reviewer. Each is owned by the pass noted; see `spryker-conventions` skill for the rationale.

   - **9a — `@api` annotation at Pyz (Pass A/F).** `@api` is a Spryker **Core** marker for a module's published public API. Pyz application interfaces (`src/Pyz/**Interface.php`) are not Core public API — flag any **added** `@api` docblock tag in a `src/Pyz/**` interface/class and recommend removal. Detect: `git diff <range> -- 'src/Pyz/**Interface.php' | grep -nE '^\+.*@api'`. Report **all** occurrences (the human comment "remove here and in N more places" is one finding per site).
   - **9b — Mapping logic belongs in a Mapper (Pass A/B).** Inline transfer→transfer / array→transfer / entity→transfer mapping (≥~3 lines, or ≥2 such methods) inside an Expander / Controller / Resolver / Hydrator / Reader → flag "extract to a dedicated `*Mapper`" (not a generic "builder"/"helper" — name the Spryker **Mapper** pattern). One finding per method that should move.
   - **9c — Exceptions are not control flow (Pass A/B).** In Zed, flag `throw`/`...OrFail()` used to signal an **expected** "not found / empty" outcome, and any `try { } catch (\Exception ...)` that downgrades a thrown exception into a default/empty return. Spryker handles exceptions in a central handler and they must not steer the workflow. Cite https://docs.spryker.com/docs/dg/dev/backend-development/zed/business-layer/custom-exceptions. Prefer a nullable return / explicit "exists" check over throw-then-catch.
   - **9d — Test-support placement (Pass E).** In `tests/PyzTest/**`, flag any non-test code (fixture builders, transfer/entity graph assembly, data-setup helpers, factory wiring, `have*`/`create*`/`build*` data helpers, shared response builders) living in the test case itself — it belongs in the Codeception **Tester** / Helper / `_support` class. **This is a repeat human-reviewer comment ("Please move to the tester everything that is not a test or mock creation") — rate it Major, not Nit,** so the author moves it before the MR reaches a reviewer. Carve-out: **mock creators specific to a single test class** (`createControllerMock()`, `createFactoryMock()` returning `createMock(...)`) may stay private in the case. Strong signal: any builder that assembles a domain transfer/entity graph, OR that is **duplicated across ≥2 test files** (grep the method name across `tests/`), is definitively misplaced ⇒ recommend one parameterized Tester action (`$this->tester->have*/create*`) and note the rebuild step `vendor/bin/codecept build -c tests/PyzTest/<Layer>/<Module>`. One finding per misplaced helper block; report ALL (exempt from cap).
   - **9e — One-shot data fix simplicity (Pass A/B/C2).** When a backfill **console + a new Facade/EntityManager method** is added solely for a one-time data migration, raise a YAGNI question: could it be the migration's `postUp()` (or at least a thin console without a dedicated facade method)? Flag the speculative public-API surface created for a throwaway task.
   - **9f — Systematic least-visibility sweep (Pass A/B).** This is a *complete enumeration*, not a spot-check. For **every** method and factory `createX`/`getX` added or changed in the diff, grep callers across `src/` and `tests/` (`grep -rn '->methodName\|::methodName'`). No caller outside the declaring class ⇒ recommend `private`; callers only within the same module ⇒ recommend `protected`; `public` only if an external caller exists. Report every method whose declared visibility is wider than its actual usage.
   - **9g — Domain-logic traits hide dependencies behind docblock hints (Pass A).** A trait used for domain/business logic (Communication controllers, view-data assembly, resolvers) that declares its required collaborators only via `@method` class-level docblock hints — rather than `abstract` method declarations or explicit constructor injection — lets a composing class omit a required method and fail only at runtime (`Call to undefined method`), invisible to PHPStan. Detect: `git diff <range> -- 'src/Pyz/**/*Trait.php' | grep -B5 '@method'` — if the trait lives outside a Kernel/infrastructure namespace and its `@method` hints reach a facade/factory (`getFactory()`, `->facade()`), flag it. Stateless Kernel-level mixins (e.g. `FeatureFlagAwareTrait`) are the accepted exception; a trait assembling module view-data or business decisions from injected facades is not. Preferred fix: extract into a dedicated class (a `*Provider`/`*Service`) with facades wired through the module's DependencyProvider; minimal fallback: declare the required methods `abstract protected` so the contract is enforced statically instead of documented.
   - **9h — Side-effect-inferred test conditionals (Pass E).** A test conditional — `markTestSkipped()`, an `if`/`else` branch, or a boolean property assigned once in `_before` and branched on later — driven by an OBSERVED SIDE EFFECT (DOM presence/absence, element count, an attribute value, response shape) rather than by the actual controlling state (a feature-flag client call, a config value). Two distinct failure modes, both reportable:
     - **Can't distinguish "off" from "broken".** A renderer that silently fails with the flag ON looks identical to flag-OFF, so the test reports SKIPPED (or takes the wrong branch) instead of FAILED — CI stays green while the feature is completely broken. Also confirm the flag is actually exercised in CI: a flag defaulting off with nothing enabling it in the pipeline means every gated test silently skips forever.
     - **Becomes dead scaffolding once the flag is gone.** After the inferred flag is removed and its ON path made unconditional, the condition is *constant* and the other branch is unreachable — dead code a flag-removal MR must delete (9o(a)). A DOM-inferred boolean is especially easy to miss here because nothing references the flag name any more, so a flag-name grep comes back clean while the scaffolding remains.

     Detect both: `grep -rn 'markTestSkipped\|grabMultiple\|grabAttributeFrom' tests/PyzTest/**/*Cest.php`. For each hit, first confirm whether the condition queries the flag/config directly (`getLocator()->featureFlag()->client()->isEnabled(...)`) rather than inferring it from the page (`!$i->see(...)`, `count($i->grabMultiple(...)) === 0`, an absence-based DOM check); then ask whether the state it infers can still vary at all — if the source now hardcodes it (e.g. a twig attribute fixed to `"true"`), every branch keyed to it is dead.
   - **9i — Removed test scenarios without replacement (Pass E).** A spec/Cest rewrite that deletes a previously-covered interaction sequence without an equivalent replacement is a coverage regression even though the suite still "passes" — especially risky when the underlying implementation became MORE stateful/shared (e.g. per-instance state promoted to a shared singleton/observable), which makes the dropped sequence riskier, not safer, to leave untested. Detect: `git diff <range> -- '*.spec.ts' 'tests/**/*Cest.php' | grep -E '^-.*\bit\(|^-.*public function test'` and confirm each removed case has a like-for-like replacement elsewhere in the diff; if not, flag it — don't assume a rewrite preserved coverage just because the file still has tests.
   - **9j — Cross-module Propel Query access must go through the DependencyProvider (Pass A).** A `*PersistenceFactory.php` / `*Repository.php` / `*EntityManager.php` that directly `use`s an `Orm\Zed\<OtherModule>\Persistence\...Query` class from a module other than its own is bypassing the DependencyProvider — the owning module should expose this data via its Facade, injected through `add*Facade()`, not have a foreign module's raw Propel Query instantiated inside this module's Persistence layer. Detect: `grep -rn 'use Orm\\Zed\\' src/Pyz/Zed/<Module>/Persistence/*.php` and flag any hit where `<OtherModule>` ≠ the current module. Preferred fix: add the foreign module's Facade to this module's DependencyProvider and inject it, rather than constructing the other module's Query class directly.
   - **9k — Test namespace/suite must mirror the actual layer under test (Pass E).** A test file's declared `namespace` must mirror the layer of the class it tests (`Persistence` / `Business` / `Communication\Plugin\...`), not just where it happens to sit. When a module lacks a suite for the layer being tested (e.g. no `Persistence` suite exists), a repository test gets misfiled under an unrelated suite by necessity — flag both the namespace mismatch AND the missing suite as the root cause, recommending the suite be added (matching repo-wide precedent, e.g. `AgentRepositoryTest`, `CampaignRepositoryTest`) rather than the test permanently misfiled under a layer it doesn't belong to.
   - **9l — Missing modified-columns guard on event-triggered republish (Pass D).** A Publisher/Listener plugin that subscribes to an entity's create/update event to trigger a downstream read-model republish or notification, but republishes on *any* column change, over-triggers when the read model only depends on a subset of columns. Detect: `grep -rn 'getEventTransfersByModifiedColumns'` for sibling precedent in the repo — if similar classes already guard on modified columns for a comparable entity/read-model relationship and this new one doesn't, flag it and name the specific columns the read model actually depends on.
   - **9m — Facade / Service / Client entry-point classes are thin delegators (Pass A/B; specializes rule 6 SoC).** A method body in `*Facade.php` / `*Service.php` / `*Client.php` should be a single delegation to a factory-built model (`return $this->getFactory()->createX()->doWork($transfer);`). Any conditional, loop, inline transfer mapping, or multi-step orchestration in that body is logic belonging in the model the factory creates (`*Calculator` / `*Reader` / `*Writer` / `*Expander` / `*Mapper`). Detect: for every `*Facade.php`/`*Service.php`/`*Client.php` in the diff, scan each added/changed body for `if `, `foreach `, `->fromArray(`, or >1 statement, and compare against the **sibling methods in the same class** — they almost always show the correct one-line shape, which makes the violation self-evident. Branching / looping / mapping in the body ⇒ **Major**; benign multi-statement ⇒ Minor. "It needs to be callable from here" is not a reason to put logic here — keep the delegation, move the logic behind it.
   - **9n — Enrichment inside an overridden Core class when the Core module publishes an extension point (Pass A; specializes rule 3 OCP "plugin extension points where Spryker expects them" — that parent is judgment-only and never fired; this is its deterministic form).** When Pyz overrides a Core class (controller, data provider, table configuration provider, hydrator) and the override adds per-record enrichment of transfers it just received from a facade — a `foreach` over the result that derives or sets extra fields — first check whether the **owning** Core module publishes an extension point for exactly that data. Detect: `grep -nE 'add[A-Za-z]*(Expander|Hydrator|Mapper)Plugins' vendor/spryker*/src/Spryker/<Layer>/<OwningModule>/<OwningModule>DependencyProvider.php` for the module behind the facade method being called, plus `find src/Pyz -name '*ExpanderPlugin.php'` for sibling precedent already in this repo. If a stack exists ⇒ **Major**: the enrichment belongs in a `*ExpanderPlugin` registered in that stack, which usually removes the need to override the Core method at all. **The same enrichment duplicated across ≥2 Pyz overrides of Core classes is the strongest signal — and must NOT be resolved by extracting a shared helper both overrides call.** That duplication exists because the logic is in the wrong layer; a shared helper preserves the defect and adds indirection. Name the concrete stack and one existing sibling plugin in the suggested fix.
   - **9o — Feature-flag removal completeness (Pass B/E; specializes rule 7 dead code).** When a diff removes a released flag and keeps the ON path unconditionally, the source-side deletion is the easy half. Enumerate three consequences: **(a) test-side dual-path scaffolding** — conditionals in tests keyed to the removed flag's *observable* state are now constant and their other branch is dead (detection lives in 9h); **(b) coverage the flag was blocking** — behavior left untested *because* asserting it would have required controlling flag state (which typically can't be done per-test without flipping it globally) is now single-path and straightforward to test, so the removal MR is where that test lands, **not** a follow-up ticket; **(c) newly-unconditional tests** — a test whose skip guard was removed now actually runs in CI for the first time, so its timing and environment assumptions are being exercised for the first time (feeds 9p). Detect: `git diff <range> -- 'src/Pyz/Shared/FeatureFlag/*' | grep -E '^-.*(const|=>)'` to list removed flags, then for each, grep the behavior it gated for test references. Severity: (a) Major, (b) Major, (c) Major when a guard was removed from a Cest.
   - **9p — Test timing correctness (Pass E; new territory).** Two shapes. **(a) Instant assertion on a child of a just-awaited element:** `waitForElement(HOST)` followed within a few lines by `seeElement(HOST_CHILD)` where the asserted selector is *different and nested* — the host mounts before its projected/async children, so the assertion races rendering and fails intermittently while the element is plainly present in the failure artifact's page source. Convert the child assertions to `waitForElement`. **Scope this to Cests the diff touches** — a repo-wide sweep matches ~90 sites and is noise, not thoroughness — and report **one aggregate finding per Cest**, not per site. Prioritise Cests that lost a skip guard in this diff (9o(c)), since those are running unconditionally for the first time. Do NOT flag `waitForElement(X)` followed by `seeElement(X)` on the *same* selector; that is not a race. **(b) Uncontrolled timers in specs:** a component scheduling `setTimeout` / `requestAnimationFrame` / a detached promise whose sibling `*.spec.ts` has no `fakeAsync` / `tick()` — the timer fires after the test finishes, so the behavior is unasserted *and* the pending timer leaks into later tests. Detect: `grep -l 'setTimeout(\|requestAnimationFrame(' <changed>.component.ts`, then `grep -cE 'fakeAsync|\btick\(' <same>.component.spec.ts` — zero means flag it.
   - **9q — Compiler/analyzer escape hatch added without stated need (Pass A/B/B'; new territory).** Flag any **added** suppression that trades one local error for a file- or module-wide loss of checking: `CUSTOM_ELEMENTS_SCHEMA`, `NO_ERRORS_SCHEMA`, `@ts-ignore`, `@ts-nocheck`, a widening `: any`, `@phpstan-ignore-*`, `@psalm-suppress`, `// eslint-disable*`, `@SuppressWarnings`. `CUSTOM_ELEMENTS_SCHEMA` is the archetype: it disables unknown-element checks for *every* template in the module, so a typo'd component selector or a missing module import stops being a build error and instead renders an empty tag at runtime — silently missing UI. Require the MR to name the specific error being suppressed and to use the narrowest form available (line-scoped over file-scoped over module-scoped); if the build and tests pass without it, recommend removal along with its now-unused import. Detect: `git diff <range> | grep -nE '^\+.*(CUSTOM_ELEMENTS_SCHEMA|NO_ERRORS_SCHEMA|@ts-ignore|@ts-nocheck|@phpstan-ignore|@psalm-suppress|eslint-disable|SuppressWarnings)'`. **Fires only on `^+` lines** — pre-existing suppressions in modules the diff doesn't touch are out of scope, and many modules legitimately carry them. Severity: Major when the scope is module- or file-wide, Minor when line-scoped.

## Step 3: Dispatch review passes (parallel Agent calls)

Use the **Agent** tool. Send all independent passes in a SINGLE message with multiple tool calls.

### Mode: full (default) vs light

| Pass | Full mode | Light mode (`--light`) | Trigger (per Step 1A) |
|---|---|---|---|
| A — Architectural | ✅ always | ✅ always | always |
| B — Code quality + dead code | ✅ always | ✅ always | always |
| B' — Frontend (heavy) | ✅ if frontend hotspot | ❌ skipped | `Presentation/assets/**` numstat sum >= 200 (see Step 1A) |
| C1 — Security | ✅ if ACL/authz hotspot | ✅ if ACL/authz hotspot | ACL / authz / multi-tenant pattern |
| C2 — Backend-architect | ✅ if migration/post-deploy hotspot | ❌ skipped | migrations or post-deploy |
| D — Performance / N+1 | ✅ if perf hotspot | ❌ skipped | Repository/EntityManager/console loops, OR a shared read-request builder gains a field/default (Step 1A) |
| E — Test quality | ✅ if test hotspot or hotspots flagged | ❌ skipped | tests changed OR Pass A/C flagged untested code |
| F — API/contract | ✅ if API surface hotspot | ❌ skipped | Facade/Plugin/Transfer XML |
| G — Deploy & runtime smoke | ✅ if Deploy/runtime hotspot | ✅ if Deploy/runtime hotspot | per Step 1A (DP `add*` removed/moved, new console, new post-deploy YAML) |
| Validator | ✅ always | ✅ always | always |

⚠️ **Light mode cannot discharge the CLAUDE.md testing gate.** It skips Pass E, and "tests absent / assertion-free" is a **Blocker** per Step 1D (CLAUDE.md's testing requirements are project law). A `--light` run therefore structurally cannot detect one whole Blocker class. So: in light mode, Pass B additionally performs a **test-presence check only** — `git diff --name-only <range> -- 'tests/**'`; if the diff changes `src/Pyz/**` business or persistence code and touches no test file, raise a Blocker "CLAUDE.md testing gate undischarged — Pass E not run in light mode; full review required to assess coverage." State in the synthesis that light mode did not assess test *quality*.

**Worst-case agent count:**
- Full mode: 9 parallel + 1 validator = **10 agents** (G fires only when its narrow trigger matches; on most reviews 8 parallel)
- Light mode: 3 parallel (A + B + C1) + 1 validator = **4 agents** (+ G if its trigger fires)
- Hotspots-only (user picks at Step 1): 1–2 parallel + validator = **2–3 agents**

### Agent prompt requirements

Each agent prompt MUST:
- State the goal and the specific files in scope (don't make them re-discover).
- Include the Step 2 rule sheet inline. For Pass A/B/E, include the relevant **rule-9 idiom checks verbatim** and the grep commands — these are the findings most often missed, and they must be passed explicitly, not summarized away.
- Demand findings as `severity | file:line | primary rule | what's wrong | suggested fix`. List secondary rules in parentheses if relevant (e.g. `SoC (also: KISS, SRP)`); pick **one primary rule** per finding to avoid triple-counting.
- For security findings: include a Given/When/Then attack scenario.
- **Signal budget — strict:**
  - Only include findings that would change code or shipping decisions. Drop pure preference, drop "smell-only" with no concrete fix.
  - Cap each pass at **30 substantive findings**. If you would exceed 30, drop the lowest-severity items and note "N additional Nit-level items omitted for signal" at the end.
  - **Exception — Spryker idiom & convention checks (rule 9) are NEVER dropped for being low-severity.** They are deterministic, grep-detectable, and reliably flagged by human reviewers; report every instance even if Minor/Nit and even past the 30-finding cap. List them in a separate "Convention" block so they don't crowd out judgment-based findings. The "drop lowest tier" rule never applies to them.
  - **But the Convention block has its own ceiling: 40 items.** Several rule-9 checks are complete enumerations by construction (9f over every added/changed method; 9a over every added `@api`; 9d over every misplaced test helper), so on a large diff the exempt category can grow without bound — which reopens the signal problem from the other side. Past 40, **collapse per rule into one aggregate finding** rather than truncating: `9f — 23 further methods declared wider than their usage: <file:line list>`. This preserves completeness (nothing is silently dropped, the list is still actionable) while keeping the block readable. Never emit an aggregate for Critical or Blocker items.
  - Severity: use the five canonical tiers defined in **Step 1D** and no other words. If unsure between two tiers, drop one tier — except rule-9 items, which are reported at whatever tier they merit regardless.
- **Self-check every suggested fix against rules 1–9 before writing it down.** The fix is part of the review's output and is held to the same standard as the code. Before proposing "move it to X" / "extract to Y", verify the destination is architecturally legal: is X a thin entry-point class that must not hold logic (9m)? Does an existing Core extension point already own this (9n)? Does X's layer permit it? Does it mint speculative public API (9e) or an interface the project forbids (ISP)? **A fix a human reviewer would themselves comment on is worse than no fix — it launders one finding into a new violation.** *(The Step 4 validator re-checks this independently. That duplication is deliberate — this is a failure mode the review has actually shipped, so it gets two gates. The pass author is responsible for getting the fix right; the validator is the backstop and has final say. Do not "simplify" by removing either.)*
- Cap response length (~400-600 words per agent).
- **Forbid edits** — review only. Agents must NOT use Write/Edit tools.

### Pass A — Architectural / Spryker conformance
- `subagent_type: general-purpose`
- Scope: layer boundaries, facade-only cross-module access (per Bridge nuance in rule 2), factory conventions (`createX` protected/private, `getX` for cached deps), dependency provider correctness (constants, late-bound closures, plugin stacks), transfer object usage vs raw arrays, persistence isolated to `Persistence/`, no business logic in plugins/controllers, module public vs private API, Propel schema changes paired with migrations.
- **Also run rule-9 idiom checks (report all, exempt from cap):** 9a (`@api` removed from `src/Pyz/**` interfaces), 9c (exceptions not used as control flow in Zed), 9e (one-shot fix over-engineering), 9f (systematic least-visibility sweep over every added/changed method — do NOT spot-check), 9g (domain-logic traits with implicit `@method` dependency contracts — flag and recommend extraction to a dedicated class), 9m (Facade/Service/Client bodies are thin delegators), 9n (enrichment in an overridden Core class where an extension point exists — grep the owning module's DependencyProvider for a plugin stack before proposing any other fix), 9q (compiler/analyzer escape hatch added without stated need — `^+` lines only). 9b (mapping → Mapper) is shared with Pass B.
- Must use context7 `/spryker/spryker-docs` to verify any rule before citing it.

### Pass B — Code quality (SOLID / DRY / YAGNI / KISS / TDA / SoC / dead code / project style)
- `subagent_type: refactoring-expert` (or general-purpose)
- Scope:
  - **SOLID / DRY / YAGNI / KISS / TDA / SoC** per Step 2 rules 3–6
  - **Dead code & unused artifacts** per Step 2 rule 7 — explicitly run translation cross-reference and Facade-caller grep
  - **Project style** — visibility, native types on constants, magic numbers, comments restating code, line length > 120, interface scope
  - **Rule-9 idiom checks (report all, exempt from cap):** 9b (inline mapping → extract a `*Mapper`), 9e (one-shot console+facade that could be migration `postUp`), 9m (logic in a Facade/Service/Client body — compare against sibling methods in the same class), and contribute to 9f (least-visibility) alongside Pass A.
- For frontend changes < ~200 LOC, also includes JS/TS/SCSS smells (magic numbers, missing `*.constant.ts`, inline HTML in PHP). Heavier frontend goes to Pass B'.
- **Style sweep is mandatory and runs LAST, after the Q1–Q6 enumeration — never skipped because the diff "looks clean."** The pre-flight questions surface high-severity bugs and tend to crowd out low-severity checklist items; counter this mechanically. For every file in scope, enumerate each *new or edited* declaration and check it against the project Code Style list, reporting Nit-level misses even when Q-findings dominate the diff:
  - every new `const` → has a native type (`private const string FOO`), not bare `private const FOO`
  - every numeric/string literal in logic → extracted to a constant / `*.constant.ts`
  - every new method/property/const → least possible visibility
  - lines > 120 chars; comments that merely restate the code
  If a style item has zero misses, say so in one line; do not silently omit the sweep. (Origin note: in eval, untyped-constant misses were the single class the Q-questions failed to catch — this bullet exists to close that gap.)

### Pass B' — Frontend (only when frontend hotspot per Step 1A)
- `subagent_type: frontend-architect`
- Scope:
  - **Magic numbers / strings**: must live in `*.constant.ts` (or equivalent constants module per CLAUDE.md)
  - **Selectors / DOM coupling**: avoid duplicated CSS class strings across modules; centralise
  - **Inline HTML in PHP**: PHP table classes / forms emitting `<div>...` should delegate to twig partials
  - **Twig changes**: i18n keys (cross-check with Pass B's translation rule), accessibility (alt text, aria-labels, button vs anchor for state changes)
  - **Bundle / asset entries**: check `*.entry.js` registrations match what's actually loaded
  - **Reusability**: components copy-pasted between merchant-portal / backoffice / yves
  - **State management**: shared mutable state on the window/global scope; event listener cleanup on widget destroy
  - **Rule-9q — compiler/analyzer escape hatches (report all, exempt from cap):** any **added** `CUSTOM_ELEMENTS_SCHEMA` / `NO_ERRORS_SCHEMA` / `@ts-ignore` / `@ts-nocheck` / widening `: any` / `eslint-disable`. Module-wide scope ⇒ Major. Check whether the module compiled without it before the diff, and whether this diff actually introduces a new custom element — if not, recommend removal plus the now-unused import.

### Pass C — Risk hotspots (only when Step 1 found applicable hotspots)
- `subagent_type: security-engineer` for ACL / multi-user / authz / IDOR / CSRF / session-trust diffs
- `subagent_type: backend-architect` for migration / post-deploy / data-integrity / FK-cascade / backfill-console diffs

### Pass D — Performance & N+1 (when diff touches `*Repository.php`, `*EntityManager.php`, business loops, console commands, or new query paths)
- `subagent_type: performance-engineer`
- Scope:
  - **N+1 queries**: queries inside `foreach`, `find()` calls in loops, `getX()` accessor chains that hit DB lazily, `findOneBy*` per row
  - **Missing indexes**: new FK without covering index, new WHERE/ORDER BY column without index. Also applies without a schema-file diff: if the code diff introduces the FIRST query filtering by a NON-LEADING column of an EXISTING composite unique/index, confirm that column actually gets index support — Postgres composite indexes only accelerate leftmost-prefix lookups, so a `filterByFkX_In()` on the trailing column of a `(fk_other, fk_x)` unique constraint typically seq-scans even though the schema itself wasn't touched.
  - **Hydration cost**: full-table `find()` without `findEach`/batching/`select()` projection
  - **Eager vs lazy loading**: missing `joinWith`/`useXQuery` where multi-table access is obvious
  - **Caching**: cache invalidation on writes, missing cache on hot reads, stale cache contracts
  - **Blocking I/O in request paths**: HTTP calls, file reads, large queries in controller flow without timeouts
  - **Idempotency** for write operations re-runnable safely
  - **Over-fetch via shared request builder**: when a default, field, or eager-join is added to a request/criteria mapper used by multiple consumers, enumerate EVERY consumer (`grep -rn` the builder method) and check whether each actually reads the added data. A count-only / exists-only / autocomplete path that now carries `maxNum*`, `withDetails`, a joined relation, or history flags fetches and hydrates data it never uses ⇒ flag **Major** with the wasted-work path named (e.g. "`getFilteredIssuesCount()` routes through `mapToTorskRequest` and now requests file history the COUNT never reads"). Do NOT treat "all consumers share one builder" as automatically good — that symmetry is exactly what hides the over-fetch; the fix is usually to inject the default at the list-path call site, not inside the shared builder.
  - Output should include estimated impact (e.g. "1+N queries per merchant — at 1000 merchants ≈ 1001 queries").

### Pass E — Test quality & coverage (always run if any tests added/modified, or if Pass A/C flagged untested hotspots)
- `subagent_type: quality-engineer`
- Scope:
  - **Real assertions**: no `assertTrue(true)`, no assertion-free tests; per project CLAUDE.md
  - **Mock-of-the-behavior-under-test**: a test that mocks the exact repository/facade whose filtering or business logic IS the feature being verified only proves wiring (the right method was called with the right args), not that the behavior is correct — it would keep passing even if that collaborator's logic were broken. Distinguish this from legitimately mocking an unrelated dependency. Prefer direct instantiation with real facades + DB-backed fixtures for Communication-layer providers testing flag-gated business logic, matching the established direct-`new` + `FeatureFlagHelper` pattern already used elsewhere in the module's test suite.
  - **State-transition coverage**: for every new/modified Facade method, every new/modified Plugin, and every new column with semantic meaning (`is_default`, `is_disabled`, `fk_*`):
    1. List every code path that **writes** the column / triggers the state transition
    2. List every **invariant** the column carries (e.g. "every active user has exactly one `is_default=true` row")
    3. For each (write-path × invariant), assert a test exists that exercises the path AND asserts the invariant holds afterwards
    If no test exists ⇒ flag with sev=Major (untested state transition), even if the production code looks correct. Subsumes the older "every Critical/Major finding has a test" check.
  - **Guard-path coverage**: every defensive guard the diff ADDS — a `throw`, early-return, or error branch whose job is to protect an invariant or prevent a corrupt / truncated / partial result — must have a test that *triggers* it. Guards are the highest-value, least-covered code: the happy path exercises everything except them. Guard present but no test that drives it down the failure branch ⇒ Major.
  - **Structural-variant coverage**: when a feature/flag/component rolls out to N structurally-similar consumers (pages, modules, components), enumerate every axis of STRUCTURAL variation across them — nesting depth, sibling-vs-nested composition, inline rendering vs `ng-content`/slot projection — not just entity/CRUD-state variation. A single "representative" consumer chosen for being the simplest does not cover a differently-composed sibling; confirm at least one test exercises each distinct structural shape.
  - **Pyramid balance**: too many slow Acceptance tests vs missing Unit tests
  - **Edge cases**: nulls, empty collections, boundary values, error paths, concurrent writes (where relevant)
  - **Snapshot/contract tests** for new public Facade methods
  - **Static cache contamination** patterns (per CLAUDE.md "Test Splitting" section): `_before()` clearing, `StaticCacheHelper` usage where shared static state exists
  - **Rule-9d — test-support placement (report all, exempt from cap, rate Major):** flag non-test code in the test case (fixture builders, transfer/entity graph assembly, data setup, `have*`/`create*`/`build*` helpers, factory wiring) that belongs in the Codeception Tester / Helper / `_support` class. Mock creators specific to this one test class may stay. Any builder duplicated across ≥2 test files (grep the method name across `tests/`) is definitively misplaced. This is a recurring human-reviewer comment — flag it as Major so it is fixed before the MR is opened.
  - **Rule-9 idiom checks (report all, exempt from cap):** 9h (side-effect-inferred test conditionals — both failure modes: can't distinguish off-from-broken, AND dead scaffolding once the inferred flag is gone), 9i (removed test scenarios — a spec/Cest rewrite that drops previously-covered interaction sequences without an equivalent, especially when the underlying implementation became more stateful/shared), 9o (feature-flag removal completeness — dual-path test scaffolding, coverage the flag was blocking, newly-unconditional tests), 9p (test timing — instant child assertions after `waitForElement`, scoped to Cests in the diff and aggregated per file; plus components with unfaked `setTimeout` in their specs).
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

### Pass G — Deploy & runtime smoke (only when Deploy/runtime hotspot triggers per Step 1A)
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
  - **Symmetry-is-good rationalization** — when a pass praised a change as "shared / centralized / symmetric / all paths route through one builder", do NOT accept it at face value. That framing is a common blind spot: verify each shared consumer actually needs the shared behavior (see over-fetch cross-check below) before endorsing it.
  - **CLAUDE.md-backed findings** — never mark a finding CONFIRMED-then-downgraded or DISPUTED on the grounds of "team taste" when it cites a `./.claude/CLAUDE.md` rule; those are crucial (Blocker minimum). Equally, do not upgrade a "the code matches CLAUDE.md, so it's fine" note into a settled Confirmation when a human reviewer might request the opposite — re-file it as an open question.
  - **Suggested-fix legality.** Re-read the *fix* text of every Critical/Major/Minor finding, not only its diagnosis. Reject and rewrite any fix whose destination violates the rule sheet: logic pushed into a `*Facade`/`*Service`/`*Client` body (9m); a shared helper introduced to de-duplicate logic an existing Core extension point should own (9n); a new public Facade/Service method for a one-off (9e); a new interface outside Facade/Plugin/Service/Client (ISP). A correct diagnosis with an illegal fix still earns a reviewer comment — mark it **CONFIRMED-with-corrected-fix** and state the legal destination.
- **Cross-pass synthesis checks** (mechanical; raise NEW findings if matched):
  - **Runtime parity.** If Pass C2 found a backfill console / migration / post-deploy task that fixes a column or state, search Pass A/B findings for the matching runtime creation path. If no Pass A/B finding asserts the runtime path also sets it ⇒ raise a NEW Critical "one-shot fix without runtime parity" (e.g. `BackfillMerchantAclGroupFkMerchantConsole` exists, but runtime `AclEntityCreator::createAclEntitiesForMerchant` doesn't `setFkMerchant` — new merchants leak ACL). Existence of a runtime path is not sufficient on its own: if that runtime path's own *trigger* (e.g. a Publisher/Listener's republish condition) is itself gated by the SAME flag it exists to reconcile, check BOTH toggle directions — flag ON→OFF (rollback) and flag OFF→ON-after-initial-deploy (the one-shot backfill already ran as a no-op while off) — for staleness, not just whether the path exists at all. A trigger gated on its own reconciliation flag is a one-way valve.
  - **DI consistency.** If Pass A flagged an `add*` removal/move in any DependencyProvider, confirm Pass G ran AND its container-key trace passed. If Pass G didn't run (trigger missed it) ⇒ run the trace yourself: grep all consumers of the moved key; if any consumer is in a different layer's factory ⇒ raise NEW Critical "container-key resolution broken across layers".
  - **Invariant symmetry.** For every "missing X on creation path" finding (i.e. asymmetry surfaced by rule 8 Q1), check the symmetrical update / unassign / delete handlers in the same module. If any are also missing the X-handling ⇒ raise NEW Major per missing handler.
  - **Shared-builder over-fetch.** If the diff added a default/field/eager-join to a request/criteria/query builder called by ≥2 consumers (Step 1A "Shared read-request builder"), grep every caller of that builder method yourself. For each caller, decide whether it consumes the added data. Any count-only / exists-only / autocomplete caller that now fetches unused data ⇒ raise NEW **Major** "over-fetch via shared builder", naming the wasteful call site — even if Pass D didn't run (trigger missed it). This is the exact failure the "symmetry-is-good" blind spot hides.
  - **Falsy default-injection.** If the diff added a `!$x` / `empty($x)` / `?:` / `??` guard that supplies a default, and `$x` is a value a caller can set to `0` / `'0'` / `''` / `false` with real meaning, confirm a NEW **Major** was raised for the silently-clobbered value (rule 8 Q1). If not, raise it.
  - **Rule-9 completeness — generic sweep over ALL rule-9 bullets, not a fixed shortlist.** This is the class of finding most often missed by spot-checking, so verify it mechanically: walk rule 9 bullet by bullet (9a … through the last letter — do **not** work from a hardcoded subset, the list grows) and for each, ask "does this diff contain the file shape this bullet triggers on?" If yes, confirm the owning pass rendered an explicit verdict for **every** matching site. Where a verdict is missing, run that bullet's own `Detect:` command yourself and raise the missing findings (Minor/Nit, but raise them all). Report the sweep as a one-line-per-bullet ledger — `9c: n/a (no Zed throw/catch in diff)` / `9f: 12 sites, 12 verdicts ✓` / `9m: 2 Service bodies, 0 verdicts ✗ → raised` — so a skipped bullet is visible rather than silently absent. A bullet whose trigger shape is absent from the diff is `n/a`, which is a *rendered verdict*; an unexamined bullet is a gap.
- Returns adjustments: severity changes, false-positive list, plus any **new issues** spotted while validating.

The validator's output is folded into the final synthesis. Disputed findings are kept but flagged.

## Step 5: Synthesize and persist

Consolidate findings into a single punch list grouped by the five **Step 1D** tiers (Critical / Blocker / Major / Minor / Nit — in that order), deduplicated across passes, each item with `file:line` and the violated rule. Include:
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
rule_sheet_revision: 14               # count of rule-9 bullets (9a..9n) in commands/review.md at run time
ticket: ABC-123                       # or null
commit: ef0d08c75
branch: abc-123-feature-branch
mode: full                            # full | light | hotspots-only
passes_run: [A, B, C1, C2, D, E, F, G, validator]
passes_degraded: []                   # passes that errored / returned empty / were cut short (Step 1C).
                                      # NOT the same as intentionally-untriggered passes, which go in `passes_skipped`.
passes_skipped: [B']                  # triggers didn't match — expected, not a degradation
verdict: block_merge                  # block_merge | proceed_with_caveats | clean
                                      # `clean` requires: 0 critical AND 0 blocker AND passes_degraded == []
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

`rule_sheet_revision` versions the **rules**, not the schema — set it to the number of rule-9 bullets present when the review ran (`grep -c '^   - \*\*9' commands/review.md`). It exists so a later comparison against this synthesis can tell whether a "miss" was a genuine gap or simply predates a rule that didn't exist yet. `/spryker-review:learn` reads it for exactly that purpose.

## Step 6: Exit criteria (review-complete check)

The review is "complete" when **all** of:
1. All Critical/Blocker items have an explicit decision: accepted with rationale, deferred to a follow-up ticket, or handed to the implementer to fix
2. All Major items are at minimum triaged (decision made, even if "deferred")
3. Validator pass has run and adjustments are applied — including its rule-9 completeness ledger (Step 4)
4. Synthesis file is saved
5. `passes_degraded` is empty, OR every degraded pass is named in the synthesis with its reason and the verdict is not `clean`

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
| Quick auth-only check on an ACL diff | `/spryker-review:review` then answer "hotspots-only" at Step 1, question 6 | Runs only Pass C1; ~2-3 agents |
| Big diff (>5,000 LOC) | Same as full, but Step 1, item 5 will warn — consider splitting per module first | Single review on huge diffs produces unfocused output |
| Pure refactor (no schema, no auth, no perf concern) | `/spryker-review --light` | Full mode would spin up D/E/F passes that have nothing to do |
