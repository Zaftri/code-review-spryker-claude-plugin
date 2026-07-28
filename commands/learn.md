---
description: Diagnose why /spryker-review:review missed a human reviewer's MR comment, and harden the plugin's rule sheet with a generalized rule — never a one-off special case.
---

# Spryker Review — Learn from Human Feedback

## Input
$ARGUMENTS

Parse as `<MR-ref> [<synthesis-file-override>]` where `<MR-ref>` is a GitLab MR URL, `!1234`, `MR 1234`, or a ticket key whose open/merged MR will be looked up via `glab mr list --search`. `<synthesis-file-override>` optionally points at a specific `reviews/*.md` file if the ticket-based naming convention (`reviews/<TICKET>-spryker-review.md`) doesn't match this MR.

This command improves the **spryker-review plugin itself** — it does not review or touch the project's source code. It:
1. Reads a reviewed MR's human reviewer comments.
2. For each one, diagnoses whether `/spryker-review:review` already caught it, and if not, *why not*.
3. Proposes a **generalized** rule-sheet edit — the underlying pattern the comment exemplifies, not a special case keyed to this MR's specific identifiers — and applies it (with approval) to this plugin's own files.

## Step 1: Resolve the MR and locate this plugin's own files

1. Resolve `<MR-ref>` to a project path + MR IID:
   - URL → parse `https://gitlab.com/<path>/-/merge_requests/<iid>`
   - `!1234` / `MR 1234` → IID `1234`; project path from `git remote get-url origin`
   - Ticket key → `glab mr list --search "<TICKET>"`; if more than one match, ask the user which.
2. Locate this plugin's `commands/review.md`. **There should be exactly one real copy.** The plugin is registered as a local `directory`-source marketplace whose plugin source is `"./"`, so Claude Code loads it from the marketplace's `installLocation` — the source repo itself. The repo you edit *is* the live plugin. Resolve it and check the invariant:
   ```bash
   # the single source of truth
   MP=$(python3 -c "import json;print(json.load(open('$HOME/.claude/plugins/known_marketplaces.json'))['code-review-spryker-claude-plugin']['installLocation'])")
   echo "plugin root: $MP"
   # THE authoritative check — must print exactly one path
   find ~ -maxdepth 8 -path "*/commands/review.md" -path "*spryker*" 2>/dev/null | xargs -I{} realpath {} | sort -u
   ```
   **If it prints more than one, STOP** — the single-copy setup has regressed into independent clones (most likely a `claude plugin install` cycle materialised a real cache directory). Tell the user, show the differences, and ask which is authoritative before continuing. Do not silently pick one.

   The `installed_plugins.json` `installPath` still names a path under `~/.claude/plugins/cache/…`, but for a `directory`-source marketplace that entry is **vestigial**: it may be absent, or a symlink, or get removed by the periodic in-use sweep, and **none of those is a regression** — the plugin keeps loading from `installLocation`. Verify with `claude plugin details spryker-review` (expect 3 skills: `review`, `learn`, `spryker-conventions`) rather than by inspecting the cache path. Do not "repair" a missing cache directory.
3. The sibling `skills/spryker-conventions/SKILL.md`, `CHANGELOG.md`, and `fixtures/` all live under that same plugin root. Because there is one copy, Step 7 has nothing to sync — but it still verifies the single-copy invariant afterwards.

## Step 2: Fetch human reviewer feedback

```bash
PROJECT_ENC=$(git remote get-url origin | sed -E 's#.*[:/]([^/]+/[^/]+)\.git#\1#' | sed 's#/#%2F#')
glab api "projects/${PROJECT_ENC}/merge_requests/<iid>/discussions" --paginate
```

Keep only notes where `system == false`. For each kept note capture: author, `position.new_path` + `position.new_line` (fall back to `old_path`/`old_line` if the new side is empty — a comment on a deleted line), body, `resolvable`/`resolved`, and whether the note's author is the MR author (mark as **author reply** — context only, never itself a miss candidate) vs. an actual reviewer.

If there are zero non-system, non-author-reply notes: report "no reviewer feedback on this MR — nothing to learn from" and stop here.

