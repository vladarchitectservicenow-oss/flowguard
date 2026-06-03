# Validation Checklist — ServiceNow FlowGuard v1

**Product:** ServiceNow FlowGuard
**Version:** 1.0.0
**Author:** Vladimir Kapustin
**License:** AGPL-3.0
**Date:** 2026-06-03

---

## Overview

This checklist covers all quality gates (G0-G8 from the ServiceNow Scoped App Factory) plus additional functional, security, documentation, and deployment checks. Every item must be verified before a release is considered production-ready.

Legend:
- **D** = Documentation
- **T** = Testing
- **R** = Review/Code Quality
- **L** = Licensing/Legal
- **S** = Security
- **G** = Git/GitHub
- **F** = Functional/Platform

---

## G0: Test Suite SOP (≥10 Scenarios)

- [ ] G0-01: test_suite_SOP.md exists with ≥10 TXX scenarios
- [ ] G0-02: At least 2 negative/error-path scenarios included
- [ ] G0-03: At least 1 stress/boundary scenario (large flow, concurrent access)
- [ ] G0-04: Test environment prerequisites documented
- [ ] G0-05: Execution order with dependencies clearly specified
- [ ] G0-06: Pass/fail criteria defined for every scenario
- [ ] G0-07: Test data requirements table present
- [ ] G0-08: Test execution log template provided
- [ ] G0-09: Regression triggers table present
- [ ] G0-10: Pass/fail decision matrix defined

## G1: Test Execution History

- [ ] G1-01: All test scenarios T01-T15 documented with expected results
- [ ] G1-02: Regression cases R01-R10 documented
- [ ] G1-03: Edge cases E01-E08 documented
- [ ] G1-04: Validation checklist (this document) completed
- [ ] G1-05: Test execution log directory (tests/execution_history/) exists
- [ ] G1-06: At least one execution log file present with PASS/FAIL results
- [ ] G1-07: All P0 tests (T01-T05, T08-T09) passing in latest log
- [ ] G1-08: No blocking P0 failures in latest execution

## G2: README Quality (≥2000 Words)

- [ ] G2-01: README.md ≥2000 words
- [ ] G2-02: Contains Mermaid architecture diagram(s)
- [ ] G2-03: Contains ROI analysis with quantifiable metrics
- [ ] G2-04: Contains Troubleshooting section with common issues + solutions
- [ ] G2-05: Contains Environment Compatibility Matrix
- [ ] G2-06: Contains Installation/Configuration guide
- [ ] G2-07: Contains Version History
- [ ] G2-08: Contains Usage Examples
- [ ] G2-09: No duplicate `## ` sections (12-18 total)
- [ ] G2-10: License badge matches LICENSE file (AGPL-3.0)

## G3: Copyright Headers

- [ ] G3-01: FlowGuardCrossEnvValidator.js has AGPL-3.0 copyright header with "Vladimir Kapustin"
- [ ] G3-02: FlowGuardMigrator.js has AGPL-3.0 copyright header with "Vladimir Kapustin"
- [ ] G3-03: All JS headers use `//` line-comment format, NOT `/** */` JSDoc blocks
- [ ] G3-04: `Copyright (C)` uppercase in all headers
- [ ] G3-05: SPDX-License-Identifier on its own line
- [ ] G3-06: Product name "ServiceNow FlowGuard" not abbreviated in headers
- [ ] G3-07: XML files (flowguard_api.xml, x_flowguard_environment.xml) have XML-style headers
- [ ] G3-08: No XML files using JS-style `//` comments for copyright

## G4: Git Push Verified

- [ ] G4-01: Repository pushed to GitHub (vladarchitectservicenow-oss/servicenow-flowguard)
- [ ] G4-02: Remote branch `main` exists
- [ ] G4-03: Latest commit is the current build
- [ ] G4-04: `git push` verified via GitHub API (HTTP 200 on branches endpoint)
- [ ] G4-05: No unpushed commits remaining locally
- [ ] G4-06: DONE.marker exists in repository root

## G5: No Hardcoded Credentials

- [ ] G5-01: No literal passwords in source code (`DEFAULT_PASS=`, `password='...'`)
- [ ] G5-02: No inline API tokens or bearer tokens
- [ ] G5-03: All credentials use `process.env` or ServiceNow password2 fields
- [ ] G5-04: `envPass` guard handles null/empty password2 objects
- [ ] G5-05: REST endpoint authentication uses instance credentials from x_flowguard_environment table
- [ ] G5-06: No credentials in README or documentation examples (use placeholders like `YOUR_PASSWORD`)

