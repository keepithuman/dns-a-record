# As-Built — DNS A Record Provisioning (Simple)

**Date:** 2026-04-15
**Platform:** https://platform-6-aidev.se.itential.io
**Builder:** tester (OAuth client `69ada6313f6ac74ee0dbbe78`)
**Status:** Assets deployed. End-to-end test pending.

---

## Workflow Diagram

**File:** `workflow-diagram.drawio` — open in [draw.io](https://app.diagrams.net) or VS Code draw.io extension.

Flow: happy path (green/blue) runs left-to-right. Error paths (red dashed) drop down from `createARecord` and `mailWithOptions`. Both converge at `workflow_end`.

---

## Delivered Assets

| # | Asset | Type | ID | Status |
|---|-------|------|----|--------|
| 1 | `@69dfa72a8bd76ce3c64563f6: DNS A Record Provisioning - Simple` | Workflow | `_id: 25658482-086e-46a7-9ba4-4f202ae46dc6` `uuid: 3bfe783e-1060-40be-b35b-a18821002083` | Deployed — draft state (see deviations) |
| 2 | `DNS A Record Provisioning - Simple` | Automation Studio Project | `_id: 69dfa72a8bd76ce3c64563f6` | Deployed |
| 3 | `DNS A Record - Simple` | JSON Form (Operations Manager) | `id: 69dfa77fc35d7fda99621936` | Deployed |
| 4 | `DNS A Record Provisioning - Simple` | Operations Manager Automation | `_id: 69dfa7c894b95ba404cc137f` | Deployed |
| 5 | `Manual` | Operations Manager Trigger | `_id: 69dfa7d394b95ba404cc1380` | Deployed |

### Project Membership

| Type | Role | Identity | Reference |
|------|------|----------|-----------|
| account | owner | `ankit.bhansali@itential.com` | `67eaf8b49b093bfbf0e62a9f` |
| group | editor | `Solutions Engineering` | `67c85954abe686cf9cb78b2e` |

---

## Workflow — Resolved Adapter IDs

| Role | Instance ID | App / locationType |
|------|-------------|--------------------|
| Infoblox | `infobloxv9` | `Infoblox` |
| Email | `email` | `EmailOpensource` |

---

## Acceptance Criteria Status

| # | Criterion | Status |
|---|-----------|--------|
| 1 | Operator fills form → workflow runs automatically | Pending test |
| 2 | A record created in Infoblox, `_ref` in job output | Pending test |
| 3 | Email sent to `ankit.bhansali@itential.com` on success | Pending test |
| 4 | Infoblox errors route to error handler | Pending test |
| 5 | Email failure warns but does not fail job | Pending test |
| 6 | Exactly 10 tasks, no manual tasks | ✓ Verified in workflow JSON |

> **Unblocking the test:** The workflow is currently in draft state (see deviation #1 below). Resolve the draft before running the end-to-end test. Once the workflow validates cleanly, start the test job with: `POST /operations-manager/jobs/start` using workflow name `@69dfa72a8bd76ce3c64563f6: DNS A Record Provisioning - Simple`.

---

## Deviations from Design

### Deviation 1 — `mailWithOptions` outgoing field: `result` → `response`

**Designed:** `outgoing: { "result": null }`
**Actual:** Schema requires `outgoing: { "response": null }`
**Discovered:** POST /automation-studio/automations returned a validation error in the `errors[]` array: `Output: "result" does not match model output: "response"`
**Fixed:** PUT /automation-studio/automations/{id} corrected the outgoing field to `response`
**Impact:** Workflow was created in draft state due to this initial mismatch. Fix applied but draft status may persist until validated cleanly in Studio.

**Design update:** In `solution-design.md` Task Map, `b2c2 mailWithOptions` outgoing is `response` (not `result`).

### Deviation 2 — Component add requires `_id`, not `uuid`

**Designed:** "Use uuid as reference for component add"
**Actual:** `POST /projects/{id}/components/add` with `reference: uuid` returned a failed component. Succeeded with `reference: _id`.
**Impact:** None — component added successfully on second attempt.
**Learning:** On this platform version (5.55.5), the `components/add` reference field expects the workflow's `_id` (UUID-format string), not the separate `uuid` field.

### Deviation 3 — Workflow `_id` is UUID format, not 24-char hex

**Designed:** Standard MongoDB ObjectId (`_id`) for component references
**Actual:** Workflow `_id` is in UUID format (`25658482-086e-46a7-9ba4-4f202ae46dc6`) on IAP 5.55.5. This is platform-version-specific behaviour — newer IAP versions use UUID-format `_id` for workflows.
**Impact:** None functional. Builder must not assume 24-char hex format for workflow IDs on this platform.

---

## Build Learnings

1. **`mailWithOptions` outgoing is `response`, not `result`** — always fetch the task schema via `multipleTaskDetails` before wiring adapter outgoing fields. Do not assume `result` for all adapter tasks.

2. **`components/add` reference = workflow `_id`, not `uuid`** — despite the helper docs saying to use `uuid`, this platform requires `_id`. Verify by reading the workflow create response and using the `created._id` field.

3. **Workflow draft state from validation errors** — if the initial POST creates the workflow with items in the `errors[]` array, the workflow is in draft state. Fix the errors via PUT and verify the workflow validates cleanly in Studio before testing.

4. **`GET /automation-studio/workflows?limit=N` does not return all workflows** — did not return our newly-created workflow during the build. Use the project endpoint (`GET /automation-studio/projects/{id}`) to confirm component membership, and `POST /operations-manager/jobs/start` with the full `@projectId:` name to test.

5. **Group lookup requires scanning individual project GETs** — the project list endpoint (`GET /projects?limit=N`) does not include member details. Resolved by looping over individual `GET /projects/{id}` calls and extracting `members[]` with `name` field.

6. **Solutions Engineering group reference:** `67c85954abe686cf9cb78b2e` — resolved on this platform. Reuse for future projects on `platform-6-aidev.se.itential.io`.

---

## Open Items

| # | Item | Action |
|---|------|--------|
| 1 | Workflow in draft state | Open in Automation Studio, resolve any remaining validation warnings, save to promote from draft |
| 2 | End-to-end test not run | Run `POST /operations-manager/jobs/start` with test variables after draft resolved |
| 3 | Zone dropdown values are placeholders | Update JSON Form zone enum with real Infoblox authoritative zones when known |

---

## Environment Reference (platform-6-aidev.se.itential.io)

| Resource | Reference ID |
|----------|-------------|
| `ankit.bhansali@itential.com` | `67eaf8b49b093bfbf0e62a9f` |
| `Solutions Engineering` group | `67c85954abe686cf9cb78b2e` |
| Infoblox adapter instance | `infobloxv9` |
| Email adapter instance | `email` |
