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
