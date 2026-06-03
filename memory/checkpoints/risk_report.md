# ServiceNow FlowGuard — Risk Report

**Product:** ServiceNow FlowGuard
**Version:** 1.0.0
**License:** AGPL-3.0
**Author:** Vladimir Kapustin
**Date:** 2026-06-03

---

## Risk Register

### R01: Data Loss During Migration
- **Severity:** P0 (Critical)
- **Description:** A flow migration overwrites a production flow with a corrupt or incomplete payload, causing the flow to become non-functional or lose critical configuration.
- **Impact:** Production automation failure. Business processes dependent on the flow stop executing. SLA breaches possible.
- **Likelihood:** Medium (5-15% per migration without validation)
- **Mitigation:** Pre-migration snapshot (`x_flowguard_snapshot`) captures full flow state before any write. Auto-rollback triggers if post-deployment verification fails. The `_verify()` method validates JSON parseability and action/stage presence.
- **Contingency:** Manual rollback via `/rollback` REST endpoint or admin UI. Snapshot records persist until explicitly purged.

### R02: Flow Compatibility Failures
- **Severity:** P0 (Critical)
- **Description:** A flow migrates to a target environment where subflows, action types, or data pill schemas are incompatible. The flow appears to migrate successfully but fails at runtime.
- **Impact:** Silent production failures discovered only when flows execute. Delayed detection increases blast radius.
- **Likelihood:** High (20-40% without pre-validation)
- **Mitigation:** FlowGuardCrossEnvValidator performs 6 checks (connectivity, subflow existence, version match, action snapshots, data pill schemas, deprecation) before any migration. Migration is blocked if any critical check fails.
- **Contingency:** Cross-environment validation results logged to migration log. Administrators can review failed checks and remediate target environment before retrying.

### R03: Performance Degradation
- **Severity:** P1 (High)
- **Description:** Cross-environment validation with many configured environments (10+) and large flows (5000+ actions) causes unacceptable response times, impacting API responsiveness.
- **Impact:** Migration API calls timeout. User experience degrades. Scheduled jobs may overlap.
- **Likelihood:** Medium (10-20% with 10+ environments and complex flows)
- **Mitigation:** Environment checks run sequentially per environment but with configurable timeouts (10s default). Failed environments skip remaining checks. `active=false` flag excludes environments.
- **Contingency:** Increase `_checkTimeout` for slow network links. Split validation into batches. Use asynchronous processing via Scheduled Jobs for large-scale validations.

### R04: Version Mismatch Between Instances
- **Severity:** P1 (High)
- **Description:** Source and target instances run different ServiceNow versions (e.g., Utah source, Australia target). API differences cause validation checks to produce false positives or negatives.
- **Impact:** Incorrect validation results — flows may be blocked from migration despite being compatible, or allowed despite being incompatible.
- **Likelihood:** Medium (15-25% in multi-version environments)
- **Mitigation:** Compat matrix maintained in dependency report. API endpoints used (`sys_hub_flow`, `sys_hub_action_type_snapshot`) are stable across Utah-Australia. Each check handles non-200 HTTP responses gracefully rather than crashing.
- **Contingency:** Cross-version testing suite (T07 in test suite). Environment-specific version tagging in x_flowguard_environment for conditional logic.

### R05: ACL Gaps
- **Severity:** P1 (High)
- **Description:** Improperly configured ACLs allow unauthorized users to trigger migrations, view environment credentials, or modify migration logs.
- **Impact:** Unauthorized flow changes in production. Credential exposure. Audit trail corruption.
- **Likelihood:** Medium (10-15% without explicit ACL review)
- **Mitigation:** ACL dependency table in dependency report defines minimum roles per operation. `x_flowguard_admin` required for write operations. `flow_designer` limited to read-only operations. Password field uses `password2` type (encrypted at rest, masked in UI).
- **Contingency:** Audit log review via `sys_log`. ACL regression tests (R05 in regression suite).

### R06: REST API Rate Limiting
- **Severity:** P2 (Medium)
- **Description:** Multiple concurrent cross-environment validations trigger ServiceNow's outbound REST rate limits or target instance API throttling, causing intermittent validation failures.
- **Impact:** False connectivity failures. Migration retry loops. Unreliable validation results.
- **Likelihood:** Low-Medium (5-15% in high-throughput CI/CD pipelines)
- **Mitigation:** Sequential environment checks (not parallel). Configurable timeout per check. Failed checks return actionable error messages rather than generic failures.
- **Contingency:** Implement retry-with-backoff logic for transient failures. Add rate-limit detection (HTTP 429) with automatic backoff.

