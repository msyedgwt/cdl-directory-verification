# Flow 3 — Reminder & Escalation Handler

> **Development Log**

| | |
|---|---|
| **Flow Name** | Flow 3 - Reminder & Escalation Handler |
| **Platform** | Power Automate (Cloud Flow) |
| **Trigger** | Recurrence *(daily, 08:00 Central Time)* |
| **Owner** | Mohtashim Syed |
| **Last Updated** | 2026-07-01 |

---

## Purpose

Scans the CDL Directory daily and sends graduated reminders to CDLs who have not yet verified their account.

| Trigger Condition | Action |
|---|---|
| `daysSince >= 7` AND `ReminderCount = 0` | Reminder #1 |
| `daysSince >= 14` AND `ReminderCount = 1` | Reminder #2 *(warns of upcoming RDL CC)* |
| `daysSince >= 21` AND `ReminderCount = 2` | Escalation *(CCs Regional Delivery Leader, sets `Status = Escalated`)* |

This is the third flow in a three-flow system:

- **Flow 1:** Distributes monthly verification emails.
- **Flow 2:** Records the CDL's verification response.
- **Flow 3 (this flow):** Chases unverified accounts.

---

## Trigger

| Setting | Value |
|---|---|
| **Type** | Recurrence |
| **Frequency** | Day |
| **Interval** | 1 |
| **Time** | 08:00 *(Central Time, US & Canada)* |

---

## Data Source

| | |
|---|---|
| **Connector** | Excel Online (Business) |
| **File** | `CDL Directory - 5.19.2026.xlsx` |
| **Location** | OneDrive for Business |
| **Table** | `Table1` |
| **Key column for updates** | `Account` |

---

## Flow Logic

### For each row in the CDL Directory

