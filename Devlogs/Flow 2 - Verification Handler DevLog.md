# Flow 2 — Verification Handler

> **Development Log**

| | |
|---|---|
| **Flow Name** | Flow 2 - Verification Handler |
| **Platform** | Power Automate (Cloud Flow) |
| **Trigger** | PowerApps (V2) |
| **Owner** | Mohtashim Syed |
| **Last Updated** | 2026-07-01 |

---

## Purpose

Handles the "Verify" submission from the CDL Directory Verification Power App. When a CDL clicks Submit, this flow updates that account's row in the CDL Directory Excel table to mark it verified for the current month.

This is the second flow in the three-flow system:

- **Flow 1:** Distributes monthly verification emails.
- **Flow 2 (this flow):** Records the response when the CDL verifies.
- **Flow 3:** Handles reminders and escalation.

---

## Trigger Inputs

| Name | Type | Source |
|---|---|---|
| `Account` | Text | `First(CDLGallery.AllItems).Account` |
| `Region` | Text | `First(CDLGallery.AllItems).Region` |
| `ChangesJSON` | Text | JSON payload of edited rows *(may be `"[]"`)* |

---

## Data Source (Test File)

| | |
|---|---|
| **Connector** | Excel Online (Business) |
| **File** | `CDL Directory - 5.19.2026.xlsx` |
| **Location** | OneDrive for Business |
| **Table** | `Table1` |
| **Sheet** | `VerificationTable` |
| **Key column for updates** | `Account` |

---

## Flow Logic

When the Power App calls this flow:

1. Receive `Account`, `Region`, `ChangesJSON` from Power Apps.
2. Update the row matching the submitted `Account`:
   - `Verification Status` → `Verified`
   - `Verification Timestamp` → `utcNow()`
   - `Last Verified` → `formatDateTime(utcNow(), 'MMMM yyyy')`

---

## Actions

### 1. PowerApps (V2) Trigger

**Inputs:**

| Input | Type |
|---|---|
| `Account` | Text |
| `Region` | Text |
| `ChangesJSON` | Text |

### 2. Update a row *(Excel Online Business)*

| Setting | Value |
|---|---|
| **Key Column** | `Account` |
| **Key Value** | `triggerBody()?['text']` |

**Fields updated:**

| Field | Value |
|---|---|
| `Verification Status` | `Verified` |
| `Verification Timestamp` | `utcNow()` |
| `Last Verified` | `formatDateTime(utcNow(),'MMMM yyyy')` |

---

## Design Notes

- Earlier draft used **List rows + Filter array + Condition** before Update a row. This was collapsed to a single Update a row keyed off `triggerBody()` — the Excel connector performs the row match natively. Simpler and faster.

- `ChangesJSON` is received but **not currently consumed** by this flow. The Power App writes change details directly to the `VerificationResponses` table on submit; `ChangesJSON` is reserved for potential future audit expansion within this flow.

- `Last Verified` stores a **human-readable "Month YYYY"** format for the directory display, while `Verification Timestamp` stores the **precise ISO timestamp** for audit precision.

---

## Known Issues / Lessons Learned

### 2026-06-29 — `[Expression.Error] The column 'Verification Timestamp' of the table wasn't found.`
**Cause:** Column existed in the sheet but was changed in the Excel Table.
**Resolution:** Excel → **Queries and Connections** → **Redo Query** to include the updated columns, then refresh dynamic content in Update a row.

### 2026-06-29 — Flow did not appear in Power Apps
**Cause:** Existing in Power Automate alone is not sufficient — the flow must be explicitly attached to the app.
**Resolution:** Added the flow via the **Power Automate pane inside Power Apps Studio**.

### Environment alignment requirement
**Cause:** Power Apps and Power Automate must be in the **same environment** for the flow to be selectable from the app.
**Resolution:** Verified both Power App and Flow 2 reside in the same environment before publishing.

---

## Dependencies

| Type | Resource |
|---|---|
| **Power App** | CDL Directory Verification App |
| **Companion Flow** | Flow 1 - Monthly Verification Distribution |
| **Companion Flow** | Flow 3 - Reminder & Escalation Handler |
| **Excel File** | `CDL Directory - 5.19.2026.xlsx` |
| **Change Log** | `VerificationResponses` table *(written from Power Apps, not from this flow)* |

---

## Open Items / Next Steps

- [ ] Persist `ChangesJSON` to a dedicated audit table for change history.
- [ ] Add Teams notification to the Regional Delivery Leader when a verification is recorded.

---

*End of log*