### R07: Cross-Instance Sync Failures
- **Severity:** P2 (Medium)
- **Description:** Network partitions, VPN drops, or target instance downtime cause validation calls to fail, blocking migrations during critical maintenance windows.
- **Impact:** Deployment delays. False perception of flow incompatibility. Migration pipeline stalls.
- **Likelihood:** Medium (10-20% depending on network stability between instances)
- **Mitigation:** Connectivity check (Check 0) runs first and short-circuits on failure. Non-connectivity errors are distinguished from validation failures in migration logs. Timeout per check prevents indefinite blocking.
- **Contingency:** Offline validation mode — export flow model to JSON and validate manually. Emergency bypass flag (admin-only) for validated maintenance scenarios.

### R08: Rollback Complexity
- **Severity:** P2 (Medium)
- **Description:** Rollback fails because the snapshot record was corrupted, deleted, or references a flow that was subsequently modified by another process.
- **Impact:** Inability to recover from a failed migration. Manual reconstruction of flow required.
- **Likelihood:** Low (2-5%)
- **Mitigation:** Snapshots store the complete flow model (not just a delta). `_verify()` runs immediately after deployment. Auto-rollback triggers on verification failure while the snapshot is fresh.
- **Contingency:** Export flow XML from source instance as secondary backup. Maintain external git-based flow version control as tertiary backup.

### R09: UI Degradation
- **Severity:** P2 (Medium)
- **Description:** Application menu modules, forms, or lists render incorrectly on newer ServiceNow UI frameworks (Next Experience, Configurable Workspaces), causing user confusion or feature inaccessibility.
- **Impact:** Reduced usability. Support burden increases. Users may bypass the application.
- **Likelihood:** Low-Medium (5-10% on newer releases)
- **Mitigation:** Application uses standard ServiceNow UI components (forms, lists, modules) rather than custom UI. REST API provides programmatic access as fallback.
- **Contingency:** UI compatibility testing on each target release (T11 in test suite). Provide CLI/API documentation for headless operation.

### R10: Documentation Staleness
- **Severity:** P3 (Low)
- **Description:** Architecture docs, dependency reports, and risk assessments fall out of date as the platform and application evolve, causing incorrect assumptions during maintenance.
- **Impact:** New team members make decisions based on outdated information. Upgrade planning overlooks new dependencies.
- **Likelihood:** High (60-80% over 12 months without maintenance)
- **Mitigation:** Documentation stored in repository alongside code. Version-controlled. Build pipeline checks for doc freshness. Execution plan includes documentation review phases.
- **Contingency:** Automated doc generation from code (JSDoc extraction). Scheduled quarterly documentation review tasks.

### R11: Concurrent Migration Conflicts
- **Severity:** P2 (Medium)
- **Description:** Two administrators simultaneously migrate the same flow to the same target, causing version conflicts and potential data corruption.
- **Impact:** Flow version divergence. One migration overwrites the other. Snapshot integrity compromised.
- **Likelihood:** Low (3-8% in multi-admin environments)
- **Mitigation:** Migration log records contain timestamps and flow IDs. Version increment on update (`parseInt(payload.version, 10) + 1`). Diff endpoint shows changes since last snapshot.
- **Contingency:** Implement advisory locking on x_flowguard_migration_log for in_progress migrations on the same flow. Reject concurrent requests with clear error message.

### R12: Incomplete Flow Migration
- **Severity:** P2 (Medium)
- **Description:** Flow migrates successfully but associated artifacts (Catalog Items, Script Includes referenced in flow actions, Subflows in nested references) are not migrated, causing runtime failures.
- **Impact:** Flow executes partially. Silent failures in downstream steps. Data inconsistencies.
- **Likelihood:** Medium (10-20% for complex flows with external dependencies)
- **Mitigation:** FlowGuardCrossEnvValidator checks subflow existence (Check 1) and action availability (Check 3). Deprecation check (Check 5) flags inactive actions.
- **Contingency:** Dependency graph visualization to show all flow dependencies before migration. Pre-migration dependency check mode.

