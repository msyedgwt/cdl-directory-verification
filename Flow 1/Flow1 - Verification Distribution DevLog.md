# CDL Directory Verification Automation

# Development Log – Flow 1: Verification Distribution

## Purpose

Flow 1 is responsible for distributing monthly verification emails to all Client Delivery Leads (CDLs) listed in the CDL Directory. The flow retrieves account records from Excel, validates required fields, sends a customized verification email containing the current directory information for each account, updates verification tracking fields, and generates an audit log for reporting purposes.

---

# Flow Information

**Flow Name**

CDL Monthly Verification Distribution

**Flow Type**

Scheduled Cloud Flow

**Frequency**

Monthly

**Interval**

1

---

# Data Source

## Connector

Excel Online (Business)

## Action

List rows present in a table

## Purpose

Retrieves all account records from the CDL Directory workbook.

## Expected Columns

* Account Name
* CDL
* CDL Email
* Region
* Regional Delivery Leader
* Account Architect
* AGM
* Contract Manager
* CX Leader
* Information Security Lead
* Infrastructure Delivery Lead
* Legal Contact
* PMO Lead
* Primary Solution Implemented
* Procurement Contact
* Quality Manager
* Change Manager
* Release Manager
* Sector Leader
* Systems Manager
* Test Manager
* Last Verified
* VerificationStatus
* VerificationSentDate
* ReminderCount

---

# Variable Initialization

The following variables are initialized before entering the processing loop.

## AuditLog

### Action

Initialize Variable

### Configuration

```json
{
  "Name": "AuditLog",
  "Type": "Array",
  "Value": []
}
```

### Purpose

Stores processing results for audit reporting.

Examples:

```json
{
  "Account":"AK",
  "Status":"Email Sent"
}
```

```json
{
  "Account":"OH",
  "Status":"Missing CDL Email"
}
```

---

## MissingEmails

### Action

Initialize Variable

### Configuration

```json
{
  "Name": "MissingEmails",
  "Type": "Integer",
  "Value": 0
}
```

### Purpose

Tracks accounts missing required email information.

---

## TotalAccounts

### Action

Initialize Variable

### Configuration

```json
{
  "Name": "TotalAccounts",
  "Type": "Integer",
  "Value": 0
}
```

### Purpose

Tracks the total number of account records processed.

---

# Apply to Each Loop

### Source

value

(from List rows present in a table)

### Purpose

Processes each account individually.

Flow structure:

```text
For Each Account
    Validate Required Fields
    Generate Verification URL
    Send Email
    Update Tracking Fields
    Write Audit Log
```

---

# Account Counter

### Action

Increment Variable

### Variable

TotalAccounts

### Value

1

### Purpose

Tracks total records processed during the run.

---

# Required Field Validation

### Action

Condition

### Expression

```text
or(
 empty(items('Apply_to_each')?['Account Name']),
 empty(items('Apply_to_each')?['CDL Email'])
)
```

### Purpose

Prevents sending verification requests for incomplete records.

---

## YES Branch

Condition met:

```text
Account Name missing
OR
CDL Email missing
```

### Increment Missing Email Counter

```text
MissingEmails + 1
```

### Append Audit Log Entry

```json
{
  "Account":"@{items('Apply_to_each')?['Account Name']}",
  "Status":"Missing CDL Email"
}
```

### Result

Account is skipped and no email is sent.

---

## NO Branch

Required information exists.

Continue processing.

---

# Verification URL Generation

### Action

Compose

---

# Verification Email

### Action

Send an Email (V2)

## Recipient

CDL Email

## Subject

```text
Monthly CDL Directory Verification Required – @{items('Apply_to_each')?['Account Name']}
```

## Email Body Structure

### Greeting

```html
Hello @{items('Apply_to_each')?['CDL']},
```

### Directory Snapshot Table

The email displays the current directory information in HTML table format.

Example row:

```html
<tr>
<td style="background:#f2f2f2;font-weight:bold;padding:8px;border:1px solid #ddd;">
Regional Delivery Leader
</td>
<td style="padding:8px;border:1px solid #ddd;">
@{items('Apply_to_each')?['Regional Delivery Leader']}
</td>
</tr>
```

