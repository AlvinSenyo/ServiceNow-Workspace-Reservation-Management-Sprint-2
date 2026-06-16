# ServiceNow-Workspace-Reservation-Management-Sprint-2
An extension of the Sprint 1 WRM scoped application that automates the full reservation lifecycle — client-side validation, Flow Designer approval routing, Discord notifications, role-based ACLs, email confirmations, SLA tracking, and a self-service bulk workspace import.

**Product:** ServiceNow Scoped Application — Flow Designer + Import Framework  
**Release:** ServiceNow Zurich  
**Stories Completed:** 15 – 21  
**Prerequisite:** Sprint 1 (Stories 1 – 14) must be fully deployed before starting Sprint 2.

---

## 📝 What's New in Sprint 2

Sprint 1 built the foundation — tables, roles, modules, UI Policies, UI Actions, and the catalog item. Sprint 2 wires everything together with automation, security, and notifications.

| Capability | Sprint 1 | Sprint 2 |
| :--- | :---: | :---: |
| Reservation catalog item | ✅ | — |
| Date validation on catalog form | — | ✅ |
| Location auto-populate on catalog form | — | ✅ |
| Automated approval flow (Flow Designer) | — | ✅ |
| Discord notifications (approval + rejection) | — | ✅ |
| Role-based ACLs (Read/Create/Write/Delete × 3 tables) | — | ✅ |
| Reservation confirmation email | — | ✅ |
| Fulfillment SLA — 4 business hours + workflow | — | ✅ |
| Agent check-out alert email | — | ✅ |
| Bulk workspace import on demand | — | ✅ |

---

## 🏛️ Architecture

**Flow Designer First.** All multi-step automation lives in Flow Designer — logic is visible, maintainable, and role-aware. The `Workspace Reservation` flow handles the full approval lifecycle. The `WRM Bulk Import Trigger` flow handles file ingestion.

**Script Include Pattern.** The Discord integration is encapsulated in a reusable `discordNotification` Script Include (scoped). Two custom Flow Actions call it with the right message and emoji. Adding a third notification in the future means writing one new Action — the delivery logic is never touched.

**System Property for Config.** The Discord webhook URL lives in a System Property (`x_#####_wrm.discord.webhook_url`), not in code. Rotating the URL means updating one record.

**Global Script Include for Import APIs.** The `importOnDemand` Script Include lives in **global** scope because `GlideImportSetLoader` and `GlideImportSetTransformer` are global APIs. It is called from the scoped `Import Workspace Data` Flow Action using `new global.importOnDemand()`.

**ACL-First Security.** Every table has explicit Read/Create/Write/Delete ACLs scoped to the correct roles. No table relies on default platform behavior.

**Notification over Scripting.** Email notifications for Stories 18 and 20 use the platform Notification engine with dot-walking variable substitution — no Business Rules or email scripts required.

**Tracker Record Before Discord.** In the approval flow, the Reservation Tracker (WRT) record is created **before** the Discord notification fires. The approval message includes a clickable link to the WRT record — which requires the record's `sys_id` to exist first.

---

## 📂 File Inventory

| File | Description |
| :--- | :--- |
| `Technical_Manual_WRM_Sprint2_v4.txt` | Click-by-click build guide for all 7 stories with Option A/B, complete scripts, and verification checklists |
| `WRM-Sprint2-Implementation-Guide-v4.docx` | Formatted step-by-step guide with navigation boxes, field tables, and production-ready scripts |
| `WRM_Workspace_Bulk_Import_Sample.xlsx` | 10-row sample Excel file for testing Story 21. Headers match the import staging table column names exactly |
| `README.md` | This file |

---

## 🗂️ Tables Referenced (from Sprint 1)

| Table Label | Internal Name | Prefix | Extends |
| :--- | :--- | :--- | :--- |
| Workspace Options | `x_#####_wrm_workspace_options` | WRO | — |
| Reservation Tracker | `x_#####_wrm_reservation_tracker` | WRT | task |
| Workspace Maintenance | `x_#####_wrm_workspace_maintenance` | WRM | — |

