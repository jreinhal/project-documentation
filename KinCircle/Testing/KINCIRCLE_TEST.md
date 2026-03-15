KinCircle Dev Test Plan (Master)
===============================

EXECUTION METHOD (IMPORTANT)
- All functional test steps must be performed through the UI in the tester's browser of choice.
- Automation is allowed only if it drives the chosen browser UI (e.g., Playwright in headed mode).
- This is a FULL UI RUN via browser; do not validate via curl/terminal API calls.
- Do NOT use curl/terminal API calls for functional testing or verification.
- Terminal/API calls are allowed only for setup (start services, set env vars)
  and for collecting metadata (version, git SHA).

TABLE OF CONTENTS
1. Scope
2. Environments & Setup
3. Test Data (Canonical Dataset)
4. Execution Rules
5. Smoke Tests
6. Full Functional Suites
7. AI & OCR Suites (Real Mode)
8. Security & Privacy
9. PWA / Offline
10. Supabase (Optional)
11. Accessibility & UX
12. Performance & Stability
13. Error Injection & Recovery
14. Concurrency & Multi-Tab
15. Regression Matrix (Full)
16. Release Checklist
17. Test Run Log Template
18. Appendix A: Per-Screen Step-by-Step Scripts
19. Appendix B: Regression Matrix Table
20. Appendix C: Bug Report Template
21. Appendix D: Field Validation Matrix (Exhaustive)
22. Appendix E: Test Case Catalog (ID + Expected Results)
23. Appendix F: Data Persistence & Migration Matrix

===============================================================================
1) SCOPE
===============================================================================
This plan validates KinCircle end-to-end behavior across UI, storage, security,
AI integrations, mobile UX, and regression risk areas.

===============================================================================
2) ENVIRONMENTS & SETUP
===============================================================================
Required (local dev):
- Frontend: npm run dev
- API: npm run dev:api
Optional:
- OCR: LightOnOCR service (VITE_OCR_ENABLED=true)
- Supabase: VITE_SUPABASE_URL + VITE_SUPABASE_ANON_KEY

Environment Variables (primary):
- GEMINI_API_KEY (server-side, required for real AI)
- VITE_API_BASE_URL (default http://localhost:8787)
- KIN_API_TOKEN / VITE_KIN_API_TOKEN (optional shared API key)
- VITE_GEMINI_MOCK (default true unless set to "false")
- VITE_OCR_ENABLED, VITE_OCR_SERVICE_URL
- KIN_PRIVACY_REQUIRED (force privacy mode server-side)
- KIN_RATE_LIMIT_MAX, KIN_AI_RATE_LIMIT_MAX
- VITE_STORAGE_PROVIDER (local | supabase)
- VITE_SUPABASE_URL, VITE_SUPABASE_ANON_KEY

===============================================================================
3) TEST DATA (CANONICAL DATASET)
===============================================================================
Users:
- Admin: Sarah Miller (ADMIN)
- Contributor: David Miller (CONTRIBUTOR)
- Viewer: Viewer User (VIEWER) if configured

Ledger Entries (seed):
1) Expense: Groceries $120.50 (today)
2) Expense: Pharmacy $45.10 (last week)
3) Time: Caregiving 2.5 hours (today)
4) Expense: Utilities $90.00 (two weeks ago)

Tasks:
- Upcoming: "Pick up prescriptions" (today)
- Completed: "Call insurance" (yesterday)

Recurring Expense:
- "Home Care" $300 monthly

Help Calendar:
- "Sunday meal delivery" (next Sunday)
- "Doctor transport" (tomorrow)

Medication:
- "Lisinopril" 10mg daily (active)

Vault:
- Upload a PDF or image sample

Family Journal:
- Select a photo (placeholder behavior)

===============================================================================
4) EXECUTION RULES
===============================================================================
- Use your primary browser of choice for all UI validation.
- Record evidence (screenshots) for FAIL or BLOCKED steps.
- If a feature requires configuration (Supabase, OCR), mark BLOCKED if not set.
- Always verify expected outputs explicitly (not just presence of UI elements).

