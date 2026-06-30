# CDL Directory Verification System — Test Plan

| | |
|---|---|
| **System** | CDL Directory Verification System |
| **Components** | Power App + 3 Power Automate Flows + Excel Source |
| **Owner** | Mohtashim Syed |
| **Last Updated** | 2026-06-30 |

---

## Test Environment Setup

Before running any test cases:

- [ ] Identify a test row in `CDL Directory - 5.19.2026.xlsx` with `CDL Email` set to the tester's email address.
- [ ] Note the test row's `Account` value (you'll need it to verify writes).
- [ ] Set the test row's `RDL Email` to the tester's email address (so Day 21 CC lands in the tester's inbox).
- [ ] Confirm the row is inside the Excel Table boundary.
- [ ] Confirm all three flows are turned **On** in Power Automate.
- [ ] Confirm the Power App is published and accessible at its app URL.

---

## Test Suite 1: Flow 1 — Monthly Distribution

### TC-1.1: Happy-path email distribution
**Preconditions:** Test row has `CDL Email` populated, `Verification Status` = anything.
**Steps:**
1. Manually trigger Flow 1.
2. Wait for the run to complete.
3. Check the tester's inbox.

**Expected Results:**
- [ ] Verification email arrives within 2 minutes.
- [ ] Subject line is `Monthly CDL Directory Verification Required - [Account]` *(Account substituted correctly)*.
- [ ] Email body renders the full directory snapshot in a formatted HTML table.
- [ ] `VERIFY DIRECTORY` button is present and styled in green.
- [ ] Clicking the button opens the Power App with `?account=[Account]` in the URL.
- [ ] In Excel, the test row shows:
  - `Verification Status` = `Pending`
  - `Verification SentDate` = timestamp within last 2 minutes
  - `ReminderCount` = `0`
  - `Reminder1Sent`, `Reminder2Sent`, `EscalationSent`, `Verification Timestamp` = blank *(not "False")*

### TC-1.2: Missing CDL Email is skipped
**Preconditions:** Pick a row, blank out its `CDL Email`, save Excel.
**Steps:**
1. Trigger Flow 1.
2. Open the run history.
3. Check the inbox.

**Expected Results:**
- [ ] No email sent to that row.
- [ ] `MissingEmails` counter incremented in flow run.
- [ ] Audit CSV in OneDrive contains entry: `{ Account, Status: "Missing CDL Email" }`.
- [ ] Other valid rows in the same run still received their emails.

### TC-1.3: Audit CSV is generated
**Preconditions:** Flow 1 ran in TC-1.1.
**Steps:**
1. Open OneDrive → `/Desktop/CDLVerification/`.

**Expected Results:**
- [ ] CSV file exists with name pattern `CDL_Verification_Audit_[timestamp].csv`.
- [ ] CSV contains one row per processed account.
- [ ] Status column shows `Email Sent` or `Missing CDL Email`.

### TC-1.4: Office Script clears cells to true blank
**Preconditions:** Manually set `Reminder1Sent` of a test row to a date value, save.
**Steps:**
1. Trigger Flow 1.
2. After completion, open Excel.

**Expected Results:**
- [ ] `Reminder1Sent` cell is now **truly empty** (not the string `"False"`, not a date).

---

## Test Suite 2: Power App — Verification Form

### TC-2.1: App opens to the correct account
**Preconditions:** Email from TC-1.1 received.
**Steps:**
1. Click `VERIFY DIRECTORY` in the email.

**Expected Results:**
- [ ] App opens in the browser.
- [ ] Gallery displays the directory fields for the Account in the URL.
- [ ] No fields from other accounts appear.

### TC-2.2: Direct URL launch (without email)
**Preconditions:** Tester knows their test Account code.
**Steps:**
1. Construct URL manually with `?account=[TestAccount]` appended.
2. Open in browser.

**Expected Results:**
- [ ] Same behavior as TC-2.1.

### TC-2.3: Missing `?account=` parameter
**Preconditions:** None.
**Steps:**
1. Open app URL without any account parameter.

**Expected Results:**
- [ ] App handles gracefully — either empty gallery, default account, or error message *(document actual behavior)*.

### TC-2.4: Edit a field and submit
**Preconditions:** App open to test account.
**Steps:**
1. In the gallery, find one row (e.g., AGM).
2. Type a new value into the text input.
3. Click `Submit`.

**Expected Results:**
- [ ] "Verification Submitted" toast appears.
- [ ] `VerificationResponses` table in Excel has new rows — one per gallery row.
- [ ] For the edited row, `CurrentEmployee` ≠ `NewEmployee`.
- [ ] `SubmittedBy` = tester's full name.
- [ ] `SubmittedDate` = within last 2 minutes.
- [ ] In `Table1`, the test row's `Verification Status` = `Verified`.
- [ ] `Verification Timestamp` = within last 2 minutes (ISO 8601).
- [ ] `Last Verified` = current Month Year (e.g., `June 2026`).

### TC-2.5: Submit without changes (pure verification)
**Preconditions:** App open to test account, no edits made.
**Steps:**
1. Click `Submit` immediately.

**Expected Results:**
- [ ] "Verification Submitted" toast appears.
- [ ] `VerificationResponses` rows written with `CurrentEmployee` = `NewEmployee` (or `NewEmployee` blank).
- [ ] In `Table1`, the test row's `Verification Status` = `Verified`.
- [ ] Flow 2 still completes successfully (ChangesJSON = `"[]"`).

### TC-2.6: Submit multiple field edits
**Preconditions:** App open to test account.
**Steps:**
1. Edit 3 different gallery rows with new values.
2. Click `Submit`.

**Expected Results:**
- [ ] All 3 edits appear in `VerificationResponses` as separate rows.
- [ ] All non-edited rows are also logged with `CurrentEmployee` = `NewEmployee`.
- [ ] `Table1` row marked Verified.

---

## Test Suite 3: Flow 2 — Verification Handler

### TC-3.1: Flow 2 fires on Power Apps Submit
**Preconditions:** TC-2.4 just completed.
**Steps:**
1. Open Flow 2 run history.

**Expected Results:**
- [ ] One successful run logged within the last 2 minutes.
- [ ] Inputs show correct Account, Region, ChangesJSON.
- [ ] Update a row action succeeded.

### TC-3.2: Flow 2 handles unknown account
**Preconditions:** None.
**Steps:**
1. Manually run Flow 2 from Power Automate with `Account` = `INVALID_TEST_ACCOUNT`.

**Expected Results:**
- [ ] Update a row fails gracefully with `row not found` error.
- [ ] No data corruption in `Table1`.

---

## Test Suite 4: Flow 3 — Reminders & Escalation

### TC-4.1: Reminder #1 fires at Day 7
**Preconditions:** Test row state:
- `Verification Status` = `Pending`
- `Verification SentDate` = 8 days ago (ISO 8601)
- `ReminderCount` = `0`

**Steps:**
1. Manually trigger Flow 3.
2. Check inbox.

**Expected Results:**
- [ ] Email with subject `Reminder: CDL Directory Verification Needed - [Account]` arrives.
- [ ] Email body shows current date and original send date correctly.
- [ ] In Excel: `ReminderCount` = `1`, `Reminder1Sent` = timestamp within last 2 minutes.

### TC-4.2: Reminder #2 fires at Day 14
**Preconditions:** Test row state:
- `Verification Status` = `Pending`
- `Verification SentDate` = 15 days ago
- `ReminderCount` = `1`

**Steps:**
1. Trigger Flow 3.
2. Check inbox.

**Expected Results:**
- [ ] Email subject = `Second Reminder: CDL Directory Verification Needed - [Account]`.
- [ ] Email body includes the **yellow callout** warning about RDL CC.
- [ ] Email body names the RDL by name *(from `Regional Delivery Leader` column)*.
- [ ] In Excel: `ReminderCount` = `2`, `Reminder2Sent` = timestamp.

### TC-4.3: Escalation fires at Day 21
**Preconditions:** Test row state:
- `Verification Status` = `Pending`
- `Verification SentDate` = 22 days ago
- `ReminderCount` = `2`
- `RDL Email` populated

**Steps:**
1. Trigger Flow 3.
2. Check inbox (both `To` and `CC`).

**Expected Results:**
- [ ] Email subject = `Escalation: CDL Directory Verification Overdue - [Account]`.
- [ ] Email To = CDL Email.
- [ ] Email CC = RDL Email.
- [ ] Email body includes the **red callout** acknowledging RDL is copied.
- [ ] In Excel: `ReminderCount` = `3`, `EscalationSent` = timestamp, `Verification Status` = `Escalated`.

### TC-4.4: Verified rows are skipped
**Preconditions:** Test row state:
- `Verification Status` = `Verified`
- `Verification SentDate` = 8 days ago
- `ReminderCount` = `0`

**Steps:**
1. Trigger Flow 3.

**Expected Results:**
- [ ] No email sent for this row.
- [ ] Row state unchanged.

### TC-4.5: Escalated rows are skipped (no double-escalation)
**Preconditions:** Test row state:
- `Verification Status` = `Escalated`
- `ReminderCount` = `3`

**Steps:**
1. Trigger Flow 3.

**Expected Results:**
- [ ] No email sent for this row.
- [ ] Row state unchanged.

### TC-4.6: Missing CDL Email is skipped
**Preconditions:** Test row state:
- `Verification Status` = `Pending`
- `CDL Email` = blank
- `Verification SentDate` = 8 days ago

**Steps:**
1. Trigger Flow 3.

**Expected Results:**
- [ ] No email sent for this row.
- [ ] No errors in flow run.

### TC-4.7: Too-early row is skipped
**Preconditions:** Test row state:
- `Verification Status` = `Pending`
- `Verification SentDate` = 3 days ago
- `ReminderCount` = `0`

**Steps:**
1. Trigger Flow 3.

**Expected Results:**
- [ ] No email sent for this row.
- [ ] Row state unchanged.

### TC-4.8: Same-day cascading is prevented
**Preconditions:** Test row state:
- `Verification Status` = `Pending`
- `Verification SentDate` = 30 days ago *(way past all thresholds)*
- `ReminderCount` = `0`

**Steps:**
1. Trigger Flow 3.

**Expected Results:**
- [ ] Only Reminder #1 is sent *(not #2 or Escalation)*.
- [ ] `ReminderCount` = `1` *(not `2` or `3`)*.
- [ ] System walks through reminders one per day even when overdue.

---

## Test Suite 5: End-to-End Integration

### TC-5.1: Full happy-path lifecycle
**Steps:**
1. Trigger Flow 1 → email received.
2. Click VERIFY DIRECTORY → app opens.
3. Make one edit → click Submit.
4. Verify Excel updated.
5. Next day, trigger Flow 3.

**Expected Results:**
- [ ] Email received from Flow 1.
- [ ] App opens to correct account.
- [ ] Submit succeeds, toast shown.
- [ ] `Verification Status` = `Verified`.
- [ ] Flow 3 skips this row (no reminder sent).

### TC-5.2: Full escalation path
**Steps:**
1. Trigger Flow 1 → email received, do nothing.
2. Skip 7 days (or manually set `Verification SentDate` to 8 days ago).
3. Trigger Flow 3 → Reminder #1 received.
4. Set `Verification SentDate` to 15 days ago, leave `ReminderCount = 1`.
5. Trigger Flow 3 → Reminder #2 received.
6. Set `Verification SentDate` to 22 days ago, leave `ReminderCount = 2`.
7. Trigger Flow 3 → Escalation received with RDL CC'd.

**Expected Results:**
- [ ] All four emails received in order *(Original, R1, R2, Escalation)*.
- [ ] Each one writes its corresponding timestamp column.
- [ ] Final state: `Verification Status` = `Escalated`, `ReminderCount` = `3`.

---

## Test Suite 6: Data Integrity

### TC-6.1: No `"False"` values written to any cell
**Preconditions:** Run all test cases above.
**Steps:**
1. Open Excel.
2. Search the columns `Reminder1Sent`, `Reminder2Sent`, `EscalationSent`, `Verification Timestamp` for the literal string `False`.

**Expected Results:**
- [ ] No `"False"` strings found anywhere.

### TC-6.2: No data corruption on other accounts
**Preconditions:** Multiple test rows have been used.
**Steps:**
1. Compare `Table1` state before and after the test run.
2. Verify only intended rows were updated.

**Expected Results:**
- [ ] All non-test rows are unchanged.

### TC-6.3: Concurrent submission test
**Preconditions:** Two testers with two different account assignments.
**Steps:**
1. Both testers open the app at the same time.
2. Both submit within seconds of each other.

**Expected Results:**
- [ ] Both rows updated correctly in `Table1`.
- [ ] No row-overwrite or race condition.

---

## Sign-off

| Tester | Date | Result |
|---|---|---|
| | | ☐ Pass &nbsp;&nbsp; ☐ Fail |

**Bugs / Issues Found:**

_Document any failures here with TC ID, expected vs. actual, screenshots._

---

*End of Test Plan*
