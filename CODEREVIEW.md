# Code Review: codraw/application

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
**composer.json (M2, M5):**

- Added `doctrine/persistence` `^2.2 || ^3.0` to `require` (hard-imported by `Configuration/DoctrineConfigurationRegistry.php` and `DependencyInjection/SystemMonitoringIntegration.php`).
- Moved `symfony/console` `^6.4.0` from `require-dev` to `require` (hard-imported by `SystemMonitoring/Command/SystemStatusesCommand.php` and `Versioning/Command/UpdateDeployedVersionCommand.php`).
- Added `symfony/http-foundation` `^6.4.0` to `require` (hard-imported by `SystemMonitoring/Action/PingAction.php`).
- Added `symfony/validator` `^6.4.0` to `require` (attributes in `Configuration/Entity/Config.php`; already guaranteed transitively via `codraw/core`).
- Added a `suggest` section: `doctrine/dbal` (DBAL connection status providers), `symfony/messenger` (MessengerStatusProvider bridge — both bridges are activated conditionally via `interface_exists()` in `SystemMonitoringIntegration::prepend()`), and `codraw/framework-extra-bundle` (Symfony integration, matching the `codraw/messenger` package convention).
- `symfony/http-kernel` did NOT need to be added: the M2 code fix removed the only reference to it.
- Open item (unchanged, per repo convention): the `DependencyInjection/*` integration classes still depend on dev-only `codraw/dependency-injection` / `symfony/config`; siblings (`codraw/messenger`, `codraw/doctrine-extra`) treat these integration classes as active only under `codraw/framework-extra-bundle`, now documented via `suggest`.

**Code fixes:**

- **H1** — `Configuration/Entity/Config.php`: `getValue()` now uses `?? null` instead of `?: null`, so falsy stored values (`false`, `0`, `''`, `[]`) are returned intact.
- **H2** — `Feature/FeatureInitializer.php`: a missing configuration key now `continue`s to the next property instead of `return`ing out of `initialize()`, so remaining properties and the `SelfInitializeFeatureInterface::initialize()` hook run.
- **H3** — `Configuration/DoctrineConfigurationRegistry.php`: `has()` now uses `isset()` instead of `\array_key_exists()`, so a cached `null` from a previous `get()` miss no longer makes `has()` return `true`.
- **M1** — `Versioning/EventListener/FetchRunningVersionListener.php`: the project directory interpolated into the `git describe` shell command is now wrapped in `escapeshellarg()`.
- **M2** — `Feature/FeatureInitializer.php`: replaced `ArgumentMetadata::IS_INSTANCEOF` (undeclared `symfony/http-kernel`) with the native `\ReflectionAttribute::IS_INSTANCEOF` and removed the import.
- **M3** — `Versioning/VersionManager.php`: the `''` sentinel is reset to `null` when the dispatch throws, so a transient failure is no longer permanently cached as "no version".
- **M4** — `SystemMonitoring/Bridge/Doctrine/DBALPrimaryReadReplicaConnectionStatusProvider.php`: the trailing connection-restore step is now a best-effort `try`/`catch`, so a down database yields the intended ERROR statuses instead of throwing out of the generator.
- **M6** — `SystemMonitoring/Bridge/Symfony/Messenger/MessengerStatusProvider.php`: switched to the `Symfony\Component\Messenger\Transport\InMemory\InMemoryTransport` namespace (alias removed in Symfony 7).
- **L2** — `SystemMonitoring/Action/PingAction.php`: `json_encode()` now uses `\JSON_INVALID_UTF8_SUBSTITUTE | \JSON_THROW_ON_ERROR`, so invalid UTF-8 in driver error messages is substituted rather than producing a `TypeError` in `JsonResponse`.
- **L5** — `Versioning/EventListener/FetchRunningVersionListener.php`: a `false` return from `file_get_contents()` on `public/version.txt` now returns early, letting the git fallback run instead of setting an empty version.

**Not fixed** (see findings below): L1 (cache/refresh design decision), L3 (adding `isRequired()` to `service` could reject currently-valid disabled provider entries), L4 (API-timing refactor, no current caller affected).

