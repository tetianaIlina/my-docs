ISSUER ONBOARDING - QA & TESTING CHECKLIST (ESSENTIAL)
=======================================================

Document Version: 2.1 (with detailed test criteria)
Date: 2026-02-20
QA Engineer: ___________________________
Status: In Progress

TEST STATUS LEGEND
==================
✓ = Test Passed
✗ = Test Failed
○ = Not Tested
~ = Blocked/Skipped
N/A = Not Applicable


OVERVIEW
========
This is a streamlined checklist covering CRITICAL (P0) and MAJOR (P1) test cases for issuer onboarding.
Focus areas: Core API, Decision rules, Database integrity, UI workflows, Error handling.


================================================================================
TEST ENVIRONMENT SETUP
================================================================================

Item                                              | Status | Notes
--------------------------------------------------|--------|------------------
Test environment configured (staging)             | ○      |
API authentication tokens obtained                | ○      |
Database access configured                        | ○      |
Partner Platform Integration Service running      | ○      |
Decision Engine accessible                        | ○      |


================================================================================
SECTION 1: CRITICAL API TESTS (P0)
================================================================================

1.1 Create Issuer - Happy Path
-------------------------------

ID    | Test Description                           | Criteria                                          | Pass/Fail | Notes
------|--------------------------------------------|----------------------------------------------------|-----------|-------
TC001 | Create issuer with valid data              | POST /api/v1/issuer/event with: organisationNumber (valid format), country (SE/NO/DK/FI), name, channelId → HTTP 201 | ○ |
TC002 | Verify issuer record in database           | Query: SELECT * FROM issuers WHERE organisation_number = '...' → Record exists with correct data | ○ |
TC003 | Verify status is AWAITING_ACTIVATION       | issuer.status = 'AWAITING_ACTIVATION' in database | ○ |
TC004 | Kafka event published successfully         | Event on topic 'issuer.created' with correct payload (issuerId, org#, country, timestamp) | ○ |


1.2 Create Issuer - Critical Negative Tests
---------------------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
TC101  | Create without authentication (401)       | No Authorization header → HTTP 401, message: "Unauthorized" | ○ |
TC102  | Create with missing org number (422)      | Request body missing organisationNumber → HTTP 422, validation error | ○ |
TC103  | Create with missing country code (422)    | Request body missing country → HTTP 422, validation error | ○ |
TC104  | Create duplicate issuer (409)             | Same org# + country exists → HTTP 409, message: "Issuer already exists" | ○ |


1.3 Get Issuer Status
----------------------

ID    | Test Description                           | Criteria                                          | Pass/Fail | Notes
------|--------------------------------------------|----------------------------------------------------|-----------|-------
TC201 | GET issuer status returns correct data     | GET /api/v1/internal/issuer/:id → Returns aggregatedStatus, systemStatus, financialPartnerStatus | ○ |
TC202 | Verify aggregated status displayed         | Response contains status enum: ACTIVE / AWAITING_ACTIVATION / INACTIVE / NEED_ACTION | ○ |
TC203 | 404 for non-existent issuer                | GET with non-existent UUID → HTTP 404, message: "Issuer not found" | ○ |


1.4 Invoice Creation - Status Check
-------------------------------------

ID    | Test Description                           | Criteria                                          | Pass/Fail | Notes
------|--------------------------------------------|----------------------------------------------------|-----------|-------
TC301 | Invoice ALLOWED for ACTIVE issuer          | Issuer with aggregatedStatus = 'ACTIVE' → POST /api/v1/invoice/event → HTTP 201 | ○ |
TC302 | Invoice BLOCKED for AWAITING_ACTIVATION    | Issuer status = 'AWAITING_ACTIVATION' → POST invoice → HTTP 403, error: "Issuer not active" | ○ |
TC303 | Invoice BLOCKED for INACTIVE issuer        | Issuer status = 'INACTIVE' → POST invoice → HTTP 403, error: "Issuer not active" | ○ |
TC304 | Error message is clear                     | Error response includes: status reason, issuerId, helpful message | ○ |


================================================================================
SECTION 2: DECISION RULES TESTING (9 MANDATORY RULES) - P0
================================================================================

2.1 Rule 1: Industry Code (D&B)
---------------------------------
Data Source: Dun & Bradstreet
Decision: Industry code must NOT be in the denied list

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR101  | Approved industry code → GRANTED          | D&B industry code in approved list: 47111, 47190, 46900 (retail/wholesale) → decision = GRANTED | ○ |
DR102  | Denied industry code → DENIED             | D&B industry code in denied list: 64191, 64929, 66300 (financial/gambling/betting) → decision = DENIED | ○ |
DR103  | Missing industry code → REFERRED          | D&B returns null/empty industry code → decision = REFERRED, requires manual review | ○ |


2.2 Rule 2: Failure Score (D&B)
---------------------------------
Data Source: Dun & Bradstreet
Decision: Failure score must be < 25 (lower is better)

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR201  | Score < 25 → GRANTED                      | D&B failure score: 0-24 (e.g., score = 15) → decision = GRANTED | ○ |
DR202  | Score >= 25 → DENIED                      | D&B failure score: 25 or higher (e.g., score = 30, 50, 75) → decision = DENIED | ○ |
DR203  | Missing score → REFERRED                  | D&B returns null failure score → decision = REFERRED | ○ |


2.3 Rule 3: Address Validation (D&B)
--------------------------------------
Data Source: Dun & Bradstreet + AML Manager high-risk countries list
Decision: Registered office address must NOT be in high-risk country

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR301  | Valid address (safe country) → GRANTED    | D&B address country in approved list: SE, NO, DK, FI, DE, UK, NL, FR → decision = GRANTED | ○ |
DR302  | High-risk country address → DENIED        | D&B address country in high-risk list: IR (Iran), KP (North Korea), SY (Syria), AF (Afghanistan) → decision = DENIED | ○ |
DR303  | Missing address → REFERRED                | D&B returns null/empty address or country → decision = REFERRED | ○ |


2.4 Rule 4: Sanction List (Creditsafe)
----------------------------------------
Data Source: Creditsafe PEP & Sanction Lists
Decision: Company or beneficial owners NOT on sanction list

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR401  | Not on sanction list → GRANTED            | Creditsafe check for company + all BOs returns: sanctionMatch = false → decision = GRANTED | ○ |
DR402  | On sanction list → DENIED                 | Creditsafe returns: sanctionMatch = true, matchedEntity name provided → decision = DENIED | ○ |
DR403  | Creditsafe API unavailable → REFERRED     | Creditsafe API timeout (> 30s) or 500/503 error → decision = REFERRED, retry later | ○ |


2.5 Rule 5: KYC Approval (Partner Platform API + Manual)
----------------------------------------------
Data Source: Partner Platform API + Manual Operator Review
Decision: KYC information complete and manually approved by operations team

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR501  | KYC data complete → Ready for approval    | Partner Platform returns: 1-4 beneficial owners with: name, DOB, citizenship, address, ownership% (≥25%) → status = PENDING_APPROVAL | ○ |
DR502  | Manual approve → GRANTED                  | Operator reviews KYC tab, clicks "Approve KYC" → decision = GRANTED, kycApprovalDate set | ○ |
DR503  | Manual deny → DENIED                      | Operator clicks "Deny KYC", enters reason (required field) → decision = DENIED | ○ |
DR504  | Missing beneficial owners → REFERRED      | Partner Platform returns 0 BOs OR incomplete BO data (missing name/DOB/etc) → decision = REFERRED | ○ |


2.6 Rule 6: Contact Information (Partner Platform API)
--------------------------------------------
Data Source: Partner Platform API
Decision: Valid phone number AND email address required

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR601  | Contact info complete → GRANTED           | Partner Platform returns: phoneNumber (format: +46XXXXXXXXX or +47XXXXXXXX) AND email (valid format) → decision = GRANTED | ○ |
DR602  | Missing phone/email → DENIED              | Partner Platform returns: null phone OR null email OR invalid format → decision = DENIED | ○ |


2.7 Rule 7: IBAN Validation (Partner Platform API)
----------------------------------------
Data Source: Partner Platform API
Decision: Valid IBAN format from supported country

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR701  | Valid IBAN → GRANTED                      | IBAN format valid: starts with SE/NO/DK/FI, correct length (SE:24, NO:15, DK:18, FI:18 chars), passes mod-97 checksum → decision = GRANTED | ○ |
DR702  | Invalid IBAN → DENIED                     | IBAN: wrong length, invalid checksum, unsupported country (e.g., RU, CN) → decision = DENIED | ○ |
DR703  | Missing IBAN → REFERRED                   | Partner Platform returns null/empty IBAN → decision = REFERRED | ○ |


2.8 Rule 8: Agreement Validation (Partner Platform API)
---------------------------------------------
Data Source: Partner Platform API
Decision: Signed financial agreement must exist

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR801  | Signed agreement → GRANTED                | Partner Platform returns: agreementId (not null), signedDate (valid timestamp), signatoryName → decision = GRANTED | ○ |
DR802  | Missing agreement → DENIED                | Partner Platform returns: null agreementId OR signedDate = null OR agreement not signed → decision = DENIED | ○ |


2.9 Rule 9: Blacklist Check (Internal)
----------------------------------------
Data Source: Internal Blacklist Service
Decision: Organisation number NOT on internal denied list

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DR901  | Not on blacklist → GRANTED                | Blacklist Service query (org# + country) returns: blacklisted = false → decision = GRANTED | ○ |
DR902  | On blacklist → DENIED                     | Blacklist Service returns: blacklisted = true, reason = "fraud" or "default" → decision = DENIED | ○ |


2.10 High-Risk Flags
---------------------
Data Source: Creditsafe (PEP), AML Manager (high-risk lists)

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
HR101  | PEP detected → HIGH RISK flagged          | Creditsafe returns: any BO with pepStatus = true → Flag type: HIGH_RISK_PEP, requiresEDD = true | ○ |
HR102  | High-risk country → HIGH RISK flagged     | BO citizenship/tax residence in AML Manager list: IR, KP, SY, AF, YE, MM, VE → Flag type: HIGH_RISK_COUNTRY, requiresEDD = true | ○ |
HR103  | High-risk industry → HIGH RISK flagged    | D&B industry code in AML Manager list: 64191, 64929, 66300, 92000 (finance/gambling/adult) → Flag type: HIGH_RISK_INDUSTRY, requiresEDD = true | ○ |
HR104  | HIGH RISK requires EDD                    | If any HIGH_RISK flag exists → issuer.requiresEDD = true, KYC cannot be approved until EDD complete | ○ |


================================================================================
SECTION 3: DATABASE VALIDATION (P0)
================================================================================

3.1 Core Tables
----------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DB101  | Issuer record created in issuers table    | Query: SELECT * FROM issuers WHERE id = '...' → Record exists with: organisation_number, country, name, status, tenant_id, created_at | ○ |
DB102  | Bank account saved correctly              | Query: SELECT * FROM bank_accounts WHERE issuer_id = '...' → Record exists with: iban, bank_name, country, currency | ○ |
DB103  | Contact info saved correctly              | Query: SELECT * FROM contact_information WHERE issuer_id = '...' → Record with: phone_number, email, contact_person | ○ |
DB104  | Beneficial owners saved (if provided)     | Query: SELECT * FROM beneficial_owners WHERE issuer_id = '...' → 1-4 records with: name, date_of_birth, ownership_percentage | ○ |
DB105  | Status history logged                     | Query: SELECT * FROM status_history WHERE issuer_id = '...' → Record with: old_status, new_status, changed_by, changed_at | ○ |
DB106  | Decision rules results saved              | Query: SELECT * FROM decision_rules WHERE issuer_id = '...' → 9 records (one per rule) with: rule_name, decision, evaluated_at | ○ |


3.2 Data Integrity
-------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
DB201  | No duplicate issuers allowed              | UNIQUE constraint on (organisation_number, country, tenant_id) → Insert duplicate returns error | ○ |
DB202  | Foreign key constraints enforced          | Try INSERT bank_account with non-existent issuer_id → Database rejects with FK constraint error | ○ |
DB203  | Required fields validated                 | NOT NULL constraints on: organisation_number, country, name, status, tenant_id → Cannot insert null | ○ |
DB204  | Audit trail created for changes           | Query: SELECT * FROM audit_logs WHERE entity_id = '...' → Log entry for: CREATE, UPDATE, STATUS_CHANGE events | ○ |


================================================================================
SECTION 4: THIRD-PARTY INTEGRATIONS (P0)
================================================================================

4.1 Dun & Bradstreet (D&B)
---------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
INT101 | D&B API returns industry code             | Call D&B with org# → Response HTTP 200, body contains: industryCode (5 digits), description → Log response | ○ |
INT102 | D&B API returns failure score             | D&B response contains: failureScore (0-100 integer) → Log score value | ○ |
INT103 | D&B API returns company address           | D&B response contains: address object with street, city, postalCode, country (ISO code) | ○ |
INT104 | Handle D&B API timeout gracefully         | Simulate timeout (> 30s) → System logs error, sets rule decision = REFERRED, doesn't crash | ○ |


4.2 Creditsafe
---------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
INT201 | Creditsafe returns PEP status             | Call Creditsafe PEP check → Response HTTP 200, body contains: pepStatus (true/false) for each BO | ○ |
INT202 | Creditsafe returns sanction list check    | Creditsafe sanction check → Response contains: sanctionMatch (boolean), matchedEntity (if true) | ○ |
INT203 | Handle Creditsafe API error               | Simulate 503 error → System logs error, sets rule decision = REFERRED, retries 3 times | ○ |


4.3 Partner Platform API
----------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
INT301 | Board issuer to Partner Platform successfully         | POST to Partner Platform /creditor/board → HTTP 201, creditorId returned, stored in database | ○ |
INT302 | Get KYC data from Partner Platform                    | GET /creditor/:id/kyc → HTTP 200, returns: beneficialOwners array with complete data | ○ |
INT303 | Get agreement data from Partner Platform              | GET /creditor/:id/agreements → HTTP 200, returns: agreementId, signedDate, signatoryName | ○ |
INT304 | Get IBAN from Partner Platform                        | GET /creditor/:id/bank-account → HTTP 200, returns: iban, bankName, currency | ○ |
INT305 | Handle Partner Platform API error (500)               | Simulate 500 error → System logs error, sets affected rules to REFERRED, sends alert | ○ |


================================================================================
SECTION 5: STATUS MANAGEMENT (P0)
================================================================================

5.1 Status Transitions
-----------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
ST101  | New issuer → AWAITING_ACTIVATION          | On CREATE: issuer.aggregatedStatus = 'AWAITING_ACTIVATION', systemStatus = 'PENDING', financialPartnerStatus = 'NOT_BOARDED' | ○ |
ST102  | All rules GRANTED → ACTIVE                | When all 9 decision rules have decision = 'GRANTED' → aggregatedStatus changes to 'ACTIVE' automatically | ○ |
ST103  | Any rule DENIED → INACTIVE                | When any decision rule has decision = 'DENIED' → aggregatedStatus = 'INACTIVE', issuer cannot proceed | ○ |
ST104  | Rules REFERRED → NEED_ACTION              | When any rule decision = 'REFERRED' AND not all GRANTED → aggregatedStatus = 'NEED_ACTION', requires manual intervention | ○ |
ST105  | HIGH RISK flag → Requires EDD             | When requiresEDD = true → issuer cannot move to ACTIVE until EDD completed and approved | ○ |


5.2 Status Enforcement
-----------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
ST201  | Only ACTIVE issuer can create invoices    | Issuer with aggregatedStatus = 'ACTIVE' → POST /api/v1/invoice/event → HTTP 201 (allowed) | ○ |
ST202  | Non-ACTIVE issuer blocked from invoices   | Issuer status ≠ 'ACTIVE' (e.g., AWAITING_ACTIVATION, INACTIVE) → POST invoice → HTTP 403, clear error message | ○ |
ST203  | Status change logged in history           | Every status change → INSERT into status_history table with: old_status, new_status, reason, changed_by, changed_at | ○ |


================================================================================
SECTION 6: KAFKA EVENT PROCESSING (P1)
================================================================================

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
KF101  | Issuer created event published            | Topic: 'issuer.created', payload contains: issuerId, organisationNumber, country, timestamp, eventId | ○ |
KF102  | Issuer boarding event consumed            | Consumer processes 'issuer.boarding.complete' event → Triggers decision rules evaluation | ○ |
KF103  | Decision rules event published            | Topic: 'issuer.decision-rules.evaluated', payload: issuerId, all 9 rule results with decisions | ○ |
KF104  | Status change event published             | Topic: 'issuer.status.changed', payload: issuerId, oldStatus, newStatus, reason, timestamp | ○ |
KF105  | Event idempotency - duplicate ignored     | Send same event twice (same eventId) → Processed once only, duplicate logged and ignored | ○ |


================================================================================
SECTION 7: SECURITY TESTING (P0)
================================================================================

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
SEC101 | Authentication required for all endpoints | Call any endpoint without Authorization header → HTTP 401, message: "Unauthorized" | ○ |
SEC102 | Invalid JWT rejected (401)                | Use malformed/tampered JWT token → HTTP 401, message: "Invalid token" | ○ |
SEC103 | Expired JWT rejected (401)                | Use JWT with exp timestamp in past → HTTP 401, message: "Token expired" | ○ |
SEC104 | SQL injection prevented                   | Send org# = "'; DROP TABLE issuers; --" → Request safely handled, no SQL execution, validation error returned | ○ |
SEC105 | Unauthorized access blocked (403)         | User without 'issuer:write' permission → POST /issuer/event → HTTP 403, message: "Forbidden" | ○ |


================================================================================
SECTION 8: UI TESTING - ADMIN PORTAL (P0)
================================================================================

8.1 Issuer List View
---------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UI001  | Issuer list loads successfully            | Navigate to /issuers page → Page loads in < 2s, table displays with columns: org#, name, country, status | ○ |
UI002  | Status badge displayed for each issuer    | Each row shows status badge: ACTIVE (green), INACTIVE (red), AWAITING_ACTIVATION (yellow), NEED_ACTION (orange) | ○ |
UI003  | Filter by status (ACTIVE/INACTIVE) works  | Select status filter dropdown → Click ACTIVE → List shows only ACTIVE issuers | ○ |
UI004  | Search by org number works                | Enter org# in search box → Press Enter → List filters to matching issuer(s) | ○ |


8.2 Issuer Detail Page - Main Tab
-----------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UI101  | Issuer detail page loads                  | Click issuer from list → Detail page loads in < 1s with all tabs visible | ○ |
UI102  | Company info displayed correctly          | Page shows: company name, organisation number, country code, Partner Platform creditor ID | ○ |
UI103  | Status badge visible and accurate         | Status badge matches database: color and text correct (ACTIVE/INACTIVE/etc) | ○ |
UI104  | D&B data displayed (industry, score)      | Section shows: Industry Code (5 digits + description), Failure Score (number 0-100), Company Address | ○ |


8.3 KYC Tab - Review & Approve
--------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UI201  | KYC tab loads with all data               | Click KYC tab → Loads in < 1s, displays: business financing method, purpose, expected financing, BOs | ○ |
UI202  | Beneficial owners list displayed          | Shows 1-4 beneficial owners with: full name, DOB, citizenship, ownership %, address | ○ |
UI203  | PEP status visible for each BO            | Each BO row shows: PEP badge (YES in red / NO in green) clearly visible | ○ |
UI204  | "Approve KYC" button works                | Button visible and enabled → Click button → Confirmation modal opens | ○ |
UI205  | Confirmation dialog appears               | Modal title: "Approve KYC?", with approval notes textarea, Cancel + Confirm buttons | ○ |
UI206  | Success message after approval            | Click Confirm → Green toast appears: "KYC approved successfully" | ○ |
UI207  | KYC status updates to GRANTED             | After approval → KYC badge changes from PENDING to GRANTED (green), timestamp updated | ○ |


8.4 Decision Rules Tab
-----------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UI301  | Decision Rules tab loads                  | Click Decision Rules tab → Loads in < 1s, shows list of 9 rules with status badges | ○ |
UI302  | All 9 rules displayed                     | List shows: Industry Code, Failure Score, Address, Sanction List, KYC, Contact, IBAN, Agreement, Blacklist | ○ |
UI303  | Status badges correct (GREEN/RED/YELLOW)  | GRANTED = green checkmark, DENIED = red X, REFERRED = yellow warning icon | ○ |
UI304  | GRANTED shown with green checkmark        | Rule with decision = GRANTED → Badge displays: ✓ GRANTED in green color | ○ |
UI305  | DENIED shown with red X                   | Rule with decision = DENIED → Badge displays: ✗ DENIED in red color | ○ |
UI306  | Expand rule to view details               | Click rule row → Expands to show: data source, evaluation result JSON, timestamp, evaluated by | ○ |


8.5 High-Risk & EDD Tab
------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UI401  | HIGH RISK badge visible on issuer         | Issuer with requiresEDD = true → Red "HIGH RISK" badge displayed prominently on detail page header | ○ |
UI402  | High-Risk tab shows all flags             | Click High-Risk tab → Shows list of flags: type (PEP/COUNTRY/INDUSTRY), details, BO name if PEP | ○ |
UI403  | EDD tab accessible for high-risk issuers  | HIGH RISK issuer → EDD tab visible and clickable | ○ |
UI404  | Upload EDD documents works                | Click "Upload Document" → File picker opens → Select PDF/DOCX → File uploads, shows in list | ○ |
UI405  | Save EDD findings button works            | Enter notes, upload docs → Click "Save EDD" → Success message, eddCompletedAt timestamp set | ○ |


8.6 History/Audit Tab
----------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UI501  | History tab loads                         | Click History tab → Loads in < 1s, shows chronological list of events (newest first) | ○ |
UI502  | Status changes displayed with timestamps  | Each status change shows: old status → new status, timestamp (DD/MM/YYYY HH:MM:SS) | ○ |
UI503  | User who made change shown                | Each event shows: "Changed by: user@example.com" or "System" for automated changes | ○ |


================================================================================
SECTION 9: UI ERROR HANDLING (P0)
================================================================================

9.1 Form Validation
--------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UE101  | Empty required field shows error          | Leave org# field empty → Click Submit → Error appears: "Organisation number is required" in red below field | ○ |
UE102  | Error message is clear                    | Error text is specific, actionable, e.g., "Please enter a valid email address" (not just "Invalid input") | ○ |
UE103  | Cannot submit with validation errors      | Form has validation errors → Submit button disabled OR click shows errors, form not submitted | ○ |


9.2 API Error Display
----------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UE201  | 401 Unauthorized - redirect to login      | Session expired → Any API call returns 401 → Error toast + redirect to /login within 2s | ○ |
UE202  | 404 Not Found - clear message             | API returns 404 → Error banner: "Issuer not found. It may have been deleted." with "Back to List" button | ○ |
UE203  | 409 Duplicate - explains conflict         | Create duplicate → 409 error → Message: "Issuer with org# 123456789 (SE) already exists. [View Existing]" | ○ |
UE204  | 500 Server error - user-friendly message  | API returns 500 → Error: "Something went wrong. Please try again or contact support." with "Retry" button | ○ |
UE205  | Network timeout - retry option shown      | API timeout → Error banner: "Request timed out. Check your connection." with "Retry" button | ○ |


9.3 Status-Based Errors
------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UE301  | AWAITING_ACTIVATION - explains waiting    | Info message: "Issuer is awaiting activation from Partner Platform. This may take 24-48 hours." | ○ |
UE302  | INACTIVE - shows warning                  | Red warning banner: "This issuer is INACTIVE due to: [reason]. Cannot create invoices." | ○ |
UE303  | Cannot create invoice - clear error msg   | Try create invoice for non-ACTIVE → Error: "Invoice creation not allowed. Issuer must be ACTIVE." | ○ |
UE304  | Invoice button disabled for non-ACTIVE    | "Create Invoice" button grayed out, tooltip on hover: "Issuer must be ACTIVE to create invoices" | ○ |


9.4 Decision Rule Error Display
---------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
UE401  | DENIED rule shown in red                  | DENIED rule has: red background, ✗ icon, bold "DENIED" text, clear visual separation from other rules | ○ |
UE402  | Clear explanation of denial reason        | Expanded DENIED rule shows specific reason, e.g., "Industry code 64191 (Banking) is not permitted" | ○ |
UE403  | Cannot override DENIED rule in UI         | No "Override" or "Approve" button for DENIED rules, only "View Details" and "Contact Support" | ○ |
UE404  | Contact support option visible            | DENIED rule shows: "Contact support: support@moank.se" or "Submit Manual Review Request" button | ○ |


================================================================================
SECTION 10: UI USER WORKFLOWS (P0)
================================================================================

10.1 Complete Workflow: Onboard Issuer (Happy Path)
-----------------------------------------------------

ID      | Test Description                         | Criteria                                          | Pass/Fail | Notes
--------|------------------------------------------|----------------------------------------------------|-----------|-------
WF101   | Navigate to "Create Issuer" page         | Click "New Issuer" button in top-right → Form page loads with empty fields | ○ |
WF102   | Fill in company info (org#, name, ctry)  | Enter: org# = "556677889", name = "Test AB", country = "SE" (dropdown) | ○ |
WF103   | Fill in banking info (IBAN, bank)        | Enter: IBAN = "SE1234567890123456789012", bank_name = "Swedbank" | ○ |
WF104   | Submit form successfully                 | Click "Create Issuer" → Loading spinner → Form submits, API returns 201 | ○ |
WF105   | Success message displayed                | Green toast appears: "Issuer created successfully" auto-dismisses after 5s | ○ |
WF106   | Redirected to issuer detail page         | URL changes to /issuers/:id, detail page loads showing created issuer | ○ |
WF107   | Status shows AWAITING_ACTIVATION         | Status badge: yellow "AWAITING_ACTIVATION", info message explains waiting for Partner Platform | ○ |


10.2 Complete Workflow: Approve KYC
-------------------------------------

ID      | Test Description                         | Criteria                                          | Pass/Fail | Notes
--------|------------------------------------------|----------------------------------------------------|-----------|-------
WF201   | Navigate to issuer detail                | From list, click issuer row → Detail page opens with Main tab active | ○ |
WF202   | Review Main tab (D&B data)               | Verify present: Industry Code (e.g., 47111 - Retail), Failure Score (e.g., 15), Address (complete) | ○ |
WF203   | Switch to KYC tab                        | Click "KYC" tab → Tab activates, content loads in < 1s | ○ |
WF204   | Review beneficial owners & PEP status    | Check: 1-4 BOs listed, each with PEP status badge (YES/NO), ownership % shown (must sum ≥25%) | ○ |
WF205   | Switch to Decision Rules tab             | Click "Decision Rules" tab → View all 9 rules | ○ |
WF206   | Verify all rules GRANTED/REFERRED        | Check: No rules show DENIED (red X), only ✓ GRANTED (green) or ⚠ REFERRED (yellow) | ○ |
WF207   | Return to KYC tab                        | Click "KYC" tab to return | ○ |
WF208   | Click "Approve KYC" button               | Button enabled (green) → Click → Confirmation modal appears | ○ |
WF209   | Enter approval notes in modal            | Modal has textarea "Approval Notes" → Enter: "All documents verified. Approved." | ○ |
WF210   | Confirm approval                         | Click "Confirm" button → Modal closes, API call sent | ○ |
WF211   | Success toast displayed                  | Green toast: "KYC approved successfully" appears top-right | ○ |
WF212   | KYC status changes to GRANTED            | KYC badge updates from PENDING → ✓ GRANTED (green), page refreshes with new data | ○ |


10.3 Complete Workflow: Handle High-Risk Issuer
-------------------------------------------------

ID      | Test Description                         | Criteria                                          | Pass/Fail | Notes
--------|------------------------------------------|----------------------------------------------------|-----------|-------
WF301   | Identify HIGH RISK badge on issuer       | Issuer list/detail shows: red "HIGH RISK" badge prominently displayed | ○ |
WF302   | Open High-Risk tab                       | Click "High-Risk" tab → Tab opens, shows warning banner | ○ |
WF303   | Review all flags (PEP, Country, Ind)     | List shows flags with: type, details (e.g., "BO John Doe is PEP", "Registered in Iran") | ○ |
WF304   | Navigate to EDD tab                      | Click "EDD" tab (or "Perform EDD" button) → EDD form opens | ○ |
WF305   | Enter EDD investigation notes            | Textarea for notes → Enter: "Reviewed source of funds. Business legitimate. Low risk assessed." (min 50 chars) | ○ |
WF306   | Upload supporting documents              | Click "Upload" → Select PDF → File uploads successfully, appears in documents list | ○ |
WF307   | Save EDD findings                        | Click "Save EDD Findings" → API call succeeds, eddCompletedAt timestamp set | ○ |
WF308   | EDD marked complete in history           | History tab shows new entry: "EDD completed by user@example.com" with timestamp | ○ |
WF309   | Return to KYC tab                        | Click "KYC" tab → High-risk flag still visible but EDD complete checkmark shown | ○ |
WF310   | Approve KYC after EDD complete           | "Approve KYC" button now enabled → Click → Approval succeeds, issuer can proceed | ○ |


10.4 Complete Workflow: DENIED Rule Handling
----------------------------------------------

ID      | Test Description                         | Criteria                                          | Pass/Fail | Notes
--------|------------------------------------------|----------------------------------------------------|-----------|-------
WF401   | Open issuer with DENIED rule             | Navigate to issuer with at least one rule decision = DENIED → Status should be INACTIVE | ○ |
WF402   | Navigate to Decision Rules tab           | Click "Decision Rules" tab → Tab loads showing all 9 rules | ○ |
WF403   | Identify DENIED rule (red indicator)     | Visual scan: One or more rules show: ✗ DENIED in red with red background highlight | ○ |
WF404   | Expand rule to view details              | Click DENIED rule row → Expands to show: data source, denial reason, evaluation result | ○ |
WF405   | Read denial reason clearly               | Expanded view shows specific reason, e.g., "Failure score 45 exceeds threshold of 25" | ○ |
WF406   | Understand cannot proceed                | No "Override" or "Approve" buttons, message: "This issuer cannot be activated due to DENIED rules" | ○ |
WF407   | Issuer remains INACTIVE                  | Status badge stays INACTIVE (red), no way to change in UI | ○ |


================================================================================
SECTION 11: UI ACCESSIBILITY & COMPATIBILITY (P1)
================================================================================

11.1 Browser Compatibility
---------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
BR101  | Chrome (latest) - all features work       | Test in Chrome v120+ → All workflows complete, no console errors, layout correct | ○ |
BR102  | Firefox (latest) - all features work      | Test in Firefox v121+ → All workflows complete, no console errors, layout correct | ○ |
BR103  | Safari (latest) - all features work       | Test in Safari v17+ → All workflows complete, no console errors, layout correct | ○ |


11.2 Basic Accessibility
-------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
AC101  | Keyboard navigation works                 | Use Tab key only → Can navigate to all interactive elements (buttons, links, form fields) without mouse | ○ |
AC102  | Tab order is logical                      | Tab order follows: top→bottom, left→right reading flow, no focus jumping randomly | ○ |
AC103  | Focus indicators visible                  | Each focused element has clear blue outline (2px solid), visible against all backgrounds | ○ |
AC104  | Color contrast meets WCAG AA              | Check with contrast checker: text/background ratio ≥ 4.5:1, status badges readable | ○ |


================================================================================
SECTION 12: PERFORMANCE (P1)
================================================================================

ID     | Test Description                          | Target    | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|-----------|---------------------------------------------------|-----------|-------
PF101  | Create issuer API response time           | < 2s      | Send POST request → Measure time from send to response received → Must be < 2000ms | ○ |
PF102  | Get issuer status API response            | < 500ms   | Send GET /issuer/:id → Response time < 500ms (use browser DevTools Network tab) | ○ |
PF103  | Decision rules evaluation time            | < 5s      | From event consumed to all 9 rules evaluated → Total time < 5000ms (check logs) | ○ |
PF104  | UI page load time                         | < 2s      | Navigate to issuer detail page → DOMContentLoaded event < 2000ms (DevTools Performance) | ○ |


================================================================================
SECTION 13: REGRESSION TESTS (P0) - Run Before Release
================================================================================

Critical Path - Must pass before any production release
---------------------------------------------------------

ID    | Test Description                           | Criteria                                          | Pass/Fail | Notes
------|--------------------------------------------|----------------------------------------------------|-----------|-------
RG001 | Create issuer with valid data              | End-to-end: POST issuer → 201 response → Record in DB → Kafka event published | ○ |
RG002 | All 9 decision rules evaluate correctly    | Create issuer → All 9 rules execute → Each returns GRANTED/DENIED/REFERRED → Results saved in DB | ○ |
RG003 | Status transitions work (AWAITING→ACTIVE)  | New issuer (AWAITING_ACTIVATION) → All rules GRANTED → Status auto-updates to ACTIVE | ○ |
RG004 | Invoice blocked for non-ACTIVE issuer      | Issuer with status ≠ ACTIVE → POST invoice → HTTP 403 with clear error message | ○ |
RG005 | Database integrity maintained              | Run integrity checks: no orphaned records, all FKs valid, required fields not null | ○ |
RG006 | Third-party integrations working           | D&B call succeeds → Creditsafe call succeeds → Partner Platform boarding succeeds | ○ |
RG007 | UI workflows complete successfully         | Complete onboarding workflow in UI → Approve KYC → Issuer becomes ACTIVE | ○ |
RG008 | Authentication & authorization working     | Valid JWT → Access granted, Invalid/expired JWT → 401, Missing permissions → 403 | ○ |


================================================================================
TEST EXECUTION SUMMARY - ESSENTIAL TESTS (P0 & P1)
===================================================

Test Category                    | Total | Passed | Failed | Not Tested | % Complete
---------------------------------|-------|--------|--------|------------|------------
Critical API Tests               | 15    | 0      | 0      | 15         | 0%
Decision Rules (9 rules)         | 28    | 0      | 0      | 28         | 0%
Database Validation              | 10    | 0      | 0      | 10         | 0%
Third-Party Integration          | 12    | 0      | 0      | 12         | 0%
Status Management                | 8     | 0      | 0      | 8          | 0%
Kafka Events                     | 5     | 0      | 0      | 5          | 0%
Security                         | 5     | 0      | 0      | 5          | 0%
UI - Core Functionality          | 26    | 0      | 0      | 26         | 0%
UI - Error Handling              | 20    | 0      | 0      | 20         | 0%
UI - User Workflows              | 31    | 0      | 0      | 31         | 0%
UI - Accessibility & Browser     | 7     | 0      | 0      | 7          | 0%
Performance                      | 4     | 0      | 0      | 4          | 0%
Regression                       | 8     | 0      | 0      | 8          | 0%
---------------------------------|-------|--------|--------|------------|------------
TOTAL ESSENTIAL TESTS (P0/P1)    | 179   | 0      | 0      | 179        | 0%

Note: Additional 114 optional tests (P2) are listed at the end of this document


OVERALL TEST RESULTS
====================

Total Tests Executed: _____ / 179
Total Passed: _____ (___%)
Total Failed: _____ (___%)
Total Blocked/Skipped: _____

Priority Breakdown:
- P0 (Critical): _____ / 144
- P1 (Major): _____ / 35


DEFECTS SUMMARY
===============

Bug ID    | Severity | Test ID | Description                           | Status
----------|----------|---------|---------------------------------------|----------
BUG-001   |          |         |                                       |
BUG-002   |          |         |                                       |
BUG-003   |          |         |                                       |
BUG-004   |          |         |                                       |


Critical Bugs: _____
High Priority Bugs: _____
Medium Priority Bugs: _____
Low Priority Bugs: _____


SIGN-OFF CRITERIA
=================

Criterion                                              | Required  | Status
-------------------------------------------------------|-----------|--------
All P0 (Critical) tests passed                         | 100%      | ○
All P1 (Major) tests passed                            | >= 95%    | ○
All 9 decision rules verified                          | 100%      | ○
Status transitions working correctly                   | 100%      | ○
Database integrity verified                            | 100%      | ○
Third-party integrations functional                    | 100%      | ○
UI critical workflows completed                        | 100%      | ○
No Critical or High severity bugs open                 | 0 bugs    | ○
Test execution report completed                        | Yes       | ○


SIGN-OFF APPROVALS
==================

QA Lead
Name: _______________________
Signature: __________________
Date: _______________________
Status: [Approved / Rejected / Conditional]


Product Owner
Name: _______________________
Signature: __________________
Date: _______________________
Status: [Approved / Rejected / Conditional]


Development Lead
Name: _______________________
Signature: __________________
Date: _______________________
Status: [Approved / Rejected / Conditional]


CONDITIONS FOR APPROVAL (if any)
=================================

[List any conditions or issues that must be resolved before final approval]

1. _______________________________________________
2. _______________________________________________
3. _______________________________________________


================================================================================
TESTING NOTES & OBSERVATIONS
================================================================================

[Add any observations, risks, or recommendations discovered during testing]







################################################################################
################################################################################
##                                                                            ##
##              OPTIONAL / NICE-TO-HAVE TEST CASES (P2)                       ##
##                    (Execute if time permits)                               ##
##                                                                            ##
################################################################################
################################################################################


================================================================================
SECTION A: EXTENDED API EDGE CASES
================================================================================

A.1 API Request Edge Cases
----------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX001  | Company name with special chars (å,ä,ö)   | Create with name = "Åkersberga Företag AB" → HTTP 201, UTF-8 encoded correctly in DB, displays correctly | ○ |
EX002  | Company name at max length (255 chars)    | Name with exactly 255 chars → Accepted, 256 chars → HTTP 422 error | ○ |
EX003  | Org number with hyphens/spaces            | Org# = "556677-8899" → System normalizes to "5566778899", validates against country format | ○ |
EX004  | Very long organisation number             | Org# with 20 digits → Validation checks max length per country (SE:10, NO:9), rejects if too long | ○ |
EX005  | Email with unusual but valid format       | Email = "user+tag@subdomain.example.co.uk" → Passes RFC 5322 validation, accepted | ○ |
EX006  | IBAN with spaces (formatted)              | IBAN = "SE12 3456 7890 1234 5678 9012" → Spaces removed, mod-97 validated, stored without spaces | ○ |
EX007  | Create while Kafka is temporarily down    | Kafka unavailable → API still returns 201, event stored in outbox table, published when Kafka recovers | ○ |
EX008  | Create with network latency (slow)        | Client with 500ms latency → Request succeeds, no timeout (timeout > 30s), idempotency key prevents duplicates | ○ |
EX009  | Concurrent creation of same issuer        | 2 POST requests with same org# + country within 100ms → One gets 201, other gets 409 conflict | ○ |
EX010  | Request with extra/unknown fields         | Body includes: {valid fields + unknownField: "value"} → Unknown field ignored, no error, valid fields processed | ○ |


A.2 Decision Rules - Extended Scenarios
-----------------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX101  | Failure score: Boundary value (24)        | D&B failureScore = 24 (integer, just below 25) → Failure Score decision = GRANTED | ○ |
EX102  | Failure score: Boundary value (25)        | D&B failureScore = 25 (exactly at threshold) → Failure Score decision = DENIED | ○ |
EX103  | Failure score: Boundary value (24.9)      | D&B failureScore = 24.9 (decimal) → Rounded or compared as < 25 → decision = GRANTED | ○ |
EX104  | Multiple high-risk flags simultaneously   | Issuer triggers: PEP (BO1) + high-risk country (BO2 from IR) + high-risk industry (64191) → 3 separate flags, all requiresEDD = true | ○ |
EX105  | All beneficial owners are PEP             | 4 BOs, all pepStatus = true → 4 HIGH_RISK_PEP flags created, EDD must cover all | ○ |
EX106  | Beneficial owner from high-risk country   | BO with citizenship = "SY" (Syria, high-risk) → HIGH_RISK_COUNTRY flag, requiresEDD = true | ○ |
EX107  | KYC with 0 beneficial owners              | Partner Platform returns: beneficialOwners = [] → KYC rule decision = REFERRED | ○ |
EX108  | KYC with maximum BOs (4)                  | Exactly 4 BOs with ownership: 30%, 25%, 25%, 20% → All saved to DB, KYC can be approved | ○ |
EX109  | Address in high-risk country              | D&B registeredOffice.country = "AF" (Afghanistan) → Address decision = DENIED | ○ |
EX110  | IBAN from unsupported country             | IBAN = "RU1234567890..." (Russia, not in approved list) → IBAN decision = DENIED | ○ |


A.3 Third-Party Service Failures
----------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX201  | D&B API returns 500 error                 | Mock D&B to return 500 → System retries 3x (1s, 5s, 15s delays), then sets rules = REFERRED, logs error | ○ |
EX202  | D&B API timeout (> 30s)                   | Mock D&B with 35s delay → Request times out at 30s, rules = REFERRED, timeout logged | ○ |
EX203  | D&B returns partial data                  | D&B returns: industryCode = "47111" but failureScore = null → Industry rule evaluates, Score rule = REFERRED | ○ |
EX204  | Creditsafe API rate limit exceeded        | Creditsafe returns 429 with Retry-After: 60 → System waits 60s, retries, eventually REFERRED if fails | ○ |
EX205  | Creditsafe API returns malformed response | Creditsafe returns: invalid JSON → JSON parse error caught, Sanction rule = REFERRED, error logged | ○ |
EX206  | Partner Platform API returns 503 unavailable          | Partner Platform 503 during boarding → Boarding fails gracefully, issuer stays AWAITING_ACTIVATION, retry scheduled | ○ |
EX207  | Partner Platform API authentication failure           | Partner Platform returns 401 → System calls Identity Service for new token, retries request, succeeds or logs persistent failure | ○ |
EX208  | All third-party services down             | D&B, Creditsafe, Partner Platform all return errors → All rules = REFERRED, alert sent to ops, issuer in NEED_ACTION | ○ |
EX209  | Retry logic for failed API calls          | Any service fails → System retries: attempt 1 (immediate), 2 (after 1s), 3 (after 5s), 4 (after 15s), then gives up | ○ |


A.4 Database Edge Cases
------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX301  | Database connection pool exhausted        | Simulate 50 concurrent requests with pool size 20 → Requests wait in queue, no errors, all eventually complete | ○ |
EX302  | Query timeout handling                    | Simulate slow query (> 30s) → Query timeout triggered, transaction rolled back, HTTP 503 returned | ○ |
EX303  | Transaction rollback on error             | Start transaction → Insert issuer → Insert bank (fails) → Rollback → Issuer not in DB | ○ |
EX304  | Deadlock detection and retry              | Two transactions update same issuer → Deadlock detected → One transaction automatically retried, both succeed | ○ |
EX305  | Very large result sets (1000+ records)    | Query 10,000 issuers → Pagination returns 100 per page, no memory overflow, performance < 2s per page | ○ |


================================================================================
SECTION B: EXTENDED UI TESTS
================================================================================

B.1 UI List View - Extended
-----------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX401  | Sort by multiple columns                  | Click org# column header → Sort A-Z, click again → Sort Z-A, works for all sortable columns | ○ |
EX402  | Filter by multiple criteria together      | Select status = ACTIVE + country = SE → List shows only ACTIVE Swedish issuers | ○ |
EX403  | Export issuer list as CSV                 | Click "Export CSV" button → File downloads with all filtered issuers, correct columns, UTF-8 encoded | ○ |
EX404  | Bulk actions on multiple issuers          | Select checkboxes for 5 issuers → Click "Bulk Deactivate" → Confirmation modal → All 5 deactivated | ○ |
EX405  | Save custom filter presets                | Set filters (status + country) → Click "Save Filter" → Name it "Active Swedish" → Recall later from dropdown | ○ |
EX406  | Zero issuers - empty state                | When no issuers match filters → Show: "No issuers found" with illustration, "Clear Filters" button | ○ |
EX407  | Thousands of issuers load correctly       | List with 5000+ issuers → Pagination works, page loads < 2s, can navigate to any page | ○ |


B.2 UI Detail View - Extended Tabs
------------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX501  | Banking tab - multiple bank accounts      | Issuer with 2+ bank accounts → Banking tab shows all in table, can set one as primary | ○ |
EX502  | History tab - filter by event type        | Dropdown filter: All / Status Changes / Rule Evaluations / KYC → List filters accordingly | ○ |
EX503  | History tab - export as PDF/CSV           | Click "Export" → Select PDF → PDF downloads with issuer name, all history events, timestamps | ○ |
EX504  | Decision Rules - re-evaluate button works | Click "Re-evaluate All Rules" → Loading spinner → All 9 rules re-run, results update | ○ |
EX505  | Decision Rules - manual override (if any) | For REFERRED rule: Click "Override" → Requires admin role → Select GRANTED/DENIED + reason → Saves | ○ |
EX506  | EDD tab - download uploaded docs          | Click document name link → File downloads with original filename | ○ |
EX507  | EDD tab - delete uploaded document        | Click delete icon → Confirmation: "Delete document?" → Confirm → Document removed | ○ |


B.3 UI Form Interactions - Extended
-------------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX601  | Auto-save draft functionality             | Start filling form → Data auto-saves to local storage every 30s → Close browser → Reopen → Data restored | ○ |
EX602  | Discard changes confirmation              | Edit form → Click "Cancel" → Modal: "Discard changes?" → Confirm → Form resets | ○ |
EX603  | Form pre-fill from previous data          | Create 2nd issuer → System suggests: previous country, bank name from history | ○ |
EX604  | Copy data from another issuer             | Click "Copy from..." → Select issuer → Name, country, bank pre-filled (org# blank) | ○ |
EX605  | Field-level help tooltips                 | Hover over (i) icon next to field → Tooltip appears with: explanation, format example | ○ |


B.4 UI Error Handling - Extended
----------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX701  | File upload - file too large              | Upload 25MB file (max 10MB) → Error: "File too large. Maximum size is 10MB." | ○ |
EX702  | File upload - invalid file type           | Upload .exe file (only PDF/DOCX/images allowed) → Error: "Invalid file type. Allowed: PDF, DOCX, PNG, JPG" | ○ |
EX703  | File upload - corrupt file                | Upload corrupted PDF → Upload fails, error: "File is corrupted or invalid" | ○ |
EX704  | Session timeout during form fill          | Fill form for 20 minutes (session expires at 15 min) → Submit → Redirect to login with message: "Session expired" | ○ |
EX705  | Stale data warning (concurrent edit)      | User A views issuer → User B updates status → User A tries to save → Warning: "Data has changed. Refresh to see latest." | ○ |
EX706  | Optimistic locking conflict resolution    | Both users edit → First save succeeds → Second save shows conflict → Option to: "Overwrite" or "Discard" | ○ |


B.5 UI Special Characters & Encoding
--------------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX801  | Swedish characters (å, ä, ö) display      | Company "Åkersberga AB" → Characters render correctly everywhere: list, detail, exports | ○ |
EX802  | Norwegian characters (æ, ø) display       | Company "Kjøbenhavn AS" → Characters render correctly | ○ |
EX803  | Unicode characters handled                | Company with emoji/Chinese chars → Stored correctly, no encoding errors | ○ |
EX804  | Emoji in notes don't break UI             | Enter notes: "Approved ✓ 👍" → Saves, displays correctly, doesn't break layout | ○ |
EX805  | Very long text truncated with tooltip     | Company name 100 chars → Truncated to 50 chars with "...", hover shows full name in tooltip | ○ |
EX806  | HTML tags in input don't execute          | Enter name: "<script>alert('xss')</script>" → Sanitized, stored as text, doesn't execute | ○ |
EX807  | SQL injection attempt prevented           | Enter org#: "'; DROP TABLE issuers; --" → Treated as string, parameterized query prevents injection | ○ |


B.6 UI Responsive Design
-------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX901  | Desktop (1920x1080) optimal layout        | All content visible, no unnecessary scrolling, sidebars and main content well-spaced | ○ |
EX902  | Laptop (1366x768) usable                  | All features accessible, may have some scrolling, buttons not cut off | ○ |
EX903  | Tablet (768x1024) acceptable              | Responsive layout adapts, touch-friendly targets (min 44px), tables scroll horizontally | ○ |
EX904  | Tables scroll on smaller screens          | Wide tables on small screen → Horizontal scroll appears, sticky first column (if designed) | ○ |


B.7 UI Advanced Accessibility
-------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1001 | Screen reader announces all changes       | Use NVDA/JAWS → Status changes, errors, success messages announced audibly | ○ |
EX1002 | ARIA labels on all interactive elements   | Buttons without visible text have aria-label, form fields have aria-describedby for help text | ○ |
EX1003 | Skip to main content link                 | Press Tab on page load → First element is "Skip to main content" link → Press Enter → Focus jumps to main | ○ |
EX1004 | All functionality via keyboard only       | Complete entire onboarding workflow using only keyboard (Tab, Enter, Arrow keys, Esc) → No mouse needed | ○ |


B.8 UI Visual Polish
---------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1101 | Loading skeletons for async content       | While loading → Gray animated skeleton boxes appear matching content layout (no empty white space) | ○ |
EX1102 | Smooth transitions between states         | Status changes, tab switches → CSS transitions (200-300ms), no jarring jumps | ○ |
EX1103 | Toast notifications stack properly        | Trigger 3 toasts rapidly → All 3 stack vertically (top-right), each auto-dismisses independently | ○ |
EX1104 | Consistent spacing across all pages       | All pages use same spacing units: 4px, 8px, 16px, 24px (design system), no random gaps | ○ |
EX1105 | Icons consistent and meaningful           | Same icon for same action everywhere (e.g., ✓ for success, ✗ for error), intuitive meanings | ○ |


================================================================================
SECTION C: PERFORMANCE & LOAD TESTING (EXTENDED)
================================================================================

C.1 Load Testing
-----------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1201 | 10 concurrent issuer creations            | Send 10 POST requests simultaneously → All complete in < 5s, all return 201, no errors | ○ |
EX1202 | 50 concurrent issuer creations            | Send 50 POST requests → All complete in < 15s, success rate ≥ 95%, no system crash | ○ |
EX1203 | 100 concurrent API requests               | 100 mixed GET/POST requests → Response times < 3s avg, error rate < 5%, system responsive | ○ |
EX1204 | System stable under sustained load        | 1000 requests over 10 minutes → CPU < 80%, Memory < 85%, no crashes, response times stable | ○ |
EX1205 | No memory leaks during load test          | Monitor heap memory → After load test, memory returns to baseline (±10%), no continuous growth | ○ |


C.2 Database Performance
-------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1301 | Query optimization verified               | Run EXPLAIN on all queries → All use indexes, no full table scans on large tables | ○ |
EX1302 | Indexes used correctly                    | Check query plans → Indexes on: (org_number, country, tenant_id), (status), (created_at) are used | ○ |
EX1303 | No N+1 query problems                     | Load 100 issuers with BOs → Uses JOIN or batch query, not 1+100 queries (check slow query log) | ○ |
EX1304 | Connection pool sized appropriately       | Pool size matches expected load: min 5, max 20 connections, no pool exhaustion under normal load | ○ |


================================================================================
SECTION D: ADDITIONAL NEGATIVE SCENARIOS
================================================================================

D.1 Network & Resilience
-------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1401 | Partial network failure (one service)     | D&B unavailable but others work → D&B rules = REFERRED, other rules evaluate normally, process continues | ○ |
EX1402 | Complete network disconnect               | Disconnect network during request → UI shows error: "Network error. Check connection.", retry available | ○ |
EX1403 | Network reconnect - resume operations     | Network back online → Pending events process, queued requests sent, system recovers automatically | ○ |
EX1404 | Slow network (3G simulation)              | Throttle to 3G speed → Requests slower but complete, no timeouts, loading indicators shown | ○ |
EX1405 | Request timeout handling                  | Service takes > 30s → Client timeout, error shown, request can be retried | ○ |


D.2 Data Consistency
---------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1501 | Race condition handling                   | Two processes update same issuer status simultaneously → One succeeds, other retries or fails gracefully | ○ |
EX1502 | Eventual consistency verified             | Kafka event processed → Database updated within 5s → All replicas consistent within 30s | ○ |
EX1503 | Orphaned records cleaned up               | Create issuer → Boarding fails → Cleanup job runs → Orphaned records removed or marked | ○ |
EX1504 | Data migration scenarios                  | Schema change deployed → Old data migrates correctly, no data loss, backward compatible | ○ |


D.3 Browser Edge Cases
-----------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1601 | Browser back button doesn't break state   | Navigate: List → Detail → KYC → Press Back → Returns to Detail (Main tab), data intact | ○ |
EX1602 | Browser refresh maintains data            | On detail page → Press F5 → Page reloads, same issuer shown, tab position may reset (acceptable) | ○ |
EX1603 | Browser cache doesn't show stale data     | User A updates issuer → User B refreshes list → B sees updated data (cache bust working) | ○ |
EX1604 | Multiple tabs with same issuer open       | Open issuer in 2 tabs → Update in Tab 1 → Tab 2 shows stale data warning when interacting | ○ |
EX1605 | Browser local storage limits              | Fill 5MB+ data in local storage → System handles gracefully, clears old drafts if needed | ○ |


================================================================================
SECTION E: EXTENDED SECURITY TESTS
================================================================================

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1701 | XSS attack prevention                     | Enter: name = "<script>alert('xss')</script>" → Stored safely, HTML escaped on display, script doesn't execute | ○ |
EX1702 | CSRF token validation                     | Submit form without CSRF token → HTTP 403, message: "Invalid CSRF token" | ○ |
EX1703 | Rate limiting on API endpoints            | Send 100 requests in 1 minute from same IP → After 50 requests: HTTP 429 "Too many requests", Retry-After header | ○ |
EX1704 | Sensitive data not in logs                | Check logs → IBAN masked (shows SE12****9012), no full personal data (only IDs), no JWT tokens | ○ |
EX1705 | Encryption at rest verified               | Database sensitive fields encrypted (IBAN, DOB, email), query returns decrypted, backup files encrypted | ○ |
EX1706 | Encryption in transit (HTTPS)             | All API calls use HTTPS, HTTP redirects to HTTPS, TLS 1.2+, certificate valid | ○ |


================================================================================
SECTION F: EXTENDED WORKFLOW VARIATIONS
================================================================================

F.1 Alternative User Journeys
-------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1801 | Approve issuer with one REFERRED rule     | 8 rules GRANTED, 1 REFERRED → Operator can override REFERRED → Approve → Issuer becomes ACTIVE | ○ |
EX1802 | Deny KYC with reason provided             | Click "Deny KYC" → Enter reason: "Incomplete documents" → Confirm → KYC decision = DENIED, issuer INACTIVE | ○ |
EX1803 | Re-evaluate decision rules after update   | Issuer data updated in Partner Platform → Click "Re-evaluate" → All 9 rules run again with fresh data | ○ |
EX1804 | Reactivate INACTIVE issuer                | INACTIVE issuer → Admin fixes issue → Click "Reactivate" → Re-run rules → If pass, status → ACTIVE | ○ |
EX1805 | Deactivate ACTIVE issuer                  | ACTIVE issuer → Admin clicks "Deactivate" → Enter reason → Confirm → Status → INACTIVE, invoices blocked | ○ |


F.2 Bulk Operations (if supported)
------------------------------------

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX1901 | Bulk import issuers from CSV              | Upload CSV with 50 issuers → Validate format → Import → All created, errors logged, summary shown | ○ |
EX1902 | Bulk status change                        | Select 10 issuers → "Bulk Deactivate" → Confirmation → All 10 change to INACTIVE simultaneously | ○ |
EX1903 | Bulk export with filters applied          | Filter: status = ACTIVE, country = SE → Export → CSV contains only matching issuers (50 records) | ○ |


================================================================================
SECTION G: MONITORING & OBSERVABILITY
================================================================================

ID     | Test Description                          | Criteria                                          | Pass/Fail | Notes
-------|-------------------------------------------|----------------------------------------------------|-----------|-------
EX2001 | Application logs contain issuer events    | Check logs → Contains: issuer.created, issuer.boarding.complete, decision.evaluated with issuerId, timestamp | ○ |
EX2002 | Metrics exposed (Prometheus/Grafana)      | Metrics endpoint /metrics → Exposes: issuer_created_total, decision_rules_evaluated, api_response_time | ○ |
EX2003 | Alerts configured for failures            | D&B fails → Alert fired to Slack/Email within 5 min, 3+ failures → PagerDuty alert | ○ |
EX2004 | Dashboard shows issuer stats              | Grafana dashboard shows: total issuers, by status, decision rule success rate, API latency graphs | ○ |
EX2005 | Tracing spans for distributed calls       | Trace ID propagates: API → Decision Engine → D&B/Creditsafe → Partner Platform, viewable in Jaeger/Datadog | ○ |


================================================================================
ADDITIONAL TESTS SUMMARY
================================================================================

Category                         | Total | Passed | Failed | Not Tested
---------------------------------|-------|--------|--------|-----------
Extended API Edge Cases          | 20    | 0      | 0      | 20
Extended Decision Rules          | 10    | 0      | 0      | 10
Extended Integration Tests       | 9     | 0      | 0      | 9
Extended Database Tests          | 5     | 0      | 0      | 5
Extended UI Tests                | 32    | 0      | 0      | 32
Performance & Load Testing       | 9     | 0      | 0      | 9
Additional Negative Scenarios    | 14    | 0      | 0      | 14
Extended Security Tests          | 6     | 0      | 0      | 6
Extended Workflow Variations     | 8     | 0      | 0      | 8
Monitoring & Observability       | 5     | 0      | 0      | 5
---------------------------------|-------|--------|--------|-----------
TOTAL OPTIONAL TESTS             | 118   | 0      | 0      | 118


GRAND TOTAL (ESSENTIAL + OPTIONAL)
===================================

Essential Tests (P0/P1): 179
Optional Tests (P2): 118
GRAND TOTAL: 297 tests


================================================================================
DOCUMENT INFORMATION
================================================================================

Document Title: Issuer Onboarding - QA & Testing Checklist (Essential with Criteria)
Version: 2.1
Last Updated: 2026-02-20
Test Environment: [Staging / Production]
Test Cycle: [Cycle Number/Name]

Distribution:
- QA Team
- Development Team
- Product Owners
- Operations Team

Related Documents:
- ISSUER_ONBOARDING_CHECKLIST.md (Detailed operations checklist)
- ISSUER_ONBOARDING_HAPPY_FLOW.md (Happy path documentation)
- ISSUER_DECISION_RULES_TESTS.md (Comprehensive decision rule tests)
- ISSUER_DATABASE_CHECKS.md (Database validation reference)

For Excel Format:
- Open ISSUER_ONBOARDING_QA_CHECKLIST.csv in Excel for auto-separated columns


================================================================================
END OF DOCUMENT
================================================================================
