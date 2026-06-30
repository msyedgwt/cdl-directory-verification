================================================================================
FLOW 1 — MONTHLY VERIFICATION DISTRIBUTION | DEVELOPMENT LOG
================================================================================

# Flow Overview

- Flow Name:    CDL Monthly Verification Distribution
- Platform:     Power Automate (Cloud Flow)
- Trigger:      Recurrence (monthly right now, future quarterly)
- Owner:        Mohtashim Syed
- Last Updated: 2026-06-30

--------------------------------------------------------------------------------

# Purpose

Distributes the monthly CDL Directory verification emails to every Client
Delivery Lead in the directory. Each email contains that account's current
directory snapshot and a link to the Power App verification form. Also resets
all per-cycle tracking fields so each month starts clean.

This is the first flow in a three-flow system:
- Flow 1 (this flow): Distributes monthly verification emails.
- Flow 2: Records the response when the CDL submits the Power App form.
- Flow 3: Handles reminders (Day 7, Day 14) and escalation (Day 21).

--------------------------------------------------------------------------------

# Trigger

- Type:       Recurrence
- Frequency:  Month
- Interval:   1
- Day/Time:   1st of each month, 08:00 Central Time

--------------------------------------------------------------------------------

# Data Source (Test File)

- Connector:  Excel Online (Business)
- File:       CDL Directory - 5.19.2026.xlsx
- Location:   OneDrive for Business
- Table:      Table1 (formatted as Excel Table)
- Sheet:      [Sheet1]

Key column for updates: Account

--------------------------------------------------------------------------------

# Variable Initialization

- TotalAccounts  (Integer, 0)  — total rows processed
- MissingEmails  (Integer, 0)  — rows skipped due to blank CDL Email
- AuditLog       (Array,  [])  — per-row results for CSV report

--------------------------------------------------------------------------------

# Flow Logic

For each row in the CDL Directory:
  1. Increment TotalAccounts.
  2. Validate: skip if Account Name OR CDL Email is blank
       -> log to MissingEmails counter and AuditLog.
  3. Run Office Script ClearReminderColumns to clear:
       Reminder1Sent, Reminder2Sent, EscalationSent, Verification Timestamp.
  4. Update row to reset cycle state:
       Verification Status   -> Pending
       Verification SentDate -> utcNow()
       ReminderCount         -> 0
  5. Send verification email (HTML body with full directory snapshot
     and VERIFY DIRECTORY button linking to the Power App).
  6. Append AuditLog entry: { Account, Status: "Email Sent" }.

After the loop:
  7. Create CSV table from AuditLog.
  8. Save CSV to /Desktop/CDLVerification/ in OneDrive with timestamp.

--------------------------------------------------------------------------------

# Key Actions

1. PowerApps (V2) Trigger / Recurrence
2. Initialize Variables (TotalAccounts, MissingEmails, AuditLog)
3. List rows present in a table  — DateTime Format: ISO 8601
4. Apply to each (over body/value)
   4a. Increment TotalAccounts
   4b. Condition — Account Name blank OR CDL Email blank?
        - True  -> Increment MissingEmails, append AuditLog "Missing CDL Email"
        - False -> proceed
   4c. Run script: ClearReminderColumns (accountName = Account)
   4d. Update a row (Verification Status, Verification SentDate, ReminderCount)
   4e. Send an email (V2) with directory snapshot HTML
   4f. Append to AuditLog: { Account, Status: "Email Sent" }
5. Create CSV table (from AuditLog)
6. Create File (OneDrive: CDL_Verification_Audit_[timestamp].csv)

--------------------------------------------------------------------------------

# Office Script Dependency

- Script Name: ClearReminderColumns
- Location:    OneDrive for Business » Documents » Office Scripts
- Purpose:     Clears Reminder1Sent, Reminder2Sent, EscalationSent, and
               Verification Timestamp cells to TRUE blank (avoids the Excel
               Online connector's null -> "False" bug).
- Parameters:  accountName (string) — passed from current Apply to each row.

--------------------------------------------------------------------------------

# Email Template Highlights

- Subject:  Monthly CDL Directory Verification Required - [Account Name]
- Body:     Full directory snapshot in HTML table (20 fields), VERIFY
            DIRECTORY button linking to Power App with ?account=[Account]
            in the URL so the form opens to the right row.
- Sign-off: "Selecting VERIFY DIRECTORY will open the verification form
            where you can review the directory, make any updates, and
            submit your confirmation."

--------------------------------------------------------------------------------

# Design Notes

- The connector returns numeric columns as strings ("0", "1", etc.). Flow 3
  uses int() to convert before comparing. Flow 1 writes numeric literals
  directly to ReminderCount (which the connector accepts).

- Office Script is required for true cell-blanking. Direct null/empty
  writes via the connector produce the literal string "False" in Excel.

- The Apply to each loop is intentionally not parallelized — Excel Online
  serialization of row updates is required to avoid race conditions.

--------------------------------------------------------------------------------

# Known Issues / Lessons Learned

- 2026-06-29: Connector wrote "False" into reminder columns when given
  null/empty values. Resolved by introducing the ClearReminderColumns
  Office Script and routing all blanking through it.

- 2026-06-30: Trailing-space column headers (AGM_, Legal Contact_) caused
  dynamic content tokens to silently break. Resolved by renaming headers
  and refreshing dynamic content.

- 2026-06-30: Reminder1Sent / Reminder2Sent / EscalationSent /
  Verification Timestamp columns historically held the literal string
  "False" from prior Update a row failures. One-time bulk clear performed.

--------------------------------------------------------------------------------

# Dependencies

- Power App:   CDL Directory Verification App
- Flow 2:      Verification Handler (writes Verification Status = Verified)
- Flow 3:      Reminder & Escalation Handler (reads Verification SentDate
               and ReminderCount set by this flow)
- Excel File:  CDL Directory - 5.19.2026.xlsx
- Script:      ClearReminderColumns (OneDrive Office Scripts)

--------------------------------------------------------------------------------

# Open Items / Next Steps

- [ ] Enable failure notifications on this flow.
- [ ] Add Power BI dashboard sourced from this Excel file.
- [ ] Consider migrating data source from Excel to Dataverse to eliminate
      the null-handling and string-typed-numerics issues.

================================================================================
END OF LOG
================================================================================
