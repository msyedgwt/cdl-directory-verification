CDL DIRECTORY VERIFICATION SYSTEM
================================================================================

Owner:        Mohtashim Syed
Team:         Client Delivery Organization, Gainwell Technologies
Status:       Production
Last Updated: 2026-06-30
Version:      1.0

--------------------------------------------------------------------------------

# What This System Does

Every month, the CDL Directory must be re-verified by the Client Delivery Lead
(CDL) of each account. Verification means the CDL either confirms the
directory record is accurate or updates the fields that have changed.

Before this system existed, verification was a manual, ad-hoc email chase
that was inconsistent, slow, and impossible to audit. This system automates
the entire cycle end-to-end:

  1. Emails each CDL a snapshot of their directory record on Day 1.
  2. Lets them verify or edit it through a Power App with a Submit button.
  3. Records the verification in the source Excel directory automatically.
  4. Chases unverified CDLs with reminders on Days 7 and 14.
  5. Escalates to the Regional Delivery Leader on Day 21.

Goal: maintain an accurate, auditable, self-managed CDL Directory with
zero ongoing manual effort from the directory owner.

--------------------------------------------------------------------------------

# Architecture at a Glance

   +-----------------+
   |    Excel File   |  <--- single source of truth
   | CDL Directory   |
   +--------+--------+
            ^
            |  reads + writes
            |
   +--------+--------+        +--------------------+
   |     Flow 1      |        |       Flow 3       |
   | Monthly Sender  |        | Reminders / Escal. |
   | (runs monthly)  |        | (runs daily)       |
   +--------+--------+        +---------+----------+
            |                           ^
            |  sends email              |  reads state
            v                           |
   +-----------------+        +---------+----------+
   |  CDL receives   | -----> |     Power App      |
   | verification    | click  | Verification Form  |
   | email + button  | Verify | (gallery + Submit) |
   +-----------------+        +---------+----------+
                                        |
                                        |  Submit
                                        v
                              +---------+----------+
                              |       Flow 2       |
                              |   Verify Handler   |
                              | (triggered on tap) |
                              +--------------------+
                                        |
                                        |  writes back
                                        v
                                  [Excel File]

--------------------------------------------------------------------------------

# The Three Flows

| Flow   | Trigger              | Purpose                                  |
|--------|----------------------|------------------------------------------|
| Flow 1 | Monthly recurrence   | Distributes verification emails + resets |
|        | (1st of each month)  | tracking fields for the new cycle.       |
| Flow 2 | PowerApps (V2)       | Records the CDL's verification when they |
|        | (from Power App)     | click Submit in the form.                |
| Flow 3 | Daily recurrence     | Sends Day 7 and Day 14 reminders, then   |
|        | (08:00 CT)           | escalates to the RDL on Day 21.          |

Detailed action-level documentation lives in each flow's devlog:
  - Devlogs/Flow1_MonthlyVerificationDistribution_Devlog.txt
  - Devlogs/Flow2_VerificationHandler_Devlog.txt
  - Devlogs/Flow3_ReminderEscalationHandler_Devlog.txt

--------------------------------------------------------------------------------

# Data Model — CDL Directory Excel File

File:  CDL Directory - 5.19.2026.xlsx
Path:  OneDrive for Business » Documents » [path]
Table: Table1 (must be a formatted Excel Table — Ctrl+T)

Key column for all updates: Account (must be unique per row)

## Identity Columns
  - Account                       — short account code, primary key
  - Region                        — region grouping
  - Primary Solution Implemented  — solution / product context

## Contact Columns
  - CDL                           — Client Delivery Lead name
  - CDL Email                     — CDL's email (REQUIRED for sending)
  - Regional Delivery Leader      — RDL name
  - RDL Email                     — RDL's email (REQUIRED for Day 21 CC)
  - [Plus 15+ other leadership / contact columns the email displays]