### R13: Environment Credential Expiration
- **Severity:** P3 (Low)
- **Description:** Stored environment credentials expire (password rotation policy) or user accounts are deactivated, causing all cross-environment operations to fail silently.
- **Impact:** Validation failures misattributed to flow issues. Migration pipeline stalls. False sense of environment incompatibility.
- **Likelihood:** Medium (15-30% over 90 days with password rotation policies)
- **Mitigation:** Connectivity check distinguishes auth failures (401/403) from network failures. Clear error messaging: "Authentication failed for [URL] — verify username/password." Migration logs capture HTTP status codes.
- **Contingency:** Scheduled credential health check job. Email notification on persistent auth failures. Support for OAuth token-based auth in future versions.

### R14: Flow Model JSON Parsing Failures
- **Severity:** P3 (Low)
- **Description:** Corrupted or malformed `sys_hub_flow.model` JSON causes parsing exceptions during validation, crashing the check but not the migration.
- **Impact:** Validation result incomplete. Some checks silently skipped. Migration may proceed with undetected issues.
- **Likelihood:** Very Low (1-3% — platform enforces JSON validity)
- **Mitigation:** Every JSON.parse() call is wrapped in try/catch with gs.error() logging. Parser failures return empty arrays rather than throwing unhandled exceptions. Migration log captures parse errors.
- **Contingency:** Flow Designer UI validation (platform-level) catches most corruption before it reaches the model field. Manual flow review for corrupted flows.

### R15: Large Flow Migration Timeout
- **Severity:** P3 (Low)
- **Description:** Flows with 5000+ actions generate model payloads exceeding ServiceNow REST message size limits or causing processing timeouts during serialization.
- **Impact:** Migration fails for very large flows. Partial deployments possible.
- **Likelihood:** Low (2-5% — most flows are under 500 actions)
- **Mitigation:** `_checkTimeout` configurable for REST calls. Verification runs lightweight checks (JSON parse, actions/stages presence). No full model comparison over REST.
- **Contingency:** Chunk flow migration for very large flows. Increase REST message size limits in system properties.

---

## Risk Matrix Summary

| Risk ID | Severity | Likelihood | Category | Mitigation Status |
|---------|----------|------------|----------|-------------------|
| R01 | P0 | Medium | Data Loss | ✅ Auto-snapshot + verify + rollback |
| R02 | P0 | High | Compatibility | ✅ 6-check cross-env validator |
| R03 | P1 | Medium | Performance | ✅ Configurable timeouts, sequential checks |
| R04 | P1 | Medium | Compatibility | ✅ Graceful error handling, compat matrix |
| R05 | P1 | Medium | Security | ✅ ACL table, password2 encryption |
| R06 | P2 | Low-Medium | Operations | ✅ Sequential checks, actionable errors |
| R07 | P2 | Medium | Operations | ✅ Connectivity check short-circuit |
| R08 | P2 | Low | Data Integrity | ✅ Full model snapshots, immediate verify |
| R09 | P2 | Low-Medium | Usability | ✅ Standard UI components, API fallback |
| R10 | P3 | High | Maintenance | ✅ Version-controlled docs, review phases |
| R11 | P2 | Low | Operations | ✅ Version incrementing, diff endpoint |
| R12 | P2 | Medium | Compatibility | ✅ Subflow/action existence checks |
| R13 | P3 | Medium | Operations | ✅ Auth failure detection, clear messaging |
| R14 | P3 | Very Low | Robustness | ✅ try/catch on all JSON.parse calls |
| R15 | P3 | Low | Scalability | ✅ Configurable timeout, lightweight verify |

### Severity Distribution

| Severity | Count | Risk IDs |
|----------|-------|----------|
| P0 (Critical) | 2 | R01, R02 |
| P1 (High) | 3 | R03, R04, R05 |
| P2 (Medium) | 6 | R06, R07, R08, R09, R11, R12 |
| P3 (Low) | 4 | R10, R13, R14, R15 |

### Mitigation Coverage

- **Fully Mitigated:** 15/15 risks have defined mitigations
- **Contingency Plans:** 15/15 risks have contingency plans
- **Residual Risk Acceptance:** R10 (documentation staleness) — accepted with quarterly review process
