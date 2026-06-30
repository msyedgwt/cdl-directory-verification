# CDL Directory Verification App — Power App

> **Development Log**

| | |
|---|---|
| **App Name** | CDL Directory Verification App |
| **Platform** | Power Apps (Canvas App) |
| **App ID** | `0c819325-4583-4266-936f-4a8856f8ad82` |
| **Environment ID** | `default-c663f89c-ef9b-418f-bd3d-41e46c0ce068` |
| **Owner** | Mohtashim Syed |
| **Last Updated** | 2026-06-30 |

---

## Purpose

Provides the user-facing interface of the CDL Directory Verification System. When a CDL receives a verification email from Flow 1 (or a reminder from Flow 3) and clicks the **VERIFY DIRECTORY** button, the app opens to *their specific account row*, displays the current directory record, lets them edit any fields, and submits the verification with a single button click.

This is the front-end of a four-component system:

- **Flow 1:** Distributes monthly verification emails *(server)*.
- **Flow 2:** Records the response when the CDL submits the form *(server)*.
- **Flow 3:** Handles reminders and escalation *(server)*.
- **Power App (this component):** User-facing verification form *(client)*.

---

## App Launch URL

```
https://apps.powerapps.com/play/e/default-c663f89c-ef9b-418f-bd3d-41e46c0ce068
  /a/0c819325-4583-4266-936f-4a8856f8ad82
  ?tenantId=c663f89c-ef9b-418f-bd3d-41e46c0ce068
  &hint=5f2348b8-7f6d-4ef0-a3c2-07f5260d22ed
  &sourcetime=1782783469751
  &account=[Account]
```

The `?account=[Account]` query parameter is appended dynamically by each Flow 1 and Flow 3 email so the app opens directly to the CDL's row. The Flow's `Send an email (V2)` HTML body substitutes the Account code at send time.

---

## Data Sources

| Source | Type | Purpose |
|---|---|---|
| `Table1` *(CDL Directory)* | Excel Online (Business) | Read-only directory snapshot for display in the gallery |
| `VerificationResponses` | Excel Online (Business) | Change log — every Submit writes one row per gallery item |
| **Flow 2 - Verification Handler** | Power Automate flow | Called on Submit; updates the master directory row |

> ⚠️ The app and Flow 2 must be in the **same environment** or Flow 2 won't be selectable from the app.

---

## Screens

### Main Screen (verification form)

| Element | Purpose |
|---|---|
| **Header banner** | App title and account context |
| **`CDLGallery`** | Editable gallery showing each directory field for the user's account |
| **`txtNewEmployee`** *(per gallery row)* | Text input allowing the user to overwrite the current value |
| **`Submit` button** | Footer button — fires the full submission sequence |

### Thank-You Screen *(optional, post-submit)*
Confirms submission and offers a close/return action.

---

## App.OnStart Logic

Pulls the `account` query parameter from the launch URL into a context variable so the gallery and any related controls can filter to the right row.

```powerapps
Set(varAccount, Param("account"));
Set(varRegion,
    LookUp(Table1, Account = varAccount).Region
)
```

---

## Submit Button OnSelect Formula

Current implementation:

```powerapps
// 1. Save all gallery edits to the VerificationResponses change log
ForAll(
    CDLGallery.AllItems As CDLRecord,
    Patch(
        VerificationResponses,
        Defaults(VerificationResponses),
        {
            Account:         CDLRecord.Account,
            Region:          CDLRecord.Region,
            Position:        CDLRecord.Position,
            CurrentEmployee: CDLRecord.Employee,
            NewEmployee:     CDLRecord.txtNewEmployee.Text,
            SubmittedBy:     User().FullName,
            SubmittedDate:   Now()
        }
    )
);

// 2. Fire Flow 2 to update the master directory row
'Flow2-VerificationHandler'.Run(
    First(CDLGallery.AllItems).Account,
    First(CDLGallery.AllItems).Region,
    JSON(
        ForAll(
            Filter(
                CDLGallery.AllItems,
                !IsBlank(txtNewEmployee.Text) && txtNewEmployee.Text <> Employee
            ),
            {
                Position:    Position,
                OldEmployee: Employee,
                NewEmployee: txtNewEmployee.Text
            }
        ),
        JSONFormat.IgnoreBinaryData
    )
);

// 3. Notify the user
Notify(
    "Verification Submitted",
    NotificationType.Success
)
```

### What this formula does

1. **Patches each gallery row** into the `VerificationResponses` change log *(one row per directory field)*, capturing both old and new values plus who submitted and when.
2. **Calls Flow 2**, passing the Account, Region, and a JSON payload describing only the rows the user actually changed.
3. **Notifies the user** that the verification was submitted successfully.

---

## VerificationResponses Schema

