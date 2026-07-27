# MD-2505 — human reviewer comments

Source: https://gitlab.com/mercanto/laks/-/merge_requests/17068
Reviewer: anton.smarovydlo (MR author: wasiewiczc)
Fetched: 2026-07-27

## 1. `src/Pyz/Service/ProductDeposit/ProductDepositService.php:36`

> Services should be thin, please move it to DepositCalculator

Unresolved. Lands on `calculateItemDepositTotalFromMetadata()`, whose body holds a
conditional + `foreach` + inline `->fromArray()` mapping. The sibling method at `:26`
is a correct one-line delegation to `createProductDepositCalculator()`.

## 2. `src/Pyz/Zed/SalesMerchantPortalGui/Communication/Controller/DetailController.php:68`

> Maybe it worth implementing plugin for `OrderItemExpanderPluginInterface` plugin
> stack, instead of doing this in the controller?

Unresolved. Lands on a `foreach` over `$expandedItems` inside an override of Core's
`DetailController::totalItemListAction()`, computing a per-item deposit total.
