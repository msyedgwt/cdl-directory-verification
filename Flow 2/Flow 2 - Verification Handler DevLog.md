FLOW 2 — VERIFICATION HANDLER | DEVELOPMENT LOG
================================================================================

# Flow Overview

- Flow Name:    Flow 2 - Verification Handler
- Platform:     Power Automate (Cloud Flow)
- Trigger:      PowerApps (V2)
- Owner:        Mohtashim Syed

--------------------------------------------------------------------------------

# Purpose

Handles the "Verify" submission from the CDL Directory Verification Power App.
When a CDL clicks Submit, this flow updates that account's row in the CDL
Directory Excel table to mark it verified for the current month.

This is the second flow of a multi-flow system:
- Flow 1 distributes the monthly verification emails.
- Flow 2 (this flow) records the response when the CDL verifies.

--------------------------------------------------------------------------------

# Trigger Inputs

| Name         | Type | Source                                      |
|--------------|------|---------------------------------------------|
| Account      | Text | First(CDLGallery.AllItems).Account          |
| Region       | Text | First(CDLGallery.AllItems).Region           |
| ChangesJSON  | Text | JSON payload of edited rows (may be "[]")   |

--------------------------------------------------------------------------------

# Data Source

- Connector:  Excel Online (Business)
- File:       CDL Directory - 5.19.2026.xlsx (test file)
- Location:   OneDrive for Business
- Table:      Table1 (formatted as Excel Table — required by connector)

Key column for updates: Account

--------------------------------------------------------------------------------

# Actions

1. PowerApps (V2) Trigger
   - Receives Account, Region, ChangesJSON from the Power App.

2. List rows present in a table
   - Pulls all rows from the CDL Directory table.

3. Filter array
   - From:  body('List_rows_present_in_a_table')?['value']
   - Query: item()?['Account'] is equal to triggerBody()?['text']
   - Returns the single row matching the submitted Account.

4. Condition
   - Expression: length(body('Filter_array')) is greater than 0
   - True branch  -> Update a row (see below)
   - False branch -> No action (account not found in directory)

5. Update a row (True branch only)
   - Key Column:        Account
   - Key Value:         first(body('Filter_array'))?['Account']
   - Fields updated:
       * Verification Timestamp -> utcNow()
       * Last Verified          -> formatDateTime(utcNow(),'MMMM yyyy')
       * VerificationStatus     -> Verified
       * ReminderCount          -> 0

--------------------------------------------------------------------------------

# Design Notes

- Excel column names must match the Excel Table headers exactly
  (case-sensitive, space-sensitive). Use Dynamic Content, not hand-typed names.

- The Condition step is a guard, not business logic. It prevents a runtime
  error when first(body('Filter_array'))?['Account'] is called on an empty
  array. If the Update step is rewritten to key directly off triggerBody(),
  the List rows + Filter array + Condition chain can be removed entirely.

- ChangesJSON is received but not currently written to Excel. Reserved for
  future audit logging (Flow 3 or expansion of this flow).

--------------------------------------------------------------------------------

# Known Issues / Lessons Learned

- 2026-06-29: "[Expression.Error] The column 'Verification Timestamp' of the
  table wasn't found." Root cause: column was not inside the Excel Table
  boundary. Fix: Table Design -> Resize Table to include the new columns,
  then refresh dynamic content in the Update a row action.

- Flow only appears in Power Apps after being added via the Power Automate
  pane inside Power Apps Studio. Existing in Power Automate is not enough.

- Power Apps and Power Automate must be in the same environment for the
  flow to be selectable from the app.

--------------------------------------------------------------------------------

# Dependencies

- Power App:     CDL Directory Verification App
- Companion:    Flow 1 - Monthly Verification Distribution
- Excel File:   CDL Directory - 5.19.2026.xlsx (OneDrive for Business)
- Change Log:   VerificationResponses table (written from Power Apps, not
                from this flow)

--------------------------------------------------------------------------------

# Open Items / Next Steps

- [ ] Decide whether to collapse List rows + Filter array + Condition into a
      single Update a row keyed off triggerBody()?['text'].
- [ ] Add a failure branch (Configure run after) that logs failures to a
      separate Errors table.
- [ ] Persist ChangesJSON to an audit table for change history.
- [ ] Add Teams/email notification to the Regional Delivery Leader when a
      verification is recorded.