## Verification Tracking Columns (written by the flows)
  - Verification Status      — Pending / Verified / Escalated
  - Verification SentDate    — when Flow 1 sent the cycle's email (ISO 8601)
  - Verification Timestamp   — when Flow 2 recorded the verification
  - Last Verified            — human-readable "Month YYYY" of last verify
  - ReminderCount            — integer 0-3, incremented by Flow 3
  - Reminder1Sent            — timestamp Flow 3 sent Reminder #1
  - Reminder2Sent            — timestamp Flow 3 sent Reminder #2
  - EscalationSent           — timestamp Flow 3 sent Escalation

## Critical Schema Rules

  - The table must be a real Excel Table (formatted via Ctrl+T or
    Insert > Table), not a plain range. Power Automate's Excel connector
    only sees columns that are inside the table boundary.

  - Column headers must NOT have trailing spaces or non-breaking spaces.
    These silently break dynamic content references in the flows.

  - Numeric columns (ReminderCount) are returned by the connector as
    strings. The flows use int() to convert before comparing.

  - The Excel connector cannot write a true blank value — null and ""
    are stored as the literal string "False". The ClearReminderColumns
    Office Script bypasses this by clearing cells from inside Excel
    itself. Do NOT attempt to write null/empty values from any flow.

--------------------------------------------------------------------------------

# The Verification Lifecycle (one row's journey)

   Day 0:   Flow 1 runs.
            -> CDL Email sent.
            -> Verification Status = Pending
            -> Verification SentDate = utcNow()
            -> ReminderCount = 0
            -> Reminder/Timestamp columns cleared

   Day 1-6: Flow 3 runs daily but takes no action (too early).

   Day 7+:  Flow 3 sends Reminder #1.
            -> ReminderCount = 1
            -> Reminder1Sent = utcNow()

   Day 14+: Flow 3 sends Reminder #2 (warns RDL will be CC'd next).
            -> ReminderCount = 2
            -> Reminder2Sent = utcNow()

   Day 21+: Flow 3 sends Escalation (CCs RDL).
            -> ReminderCount = 3
            -> EscalationSent = utcNow()
            -> Verification Status = Escalated
            (Row will not receive further action this cycle.)

   ANYTIME (Day 0 through Day 21):
            CDL clicks VERIFY DIRECTORY button in any email.
            -> Opens the Power App form.
            -> CDL reviews / edits / submits.
            -> Flow 2 runs.
            -> Verification Status = Verified
            -> Verification Timestamp = utcNow()
            -> Last Verified = "Month YYYY"
            (Flow 3 will skip this row for the rest of the cycle.)

--------------------------------------------------------------------------------

# Power App

Name:        CDL Directory Verification App
Environment: [same environment as Flow 2 — CRITICAL]
Data source: Excel Online (Business) — Table1
Change log:  VerificationResponses (separate Excel table)

The app opens to the account passed in the URL (?account=AK) and shows
a gallery of directory fields with editable text inputs. The Submit
button on the footer:
  1. Writes each edit to VerificationResponses (Patch).
  2. Builds a JSON summary of the changes.
  3. Calls Flow 2 with (Account, Region, ChangesJSON).
  4. Notifies the user "Verification Submitted."

The Power App must be in the same environment as Flow 2, or the flow
will not be selectable from the app.

--------------------------------------------------------------------------------

# Office Script Dependency

Name:      ClearReminderColumns
Location:  OneDrive » Documents » Office Scripts
Used by:   Flow 1 (called once per row, before Update a row)
Purpose:   Clears Reminder1Sent, Reminder2Sent, EscalationSent, and
           Verification Timestamp cells to TRUE blank. Required because
           the Excel Online connector cannot write null/empty values.
Parameter: accountName (string)

--------------------------------------------------------------------------------

# Operating Procedures

## Monthly cycle (automatic)
Flow 1 runs on the 1st of each month at 08:00 CT. No action required.

## Daily reminder cycle (automatic)
Flow 3 runs every day at 08:00 CT. No action required.

## When a CDL changes
1. Open the Excel file.
2. Find the row by Account.
3. Update CDL name and CDL Email.
4. Save. The next monthly run picks up the new CDL automatically.

## When an RDL changes
1. Open the Excel file.
2. Update RDL name and RDL Email across all affected accounts (filter
   by Region for efficiency).