===============================================================================
5) SMOKE TESTS (5-10 MIN)
===============================================================================
1) App loads, no blocking console errors
2) Onboarding completes and lands in Dashboard
3) Add Expense entry via Entry Form
4) Ledger shows entry; Export CSV works
5) Mobile nav: floating hamburger opens sidebar
6) Chat Assistant (mock) returns response to "Total expenses?"
7) Settings: theme change persists after refresh
8) PIN set + lock screen unlocks

===============================================================================
6) FULL FUNCTIONAL SUITES
===============================================================================
See Appendix A for step-by-step scripts.

Toast Notifications:
- Success toast appears after entry creation
- Success toast appears after task completion
- Error toast appears on validation failure
- Error toast appears on network/API error
- Toast auto-dismisses after timeout (~3-5 seconds)
- Multiple toasts stack correctly (newest on top)
- Toast can be manually dismissed by clicking X

===============================================================================
7) AI & OCR SUITES (REAL MODE)
===============================================================================
Preconditions:
- GEMINI_API_KEY set
- VITE_GEMINI_MOCK=false
- API server running

Tests:
- Chat Assistant ledger query
- Chat Assistant external search query
- Receipt parsing
- Voice parsing
- Medicaid analysis
- Rate limiting

===============================================================================
8) SECURITY & PRIVACY
===============================================================================
- Privacy mode scrubbing for names/PII
- Auto-lock + PIN enforcement
- Encryption validation in localStorage
- Security logs appended correctly
- CSP / security headers present
- API token enforcement

RBAC (Role-Based Access Control):
- ADMIN can delete any entry
- ADMIN can modify family settings (hourly rate, patient name)
- CONTRIBUTOR can add entries but cannot delete others' entries
- CONTRIBUTOR can edit own entries only
- VIEWER can only view data (no add/edit/delete buttons visible)
- Role displayed correctly in Family Circle member list
- Role change propagates to UI permissions immediately

LockScreen Additional Tests:
- Backspace/delete key removes last entered digit
- Entering 4th digit auto-submits PIN
- Failed PIN attempt logged to security logs
- Lock screen appears after configured idle timeout
- Lock screen blocks all navigation until unlocked
- Emergency access button (if present) logs EMERGENCY_ACCESS event

===============================================================================
9) PWA / OFFLINE
===============================================================================
- Service worker registers
- Offline shell loads after cache
- Navigation between views while offline

===============================================================================
10) SUPABASE (OPTIONAL)
===============================================================================
- Data persistence across reloads
- Cloud Sync uploads data
- Verify table mappings for entries, tasks, settings

===============================================================================
11) ACCESSIBILITY & UX
===============================================================================
- Keyboard navigation
- Focus indicators visible
- Buttons have aria-labels
- Touch targets >= 44px
- Safe-area padding correct

===============================================================================
12) PERFORMANCE & STABILITY
===============================================================================
- Cold start time
- Hot reload time
- AI response latency
- 100+ entry list render
- No unhandled promise rejections

===============================================================================
13) ERROR INJECTION & RECOVERY
===============================================================================
- API down during AI query
- Network loss mid-session
- File upload errors
- LocalStorage corruption
- Invalid inputs

Error Boundary Testing:
- Component crash shows fallback UI (not blank screen)
- Error boundary catches unhandled React exceptions
- User can recover by navigating away or refreshing
- Error details logged to console for debugging

Voice/Receipt Upload Error States:
- Upload unsupported file type (e.g., .exe, .txt) shows error
- Upload corrupted/invalid audio file shows error
- Network timeout during OCR shows retry option
- File exceeds size limit shows appropriate message
- Form remains functional after upload error (can retry or cancel)
- Partial OCR results handled gracefully (missing fields editable)

===============================================================================
14) CONCURRENCY & MULTI-TAB
===============================================================================
- Multi-tab storage sync
- Simultaneous edits (last write wins)
- Encryption boundary across tabs

===============================================================================
15) REGRESSION MATRIX (FULL)
===============================================================================
See Appendix B.

