# Regression Test Cases — ServiceNow FlowGuard v1

**Product:** ServiceNow FlowGuard
**Version:** 1.0.0
**Author:** Vladimir Kapustin
**License:** AGPL-3.0
**Date:** 2026-06-03

---

## Overview

This document defines regression test cases for FlowGuard v1.0.0. Regression tests verify that existing functionality remains intact after code changes, platform upgrades, or configuration modifications. Each case references historical issues where applicable to prevent recurrence.

---

## Test Cases

### R01: Flow Export/Import Integrity

| Field | Value |
|-------|-------|
| **ID** | R01 |
| **Scope** | Flow serialization and deserialization (FlowGuardMigrator._serializeFlow, _restorePayload, _createFromPayload) |
| **Historical Issue Reference** | Version skew observed when model JSON contained non-standard keys not captured in serialization |
| **Test Steps** | 1. Create a flow with all standard fields (name, description, model, version, active, category, type). 2. Migrate flow to target (update existing). 3. Migrate a different flow to target (create new). 4. Verify all fields match source after migration. 5. Verify version increments correctly on update. 6. Verify version is 1 on create. 7. Verify active=false on create. |
| **Expected Behavior** | All serialized fields are preserved across migration. Version is `source_version + 1` on update, `1` on create. New flows are created inactive. |
| **Pass Criteria** | Field-by-field comparison between source and post-migration target flows shows no data loss or corruption |
| **Dependencies** | T08 |

### R02: Environment Matching Logic

| Field | Value |
|-------|-------|
| **ID** | R02 |
| **Scope** | Environment filtering and ordering logic in FlowGuardCrossEnvValidator.validateAllEnvironments() |
| **Historical Issue Reference** | Inactive environments were incorrectly included in validation; environments were processed in random order causing inconsistent results |
| **Test Steps** | 1. Configure 5 environments: 3 active (order: 10, 50, 200), 2 inactive (order: 30, 100). 2. Call cross-validate. 3. Verify only 3 environments are processed. 4. Verify processing order matches 'order' field (10 → 50 → 200). 5. Deactivate one environment and re-run; verify count decreases. 6. Reactivate and re-run; verify count increases. |
| **Expected Behavior** | Only active environments are processed. Processing order respects the 'order' column ascending. Changes to active flag are reflected immediately. |
| **Pass Criteria** | Results array contains exactly 3 entries for active environments only, in ascending order |
| **Dependencies** | T01 |

### R03: Error Handling Paths — Network Failure

| Field | Value |
|-------|-------|
| **ID** | R03 |
| **Scope** | Graceful error handling when target environment is unreachable |
| **Historical Issue Reference** | Unhandled exceptions in REST calls crashed the entire validation run instead of reporting individual environment failures |
| **Test Steps** | 1. Configure 3 environments: Env A (reachable), Env B (unreachable — wrong URL or port), Env C (reachable). 2. Call cross-validate. 3. Verify Env A returns results. 4. Verify Env B returns connectivity check failure with actionable message and does NOT attempt further checks. 5. Verify Env C still runs (subsequent environments are not skipped due to one failure). |
| **Expected Behavior** | Unreachable environment fails gracefully with connectivity check: `passed: false`, issues containing "Cannot reach", `actionable: true`. Other environments continue processing. |
| **Pass Criteria** | All 3 environments in results. Env B shows connectivity failure. Env C shows complete check results. No unhandled exceptions in system logs. |
| **Dependencies** | T01 |

### R04: Error Handling Paths — Authentication Failure

| Field | Value |
|-------|-------|
| **ID** | R04 |
| **Scope** | Authentication failure detection and messaging |
| **Historical Issue Reference** | 401 responses were treated identically to network failures, causing confusion during credential rotation |
| **Test Steps** | 1. Configure an environment with invalid credentials. 2. Call cross-validate. 3. Verify HTTP 401/403 is detected as authentication failure (not generic connectivity failure). 4. Verify issue message includes "Authentication failed". 5. Verify action message references username/password verification. |
| **Expected Behavior** | Authentication failures (401/403) are distinguished from network failures. Messages clearly indicate credential issues. |
| **Pass Criteria** | Issue message contains "Authentication failed" and references HTTP status code. Action message mentions username/password. |
| **Dependencies** | T01 |

### R05: Concurrent Access — Snapshot Integrity

| Field | Value |
|-------|-------|
| **ID** | R05 |
| **Scope** | Snapshot integrity during concurrent migrations on different flows |
| **Historical Issue Reference** | Snapshot records were not properly scoped to migration_id, causing cross-contamination between concurrent migrations |
| **Test Steps** | 1. Initiate migration M1 for Flow A (updates existing). 2. Immediately initiate migration M2 for Flow B (updates existing). 3. Verify both migrations create distinct snapshots with correct migration_id. 4. Verify M1's snapshot references Flow A, not Flow B. 5. Verify M2's snapshot references Flow B, not Flow A. |
| **Expected Behavior** | Each migration creates its own snapshot scoped to its migration_id. Snapshots are not cross-contaminated. |
| **Pass Criteria** | Both snapshots exist with correct flow_id and migration_id. No cross-contamination. |
| **Dependencies** | T09, T15 |

