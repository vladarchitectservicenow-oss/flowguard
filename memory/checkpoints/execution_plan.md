# ServiceNow FlowGuard — Execution Plan

**Product:** ServiceNow FlowGuard
**Version:** 1.0.0
**License:** AGPL-3.0
**Author:** Vladimir Kapustin
**Date:** 2026-06-03

---

## Plan Overview

ServiceNow FlowGuard development follows a 6-phase waterfall-with-iterations lifecycle. Each phase produces specific, verifiable artifacts. This plan tracks all tasks, owners, status, and estimated effort.

**Total Estimated Effort:** 120-160 hours
**Target Completion:** Q3 2026
**Primary Developer:** Vladimir Kapustin

---

## Phase 0: Environment Setup & PDI Provisioning

**Duration:** 8-12 hours | **Status:** COMPLETE

| Task ID | Task | Owner | Status | Est. Hours | Dependency |
|---------|------|-------|--------|------------|------------|
| P0-01 | Provision ServiceNow PDI (Australia release) | Kapustin | ✅ Done | 2 | None |
| P0-02 | Provision second PDI for cross-environment testing | Kapustin | ✅ Done | 2 | None |
| P0-03 | Configure scoped application `x_flowguard` | Kapustin | ✅ Done | 1 | P0-01 |
| P0-04 | Set up GitHub repository with AGPL-3.0 license | Kapustin | ✅ Done | 1 | None |
| P0-05 | Create table schemas: x_flowguard_environment, x_flowguard_migration_log, x_flowguard_snapshot | Kapustin | ✅ Done | 2 | P0-03 |
| P0-06 | Configure ACLs and roles (x_flowguard_admin) | Kapustin | ✅ Done | 2 | P0-05 |
| P0-07 | Set up REST endpoint skeleton (flowguard_api) | Kapustin | ✅ Done | 1 | P0-03 |
| P0-08 | Verify cross-instance connectivity between PDIs | Kapustin | ✅ Done | 1 | P0-02 |

**Deliverables:** 2 configured PDIs, 3 application tables, 1 REST endpoint, ACL configuration, GitHub repo

---

## Phase 1: Core Engine Build

**Duration:** 40-50 hours | **Status:** COMPLETE

| Task ID | Task | Owner | Status | Est. Hours | Dependency |
|---------|------|-------|--------|------------|------------|
| P1-01 | Implement FlowGuardCrossEnvValidator — connectivity check (Check 0) | Kapustin | ✅ Done | 3 | P0-08 |
| P1-02 | Implement subflow existence check (Check 1) | Kapustin | ✅ Done | 5 | P1-01 |
| P1-03 | Implement subflow version mismatch check (Check 2) | Kapustin | ✅ Done | 6 | P1-02 |
| P1-04 | Implement action type snapshot check (Check 3) | Kapustin | ✅ Done | 8 | P1-01 |
| P1-05 | Implement data pill schema check (Check 4) | Kapustin | ✅ Done | 10 | P1-01 |
| P1-06 | Implement action deprecation check (Check 5) | Kapustin | ✅ Done | 5 | P1-01 |
| P1-07 | Implement helper methods (_getSubflowsFromSource, _getFlowActions, _buildSignatureMap, etc.) | Kapustin | ✅ Done | 6 | P1-01 |
| P1-08 | Implement FlowGuardMigrator.migrate() — full migration pipeline | Kapustin | ✅ Done | 8 | P1-03, P0-05 |
| P1-09 | Implement snapshot/rollback logic (_snapshot, _rollbackFromSnapshot) | Kapustin | ✅ Done | 4 | P0-05 |
| P1-10 | Implement flow serialization (_serializeFlow, _restorePayload, _createFromPayload) | Kapustin | ✅ Done | 3 | P1-08 |
| P1-11 | Implement post-deployment verification (_verify) | Kapustin | ✅ Done | 2 | P1-08 |
| P1-12 | Implement diff engine (diff, _deepEqual, _parseActions) | Kapustin | ✅ Done | 5 | P1-09 |
| P1-13 | Wire REST endpoints to Script Includes (validate, migrate, rollback, diff, cross-validate) | Kapustin | ✅ Done | 3 | P1-11, P0-07 |
| P1-14 | Add error handling, logging (gs.error, gs.info), and try/catch guards | Kapustin | ✅ Done | 4 | P1-08 |
| P1-15 | Unit test all 6 validation checks individually | Kapustin | ✅ Done | 6 | P1-07 |

