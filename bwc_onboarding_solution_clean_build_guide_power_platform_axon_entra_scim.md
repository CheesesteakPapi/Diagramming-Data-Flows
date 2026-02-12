# BWC Onboarding Solution — Clean Build Guide (Power Platform + Axon + Entra SCIM)

This is the “one clean source of truth” build plan for the solution we designed.

**Goal (end-state):**

1. Create/maintain Axon user accounts (short-term via Partner API; long-term via Entra SCIM)
2. Route users to the correct **non-evidence** Axon groups/roles (long-term via SCIM group provisioning)
3. Assign/unassign devices (Body 4, TASER, loaners) via Partner API

---

## 1) Architecture (simple and stable)

### Long-term lane (Identity + Group Routing)

- **Microsoft Entra ID (SCIM)** provisions:
  - Users
  - Groups
  - Group membership (routing)

### Short-term lane (Training ops + Devices + Exceptions)

- **Power Platform (Dataverse + Power Automate + Power Apps)** handles:
  - Training registration
  - Loaner vs permanent decisioning
  - Device assignment/unassignment
  - Bulk job submissions (when needed)
  - Audit trail + troubleshooting

### Always-on “Truth Mirror”

- Hourly flow syncs Axon Users → Dataverse reference table for:
  - reporting
  - validation
  - exception processing

---

## 2) Build order (do not deviate)

1. Create **Dataverse tables** (Section 3)
2. Create **Environment Variables** (Section 4)
3. Create **Connections / Connection References** (Section 5)
4. Create **Flows** (Section 6) in this order:
   - F1 Token Cache (already done)
   - F2 Hourly Axon User Sync (pagination + upsert)
   - F3 Registration → Device Assignment
   - F4 Device Return (scheduled)
   - F5 Bulk Job Submit + Poll (optional)
5. Create **Power App** UI (Section 7)
6. Configure **Entra SCIM** pilot (Section 8)

---

## 3) Dataverse tables (clean set)

Below are the finalized table definitions including **Primary Column (Display Name, Schema Name, Requirement, Max Character Count)** and a clear **Table Description**.

---

### T0 — Course (Reusable Template)

**Table Description:** Defines reusable course templates that standardize onboarding rules, default access bundles, and device assignment policies across multiple sessions.

**Primary Column (Dataverse Primary Name)**

- Display name: Course Name
- Schema name: `bwc_coursename`
- Type: Text
- Requirement: Business required
- Max characters: 100

**Columns**

- CourseCode (Text)
- DefaultAccessBundle (Choice): Pro, Standard, Instructor
- DefaultDeviceModel (Choice): Axon Body 4, TASER 10, etc.
- RequiresDeviceAssignment (Yes/No)
- DefaultAssignmentType (Choice): Permanent, Loaner
- RequiresDetailEndDate (Yes/No)
- LicenseType (Choice): Pro, Basic, None
- IsActive (Yes/No)
- Notes (Multiline)

**Relationship**

- One Course → Many Course Sessions

---

### T1 — Course Session

**Table Description:** Defines a specific training event instance tied to a reusable Course template.

**Primary Column (Dataverse Primary Name)**

- Display name: Session Name
- Schema name: `bwc_sessionname`
- Type: Text
- Requirement: Business required
- Max characters: 100

**Columns**

- Course (Lookup → Course)
- SessionCode (Text)
- StartDateTime (DateTime)
- EndDateTime (DateTime)
- Status (Choice): Planned, Open, Closed, Completed

---

### T2 — Registration

**Table Description:** Captures user onboarding requests tied to a training session and drives device assignment and provisioning logic.

**Primary Column (Dataverse Primary Name)**

- Display name: Registration Name
- Schema name: `bwc_registrationname`
- Type: Text
- Requirement: Business required
- Max characters: 100
- Recommended value pattern (set via flow): `UPN + " - " + SessionName`

**Columns**

- Session (Lookup → Course Session)
- UPN (Text)
- Component (Choice): ERO, HSI
- DutyType (Choice): Permanent, TDY, Detail
- DetailEndDate (Date)
- HasHomeDeviceAssigned (Yes/No)
- HomeDeviceSerial (Text)
- RequestedAccessBundle (Choice): Pro, Standard, Instructor
- NeedsLoaner (Yes/No)
- RegistrationStatus (Choice): New, PendingProvisioning, DeviceAssigned, Completed, Error
- ErrorNotes (Multiline)

**Business rule**

- If DutyType is TDY or Detail → DetailEndDate required

---

### T3 — Device

**Table Description:** Maintains inventory and real-time assignment state for Axon devices (Body 4, TASER, etc.).

**Primary Column (Dataverse Primary Name)**

- Display name: Serial Number
- Schema name: `bwc_serialnumber`
- Type: Text
- Requirement: Business required
- Max characters: 100

**Columns**

- Model (Choice): Axon Body 4, TASER 10, etc.
- IsLoaner (Yes/No)
- DeviceStatus (Choice): Available, Assigned, Repair, Retired
- AssignedUPN (Text)
- AssignmentType (Choice): Permanent, Loaner
- ExpectedReturnDate (Date)
- AssignedSession (Lookup → Course Session)

**Key**

- Alternate Key on SerialNumber

---

### T4 — Axon User Reference

**Table Description:** Cached mirror of Axon user accounts used for reporting, validation, and automation decision logic.

**Primary Column (Dataverse Primary Name)**

- Display name: User Display Name
- Schema name: `bwc_userdisplayname`
- Type: Text
- Requirement: Business required
- Max characters: 100
- Recommended value pattern: DisplayName (fallback to UPN)

**Columns**

- AxonUserId (Text) — REQUIRED
- UPN (Text)
- AxonState (Text): ACTIVE/INACTIVE
- LifecycleStatus (Choice): Active, Inactive, Invited, NeverLoggedIn
- LastLoginOn (DateTime)
- LastInvitedOn (DateTime)
- SourceHash (Text)
- LastSyncedOn (DateTime)

