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