**AI-assisted reviewer comments are usable, but only after independent verification.** Comments posted from a human account may still be the output of a review tool — the tells are machine-formatted severity tags (`**[MEDIUM/CI]**`), an acceptance-criteria table, a findings table, or a checklist. Such a comment is legitimate signal (the account owner chose to post it), but it is **not** an independent human judgement, and if the tool that produced it is *this plugin*, hardening rules against it is a feedback loop that teaches nothing.

So: when the format indicates AI authorship, say so explicitly in the Step 4 table, and **verify every factual claim against the repo yourself before classifying it as MISSED** — run the greps, open the cited lines, confirm the counts. Do not take "zero hits for X under tests/" or "this key is orphaned" on trust; both are cheap to check and both have been wrong. A claim that fails verification is not a rule gap, and must be reported as a *rejected* comment rather than quietly dropped.

## Step 3: Load the baseline (if one exists)

Determine the ticket key (from the branch name or MR title, same convention `/spryker-review:review` uses) and look for `reviews/<TICKET>-spryker-review.md` in the project repo (or the override path if given). If found, read its YAML frontmatter (`passes_run`, `passes_degraded`, `rule_sheet_revision`, `verdict`, counts) and its findings tables. If not found, proceed without a baseline and say so explicitly — every reviewer comment below is **untested**, not a **confirmed miss**; don't overclaim.

The synthesis may not be in the project's own `reviews/` directory — a review run inside a per-ticket worktree (e.g. `~/.menv/trees/<ticket>/reviews/`) writes it there. If the conventional path misses, search before concluding there's no baseline:
```bash
find ~ -maxdepth 8 -path "*/reviews/<TICKET>-spryker-review.md" 2>/dev/null
```

**Two baseline-validity checks, both mandatory before you classify anything:**

1. **Rule-sheet drift.** Compare the baseline's `rule_sheet_revision` against the current sheet (`grep -c '^   - \*\*9' commands/review.md`). If the current count is higher, the baseline ran under an older rule set, so a "MISSED" may simply predate the rule that now covers it. Before classifying any comment as MISSED, check whether a rule added *since* that revision already covers it — if so, classify as **ALREADY-FIXED** (no edit needed; say so and move on). If the baseline has no `rule_sheet_revision` field at all, it predates the field: state that the comparison is unversioned and treat every MISSED as provisional.
2. **Degraded passes.** If the baseline lists a non-empty `passes_degraded`, a comment owned by a degraded pass is **NOT a rule gap** — the rule may well exist and simply never ran. Classify those as **PASS-DEGRADED** and do not add a rule for them; the fix is re-running the review, not hardening the sheet. Adding a rule here is how the sheet accretes redundant bullets.

## Step 4: Classify every reviewer comment

For each reviewer comment (author-replies are context only, skip classifying them directly), decide one of:

- **MATCHED** — the baseline synthesis has a finding at the same/adjacent file:line whose summary covers the same issue. Not a miss.
  - Sub-case: if that finding was marked `DISPUTED` by the validator but the human comment confirms it's real, flag separately as a **VALIDATOR-GAP** — the rule exists and fired; the validator wrongly waved it off. This needs a Step-4/validator-prompt fix, not a new rule.
- **SELF-INFLICTED** — the baseline didn't just miss the issue, it **recommended** the code the human flagged. Check this before concluding MISSED: search the baseline's *fix* text (not only its findings) for the file, class, or method the comment lands on. A review that prescribes a violation is a worse failure than one that stays silent, and it needs a different remedy — a rule that constrains **fix destinations** (Step 3 self-check / Step 4 validator legality watch-list), not only one that detects the pattern in code. Say so explicitly in the table; do not soften it to MISSED.
- **ALREADY-FIXED** — a rule added after the baseline's `rule_sheet_revision` already covers this (see Step 3). No edit needed.
- **PASS-DEGRADED** — the owning pass is listed in the baseline's `passes_degraded` (see Step 3). Not a rule gap. No edit.
- **MISSED** — no baseline, or no matching finding, and none of the above apply. Root-cause it by reading the CURRENT `commands/review.md` (from Step 1, item 2):
  - Which Step-2 rule (1–9) or Step-3 pass conceptually *owns* this class of issue?
  - Did that pass even run? Check the **Step 1A** hotspot trigger table against the changed file's path — if the glob didn't match, the trigger is too narrow.
  - If the pass ran: was the pattern outside its stated scope, or was it in scope but cut by the Step-3 "signal budget" cap (check the synthesis for a "N additional ... omitted for signal" note, or a rule-9 aggregate-overflow line)?
  - Or is this genuinely new territory no existing rule addresses?

