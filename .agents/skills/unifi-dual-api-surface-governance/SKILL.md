---
name: myco:unifi-dual-api-surface-governance
description: |
  Apply this skill whenever you are adding, extending, or debugging an MCP tool that
  touches a UniFi feature served by more than one API surface — specifically when a
  feature appears on both the Protect private API (/proxy/protect/api/) and the
  UniFi OS-level v2 service (/api/v2/). Covers: surface mapping, version-transparent
  façade with automatic backend selection, alarm rule mutation routing by ID family
  (UUID to v2 service, Mongo ObjectID to legacy), vocabulary translation between
  legacy (name/conditions) and v2 (title/triggers), SuperAdmin permission for /api/v2/,
  _meta coverage signals, empty-body fixes via api_request_raw, the
  require_non_empty_actions guard, and capture-first for unknown mutation endpoints.
  Apply even if not explicitly asked about API versioning — any time you're touching
  alarm rules, arm/disarm profiles, AlarmRulesFacade mutations, or any Protect domain
  with both a /proxy/protect/api/ and /api/v2/ surface.
managed_by: myco
user-invocable: true
allowed-tools: Read, Edit, Write, Bash, Grep, Glob
---

# UniFi Dual API Surface Governance

UniFi Protect features can be served by two (or three) completely separate API
services: the Protect private API at `/proxy/protect/api/`, and the UniFi OS-level
service at `/api/v2/`. These surfaces are independent — different auth requirements,
different ID schemes, different data models, and different coverage. Failing to bridge
them correctly causes agents to see silent data gaps with no signal that anything is
missing. This skill teaches you how to discover surface boundaries, abstract them
behind a stable façade, handle auth and protocol quirks, signal coverage gaps
explicitly, and safely implement mutations across surfaces.

## Prerequisites

Before working in this domain:

1. **Identify all API surfaces.** Open the Protect UI in a browser, navigate to the
   feature area (e.g., Alarm Manager, arm/disarm profiles), and capture network
   traffic. UI network calls reveal the actual endpoint paths.

2. **Understand the three known surfaces for Alarm-related features:**

   | Surface | Base Path | Contents | Auth |
   |---|---|---|---|
   | Protect private API | `/proxy/protect/api/automations` | Legacy rules (Mongo ObjectIDs) | Standard admin |
   | UniFi OS v2 | `/api/v2/alarms/` | Legacy + AI-NL alarms, UUID IDs | **SuperAdmin** |
   | UniFi OS v2 (Protect) | `/api/v2/alarms/protect` | Full ruleset for Protect | **SuperAdmin** |
   | UniFi OS v2 (profiles) | `/api/v2/alarms/profiles` | Arm/disarm profiles | **SuperAdmin** |

3. **Account permissions.** The `/api/v2/*` service requires **SuperAdmin** role.
   A 403 from `/api/v2/` means permissions, not routing — a routing failure produces a 404.

4. **Routing v2 calls.** `uiprotect.api_request(api_path="/api/v2/alarms/", url="...")`
   routes to the OS-level service through the existing client.

5. **Capture before Phase 3 mutations.** v2 create/update/delete request shapes are
   unknown until captured from a live console — do not implement against assumed shapes
   (see Procedure I).

## Procedure A: Mapping a New Dual-Surface Feature

When you encounter a Protect UI feature that doesn't match the data your code returns,
suspect a dual-surface gap.

1. **Live comparison test.** Fetch from both surfaces and compare record counts and ID
   formats. If v2 count > legacy count, v2 has additional data types the legacy API
   cannot represent (e.g., AI natural-language alarms).

2. **Check ID schemes.** Legacy returns Mongo ObjectIDs (24-char hex); v2 returns
   UUIDs (8-4-4-4-12). These are different representations — not duplicates.

3. **Determine subset relationship.** For alarm rules: legacy is a strict subset of v2
   (52/52 legacy rules appear in v2, plus AI-NL alarms exclusive to v2). Document the
   relationship — it determines which surface should be preferred and which is the
   fallback.