**Validation (2026-07-20):** `composer install` resolves cleanly with the updated constraints; full PHPUnit suite passes (59 tests, 147 assertions — the 14 PHPUnit "mock without expectations" notices are pre-existing, verified identical with the fixes stashed); PHPStan (`phpstan.dist.neon`) reports no errors and required no baseline changes; markdownlint-cli2 reports no violations. No test expectations needed updating — no existing test pinned the old buggy behavior.

Reviewed package: `/Users/tlacroix/Sites/localhost/codraw/codraw-application` (namespace `Draw\Component\Application`), a Symfony bundle component providing Configuration (DB-backed key/value registry), Feature initialization, System Monitoring, and Versioning.

## Overall Assessment

The package is small, well-organized, and follows a consistent architecture shared with its sibling "codraw" packages (DI integration classes, tagged-iterator extensibility, contract interfaces from `codraw/contracts`). Exception handling deliberately wraps infrastructure failures into domain exceptions, and the monitoring model (Status aggregation, contexts, providers) is cleanly designed. However, the review found three high-severity correctness bugs in normal-use paths (falsy config values silently lost, `FeatureInitializer` aborting initialization mid-way, `has()` returning true for known-missing keys), several medium issues (unescaped shell interpolation, an undeclared `symfony/http-kernel` dependency in a core code path, failure-state caching in `VersionManager`, an exception escape path in the read-replica monitor), and a significant test-coverage gap: the entire SystemMonitoring runtime and the FeatureInitializer have no behavioral tests.

## Findings

### High

#### **[FIXED]** H1. Falsy configuration values are silently converted to `null`

`Configuration/Entity/Config.php:57`

```php
public function getValue()
{
    return $this->data['value'] ?: null;
}
```

The short-ternary coerces every falsy stored value (`false`, `0`, `0.0`, `''`, `[]`) to `null`. Storing a boolean flag via `DoctrineConfigurationRegistry::set('flag', false)` then reading it back returns `null`, and because `DoctrineConfigurationRegistry::get()` (`Configuration/DoctrineConfigurationRegistry.php:46`) returns `getValue()` whenever the entity exists, the caller's `$default` is not applied either — the stored `false` is unrecoverable. Should be `return $this->data['value'] ?? null;` (or simply `return $this->data['value'];` since the key is always set). This is a data-fidelity bug in the primary use case of a configuration registry (feature flags).

#### **[FIXED]** H2. `FeatureInitializer` aborts initialization with `return` instead of `continue`

`Feature/FeatureInitializer.php:28-30`

```php
if (!\array_key_exists($property->getName(), $configuration)) {
    return;
}
```

When any `#[Config]`-attributed property is missing from the stored configuration, the whole `initialize()` method returns: remaining properties are skipped and `SelfInitializeFeatureInterface::initialize()` (line 35-37) is never called. Worse, properties processed *before* the missing one were already assigned, leaving the feature object in a partially-initialized, order-dependent state. Adding one new property to a feature class silently disables initialization of everything declared after it plus the self-initialize hook. This should almost certainly be `continue`. The class has no unit test, so the bug is undetected.

#### **[FIXED]** H3. `has()` returns `true` for a key that `get()` just determined does not exist

`Configuration/DoctrineConfigurationRegistry.php:35-36` and `:56`

`get()` caches the result of `find()` unconditionally: `$this->configs[$name] = $this->find($name);` — including `null` when the row does not exist. `has()` then uses `\array_key_exists($name, $this->configs)`, which returns `true` for that cached `null`. Sequence: `$registry->get('missing'); $registry->has('missing');` → `true`, contradicting reality. `isset()` in `get()` itself masks the problem for reads, but `has()` is broken after any miss-read. Either don't cache `null` results, or make `has()` check `isset()` / re-query.

### Medium

#### **[FIXED]** M1. Unescaped shell interpolation of the project directory

`Versioning/EventListener/FetchRunningVersionListener.php:45-52`

```php
$version = exec(
    \sprintf('(cd %s && git describe --tags --always --dirty) 2>&1', $this->projectDirectory),
    ...
);
```

`$projectDirectory` is injected verbatim into a shell command. With the default wiring it is `%kernel.project_dir%`, so exploitation requires control of the deployment path, but any path containing spaces, `$`, `&`, or quotes breaks the command or executes unintended shell code. Use `escapeshellarg()`, or better `git -C <dir> describe ...` via `Symfony\Component\Process\Process` with an argument array (no shell at all).