1. **Skip** if `Verification Status` ≠ `"Pending"`.
2. **Skip** if `CDL Email` is blank.
3. **Calculate** `daysSince` from `Verification SentDate` to today.
4. **Switch** on:
   - `(daysSince >= 21 AND ReminderCount = 2)` → `3` *(Escalation)*
   - `(daysSince >= 14 AND ReminderCount = 1)` → `2` *(Reminder #2)*
   - `(daysSince >= 7  AND ReminderCount = 0)` → `1` *(Reminder #1)*
   - otherwise → `0` *(no action)*

### After any send

- Stamp the corresponding column *(`Reminder1Sent` / `Reminder2Sent` / `EscalationSent`)* with `utcNow()`.
- Increment `ReminderCount`.
- On **Escalation**, also set `Verification Status` = `"Escalated"`.

---

## Key Actions

### 1. Recurrence Trigger
Daily at 08:00 CT.

### 2. List rows present in a table
**DateTime Format:** `ISO 8601`

### 3. Apply to each *(over `body/value`)*

#### 3a. Compose: `daysSince`

```text
div(
  sub(
    ticks(utcNow()),
    ticks(items('Apply_to_each')?['Verification SentDate'])
  ),
  864000000000
)
```

#### 3b. Condition *(gatekeeper)*

```text
and(
  equals(items('Apply_to_each')?['Verification Status'],'Pending'),
  not(empty(items('Apply_to_each')?['CDL Email']))
)
```

#### 3c. Switch *(On expression)*

```text
if(and(greaterOrEquals(outputs('daysSince'),21),
       equals(int(items('Apply_to_each')?['ReminderCount']),2)),3,
if(and(greaterOrEquals(outputs('daysSince'),14),
       equals(int(items('Apply_to_each')?['ReminderCount']),1)),2,
if(and(greaterOrEquals(outputs('daysSince'),7),
       equals(int(items('Apply_to_each')?['ReminderCount']),0)),1,
0)))
```

**Switch Cases:**

| Case | Returns | Action | Updates |
|---|---|---|---|
| **Case 1** | `1` | Send Reminder #1 | `Reminder1Sent` = `utcNow()`, `ReminderCount` = `1` |
| **Case 2** | `2` | Send Reminder #2 | `Reminder2Sent` = `utcNow()`, `ReminderCount` = `2` |
| **Case 3** | `3` | Send Escalation | `EscalationSent` = `utcNow()`, `ReminderCount` = `3`, `Verification Status` = `Escalated` |
| **Default** | `0` | No action | *(none)* |

---

## Email Templates Summary

| Type | Subject |
|---|---|
| **Reminder #1** | `Reminder: CDL Directory Verification Needed - [Account]` |
| **Reminder #2** | `Second Reminder: CDL Directory Verification Needed - [Account]` |
| **Escalation** | `Escalation: CDL Directory Verification Overdue - [Account]` |

- All three include the **VERIFY DIRECTORY** button linking to the Power App.
- **Reminder #2** includes a yellow callout warning that the next reminder will copy the RDL by name.
- **Escalation** CCs `RDL Email` and includes a red callout acknowledging the RDL has been copied.

---

## Design Notes

- Excel Online returns **numeric columns as strings**. The Switch expression wraps `ReminderCount` references in `int()` to enable correct comparison. This was discovered during testing — without `int()`, all cases evaluate `false` and the flow silently does nothing.

- **Case numbers represent the reminder being sent, NOT the row's current `ReminderCount`.** Case 1 fires when `ReminderCount` is `0` *(about to send the first reminder)*. This is by design — the system always walks through reminders in order rather than skipping steps.

- The **outer Condition** (`Verification Status = Pending`) prevents re-escalating already-escalated rows. A row sits dormant after Escalation until either:
  - **(a)** Flow 2 marks it `Verified`, or
  - **(b)** Flow 1 resets it for the next monthly cycle.

- `ReminderCount` is the **source of truth for "which reminder is next."** `daysSince` is the **threshold trigger**. Both must align before any send.

- Cadence is intentionally **one reminder per day maximum per row**. Same-day cascading *(e.g., Reminder #1 then Reminder #2 in one run)* is impossible by design.

---

## Known Issues / Lessons Learned

### 2026-06-29 — `int()` conversion required for `ReminderCount` comparisons
**Cause:** Excel connector returns numeric columns as strings, so `equals("0", 0)` evaluates `false`.
**Resolution:** Wrapped all `ReminderCount` references in `int()` inside the Switch expression.

### 2026-06-29 — Already-escalated rows do not re-trigger Case 3
**Cause:** The outer Condition filters out any row where `Verification Status` ≠ `"Pending"`.
**Resolution:** This is correct behavior. Discovered during Case 3 testing when an already-escalated test row appeared not to fire. To test Case 3, the test row must have `Verification Status = "Pending"`, `ReminderCount = 2`, and a `Verification SentDate` 21+ days old.

### 2026-06-29 — Cases must be tested with appropriate `ReminderCount` state
**Cause:** Brand-new rows *(`ReminderCount = 0`)* always trigger Case 1 regardless of how old `Verification SentDate` is.
**Resolution:** To test Case 2 or Case 3, the row's `ReminderCount` must be manually pre-set to `1` or `2` respectively before running the flow.

---

## Dependencies

| Type | Resource |
|---|---|
| **Power App** | CDL Directory Verification App |
| **Flow 1** | Sets `Verification SentDate`, `Verification Status = Pending`, `ReminderCount = 0` at the start of each cycle. |
| **Flow 2** | Sets `Verification Status = Verified` when CDL submits. Flow 3 skips these rows via the outer Condition. |
| **Excel File** | `CDL Directory - 5.19.2026.xlsx` |

### Required Columns

| Column | Purpose |
|---|---|
| `RDL Email` | Required for Escalation CC |
| `Reminder1Sent`, `Reminder2Sent`, `EscalationSent` | Timestamps written after each send |
| `ReminderCount` | Numeric, `0–3` |
| `Verification Status` | `Pending` / `Verified` / `Escalated` |

---

## Open Items / Next Steps

- [ ] Add a Day 28 admin notification for rows still stuck at `Escalated` one week after escalation fired.

---

*End of log*
