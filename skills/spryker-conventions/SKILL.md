---
name: spryker-conventions
description: Canonical Spryker architectural conventions for code review and implementation: layer rules (Zed/Yves/Client/Shared/Service), Bridge class-vs-interface distinction at Pyz level, factory createX/getX visibility, dependency provider conventions, module public API rules. Activate when reviewing or writing Spryker code, especially when judging cross-module dependencies, Bridge usage, factory shape, or interface placement.
---

# Spryker conventions reference

This skill encodes the architectural rules used by `/spryker-review:review` and applies whenever you're reviewing or writing Spryker code.

## Canonical sources

When in doubt, query `/spryker/spryker-docs` via `mcp__context7__query-docs` rather than relying on training data. URLs:
- Architecture landing: https://docs.spryker.com/docs/dg/dev/architecture/architecture.html
- Modules and layers: https://docs.spryker.com/docs/dg/dev/architecture/modules-and-application-layers.html
- Architectural convention: https://docs.spryker.com/docs/dg/dev/architecture/architectural-convention.html
- Module API (public vs private): https://docs.spryker.com/docs/dg/dev/architecture/module-api/declaration-of-module-apis-public-and-private.html
- Dependency Injection: https://docs.spryker.com/docs/dg/dev/architecture/dependency-injection/implementation-and-usage.html
- Extending core used by another module: https://docs.spryker.com/docs/dg/dev/backend-development/extend-spryker/spryker-os-module-customisation/extend-a-core-module-that-is-used-by-another-module.html

## Layers

| Layer | Purpose | What lives here |
|---|---|---|
| **Zed** | Backend / business / persistence | Facade, Business/, Persistence/, Communication/ (controllers, plugins, forms, tables, console) |
| **Yves** | Storefront | Theme, Controllers, Twig views, Client wrappers |
| **Client** | API to remote services (Zed/storage/search) from Yves | Read-only surface; no business logic |
| **Shared** | Cross-process constants, transfer XML | Transfers, constants only |
| **Service** | Pure stateless services | Calculation/format helpers |

