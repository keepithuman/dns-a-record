# Solution Design — DNS A Record Provisioning (Simple)

**Date:** 2026-04-15
**Platform:** https://platform-6-aidev.se.itential.io
**Spec:** customer-spec.md (approved)
**Feasibility:** feasibility.md (approved)

---

## A. Environment Summary

The target platform is `platform-6-aidev.se.itential.io` (cloud, OAuth). The Infoblox adapter instance `infobloxv9` is running and online with the `createARecord` task confirmed available. The email adapter instance `email` is running and online with `mailWithOptions` confirmed. All WorkFlowEngine tasks (`merge`, `makeData`, `query`, `newVariable`) are present. No reuse candidates — building new from scratch.

---

## B. Requirements Resolution

| Spec Requirement | Status | Resolution |
|-----------------|--------|------------|
| Create DNS A record via Infoblox | ✓ | `createARecord` — app: `infobloxv9`, locationType: `Infoblox` |
| Build FQDN from hostname + zone | ✓ | `merge` → `makeData` with `<!var!>` placeholders |
| Build Infoblox request body | ✓ | `merge` — WorkFlowEngine |
| Extract `_ref` from adapter response | ✓ | `query` — WorkFlowEngine |
| Send email notification on success | ✓ | `mailWithOptions` — app: `email`, locationType: `EmailOpensource` |
| Set error/warning variables | ✓ | `newVariable` — WorkFlowEngine |
| Self-service form trigger in Operations Manager | ✓ | JSON Form + manual trigger (`legacyWrapper: false`) |
| Fully automatic execution (no manual tasks) | ✓ | All tasks are `type: automatic` or `type: operation` |

---

## C. Design Decisions

| Decision | Spec Constraint | In This Environment |
|----------|----------------|---------------------|
| Adapter instance — Infoblox | Resolve from adapters at build time | `infobloxv9` |
| Adapter instance — Email | Resolve from adapters at build time | `email` |
| FQDN construction method | Use merge + makeData (not stringConcat) | `merge` → `makeData` with `<!hostname!>.<!zone!>` |
| Variable wiring | Prefer task-to-task over job vars | Email body wired `$var.<taskId>.output` directly |
| `legacyWrapper` | Must be `false` | Set on manual trigger — form fields map to job variables |
| Child workflows | None — spec is a flat 10-task workflow | Single workflow, no orchestrator pattern |
| Zone dropdown | Placeholder values — resolve at build time from Infoblox | Kept as-is in form; operator can also type zone manually |

---

## D. Component Inventory

| # | Component | Type | Action | Notes |
|---|-----------|------|--------|-------|
| 1 | `DNS A Record Provisioning - Simple` | Workflow (Automation Studio) | Build | 10 tasks, fully automatic |
| 2 | `DNS A Record - Simple` | JSON Form (Operations Manager) | Build | 6 fields: hostname, zone (dropdown), ip_address, dns_view, ttl, comment |
| 3 | `DNS A Record Provisioning - Simple` | Operations Manager Automation | Build | Binds form to workflow; manual trigger, `legacyWrapper: false` |
| 4 | `DNS A Record Provisioning - Simple` | Automation Studio Project | Build | Owns the workflow; account owner: `ankit.bhansali@itential.com` |

> **No child workflows.** The spec explicitly requires a flat 10-task workflow. All logic lives in the single workflow.

---

## E. Workflow — Task Map

```
workflow_start
  │
  ├─[T1] merge          Build FQDN Variables     WorkFlowEngine / operation
  ├─[T2] makeData       Build FQDN               WorkFlowEngine / operation
  ├─[T3] merge          Build Request Body        WorkFlowEngine / operation
  ├─[T4] createARecord  Create A Record           infobloxv9 / automatic
  │         ├── success
  │         │   ├─[T5] query       Extract Infoblox Ref    WorkFlowEngine / operation
  │         │   ├─[T6] merge       Build Email Variables   WorkFlowEngine / operation
  │         │   ├─[T7] makeData    Build Email Body        WorkFlowEngine / operation
  │         │   └─[T8] mailWithOptions  Send Email         email / automatic
  │         │             ├── success → workflow_end
  │         │             └── error
  │         │                 └─[T9] newVariable  Email Warning  → workflow_end
  │         └── error
  │             └─[T10] newVariable  Error Handler  → workflow_end
```

**Adapter IDs (hardcoded in tasks, not job variables):**
- T4 `createARecord`: `adapter_id = "infobloxv9"`
- T8 `mailWithOptions`: `adapter_id = "email"`

---

## F. JSON Form Field Spec

Form name (exact): `DNS A Record - Simple`

| Field Key | Label | Type | Required | Default | Widget |
|-----------|-------|------|----------|---------|--------|
| `hostname` | Hostname | string | yes | — | text input |
| `zone` | DNS Zone | string | yes | — | dropdown (enum) |
| `ip_address` | IP Address | string | yes | — | text input |
| `dns_view` | DNS View | string | yes | `default` | text input |
| `ttl` | TTL (seconds) | integer | no | `3600` | number input |
| `comment` | Comment | string | no | — | text input |

Zone dropdown enum values:
```json
["corp.example.com", "lab.example.com", "dev.example.com", "staging.example.com"]
```

---

## G. Implementation Plan

Build order — each step is independently verifiable before the next:

| Step | Action | Test |
|------|--------|------|
| 1 | Create Automation Studio project `DNS A Record Provisioning - Simple` | Project visible in Studio |
| 2 | Build workflow with all 10 tasks and correct transitions | Workflow validates in Studio |
| 3 | Build JSON Form `DNS A Record - Simple` with all 6 fields | Form renders correctly in preview |
| 4 | Create Operations Manager automation — bind form to workflow, `legacyWrapper: false` | Automation appears in OM |
| 5 | Run end-to-end test with a real hostname + IP | Job completes, `_ref` in output, email received |

---

## H. Acceptance Criteria → Tests

| # | Criterion | How to Verify |
|---|-----------|---------------|
| 1 | Operator fills form → workflow runs automatically | Run from Operations Manager — job starts immediately, no pause |
| 2 | A record created in Infoblox, `_ref` in job output | Check `$var.job.infoblox_ref` in job variables after run |
| 3 | Email sent on success with FQDN, IP, view, ref | Check inbox at `ankit.bhansali@itential.com` |
| 4 | Infoblox errors route to error handler | Check `$var.job.error_message` on a failed run |
| 5 | Email failure warns but does not fail job | Check `$var.job.email_warning` when email adapter is degraded |
| 6 | Exactly 10 tasks, no manual tasks | Count tasks in workflow canvas |

---

## Resolved Adapter IDs

| Role | Instance ID | App Name (locationType) |
|------|-------------|------------------------|
| Infoblox | `infobloxv9` | `Infoblox` |
| Email | `email` | `EmailOpensource` |