**Deliverables:**
- `src/script_includes/FlowGuardCrossEnvValidator.js` (576 lines)
- `src/script_includes/FlowGuardMigrator.js` (328 lines)
- `src/rest_endpoints/flowguard_api.xml` (83 lines, 5 operations)

---

## Phase 2: REST API & Table Schema Finalization

**Duration:** 15-20 hours | **Status:** COMPLETE

| Task ID | Task | Owner | Status | Est. Hours | Dependency |
|---------|------|-------|--------|------------|------------|
| P2-01 | Finalize x_flowguard_environment table schema with all fields and indexes | Kapustin | ✅ Done | 2 | P0-05 |
| P2-02 | Add environment ordering logic (order field for priority processing) | Kapustin | ✅ Done | 1 | P2-01 |
| P2-03 | Finalize x_flowguard_migration_log table with status workflow | Kapustin | ✅ Done | 2 | P0-05 |
| P2-04 | Finalize x_flowguard_snapshot table for rollback support | Kapustin | ✅ Done | 1 | P0-05 |
| P2-05 | Implement flowguard_api POST /validate endpoint | Kapustin | ✅ Done | 2 | P1-13 |
| P2-06 | Implement flowguard_api POST /migrate endpoint | Kapustin | ✅ Done | 2 | P1-13 |
| P2-07 | Implement flowguard_api POST /rollback endpoint | Kapustin | ✅ Done | 1 | P1-13 |
| P2-08 | Implement flowguard_api GET /diff endpoint | Kapustin | ✅ Done | 1 | P1-13 |
| P2-09 | Implement flowguard_api POST /cross-validate endpoint | Kapustin | ✅ Done | 2 | P1-13 |
| P2-10 | Add request validation and error responses to all endpoints | Kapustin | ✅ Done | 2 | P2-05 |
| P2-11 | Configure ACLs for each REST endpoint | Kapustin | ✅ Done | 2 | P0-06 |
| P2-12 | Test all endpoints with Postman/curl | Kapustin | ✅ Done | 2 | P2-10 |

**Deliverables:**
- Table XML exports in `src/tables/`, REST endpoint XML in `src/rest_endpoints/`

---

## Phase 3: Testing

**Duration:** 25-35 hours | **Status:** IN PROGRESS