**Key**

- Alternate Key on AxonUserId

---

### T5 — Axon Job

**Table Description:** Tracks asynchronous bulk operations submitted to Axon (user imports, group changes, device actions).

**Primary Column (Dataverse Primary Name)**

- Display name: Axon Job ID
- Schema name: `bwc_axonjobid`
- Type: Text
- Requirement: Business required
- Max characters: 100

**Columns**

- JobType (Choice): UserImport, GroupImport, DeviceAssign, RoleUpdate
- Status (Choice): queued, processing, success, error
- Session (Lookup → Course Session)
- SubmittedOn (DateTime)
- ErrorSummary (Multiline)
- RequestPayload (Multiline)
- ResponsePayload (Multiline)

**Key**

- Alternate Key on AxonJobId

---

### T6 — Axon Job Result

**Table Description:** Stores per-record results from Axon bulk jobs, including success, updates, and skipped/error records.

**Primary Column (Dataverse Primary Name)**

- Display name: Result Name
- Schema name: `bwc_resultname`
- Type: Text
- Requirement: Business required
- Max characters: 100
- Recommended value pattern (set via flow): `UPN + " - " + Action`

**Columns**

- AxonJob (Lookup → Axon Job)
- UPN (Text)
- Action (Choice): Created, Updated, Removed, Skipped
- ErrorCode (Text)
- ErrorReason (Multiline)
- RawRow (Multiline JSON)

---

---

## 3A) Enterprise Hardening Standards

This section formalizes schema naming, ownership model, indexing, and scalability controls to support growth toward 22,000+ users.

---

### 1) Schema Naming Convention (Required)

All custom tables and columns must use prefix:

`bwc_`

Example:

- Table logical name: `bwc_course`
- Column logical name: `bwc_upn`
- Lookup logical name: `bwc_sessionid`

Do NOT rely on default system-generated schema names.

---

### 2) Table Ownership Model

For this solution, use **Organization-owned tables** unless row-level security is required.

Recommended ownership:

- Course → Organization-owned
- Course Session → Organization-owned
- Registration → Organization-owned
- Device → Organization-owned
- Axon User Reference → Organization-owned
- Axon Job → Organization-owned
- Axon Job Result → Organization-owned

Rationale:

- Centralized automation
- No dependency on user ownership
- Cleaner ALM deployment
- Simpler reporting and filtering

Only switch to User-owned if per-user security partitioning becomes necessary.

---

### 3) Indexing & Performance Controls (Critical for 22k+ users)

Create indexes on the following columns:

Axon User Reference:

- `bwc_axonuserid` (Alternate Key — already defined)
- `bwc_upn`
- `bwc_lifecyclestatus`
- `bwc_axonstate`

Registration:

- `bwc_upn`
- `bwc_registrationstatus`
- `bwc_sessionid`

Device:

- `bwc_serialnumber` (Alternate Key)
- `bwc_devicestatus`
- `bwc_assignedupn`

Axon Job:

- `bwc_axonjobid` (Alternate Key)
- `bwc_status`

---

### 4) Column Hardening Standards

Text Columns:

- UPN → Max 255 characters
- AxonUserId → Max 100
- SerialNumber → Max 100
- Hash fields → Max 64

Multiline fields:

- ErrorSummary → 4000 characters
- RequestPayload / ResponsePayload → 100000+ (enable large text if supported)

Choice fields:

- Use global choice sets where possible (Component, DutyType, Status)
- Prevent duplicate local choice definitions across tables

---

### 5) Concurrency Controls (Flow-Level Hardening)

For hourly sync flow:

- Enable concurrency control
- Degree of parallelism = 10 (from EV\_AxonSyncConcurrency)

For device assignment flow:

- Turn OFF concurrency (avoid double-assigning devices)

---

### 6) Data Retention Strategy

- Axon Job Results older than 180 days → archive or purge
- Device audit history retained indefinitely
- Axon User Reference retained indefinitely (system of record mirror)

---

### 7) Auditing

Enable Dataverse auditing on:

- RegistrationStatus
- DeviceStatus
- AssignmentType
- LifecycleStatus

Disable auditing on high-volume hash or sync timestamp columns.

---

## 4) Environment Variables (Solution Parameters)



Create these in the solution:

- EV\_AxonBaseUrl (Text)
- EV\_AxonUsersEndpoint (Text) — optional; otherwise build from base
- EV\_AxonPageLimit (Number) — default 500
- EV\_AxonRetryWaitSeconds (Number) — default 300
- EV\_AxonSyncConcurrency (Number) — default 10
- EV\_InviteStaleDays (Number) — default 7
- EV\_NeverLoginStaleDays (Number) — default 14

---

## 5) Connections / Connection References

In the solution create Connection References for:

- Microsoft Dataverse
- HTTP (used for Axon Partner API)
- SharePoint (REQUIRED — used to store CSV files and job artifacts)

### SharePoint storage standard (CSV + artifacts)

Create a SharePoint site/library for this solution, then standardize folders:

- **Library:** `Axon Onboarding Artifacts`
  - `/CSV/CourseTemplates/` (optional)
  - `/CSV/Submissions/` (generated CSVs sent to Axon)
  - `/CSV/Results/` (Axon results exported to CSV, optional)
  - `/Jobs/<AxonJobId>/` (job-specific payloads and outputs)

**File naming convention**

- User import submission: `UserImport_<CourseCode>_<SessionCode>_<yyyyMMdd_HHmmss>.csv`
- Group import submission: `GroupImport_<CourseCode>_<SessionCode>_<yyyyMMdd_HHmmss>.csv`
- Job snapshot JSON: `<AxonJobId>_status_<timestamp>.json`

**Why store files in SharePoint**

- Operational audit trail
- Re-run capability
- Easy handoff to stakeholders
- Keeps Dataverse lean (Dataverse stores metadata + pointers; SharePoint stores large artifacts)