Present as a table: `comment (file:line, author, resolved?) | MATCHED / SELF-INFLICTED / ALREADY-FIXED / PASS-DEGRADED / VALIDATOR-GAP / MISSED | root cause`.

## Step 5: Draft a GENERALIZED rule for every MISSED / VALIDATOR-GAP item

The point of this command is this step — do not skip or shortcut it.

1. **Abstract the pattern.** State the underlying shape of the problem the comment exemplifies — not the literal class/method/file names from this MR. Self-test: *would this rule fire on a differently-named class doing the same shape of thing, in a different module?* If you can only picture it firing on this MR's exact identifiers, generalize further before writing anything down. (Mirror the existing rule-8 style — e.g. "Default-injection guards" states a pattern class with one illustrative example in parentheses, never a rule about one specific field.)
2. **Check for a near-duplicate first.** `grep -in '<keyword>' commands/review.md`. If a related rule already exists but is worded too narrowly to have caught this, propose *broadening the existing bullet* instead of adding a redundant new one.
3. **Pick the destination**, in this priority order:
   - A **new rule-9 idiom bullet** if the pattern is deterministic and grep-detectable — follow the existing `9a`/`9b`/... lettering and the "(Pass X)" ownership tag. **Every new 9x bullet must name the general rule it specializes** (e.g. "specializes rule 6 SoC", "specializes rule 3 OCP"), so the parent's now-redundant vague wording can later be pruned or pointed at its children. A 9x bullet with no identifiable parent is fine — say "new territory" explicitly.
   - A **tightened Step 1A hotspot trigger row** if the real problem is that the relevant pass never fired on this file shape.
   - A **tightened/expanded Step-2 rule 1–8** wording if an existing judgment-based rule was just too narrow.
   - A **Step-3 pass-trigger table adjustment** if the wrong pass mix ran for this hotspot.
   - A **validator Step-4 watch-list addition** if this was a validator-calibration gap, not a missing rule.
   - A **`skills/spryker-conventions/SKILL.md` addition** instead of `review.md`, if the underlying fact is a Spryker/architecture convention rather than a review-process rule.
4. Produce the actual proposed edit as an old_string/new_string diff against the exact file and location chosen.

## Step 5.5: Consolidation check (run whenever the rule-9 count would cross a multiple of 5)

Rule 9 grows one to two bullets per invocation of this command and nothing ever shrinks it. Left unchecked it outgrows the agent prompt that must carry it verbatim (Step 3 of `review.md` requires the rule-9 checks be passed to Pass A/B/E **in full**, not summarized). So: after drafting this run's edits, compute the resulting bullet count. If it crosses a multiple of 5 (15, 20, 25…), do a consolidation pass **in the same run** and include it in the Step 6 approval request:

1. Group every 9x bullet by its owning pass (the `(Pass X)` tag).
2. Within each group, look for pairs whose *detection* overlaps — same file shape, same grep, or one is a strict subset of the other. Propose merging them into one bullet with two named sub-cases, preserving both `Detect:` commands. Merging must never drop a detectable case; if a merge would, don't merge.
3. Check each bullet's stated parent rule (Step 5.3). Where several 9x bullets specialize the same rule 1–8, propose replacing that parent's vague wording with a pointer to its children ("see 9m, 9n for the deterministic cases") so agents aren't given two overlapping instructions for one pattern.
4. Propose retiring any bullet that has become unreachable — e.g. its pattern is now caught by PHPStan/Psalm in CI, or the codebase convention it guarded no longer exists. Retire by deletion with a CHANGELOG line, not by leaving it in place "just in case".

Report the count before and after. If no merge is safe, say so — an honest "14 → 16, no safe consolidation found" is a valid outcome; a forced merge that loses a detection case is not.

