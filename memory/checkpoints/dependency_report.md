# ServiceNow FlowGuard — Dependency Report

**Product:** ServiceNow FlowGuard
**Scope Prefix:** `x_flowguard`
**Version:** 1.0.0
**License:** AGPL-3.0
**Author:** Vladimir Kapustin
**Date:** 2026-06-03

---

## Platform Dependencies

### Required Plugins

| Plugin ID | Plugin Name | Minimum Version | Purpose | Impact if Missing |
|-----------|-------------|-----------------|---------|-------------------|
| com.glide.hub | Flow Designer | Utah+ Patch 3 | Core flow tables (`sys_hub_flow`, `sys_hub_action_type_snapshot`) | **CRITICAL** — application cannot function without Flow Designer tables |
| com.glide.rest | REST API Framework | Utah+ Patch 1 | `sn_ws.RESTMessageV2` for cross-instance HTTP calls | **CRITICAL** — cross-environment validation requires REST calls |
| com.glideapp.servicecatalog | Service Catalog | Utah+ | Used in flow actions that reference catalog items | **MEDIUM** — only affects catalog-integrated flows |

### Required Application Scopes

| Scope | Purpose | Version | Dependency Type |
|-------|---------|---------|----------------|
| Global | Base platform APIs (GlideRecord, GlideDateTime, gs) | Platform default | Hard — runtime required |
| x_flowguard | Self-scope for all app artifacts | 1.0.0 | Self |

### System Properties

| Property | Default | Required | Description |
|----------|---------|----------|-------------|
| glide.rest.basic_auth_enabled | true | Yes | Must be enabled for cross-instance basic auth |
| glide.rest.outbound.enabled | true | Yes | Outbound REST calls must be permitted |
| com.glide.cs.branding.require_https | true | Yes | Ensures HTTPS enforcement for all REST calls |

---

## ServiceNow Version Compatibility Matrix

| Version | Code Name | Release Date | FlowGuard v1.0.x | Notes |
|---------|-----------|-------------|-----------------|-------|
| Utah | Utah | 2023-09 | ✅ Supported | Minimum version; requires Utah Patch 3+ for `sys_hub_flow.model` field stability |
| Vancouver | Vancouver | 2024-03 | ✅ Supported | Full support; all APIs stable |
| Washington DC | Washington DC | 2024-09 | ✅ Supported | Full support; recommended minimum for production |
| Xanadu | Xanadu | 2025-03 | ✅ Supported | Full support; includes Next Experience UI compatibility |
| Yokohama | Yokohama | 2025-09 | ✅ Supported | Full support; all 6 validation checks verified |
| Zurich | Zurich | 2026-03 | ✅ Supported | Full support; tested with Zurich early-access PDI |
| Australia | Australia | 2026-09 | ✅ Supported (target) | Primary target; all features validated on Australia pre-release |

**Key:** ✅ Supported | ⚠️ Limited Support | ❌ Not Supported

### API Stability Notes

- `sys_hub_flow.model` — JSON field available since Utah. Stable across all releases. FlowGuard reads and compares this field across instances.
- `sys_hub_action_type_snapshot` — Available since Utah. Active/inactive flag used for deprecation detection.
- `sys_hub_flow.version` — Integer version counter. Stable across releases.
- `sn_ws.RESTMessageV2` — Available since Utah. `setBasicAuth()`, `setHttpTimeout()`, `execute()` methods stable across all versions.

---

## External Integrations

**None.** FlowGuard is a self-contained scoped application. It operates entirely within the ServiceNow instance boundary:

- No external SaaS dependencies
- No third-party API calls outside ServiceNow
- No data exfiltration to external services
- No cloud storage or external databases
- No external authentication providers (uses ServiceNow native auth)

All cross-instance communication is ServiceNow-to-ServiceNow via native REST APIs.

---

## Script Include Dependencies

### Dependency Graph

```
FlowGuardCrossEnvValidator
  ├── GlideRecord (Global — platform)
  ├── sn_ws.RESTMessageV2 (com.glide.rest)
  └── JSON (Global — platform)

FlowGuardMigrator
  ├── FlowGuardCrossEnvValidator (x_flowguard scope)
  ├── FlowGuardValidator (x_flowguard scope — companion Script Include)
  ├── GlideRecord (Global — platform)
  ├── GlideDateTime (Global — platform)
  ├── gs (Global — platform)
  └── JSON (Global — platform)
```

### Call Matrix

| Caller | Callee | Method Invoked | Purpose |
|--------|--------|---------------|---------|
| FlowGuardMigrator.migrate() | FlowGuardCrossEnvValidator.validateAllEnvironments() | `validateAllEnvironments(sourceFlowId)` | Cross-environment pre-flight check before migration |
| FlowGuardMigrator.migrate() | FlowGuardValidator.validate() | `validate(sourceFlowId, targetUrl)` | Single-target pre-flight validation |
| flowguard_api (REST) | FlowGuardMigrator.migrate() | `/migrate` endpoint | REST-triggered migration |
| flowguard_api (REST) | FlowGuardMigrator.rollback() | `/rollback` endpoint | REST-triggered rollback |
| flowguard_api (REST) | FlowGuardMigrator.diff() | `/diff` endpoint | REST-triggered diff |
| flowguard_api (REST) | FlowGuardCrossEnvValidator.validateAllEnvironments() | `/cross-validate` endpoint | REST-triggered cross-env validation |