---

## 6) Power Automate Flows (clean set) (clean set)

### F1 — Get Valid Axon Token (you already built)

**Purpose:** returns a valid bearer token for other flows

**Output (recommended):** token string and expiresAt

---

### F2 — Hourly Axon User Sync (pagination + upsert + change detection)

**Purpose:** mirror Axon accounts into Dataverse reference table

**Trigger:** Recurrence (hourly)

**High-level steps**

1. Call F1 to get token
2. Initialize variables:
   - NextUrl = first users URL with limit
   - HasMore = true
3. Do until HasMore = false
   - HTTP GET NextUrl
   - If status != 200 → handle retry rules (429/5xx/401)
   - Parse JSON (null-safe)
   - Apply to each data[]
     - Compute LifecycleStatus
     - Compute SourceHash
     - Upsert Axon User Reference by AxonUserId
   - NextUrl = links.next
   - HasMore = NextUrl not empty

**Critical rules**

- Do not accumulate all users in one array.
- Process page-by-page.
- Parse JSON schema must allow null timestamps.

---

### F3 — Registration → Device Assignment

**Purpose:** assign a device (loaner/permanent) after a registration

**Trigger:** When a Registration row is added

**Decision logic**

- NeedsLoaner = TDY/Detail OR HasHomeDeviceAssigned
- If TDY/Detail and DetailEndDate missing → Error

**Steps**

1. Normalize UPN
2. Check Axon User Reference by UPN
   - If not found: set RegistrationStatus = PendingProvisioning (SCIM will create)
3. Select an available device
   - If NeedsLoaner: IsLoaner=true AND Available
   - Else: IsLoaner=false AND Available
4. Assign device via Axon Partner API
5. Update Device row to Assigned
6. Update RegistrationStatus = DeviceAssigned/Completed

---

### F4 — Device Return (scheduled)

**Purpose:** automatically unassign loaner devices at end of detail/TDY

**Trigger:** Scheduled daily OR runs hourly

**Steps**

1. List Devices where:
   - AssignmentType = Loaner
   - ExpectedReturnDate <= today
   - DeviceStatus = Assigned
2. For each:
   - Call Axon unassign endpoint
   - Update Device to Available

---

### F5 — Bulk Job Submit + Poll (optional)

**Purpose:** use Axon bulk endpoints for mass user creation, group membership changes, role updates

**Steps**

1. Build CSV from Registrations
2. Submit bulk job
3. Write Axon Job row (AxonJobId)
4. Poll job status until success/error
5. Fetch results → write Axon Job Results

---

## 7) Power App (Operator UI)

You can build this as either a **Canvas app** (fast, tailored) or a **Model-driven app** (governed, form/view based). The solution supports both.

### Recommended approach

- **Phase 1 (Training surge / fast iteration): Canvas app**
- **Phase 2 (Steady-state operations): Model-driven app** (optional migration once the data model stabilizes)

---

### Option A — Canvas App (recommended for Phase 1)

**Why Canvas for training onboarding**

- Faster to build a purpose-built workflow UI (session-based intake + device board + exception queue)
- Better “one-screen ops console” experience for coordinators
- Easier to add guided steps, validation, and status indicators

---

## Canvas App formulas (exact)

> Naming assumptions (adjust to your real names):
>
> - Dataverse tables: `Course Session`, `Registration`, `Device`
> - Primary app variables: `varSession` (selected session record), `varRegistration` (selected registration record), `varDevice` (selected device record)
> - Controls:
>   - `ddDutyType` (Dropdown)
>   - `dpDetailEndDate` (DatePicker)
>   - `txtUPN` (Text input)
>   - `ddComponent` (Dropdown)
>   - `ddBundle` (Dropdown)
>   - `tglHasHomeDevice` (Toggle)
>   - `galSessions`, `galRegistrations`, `galDevicesAvailable`, `galDevicesAssigned`

### 1) TDY/Detail conditional visibility + required behavior

**A) Detail end date visible only for TDY/Detail** Set `dpDetailEndDate.Visible`:

```powerfx
(ddDutyType.Selected.Value = "TDY") || (ddDutyType.Selected.Value = "Detail")
```

**B) "Required" visual indicator** Set label `lblDetailEndDateRequired.Visible`:

```powerfx
(dpDetailEndDate.Visible)
```

**C) Block submission if required field missing** Use this validation expression (reused in Submit logic):

```powerfx
If(
    ((ddDutyType.Selected.Value = "TDY") || (ddDutyType.Selected.Value = "Detail")) && IsBlank(dpDetailEndDate.SelectedDate),
    Notify("Detail End Date is required for TDY/Detail.", NotificationType.Error);
    false,
    true
)
```

---

### 2) Submit button logic (create registration + trigger flow)

**Button: **``

This creates the Registration row and (optionally) triggers a flow that performs provisioning/device assignment.

