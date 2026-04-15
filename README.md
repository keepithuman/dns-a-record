# DNS A Record Provisioning — Simple

Self-service DNS A record creation in Infoblox via an Operations Manager form. Operators fill in a hostname, zone, and IP — the workflow runs fully automatically, creates the record, and sends an email confirmation. No manual steps, no approval gate.

---

## What Was Built

| Asset | Name | ID |
|-------|------|----|
| Workflow | `DNS A Record Provisioning - Simple` | `25658482-086e-46a7-9ba4-4f202ae46dc6` |
| Project | `DNS A Record Provisioning - Simple` | `69dfa72a8bd76ce3c64563f6` |
| JSON Form | `DNS A Record - Simple` | `69dfa77fc35d7fda99621936` |
| OM Automation | `DNS A Record Provisioning - Simple` | `69dfa7c894b95ba404cc137f` |
| OM Trigger | `Manual` | `69dfa7d394b95ba404cc1380` |

**Platform:** https://platform-6-aidev.se.itential.io
**Access:** `ankit.bhansali@itential.com` (owner), `Solutions Engineering` (editor)

---

## How to Use It

1. Log in to the platform and open **Operations Manager**
2. Find **DNS A Record Provisioning - Simple**
3. Click **Run** — the JSON form opens
4. Fill in the fields:

| Field | Required | Example |
|-------|----------|---------|
| Hostname | Yes | `web01` |
| DNS Zone | Yes | `lab.example.com` (dropdown) |
| IP Address | Yes | `192.0.2.100` |
| DNS View | Yes | `default` |
| TTL (seconds) | No | `3600` |
| Comment | No | `Deployed by automation` |

5. Click **Run** — the workflow executes automatically
6. On success: A record appears in Infoblox and a confirmation email is sent to `ankit.bhansali@itential.com`

---

## Workflow Flow

```
Operator fills form
        │
        ▼
  [merge] Build FQDN vars (hostname + zone)
        │
        ▼
  [makeData] Build FQDN  →  hostname.zone
        │
        ▼
  [merge] Build Infoblox body (fqdn, ipv4addr, view, ttl, comment)
        │
        ▼
  [createARecord] → Infoblox WAPI POST /record:a
        │                           │
      success                     error
        │                           │
        ▼                           ▼
  [query] Extract _ref       [newVariable] Set error_message
        │                           │
        ▼                           ▼
  [merge] Build email vars    workflow_end
        │
        ▼
  [makeData] Build email body
        │
        ▼
  [mailWithOptions] → Send email
        │                    │
      success              error
        │                    │
        ▼                    ▼
  workflow_end       [newVariable] Set email_warning
                             │
                             ▼
                       workflow_end
```

Full diagram: open `workflow-diagram.drawio` in [draw.io](https://app.diagrams.net) or the VS Code draw.io extension.

---

## Before First Test

The workflow was deployed in **draft state** due to a schema mismatch on the `mailWithOptions` outgoing field (fixed, but Studio validation is needed to promote it).

**Steps to unblock:**
1. Open the platform → Automation Studio → Projects → `DNS A Record Provisioning - Simple`
2. Open the workflow → check for any remaining validation warnings → save
3. The workflow will be promoted from draft and can be started

---

## Open Items

| # | Item | Action |
|---|------|--------|
| 1 | Workflow in draft state | Open in Studio, resolve warnings, save |
| 2 | End-to-end test not run | Run after draft is resolved |
| 3 | Zone dropdown uses placeholder values | Update the JSON Form with real Infoblox authoritative zones |

---

## File Inventory

| File | Purpose |
|------|---------|
| `customer-spec.md` | Approved requirements (HLD) |
| `feasibility.md` | Platform assessment — adapters, capabilities |
| `solution-design.md` | Approved solution design (LLD) — component inventory, build decisions |
| `as-built.md` | What was delivered, deviations from design, learnings |
| `workflow-diagram.drawio` | Draw.io flow diagram of the 10-task workflow |
| `workflow.json` | Workflow build artifact — the JSON posted to the platform |
| `apps.json` | Platform app/adapter catalog (pulled during design) |
| `adapters.json` | Adapter instance health snapshot |
| `*-response.json` | API responses from each create call — contains all platform IDs |
| `.gitignore` | Excludes `.env`, `.auth.json`, large platform files |

---

## Continuing Work

To re-authenticate and continue work on this use case:

```bash
# Drop credentials into .env (not committed)
cat > .env <<EOF
PLATFORM_URL=https://platform-6-aidev.se.itential.io
AUTH_METHOD=oauth
CLIENT_ID=<your-client-id>
CLIENT_SECRET=<your-client-secret>
EOF
```

Then run `/builder-agent` to pick up from the as-built state, or `/solution-arch-agent design-only` if the design needs updating.

---

## Key Platform References

| Resource | ID |
|----------|----|
| Infoblox adapter instance | `infobloxv9` |
| Email adapter instance | `email` |
| `ankit.bhansali@itential.com` | `67eaf8b49b093bfbf0e62a9f` |
| Solutions Engineering group | `67c85954abe686cf9cb78b2e` |
