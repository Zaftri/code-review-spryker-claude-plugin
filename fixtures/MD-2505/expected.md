# MD-2505 — expected rule behaviour

Baseline synthesis when the human comments arrived: `rule_sheet_revision: 12` (9a–9l),
verdict `block_merge`, `critical: 0, blocker: 1, major: 6, minor: 4, nit: 2`.
Rules 9m and 9n were added *because of* this MR — this fixture is their regression case.

## MUST fire

| Rule | Site | Why |
|---|---|---|
| **9m** | `src/Pyz/Service/ProductDeposit/ProductDepositService.php` — `calculateItemDepositTotalFromMetadata()` | Body has `if` + `foreach` + `->fromArray()` instead of a single delegation. Sibling method in the same class shows the correct shape. Severity: Major. |
| **9n** | `src/Pyz/Zed/SalesMerchantPortalGui/Communication/Controller/DetailController.php` — `totalItemListAction()` | Per-item enrichment inside an override of a Core controller, where `SalesDependencyProvider::addOrderItemExpanderPlugins` publishes a stack and ~6 sibling `*OrderItemExpanderPlugin` classes already exist in `src/Pyz`. Severity: Major. |
| Rule 1 (Blocker) | `MerchantOrderItemGuiTableConfigurationProvider.php` — new consts | Untyped `const` violates CLAUDE.md "add types to constants if missing in edited files". |
| Rule 1 (Blocker) | whole diff | No test files touched; CLAUDE.md testing gate undischarged. |

## MUST NOT fire (false-positive guards)

| Rule | Site | Why silent |
|---|---|---|
| **9m** | `ProductDepositService::calculateItemDepositTotal()` (`:26`) | Already a correct single delegation — the rule must distinguish it from its sibling. A rule that flags both is matching on class name, not body shape. |
| Bridge/DIP violation | `->getLocator()->productDeposit()->service()` in the DependencyProvider | Canonical Pyz locator injection, explicitly sanctioned by rule 2. The baseline correctly did not flag this; a regression here means the Bridge nuance was lost. |
| Dead translation | the 4 new glossary keys | Present in all 4 locales and referenced; `included_fees` still used by the flag-OFF path. |
| **9n** | `MerchantOrderItemGuiTableDataProvider` deposit *column* config | Table-column configuration is not per-record transfer enrichment. 9n must not swallow ordinary data-provider work. |

## Fix-destination constraint (the SELF-INFLICTED case)

The baseline's M1 recommended *"move the metadata-fallback + calc into `ProductDepositService`
(e.g. `calculateItemDepositTotalFromMetadata(ItemTransfer)`)"* — the author implemented it
verbatim and reviewer comment #1 flagged the result.

Any future run on this diff MUST NOT propose that destination. Specifically:
- A fix that lands logic in a `*Service`/`*Facade`/`*Client` body ⇒ Step 3 self-check should
  reject it before it is written; Step 4 validator legality watch-list is the backstop.