### R06: Session/Timeout Handling

| Field | Value |
|-------|-------|
| **ID** | R06 |
| **Scope** | REST call timeout behavior and session management |
| **Historical Issue Reference** | Long-running checks blocked the entire thread; no timeout configuration was respected |
| **Test Steps** | 1. Configure an environment with a URL that responds slowly (simulate with high-latency endpoint or timeout threshold tuning). 2. Set `_checkTimeout` to 3000ms (3 seconds). 3. Call cross-validate. 4. Verify the slow environment times out after ~3 seconds. 5. Verify the timeout is reported as a connectivity issue (not an unhandled exception). 6. Verify remaining environments still process. |
| **Expected Behavior** | Checks respect configured timeout. Timeout results in clear error message. Subsequent environments are not blocked. |
| **Pass Criteria** | Timeout fires at approximately `_checkTimeout` ms. Error message is clear. Other environments process normally. |
| **Dependencies** | T01, R03 |

### R07: Rollback After Partial Migration

| Field | Value |
|-------|-------|
| **ID** | R07 |
| **Scope** | Rollback integrity when migration fails mid-pipeline (e.g., during post-deploy verification) |
| **Historical Issue Reference** | Rollback was only triggered for exceptions, not for verification failures; flows were left in corrupted state |
| **Test Steps** | 1. Create a flow in target with known-good model. 2. Simulate a migration where _verify() returns failure (e.g., by making model invalid). 3. Verify auto-rollback triggers. 4. Verify target flow model matches pre-migration snapshot exactly. 5. Verify migration log shows status=rolled_back with issues from verification failure AND rollback confirmation. |
| **Expected Behavior** | Auto-rollback triggers on verification failure. Target flow is fully restored. Log captures both the verification failure and rollback success. |
| **Pass Criteria** | Target flow model unchanged from pre-migration state. Log contains verification failure issues AND rollback confirmation. |
| **Dependencies** | T09 |

### R08: Diff Computation Consistency

| Field | Value |
|-------|-------|
| **ID** | R08 |
| **Scope** | Diff engine accuracy and idempotency |
| **Historical Issue Reference** | _deepEqual produced false negatives for nested objects due to key ordering differences in JSON serialization |
| **Test Steps** | 1. Create flow with 10 actions. 2. Run migration (creates snapshot). 3. Add 2 new actions, remove 1 action, modify 1 action. 4. Run diff. 5. Verify diff reports 2 added, 1 removed, 1 modified. 6. Run diff again — verify same results (idempotent). |
| **Expected Behavior** | Diff accurately reports added, removed, and modified actions. Deep comparison is order-independent for object keys. Results are idempotent. |
| **Pass Criteria** | `diff.added.length === 2`, `diff.removed.length === 1`, `diff.modified.length === 1`. Second run produces identical diff. |
| **Dependencies** | T08 |

### R09: JSON Parse Safety

| Field | Value |
|-------|-------|
| **ID** | R09 |
| **Scope** | JSON.parse() safety across all parser invocations |
| **Historical Issue Reference** | Corrupted model field in sys_hub_flow caused unhandled exception that crashed the entire validator |
| **Test Steps** | 1. Create a flow with intentionally corrupted model JSON (e.g., trailing comma, missing brace). 2. Call cross-validate. 3. Verify validator does NOT crash. 4. Verify error is logged via gs.error(). 5. Verify other checks continue executing (degraded but not crashed). |
| **Expected Behavior** | All JSON.parse() calls are wrapped in try/catch. Parse failures are logged and handled gracefully. Validation continues. |
| **Pass Criteria** | No unhandled exception. gs.error() message logged. Other checks complete normally. |
| **Dependencies** | T01 |

### R10: Migration Log Audit Trail Completeness

| Field | Value |
|-------|-------|
| **ID** | R10 |
| **Scope** | Migration log record completeness for all status paths |
| **Historical Issue Reference** | Migration log entries were incomplete for early-exit paths (e.g., cross-env validation failure left started_at but no completed_at or issues) |
| **Test Steps** | 1. Trigger a migration that fails at cross-env validation. 2. Trigger a migration that fails at pre-flight validation. 3. Trigger a migration that succeeds. 4. Trigger a migration that fails and rolls back. 5. Verify all 4 log entries have: migration_id, source_flow_id, status, requested_by, started_at, and (where applicable) completed_at and issues. |
| **Expected Behavior** | Every migration attempt produces a complete log entry regardless of outcome. Status accurately reflects the failure point. |
| **Pass Criteria** | All 4 log entries complete with appropriate fields. Status values: cross_env_validation_failed, validation_failed, success, rolled_back. |
| **Dependencies** | T08, T09 |