| Task ID | Task | Owner | Status | Est. Hours | Dependency |
|---------|------|-------|--------|------------|------------|
| P3-01 | Write test suite SOP with 15 scenarios (T01-T15) | Kapustin | ✅ Done | 6 | P1-15 |
| P3-02 | Write regression test cases (8 cases, R01-R10) | Kapustin | ✅ Done | 4 | P1-15 |
| P3-03 | Write edge case documentation (6+ cases, E01-E08) | Kapustin | ✅ Done | 3 | P1-15 |
| P3-04 | Write validation checklist (60+ items across all quality gates) | Kapustin | ✅ Done | 4 | P3-01 |
| P3-05 | T01: Connectivity validation — positive case | Kapustin | ⬜ Pending | 2 | P3-01 |
| P3-06 | T02: Subflow existence validation | Kapustin | ⬜ Pending | 2 | P3-01 |
| P3-07 | T03: Version mismatch detection | Kapustin | ⬜ Pending | 2 | P3-01 |
| P3-08 | T04: Action snapshot compatibility | Kapustin | ⬜ Pending | 2 | P3-01 |
| P3-09 | T05: Data pill schema validation | Kapustin | ⬜ Pending | 2 | P3-01 |
| P3-10 | T06: Deprecated action detection | Kapustin | ⬜ Pending | 2 | P3-01 |
| P3-11 | T07: Cross-version compatibility (Utah→Australia) | Kapustin | ⬜ Pending | 3 | P3-01 |
| P3-12 | T08: Full migration pipeline (end-to-end) | Kapustin | ⬜ Pending | 3 | P3-01 |
| P3-13 | T09: Auto-rollback on verification failure | Kapustin | ⬜ Pending | 2 | P3-02 |
| P3-14 | T10: Diff computation accuracy | Kapustin | ⬜ Pending | 2 | P3-02 |
| P3-15 | T11: All negative/error cases from test suite | Kapustin | ⬜ Pending | 3 | P3-01 |
| P3-16 | Execute regression suite (R01-R10) on final build | Kapustin | ⬜ Pending | 4 | P3-02 |
| P3-17 | Execute edge case suite (E01-E08) | Kapustin | ⬜ Pending | 2 | P3-03 |
| P3-18 | Performance benchmark: 5-env cross-validation timing | Kapustin | ⬜ Pending | 2 | P2-12 |
| P3-19 | Security review: ACL verification, credential exposure | Kapustin | ⬜ Pending | 3 | P2-11 |

**Deliverables:**
- `Validation/TEST CASES/flowguard_v1/test_suite_SOP.md`
- `Validation/TEST CASES/flowguard_v1/regression_cases.md`
- `Validation/TEST CASES/flowguard_v1/edge_cases.md`
- `Validation/TEST CASES/flowguard_v1/validation_checklist.md`

---

## Phase 4: Documentation & Marketing

**Duration:** 15-20 hours | **Status:** IN PROGRESS

| Task ID | Task | Owner | Status | Est. Hours | Dependency |
|---------|------|-------|--------|------------|------------|
| P4-01 | Write architecture summary (80+ lines) | Kapustin | ✅ Done | 4 | P1-15 |
| P4-02 | Write dependency report (80+ lines, compat matrix) | Kapustin | ✅ Done | 3 | P1-15 |
| P4-03 | Write risk report (15 risks, mitigation table) | Kapustin | ✅ Done | 5 | P1-15 |
| P4-04 | Write execution plan (this document, 6+ phases) | Kapustin | ✅ Done | 3 | P4-01 |
| P4-05 | README deduplication (37→12-18 sections) | Kapustin | ⬜ Pending | 2 | P4-01 |
| P4-06 | README expansion with product-specific content (2000+ words) | Kapustin | ⬜ Pending | 4 | P4-05 |
| P4-07 | Add Mermaid architecture diagram to README | Kapustin | ⬜ Pending | 1 | P4-06 |
| P4-08 | Add environment compatibility matrix to README | Kapustin | ⬜ Pending | 1 | P4-02 |
| P4-09 | Add ROI analysis to README | Kapustin | ⬜ Pending | 1 | P4-06 |
| P4-10 | Add troubleshooting section to README | Kapustin | ⬜ Pending | 2 | P4-06 |
| P4-11 | Add version history to README | Kapustin | ⬜ Pending | 1 | P4-06 |
| P4-12 | Add copyright headers to all source files (G3 gate) | Kapustin | ⬜ Pending | 1 | P1-15 |

**Deliverables:**
- 4 memory/checkpoints docs (this phase)
- Complete README.md (2000+ words)
- Copyright headers on all .js and .xml files

---

## Phase 5: Deployment & Validation

**Duration:** 10-15 hours | **Status:** PENDING