---

## 🔐 Roles & Groups

| Name | Type | Purpose |
| :--- | :--- | :--- |
| `wrm_user` | Role | Employees who submit reservations via the Service Catalog |
| `wrm_admin` | Role | Administrators who manage workspace inventory |
| `wrm_agent` | Role | Agents who process reservations and manage the full lifecycle |
| `Workspace Agents` | Group | Receives and approves reservation requests. Has `itil`, `approver_user`, and `wrm_agent` roles |

---

## 🔒 ACL Matrix

### Workspace Options

| Operation | Allowed Roles |
| :--- | :--- |
| Read | No role required |
| Create | `wrm_admin` or table role |
| Write | `wrm_admin` or table role |
| Delete | `admin` only |

### Reservation Tracker

| Operation | Allowed Roles |
| :--- | :--- |
| Read | `wrm_agent`, `wrm_user` |
| Create | `wrm_agent`, `wrm_user` |
| Write | `wrm_agent`, `wrm_user` |
| Delete | `admin` only |

### Workspace Maintenance

| Operation | Allowed Roles |
| :--- | :--- |
| Read | `wrm_agent` or maintenance table role |
| Create | `wrm_agent` or maintenance table role |
| Write | Maintenance table role |
| Delete | `admin` only |

---

## 🔁 Reservation Lifecycle — Sprint 2

```
[User submits Workspace Reservation Request]
         │
         ▼
  RITM Created (sc_req_item)
  SLA clock starts ── 4 business hours
         │
         ▼
  Flow Designer: Workspace Reservation triggers
         │
         ▼
  Approval sent ──► Workspace Agents group
         │
         ├── [Approved] ─────────────────────────────────────────────┐
         │       │                                                    │
         │       ▼                                                    │
         │  WRT record created                                        │
         │  Status: Pending check-in  │  State: Open                 │
         │  Assignment group: Workspace Agents                       │
         │       │                                                    │
         │       ▼                                                    │
         │  Discord ✅  approved message + clickable WRT link         │
         │       │                                                    │
         │       ▼                                                    │
         │  RITM ── Closed Complete                                   │
         │  SLA clock stops                                          │
         │  Confirmation email ──► submitter                         │
         │       │                                                    │
         │  [Check-In] ──► Status: Checked-in                        │
         │       │                                                    │
         │  [Check-Out] ──► Status: Checked-out                      │
         │                  State: Closed Complete                    │
         │                  Alert email ──► assigned agent            │
         │                       │                                    │
         │               [Needs Maintenance]                          │
         │                       │                                    │
         │               Maintenance record created                   │
         │                                                            │
         └── [Rejected] ──────────────────────────────────────────────┘
                 │
                 ▼
         Discord ❌  rejection message
                 │
                 ▼
         RITM ── Closed Incomplete
         Rejection email ──► submitter
```

---

## 💡 Key Features

### Client-Side Date Validation (Story 15)
An `onChange` Catalog Client Script fires the moment the user sets End date/time. If End is before Start, an inline error fires and the End field clears — in real time, before Submit is clicked. Location auto-populates from the selected workspace via a `javascript:` Default value expression — no separate client script record required.

### Automated Reservation Flow (Story 16)
Triggered by catalog submission. Routes approval to the Workspace Agents group. On approval: creates the WRT record first (so the sys_id exists for the Discord link), fires the Discord ✅ message with a clickable link, then closes the RITM as Closed Complete. On rejection: fires the Discord ❌ message, closes the RITM as Closed Incomplete, and sends a rejection email.

### Discord Integration (Story 16)
A `discordNotification` Script Include wraps `sn_ws.RESTMessageV2` to POST messages to a Discord channel via webhook. The webhook URL is stored in a System Property. A `try/catch` block ensures a failed REST call logs a warning without crashing the flow. Two custom Flow Actions (`Send Discord Approval Notification`, `Send Discord Rejection Notification`) call this Script Include with the correct message and emoji.

