# Changelog

Rule-sheet edits applied via `/spryker-review:learn`, tracing each addition back to the human MR feedback that motivated it.

- 2026-07-21 — MR !16994 (MD-4646): domain-logic traits with implicit `@method` dependency contracts are a convention violation (rule-9g + SKILL.md "Traits" section) → `commands/review.md`, `skills/spryker-conventions/SKILL.md`
- 2026-07-21 — MR !16994 (MD-4646): Q1 invariant enumeration now includes shared-state (BehaviorSubject/Observable) reset symmetry across event handlers → `commands/review.md`
- 2026-07-21 — MR !16994 (MD-4646): skip-guards must query the controlling state directly, not infer it from an observed side effect (rule-9h) → `commands/review.md`
- 2026-07-21 — MR !16994 (MD-4646): a spec/Cest rewrite must not silently drop a covered scenario without a replacement (rule-9i) → `commands/review.md`
- 2026-07-21 — MR !16994 (MD-4646): Pass E test-coverage checks now enumerate structural (not just CRUD-state) variation across rollout consumers → `commands/review.md`
- 2026-07-22 — MR !17038 (MD-3686): a Persistence Factory directly `use`-ing another module's Propel Query class bypasses the DependencyProvider (rule-9j) → `commands/review.md`
- 2026-07-22 — MR !17038 (MD-3686): a test file's declared namespace must mirror the actual layer it tests, and a missing suite (e.g. no Persistence suite) is the root cause when tests get misfiled (rule-9k) → `commands/review.md`
- 2026-07-22 — MR !17038 (MD-3686): an event-triggered republish/listener plugin needs a modified-columns guard when the read model only depends on a subset of columns (rule-9l) → `commands/review.md`
- 2026-07-22 — MR !17038 (MD-3686): validator's "Runtime parity" check now checks BOTH flag-toggle directions when the reconciliation trigger is gated by the same flag it exists to reconcile → `commands/review.md`
- 2026-07-22 — MR !17038 (MD-3686): Pass D's missing-indexes check now covers non-leading columns of an existing composite index/unique constraint, even without a schema-file diff → `commands/review.md`
- 2026-07-22 — MR !17038 (MD-3686): a test that mocks the exact collaborator whose behavior is under test only proves wiring, not behavior (Pass E) → `commands/review.md`
- 2026-07-27 — MR !17068 (MD-2505): `*Facade`/`*Service`/`*Client` method bodies must be thin one-line delegations to a factory-built model; branching/looping/mapping there belongs behind the delegation (rule-9m) → `commands/review.md`, `skills/spryker-conventions/SKILL.md`
- 2026-07-27 — MR !17068 (MD-2505): per-record enrichment inside a Pyz override of a Core class must first check the owning module's published `add*ExpanderPlugins` stack; duplication across ≥2 overrides is wrong-layer evidence, not a DRY problem for a shared helper (rule-9n) → `commands/review.md`, `skills/spryker-conventions/SKILL.md`
- 2026-07-27 — MR !17068 (MD-2505): every suggested fix must be self-checked against rules 1–9 before being written down — a fix a reviewer would themselves comment on launders a finding into a new violation (Step 3 agent prompt requirements) → `commands/review.md`
- 2026-07-27 — MR !17068 (MD-2505): validator now re-reads the *fix* text of each finding, not only its diagnosis, and rewrites fixes whose destination violates the rule sheet (Step 4 watch list) → `commands/review.md`
- 2026-07-27 — spec review (/sc:spec-panel): canonical five-tier severity table added as Step 1D; "Blocker" was previously used in 4 places but never defined, and Step 3 defined only 4 tiers → `commands/review.md`
- 2026-07-27 — spec review: Step 1.4/1.5/1.6 headings relettered to Step 1A/1B/1C (out-of-scope now precedes degradation); 2 cross-references pointed at the wrong section because heading numbers collided with Step 1's list items → `commands/review.md`
- 2026-07-27 — spec review: rule-9 exempt "Convention" block capped at 40 items with per-rule aggregate overflow — the cap exemption was previously unbounded, reopening the signal problem it was an exception to → `commands/review.md`
- 2026-07-27 — spec review: agent-failure row added to Step 1C + `passes_degraded`/`passes_skipped` frontmatter; a failed pass was previously indistinguishable from a pass that found nothing (the same false-negative shape rule 9h flags) → `commands/review.md`
- 2026-07-27 — spec review: validator rule-9 completeness generalized from a hardcoded 3-of-14 shortlist to a ledger over every 9x bullet → `commands/review.md`
- 2026-07-27 — spec review: light mode now runs a test-presence check in Pass B — it skips Pass E and so could not detect the CLAUDE.md testing gate, a whole Blocker class → `commands/review.md`
- 2026-07-27 — spec review: frontend-heavy trigger threshold made computable (`git diff --numstat` sum >= 200) instead of "~200 LOC" → `commands/review.md`
- 2026-07-27 — spec review: `rule_sheet_revision` added to synthesis frontmatter; /learn compares it to detect that a "miss" predates the rule now covering it → `commands/review.md`, `commands/learn.md`
- 2026-07-27 — spec review: /learn gains SELF-INFLICTED, ALREADY-FIXED and PASS-DEGRADED classifications — a review that *prescribes* a violation needs a fix-destination rule, not a detection rule → `commands/learn.md`
- 2026-07-27 — spec review: /learn Step 5.5 consolidation pass every 5th rule-9 bullet (merge overlapping, retire unreachable, point vague parents at their children); each new 9x must name the rule it specializes → `commands/learn.md`
- 2026-07-27 — spec review: /learn Step 5.6 regression check against a new `fixtures/` corpus; an unvalidated rule addition is the plugin's own assertion-free test → `commands/learn.md`, `fixtures/`
- 2026-07-27 — spec review: /learn now searches beyond the project `reviews/` dir for a baseline (worktree runs write to `~/.menv/trees/<ticket>/reviews/`) → `commands/learn.md`
- 2026-07-27 — spec review: corpus seeded with its first entry, `fixtures/MD-2505/` (diff + reviewer comments + MUST-fire/MUST-NOT-fire expectations for 9m/9n) → `fixtures/MD-2505/`
- 2026-07-27 — infra: collapsed the three divergent on-disk copies to one. Marketplace plugin source changed from `github` to `"./"` (self-referencing local path), repo re-registered as a `directory`-source marketplace, and the plugin-cache path symlinked back to the repo — the repo is now the live plugin → `.claude-plugin/marketplace.json`, `README.md`
- 2026-07-27 — infra: /learn Step 1.2 replaced multi-copy discovery ("find every copy, STOP if they differ") with a single-copy invariant check that resolves symlinks; Step 7 no longer syncs and instead re-asserts the invariant after editing → `commands/learn.md`