===============================================================================
16) RELEASE CHECKLIST
===============================================================================
[ ] Smoke tests pass
[ ] AI tests pass (if real mode enabled)
[ ] Security tests pass
[ ] Mobile regression pass
[ ] No blocker errors in console
[ ] All FAILs resolved or documented

===============================================================================
17) TEST RUN LOG TEMPLATE
===============================================================================
Run #: ___
Date: ___
Tester: ___
Build: dev / preview / prod
Storage Provider: local / supabase
AI Mode: mock / real

FUNCTIONAL TESTS:
[ ] Onboarding: PASS / FAIL
[ ] Navigation & Mobile: PASS / FAIL
[ ] Entry Form: PASS / FAIL
[ ] Ledger (search/filter/export): PASS / FAIL
[ ] Schedule / Tasks: PASS / FAIL
[ ] Recurring Expenses: PASS / FAIL
[ ] Help Calendar: PASS / FAIL
[ ] Medications: PASS / FAIL
[ ] Vault: PASS / FAIL
[ ] Family Invite: PASS / FAIL
[ ] Agent Lab: PASS / FAIL
[ ] Settings + Security: PASS / FAIL
[ ] Data Sync: PASS / FAIL

AI TESTS (REAL MODE):
[ ] Chat Assistant (ledger): PASS / FAIL
[ ] Chat Assistant (external): PASS / FAIL
[ ] Receipt parsing: PASS / FAIL
[ ] Voice parsing: PASS / FAIL
[ ] Medicaid report: PASS / FAIL

SECURITY TESTS:
[ ] Privacy mode masking: PASS / FAIL
[ ] Auto-lock + PIN: PASS / FAIL
[ ] Encryption in storage: PASS / FAIL
[ ] CSP/security headers: PASS / FAIL
[ ] Rate limiting: PASS / FAIL
[ ] RBAC - Admin permissions: PASS / FAIL
[ ] RBAC - Contributor permissions: PASS / FAIL
[ ] RBAC - Viewer permissions: PASS / FAIL
[ ] Security logs display: PASS / FAIL

UI/UX TESTS:
[ ] Toast notifications: PASS / FAIL
[ ] Error boundary recovery: PASS / FAIL
[ ] Theme persistence (Light/Dark/System): PASS / FAIL
[ ] Settlement Details modal: PASS / FAIL

PWA TESTS:
[ ] Service worker register: PASS / FAIL
[ ] Offline shell: PASS / FAIL

Notes:
___________________________________

Overall Result: PASS / FAIL with ___ errors
Security Issues Found: ___

===============================================================================
18) APPENDIX A: PER-SCREEN STEP-BY-STEP SCRIPTS
===============================================================================

A1) Onboarding Wizard
Steps:
1) Clear localStorage and reload app
2) Verify wizard step 1 visible
3) Enter patient name; click Next
4) Set hourly rate; click Next
5) Verify summary step; click Finish
Expected:
- Dashboard loads
- hasCompletedOnboarding true
- Patient name displayed in dashboard copy

A2) Dashboard
Steps:
1) Verify KPIs show totals
2) Confirm Top Contributor matches highest total
3) Hover chart to see tooltip
4) Add entry via Dashboard quick action
Expected:
- KPI totals update
- Chart updates without layout break

A2.5) Settlement Details / Debt Summary
Steps:
1) Click "Settlement Details" button on Dashboard
2) Verify contribution breakdown modal appears
3) Verify each family member's total contribution shown
4) Verify net debt calculations (who owes whom)
5) Close modal
Expected:
- Modal displays all contributors with amounts
- Net amounts calculated correctly (positive = owes, negative = owed)
- Settlement suggestions shown if debts exist
- Modal closes cleanly without UI glitches

A3) Entry Form
Steps:
1) Open Add Entry
2) Create Expense entry (amount > 0)
3) Create Time entry (minutes)
4) Try missing description
Expected:
- Entries saved
- Invalid input blocked

A4) Receipt Upload
Steps:
1) Upload receipt image
2) Confirm fields populate
Expected:
- Mock: placeholder values
- Real: extracted values