```javascript
// discordNotification Script Include (WRM scope)
var discordNotification = Class.create();
discordNotification.prototype = {
    initialize: function() {
        this.webhookURL = gs.getProperty('x_#####_wrm.discord.webhook_url');
    },
    sendMessage: function(message, emoji) {
        if (!this.webhookURL) { gs.warn('Webhook URL not configured.'); return false; }
        try {
            var request = new sn_ws.RESTMessageV2();
            request.setEndpoint(this.webhookURL);
            request.setHttpMethod('POST');
            request.setRequestHeader('Content-Type', 'application/json');
            request.setRequestBody(JSON.stringify({ content: (emoji ? emoji + ' ' : '') + message }));
            var httpStatus = request.execute().getStatusCode();
            return (httpStatus == 200 || httpStatus == 204);
        } catch (ex) { gs.error('discordNotification: ' + ex.message); return false; }
    },
    type: 'discordNotification'
};
```

### 4-Hour Fulfillment SLA + Workflow (Story 19)
An SLA Definition on `sc_req_item` starts when a Workspace Reservation Request RITM opens (scoped by Cat item condition) and stops on Closed Complete. Uses a Business Hours schedule. A cloned workflow (`WRM Reservation SLA Workflow`) sends warning emails to the Workspace Agents group at 50%, 75%, and 100% of the target time, giving agents advance notice before breach.

### Agent Check-Out Alert (Story 20)
When a WRT record's `Reservation status` transitions to `Checked-out`, an email fires to the `Assigned to` agent. Uses `changes to` so it fires only once at the transition point, not on every subsequent save.

### Bulk Workspace Import On Demand (Story 21)
A catalog item lets workspace managers upload an Excel file from the Service Portal. Built on the pattern from the ServiceNow Community article by Kalisha_m (July 2025).

```javascript
// importOnDemand Script Include (GLOBAL scope)
var importOnDemand = Class.create();
importOnDemand.prototype = {
    importAndTransform: function(dataSourceSysId, transformMapSysId) {
        var grDataSource = new GlideRecord('sys_data_source');
        if (!grDataSource.get(dataSourceSysId)) { return null; }
        var loader = new GlideImportSetLoader();
        var grImportSet = loader.getImportSetGr(grDataSource);
        loader.loadImportSetTable(grImportSet, grDataSource);
        grImportSet.setValue('state', 'loaded');
        grImportSet.update();
        var transformer = new GlideImportSetTransformer();
        transformer.setImportSetID(grImportSet.getUniqueValue());
        transformer.setMapID(transformMapSysId);
        transformer.setSyncImport(true);
        transformer.transformAllMaps(grImportSet);
        return { import_set_sys_id: grImportSet.getUniqueValue() };
    },
    type: 'importOnDemand'
};
```

The `Import Workspace Data` Flow Action calls `new global.importOnDemand()` from the scoped WRM context and returns the `import_set_sys_id`, which feeds the results email with actual row counts.

The bulk import flow has **9 steps**:

1. Lookup Data Source record (by name — avoids hardcoding sys_id)
2. Lookup Attachment on RITM (`sys_attachment` where `table_name = sc_req_item`)
3. **Move Attachments** — re-parents the file from the RITM (`sc_req_item`) to the Data Source record (`sys_data_source`). Source must be `Step 2 > Attachment Record` (not the RITM itself). Without this step, the import silently processes zero rows with no error.
4. Run `Import Workspace Data` action
5. Lookup Attachment on Data Source (`sys_attachment` where `table_name = sys_data_source`)
6. Delete that attachment (data hygiene — prevents stale files from being re-processed on subsequent runs)
7. Lookup `sys_import_set_run` for row counts (Transform History Record)
8. Close RITM as Closed Complete
9. Send results email with inserted / updated / errored / skipped counts (data pills required — `${var}` syntax does not work in Flow Designer Send Email)

