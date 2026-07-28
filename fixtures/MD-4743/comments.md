# MD-4743 — reviewer comments

Source: https://gitlab.com/mercanto/laks/-/merge_requests/17075
Reviewer: bitzeta (Serhii Doroshenkov) — real account, but the comment *format*
(severity tags, AC table, checklist) indicates AI-assisted review output.
MR author: wasiewiczc. Fetched: 2026-07-28. All 6 findings unresolved at fetch time.

Scope of the MR: removes 7 released feature flags, keeping the flag-ON behavior
unconditionally and deleting all flag-OFF paths, dead controllers/forms/templates,
and flag-conditional test guards.

## 1. `tests/PyzTest/.../StickyFooterCest.php:36` — [MEDIUM/CI]

Pipeline failure on `seeElement(STICKY_FOOTER_ADD_FIELD_MARKER)` while the failure
artifact's page-source contains that element ⇒ the assertion raced the footer's
children mounting. Lines 59–63 use instant `seeElement` for footer children right
after `waitForElement(STICKY_FOOTER)`; the host appears before its projected
content renders. Convert child assertions to `waitForElement`.

## 2. `src/Pyz/Yves/MerchantPostPage/Controller/MerchantPostController.php:41` — [MEDIUM/TestCoverage]

The canonical-slug 301 redirect is now permanent behavior with no test coverage.
Deferring was justified while the flag existed (a test would have had to handle
both flag states, and flag state can't be controlled per-test). That blocker is
gone: one behavior now, so the functional test is straightforward.

## 3. `.../merchant-post-details/merchant-post-details.module.ts:26` — [MEDIUM/Quality]

`CUSTOM_ELEMENTS_SCHEMA` added with no visible need — the base module compiled
without it and this MR adds no new elements. It disables unknown-element compile
checks for every template in the module: a typo'd selector or missing import stops
being a build error and renders an empty tag at runtime.

## 4. three Cests (no file anchor) — [MEDIUM/TestCoverage]

Dead flag dual-path scaffolding left in tests:
- `MerchantProfileCest.php` / `MerchantProfileTabUrlCest.php` branch on
  `isNavReorganised`, DOM-sniffed via `grabMultiple(TAB_NOTIFICATIONS_BUTTON_SELECTOR)`,
  now permanently true — else-paths are dead.
- `DeliveryCalendarTabUrlCest.php` keeps `hasAssignedArticlesTab` skip guards
  (lines 85/105/127) that can never trigger since `show-assigned-articles-tab="true"`
  is hardcoded in `Details/index.twig`.

## 5. `.../merchant-post-details-form.component.ts:131` — [LOW/TestCoverage] (optional)

`scrollAndFocusNewElement` is now default behavior but the spec exercising
`onFormTypeSelected` never asserts it, and doesn't fake timers — the `setTimeout`
fires uncontrolled after the test finishes.

## 6. translation CSVs (no file anchor) — [LOW/Cleanup] (optional)

The removed `uploaded`/`imported` date-range filters leave 6 orphaned glossary keys
per locale: `data_import_merchant_portal_gui.{uploaded,imported}.{label,from,to}`.
Their only consumer was the removed `addFilterDateRange('uploaded'/'imported')` calls.
`imported.table.title` remains in use.
