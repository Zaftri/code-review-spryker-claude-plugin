# MD-4743 — expected rule behaviour

**No baseline synthesis exists** for MD-4743 (`/spryker-review:review` was never run
on it). Every expectation below is therefore derived from the reviewer comments and
verified by hand against the repo at branch
`md-4743-mp-remove-feature-flags-on-released-ticked-run-slow` — not from a recorded
review run. `rule_sheet_revision` at the time the comments arrived: **14** (9a–9n).

Rules 9o, 9p, 9q and the rule-7 detection fix were added because of this MR; this
fixture is their regression case.

## MUST fire

| Rule | Site | Verified evidence |
|---|---|---|
| **9o(b)** | `src/Pyz/Yves/MerchantPostPage/Controller/MerchantPostController.php:41` | `resolveCanonicalSlugAndUuid` + `Response::HTTP_MOVED_PERMANENTLY` have **0 references under `tests/`** (grep confirmed: 3 hits, all in `src/`). Flag removal makes the behavior single-path and testable. Severity: Major. |
| **9o(a)** *(folded into broadened 9h)* | `MerchantProfileCest.php`, `MerchantProfileTabUrlCest.php` | `isNavReorganised` set from `count($i->grabMultiple(TAB_NOTIFICATIONS_BUTTON_SELECTOR)) === 0` (line 47), branched on at lines 130/214/280/300 — now constant. |
| **9o(a)** *(same)* | `DeliveryCalendarTabUrlCest.php:84,104,126` | `hasAssignedArticlesTab` from `grabAttributeFrom`, guards can never fire: `Details/index.twig:19` hardcodes `show-assigned-articles-tab="true"`. |
| **9p(a)** | `StickyFooterCest.php:59–63` (+83, 195, 307) | 8 sites where `seeElement(STICKY_FOOTER_*)` follows `waitForElement(STICKY_FOOTER)` on a *different, nested* selector. This is the finding that actually failed the pipeline. |
| **9p(a)** | `LanguageTabSwitcherCest.php:77,80` | 2 further sites, same shape. Both Cests lost a skip guard in this diff (2 removed guard lines each) ⇒ newly unconditional in CI. |
| **9p(b)** | `merchant-post-details-form.component.ts:139` | Component schedules `setTimeout`; sibling `merchant-post-details-form.component.spec.ts` has **0** `fakeAsync`/`tick(` and **0** `scrollIntoView`/`focus` assertions, while containing 4 `onFormTypeSelected` cases. |
| **9q** | `merchant-post-details.module.ts` | Diff adds `+import { NgModule, CUSTOM_ELEMENTS_SCHEMA }` and `+  schemas: [CUSTOM_ELEMENTS_SCHEMA]`. 2 matched `+` lines. |
| **rule 7 (fixed detection)** | `data/translation/Zed/DataImportMerchantPortalGui/*.csv` | The 6 `{uploaded,imported}.{label,from,to}` keys have **0 real-source refs** but appear in **4 compiled locale JSON bundles** — the bundles are what made them look used. Contrast `imported.table.title`: 1 real ref. |

## MUST NOT fire (false-positive guards)

| Rule | Site | Why silent |
|---|---|---|
| **9q** | every other module already using `CUSTOM_ELEMENTS_SCHEMA` | 5+ modules (`agent-login`, `product-badges-readonly`, `image-sets-readonly`, `login-header`, …) have it pre-existing. 9q fires only on **added** (`^+`) suppressions. Verified: 0 hits on `^-` lines. |
| **9p(a)** | the other 26 Cest files in the repo | The unscoped draft matched **87 sites across 28 files** — unusable. The rule is scoped to Cests **the diff touches** *and* that lost a skip guard; that yields 10 sites in 2 files. A repo-wide sweep here is a false positive, not thoroughness. |
| **9p(a)** | `waitForElement(X)` followed by `seeElement(X)` — same selector | Not a race; the awaited element is the asserted one. Only a *different, nested* selector counts. |
| **rule 7** | `imported.table.title`, `imported.table.sub_title` | Genuinely referenced in `DataImport/index.twig:47-48`. The fixed detection must still see real twig refs. |
| **rule 7** | `merchant_profile.tab.*` keys | Still used by the dedicated profile pages — reviewer verified and explicitly did not flag them. |
| **9o** | MD-2505 fixture | Contains no feature-flag removal: 0 removed-flag lines vs 50 here. |
| **9q** | MD-2505 fixture | 0 escape-hatch additions. |

## Verification run — 2026-07-28

Detection commands executed against `diff.patch` and the live worktree:

| Check | Result |
|---|---|
| 9q fires on MD-4743 | ✅ 2 added lines |
| 9q silent on MD-2505 | ✅ 0 |
| 9q silent on removed suppressions | ✅ 0 on `^-` |
| 9o fires on MD-4743 | ✅ 50 removed-flag lines |
| 9o silent on MD-2505 | ✅ 0 |
| 9p(a) — unscoped draft | ❌ **87 sites / 28 files — rejected as too broad** |
| 9p(a) — diff-scoped + nested-selector-only | ✅ 10 sites / 2 files, incl. the exact pipeline failure at 59–63 |
| 9p(b) fires on the one component with an unfaked timer | ✅ exactly 1 file |
| rule-7 fixed detection separates real refs from compiled bundles | ✅ 0 real vs 4 bundle for the 6 orphans; 1 real for `imported.table.title` |

Not mechanically verified: whether the sticky-footer race is specifically caused by
content projection rendering a tick later (the reviewer's hypothesis). The `waitFor`
conversion is correct regardless of the underlying cause.