4. **Flag silent gaps.** If the existing tool returns from legacy only and v2 has
   records legacy cannot represent, agents see a partial picture with no signal.
   Before PR #320, `protect_alarm_list_rules` silently omitted all AI-NL alarms.

## Procedure B: Implementing a Version-Transparent Façade

When a feature lives on multiple incompatible API surfaces, the tool name must describe
capability, not implementation. Abstract the version in the backend, not the tool name.

**Do NOT** create `protect_alarm_v2_list_rules` alongside `protect_alarm_list_rules` —
this forces agents to reason about API versions and creates parallel tool families.

**Do** create a façade class with automatic backend selection. The real implementation
is in `packages/unifi-core/src/unifi_core/protect/managers/alarm_facade.py`:

```python
class AlarmRulesFacade:
    def __init__(self, service_manager: AlarmManagerService, legacy_manager: AlarmManager):
        self._service = service_manager    # /api/v2/alarms/ via AlarmManagerService
        self._legacy = legacy_manager      # /proxy/protect/api/automations via AlarmManager

    async def list_rules(self) -> tuple[list[dict], bool]:
        try:
            rules = await self._service.list_rules()
        except (AlarmManagerPermissionError, BadRequest):
            rules = None
        if rules:
            return rules, True          # v2 backend: complete coverage
        raw = await self._legacy.list_rules()
        return [alarm_rule_from_legacy(r).model_dump(exclude_none=True) for r in raw], False
```