| Column | Type | Source |
|---|---|---|
| `Account` | Text | From gallery row |
| `Region` | Text | From gallery row |
| `Position` | Text | Directory field label *(e.g., "AGM", "PMO Lead")* |
| `CurrentEmployee` | Text | Original value at time of edit |
| `NewEmployee` | Text | User's submitted value *(may match `CurrentEmployee` if unchanged)* |
| `SubmittedBy` | Text | `User().FullName` |
| `SubmittedDate` | DateTime | `Now()` |

---

## Design Notes

- **Manual Submit chosen over autosave.** Early exploration considered firing Flow 2 on every cell change (`OnChange` of each text input). This was rejected because the target users are busy delivery leads who fill out one field at a time across hours/days — autosave would generate dozens of partial submissions per CDL per cycle. A single explicit Submit gives clean transactional semantics and a clear audit point.

- **`User().FullName` over `User().Email`.** The change log is intended for human readers reviewing what changed. Names render better than email addresses in reports. The email is recoverable from the Account+SubmittedDate join if ever needed.

- **JSON payload of only actual changes.** The Submit formula filters `CDLGallery.AllItems` down to rows where `txtNewEmployee.Text` is non-blank AND differs from `Employee` *(the current value)*. A pure verification with zero edits results in `"[]"` being passed to Flow 2 — a meaningful signal of *"confirmed, no changes."*

- **One Submit, both writes.** The formula deliberately writes to `VerificationResponses` *(change log)* and triggers Flow 2 *(directory update)* in a single user action. If either fails, the user sees the failure; there is no silent split-state risk.

- **No row filtering inside the gallery.** The gallery's `Items` property is set on the screen's `OnVisible` (or App.OnStart) using `varAccount`. This keeps the Submit formula's `First(CDLGallery.AllItems)` safe — every row in the gallery belongs to the same account.

---

## Known Issues / Lessons Learned

### 2026-06-21 — `'colDrafts' isn't recognized`, `'Account' isn't recognized`
**Cause:** Collection-based draft state pattern attempted before defining the collection in `App.OnStart`. The compiler couldn't resolve the names at design time.
**Resolution:** Pivoted to direct `Patch` writes against `VerificationResponses` on Submit, eliminating the need for a draft collection entirely.

### 2026-06-21 — Autosave-on-edit approach abandoned
**Cause:** Multiple attempts to make text inputs autosave (`OnChange = Patch(...)`) produced duplicate rows, partial submissions, and a poor UX where the user couldn't tell when a save had occurred.
**Resolution:** Reverted to single Submit button. All edits stay in memory in the gallery until the user explicitly submits. Documented in Design Notes.

### 2026-06-25 — Repeated requests to add more Submit buttons
**Cause:** Iterative design exploration considered per-row Submit buttons inside the gallery.
**Resolution:** Held the line on a single footer Submit. Per-row Submit complicates the audit trail and confuses users about what "submit" actually finalizes.

### 2026-06-29 — `'Run' is an unknown or unsupported function in namespace 'CDLVerificationFlow2'`
**Cause:** Flow 2 existed in Power Automate but was not attached to the Power App via the Power Automate pane in Studio.
**Resolution:** Opened Power Apps Studio → left rail → Power Automate → **+ Add flow** → selected Flow 2. After adding, IntelliSense surfaced the correct namespace `'Flow2-VerificationHandler'` *(with quotes due to the hyphen)*.

### Environment alignment requirement
**Cause:** Power Apps and Power Automate must be in the **same environment** for the flow to be selectable from the app.
**Resolution:** Verified both reside in the default Gainwell environment before publishing.

---

## Dependencies

| Type | Resource |
|---|---|
| **Companion Flow** | Flow 1 - Monthly Verification Distribution *(generates URL with `?account=` param)* |
| **Companion Flow** | Flow 2 - Verification Handler *(called from Submit button)* |
| **Companion Flow** | Flow 3 - Reminder & Escalation Handler *(generates URL with `?account=` param)* |
| **Excel File** | `CDL Directory - 5.19.2026.xlsx` — `Table1` *(read), `VerificationResponses` (write)* |
| **Connector** | Office 365 Users *(for `User().FullName`)* |

---

## Open Items / Next Steps

- [ ] Add a **Thank-You screen** with `Navigate(ThankYouScreen, ScreenTransition.Fade)` after Submit, replacing the toast-only confirmation.
- [ ] Add **error handling** for Flow 2 failure *(catch with `IfError()`, surface a banner with a "Retry" option)*.
- [ ] Add a **"Submitting..."** spinner and disable the Submit button during the flow call to prevent double-clicks *(pattern: `DisplayMode = If(varSubmitting, Disabled, Edit)`)*.
- [ ] Add a **filter/search** to the gallery for accounts with many directory fields, to help CDLs locate the row they want to edit quickly.
- [ ] Build a separate **admin view** for the Directory Owner *(read-only directory snapshot + escalation status summary)*.
- [ ] Migrate `VerificationResponses` from Excel to **Dataverse** for proper schema, audit, and concurrency handling.

---

*End of log*
