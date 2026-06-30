================================================================================
FLOW 3 — REMINDER & ESCALATION HANDLER | DEVELOPMENT LOG
================================================================================

# Flow Overview

- Flow Name:    Flow 3 - Reminder & Escalation Handler
- Platform:     Power Automate (Cloud Flow)
- Trigger:      Recurrence (daily, 08:00 Central Time)
- Environment:  [Insert environment name]
- Owner:        Mohtashim Syed
- Last Updated: 2026-06-30

--------------------------------------------------------------------------------

# Purpose

Scans the CDL Directory daily and sends graduated reminders to CDLs who
have not yet verified their account.

  Day 7+  with ReminderCount = 0 -> Reminder #1
  Day 14+ with ReminderCount = 1 -> Reminder #2 (warns of upcoming RDL CC)
  Day 21+ with ReminderCount = 2 -> Escalation (CC's Regional Delivery
                                    Leader and sets Status = Escalated)

This is the third flow in a three-flow system:
- Flow 1: Distributes monthly verification emails.
- Flow 2: Records the CDL's verification response.
- Flow 3 (this flow): Chases unverified accounts.

--------------------------------------------------------------------------------

# Trigger

- Type:       Recurrence
- Frequency:  Day
- Interval:   1
- Time:       08:00 (Central Time, US & Canada)

--------------------------------------------------------------------------------

# Data Source

- Connector:  Excel Online (Business)
- File:       CDL Directory - 5.19.2026.xlsx
- Location:   OneDrive for Business
- Table:      Table1

Key column for updates: Account

--------------------------------------------------------------------------------

# Flow Logic

For each row in the CDL Directory:
  1. Skip if Verification Status != "Pending".
  2. Skip if CDL Email is blank.
  3. Calculate daysSince from Verification SentDate to today.
  4. Switch on:
       (daysSince >= 21 AND ReminderCount = 2) -> 3 (Escalation)
       (daysSince >= 14 AND ReminderCount = 1) -> 2 (Reminder #2)
       (daysSince >= 7  AND ReminderCount = 0) -> 1 (Reminder #1)
       otherwise                                -> 0 (no action)

After any send:
  - Stamp the corresponding column (Reminder1Sent / Reminder2Sent /
    EscalationSent) with utcNow().
  - Increment ReminderCount.
  - On Escalation, set Verification Status = "Escalated".

--------------------------------------------------------------------------------

# Key Actions

1. Recurrence Trigger (daily, 08:00 CT)

2. List rows present in a table  — DateTime Format: ISO 8601

3. Apply to each (over body/value)

   3a. Compose: daysSince
       div(
         sub(
           ticks(utcNow()),
           ticks(items('Apply_to_each')?['Verification SentDate'])
         ),
         864000000000
       )

   3b. Condition (gatekeeper):
       and(
         equals(items('Apply_to_each')?['Verification Status'],'Pending'),
         not(empty(items('Apply_to_each')?['CDL Email']))
       )

   3c. Switch (On expression):
       if(and(greaterOrEquals(outputs('daysSince'),21),
              equals(int(items('Apply_to_each')?['ReminderCount']),2)),3,
       if(and(greaterOrEquals(outputs('daysSince'),14),
              equals(int(items('Apply_to_each')?['ReminderCount']),1)),2,
       if(and(greaterOrEquals(outputs('daysSince'),7),
              equals(int(items('Apply_to_each')?['ReminderCount']),0)),1,
       0)))

       Case 1 -> Send Reminder #1     -> Update Reminder1Sent, ReminderCount=1
       Case 2 -> Send Reminder #2     -> Update Reminder2Sent, ReminderCount=2
       Case 3 -> Send Escalation      -> Update EscalationSent,
                                         ReminderCount=3, Status=Escalated
       Default -> No action

--------------------------------------------------------------------------------

# Email Templates Summary

| Type        | Subject                                                        |
|-------------|----------------------------------------------------------------|
| Reminder #1 | Reminder: CDL Directory Verification Needed - [Account]        |
| Reminder #2 | Second Reminder: CDL Directory Verification Needed - [Account] |
| Escalation  | Escalation: CDL Directory Verification Overdue - [Account]     |

- All three include the VERIFY DIRECTORY button linking to the Power App.
- Reminder #2 includes a yellow callout warning that the next reminder
  will copy the RDL by name.
- Escalation CCs RDL Email and includes a red callout acknowledging the
  RDL has been copied.

--------------------------------------------------------------------------------

# Design Notes

- Excel Online returns numeric columns as strings. The Switch expression
  wraps ReminderCount references in int() to enable correct comparison.
  This was discovered during testing — without int(), all cases evaluate
  false and the flow silently does nothing.

- Case numbers represent the reminder being sent, NOT the row's current
  ReminderCount. Case 1 fires when ReminderCount is 0 (about to send first
  reminder). This is by design — the system always walks through reminders
  in order rather than skipping steps.

- The outer Condition (Verification Status = Pending) prevents
  re-escalating already-escalated rows. A row sits dormant after Escalation
  until either (a) Flow 2 marks it Verified, or (b) Flow 1 resets it for
  the next monthly cycle.

- ReminderCount is the source of truth for "which reminder is next."
  daysSince is the threshold trigger. Both must align before any send.

- Cadence is intentionally one reminder per day maximum per row. Same-day
  cascading (e.g., Reminder #1 then Reminder #2 in one run) is impossible
  by design.

--------------------------------------------------------------------------------

# Known Issues / Lessons Learned

- 2026-06-29: int() conversion required for ReminderCount comparisons
  because Excel connector returns numeric columns as strings.

- 2026-06-29: A row with Verification Status = "Escalated" will NOT
  re-trigger Case 3, because the outer Condition filters it out. This is
  correct behavior — discovered during Case 3 testing when an already-
  escalated test row appeared not to fire.

- 2026-06-29: Cases must be tested with appropriate ReminderCount state.
  Brand-new rows (ReminderCount = 0) always trigger Case 1 regardless
  of how old Verification SentDate is. To test Case 2/3, the row's
  ReminderCount must be manually pre-set to 1 or 2 respectively.

--------------------------------------------------------------------------------

# Dependencies

- Power App:   CDL Directory Verification App
- Flow 1:      Sets Verification SentDate, Verification Status = Pending,
               ReminderCount = 0 at the start of each cycle.
- Flow 2:      Sets Verification Status = Verified when CDL submits.
               Flow 3 skips these rows via the outer Condition.
- Excel File:  CDL Directory - 5.19.2026.xlsx
- Columns:     RDL Email (required for Escalation CC),
               Reminder1Sent, Reminder2Sent, EscalationSent (timestamps),
               ReminderCount (numeric, 0-3),
               Verification Status (Pending / Verified / Escalated).

--------------------------------------------------------------------------------

# Open Items / Next Steps

- [ ] Enable failure notifications on this flow.
- [ ] Add fallback handling for blank RDL Email at Escalation time
      (send to CDL only and flag the row, vs. skip, vs. default admin CC).
- [ ] Add a Day 28 admin notification for rows still stuck at Escalated
      one week after escalation fired.
- [ ] Configure run-after on Update a row so ReminderCount doesn't
      increment if the email action failed.

================================================================================
END OF LOG
================================================================================