**Layer purity rules:**
- Business/* must NOT call Persistence directly except via repository/entity-manager.
- Communication must NOT contain business logic; delegate to Facade.
- Persistence calls (Propel `SpyXQuery::create()`) must NOT appear outside `Persistence/`.
- Presentation (HTML/twig) must NOT be inlined in PHP business code.
- Cross-module access goes through the Facade interface only — never via direct import of another module's internals.

## Bridges at Pyz — class vs interface

This is a frequently-confused rule. Authoritative distinction (verified against Spryker docs):

| Element | Spryker Core | Pyz / project |
|---|---|---|
| Bridge **class** (`XToYBridge.php` doing 1:1 method delegation) | Yes — wraps facade in DP closure | **No** — replace with `$container->getLocator()->y()->facade()` |
| Bridge **interface** (`Pyz/Zed/X/Dependency/Facade/XToYInterface`) | Yes | **Yes — recommended as factory return-type hint** |
| Locator-based facade injection in DP | n/a | Canonical Pyz pattern |

The "Core only!" annotation in Spryker's dependency-provider docs refers specifically to wrapping the facade in a Bridge class inside the DP closure — NOT to bridge interfaces. **Do NOT flag direct foreign-facade imports between Pyz modules as DIP/Blocker violations.**

### Canonical Pyz 3-step pattern

When module X needs to depend on module Y's facade — especially when extending vendor core or wanting decoupling:

**1. Pyz interface in `Pyz/Zed/X/Dependency/Facade/`** that extends the foreign facade interface:
```php
namespace Pyz\Zed\Cart\Dependency\Facade;

use Spryker\Zed\Cart\Dependency\Facade\CartToCalculationInterface as SprykerCartToCalculationInterface;

interface CartToCalculationInterface extends SprykerCartToCalculationInterface
{
    public function foo();
}
```

**2. DependencyProvider injects facade directly via locator (no Bridge class):**
```php
$container[self::FACADE_CALCULATION] = function (Container $container) {
    return $container->getLocator()->calculation()->facade();
};
```

**3. Factory accessor types the return as the Pyz interface:**
```php
public function getCalculationFacade(): CartToCalculationInterface
{
    return $this->getProvidedDependency(CartDependencyProvider::FACADE_CALCULATION);
}
```

### When you need this 3-step vs. when direct import is fine

- **3-step**: extending a core module's facade and another module must call new methods; OR you want a stable internal contract isolated from vendor changes; OR you want a narrower surface.
- **Direct `use Pyz\Zed\Y\Business\YFacadeInterface` in DP closure is fine** for plain project-to-project dependencies. Spryker docs sanction it.

## Factory conventions

- `createX()` — returns a new instance every call. Typically `protected` or `private` (only the factory's `getX()` and the facade should call it).
- `getX()` — returns a cached/shared dependency from `$this->getProvidedDependency()`. Typically `public` only when the facade or its caller needs it; otherwise `protected`.
- The facade uses `$this->getFactory()->createX()->doWork()` for business logic, not `getX()` for build-block components.

## Entry-point classes are thin delegators

`*Facade`, `*Service`, and `*Client` are the module's *published surface*, not a place to implement anything. Each method body is one delegation to a factory-built model:

```php
public function calculateItemDepositTotal(ItemTransfer $itemTransfer): int
{
    return $this->getFactory()->createProductDepositCalculator()->calculateItemDepositTotal($itemTransfer);
}
```

A conditional, a `foreach`, an inline `->fromArray()` mapping, or multi-step orchestration in that body is logic that belongs in the model behind the delegation (`*Calculator`, `*Reader`, `*Writer`, `*Expander`, `*Mapper`). "Services should be thin" is a recurring reviewer comment. The tell is usually visible inside the same file — sibling methods on the class show the correct one-line shape, so a body with branching stands out immediately. Needing a new entry point is a reason to add a *delegating* method, never a reason to host the logic on the entry-point class; if the model doesn't exist yet, add it and a `createX()` accessor.

Note this interacts with the review's own fix suggestions: "extract the duplicated logic into the Service" is an illegal destination for anything but a delegation.

## Least visibility — verify, don't assume (recurring reviewer comment)

Reviewers reliably flag methods declared wider than their actual usage ("make it private if not used anywhere else"). The check is mechanical and must be **exhaustive**, not spot-checked: for every method / factory accessor added or changed, grep callers across `src/` + `tests/` (`grep -rn '->methodName\|::methodName'`).

- No caller outside the declaring class ⇒ `private`.
- Callers only within the same module ⇒ `protected`.
- `public` only when an external caller (another module, the facade's public surface, a controller via `getFactory()`) genuinely exists.

A method called only by `getFactory()->createX()` from within the same module's controller still needs only `protected` (a factory accessor resolves on protected).

## `@api` annotation is a Core marker — not for Pyz

The `@api` docblock tag designates a Spryker **Core** module's *published* public API (the contract Core promises downstream projects). **Pyz application code does not publish a module API**, so `@api` tags do not belong on `src/Pyz/**` interfaces or classes. Adding `@api` to a Pyz `*Interface.php` is a convention violation — recommend removal (and check sibling interfaces in the same MR: the same paste often spreads it to several files).

## Mapping logic belongs in a Mapper

Transfer→transfer, array→transfer, and entity→transfer mapping is its own responsibility and belongs in a dedicated `*Mapper` class (`map<Source>To<Target>(...)`), not inlined in an Expander / Resolver / Controller / Hydrator / Reader. When you see ≥~3 lines of field-copying, or two or more `map*`/`extract*` private methods accreting on a non-mapper class, the fix is "extract to a `*Mapper`" — name the Spryker **Mapper** pattern specifically (not a generic "builder"/"helper"/"view object").

## Prefer a published extension point over enriching inside an overridden Core class

Core modules publish extension points precisely so projects don't have to override their classes. When a Pyz override of a Core controller / data provider / table-configuration provider / hydrator adds per-record enrichment to transfers it just fetched from a facade, the owning Core module usually already has a plugin stack for exactly that (`add*ExpanderPlugins` / `get*ExpanderPlugins` in its DependencyProvider, backed by a `*ExpanderPluginInterface` in the matching `*Extension` module). Registering a `*ExpanderPlugin` puts the enrichment on **every** consumer of that facade method at once, and typically removes the need for the Pyz override entirely.

How to check, in order:
1. Identify the module that owns the facade method the override calls.
2. `grep -nE 'add[A-Za-z]*(Expander|Hydrator|Mapper)Plugins' vendor/spryker*/src/Spryker/<Layer>/<Module>/<Module>DependencyProvider.php`
3. `find src/Pyz -name '*ExpanderPlugin.php'` — sibling precedent in this repo is usually plentiful and settles the question.

**Corollary on duplication:** the same enrichment appearing in two or more Pyz overrides of Core classes is not a DRY problem to be solved with a shared helper — it's evidence the logic sits in the wrong layer. Extracting a helper that both overrides call preserves the layering defect and adds a hop. Move it into the plugin stack instead.

## Exceptions are not control flow (Zed)

Per https://docs.spryker.com/docs/dg/dev/backend-development/zed/business-layer/custom-exceptions — Spryker handles exceptions and errors in a central handler that does not stop execution. **Do not use exceptions as events to steer the workflow.**

- An *expected* "not found / nothing to do / empty result" is **not** exceptional — return a nullable / empty transfer or expose an explicit `has*`/`exists*` check; don't `throw` (or `...OrFail()`) and catch.
- A `try { ... } catch (\Exception $e) { log; return new XTransfer(); }` that converts any failure into an empty/default return is an anti-pattern: it both uses exceptions for flow and swallows genuine errors into an indistinguishable "not found". Narrow the catch, or let the facade return a typed empty result without throwing.

## Test-support code belongs in the Tester

In Codeception (`tests/PyzTest/**`), the test case asserts behavior; everything else — fixture builders, data setup, `have*`/`create*` helpers, factory/mock wiring reused across cases — belongs in the module's **Tester** actor (`_support`) or a Helper. Non-test helper methods accreting inside the test class itself are a convention violation ("move everything that is not a test to tester").

## One-shot data fixes — prefer the simplest vehicle

A one-time data backfill does not automatically justify a new public Facade/EntityManager method plus a Console command. Before adding that surface, ask whether the fix belongs in the migration's `postUp()` (runs once, in order, no lasting API), or at most a thin console without a dedicated facade method. Reserve the console+facade route for work that is genuinely re-runnable, needs business logic, or must be invoked outside the migration lifecycle. A throwaway public method on a Facade is speculative API surface (YAGNI).

## DependencyProvider conventions

- Constants for every dep: `FACADE_*`, `SERVICE_*`, `PLUGINS_*`, `QUERY_CONTAINER_*`.
- Always late-bound closures: `$container->set(static::FOO, function (Container $container) { return ... })`.
- Plugin stacks return arrays: `function (Container $container) { return [new PluginA(), new PluginB()]; }`.
- Separate `provideBusinessLayerDependencies()` / `providePersistenceLayerDependencies()` / `provideCommunicationLayerDependencies()` per layer.

## Module public vs private API

Public surface (what other modules may import):
- `*Facade` + `*FacadeInterface`
- `*Client` + `*ClientInterface`
- `*Service` + `*ServiceInterface`
- `*QueryContainer` + `*QueryContainerInterface`
- Plugin interfaces in `Communication/Plugin/` or `Dependency/Plugin/`
- `*Config`

Anything else (Business/* internals, repositories, entity managers, models) is **private** to the module — must not be imported across module boundaries.

## ISP — when to add an interface

Per project rule: interfaces should exist ONLY for Facade, Plugin, Service, Client classes. Adding an interface for a Reader/Manager/Builder/etc. is a violation.

## Traits — infrastructure only, not domain logic

Spryker/PHP traits used for module-owned business or view-data logic — as opposed to stateless, Kernel-level infrastructure mixins — are a settled anti-pattern in this codebase: a trait's required collaborators (facades, flags) are only enforceable via `@method` docblock hints or `abstract` method declarations, never via a typed constructor, so a composing class can omit a dependency and discover the gap only at runtime. Prefer extracting domain/view-data logic into a dedicated class (e.g. `*Provider`/`*Service`) that receives its dependencies through the module's DependencyProvider — the same reasoning that already governs ISP applies here: traits are for Kernel-level, dependency-free mixins (e.g. `FeatureFlagAwareTrait`), not for anything that needs a facade or a feature-flag client. If `abstract` method declarations are used instead of a full extraction, PHPStan can at least enforce the contract statically.

## Transfers (DTOs)

- Always declared in `*.transfer.xml`; PHP is generated.
- Use `getX()/setX()` only — never reflection / array casts.
- Transfers are deliberately anaemic — TDA / Tell-Don't-Ask does NOT apply to them.
- A transfer field declared in XML but never `getX/setX`'d in code = unused; flag during review.

## Propel command order

When schema XML changes:
```
propel:schema:copy → propel:diff → propel:migrate → propel:model:build → transfer:generate
```
Without `propel:schema:copy` first, the merged schema is stale → silent breakage.

## Translations

- Glossary keys may live in CSV files (`data/translation/Zed/<Module>/<locale>.csv`) — file-based, no DB ID concerns.
- DB glossary (`spy_glossary_key` + `spy_glossary_translation`) uses auto-increment IDs that don't have sequences — must provide explicit IDs when inserting manually.
- Cross-locale lockstep: a key added in one locale CSV must exist in all sibling locales (typically `de_CH/en_GB/fr_CH/it_CH`).
- A key referenced in code/twig but missing from CSV causes runtime fallback to the raw key — visible UX bug.
- **Merchant-Portal modules ship generated locale bundles** at `src/Pyz/Zed/**/Presentation/Components/app/locale/**/*.json`, which mirror the CSV wholesale. They are build artifacts, not usage. When checking whether a key is still referenced, restrict the grep by extension (`--include='*.twig' --include='*.php' --include='*.ts' --include='*.html'`) — an unrestricted `grep -r <key> src/` hits the bundle for every key, including orphans, so the check silently confirms itself and no dead translation is ever detectable in an MP module. 0 real-source refs + N bundle hits means orphaned.