| Task ID | Task | Owner | Status | Est. Hours | Dependency |
|---------|------|-------|--------|------------|------------|
| P5-01 | Export application from development PDI as XML update set | Kapustin | ⬜ Pending | 1 | P3-18 |
| P5-02 | Import and test on fresh PDI (clean install validation) | Kapustin | ⬜ Pending | 2 | P5-01 |
| P5-03 | Cross-environment validation test: Dev→QA→Prod simulation | Kapustin | ⬜ Pending | 3 | P5-02 |
| P5-04 | Git commit all artifacts with semantic version tag | Kapustin | ⬜ Pending | 1 | P4-12 |
| P5-05 | Push to GitHub (vladarchitectservicenow-oss/servicenow-flowguard) | Kapustin | ⬜ Pending | 1 | P5-04 |
| P5-06 | Verify GitHub repository: LICENSE, README rendering, file structure | Kapustin | ⬜ Pending | 1 | P5-05 |
| P5-07 | Run validation checklist (60+ items) against final build | Kapustin | ⬜ Pending | 2 | P5-05 |
| P5-08 | Create DONE.marker with timestamp | Kapustin | ⬜ Pending | 0.5 | P5-07 |
| P5-09 | Update pipeline progress (servicenow-flowguard → done) | Kapustin | ⬜ Pending | 0.5 | P5-08 |
| P5-10 | Post-deployment smoke test on GitHub (clone, review, verify) | Kapustin | ⬜ Pending | 1 | P5-05 |

**Deliverables:**
- Git tag v1.0.0
- GitHub repository with complete artifact set
- DONE.marker
- Updated pipeline_progress.json

---

## Phase 6: Post-Release (v1.1 Roadmap)

**Duration:** Ongoing | **Status:** PLANNED

| Task ID | Task | Owner | Status | Est. Hours | Dependency |
|---------|------|-------|--------|------------|------------|
| P6-01 | AI compatibility scoring integration (Now Assist hooks) | Kapustin | ⬜ Planned | 20 | P5-10 |
| P6-02 | Flow dependency graph visualization | Kapustin | ⬜ Planned | 15 | P5-10 |
| P6-03 | Multi-instance federation dashboard | Kapustin | ⬜ Planned | 25 | P6-01 |
| P6-04 | OAuth 2.0 support for cross-instance auth | Kapustin | ⬜ Planned | 10 | P5-10 |
| P6-05 | Scheduled cross-environment health checks (Scheduled Job) | Kapustin | ⬜ Planned | 8 | P5-10 |
| P6-06 | ServiceNow Store publication | Kapustin | ⬜ Planned | 15 | P5-10 |
| P6-07 | Community contribution guidelines and templates | Kapustin | ⬜ Planned | 5 | P5-10 |
| P6-08 | Automated ATF test suite for CI/CD integration | Kapustin | ⬜ Planned | 20 | P5-10 |

---

## Milestone Summary

| Phase | Milestone | Target Date | Status | Key Deliverable |
|-------|-----------|-------------|--------|----------------|
| P0 | Environment Setup | 2026-05-15 | ✅ COMPLETE | 2 PDIs, 3 tables, REST skeleton |
| P1 | Core Engine | 2026-05-30 | ✅ COMPLETE | FlowGuardCrossEnvValidator.js, FlowGuardMigrator.js |
| P2 | API & Schema | 2026-06-01 | ✅ COMPLETE | flowguard_api REST endpoint (5 operations) |
| P3 | Testing | 2026-06-10 | 🔄 IN PROGRESS | Test suite, regression, edge cases, checklist |
| P4 | Documentation | 2026-06-03 | 🔄 IN PROGRESS | Architecture, deps, risks, README |
| P5 | Deployment | 2026-06-03 | ⬜ PENDING | Git push, DONE.marker |
| P6 | v1.1 Planning | Q3 2026 | 📋 PLANNED | AI scoring, dashboard, OAuth |

---

## Risk Register (Execution Risks)

| Risk | Impact | Mitigation |
|------|--------|------------|
| PDI expiration before testing complete | Blocks P3 testing | Request PDI extension; use multiple PDIs |
| API changes in Australia release | Validation checks need update | Monitor ServiceNow release notes; compat matrix |
| Cross-instance network restrictions | Validation fails | VPN configuration; on-premise instance option |
| Scope conflicts with other apps | Installation failures | Unique scope prefix `x_flowguard`; scoped app isolation |
