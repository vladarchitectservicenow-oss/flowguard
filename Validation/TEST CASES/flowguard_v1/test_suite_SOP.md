# Test Suite Standard Operating Procedure — ServiceNow FlowGuard v1

**Product:** ServiceNow FlowGuard
**Version:** 1.0.0
**Test Suite Version:** flowguard_v1
**Author:** Vladimir Kapustin
**License:** AGPL-3.0
**Date:** 2026-06-03

---

## SOP Overview

This SOP defines the complete test execution procedure for ServiceNow FlowGuard v1.0.0. It covers test environment setup, prerequisites, execution order, detailed scenarios, expected results, pass/fail criteria, and regression triggers.

### Testing Philosophy

FlowGuard tests follow a layered approach:
1. **Unit tests:** Validate individual validation checks (connectivity, existence, version, snapshots, data pills, deprecation)
2. **Integration tests:** Validate Script Include interactions and REST endpoint behavior
3. **Regression tests:** Ensure existing functionality remains intact after changes
4. **Edge case tests:** Stress the system at boundary conditions

### Test Environment Setup

| Requirement | Specification |
|-------------|---------------|
| Source PDI | ServiceNow Australia release or later, with FlowGuard scoped app installed |
| Target PDI(s) | Minimum 1 target PDI (2 recommended for cross-environment tests), Australia release |
| Network | HTTPS connectivity between source and target PDIs |
| Credentials | Admin or x_flowguard_admin user on source; REST API user on target(s) |
| Test Flow Data | Minimum 5 test flows with varying complexity (1-500 actions, 0-10 subflows) |
| Tools | ServiceNow ATF, Postman/curl, browser DevTools |

### Prerequisites

1. FlowGuard scoped application (`x_flowguard`) installed and activated on source PDI
2. Flow Designer plugin (`com.glide.hub`) active on all PDIs
3. REST API Framework (`com.glide.rest`) active on all PDIs
4. At least 3 environments configured in `x_flowguard_environment` table
5. Test user with `x_flowguard_admin` role
6. Test user with `flow_designer` role (for permission-negative tests)
7. At least 5 test flows created in Flow Designer with known subflow/action configurations

### Execution Order

Tests must be executed in this order. Dependencies are documented per scenario.

1. **T01-T06:** Individual validation check tests (no dependencies)
2. **T07-T09:** Integration tests (depend on T01-T06 passing)
3. **T10-T12:** Negative/error path tests (depend on T01-T09 passing)
4. **T13-T15:** Stress and boundary tests (depend on all prior tests)

---

## Test Scenarios

### T01: Connectivity Validation — Positive Case

| Field | Value |
|-------|-------|
| **ID** | T01 |
| **Title** | Connectivity check succeeds against reachable, authenticated environment |
| **Priority** | P0 (Critical) |
| **Preconditions** | Target PDI is online, credentials in x_flowguard_environment are valid, HTTPS reachable |
| **Steps** | 1. Configure an environment in x_flowguard_environment with valid URL, username, password. 2. Set active=true. 3. Call POST /api/x_flowguard/cross-validate with a valid flow_id. 4. Inspect the connectivity check in the response. |
| **Expected Results** | Check 0 (connectivity) returns `passed: true`, HTTP 200, no issues array entries. `valid: true` on the environment result. |
| **Pass Criteria** | `checks[0].passed === true` AND `checks[0].severity === 'critical'` AND `checks[0].issues.length === 0` |
| **Fail Criteria** | `passed: false` or HTTP errors in issues array |
| **Dependencies** | PDI network connectivity |

### T02: Subflow Existence Validation

| Field | Value |
|-------|-------|
| **ID** | T02 |
| **Title** | Subflow existence check detects missing subflow in target environment |
| **Priority** | P0 (Critical) |
| **Preconditions** | Source flow contains a reference to a subflow. That subflow does NOT exist in target PDI. |
| **Steps** | 1. Create a flow in source that calls a subflow (Subflow A). 2. Ensure Subflow A does NOT exist in target. 3. Call cross-validate for the flow. 4. Check response. |
| **Expected Results** | Check 1 (subflow_existence) returns `passed: false`, with an issue containing the subflow name and sys_id, severity `critical`, `actionable: true`, and an action message directing deployment of the missing subflow. |
| **Pass Criteria** | `checks[1].passed === false` AND `checks[1].issues[0].message` contains subflow name AND `checks[1].issues[0].actionable === true` |
| **Fail Criteria** | Check passes despite missing subflow, or issue message does not identify the subflow |
| **Dependencies** | T01 (connectivity must work) |