> **"200 OK + empty list" must also trigger fallback (PR #334):** `if rules:` evaluates
> False for both `None` (error case) and `[]` (unmigrated console). Three-state coverage:
> - SuperAdmin + migrated console: v2 returns rules → serve v2 (`complete=True`)
> - SuperAdmin + unmigrated console: v2 returns `200 []` → fall back to legacy
> - Non-SuperAdmin: v2 returns `403` → fall back to legacy

**Canonical model** lives in
`packages/unifi-core/src/unifi_core/protect/models/alarm_rules.py`. Use
`model.model_dump(exclude_none=True)` when serializing legacy-sourced records so
callers see a stable field contract regardless of backend.

**Internal naming discipline.** Remove `v2` from all internal identifiers
(`AlarmV2Manager` → `AlarmManagerService`). The literal `/api/v2/alarms/` path in
routing code is preserved — it's a real URL, not a naming choice.

**TDD sequence:** canonical model tests → façade routing tests → fallback tests
(mock v2 to raise `AlarmManagerPermissionError` or return `[]`) → tool wiring tests
→ manifest regen → live smoke at both permission levels.

## Procedure C: Fixing Empty-Body Response Bugs

Protect mutation endpoints (DELETE, some POST) return an **empty body** on success.
`uiprotect`'s `api_request()` unconditionally calls `response.json()`, which throws
`Could not decode JSON` on `b""`. The operation succeeds; the error is spurious.

**Fix:** Replace `api_request` with `api_request_raw` when the response body is discarded:

```python
# Broken: empty response → JSON decode error
await self._api.api_request("delete", url, raise_exception=True)
# Fixed: handles empty bytes cleanly
await self._api.api_request_raw("delete", url, raise_exception=True)
```

**Known affected methods (all fixed), in `packages/unifi-core/src/unifi_core/protect/managers/recognition_manager.py`:**
- `apply_delete_known_face` — DELETE `/recognition/face/groups/{id}` — PR #319
- `apply_delete_known_vehicle` — DELETE vehicle group — PR #316
- `apply_merge_known_faces` — POST `/recognition/face/groups/merge` — PR #321

**Audit rule:** Any new `method="delete"` or discarded-response POST in a Protect
manager must use `api_request_raw`. Probe with `api_request_raw` first if uncertain.

## Procedure D: Emitting `_meta` Coverage Signals

When a tool serves from a degraded backend, signal this explicitly rather than silently
omitting records. Use the project's `com.github.sirkirby.unifi-mcp/` reverse-DNS
namespace. Constants defined in `apps/protect/src/unifi_protect_mcp/tools/alarm.py`:

```python
_ALARM_COVERAGE_META = "com.github.sirkirby.unifi-mcp/alarm-coverage"
```

Merge `_meta` into the serialized response when `complete=False` from the façade:

```python
def _with_alarm_coverage_meta(result: dict, complete: bool) -> dict:
    if not complete:
        result["_meta"] = {_ALARM_COVERAGE_META: {"complete": False, "reason": _ALARM_COVERAGE_NOTICE}}
    return result
```

SuperAdmin v2 responses omit `_meta` entirely. Only emit when coverage is genuinely
incomplete.

## Procedure E: SuperAdmin Elevation and Graceful Degradation

**Elevation steps:**
1. Log in to UniFi OS console as an account with existing SuperAdmin access
2. Navigate to Users → find the MCP service account (e.g., `homebridge`)
3. Change role to **SuperAdmin**; log out and back in to mint a fresh session token
4. Re-test: `GET /api/v2/alarms/profiles` should now return 200

**Security implication:** SuperAdmin has full blast-radius across the entire UniFi OS
console. Use a dedicated SuperAdmin service account for alarm-v2 operations.

**Graceful degradation is non-negotiable.** The façade's `AlarmManagerPermissionError`
and `BadRequest` catch ensures the MCP server remains fully functional with legacy alarm
coverage even without SuperAdmin. Never make SuperAdmin a hard requirement.

**Distinguishing 403 from 404:** `403` = permission boundary (account needs elevation);
`404` = routing failure (wrong path or service unavailable).

## Procedure F: Mutation Routing by ID Family

All CRUD mutations flow through `AlarmRulesFacade`. Routing is deterministic at call
time based on the ID format the agent passes back from a prior `list`/`get`.

| ID format | Example | Backend |
|---|---|---|
| UUID (36-char) | `019e89f0-8fc0-7141-9a64-7eb3f5746148` | v2 UniFi OS service (`/api/v2/alarms/`) |
| 24-hex Mongo ObjectID | `674f3052cdbf2c191e0a01b7` | Legacy Protect automations |

**Implementation** in `packages/unifi-core/src/unifi_core/protect/managers/alarm_facade.py`
(PR #366 — `_OBJECT_ID_RE` removed; UUID detection is now the sole routing criterion):

```python
_UUID_RE = re.compile(
    r"^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$"
)

@staticmethod
def _id_family(rule_id: str) -> str:
    """Route by id: v2 UUIDs go to the Alarm Manager, everything else to legacy."""
    if not isinstance(rule_id, str) or not rule_id.strip():
        raise ValueError("Alarm rule id must be a non-empty string")
    return "v2" if _UUID_RE.match(rule_id) else "legacy"
```

An agent cannot invent an ID — it received it from a prior tool call, so the ID encodes
which backend owns it. No read-before-write probe needed for update or delete.

**Create** requires an explicit read probe: check v2 reachability via the same selection
logic as `list_rules`; if SuperAdmin + v2 available → create via v2, else → legacy.

**Triage tell:** `UniFiNotFoundError` on mutation + `complete=True` on list = a v2 UUID
was passed to the legacy backend — facade mutation routing is not wired correctly.

## Procedure G: Write Schema Vocabulary Translation

The v2 and legacy backends use **fundamentally different field names** for mutations.
`rule_to_controller` (alarms.py:481) only performs snake→camel renames within the
legacy vocabulary — it is **NOT** a cross-backend serializer.

| Concept | Legacy field | v2 field |
|---|---|---|
| Rule name | `name` | `title` |
| Enabled flag | `enable` | `enabled` |
| Trigger events | `conditions` | `triggers` |
| Source list | `sources` | nested inside `scope` |

**Fix:** Implement a dedicated `alarm_rule_to_legacy_body` (genuine inverse of
`alarm_rule_from_legacy`) in `packages/unifi-core/src/unifi_core/protect/models/alarm_rules.py`.
Use canonical write shape (`title`/`enabled`/`triggers`) at the facade surface, then
translate per backend path.

**Echo `data` verbatim** in the inverse — `alarm_rule_from_legacy` stores entire legacy
dicts in `data`; the inverse must echo them, not reconstruct nested structures.

**Fetch-merge-PUT pattern for updates:** Fetch raw body from owning backend → merge
caller's changes → PUT merged body back. The facade owns this operation entirely.

> **Gotcha:** Hand-crafted test fixtures can accidentally round-trip because they lack
> the nested legacy structure. Only real captured fixtures catch structural mismatches.

## Procedure H: require_non_empty_actions Mutation Guard

**Live-tested gotcha (2026-05-27):** Creating an alarm rule with an empty `actions`
list via the API **breaks the Protect UI** — the UI becomes non-functional and the
corrupted rule can only be deleted via the API.

`require_non_empty_actions` is defined in
`packages/unifi-core/src/unifi_core/protect/models/_validators.py` and applied in
`packages/unifi-core/src/unifi_core/protect/managers/alarm_facade.py`. Apply in the
facade before any create or actions-touching update.

| Operation | Apply guard? |
|---|---|
| Create | Always |
| Update (partial, `actions` in change set) | Yes |
| Update (partial, `actions` NOT in change set) | No |

When refactoring input models, explicitly verify the guard is still applied — it must
travel with the facade's canonical path, not with any specific input model class.

## Procedure I: Phase-Based Delivery and Capture-First Discipline

**Deliver read-only capabilities before mutations.** Read paths have fewer safety
invariants and provide the live surface knowledge needed to implement mutations safely.

**Capture-first for unknown mutation endpoints:**
1. Do NOT implement against assumed shapes — fabricated fixture tests mask bugs
2. Open a capture session against a real console, record the raw HTTP exchange
3. Build the serializer and unit tests from the **real** captured fixture
4. Only then implement the mutation method

**Phased delivery checklist:**
1. ✅ Map both surfaces (paths, auth, ID formats, data overlap)
2. ✅ Implement facade with read operations and backend selection logic
3. ✅ Add `_meta` coverage signal for legacy fallback
4. ✅ Verify fallback triggers on both `403` **and** `200 []` cases
5. ✅ Capture raw HTTP exchange for all mutation endpoints before implementing
6. ✅ Implement write schema translators tested against real captured fixtures
7. ✅ Add `require_non_empty_actions` guard in facade before routing
8. ✅ Extend facade to own all mutations: update/delete by ID family; create by read probe

## Cross-Cutting Gotchas

**Silent gaps are worse than explicit errors.** Always emit `_meta` when coverage is
partial — never let incomplete results look complete.

**Never let v2 appear in tool names.** The façade pattern exists to prevent version
leakage into the MCP tool surface. Internal code may reference v2 paths (they're URLs);
public MCP tool names must be version-agnostic.

**Test both permission levels explicitly.** The façade's fallback path is only exercised
when SuperAdmin is absent. In CI, mock `AlarmManagerPermissionError` or `BadRequest`.

**Apply `api_request_raw` proactively.** Before shipping any new DELETE or
discarded-response POST endpoint in a Protect manager, probe with `api_request_raw`.

**ID spaces are non-overlapping.** Feeding a v2 UUID to the legacy backend returns
`UniFiNotFoundError` with no useful diagnostic. The facade must route; callers must
never bypass it.

**`rule_to_controller` is NOT a cross-backend serializer.** It emits
`name`/`conditions` (legacy vocabulary). Using it on the v2 serialization path silently
sends malformed bodies.

**Test serializers against real captured fixtures, not hand-crafted inputs.** Only real
fixture data catches structural mismatches in the legacy serializer inverse.

**Fallback must handle both 403 and 200 [].** A 200-empty response from an unmigrated
service looks like success — it is not.

**Future surface candidates.** Arm/disarm profiles (`/api/v2/alarms/profiles`) follow
the same dual-surface pattern if a legacy Protect path is ever added.
