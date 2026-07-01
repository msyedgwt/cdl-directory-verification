# Flow 1 — Monthly Verification Distribution

> **Development Log**

| | |
|---|---|
| **Flow Name** | CDL Monthly Verification Distribution |
| **Platform** | Power Automate (Cloud Flow) |
| **Trigger** | Recurrence (quarterly) |
| **Owner** | Mohtashim Syed |
| **Last Updated** | 2026-07-01 |

---

## Purpose

Distributes the quarterly CDL Directory verification emails to every Client Delivery Lead in the directory. Each email contains that account's current directory snapshot and a link to the Power App verification form. Also resets all per-cycle tracking fields so each month starts clean.

This is the first flow in a three-flow system:

- **Flow 1 (this flow):** Distributes quarterly verification emails.
- **Flow 2:** Records the response when the CDL submits the Power App form.
- **Flow 3:** Handles reminders (Day 7, Day 14) and escalation (Day 21).

---

## Trigger

| Setting | Value |
|---|---|
| **Type** | Recurrence |
| **Frequency** | Month |
| **Interval** | 3 |
| **Day/Time** | 1st of month, 08:00 Central Time |

---

## Data Source (Test File)

| | |
|---|---|
| **Connector** | Excel Online (Business) |
| **File** | `CDL Directory - 5.19.2026.xlsx` |
| **Location** | OneDrive for Business |
| **Table** | `Table1` *(formatted as Excel Table)* |
| **Sheet** | `Sheet1` |
| **Key column for updates** | `Account` |

---

## Variable Initialization

| Variable | Type | Initial Value | Purpose |
|---|---|---|---|
| `TotalAccounts` | Integer | `0` | Total rows processed |
| `MissingEmails` | Integer | `0` | Rows skipped due to blank CDL Email |
| `AuditLog` | Array | `[]` | Per-row results for the CSV report |

---

## Flow Logic

### For each row in the CDL Directory

1. Increment `TotalAccounts`.
2. Validate: skip if `Account Name` OR `CDL Email` is blank
   → log to `MissingEmails` counter and `AuditLog`.
3. Run Office Script `ClearReminderColumns` to clear:
   - `Reminder1Sent`
   - `Reminder2Sent`
   - `EscalationSent`
   - `Verification Timestamp`
4. Update row to reset cycle state:
   - `Verification Status` → `Pending`
   - `Verification SentDate` → `utcNow()`
   - `ReminderCount` → `0`
5. Send verification email *(HTML body with full directory snapshot and VERIFY DIRECTORY button linking to the Power App).*
6. Append `AuditLog` entry: `{ Account, Status: "Email Sent" }`.

### After the loop

7. Create CSV table from `AuditLog`.
8. Save CSV to `/Desktop/CDLVerification/` in OneDrive with timestamp.

---

## Key Actions

1. **PowerApps (V2) Trigger / Recurrence**
2. **Initialize Variables** (`TotalAccounts`, `MissingEmails`, `AuditLog`)
3. **List rows present in a table** — DateTime Format: `ISO 8601`
4. **Apply to each** (over `body/value`)
   - **4a.** Increment `TotalAccounts`
   - **4b.** Condition — Account Name blank OR CDL Email blank?
     - **True** → Increment `MissingEmails`, append AuditLog `"Missing CDL Email"`
     - **False** → proceed
   - **4c.** Run script: `ClearReminderColumns` *(accountName = Account)*
   - **4d.** Update a row *(Verification Status, Verification SentDate, ReminderCount)*
   - **4e.** Send an email (V2) with directory snapshot HTML
   - **4f.** Append to `AuditLog`: `{ Account, Status: "Email Sent" }`
5. **Create CSV table** (from `AuditLog`)
6. **Create File** *(OneDrive: `CDL_Verification_Audit_[timestamp].csv`)*

---

## Office Script Dependency

| | |
|---|---|
| **Script Name** | `ClearReminderColumns` |
| **Location** | OneDrive for Business » Documents » Office Scripts |
| **Purpose** | Clears `Reminder1Sent`, `Reminder2Sent`, `EscalationSent`, and `Verification Timestamp` cells to TRUE blank *(avoids the Excel Online connector's `null` → `"False"` bug)*. |
| **Parameters** | `accountName` *(string)* — passed from current Apply to each row |

---

## Email Template Highlights

- **Subject:** `CDL Directory Verification Required - [Account Name]`
- **Body:** Full directory snapshot in HTML table, VERIFY DIRECTORY button linking to the Power App with `?account=[Account]` in the URL so the form opens to the right row.
- **Sign-off:** *"Selecting VERIFY DIRECTORY will open the verification form where you can review the directory, make any updates, and submit your confirmation."*

---

## Design Notes

- The connector returns **numeric columns as strings** (`"0"`, `"1"`, etc.). Flow 3 uses `int()` to convert before comparing. Flow 1 writes numeric literals directly to `ReminderCount` *(which the connector accepts)*.

- Office Script is required for **true cell-blanking**. Direct `null`/empty writes via the connector produce the literal string `"False"` in Excel.

- The Apply to each loop is **intentionally not parallelized** — Excel Online serialization of row updates is required to avoid race conditions.

---

## Known Issues / Lessons Learned

### 2026-06-29 — Connector wrote `"False"` into reminder columns
**Cause:** Excel Online connector treats `null`/empty values as boolean `false`.
**Resolution:** Introduced the `ClearReminderColumns` Office Script. All blanking now routes through it.

### 2026-06-30 — Trailing-space column headers broke dynamic content
**Cause:** Headers `AGM ` and `Legal Contact ` *(with trailing space / `&nbsp;`)* silently broke dynamic content tokens.
**Resolution:** Renamed headers in Excel, refreshed dynamic content in affected actions.

### 2026-06-30 — Historical `"False"` values across timestamp columns
**Cause:** `Reminder1Sent`, `Reminder2Sent`, `EscalationSent`, and `Verification Timestamp` columns historically held the literal string `"False"` from prior Update a row failures.
**Resolution:** One-time bulk clear performed across all rows.

---

## Dependencies

| Type | Resource |
|---|---|
| **Power App** | CDL Directory Verification App |
| **Flow 2** | Verification Handler *(writes `Verification Status = Verified`)* |
| **Flow 3** | Reminder & Escalation Handler *(reads `Verification SentDate` and `ReminderCount` set by this flow)* |
| **Excel File** | `CDL Directory - 5.19.2026.xlsx` |
| **Office Script** | `ClearReminderColumns` *(OneDrive Office Scripts)* |

---

## Open Items / Next Steps

- [ ] Add Power BI dashboard sourced from this Excel file.
- [ ] Consider migrating data source from Excel to **Dataverse** to eliminate the null-handling and string-typed-numerics issues.

---

*End of log*