### Included Directory Fields

* Account
* CDL
* Region
* Regional Delivery Leader
* Account Architect
* AGM
* Contract Manager
* CX Leader
* Information Security Lead
* Infrastructure Delivery Lead
* Legal Contact
* PMO Lead
* Primary Solution Implemented
* Procurement Contact
* Quality Manager
* Change Manager
* Release Manager
* Sector Leader
* Systems Manager
* Test Manager
* Last Verified

### Verification Button

```html
<a href="https://apps.powerapps.com/play/e/default-c663f89c-ef9b-418f-bd3d-41e46c0ce068/a/0c819325-4583-4266-936f-4a8856f8ad82?tenantId=c663f89c-ef9b-418f-bd3d-41e46c0ce068&amp;hint=5f2348b8-7f6d-4ef0-a3c2-07f5260d22ed&amp;sourcetime=1781820026457&amp;account=@{items('Apply_to_each')?['Account']}" style="background:#107c10;color:white;padding:10px 20px;text-decoration:none;border-radius:4px;"
style="
background:#0078D4;
color:white;
padding:12px 20px;
text-decoration:none;
border-radius:4px;">
VERIFY DIRECTORY
</a>

OR

<a href="https://apps.powerapps.com/play/e/default-c663f89c-ef9b-418f-bd3d-41e46c0ce068/a/0c819325-4583-4266-936f-4a8856f8ad82?tenantId=c663f89c-ef9b-418f-bd3d-41e46c0ce068&amp;hint=5f2348b8-7f6d-4ef0-a3c2-07f5260d22ed&amp;sourcetime=1781820026457&amp;account=@{items('Apply_to_each')?['Account']}" style="background:#107c10;color:white;padding:10px 20px;text-decoration:none;border-radius:4px;">
VERIFY DIRECTORY
</a>
```

### Closing Message

```html
If updates are needed, please update the directory first.

Thank you,
Directory Verification Automation
```

---

# Update Verification Tracking

Immediately after successful email delivery.

### Action

Update a Row

## Key Column

RecordID

## Key Value

```text
@{items('Apply_to_each')?['RecordID']}
```

## Updated Fields

### VerificationStatus

```text
Pending
```

### VerificationSentDate

```text
utcNow()
```

### ReminderCount

```text
0
```

### Purpose

Marks the record as awaiting verification.

---

# Audit Logging

After email transmission:

### Append to AuditLog

```json
{
  "Account":"@{items('Apply_to_each')?['Account Name']}",
  "Status":"Email Sent"
}
```

### Purpose

Captures successful email deliveries.

---

# Post-Processing

After the Apply to Each loop completes:

### Create CSV Table

#### Input

```text
variables('AuditLog')
```

#### Format

```text
CSV
```

### Purpose

Converts audit records into downloadable report format.

---

# Create Audit File

### Action

Create File

### Location

OneDrive for Business

### Folder

```text
/Desktop/CDLVerification
```

### File Name

```text
CDL_Verification_Audit_@{formatDateTime(utcNow(),'yyyy-MM-dd-HHmmss')}.csv
```

### File Content

```text
Output from Create CSV Table
```

### Purpose

Stores a timestamped audit record for each monthly distribution run.

---

# Final Output

At completion, Flow 1 accomplishes the following:

## Verification Distribution

* Sends one verification email per account

## Tracking

* Sets VerificationStatus = Pending
* Sets VerificationSentDate = Current Date/Time
* Resets ReminderCount = 0

## Audit Reporting

* Tracks successful sends
* Tracks missing emails
* Creates CSV audit report

## Metrics Captured

* Total Accounts Processed
* Missing Email Count
* Email Distribution Results
* Verification Status Updates

---

# Flow Summary

```text
Scheduled Monthly Trigger
        ↓
List Rows from Excel
        ↓
Initialize Variables
        ↓
Apply to Each Account
        ↓
Validate Required Fields
        ↓
Generate Verification URL
        ↓
Send Verification Email
        ↓
Update Verification Tracking Fields
        ↓
Append Audit Log
        ↓
Create CSV Audit Report
        ↓
Save Audit File to OneDrive
```