A5) Voice Upload
Steps:
1) Upload voice note
2) Confirm parsed fields
Expected:
- Mock: placeholder values
- Real: extracted values

A6) Ledger
Steps:
1) Search by category
2) Filter Expense only
3) Export CSV
4) Delete entry as Admin
Expected:
- Filtered rows correct
- CSV downloads
- Delete removes entry

A7) Schedule
Steps:
1) Add task
2) Mark complete
3) Edit task
4) Convert to ledger
Expected:
- Task moves to Completed
- Ledger entry created

A8) Recurring Expenses
Steps:
1) Add monthly expense (description, amount, category)
2) Verify next due date calculated correctly
3) Edit expense: change amount
4) Edit expense: change frequency to weekly
5) Verify next due date recalculates after frequency change
6) Pause (toggle inactive) recurring expense
7) Verify paused expense moves to "Paused" section
8) Resume paused expense
9) Log expense manually ("Log Now" button)
10) Delete expense
Expected:
- Next due date updates based on frequency
- Paused expenses not auto-logged
- Resume returns expense to active list
- Manual log creates ledger entry
- Deletion removes item from list

A9) Help Calendar
Steps:
1) Add task in calendar
2) Claim task
3) Mark complete
Expected:
- Status updates and displays

A10) Medications
Steps:
1) Add medication (name, dosage, frequency)
2) Log dose as "taken"
3) Log dose as "skipped" with notes
4) Log dose as "late" with notes
5) Edit medication (change dosage)
6) Mark medication as inactive (discontinue)
7) View discontinued medications list
8) Reactivate a discontinued medication
Expected:
- Log entry visible with correct status badge
- "Taken" shows green indicator
- "Skipped" shows yellow indicator with notes
- "Late" shows orange indicator with notes
- Edit changes persist after save
- Inactive list shows discontinued medications
- Reactivated medication returns to active list

A11) Vault
Steps:
1) Add document
2) Toggle emergency mode
Expected:
- Overlay appears and exits

A12) Family Invites
Steps:
1) Create invite
2) Update status to accepted
3) Cancel invite
Expected:
- Status badge updates
- Invite removed on cancel

A13) Family Journal (placeholder)
Steps:
1) Choose photo
Expected:
- Filename shown
- No persistence after reload

A14) Chat Assistant
Steps:
1) Ask ledger question
2) Ask external question
Expected:
- Mock or real responses
- Sources list reflects mode

A15) Medicaid Report
Steps:
1) Run Medicaid analysis
2) Verify loading state shown during analysis
3) Verify entries display with status indicators:
   - COMPLIANT (green) - entry passes compliance check
   - WARNING (yellow) - entry needs attention
   - REVIEW_NEEDED (red) - entry flagged as potential issue
4) Click on a flagged entry to see AI analysis details
5) Verify category suggestion shown for flagged entries
6) Re-run analysis after adding new entry
Expected:
- Mock mode: returns COMPLIANT for all entries
- Real mode: returns mixed statuses based on AI analysis
- Status badges color-coded correctly
- AI reasoning displayed for each flagged entry
- New entries included in re-analysis

A16) Agent Lab
Steps:
1) Open Agent Lab (developer diagnostic tool)
2) Verify agents auto-run on open (or manual trigger)
3) Run "Integrity Agent":
   - Verifies data consistency across entries, tasks, settings
   - Checks for orphaned references
   - Reports any data corruption
4) Run "Privacy Agent":
   - Tests PII scrubbing functionality
   - Verifies names, emails, SSNs, phone numbers removed
   - Shows before/after comparison
5) Run "Scenario Agent":
   - Generates synthetic stress-test data
   - Creates edge-case entries for testing
6) Run "UX Agent":
   - Analyzes user experience patterns
   - Reports potential usability issues
7) Re-run individual agent manually
8) Clear logs and re-run all
Expected:
- Each agent produces distinct output in logs
- Integrity issues flagged with severity
- Privacy scrubbing demonstrated correctly
- Scenario data generated without errors
- Logs persist until manually cleared