#### **[FIXED]** M2. Core code path depends on undeclared `symfony/http-kernel` (and misuses `ArgumentMetadata::IS_INSTANCEOF`)

`Feature/FeatureInitializer.php:7` and `:24`

`FeatureInitializer::initialize()` calls `$property->getAttributes(Config::class, ArgumentMetadata::IS_INSTANCEOF)`. `ArgumentMetadata` belongs to `symfony/http-kernel`, which is in neither `require` nor `require-dev` of `composer.json`. If http-kernel is absent, resolving the constant throws `Error: Class "...ArgumentMetadata" not found` at runtime. The correct native constant is `\ReflectionAttribute::IS_INSTANCEOF` — no Symfony dependency needed at all.

#### **[FIXED]** M3. `VersionManager` permanently caches a transient failure as "no version"

`Versioning/VersionManager.php:25-37`

`getRunningVersion()` sets `$this->runningVersion = ''` *before* dispatching the event. If the dispatch throws, `VersionInformationIsNotAccessibleException` is raised once, but the sentinel `''` remains, so every subsequent call returns `null` without retrying and without error. Consequences: `isUpToDate()` can incorrectly compare `null`, and `updateDeployedVersion()` could store an empty version after a transient listener failure earlier in the process. Reset the property (or move the sentinel assignment after a successful dispatch) in a `catch`/`finally`.

#### **[FIXED]** M4. Read-replica status provider can throw out of the generator after yielding results

`SystemMonitoring/Bridge/Doctrine/DBALPrimaryReadReplicaConnectionStatusProvider.php:59-61`

The final connection-restore step runs outside any `try`/`catch`:

```php
$previousConnectionToPrimary
    ? $connection->ensureConnectedToPrimary()
    : $connection->ensureConnectedToReplica();
```

If the database is down (the very situation monitoring exists to report), the earlier checks correctly yield `Status::ERROR`, but this trailing call throws during generator completion, propagating uncaught through `DoctrineConnectionServiceStatusProvider` and `System::getServiceStatuses()`. The ping endpoint then returns a 500 with a stack trace instead of the intended clean 502 status report. Wrap the restore in a try/catch (best-effort restore).

#### **[FIXED]** M5. Multiple undeclared runtime dependencies; no `suggest` section

`composer.json`

Beyond M2, the package's shipped source references packages absent from `require`: `symfony/http-foundation` (`SystemMonitoring/Action/PingAction.php`), `symfony/console` (`SystemMonitoring/Command/SystemStatusesCommand.php`, `Versioning/Command/UpdateDeployedVersionCommand.php` — only in `require-dev`), `symfony/validator` (`Configuration/Entity/Config.php` attributes), `doctrine/dbal` (SystemMonitoring Doctrine bridge), `symfony/messenger` (Messenger bridge), `symfony/config`/`symfony/dependency-injection` (`DependencyInjection/*`). The bridge classes are arguably intentionally optional, but there is no `suggest` block documenting that, and non-bridge classes (PingAction, both Commands, the entity) fail hard if installed standalone.

#### **[FIXED]** M6. Deprecated `InMemoryTransport` class alias — breaks under Symfony 7

`SystemMonitoring/Bridge/Symfony/Messenger/MessengerStatusProvider.php:9,28`

The code imports `Symfony\Component\Messenger\Transport\InMemoryTransport`, deprecated since Symfony 6.3 in favor of `Symfony\Component\Messenger\Transport\InMemory\InMemoryTransport`; the alias is removed in Symfony 7. The sibling package `codraw-tester-bundle` already uses the new namespace (`Messenger/TransportTester.php`), so this file is inconsistent with the rest of the framework and will fatal when the constraint set eventually moves past 6.4.

### Low

#### L1. `get()` performs a `refresh()` round-trip on every cached read

`Configuration/DoctrineConfigurationRegistry.php:43`

Every cache hit calls `EntityManager::refresh()`, i.e., a `SELECT` per read — the in-memory `$configs` cache saves nothing but object allocation. If always-fresh reads are the intent, the cache is redundant; if caching is the intent, refresh defeats it. Config values read in hot paths (feature flags) will hammer the database.

#### **[FIXED]** L2. `json_encode()` failure unhandled in PingAction

`SystemMonitoring/Action/PingAction.php:47-54`