```powerfx
// Normalize UPN
Set(varUPN, Lower(Trim(txtUPN.Text)));

// Validate inputs
If(
    IsBlank(varUPN),
    Notify("UPN is required.", NotificationType.Error);
    Exit(),
    true
);

If(
    IsBlank(ddComponent.Selected.Value),
    Notify("Component is required.", NotificationType.Error);
    Exit(),
    true
);

If(
    IsBlank(ddDutyType.Selected.Value),
    Notify("Duty Type is required.", NotificationType.Error);
    Exit(),
    true
);

If(
    ((ddDutyType.Selected.Value = "TDY") || (ddDutyType.Selected.Value = "Detail")) && IsBlank(dpDetailEndDate.SelectedDate),
    Notify("Detail End Date is required for TDY/Detail.", NotificationType.Error);
    Exit(),
    true
);

If(
    IsBlank(varSession),
    Notify("Select an open session first.", NotificationType.Error);
    Exit(),
    true
);

// Create Registration
Set(
    varNewReg,
    Patch(
        Registration,
        Defaults(Registration),
        {
            'Registration Name': varUPN & " - " & varSession.'Session Name',
            Session: varSession,
            UPN: varUPN,
            Component: ddComponent.Selected,
            DutyType: ddDutyType.Selected,
            DetailEndDate: If(dpDetailEndDate.Visible, dpDetailEndDate.SelectedDate, Blank()),
            HasHomeDeviceAssigned: tglHasHomeDevice.Value,
            RequestedAccessBundle: ddBundle.Selected,
            RegistrationStatus: 'RegistrationStatus (Registration)'.New
        }
    )
);

Notify("Registration submitted.", NotificationType.Success);

// OPTIONAL: trigger Power Automate flow to start assignment/provisioning
// Replace 'Flow_AssignDevice' with your flow name and pass the Registration Row ID
// Set(varFlowResult, Flow_AssignDevice.Run(varNewReg.RegistrationId));

Reset(txtUPN);
Reset(ddComponent);
Reset(ddDutyType);
Reset(dpDetailEndDate);
Reset(ddBundle);
Reset(tglHasHomeDevice);
```

> Notes:
>
> - For Dataverse primary key, replace `varNewReg.RegistrationId` with your table’s actual ID field (e.g., `bwc_registrationid`).
> - If your choice columns are global choices, the choice reference name will differ; adjust in Patch.

---

### 3) Filtering registrations by session/status

**A) Sessions gallery (only Open sessions)** Set `galSessions.Items`:

```powerfx
Filter('Course Session', Status.Value = "Open")
```

Set `galSessions.OnSelect`:

```powerfx
Set(varSession, ThisItem)
```

**B) Registrations gallery filtered by selected session** Assume you have a status filter dropdown `ddRegStatus` with items (All/New/PendingProvisioning/DeviceAssigned/Error).

Set `ddRegStatus.Items`:

```powerfx
["All", "New", "PendingProvisioning", "DeviceAssigned", "Completed", "Error"]
```

Set `galRegistrations.Items`:

```powerfx
If(
    IsBlank(varSession),
    Filter(Registration, false),
    If(
        ddRegStatus.Selected.Value = "All",
        Filter(Registration, Session.'Course Session' = varSession.'Course Session'),
        Filter(
            Registration,
            Session.'Course Session' = varSession.'Course Session' && RegistrationStatus.Value = ddRegStatus.Selected.Value
        )
    )
)
```

Set `galRegistrations.OnSelect`:

```powerfx
Set(varRegistration, ThisItem)
```

> Tip: If you also want filtering by DutyType/Bundle, extend the Filter() with additional conditions.

---

### 4) Device assignment “select + confirm” pattern

This pattern prevents accidental assignments and avoids double-assigning devices.

**A) Available devices gallery (filter by model + availability + loaner policy)** Assume you have dropdown `ddDeviceModel` and toggle `tglLoanerOnly`.

Set `galDevicesAvailable.Items`:

```powerfx
With(
    {
        m: ddDeviceModel.Selected.Value,
        loanerOnly: tglLoanerOnly.Value
    },
    Filter(
        Device,
        DeviceStatus.Value = "Available" &&
        (IsBlank(m) || Model.Value = m) &&
        (!loanerOnly || IsLoaner = true)
    )
)
```

Set `galDevicesAvailable.OnSelect`:

```powerfx
Set(varDevice, ThisItem)
```

**B) Enable Assign button only when both selected** Set `btnAssignDevice.DisplayMode`:

```powerfx
If(IsBlank(varRegistration) || IsBlank(varDevice), DisplayMode.Disabled, DisplayMode.Edit)
```

**C) Confirm dialog** Add a hidden container `ctrConfirmAssign` and a variable `varShowConfirm`.

Set `btnAssignDevice.OnSelect`:

```powerfx
Set(varShowConfirm, true)
```

Set `ctrConfirmAssign.Visible`:

```powerfx
varShowConfirm
```

Set confirm label text `lblConfirmAssign.Text`:

```powerfx
"Assign device " & varDevice.'Serial Number' & " to " & varRegistration.UPN & "?"
```

**D) Confirm Yes → Patch + call flow** Set `btnConfirmYes.OnSelect`:

```powerfx
// Optimistic UI update to prevent another operator selecting the same device
Patch(
    Device,
    varDevice,
    {
        DeviceStatus: 'DeviceStatus (Device)'.Assigned,
        AssignedUPN: varRegistration.UPN,
        AssignmentType: If(varRegistration.NeedsLoaner, 'AssignmentType (Device)'.Loaner, 'AssignmentType (Device)'.Permanent),
        ExpectedReturnDate: If(varRegistration.NeedsLoaner, varRegistration.DetailEndDate, Blank()),
        AssignedSession: varSession
    }
);

// Update registration status
Patch(
    Registration,
    varRegistration,
    {
        RegistrationStatus: 'RegistrationStatus (Registration)'.DeviceAssigned
    }
);

// OPTIONAL: Call a flow that performs the Axon Partner API assignment (recommended)
// Set(varAssignResult, Flow_AssignSelectedDevice.Run(varRegistration.<id>, varDevice.<id>));

Notify("Device assigned.", NotificationType.Success);

Set(varShowConfirm, false);
Set(varDevice, Blank());
```

**E) Confirm No** Set `btnConfirmNo.OnSelect`:

```powerfx
Set(varShowConfirm, false)
```

**F) Assigned devices gallery** Set `galDevicesAssigned.Items`:

```powerfx
Filter(Device, DeviceStatus.Value = "Assigned")
```

---

#### Recommended Power Automate integration approach

- Canvas app performs **validation + selection**
- Flow performs **Axon API call** and writes authoritative results back to Dataverse
- If flow fails, set RegistrationStatus = Error and revert DeviceStatus to Available

---

## Bulletproofing patterns (recommended)

### A) True optimistic lock (re-check before assignment)

Problem: two operators may select the same device at the same time.