### T03: Subflow Version Mismatch Detection

| Field | Value |
|-------|-------|
| **ID** | T03 |
| **Title** | Version mismatch check correctly identifies and reports version differences |
| **Priority** | P0 (Critical) |
| **Preconditions** | Subflow exists in both source and target, but target has different version number. |
| **Steps** | 1. Create Subflow A v5 in source. 2. Deploy Subflow A v3 to target (older version). 3. Create flow referencing Subflow A in source. 4. Call cross-validate. 5. Inspect version check. |
| **Expected Results** | Check 2 (subflow_versions) returns `passed: false`, severity `critical`, issue contains `version mismatch` with source_version=5 and target_version=3. Action suggests deploying v5 to target. |
| **Pass Criteria** | `checks[2].passed === false` AND issue contains both version numbers AND action message is appropriate for source > target scenario |
| **Fail Criteria** | Version difference not detected, or wrong version numbers reported |
| **Dependencies** | T01, T02 |

### T04: Action Snapshot Compatibility

| Field | Value |
|-------|-------|
| **ID** | T04 |
| **Title** | Action snapshot check identifies missing spoke/action types in target |
| **Priority** | P0 (Critical) |
| **Preconditions** | Source flow uses an action type that is NOT installed in target (e.g., a spoke that was never deployed to target). |
| **Steps** | 1. Create flow using an action from a specific spoke. 2. Ensure spoke is NOT installed in target. 3. Call cross-validate. 4. Inspect action_snapshots check. |
| **Expected Results** | Check 3 (action_snapshots) returns `passed: false`, issue identifies the action name and action_type_id, message indicates "not found in target — spoke may be missing", `actionable: true`. |
| **Pass Criteria** | `checks[3].passed === false` AND issue references the action_type_id AND action suggests spoke installation |
| **Fail Criteria** | Missing action type not detected, or check crashes instead of reporting |
| **Dependencies** | T01 |

### T05: Data Pill Schema Validation

| Field | Value |
|-------|-------|
| **ID** | T05 |
| **Title** | Data pill schema check detects missing inputs/outputs in target subflow |
| **Priority** | P0 (Critical) |
| **Preconditions** | Subflow exists in target but with different input/output signatures. |
| **Steps** | 1. Create Subflow B in source with inputs `[incident_id, priority]` and outputs `[resolution_code]`. 2. Deploy modified Subflow B to target with only input `[incident_id]` and no outputs. 3. Create flow calling Subflow B in source. 4. Call cross-validate. |
| **Expected Results** | Check 4 (data_pill_schemas) returns `passed: false`, issues identify missing input `priority` and missing output `resolution_code`, severity `critical`, actionable messages. |
| **Pass Criteria** | `checks[4].passed === false` AND at least 2 issues detected (one for missing input, one for missing output) AND issues specify step name and missing pill name |
| **Fail Criteria** | Schema differences not detected |
| **Dependencies** | T01, T02 |

### T06: Deprecated Action Detection

| Field | Value |
|-------|-------|
| **ID** | T06 |
| **Title** | Deprecation check flags inactive/deprecated actions in target |
| **Priority** | P1 (High) |
| **Preconditions** | Source flow uses an action type that exists in target but is marked `active=false`. |
| **Steps** | 1. Identify an action type that is active in source. 2. In target, set the same action type's active flag to false (via sys_hub_action_type_snapshot). 3. Call cross-validate. 4. Inspect action_deprecation check. |
| **Expected Results** | Check 5 (action_deprecation) returns `passed: false`, severity `warning`, issue indicates "deprecated/inactive in target", actionable message suggests replacement or spoke update. |
| **Pass Criteria** | `checks[5].passed === false` AND `checks[5].severity === 'warning'` AND issue mentions deprecation/inactive status |
| **Fail Criteria** | Deprecated action not detected |
| **Dependencies** | T01 |