## G6: .gitignore

- [ ] G6-01: `.gitignore` file exists at repository root
- [ ] G6-02: Excludes `__pycache__/`
- [ ] G6-03: Excludes `*.pyc`
- [ ] G6-04: Excludes `reports/`
- [ ] G6-05: Excludes `.env` files
- [ ] G6-06: Excludes `node_modules/`
- [ ] G6-07: Excludes test execution logs if they contain instance URLs

## G7: README License Consistency

- [ ] G7-01: README license badge says AGPL-3.0
- [ ] G7-02: Root LICENSE file is full AGPL-3.0 text (675 lines)
- [ ] G7-03: No SPDX-only placeholder in LICENSE (full text required)
- [ ] G7-04: Copyright line in LICENSE footer matches: "Copyright (C) 2026 Vladimir Kapustin"

## G8: No Duplicate README Sections

- [ ] G8-01: `grep -c '^## ' README.md` returns 12-18
- [ ] G8-02: No `## ` heading appears more than once
- [ ] G8-03: All section content is unique (not copy-pasted)
- [ ] G8-04: README has logical flow: Overview → Architecture → Installation → Usage → Troubleshooting → ROI

---

## Documentation Gates (D)

- [ ] D-01: architecture_summary.md ≥80 lines with component table, data flow, performance benchmarks
- [ ] D-02: dependency_report.md ≥80 lines with table names, plugin IDs, role lists, version matrix
- [ ] D-03: risk_report.md ≥10 risk entries (R01-R15 format) with severity tags P0-P3
- [ ] D-04: execution_plan.md ≥6 phases with task/owner/status/ETA tables
- [ ] D-05: PHASE_1_BUILD_PLAN.md present and up to date
- [ ] D-06: PHASE_2_BUILD_REPORT.md present and up to date
- [ ] D-07: WHITEPAPER.md exists in marketing/ or docs/
- [ ] D-08: LINKEDIN_POST.md exists in marketing/ or docs/
- [ ] D-09: All documentation files have AGPL-3.0 copyright headers
- [ ] D-10: No placeholder or TODO text in any documentation file

## Testing Gates (T)

- [ ] T-01: Unit tests cover FlowGuardCrossEnvValidator.validateAllEnvironments()
- [ ] T-02: Unit tests cover FlowGuardMigrator.migrate() full pipeline
- [ ] T-03: Unit tests cover FlowGuardMigrator._captureSnapshot()
- [ ] T-04: Unit tests cover FlowGuardMigrator._restoreFromSnapshot()
- [ ] T-05: Unit tests cover FlowGuardMigrator._buildSignatureMap()
- [ ] T-06: Unit tests cover FlowGuardMigrator._deepEqual()
- [ ] T-07: Unit tests cover FlowGuardMigrator._verify()
- [ ] T-08: Mock GlideRecord covers create, query, next, getValue, getRowCount, update
- [ ] T-09: Mock GlideDateTime covers date comparisons
- [ ] T-10: Mock sn_ws.RESTMessageV2 covers HTTP GET/POST with timeout
- [ ] T-11: Mock gs.info/error/log for logging verification
- [ ] T-12: All mock implementations are self-contained (no external ServiceNow dependencies)
- [ ] T-13: Test files exist in tests/ directory
- [ ] T-14: npm test script defined in package.json
- [ ] T-15: All tests pass with `npm test`

## Review/Code Quality Gates (R)

- [ ] R-01: No `getFields()` where `getElements()` is required for scoped APIs
- [ ] R-02: No non-deterministic checksums in validation logic
- [ ] R-03: No double faults in REST catch blocks
- [ ] R-04: No shared instance state leakage between validation runs
- [ ] R-05: No `answer=true` ACL overrides without justification
- [ ] R-06: REST endpoints have ACLs restricting access to x_flowguard_admin
- [ ] R-07: GlideDuration compatibility verified (no direct string comparison of durations)
- [ ] R-08: Export error handling: stuck-on-export edge case handled
- [ ] R-09: Log-level appropriate: gs.info for operational, gs.error for failures, gs.debug for verbose
- [ ] R-10: Before-insert business rules have gap detection
- [ ] R-11: No dead config properties (sys_properties referenced but not defined)
- [ ] R-12: All `try` blocks have meaningful `catch` blocks (not empty/silent)
- [ ] R-13: No `eval()` used anywhere in source code

