# Edge Cases — ServiceNow FlowGuard v1

**Product:** ServiceNow FlowGuard
**Version:** 1.0.0
**Author:** Vladimir Kapustin
**License:** AGPL-3.0
**Date:** 2026-06-03

---

## Overview

This document catalogs edge cases and boundary conditions that FlowGuard must handle correctly. These cases test the system at its limits — empty states, extreme values, null/unset fields, and unusual data configurations.

---

## Edge Cases

### E01: Empty Flow List (Zero Flows in Source)

| Field | Value |
|-------|-------|
| **ID** | E01 |
| **Boundary Condition** | Source instance has zero flows in sys_hub_flow table |
| **System State** | sys_hub_flow table is empty. No flow_id can be provided. REST endpoint receives no valid flow_id. |
| **Expected Behavior** | /validate and /migrate endpoints return error: "Source flow not found" with `success: false`. /cross-validate returns error: "flow_id required". No null pointer exceptions. |
| **Rationale** | The system must handle the degenerate case of an empty flow table without crashing. Error messages should guide the user to create a flow first. |

### E02: Single-Action Flow

| Field | Value |
|-------|-------|
| **ID** | E02 |
| **Boundary Condition** | Flow contains exactly 1 action with no subflows, no data pill dependencies |
| **System State** | Flow model JSON: `{"actions": [{"name": "Log Message", ...}]}`. No subflow references, no inter-step data dependencies. |
| **Expected Behavior** | All 6 validation checks execute. Check 1 (subflow existence) finds zero subflows → `passed: true` with no issues. Check 2 (version) finds zero subflows → `passed: true`. Check 4 (data pill schemas) finds single action with no inputs/outputs → `passed: true`. Migration succeeds. |
| **Rationale** | The simplest possible flow must validate and migrate without false positives. Zero-subflow and zero-dependency cases should not be treated as errors. |

### E03: 10,000-Action Flow (Extreme Size)

| Field | Value |
|-------|-------|
| **ID** | E03 |
| **Boundary Condition** | Flow model JSON contains 10,000+ action objects, each with unique names, inputs, and outputs |
| **System State** | Flow model field is ~2 MB of JSON. GlideRecord may approach field size limits (~4 MB for string fields). JSON.parse() and signature map building must process large data. |
| **Expected Behavior** | Validation completes but may take 30-60 seconds. Memory usage is bounded — no exponential growth. Signature map (`_buildSignatureMap`) handles 10K keys without performance cliff. If field size limit exceeded, the flow is treated as unparseable (see R09). |
| **Rationale** | ServiceNow flows can grow very large in complex implementations. While 10K-action flows are rare, the system must not crash or hang. A graceful degradation (timeout or partial validation with warning) is acceptable. |

### E04: Null Environment Configuration

| Field | Value |
|-------|-------|
| **ID** | E04 |
| **Boundary Condition** | x_flowguard_environment records exist but have null values for username, password, or instance_url |
| **System State** | `envGr.getValue('username')` returns null. `envGr.getValue('password')` returns null or empty password2 object. `envGr.getValue('instance_url')` returns null. |
| **Expected Behavior** | Connectivity check handles null values: null URL → "Cannot reach" error. Null username/password → HTTP 401 (handled by R04). System does NOT crash on null getValue() calls. Password2 decryption guard (`typeof envPass === 'object'`) handles null/empty correctly. |
| **Rationale** | Database records can contain null values due to incomplete imports or manual SQL operations. Defensive coding must handle null returns from getValue(). |

### E05: Flow Deleted Mid-Migration (Race Condition)

| Field | Value |
|-------|-------|
| **ID** | E05 |
| **Boundary Condition** | Flow is deleted from source instance between validation start and migration payload serialization |
| **System State** | `FlowGuardMigrator.migrate()` calls `flowGr.get(sourceFlowId)` in Phase 2. Flow existed during Phase 0 (cross-env validation) and Phase 1 (pre-flight), but was deleted before Phase 2. |
| **Expected Behavior** | Phase 2 `flowGr.get()` returns false. Migration logs the error: "Source flow not found". Returns `success: false`. No snapshot is created (no flow to snapshot). No partial migration. |
| **Rationale** | Concurrent operations on the same flow are possible in multi-user environments. The system must detect the flow's disappearance and abort cleanly rather than proceeding with stale data. |

### E06: Special Characters in Flow Names

| Field | Value |
|-------|-------|
| **ID** | E06 |
| **Boundary Condition** | Flow name contains special characters: slashes, quotes, backticks, ampersands, angle brackets |
| **System State** | Flow name: `"Test & Verify / User's Flow <v2.0>"` — contains `&`, `/`, `'`, `<`, `>`. These characters appear in JSON payloads, REST URLs, and issue messages. |
| **Expected Behavior** | Flow name is stored and transmitted correctly. JSON serialization (JSON.stringify) properly escapes special characters. Issue messages display the name correctly without breaking JSON structure. REST endpoints handle URL-encoded characters. |
| **Rationale** | Flow Designer allows arbitrary names. Special characters that break JSON or REST must be handled. Escape sequences must be correct in both directions (serialize/deserialize). |

### E07: Unicode Flow Labels and Descriptions

| Field | Value |
|-------|-------|
| **ID** | E07 |
| **Boundary Condition** | Flow name, description, and action labels contain non-ASCII characters: CJK, Cyrillic, Arabic, emoji |
| **System State** | Flow name: `"サービス要求フロー 🌊"` (Japanese + emoji). Action names: `"Проверка доступности"` (Cyrillic). Descriptions: `"تدفق الموافقة"` (Arabic). |
| **Expected Behavior** | All unicode characters are preserved through serialization, REST transmission, and storage. JSON.parse/stringify handle full UTF-8. Issue messages display correct characters. No mojibake (garbled text). |
| **Rationale** | ServiceNow is a global platform. Non-ASCII content is common in multinational deployments. UTF-8 handling must be end-to-end correct. |

### E08: Environment with Duplicate Names

| Field | Value |
|-------|-------|
| **ID** | E08 |
| **Boundary Condition** | Two environment records have the same `name` value despite the unique index |
| **System State** | Unique index on `name` field enforced by database (sys_index unique=true). If somehow bypassed (e.g., direct SQL), two records share the same name. |
| **Expected Behavior** | Both environments are processed independently (order distinguishes them). Results array contains both entries. Environment names in results are not deduplicated. Migration log does not confuse the two environments. |
| **Rationale** | While the platform's unique index should prevent duplicates, defensive coding should handle the case where duplicates exist (e.g., from data imports that bypassed validation). The system should not assume name uniqueness for processing logic. |