## Step 5.6: Regression check against the fixture corpus

A new or broadened rule must be checked for **both** false negatives and false positives before it lands. The corpus lives at `fixtures/` in the plugin root: one directory per past MR, each containing `diff.patch`, `comments.md` (the human reviewer notes), and `expected.md` (which rules should fire, and notably which should stay silent).

- **If `fixtures/` exists:** pick the 2–3 fixtures whose file shapes most resemble this rule's trigger, plus **one deliberately unrelated fixture**. For each, run the new rule's `Detect:` command against `diff.patch` and confirm it fires where `expected.md` says it should and stays silent on the unrelated one. A rule that fires on the unrelated fixture is too broad — narrow it and re-check before proposing.
- **If `fixtures/` does not exist:** say so plainly and state that the proposed rules are **unvalidated** — the same "untested, not confirmed" honesty Step 3 requires about a missing baseline. Then seed the corpus with *this* MR as its first entry (`fixtures/<TICKET>/`) so the next run has something to check against. Do not skip this silently; an unvalidated rule addition is the plugin's own equivalent of an assertion-free test, which `review.md` rates a Blocker.

## Step 6: Present and confirm

Show one consolidated table (comment → classification → root cause → proposed destination), then each proposed diff. Also report, in one line each: the rule-9 count before → after, the Step 5.5 consolidation outcome (or why it didn't trigger), and the Step 5.6 regression-check result (or that the corpus is absent and the rules are unvalidated). Ask for approval — accept all, accept some, or reject/redirect any individual one. **Do not edit anything until approved.**

## Step 7: Apply, sync, and log

On approval:
1. Apply each edit **once**, to the single plugin root resolved in Step 1, item 2. Then re-run that step's third command to confirm it still resolves to exactly one path — an edit that somehow produced a second copy means the symlink was replaced by a directory (e.g. by a `claude plugin install` cycle), which silently reintroduces drift. Report it if so.

   Changes take effect **on the next session**, since skills are loaded at startup. Do NOT run `claude plugin update` or an `uninstall`/`install` cycle to "apply" them — `update` no-ops on an unchanged version, and a reinstall materialises a real cache directory, which is exactly how the single-copy setup regresses into clones. Just restart. If a stray cache copy does appear, delete it (`rm -rf "$HOME/.claude/plugins/cache/code-review-spryker-claude-plugin"`); the plugin loads from the marketplace `installLocation` and does not need it.
2. Append one line per applied rule to the plugin-root `CHANGELOG.md` (create it if absent):
   ```
   - YYYY-MM-DD — MR !<iid> (<ticket>): <one-line generalized rule summary> → <file touched>
   ```
   Use a date the user confirms — never invent "today" from stale context.
3. Do **not** `git commit`, `git push`, bump `plugin.json`/`marketplace.json` version, or touch the project being reviewed. If the changes are substantive, tell the user they may want to commit/push and/or bump the plugin version themselves.

## Example invocations

| Scenario | Invocation |
|---|---|
| Reviewer left comments on an MR you already ran `/spryker-review:review` on | `/spryker-review:learn 16818` |
| Ticket's MR comments, no prior synthesis on disk | `/spryker-review:learn MD-3091` |
| Synthesis file uses a non-standard name | `/spryker-review:learn 16818 reviews/custom-name.md` |

## Boundaries

**This command WILL:**
- Read the target MR's reviewer comments (read-only, via `glab`)
- Read the project's existing `reviews/*.md` synthesis if present
- Read, and with explicit approval, edit this plugin's own `commands/review.md`, `skills/spryker-conventions/SKILL.md`, and `CHANGELOG.md`
- Create and extend the plugin's own `fixtures/` regression corpus (Step 5.6)
- Propose merging or retiring existing rule-9 bullets when the count crosses a multiple of 5 (Step 5.5) — retirement still needs explicit approval like any other edit

**This command WILL NOT:**
- Edit any file in the project being reviewed
- Commit, push, or comment on the MR
- Apply a rule change without showing the diff and getting approval first
- Add a rule keyed to this MR's specific identifiers instead of the general pattern it exemplifies