### T07: Cross-Version Compatibility (Utah → Australia)

| Field | Value |
|-------|-------|
| **ID** | T07 |
| **Title** | FlowGuard handles version differences between Utah-source and Australia-target instances |
| **Priority** | P1 (High) |
| **Preconditions** | Source runs Utah, target runs Australia. Both have compatible flows. |
| **Steps** | 1. Configure source (Utah) and target (Australia) environments. 2. Create a standard flow in source. 3. Call cross-validate. 4. Verify all 6 checks execute without platform-version errors. |
| **Expected Results** | All checks execute successfully. No "API not found" or platform-version errors. Results accurately reflect cross-version flow state (matches or mismatches). |
| **Pass Criteria** | All 6 checks complete without HTTP 4xx/5xx caused by version incompatibility. Error handling catches any version-specific issues gracefully. |
| **Fail Criteria** | Any check crashes due to API differences between Utah and Australia |
| **Dependencies** | T01-T06, Utah PDI, Australia PDI |

### T08: Full Migration Pipeline (End-to-End)

| Field | Value |
|-------|-------|
| **ID** | T08 |
| **Title** | Complete flow migration from source to target succeeds end-to-end |
| **Priority** | P0 (Critical) |
| **Preconditions** | Flow exists in source, does NOT exist in target. All subflows and actions compatible. |
| **Steps** | 1. Call POST /api/x_flowguard/migrate with flow_id and target_instance. 2. Monitor migration log (x_flowguard_migration_log) for status progression. 3. Verify flow appears in target sys_hub_flow table. 4. Verify migration log shows status=success. 5. Verify snapshot was NOT created (new flow, not overwritten). |
| **Expected Results** | Migration succeeds. Log shows: cross-environment validation passed → pre-flight validation passed → flow created → post-deploy verification passed → status=success. Target flow has correct name, model, version=1, active=false. |
| **Pass Criteria** | `result.success === true`, `result.status === 'success'`, flow exists in target with correct data, migration log complete |
| **Fail Criteria** | Migration fails at any phase, or flow data is incorrect in target |
| **Dependencies** | T01-T06 |

### T09: Auto-Rollback on Verification Failure

| Field | Value |
|-------|-------|
| **ID** | T09 |
| **Title** | Migration auto-rollback restores target flow when post-deploy verification fails |
| **Priority** | P0 (Critical) |
| **Preconditions** | Flow exists in target (will be updated). Source flow has a deployable payload. Verification is configured to fail (e.g., corrupted model). |
| **Steps** | 1. Create flow in target with known-good model. 2. Configure migration to produce a flow with unparseable model. 3. Call migrate. 4. Verify migration log shows status=rolled_back. 5. Verify target flow's model is restored to pre-migration state. |
| **Expected Results** | Snapshot created pre-migration. Verification fails. Auto-rollback executes. Log shows status=rolled_back. Target flow model matches pre-migration snapshot. |
| **Pass Criteria** | `result.status === 'rolled_back'`, target flow model unchanged from pre-migration state, snapshot record exists and was used for rollback |
| **Fail Criteria** | Target flow left in corrupted state, snapshot not created, rollback not triggered |
| **Dependencies** | T08 |

### T10: Invalid Flow ID — Negative Case

| Field | Value |
|-------|-------|
| **ID** | T10 |
| **Title** | System handles invalid/non-existent flow ID gracefully |
| **Priority** | P2 (Medium) |
| **Preconditions** | None |
| **Steps** | 1. Call POST /api/x_flowguard/migrate with a flow_id that does not exist in sys_hub_flow. 2. Observe response. |
| **Expected Results** | Response returns `success: false`, errors containing "Source flow not found", migration log shows status=error. No exception thrown. |
| **Pass Criteria** | `result.success === false`, error message clearly indicates flow not found, system does not crash |
| **Fail Criteria** | Unhandled exception, empty response, or misleading error message |
| **Dependencies** | None |

### T11: Missing Permissions — Negative Case