> **System property required:** `sn_flow_designer.allowed_system_tables` must include `sys_import_set_run` before the flow can query import results.

---

## 🗺️ Artifact Summary

| Artifact | Type | Story | Scope |
| :--- | :--- | :---: | :--- |
| Validate Reservation Dates | Catalog Client Script | 15 | WRM |
| Workspace Agents | Group | 16 | Global |
| `x_#####_wrm.discord.webhook_url` | System Property | 16 | WRM |
| `discordNotification` | Script Include | 16 | WRM |
| Send Discord Approval Notification | Flow Action | 16 | WRM |
| Send Discord Rejection Notification | Flow Action | 16 | WRM |
| Workspace Reservation | Flow | 16 | WRM |
| ACL records (12 records, 3 tables) | Access Control | 17 | WRM |
| Workspace Reservation Confirmation | Email Notification | 18 | WRM |
| Workspace Reservation Fulfillment SLA | SLA Definition | 19 | WRM |
| WRM Reservation SLA Workflow | Workflow | 19 | Global |
| Workspace Check Required - Checked Out | Email Notification | 20 | WRM |
| Workspace Options Bulk Update | Catalog Item | 21 | WRM |
| WRM Workspace Options Bulk Import | Data Source | 21 | WRM |
| WRM Workspace Options Bulk Transform | Transform Map | 21 | WRM |
| `importOnDemand` | Script Include | 21 | **Global** |
| Import Workspace Data | Flow Action | 21 | WRM |
| WRM Bulk Import Trigger | Flow | 21 | WRM |
| Workspace Bulk Import Complete | Email Notification | 21 | WRM |

---

## 🚀 Deployment Instructions

Each story is packaged in its own named update set: `WRM STRY#### [description]`

1. **Source Instance:** Open each update set → set State to **Complete** → **Export to XML**
2. **Target Instance:** Navigate to `[System Update Sets]` → `[Retrieved Update Sets]` → **Import XML**
3. **Preview** the update set and resolve any conflicts
4. **Commit** — repeat for each story in order (0015 through 0021)

> **Migration order matters.** Story 16 depends on Story 15 variables. Story 17 requires Sprint 1 tables. Story 21's `importOnDemand` Script Include must be committed to the **Global** scope before the Flow Action and Flow will work. Import sequentially.

---

## ✅ Update Set Index

| Update Set Name | Story | Contents |
| :--- | :---: | :--- |
| WRM STRY0015 Catalog Reservation Validation | 15 | Date validation client script + location default value |
| WRM STRY0016 Request Workspace Reservation Flow | 16 | Workspace Agents + System Property + discordNotification + 2 Flow Actions + Flow |
| WRM STRY0017 ACL Updates | 17 | Read/Create/Write/Delete ACLs on all 3 WRM tables |
| WRM STRY0018 Reservation Confirmation Notification | 18 | Email notification on RITM Closed Complete |
| WRM STRY0019 Fulfillment SLA | 19 | SLA Definition + WRM Reservation SLA Workflow (cloned, Workspace Agents recipients) |
| WRM STRY0020 Check-Out Notification | 20 | Email to assigned agent on Checked-out status transition |
| WRM STRY0021 Bulk Import on Demand | 21 | Catalog item + Data Source + Transform Map + global importOnDemand + Flow Action + Flow + Notification |

---

## 🔭 What's Next — Sprint 3 Ideas

- **Scheduled Job** to auto-flag reservations as No-show when start time passes without a check-in
- **Service Portal Widget** for a real-time workspace availability map
- **Reporting & Dashboard** — wrm_admin dashboard showing volumes, no-show rates, and utilization
- **Cancel Reservation Flow** — self-service cancellation for pending-check-in reservations
- **Business Rule** to auto-sync `Reservation status` and `State` on every save
- **Workspace Capacity Rules** — enforce max occupancy at booking time

---

*Built by Senyo | AdvanceNow™ LLC | ServiceNow Zurich Release | Sprint 2*