3. Save.

## When a new account is added to the directory
1. Add a new row to the table (use Tab from the last cell to extend
   the table boundary automatically).
2. Fill in Account, CDL, CDL Email, Region, RDL, RDL Email, and any
   leadership/contact columns.
3. Set Verification Status = blank, ReminderCount = 0.
4. The new row will be included in the next monthly Flow 1 run.

## When an account is decommissioned
1. Delete the row from the Excel table, OR
2. Mark Verification Status = "Archived" (any non-Pending value will
   cause Flow 3 to skip the row).

## Mid-cycle re-send for a specific row
1. Open Excel.
2. Set that row's:
     - Verification Status = Pending
     - Verification SentDate = today (ISO 8601)
     - ReminderCount = 0
     - Reminder/Escalation columns = blank
3. Either manually run Flow 1 (will affect ALL rows) or send the
   verification email manually — Flow 3 will pick up reminders going
   forward from the new SentDate.

--------------------------------------------------------------------------------

# Troubleshooting

## "The flow ran but no emails went out"
- Check List rows returned rows (open the run history > List rows action).
- Check the outer Condition — Verification Status must be "Pending" and
  CDL Email must not be blank.
- Check Switch returned > 0. ReminderCount must be wrapped in int().

## "The column 'X' was not found"
- The column is outside the Excel Table boundary. Open Excel >
  Table Design > Resize Table to include all columns.
- Or: the column name has a trailing space / non-breaking space.
  Rename it in Excel and refresh dynamic content in the action.

## "Cells show 'False' instead of being blank"
- A flow wrote null/empty to those cells via Update a row. This is the
  Excel connector's known limitation. Clean them up manually, then
  ensure all blanking goes through ClearReminderColumns Office Script.

## "Flow does not appear in Power Apps"
- The flow must use the PowerApps (V2) trigger.
- The flow must be in the same environment as the Power App.
- Add the flow via the Power Automate pane inside Power Apps Studio,
  not just by referencing the name.

## "Case 3 (Escalation) won't fire during testing"
- The test row's Verification Status is probably "Escalated" already
  (which the outer Condition filters out). Set it back to "Pending"
  with ReminderCount = 2 and a SentDate of 22+ days ago.

--------------------------------------------------------------------------------

# Folder Structure

   /CDL Verification System/
     |- README.txt                                  (this file)
     |- Devlogs/
     |    |- Flow1_MonthlyVerificationDistribution_Devlog.txt
     |    |- Flow2_VerificationHandler_Devlog.txt
     |    |- Flow3_ReminderEscalationHandler_Devlog.txt
     |- Scripts/
     |    |- ClearReminderColumns.osts              (Office Script export)
     |- Specs/
     |    |- CDL_Verification_Agent_Spec.md         (original requirements)
     |- Diagrams/
          |- CDL_Verification_Flowchart.png         (process flowchart)

--------------------------------------------------------------------------------

# Roadmap / Open Items

## Production hardening (short term)
- [ ] Add as needed

## Reporting (medium term)
- [ ] Power BI dashboard sourced from the Excel file.
      Metrics: verification %, pending/verified/escalated counts,
      average days-to-verify by region, oldest unverified accounts,
      RDLs receiving the most escalations.
- [ ] Day 28 admin notification for accounts still stuck at Escalated.

## Process expansion (longer term)
- [ ] "Snooze" capability — CDL can defer verification by 7 days with
      a reason (vacation, contract transition, etc.).
- [ ] ServiceNow ticket integration on persistent escalation.
- [ ] Quarterly leadership one-pager on directory health.

## Platform evolution (when bandwidth allows)
- [ ] Migrate data source from Excel to Dataverse or SharePoint List.
      Reasons: proper null handling, real numeric types, native audit
      history, relational integrity, row-level security.

--------------------------------------------------------------------------------

# Contacts

  Owner:           Mohtashim Syed (mohtashim.syed@gainwelltechnologies.com)
  Manager:         Barnali Dash
  Skip Manager:    Himanshu Shekhar