A17) Settings
Steps:
1) Toggle theme to Dark, verify UI updates
2) Toggle theme to Light, verify UI updates
3) Toggle theme to System, verify follows OS preference
4) Set PIN and confirm (4 digits)
5) Verify PIN confirmation matches
6) Enable auto-lock
7) Verify lock timeout setting (if configurable)
Expected:
- Theme persists after page refresh
- System theme follows OS dark/light mode
- PIN enables encryption (localStorage values encrypted)
- Incorrect PIN confirmation shows error
- Lock screen appears after idle timeout

A17.5) Security Logs
Steps:
1) Open Settings
2) Navigate to Security Logs / Audit Log section
3) Verify recent security events are listed
4) Verify each log entry shows: timestamp, event type, severity
5) Trigger a new event (e.g., failed PIN attempt)
6) Verify new event appears in logs
Expected:
- Logs display in reverse chronological order (newest first)
- Event types shown: AUTH_SUCCESS, AUTH_FAILURE, SESSION_TIMEOUT,
  EMERGENCY_ACCESS, DATA_RESET, SYSTEM_INIT, SETTINGS_CHANGE
- Severity indicators: INFO (blue), WARNING (yellow), CRITICAL (red)
- Logs persist after page refresh

A18) Data Sync
Steps:
1) Export JSON
2) Import JSON
Expected:
- Import merges data

===============================================================================
19) APPENDIX B: REGRESSION MATRIX TABLE
===============================================================================
Dimensions:
- Device: Desktop (1920x1080), Mobile (390x844 iPhone 14), Tablet (768x1024)
- Theme: Light, Dark, System
- Storage: Local, Supabase
- AI: Mock, Real
- Browser: Chrome, Firefox, Safari (optional)

Minimum Matrix (CI - every push):
1) Desktop + Light + Local + Mock + Chrome
2) Desktop + Dark + Local + Mock + Chrome
3) Mobile + Light + Local + Mock + Chrome
4) Desktop + Light + Supabase + Mock + Chrome
5) Desktop + Light + Local + Real + Chrome

Extended Matrix (Nightly regression):
6) Mobile + Dark + Local + Mock + Chrome
7) Desktop + System + Local + Mock + Chrome
8) Mobile + System + Local + Mock + Chrome
9) Tablet + Light + Local + Mock + Chrome
10) Desktop + Light + Local + Mock + Firefox
11) Desktop + Light + Local + Mock + Safari (macOS only)

Full Matrix (Release candidate):
- All combinations of Device x Theme x Storage x AI
- Prioritize: Mobile + Real AI combinations
- Cross-browser: Chrome, Firefox, Safari, Edge

Matrix Notes:
- "System" theme requires OS-level dark mode toggle verification
- Supabase tests require VITE_SUPABASE_URL configured
- Real AI tests require GEMINI_API_KEY and may be rate-limited
- Mobile tests use Playwright device emulation (Pixel 5, iPhone 14)
- Safari testing requires macOS runner

===============================================================================
20) APPENDIX C: BUG REPORT TEMPLATE
===============================================================================
Title:
Severity:
Environment:
Steps to Reproduce:
Expected:
Actual:
Screenshots/Video:
Console Errors:

===============================================================================
21) APPENDIX D: FIELD VALIDATION MATRIX (EXHAUSTIVE)
===============================================================================

D1) Onboarding Wizard
- Patient Name: required, max 80 chars, allow spaces and hyphen
  - Empty -> error
  - >80 chars -> error
- Hourly Rate: required, numeric, min 0, max 1000
  - Negative -> error
  - Text -> error

D2) Entry Form
- Description: required, max 500
  - Empty -> error
- Amount (Expense): required, numeric, min 0
  - Negative -> error
  - Letters -> error
- Time Duration: required for TIME, integer min 0
  - Negative -> error
- Date: required, YYYY-MM-DD
  - Invalid date -> error
- Category: required
  - Empty -> error

D3) Schedule Task Form
- Title: required
  - Empty -> error
- Due Date: required YYYY-MM-DD
  - Invalid date -> error
- Assignee: required
  - Empty -> error

D4) Recurring Expense
- Description: required
- Amount: numeric min 0
- Frequency: required
- Next Due Date: required YYYY-MM-DD

