---
name: job-http-exchange
description: >
  Fix Renovate PRs that bump a Y-Tree lib from Feign to @HttpExchange (e.g., lib-constants v7,
  lib-private-asset-other v5). Use when the user says "fix http exchange", "fix feign migration",
  "fix lib-constants v7 PRs", or provides Renovate PRs failing due to Feign → @HttpExchange migration.
---

Fix Renovate PRs where a Y-Tree library migrated its API interfaces from `@FeignClient` to `@HttpExchange`.

## Context

When a Y-Tree library publishes a major version that replaces `@FeignClient` with `@HttpExchange`
(e.g., lib-constants v7, lib-private-asset-other v5), downstream services' Renovate PRs break because:

1. **Compilation errors** — method signatures changed (e.g., `AllConstantsGetRequest` → `List<ConstantBlocks>`)
2. **FeignClientsRegistrar NPE** — `@EnableFeignClients` scanning encounters `@HttpExchange` interfaces and throws `NullPointerException: "attributes" is null`
3. **UnsatisfiedDependencyException** — Feign-based bean factories fail because the interface no longer has `@FeignClient`/`@RequestLine` annotations

## Fix patterns

### Pattern A: Service has NO code usage of the lib's APIs

The lib API was listed in `@EnableFeignClients(clients = [...])` but never injected or called.

**Fix:** Remove the API class from the `@EnableFeignClients` clients list and its import. Still register it via `@ImportHttpServices` so the bean remains available on the classpath (other transitive consumers may depend on it).

### Pattern B: Service USES the lib's APIs

The service injects and calls the lib's API interfaces (e.g., `ConstantCurrencyApi`, `ConstantAllApi`).

**Fix:**
1. Remove the lib API interfaces from `@EnableFeignClients(clients = [...])`. If no other Feign clients remain, remove `@EnableFeignClients` entirely.
2. Remove Feign-based bean definitions for the lib APIs from `FeignClientsConfig` (or similar config class).
3. Add `@ImportHttpServices` annotation to register the APIs via Spring's HTTP service proxy:
   ```kotlin
   @ImportHttpServices(
     group = "constants",
     types = [ConstantCurrencyApi::class, ConstantAllApi::class, ConstantAssetApi::class]
   )
   ```
4. Add REST client group config in `application.properties` (only profile + base-url, NO timeouts/retry):
   ```properties
   ytree.rest-client.groups.constants.profile=internal-service
   ytree.rest-client.groups.constants.base-url=${ms.constants.base.url}
   ```
5. If Feign dependencies are no longer needed by the service (no remaining Feign clients), remove them from `build.gradle.kts`:
   - `spring-cloud-starter-openfeign`
   - `feign-okhttp`, `feign-jackson3`, `feign-form`
   - Spring Cloud BOM (`dependencyManagement` imports)
6. If a `FeignClientsConfig` class becomes empty after removing lib API beans, delete the file.

### Pattern C: Compilation errors from changed method signatures

The lib changed method signatures in the major version (e.g., parameter types/names changed).

**Fix:** Update call sites to match the new signatures. Check the lib's v7 PR or changelog for exact changes.

### Pattern D: Test configuration updates

Tests that create lib API beans via `Feign.builder()` need updating.

**Fix:** Replace Feign-based bean construction with either:
- `HttpServiceProxyFactory` + `RestClientAdapter` pointing at WireMock URL
- Or simply remove the test bean if `@ImportHttpServices` on the main config handles it (with `@TestPropertySource` overriding the base URL to WireMock)

## Steps

1. **Identify failing PRs** — user provides PR links or a search query
2. **Check build logs** to classify the failure (Pattern A/B/C/D above)
3. **Clone the repo** into `/Users/antonkiian/projects/kiiananton/AI/repos/<repo-name>` (or use existing clone)
4. **Checkout the Renovate PR branch** (e.g., `renovate/major-ytree.lib.constants`)
5. **Search the codebase** for:
   - `@EnableFeignClients` — find which lib APIs are registered
   - `FeignClientsConfig` — find Feign-based bean factories for lib APIs
   - Imports of the lib's API interfaces — determine if they're actually used
   - `Feign.builder()` in test files — find test-specific bean creation
6. **Apply the appropriate fix pattern(s)**
7. **Commit** with `Co-Authored-By: Atlas <atlas@users.noreply.github.com>`
8. **Push** to the Renovate branch

## Naming conventions

- The `group` name in `@ImportHttpServices` should match the service name (e.g., `"constants"` for ms-constants, `"private-asset-other"` for ms-private-asset-other)
- The base-url property should reference the existing property (e.g., `${ms.constants.base.url}`)
- The profile is always `internal-service` for inter-service communication

## Commit message format

```
fix: migrate <lib-name> APIs from Feign to @HttpExchange

Co-Authored-By: Atlas <atlas@users.noreply.github.com>
```