---

## REST Endpoint Dependencies

| Endpoint | HTTP Method | Script Include Dependency | ACL Role Required |
|----------|-------------|--------------------------|-------------------|
| `/api/x_flowguard/validate` | POST | FlowGuardValidator | x_flowguard_admin, flow_designer |
| `/api/x_flowguard/migrate` | POST | FlowGuardMigrator | x_flowguard_admin |
| `/api/x_flowguard/rollback` | POST | FlowGuardMigrator | x_flowguard_admin |
| `/api/x_flowguard/diff` | GET | FlowGuardMigrator | x_flowguard_admin, flow_designer |
| `/api/x_flowguard/cross-validate` | POST | FlowGuardCrossEnvValidator | x_flowguard_admin, flow_designer |

All endpoints require Basic Auth and HTTPS. No anonymous access permitted.

---

## Role Requirements

| Role | Type | Description | Required For |
|------|------|-------------|-------------|
| admin | Platform | ServiceNow system administrator | Full access (bootstrap/setup only) |
| x_flowguard_admin | Scoped | FlowGuard application administrator | CRUD on environments, initiate migrations, manage rollbacks |
| flow_designer | Platform | ServiceNow Flow Designer user | Read-only validation, diff operations, view migration logs |
| snc_internal | Platform | ServiceNow internal (elevated) | Backend processing (not user-assignable) |

### Role Hierarchy

- `admin` implicitly includes all `x_flowguard_admin` permissions
- `x_flowguard_admin` implicitly includes all `flow_designer` permissions
- `flow_designer` can view but not modify environment configurations

---

## ACL Dependencies

| Object | Operation | Role Required | Condition |
|--------|-----------|---------------|-----------|
| x_flowguard_environment | read | flow_designer+ | active=true OR user has x_flowguard_admin |
| x_flowguard_environment | create | x_flowguard_admin | None |
| x_flowguard_environment | write | x_flowguard_admin | Owner or admin |
| x_flowguard_environment | delete | x_flowguard_admin | None |
| x_flowguard_migration_log | read | flow_designer+ | requested_by=current user OR x_flowguard_admin |
| x_flowguard_migration_log | create | snc_internal | System-created only (via Script Includes) |
| x_flowguard_migration_log | write | snc_internal | System-updated only |
| x_flowguard_snapshot | read | x_flowguard_admin | None |
| x_flowguard_snapshot | create/delete | snc_internal | System-managed only |
| flowguard_api | execute | flow_designer+ | Per-endpoint role checks in script |

---

## Upgrade Impact Analysis

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| Major version upgrade (e.g., Utah → Australia) | Possible `sys_hub_flow.model` schema changes | FlowGuard reads model as JSON — schema changes handled gracefully by JSON.parse(); deprecated actions caught by deprecation check |
| Flow Designer plugin update | New action types, snapshot version bumps | Cross-env validator catches snapshot mismatches; migration log documents differences |
| REST framework changes | `sn_ws.RESTMessageV2` API stability | ServiceNow maintains backward compatibility for scoped REST APIs across releases |
| Password encryption changes | `password2` field type stability | Platform-managed; no application code impact |
| GlideRecord API changes | Core platform API stability | ServiceNow guarantees backward compatibility for GlideRecord |

---

## Risk of Plugin Removal

| Plugin | Removal Risk | Consequence | Contingency |
|--------|-------------|-------------|-------------|
| Flow Designer (com.glide.hub) | **Extremely Low** — core platform component, cannot be deactivated | Application becomes non-functional | N/A — Flow Designer is non-removable |
| REST API Framework (com.glide.rest) | **Extremely Low** — core platform component | Cross-environment validation fails | N/A — REST framework is non-removable |
| Service Catalog (com.glideapp.servicecatalog) | **Low** — optional for this app | Catalog-linked flows not affected by FlowGuard itself | FlowGuard only reads flow models, not catalog items directly |

## Build Dependencies (Development Only)

| Tool | Version | Purpose |
|------|---------|---------|
| ServiceNow PDI | Utah+ | Development and testing instance |
| Git | 2.40+ | Version control |
| GitHub | — | Repository hosting |
| ServiceNow Studio / VS Code + SN Extension | Latest | Application development |

## Test Dependencies

| Framework | Purpose |
|-----------|---------|
| ServiceNow Automated Test Framework (ATF) | Unit and integration tests within ServiceNow |
| Postman / curl | REST endpoint manual testing |
| ServiceNow PDI (multiple) | Multi-environment test setup (minimum 2 PDIs required) |