D5) Help Task
- Title: required
- Category: enum
- Date: required

D6) Medication
- Name: required
- Dosage: required
- Frequency: required
- Monthly Cost: numeric min 0

D7) Settings
- PIN: 4 digits only
  - Non-numeric -> error
  - Length != 4 -> error

D8) Family Invite Form
- Name: required, max 80 chars
  - Empty -> error
  - >80 chars -> error
- Email: required, valid email format
  - Empty -> error
  - Invalid format (no @) -> error
  - Invalid domain -> error
- Role: required, enum (ADMIN, CONTRIBUTOR, VIEWER)
  - Empty -> error

D9) Help Task Form
- Title: required, max 200 chars
  - Empty -> error
- Category: required, enum (meals, transport, errands, companionship, medical, household, other)
  - Invalid category -> error
- Date: required, YYYY-MM-DD
  - Invalid date -> error
  - Past date -> warning (allowed but flagged)
- Time Slot: optional, freeform text, max 50 chars
- Estimated Minutes: optional, numeric, min 0
  - Negative -> error
  - Non-numeric -> error

D10) Document Upload (Vault)
- File: required
  - No file selected -> error
- File size: max 10MB (or configured limit)
  - Oversized file -> error with size limit message
- File type: allowed MIME types (PDF, images, common doc formats)
  - Disallowed type (e.g., .exe, .bat) -> error
- Document name: auto-populated from filename, editable
  - Max 200 chars

D11) Security Settings
- Auto-lock timeout: numeric, min 1 minute, max 60 minutes
  - 0 or negative -> error
  - Non-numeric -> error
- PIN confirmation: must match original PIN
  - Mismatch -> error

===============================================================================
22) APPENDIX E: TEST CASE CATALOG (ID + EXPECTED RESULTS)
===============================================================================

Format: KIN-XXX

KIN-001 App Load
- Steps: Load app
- Expected: UI renders, no fatal errors

KIN-002 Onboarding Required
- Steps: Clear localStorage, reload
- Expected: onboarding wizard appears

KIN-003 Onboarding Completion
- Steps: complete onboarding
- Expected: dashboard loads, hasCompletedOnboarding true

KIN-010 Add Expense Entry
- Steps: add expense
- Expected: ledger updated with entry

KIN-011 Add Time Entry
- Steps: add time entry
- Expected: ledger updated, time value calculated

KIN-020 Ledger Search
- Steps: search term
- Expected: results filtered correctly

KIN-021 Ledger Export
- Steps: export CSV
- Expected: CSV file downloads, rows match filter

KIN-030 Task Add/Edit
- Steps: create/edit task
- Expected: updates visible

KIN-031 Task Complete
- Steps: mark complete
- Expected: task appears in Completed

KIN-032 Task Convert
- Steps: log to ledger
- Expected: ledger entry created

KIN-040 Recurring Expense Add
- Steps: create recurring expense
- Expected: list shows item

KIN-050 Help Task Claim
- Steps: claim/unclaim
- Expected: status updates

KIN-060 Medication Add/Log
- Steps: add med, log dose
- Expected: log visible

KIN-070 Vault Emergency Mode
- Steps: toggle emergency
- Expected: overlay appears

KIN-080 Family Invite
- Steps: create invite
- Expected: list updates

KIN-090 Chat Assistant Mock
- Steps: ask question
- Expected: mock response

KIN-091 Chat Assistant Real
- Steps: ask external query
- Expected: sources list shown

KIN-100 Medicaid Report
- Steps: run analysis
- Expected: items render with status

KIN-110 PIN Lock
- Steps: set pin, idle
- Expected: lock screen

KIN-120 Mobile Nav
- Steps: tap floating hamburger
- Expected: sidebar opens

KIN-111 PIN Failed Attempts Logging
- Steps: Enter wrong PIN 3 times
- Expected: Each failure logged to security logs with AUTH_FAILURE

KIN-112 RBAC - Viewer Cannot Delete
- Steps: Configure user as VIEWER role, navigate to Ledger
- Expected: Delete buttons not visible or disabled for VIEWER