- The duplication of `getItemDepositTotal()` across `DetailController` and
  `MerchantOrderItemGuiTableDataProvider` MUST NOT be resolved by extracting a shared helper
  both call (9n's corollary) — the duplication is evidence of wrong-layer placement, and the
  legal destination is a `*OrderItemExpanderPlugin` in the Sales stack.

## Round 2 — rev-14 baseline, reviewer comments 2026-07-29 / 2026-08-03

Second review of this MR (`rule_sheet_revision: 14`, verdict `proceed_with_caveats`,
`major: 2, minor: 5, nit: 3`). Rules broadened *because of* this round: 9a's Detect scope, rule 7's
schema-column sub-cases, rule 7's template-guard clause, and two validator checks.

### MUST fire

| Rule | Site | Why |
|---|---|---|
| **rule 7 — unreachable defensive guard (template clause)** | `Partials/order_items_list.twig` (`diff.patch:566`) | `(itemDepositTotals is defined ? itemDepositTotals[orderItem.idSalesOrderItem] : 0)` — `DetailController::totalItemListAction()` passes `itemDepositTotals` unconditionally (`[]` on the flag-OFF branch), so `is defined` is always true. The guard is dead **and** guards the wrong axis: the key access on the empty array is unguarded. Severity: Minor (dead guard) with the unguarded key access called out. Round 1 and the rev-14 baseline both missed this. |
| **9a (broadened Detect)** | `at-review-uncommitted.patch:43` — `@api` on `ProductDepositOrderItemExpanderPlugin::expand()` | A **concrete class**, not an interface. The pre-broadening Detect (`-- 'src/Pyz/**Interface.php'`) matched 0 lines here, which is precisely why the rev-14 baseline filed it Nit reasoning "not a strict 9a violation (9a is interface-scoped)". |
| **rule 7(ii) — write-only schema column** | `spy_merchant_sales_order.schema.xml` — `mercanto_fee_total`, `deposit_total` | Written via `MerchantSalesOrderMapper` and test-asserted, but no read path consumes them; the display reads `merchantOrder.order.totals.*` (order-wide `spy_sales_order_totals`) while these columns are merchant-scoped. Severity: **Major** — scope-correctness gap, not documentation. |
| **validator: deferred backfill** | same two columns | Both `default="0"`, no `config/post-deploy/*.yml` in the diff. Minor, as a prerequisite of the future reader. |

### MUST NOT happen (calibration guards)

| Anti-case | Why |
|---|---|
| **The write-only column finding must NOT be downgraded to Minor/doc-nit** | The rev-14 validator did exactly this — "a deliberate, tested parity mirror of the sibling `tax_total`/`refund_total`/`canceled_total` columns — not accidental dead schema". Human reviewer bitzeta raised it as blocking on 2026-08-03. A sibling column with no reader is precedent for the same gap, not a defence. This is the regression case for the "Deliberate parity mirror" watch-list entry. |
| **9a must NOT fire on `diff.patch`** | The committed diff contains 0 `@api` additions (the author removed the tag before committing). Firing there would mean the rule is matching file paths rather than added lines. |
| **The template-guard clause must NOT fire on `\| default(0)` alone** | The *fixed* form on the current branch is `itemDepositTotals[orderItem.idSalesOrderItem] \| default(0)` with no `is defined`. A rule that flags the fix is matching the variable, not the dead guard. |

### Verification run — 2026-08-04

| Check | Result |
|---|---|
| Template-guard Detect fires on `diff.patch` | ✅ 1 match, line 566 |
| Template-guard Detect silent on unrelated `MD-4743/diff.patch` | ✅ 0 |
| Broadened 9a Detect fires on `at-review-uncommitted.patch` | ✅ 1 match, line 43 (concrete class) |
| **Pre-broadening** 9a Detect on the same file | ✅ 0 — confirms the path filter was the defect, not agent error |
| Broadened 9a Detect silent on `MD-4743/diff.patch` | ✅ 0 |
| rule 7(ii) trigger (`added <column name=`) discriminates | ✅ MD-2505 = 2, MD-4743 = 0 |

Still judgment-dependent, not mechanically verified: the write-only **severity** decision and the
parity-mirror anti-downgrade both require agent reasoning over the ticket's stated data source; the
greps above only prove the trigger fires.

## Verification run — 2026-07-27

Detection commands executed against `diff.patch` (not merely asserted):

| Check | Result |
|---|---|
| 9m fires on `calculateItemDepositTotalFromMetadata()` | ✅ 3 lines matched (`if (`, `foreach (`, `->fromArray(`) |
| 9m stays silent on sibling `calculateItemDepositTotal()` | ✅ 0 branching lines — the rule discriminates on body shape, not class name |
| 9n fires on `DetailController::totalItemListAction()` | ✅ matched `foreach ($expandedItems as $itemTransfer)` |
| 9n stays silent on the table-column config change | ✅ no `foreach` added in the ConfigurationProvider |

Not yet exercised: the MUST-NOT-fire guards for Bridge/DIP and dead-translation depend on
full agent judgment rather than a grep, so they remain unverified by this mechanical pass.