## Licensing Gates (L)

- [ ] L-01: Root LICENSE file is full AGPL-3.0 text (not SPDX tag only)
- [ ] L-02: Every source file (.js, .xml) has AGPL-3.0 copyright header
- [ ] L-03: Copyright holder is "Vladimir Kapustin" (full name, not abbreviated)
- [ ] L-04: `(C)` is uppercase in all copyright headers
- [ ] L-05: SPDX-License-Identifier: AGPL-3.0 on its own line after copyright
- [ ] L-06: README license badge matches LICENSE file content

## Security Gates (S)

- [ ] S-01: Password2 field used for environment credentials (not string fields)
- [ ] S-02: REST calls use HTTPS only (no HTTP fallback)
- [ ] S-03: Environment validation rejects self-referencing URLs (same instance)
- [ ] S-04: Migration log does not contain plaintext passwords
- [ ] S-05: REST endpoint authentication required (no public endpoints)
- [ ] S-06: Cross-instance requests use instance-scoped credentials
- [ ] S-07: No CORS misconfiguration (REST endpoints are instance-to-instance, not browser)
- [ ] S-08: OAuth2 scoped tokens used where available (not basic auth)

## Git/GitHub Gates (G)

- [ ] G-01: Repository name follows convention: `servicenow-[product]`
- [ ] G-02: Repository description contains full product name
- [ ] G-03: Repository is public
- [ ] G-04: `main` is the default branch
- [ ] G-05: Conventional commit messages used (feat:, fix:, docs:, chore:)
- [ ] G-06: No merge commits without meaningful merge messages
- [ ] G-07: Tags exist for versioned releases (v1.0.0)
- [ ] G-08: README is the primary documentation entry point

## Functional/Platform Gates (F)

- [ ] F-01: Scoped application name follows convention: x_flowguard
- [ ] F-02: Scope definition (sys_app.xml) includes correct vendor and version
- [ ] F-03: Table x_flowguard_environment includes fields: name, instance_url, username, password, active, order
- [ ] F-04: Script Include FlowGuardCrossEnvValidator handles all 6 validation checks
- [ ] F-05: Script Include FlowGuardMigrator implements full pipeline: validate → snapshot → deploy → verify → rollback
- [ ] F-06: REST endpoint supports: cross-validate, migrate, rollback, diff, status
- [ ] F-07: Flow Designer actions are not broken by migration
- [ ] F-08: Subflow references are preserved through migration
- [ ] F-09: Data pill mappings are preserved through migration
- [ ] F-10: Compatible with ServiceNow Utah release through Australia release
- [ ] F-11: Does not require scoped plugins unavailable in baseline ServiceNow
- [ ] F-12: Works on both PDI and production instances

---

## Pre-Release Verification

- [ ] PRE-01: All G0-G8 gates passed
- [ ] PRE-02: All D-xx documentation gates passed
- [ ] PRE-03: All T-xx testing gates passed
- [ ] PRE-04: All S-xx security gates passed
- [ ] PRE-05: PDI smoke test completed (if PDI available)
- [ ] PRE-06: DONE.marker committed and pushed
- [ ] PRE-07: GitHub repository verified accessible and public
- [ ] PRE-08: All linked files in README resolve correctly
- [ ] PRE-09: No broken internal links in documentation
- [ ] PRE-10: Version number consistent across all files

---

## Summary

| Gate Category | Total Items | Verified | Status |
|--------------|-------------|----------|--------|
| G0: Test Suite SOP | 10 | — | — |
| G1: Test Execution | 8 | — | — |
| G2: README Quality | 10 | — | — |
| G3: Copyright Headers | 8 | — | — |
| G4: Git Push | 6 | — | — |
| G5: No Credentials | 6 | — | — |
| G6: .gitignore | 7 | — | — |
| G7: README License | 4 | — | — |
| G8: No Duplicates | 4 | — | — |
| D: Documentation | 10 | — | — |
| T: Testing Gates | 15 | — | — |
| R: Code Review | 13 | — | — |
| L: Licensing | 6 | — | — |
| S: Security | 8 | — | — |
| G: Git/GitHub | 8 | — | — |
| F: Functional | 12 | — | — |
| PRE: Pre-Release | 10 | — | — |
| **TOTAL** | **145** | — | — |

*Complete this checklist before declaring the release ready. Mark each item as [x] when verified.*