KIN-113 RBAC - Contributor Add Own Entry
- Steps: Configure user as CONTRIBUTOR, add new entry
- Expected: Entry created successfully, attributed to contributor

KIN-114 RBAC - Contributor Cannot Delete Others
- Steps: As CONTRIBUTOR, view entry created by another user
- Expected: Delete button not available for others' entries

KIN-115 RBAC - Admin Full Access
- Steps: As ADMIN, delete any entry, modify settings
- Expected: All actions permitted

KIN-121 Mobile Safe Area Padding
- Steps: View app on device with notch/safe area
- Expected: Content respects safe-area-inset, no overlap with notch

KIN-122 Mobile Keyboard Handling
- Steps: Open form on mobile, focus input field
- Expected: Keyboard appears, form scrolls to keep input visible

KIN-130 Agent Lab Integrity Check
- Steps: Corrupt localStorage data, run Integrity Agent
- Expected: Issues detected and reported in agent logs

KIN-131 Agent Lab Privacy Scrubbing
- Steps: Add entry with PII, run Privacy Agent
- Expected: Shows PII detected and scrubbed version

KIN-140 Toast Success Notification
- Steps: Create new entry successfully
- Expected: Success toast appears, auto-dismisses

KIN-141 Toast Error Notification
- Steps: Trigger validation error (empty required field)
- Expected: Error toast appears with message

KIN-142 Toast Stacking
- Steps: Trigger multiple actions rapidly
- Expected: Toasts stack correctly, dismiss in order

KIN-150 Theme System Mode
- Steps: Set theme to "System", change OS dark/light preference
- Expected: App theme follows OS preference

KIN-151 Theme Persistence
- Steps: Set theme to Dark, refresh page
- Expected: Dark theme persists after refresh

KIN-160 Settlement Details Modal
- Steps: Click "Settlement Details" on Dashboard
- Expected: Modal shows contribution breakdown and net debts

KIN-161 Debt Calculation Accuracy
- Steps: Add entries for multiple users, view Settlement Details
- Expected: Net amounts calculated correctly

KIN-170 Security Logs Display
- Steps: Open Settings > Security Logs
- Expected: Recent events listed with timestamp, type, severity

KIN-171 Security Log Event Creation
- Steps: Trigger security event (failed PIN), check logs
- Expected: New event appears in security logs

KIN-180 Error Boundary Recovery
- Steps: Trigger component error (if possible via dev tools)
- Expected: Fallback UI shown, not blank screen

KIN-190 Medication Dose Status Variations
- Steps: Log doses as taken, skipped, and late
- Expected: Each status shows correct indicator and notes

KIN-191 Medication Reactivation
- Steps: Discontinue medication, then reactivate
- Expected: Medication returns to active list

KIN-200 Recurring Expense Frequency Change
- Steps: Change recurring expense from monthly to weekly
- Expected: Next due date recalculates correctly

KIN-201 Recurring Expense Manual Log
- Steps: Click "Log Now" on active recurring expense
- Expected: Ledger entry created, next due date advances

KIN-210 Medicaid Status Indicators
- Steps: Run Medicaid analysis with varied entries
- Expected: COMPLIANT/WARNING/REVIEW_NEEDED shown with correct colors

KIN-211 Medicaid AI Analysis Details
- Steps: Click on flagged Medicaid entry
- Expected: AI reasoning and category suggestion displayed

===============================================================================
23) APPENDIX F: DATA PERSISTENCE & MIGRATION MATRIX
===============================================================================

Local Storage:
- Entries persist after refresh
- Tasks persist after refresh
- Settings persist after refresh
- Security logs persist

Supabase:
- Entries persist across devices
- Tasks persist across devices
- Settings persist across devices

Migration:
- Local -> Supabase: one-way sync
- Verify no data loss

KNOWN LIMITATIONS (as of 2026-01-30)
- Family Journal photo upload is placeholder only (no preview, no persistence)
- AI features default to mock unless VITE_GEMINI_MOCK=false
- Supabase sync requires external configuration