`json_encode(..., \JSON_PRETTY_PRINT)` can return `false` (e.g., invalid UTF-8 in an error message coming from a DB driver exception), which is then passed to `JsonResponse` with `json: true`, producing a confusing `TypeError` instead of a monitoring response. Use `\JSON_THROW_ON_ERROR | \JSON_INVALID_UTF8_SUBSTITUTE`, or just pass the array to `JsonResponse` and let it encode.

#### L3. `service` key optional in system_monitoring config but required at load time

`DependencyInjection/SystemMonitoringIntegration.php:61` and `:167`

`scalarNode('service')` has no `isRequired()`/`cannotBeEmpty()`, but `load()` executes `new Reference($serviceStatusProvider['service'])`; a provider entry without `service` produces a `TypeError` during container compilation instead of a clear configuration error message.

#### L4. Context guard in `MonitoredService::getServiceStatuses()` is deferred by generator semantics

`SystemMonitoring/MonitoredService.php:37-44`

Because the method uses `yield from`, the `RuntimeException` for an unsupported context is thrown only when the caller starts iterating, not at call time. Current callers iterate immediately, but the API contract is misleading; splitting the guard from the generator (guard in a plain method returning the generator) would make it eager.

#### **[FIXED]** L5. `file_get_contents()` result unchecked

`Versioning/EventListener/FetchRunningVersionListener.php:34`

If `public/version.txt` exists but is unreadable (permissions, race with deployment), `file_get_contents()` returns `false` with a warning and `trim(false)` sets the running version to `''` via `setRunningVersion('')` — which also calls `stopPropagation()`, preventing the git fallback from running. A readability check or `if (false === $content) return;` would let the fallback proceed.

## Strengths

- Clear, small, single-purpose classes with consistent structure across the four sub-domains (Configuration, Feature, SystemMonitoring, Versioning).
- Disciplined exception translation: infrastructure `\Throwable`s are wrapped into contract-level exceptions (`ConfigurationIsNotAccessibleException`, `VersionInformationIsNotAccessibleException`) while domain exceptions are rethrown untouched (`Configuration/DoctrineConfigurationRegistry.php:47-51`).
- Thoughtful status-aggregation semantics in `MonitoringResult` (ERROR dominates with early exit, UNKNOWN outranks OK, empty result set is UNKNOWN rather than falsely OK).
- Extensible monitoring design: tagged `ConnectionStatusProviderInterface` with priorities, per-provider contexts/options, autoconfiguration, and conditional default providers based on installed packages (`DependencyInjection/SystemMonitoringIntegration.php:109-137`).
- `DBALPrimaryReadReplicaConnectionStatusProvider` checks both primary and replica separately and attempts to restore the previous connection mode — more careful than the typical "SELECT 1" health check.
- DI integrations correctly exclude entities, attributes, events, and exceptions from service registration via the shared `IntegrationTrait` defaults, and Doctrine ORM mapping is prepended automatically for the `Config` entity.
- Modern PHP: attributes, enums (`Status`), constructor promotion, first-class callable-free clean code; nearly empty phpstan baseline (one config-builder chaining false positive).
- The `DoctrineConfigurationRegistry` test suite runs against a real MySQL schema and covers cross-scope value changes and detached-entity states — genuinely valuable integration coverage.

## Test Coverage

- **Well covered:** Versioning (VersionManager, FetchRunningVersionEvent, FetchRunningVersionListener including git and version.txt paths, UpdateDeployedVersionCommand) and Configuration (`DoctrineConfigurationRegistryTest` against real MySQL with refresh/detach edge cases; `ConfigTest` for the entity). All four DI integrations have wiring tests via `IntegrationTestCase` verifying service ids, aliases, and prepended config.
- **Not covered:** the entire SystemMonitoring runtime — `System`, `MonitoringResult` (aggregation logic), `ServiceStatus`, `MonitoredService` (context matching), `PingAction` (HTTP status mapping and payload shape), `SystemStatusesCommand`, `MessengerStatusProvider`, and all three Doctrine bridge providers have no behavioral tests, only DI registration checks.
- **Not covered:** `FeatureInitializer` has zero tests — which is exactly where the high-severity H2 bug lives; a single test with two attributed properties and a partial configuration would have caught it.
- The falsy-value bug (H1) survives because tests only store truthy strings; the `has()`-after-miss bug (H3) survives because `testHasNotSet` runs before any `get()` on the same key.