**Pattern:** Re-read the device from Dataverse right before assignment and only proceed if still Available.

1. Add a flow `Flow_LockAndAssignDevice` that takes:

- RegistrationId
- DeviceId

2. In Canvas, call the flow and do **no direct device Patch** until the flow returns success.

**Canvas: Confirm Yes (replace the optimistic Patch with flow call):**

```powerfx
Set(varShowConfirm, false);
Set(varAssignBusy, true);

// Call the flow to lock + assign atomically
Set(
    varAssignResult,
    Flow_LockAndAssignDevice.Run(
        varRegistration.'Registration',
        varDevice.'Device'
    )
);

Set(varAssignBusy, false);

If(
    varAssignResult.success = true,
    Notify("Device assigned.", NotificationType.Success);
    Set(varDevice, Blank());
    Refresh(Device);
    Refresh(Registration),
    Notify("Assignment failed: " & Coalesce(varAssignResult.message, "Unknown error"), NotificationType.Error);
    // Leave selection so user can retry or choose another device
    true
);
```

**Flow behavior (what it must do):**

- Get Device row
- If DeviceStatus != Available → return {success\:false, message:"Device already assigned"}
- Else:
  - Update Device → Assigned
  - Update Registration → DeviceAssigned
  - Call Axon Partner API assign endpoint
  - If Axon fails → rollback (below)

> Why flow? Power Fx Patch is not transactional across multiple users. The flow becomes the single-writer gate.

---

### B) Rollback on Partner API failure

Problem: You update Dataverse to Assigned but Axon assignment fails.

**Pattern:** In the flow, treat Dataverse updates as provisional until Axon returns success.

Flow logic:

1. Verify device is Available
2. Update Device to **Assigning** (new status) OR keep Available until Axon succeeds
3. Call Axon assign endpoint
4. If success:
   - Update Device → Assigned
   - Update RegistrationStatus → DeviceAssigned/Completed
5. If failure:
   - Update Device → Available
   - Clear AssignedUPN/ExpectedReturnDate/AssignmentType
   - Update RegistrationStatus → Error
   - Set ErrorNotes with Axon response

**If you want to keep the UI simple:** add `Assigning` to `DeviceStatus` choice. Then operators can see “in-flight” assignments.

---

### C) Prevent duplicate registrations (same UPN + same Session)

Problem: you don’t want multiple registrations for the same person in the same session.

**Option 1 (best): Alternate Key on Registration** Create a text column:

- `bwc_registrationkey` (Text, Business required, Max 200)

Set it to:

- `Lower(UPN) & "|" & SessionId`

Create an **Alternate Key** on `bwc_registrationkey`.

Then use Upsert patterns (or Patch will error cleanly on duplicate).

**Option 2 (Canvas guard): pre-check before Patch** Add this at the top of `btnSubmitRegistration.OnSelect` before creating the registration:

```powerfx
If(
    !IsBlank(
        LookUp(
            Registration,
            Session.'Course Session' = varSession.'Course Session' && Lower(UPN) = varUPN
        )
    ),
    Notify("User is already registered for this session.", NotificationType.Warning);
    Exit(),
    true
);
```

---

### D) Busy state + disable double-clicks

Add a variable `varAssignBusy` and use it to disable buttons during flow calls.

Set `btnConfirmYes.DisplayMode`:

```powerfx
If(varAssignBusy, DisplayMode.Disabled, DisplayMode.Edit)
```

Set `btnSubmitRegistration.DisplayMode`:

```powerfx
If(varAssignBusy, DisplayMode.Disabled, DisplayMode.Edit)
```

---

### Screen structure (for reference)

##### Screen 1 — Home / Dashboard

- KPIs (counts by status): New, PendingProvisioning, DeviceAssigned, Error
- Quick links: Active Sessions, Devices, Exceptions
- Optional: Search by UPN

##### Screen 2 — Sessions

- Gallery of Course Sessions (filter Status = Open)
- Buttons: New Session / Open / Close

##### Screen 3 — Register User

- Guided form with TDY/Detail end-date logic
- Submit button uses formulas above

##### Screen 4 — Registration Queue

- Filtered roster by session + status
- Select registration → enables device assignment

##### Screen 5 — Device Board (Enhanced Device Management)

This screen functions as the operational control center for managing devices assigned to users.

---

## Device Management Features

### 1) View Devices Assigned to a Specific User

Add a "User Device View" panel.

**Selected registration devices gallery** Set `galUserDevices.Items`:

```powerfx
Filter(
    Device,
    AssignedUPN = varRegistration.UPN
)
```

Display fields:

- Serial Number
- Model
- AssignmentType
- ExpectedReturnDate
- DeviceStatus

---

### 2) Unassign Device (Manual Return)

Add button: `btnUnassignDevice`

Set `DisplayMode`:

```powerfx
If(IsBlank(varDevice), DisplayMode.Disabled, DisplayMode.Edit)
```

Set `OnSelect` (calls flow for safe unassignment):

```powerfx
Set(varAssignBusy, true);

Set(
    varUnassignResult,
    Flow_UnassignDevice.Run(
        varDevice.'Device'
    )
);

Set(varAssignBusy, false);

If(
    varUnassignResult.success = true,
    Notify("Device unassigned.", NotificationType.Success);
    Refresh(Device);
    Refresh(Registration);
    Set(varDevice, Blank()),
    Notify("Unassignment failed: " & Coalesce(varUnassignResult.message, "Unknown error"), NotificationType.Error)
);
```

Flow must:

- Validate device is Assigned
- Call Axon unassign endpoint
- Update Device → Available
- Clear AssignedUPN, ExpectedReturnDate, AssignmentType

---

### 3) Transfer Device Between Users

Use the same Lock + Assign pattern:

1. Unassign current user (via flow)
2. Assign to new registration (via Flow\_LockAndAssignDevice)

Optional enhancement: Create a single flow `Flow_TransferDevice`:

- Inputs: DeviceId, FromRegistrationId, ToRegistrationId
- Performs atomic unassign + assign

---

### 4) Overdue Loaner Monitoring

Add badge indicator in gallery:

Set badge `Visible` property:

```powerfx
DeviceStatus.Value = "Assigned" &&
AssignmentType.Value = "Loaner" &&
ExpectedReturnDate < Today()
```

Color property:

```powerfx
If(
    ExpectedReturnDate < Today(),
    Color.Red,
    Color.Black
)
```

---

### 5) Device History Tracking (Recommended Enhancement)

Add new table: `Device Assignment History`

Primary Column:

- Display name: History Name
- Schema: `bwc_historyname`
- Requirement: Business required
- Max characters: 150

Columns:

- Device (Lookup → Device)
- UPN (Text)
- Action (Choice): Assigned, Unassigned, Transferred
- ActionDateTime (DateTime)
- Session (Lookup → Course Session)
- PerformedBy (User)

Flow updates this table whenever:

- Device assigned
- Device unassigned
- Device transferred

---

### 6) Bulk Device Assignment (Optional)

For large training sessions:

1. Select multiple registrations (use checkbox column in gallery).
2. Collect selected users:

```powerfx
ClearCollect(
    colSelectedRegs,
    Filter(galRegistrations.AllItems, chkSelect.Value = true)
);
```

3. Call bulk flow:

```powerfx
Flow_BulkAssignDevices.Run(JSON(colSelectedRegs))
```

Flow logic:

- Match available devices by model
- Assign sequentially
- Return success/failure summary

---

### 7) Device Status Lifecycle Model

Recommended DeviceStatus choices:

- Available
- Assigning
- Assigned
- Repair
- Retired

Rules:

- Only Available → Assigning → Assigned
- Assigned → Available (unassign)
- Assigned → Repair (if damaged)
- Repair → Available (after fix)
- Retired → terminal state

---

### 8) Search Devices by Serial or UPN

Add search input `txtDeviceSearch`.

Set `galDevicesAvailable.Items`:

```powerfx
Filter(
    Device,
    DeviceStatus.Value = "Available" &&
    (IsBlank(txtDeviceSearch.Text) ||
     StartsWith('Serial Number', txtDeviceSearch.Text))
)
```

Set `galDevicesAssigned.Items`:

```powerfx
Filter(
    Device,
    DeviceStatus.Value = "Assigned" &&
    (
        IsBlank(txtDeviceSearch.Text) ||
        StartsWith(AssignedUPN, txtDeviceSearch.Text) ||
        StartsWith('Serial Number', txtDeviceSearch.Text)
    )
)
```

---

### 9) Device Detail Panel (Operator Transparency)

When selecting a device:

```powerfx
Set(varDevice, ThisItem)
```

Display:

- Serial
- Model
- Loaner flag
- Current UPN
- Session
- Return date
- Status

---

### Screen 6 — Exceptions

- PendingProvisioning + Errors tabs

##### Screen 7 — Bulk Ops

- Generate CSV → store to SharePoint → submit job → track AxonJob

---

### Option B — Model-driven App (recommended for Phase 2) (for reference)

##### Screen 1 — Home / Dashboard

- KPIs (counts by status): New, PendingProvisioning, DeviceAssigned, Error
- Quick links: Active Sessions, Devices, Exceptions
- Optional: Search by UPN

##### Screen 2 — Sessions

- Gallery of Course Sessions (filter Status = Open)
- Buttons: New Session / Open / Close

##### Screen 3 — Register User

- Guided form with TDY/Detail end-date logic
- Submit button uses formulas above

##### Screen 4 — Registration Queue

- Filtered roster by session + status
- Select registration → enables device assignment

##### Screen 5 — Device Board

- Available vs Assigned view
- Select + Confirm assignment pattern

##### Screen 6 — Exceptions

- PendingProvisioning + Errors tabs

##### Screen 7 — Bulk Ops

- Generate CSV → store to SharePoint → submit job → track AxonJob

---

### Option B — Model-driven App (recommended for Phase 2) (recommended for Phase 2)

**Why model-driven later**

- Stronger governance and consistency (forms, views, business rules)
- Better for multi-team operations and audit-heavy environments
- Built-in search, filtering, dashboards, and security roles

**Model-driven app components**

- Tables: Course, Course Session, Registration, Device, Axon Job, Axon Job Result
- Views:
  - Open Sessions
  - Registrations by Status
  - Devices Available / Assigned / Repair
  - Jobs In Progress
- Dashboards:
  - Training onboarding dashboard
  - Device inventory dashboard

**Tradeoff**

- Slower to build a guided “ops console” experience
- Harder to do custom multi-step flows inside one screen

---

### If you must pick one today

Pick **Canvas app** now for speed and operator UX. Because you’re already Dataverse-based, moving to a model-driven app later is a UI swap, not a rebuild.

---

## 8) Entra SCIM (group provisioning supported) (group provisioning supported)

**Objective:** Entra becomes the routing engine for non-evidence Axon groups.

### Entra group design

- AXON-PRO-USERS
- AXON-STANDARD-USERS
- AXON-INSTRUCTORS

### Implementation steps

1. Assign groups to the Axon Enterprise App
2. Provisioning: Automatic
3. Enable Group provisioning
4. Pilot with small groups

**Operational handshake with Power Platform**

- Registration flow sets PendingProvisioning if Axon user not found.
- Hourly sync will eventually pull the user once SCIM provisions them.
- Then the Device Assignment can proceed.

---

## 9) Standard error handling (use everywhere)

Use a TRY/CATCH scope pattern for each Axon HTTP call.

**Retry rules**

- 401 → refresh token once, retry
- 429 → wait Retry-After else EV\_AxonRetryWaitSeconds, retry
- 5xx → exponential backoff, retry
- 4xx → log + stop (no blind retry)

Log errors to:

- Registration.ErrorNotes and RegistrationStatus=Error
- Axon Job.ErrorSummary (for bulk)

---

## 10) Implementation notes (avoid the traps)

- **Parse JSON schemas must allow null** for lastLoginOn/lastInvitedOn.
- **Dataverse List rows always returns an array** even with Top=1; use first().
- Do not put Axon HTTP calls inside a per-user loop unless you mean to.
- Hourly sync must use pagination; page size from EV\_AxonPageLimit.

---

## 11) Dashboard + Reporting (Operational Visibility)

This solution should provide near-real-time visibility into onboarding progress, device utilization, and exceptions.

---

### A) In-App Dashboard (Canvas App)

Add a Dashboard screen that reads directly from Dataverse and provides operator-friendly KPIs.

#### KPIs (cards)

**1) Active Sessions**

```powerfx
CountRows(Filter('Course Session', Status.Value = "Open"))
```

**2) Registrations (Selected Session)**

```powerfx
If(IsBlank(varSession), 0, CountRows(Filter(Registration, Session.'Course Session' = varSession.'Course Session')))
```

**3) Registrations by Status (Selected Session)**

- New

```powerfx
If(IsBlank(varSession), 0, CountRows(Filter(Registration, Session.'Course Session' = varSession.'Course Session' && RegistrationStatus.Value = "New")))
```

- PendingProvisioning

```powerfx
If(IsBlank(varSession), 0, CountRows(Filter(Registration, Session.'Course Session' = varSession.'Course Session' && RegistrationStatus.Value = "PendingProvisioning")))
```

- DeviceAssigned

```powerfx
If(IsBlank(varSession), 0, CountRows(Filter(Registration, Session.'Course Session' = varSession.'Course Session' && RegistrationStatus.Value = "DeviceAssigned")))
```

- Error

```powerfx
If(IsBlank(varSession), 0, CountRows(Filter(Registration, Session.'Course Session' = varSession.'Course Session' && RegistrationStatus.Value = "Error")))
```

**4) Device Inventory Counts**

- Available

```powerfx
CountRows(Filter(Device, DeviceStatus.Value = "Available"))
```

- Assigned

```powerfx
CountRows(Filter(Device, DeviceStatus.Value = "Assigned"))
```

- Repair

```powerfx
CountRows(Filter(Device, DeviceStatus.Value = "Repair"))
```

**5) Overdue Loaners**

```powerfx
CountRows(Filter(Device, AssignmentType.Value = "Loaner" && DeviceStatus.Value = "Assigned" && ExpectedReturnDate < Today()))
```

#### Visuals (simple)

- Progress bar: DeviceAssigned / Total Registrations

```powerfx
If(
    IsBlank(varSession),
    0,
    With(
        {
            total: CountRows(Filter(Registration, Session.'Course Session' = varSession.'Course Session')),
            done: CountRows(Filter(Registration, Session.'Course Session' = varSession.'Course Session' && RegistrationStatus.Value = "DeviceAssigned"))
        },
        If(total = 0, 0, done / total)
    )
)
```

---

### B) Model-driven Dashboard (optional / Phase 2)

If you deploy a model-driven app later:

- Create a Dashboard with charts:
  - Registrations by Status (stacked)
  - Devices by Status
  - Overdue Loaners
  - Axon Jobs by Status

---

### C) Reporting (Power BI recommended)

Power BI provides the best audit-friendly reporting without overloading Dataverse forms.

#### Dataset tables

- Course
- Course Session
- Registration
- Device
- Axon User Reference
- Axon Job
- Axon Job Result
- Device Assignment History (if implemented)

#### Core report pages

**1) Training Operations Summary**

- Sessions: open/closed
- Registrations by component (ERO/HSI)
- Completion rate
- Exceptions count

**2) Device Utilization**

- Assigned vs Available by model
- Loaner utilization
- Overdue returns
- Repair backlog

**3) Provisioning Health (SCIM + Axon Mirror)**

- Invited vs Active vs NeverLoggedIn
- Stale invited (days since invite)
- NeverLoggedIn aging buckets (7/14/30)

**4) Job Monitoring**

- Bulk jobs in progress
- Error rates by job type
- Top error reasons

#### Suggested measures (Power BI)

> Assumptions / prerequisites (recommended):
>
> - Dataverse already provides `Created On` for Registration.
> - Add a datetime column on Registration: **DeviceAssignedOn** (`bwc_deviceassignedon`) and set it when assignment succeeds.
> - Add a proper Date table `DimDate` and mark it as a Date table in Power BI.

##### 1) Base measures

**Total Registrations**

```DAX
Total Registrations =
COUNTROWS('Registration')
```

**Registrations Device Assigned**

```DAX
Registrations Device Assigned =
CALCULATE(
    [Total Registrations],
    'Registration'[RegistrationStatus] = "DeviceAssigned"
)
```

**Loaners Assigned**

```DAX
Loaners Assigned =
CALCULATE(
    COUNTROWS('Device'),
    'Device'[DeviceStatus] = "Assigned",
    'Device'[AssignmentType] = "Loaner"
)
```

**Overdue Loaners**

```DAX
Overdue Loaners =
CALCULATE(
    COUNTROWS('Device'),
    'Device'[DeviceStatus] = "Assigned",
    'Device'[AssignmentType] = "Loaner",
    'Device'[ExpectedReturnDate] < TODAY()
)
```

##### 2) Completion Rate

```DAX
Completion Rate =
DIVIDE(
    [Registrations Device Assigned],
    [Total Registrations],
    0
)
```

##### 3) Overdue Rate

```DAX
Loaner Overdue Rate =
DIVIDE(
    [Overdue Loaners],
    [Loaners Assigned],
    0
)
```

##### 4) Aging buckets (Invited / Never Logged In)

> These use `Axon User Reference` fields:
>
> - `LastInvitedOn`
> - `LastLoginOn`
> - `LifecycleStatus` (Invited / NeverLoggedIn)

