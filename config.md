Below is a build-ready configuration guide for every list, including column type, required/unique flags, defaults, and Title handling.
This is written so you can create the lists once and reuse them for every course.

I’ll also tell you what not to touch so SharePoint doesn’t fight you later.

⸻

GLOBAL RULES (apply to every list)
	•	Rename the Title column (never delete it)
	•	Use GUID text columns as your real IDs
	•	Do NOT rely on SharePoint ID
	•	Do NOT store large payloads in lists (CSV/JSON go in a doc library)

⸻

1) COURSES (Reusable templates)

List name: Courses

Rename Title → CourseName

Column	Type	Settings
CourseId	Single line text	Default: =GUID() • Required • Unique
CourseName (Title)	Single line text	Required
RequiresAxonAccount	Yes/No	Default: No
RequiresDeviceIssuance	Yes/No	Default: No
DefaultLoanerRequiredForTDY	Yes/No	Default: Yes
DefaultLoanerReturnDays	Number	Optional
StandardAxonRole	Single line text	Optional
Active	Yes/No	Default: Yes

Notes
	•	This list never changes during training days.
	•	Courses define behavior — sessions inherit it.

⸻

2) COURSE SESSIONS (Each class instance)

List name: CourseSessions

Rename Title → SessionName

Column	Type	Settings
SessionId	Single line text	Default: =GUID() • Required • Unique
SessionName (Title)	Single line text	Required
Course	Lookup → Courses	Required
StartDateTime	Date & Time	Required
EndDateTime	Date & Time	Required
Location	Single line text	Optional
Instructor	Person	Optional
Capacity	Number	Optional
Status	Choice	Planned / Live / Closed / Cancelled • Default: Planned

Notes
	•	Status = Live unlocks check-in
	•	Status = Closed locks edits except returns

⸻

3) REGISTRATIONS (List A – operational roster)

List name: Registrations

Rename Title → RegistrationName

Column	Type	Settings
RegistrationId	Single line text	Default: =GUID() • Required • Unique
RegistrationName (Title)	Single line text	Auto-filled (DisplayName | SessionName)
Session	Lookup → CourseSessions	Required
UserPerson	Person	People only • Required
UserUPN	Single line text	Required • Unique per session via key
RegistrationKey	Single line text	Required • Unique
Component	Choice	ERO / HSI
AttendeeType	Choice	Permanent / TDY / TDY-HasHomeCamera
DetailStartDate	Date	Optional
DetailEndDate	Date	Required if TDY
CheckInStatus	Choice	NotCheckedIn / CheckedIn / NoShow / Completed
LoanerNeeded	Yes/No	Auto-set
LoanerIssued	Yes/No	Default: No
LoanerReturned	Yes/No	Default: No
LoanerDueDate	Date	Auto-set
AxonUserStatus	Choice	NeedsCreate / CreateQueued / Created / CreateFailed
AxonJob	Lookup → AxonJobs	Optional
AxonLastError	Multiple lines	Optional
AxonLastAttempt	Date & Time	Optional

List validation (critical)

=IF(
  OR([AttendeeType]="TDY",[AttendeeType]="TDY-HasHomeCamera"),
  NOT(ISBLANK([DetailEndDate])),
  TRUE
)

Why this list matters
	•	One row = one person + one session
	•	Reusable forever
	•	Clean retries and reporting

⸻

4) DEVICES (Inventory)

List name: Devices

Rename Title → DeviceName

Column	Type	Settings
DeviceId	Single line text	Default: =GUID() • Required • Unique
DeviceName (Title)	Single line text	Serial or AssetTag
Model	Choice	Axon Body 4 / Dock / Other
SerialNumber	Single line text	Required • Unique
AssetTag	Single line text	Optional
Status	Choice	Available / Issued / Maintenance / Lost
CurrentAssignment	Lookup → Assignments	Optional
CurrentHolderUPN	Single line text	Optional


⸻

5) ASSIGNMENTS (Device issuance lifecycle)

List name: Assignments

Rename Title → AssignmentName

Column	Type	Settings
AssignmentId	Single line text	Default: =GUID() • Required • Unique
AssignmentName (Title)	Single line text	Auto (UPN | Serial | Date)
Registration	Lookup → Registrations	Required
Device	Lookup → Devices	Required
AssignmentType	Choice	Primary / Loaner
IssuedDateTime	Date & Time	Default: Now
DueDate	Date	Required for Loaner
ReturnedDateTime	Date & Time	Optional
Status	Choice	Issued / Returned / Exception
IssuedBy	Person	Optional
ReturnedBy	Person	Optional


⸻

6) AXON JOBS (Automation control plane)

List name: AxonJobs

Rename Title → JobName

Column	Type	Settings
AxonJobId	Single line text	Default: =GUID() • Required • Unique
JobName (Title)	Single line text	UserImport | Session | Timestamp
Session	Lookup → CourseSessions	Required
JobType	Choice	UserImport (future: DeviceAssign)
Status	Choice	Queued / Submitted / Processing / Completed / Failed
PartnerJobId	Single line text	Returned by Axon
SubmittedAt	Date & Time	Optional
CompletedAt	Date & Time	Optional
PayloadRef	Hyperlink	Link to CSV/JSON in doc library
ResponseSummary	Multiple lines	Optional
ErrorMessage	Multiple lines	Optional


⸻

TITLE FIELD RULE (IMPORTANT)

You rename Title for clarity, but:
	•	You never use it as a key
	•	You never enforce uniqueness on it
	•	It’s strictly for humans and UI readability

Your GUID columns are the real identifiers.

⸻

FINAL COUNT (minimum required)

List	Required
Courses	Yes
CourseSessions	Yes
Registrations	Yes
Devices	Yes (for BWC)
Assignments	Yes (for loaners)
AxonJobs	Yes (for bulk API)


⸻

What this gives you
	•	Reusable for any future course
	•	Clean Axon retries
	•	Enforced loaner returns
	•	Zero identity ambiguity
	•	Audit-ready without extra work

If you want next, I can:
	•	give you a build checklist order (what to create first),
	•	or a Power Apps schema diagram mapping screens → list writes,
	•	or the exact Power Automate filters that assume these settings.

Just tell me where you want to go next.