| Field | Value |
|-------|-------|
| **ID** | T11 |
| **Title** | REST endpoints reject requests from users without required roles |
| **Priority** | P1 (High) |
| **Preconditions** | Test user has flow_designer role but NOT x_flowguard_admin. |
| **Steps** | 1. Authenticate as flow_designer user (not admin). 2. Attempt POST /api/x_flowguard/migrate. 3. Attempt POST /api/x_flowguard/rollback. 4. Attempt GET /api/x_flowguard/diff. |
| **Expected Results** | Migrate and rollback endpoints return HTTP 403 Forbidden or equivalent ACL rejection. Diff endpoint returns results (read-only operation is permitted for flow_designer). |
| **Pass Criteria** | Write operations blocked for flow_designer. Read operations permitted. |
| **Fail Criteria** | Write operations succeed without x_flowguard_admin role |
| **Dependencies** | ACL configuration (P0-06) |

### T12: Cross-Version Incompatibility — Negative Case

| Field | Value |
|-------|-------|
| **ID** | T12 |
| **Title** | Flow with incompatible subflows is blocked from migration |
| **Priority** | P1 (High) |
| **Preconditions** | Source flow references 3 subflows. 1 subflow is missing in target, 1 has version mismatch, 1 has incompatible action type. |
| **Steps** | 1. Configure incompatible environment. 2. Call cross-validate. 3. Verify overall result is valid=false. 4. Verify all 3 issues are reported in respective checks. |
| **Expected Results** | Overall valid=false. Three distinct issues reported across checks 1, 2, and 3. Each issue is actionable with specific remediation steps. |
| **Pass Criteria** | `result.valid === false`, all 3 incompatibilities detected and reported, migration blocked |
| **Fail Criteria** | Any incompatibility undetected, migration allowed despite incompatibilities |
| **Dependencies** | T01-T06 |

### T13: Empty Environment List — Boundary Case

| Field | Value |
|-------|-------|
| **ID** | T13 |
| **Title** | Cross-environment validation handles zero configured environments |
| **Priority** | P2 (Medium) |
| **Preconditions** | x_flowguard_environment table has no active records (all records active=false or table is empty). |
| **Steps** | 1. Deactivate or delete all environment records. 2. Call POST /api/x_flowguard/cross-validate. 3. Observe response. |
| **Expected Results** | Response returns `valid: true` with empty results array. Migration proceeds (no environments to validate against). No errors. |
| **Pass Criteria** | `result.valid === true`, `result.results.length === 0`, no exceptions |
| **Fail Criteria** | Error thrown, migration blocked, or invalid response |
| **Dependencies** | None |

### T14: Large Flow Validation (1000+ Actions)

| Field | Value |
|-------|-------|
| **ID** | T14 |
| **Title** | Validator handles large flows with 1000+ actions without timeout |
| **Priority** | P2 (Medium) |
| **Preconditions** | Source flow has ≥1000 actions with various subflows, action types, and data pills. |
| **Steps** | 1. Create or import a flow with 1000+ actions. 2. Call cross-validate. 3. Measure response time. 4. Verify all checks complete without timeout. |
| **Expected Results** | All checks complete within 30 seconds (configurable timeout of 10s per environment × 3 environments max). Results array includes all checks. No timeout errors. |
| **Pass Criteria** | Response received within 30s, all checks present, no timeout errors |
| **Fail Criteria** | Timeout, partial results, or crash |
| **Dependencies** | T01-T06 |

### T15: Concurrent Access Safety

| Field | Value |
|-------|-------|
| **ID** | T15 |
| **Title** | Concurrent migration attempts on same flow are handled safely |
| **Priority** | P2 (Medium) |
| **Preconditions** | Flow exists in source. Target has matching flow. |
| **Steps** | 1. Initiate migration M1 for flow F1. 2. While M1 is in progress, initiate migration M2 for same flow F1. 3. Observe both results. 4. Verify target flow is not corrupted. |
| **Expected Results** | Both migrations complete or one is rejected. Target flow ends in a consistent state (not partially updated). Migration logs show both attempts with correct statuses. |
| **Pass Criteria** | Target flow is consistent (valid JSON, actions present). No data corruption. Both migration log entries present. |
| **Fail Criteria** | Target flow corrupted, snapshot lost, inconsistent state |
| **Dependencies** | T08 |