**Invited Age (Days) — calculated column (recommended)**

```DAX
Invited Age (Days) =
IF(
    'Axon User Reference'[LifecycleStatus] = "Invited" &&
    NOT(ISBLANK('Axon User Reference'[LastInvitedOn])),
    DATEDIFF('Axon User Reference'[LastInvitedOn], NOW(), DAY),
    BLANK()
)
```

**Never Logged In Age (Days) — calculated column (recommended)**

This uses Created On from the Axon mirror if available. If not available from Axon, use the first time you saw the user in your mirror (add `FirstSeenOn`).

```DAX
Never Login Age (Days) =
IF(
    'Axon User Reference'[LifecycleStatus] = "NeverLoggedIn" &&
    ISBLANK('Axon User Reference'[LastLoginOn]) &&
    NOT(ISBLANK('Axon User Reference'[FirstSeenOn])),
    DATEDIFF('Axon User Reference'[FirstSeenOn], NOW(), DAY),
    BLANK()
)
```

**Invited 0–7 Days**

```DAX
Invited 0–7 Days =
CALCULATE(
    COUNTROWS('Axon User Reference'),
    'Axon User Reference'[LifecycleStatus] = "Invited",
    'Axon User Reference'[Invited Age (Days)] >= 0,
    'Axon User Reference'[Invited Age (Days)] <= 7
)
```

**Invited 8–14 Days**

```DAX
Invited 8–14 Days =
CALCULATE(
    COUNTROWS('Axon User Reference'),
    'Axon User Reference'[LifecycleStatus] = "Invited",
    'Axon User Reference'[Invited Age (Days)] >= 8,
    'Axon User Reference'[Invited Age (Days)] <= 14
)
```

**Invited 15–30 Days**

```DAX
Invited 15–30 Days =
CALCULATE(
    COUNTROWS('Axon User Reference'),
    'Axon User Reference'[LifecycleStatus] = "Invited",
    'Axon User Reference'[Invited Age (Days)] >= 15,
    'Axon User Reference'[Invited Age (Days)] <= 30
)
```

**Invited 31+ Days**

```DAX
Invited 31+ Days =
CALCULATE(
    COUNTROWS('Axon User Reference'),
    'Axon User Reference'[LifecycleStatus] = "Invited",
    'Axon User Reference'[Invited Age (Days)] >= 31
)
```

> Repeat the same pattern for NeverLoggedIn buckets using `[Never Login Age (Days)]`.

##### 5) Average time-to-assign device

**Avg Time to Assign (Minutes)**

```DAX
Avg Time to Assign (Minutes) =
AVERAGEX(
    FILTER(
        'Registration',
        'Registration'[RegistrationStatus] = "DeviceAssigned" &&
        NOT(ISBLANK('Registration'[DeviceAssignedOn]))
    ),
    DATEDIFF('Registration'[Created On], 'Registration'[DeviceAssignedOn], MINUTE)
)
```

**Median Time to Assign (Minutes) (optional)**

```DAX
Median Time to Assign (Minutes) =
MEDIANX(
    FILTER(
        'Registration',
        'Registration'[RegistrationStatus] = "DeviceAssigned" &&
        NOT(ISBLANK('Registration'[DeviceAssignedOn]))
    ),
    DATEDIFF('Registration'[Created On], 'Registration'[DeviceAssignedOn], MINUTE)
)
```

---

### Star-schema recommendation (fast at scale)

**Why:** A star schema keeps Power BI models fast and stable as your volume grows (22k users and many sessions/jobs).

#### Dimensions (small, reused across visuals)

- **DimDate** (Calendar table)
- **DimCourse** (from Course)
- **DimSession** (from Course Session)
- **DimUser** (UPN; optionally enriched from Entra or Axon mirror)
- **DimDevice** (SerialNumber; includes Model, IsLoaner)
- **DimStatus** (RegistrationStatus, DeviceStatus)
- **DimDutyType** (Permanent/TDY/Detail)
- **DimComponent** (ERO/HSI)

#### Facts (large, transactional)

- **FactRegistration** (from Registration)

  - Grain: 1 row per registration
  - Keys: SessionKey, UserKey, StatusKey, DutyTypeKey, ComponentKey, CreatedDateKey
  - Measures: counts, completion rate, time-to-assign

- **FactDeviceAssignment** (from Device Assignment History if implemented)

  - Grain: 1 row per device action (Assigned/Unassigned/Transferred)
  - Keys: DeviceKey, UserKey, SessionKey, ActionDateKey
  - Measures: device utilization, transfers, return compliance

- **FactAxonJob** (from Axon Job / Axon Job Result)

  - Grain: 1 row per job (and optionally 1 row per job result)
  - Measures: job success rate, top errors, volume

#### Relationships (recommended)

- Facts → Dimensions are **many-to-one** (single direction)
- Avoid many-to-many whenever possible
- Use surrogate keys if you stage in a dataflow (recommended for clean modeling)

#### Performance settings

- Prefer Import mode for Dataverse extracts (or Fabric/Datamart) for speed
- Incremental refresh on Fact tables if volume grows
- Keep large JSON/text columns (payloads) out of the main semantic model (load only for drill-through)

---

### D) Automated Executive Summary (Optional) (Optional)

Add a scheduled flow (daily during surges) that emails a short digest:

- Open sessions
- Registrations completed vs total
- Devices available (by model)
- Overdue loaners
- Top errors

Store the digest outputs in SharePoint alongside job artifacts.

---

## 11) What “done” looks like

### Short-term

- Registration creates a row
- Device assigned from Device table
- Axon User Reference shows user status and last login/invite
- Dashboard shows operational progress and exceptions

### Long-term

- Adding a user to AXON-PRO-USERS in Entra automatically:
  - provisions Axon account
  - provisions group membership
- Power Platform only assigns devices + handles exceptions
- Power BI dashboards provide audit-ready reporting