---

## Test Data Requirements

| Data Item | Description | Used In |
|-----------|-------------|---------|
| Flow_Simple | Flow with 1 action, no subflows | T01, T06, T10 |
| Flow_Subflows | Flow referencing 3 subflows at known versions | T02, T03, T05 |
| Flow_Spoke | Flow using actions from a specific spoke | T04 |
| Flow_Complex | Flow with 50+ actions, 5+ subflows, 10+ data pills | T08, T09, T12 |
| Flow_Large | Flow with 1000+ actions | T14 |
| Env_Prod | Production environment config (valid credentials) | T01, T08 |
| Env_QA | QA environment config (partially incompatible) | T07, T12 |
| Env_Empty | No active environments | T13 |
| User_Admin | User with x_flowguard_admin role | All tests |
| User_Designer | User with flow_designer only | T11 |

---

## Test Execution Log Template

```markdown
# FlowGuard v1 Test Execution Log
**Date:** YYYY-MM-DD
**Tester:** [Name]
**Source PDI:** [URL]
**Target PDI(s):** [URLs]
**Build:** [commit hash]

| Test ID | Title | Result | Duration | Notes |
|---------|-------|--------|----------|-------|
| T01 | Connectivity Validation | PASS/FAIL | X.Xs | |
| T02 | Subflow Existence | PASS/FAIL | X.Xs | |
| T03 | Version Mismatch | PASS/FAIL | X.Xs | |
| T04 | Action Snapshots | PASS/FAIL | X.Xs | |
| T05 | Data Pill Schemas | PASS/FAIL | X.Xs | |
| T06 | Deprecated Actions | PASS/FAIL | X.Xs | |
| T07 | Cross-Version Compat | PASS/FAIL | X.Xs | |
| T08 | Full Migration E2E | PASS/FAIL | X.Xs | |
| T09 | Auto-Rollback | PASS/FAIL | X.Xs | |
| T10 | Invalid Flow ID | PASS/FAIL | X.Xs | |
| T11 | Missing Permissions | PASS/FAIL | X.Xs | |
| T12 | Cross-Version Incompat | PASS/FAIL | X.Xs | |
| T13 | Empty Environment | PASS/FAIL | X.Xs | |
| T14 | Large Flow (1000+) | PASS/FAIL | X.Xs | |
| T15 | Concurrent Access | PASS/FAIL | X.Xs | |

**Summary:** XX/15 PASS, XX/15 FAIL
**Blocking Issues:** [List any P0/P1 failures]
**Recommendation:** [Proceed / Do Not Proceed to deployment]
```

---

## Regression Test Triggers

The following events trigger a mandatory re-execution of the full test suite:

| Trigger | Tests Required | Rationale |
|---------|---------------|-----------|
| Any change to FlowGuardCrossEnvValidator.js | T01-T15 (full suite) | All validation checks could be affected |
| Any change to FlowGuardMigrator.js | T08-T12, T15 | Migration pipeline changes |
| ServiceNow platform upgrade (source or target) | T01-T07, T12 | API compatibility must be re-verified |
| New environment added to x_flowguard_environment | T01 (connectivity only) | Verify new env is reachable |
| ACL or role configuration change | T11 | Permission model must be re-verified |
| Flow Designer plugin update | T01-T06 | Table schemas may change |
| REST endpoint configuration change | T08-T12 | Endpoint behavior must be re-verified |
| Before any production deployment | T01-T15 (full suite) | Pre-release quality gate |
| Monthly scheduled regression | T01-T15 (full suite) | Ongoing quality assurance |

---

## Pass/Fail Decision Matrix

| Condition | Action |
|-----------|--------|
| All P0 tests (T01-T05, T08-T09) pass | ✅ Proceed to deployment |
| Any P0 test fails | ❌ BLOCK — must fix before deployment |
| All P1 tests (T06-T07, T11-T12) pass, P0 pass | ✅ Proceed with noted risks |
| >1 P1 test fails | ⚠️ Review — potentially block |
| P2 tests (T10, T13-T15) fail | ⚠️ Document as known limitations, proceed |
