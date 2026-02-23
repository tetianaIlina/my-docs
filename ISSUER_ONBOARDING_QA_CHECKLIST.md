# 🔍 Issuer Onboarding - QA & Testing Checklist

![Test Type](https://img.shields.io/badge/Test_Type-QA_Checklist-blue)
![Version](https://img.shields.io/badge/Version-1.1-green)
![Status](https://img.shields.io/badge/Status-In_Progress-yellow)
![Environment](https://img.shields.io/badge/Environment-Staging-orange)

> [!IMPORTANT]
> **Document Version:** 1.1 (with detailed test criteria)  
> **Date:** 2026-02-20  
> **QA Engineer:** ___________________________  
> **Status:** In Progress

---

## 📋 Table of Contents

- [Test Status Legend](#-test-status-legend)
- [Overview](#-overview)
- [Test Environment Setup](#-test-environment-setup)
- [Section 1: Critical API Tests (P0)](#section-1-critical-api-tests-p0)
- [Section 2: Decision Rules Testing (P0)](#section-2-decision-rules-testing-9-mandatory-rules---p0)
- [Section 3: Database Validation (P0)](#section-3-database-validation-p0)
- [Section 4: Third-Party Integrations (P0)](#section-4-third-party-integrations-p0)
- [Section 5: UI Testing (P1)](#section-5-ui-testing-p1)
- [Section 6: Error Handling (P0)](#section-6-error-handling-p0)
- [Section 7: Performance (P1)](#section-7-performance-p1)
- [Section 8: Security (P1)](#section-8-security-p1)
- [Section 9: Deactivation Workflow (P1)](#section-9-deactivation-workflow-p1)
- [Section 10: Observability & Monitoring (P1)](#section-10-observability--monitoring-p1)
- [Test Summary](#-test-summary)
- [Sign-Off](#-sign-off)

---

## 📊 Test Status Legend

| Symbol | Meaning |
|:------:|---------|
| ☐ | Not Tested |
| ☑ | Test Passed |
| ☒ | Test Failed |
| ⊗ | Blocked/Skipped |
| N/A | Not Applicable |

---

## 📖 Overview

This is a streamlined checklist covering **CRITICAL (P0)** and **MAJOR (P1)** test cases for issuer onboarding.

**Focus areas:**
- Core API
- Decision rules
- Database integrity
- UI workflows
- Error handling

---

## 🔧 Test Environment Setup

| Item | ✅ | Notes |
|------|:---:|-------|
| Test environment configured (staging) | ☐ | |
| API authentication tokens obtained | ☐ | |
| Database access configured | ☐ | |
| Partner Platform Integration Service running | ☐ | |
| Decision Engine accessible | ☐ | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 1: Critical API Tests (P0)

### 1.1 Create Issuer - Happy Path

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| TC001 | ☐ | Create issuer with valid data | POST /api/v1/issuer/event with: organisationNumber (valid format), country (SE/NO/DK/FI), name, channelId → HTTP 201 | |
| TC002 | ☐ | Verify issuer record in database | Query: SELECT * FROM issuers WHERE organisation_number = '...' → Record exists with correct data | |
| TC003 | ☐ | Verify status is AWAITING_ACTIVATION | issuer.status = 'AWAITING_ACTIVATION' in database | |
| TC004 | ☐ | Kafka event published successfully | Event on topic 'issuer.created' with correct payload (issuerId, org#, country, timestamp) | |

### 1.2 Create Issuer - Critical Negative Tests

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| TC101 | ☐ | Create without authentication (401) | No Authorization header → HTTP 401, message: "Unauthorized" | |
| TC102 | ☐ | Create with missing org number (422) | Request body missing organisationNumber → HTTP 422, validation error | |
| TC103 | ☐ | Create with missing country code (422) | Request body missing country → HTTP 422, validation error | |
| TC104 | ☐ | Create duplicate issuer (409) | Same org# + country exists → HTTP 409, message: "Issuer already exists" | |

### 1.3 Get Issuer Status

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| TC201 | ☐ | GET issuer status returns correct data | GET /api/v1/internal/issuer/:id → Returns aggregatedStatus, systemStatus, financialPartnerStatus | |
| TC202 | ☐ | Verify aggregated status displayed | Response contains status enum: ACTIVE / AWAITING_ACTIVATION / INACTIVE / NEED_ACTION | |
| TC203 | ☐ | 404 for non-existent issuer | GET with non-existent UUID → HTTP 404, message: "Issuer not found" | |

### 1.4 Invoice Creation - Status Check

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| TC301 | ☐ | Invoice ALLOWED for ACTIVE issuer | Issuer with aggregatedStatus = 'ACTIVE' → POST /api/v1/invoice/event → HTTP 201 | |
| TC302 | ☐ | Invoice BLOCKED for AWAITING_ACTIVATION | Issuer status = 'AWAITING_ACTIVATION' → POST invoice → HTTP 403, error: "Issuer not active" | |
| TC303 | ☐ | Invoice BLOCKED for INACTIVE issuer | Issuer status = 'INACTIVE' → POST invoice → HTTP 403, error: "Issuer not active" | |
| TC304 | ☐ | Error message is clear | Error response includes: status reason, issuerId, helpful message | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 2: Decision Rules Testing (9 Mandatory Rules) - P0

### 2.1 Rule 1: Industry Code (D&B)

**Data Source:** Dun & Bradstreet  
**Decision:** Industry code must NOT be in the denied list

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR101 | ☐ | Approved industry code → GRANTED | D&B industry code in approved list: 47111, 47190, 46900 (retail/wholesale) → decision = GRANTED | |
| DR102 | ☐ | Denied industry code → DENIED | D&B industry code in denied list: 64191, 64929, 66300 (financial/gambling/betting) → decision = DENIED | |
| DR103 | ☐ | Missing industry code → REFERRED | D&B returns null/empty industry code → decision = REFERRED, requires manual review | |

### 2.2 Rule 2: Failure Score (D&B)

**Data Source:** Dun & Bradstreet  
**Decision:** Failure score must be < 25 (lower is better)

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR201 | ☐ | Score < 25 → GRANTED | D&B failure score: 0-24 (e.g., score = 15) → decision = GRANTED | |
| DR202 | ☐ | Score >= 25 → DENIED | D&B failure score: 25 or higher (e.g., score = 30, 50, 75) → decision = DENIED | |
| DR203 | ☐ | Missing score → REFERRED | D&B returns null failure score → decision = REFERRED | |

### 2.3 Rule 3: Address Validation (D&B)

**Data Source:** Dun & Bradstreet + AML Manager high-risk countries list  
**Decision:** Registered office address must NOT be in high-risk country

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR301 | ☐ | Valid address (safe country) → GRANTED | D&B address country in approved list: SE, NO, DK, FI, DE, UK, NL, FR → decision = GRANTED | |
| DR302 | ☐ | High-risk country address → DENIED | D&B address country in high-risk list: IR (Iran), KP (North Korea), SY (Syria), AF (Afghanistan) → decision = DENIED | |
| DR303 | ☐ | Missing address → REFERRED | D&B returns null/empty address or country → decision = REFERRED | |

### 2.4 Rule 4: Sanction List (Creditsafe)

**Data Source:** Creditsafe PEP & Sanction Lists  
**Decision:** Company or beneficial owners NOT on sanction list

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR401 | ☐ | Not on sanction list → GRANTED | Creditsafe check for company + all BOs returns: sanctionMatch = false → decision = GRANTED | |
| DR402 | ☐ | On sanction list → DENIED | Creditsafe returns: sanctionMatch = true, matchedEntity name provided → decision = DENIED | |
| DR403 | ☐ | Creditsafe API unavailable → REFERRED | Creditsafe API timeout (> 30s) or 500/503 error → decision = REFERRED, retry later | |

### 2.5 Rule 5: KYC Approval (Partner Platform API + Manual)

**Data Source:** Partner Platform API + Manual Operator Review  
**Decision:** KYC information complete and manually approved by operations team

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR501 | ☐ | KYC data complete → Ready for approval | Partner Platform returns: 1-4 beneficial owners with: name, DOB, citizenship, address, ownership% (≥25%) → status = PENDING_APPROVAL | |
| DR502 | ☐ | Manual approve → GRANTED | Operator reviews KYC tab, clicks "Approve KYC" → decision = GRANTED, kycApprovalDate set | |
| DR503 | ☐ | Manual deny → DENIED | Operator clicks "Deny KYC", enters reason (required field) → decision = DENIED | |
| DR504 | ☐ | Missing beneficial owners → REFERRED | Partner Platform returns 0 BOs OR incomplete BO data (missing name/DOB/etc) → decision = REFERRED | |

### 2.6 Rule 6: Contact Information (Partner Platform API)

**Data Source:** Partner Platform API  
**Decision:** Valid phone number AND email address required

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR601 | ☐ | Contact info complete → GRANTED | Partner Platform returns: phoneNumber (format: +46XXXXXXXXX or +47XXXXXXXX) AND email (valid format) → decision = GRANTED | |
| DR602 | ☐ | Missing phone/email → DENIED | Partner Platform returns: null phone OR null email OR invalid format → decision = DENIED | |

### 2.7 Rule 7: IBAN Validation (Partner Platform API)

**Data Source:** Partner Platform API  
**Decision:** Valid IBAN format from supported country

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR701 | ☐ | Valid IBAN → GRANTED | IBAN format valid: starts with SE/NO/DK/FI, correct length (SE:24, NO:15, DK:18, FI:18 chars), passes mod-97 checksum → decision = GRANTED | |
| DR702 | ☐ | Invalid IBAN → DENIED | IBAN: wrong length, invalid checksum, unsupported country (e.g., RU, CN) → decision = DENIED | |
| DR703 | ☐ | Missing IBAN → REFERRED | Partner Platform returns null/empty IBAN → decision = REFERRED | |

### 2.8 Rule 8: Agreement Validation (Partner Platform API)

**Data Source:** Partner Platform API  
**Decision:** Signed financial agreement must exist

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR801 | ☐ | Signed agreement → GRANTED | Partner Platform returns: agreementId (not null), signedDate (valid timestamp), signatoryName → decision = GRANTED | |
| DR802 | ☐ | Missing agreement → DENIED | Partner Platform returns: null agreementId OR signedDate = null OR agreement not signed → decision = DENIED | |

### 2.9 Rule 9: Blacklist Check (Internal)

**Data Source:** Internal Blacklist Service  
**Decision:** Organisation number NOT on internal denied list

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DR901 | ☐ | Not on blacklist → GRANTED | Blacklist Service query (org# + country) returns: blacklisted = false → decision = GRANTED | |
| DR902 | ☐ | On blacklist → DENIED | Blacklist Service returns: blacklisted = true, reason = "fraud" or "default" → decision = DENIED | |

### 2.10 High-Risk Flags

**Data Source:** Creditsafe (PEP), AML Manager (high-risk lists)

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| HR101 | ☐ | PEP detected → HIGH RISK flagged | Creditsafe returns: any BO with pepStatus = true → Flag type: HIGH_RISK_PEP, requiresEDD = true | |
| HR102 | ☐ | High-risk country → HIGH RISK flagged | BO citizenship/tax residence in AML Manager list: IR, KP, SY, AF, YE, MM, VE → Flag type: HIGH_RISK_COUNTRY, requiresEDD = true | |
| HR103 | ☐ | High-risk industry → HIGH RISK flagged | D&B industry code in AML Manager list: 64191, 64929, 66300, 92000 (finance/gambling/adult) → Flag type: HIGH_RISK_INDUSTRY, requiresEDD = true | |
| HR104 | ☐ | HIGH RISK requires EDD | If any HIGH_RISK flag exists → issuer.requiresEDD = true, KYC cannot be approved until EDD complete | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 3: Database Validation (P0)

### 3.1 Core Tables

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DB101 | ☐ | Issuer record created in issuers table | Query: SELECT * FROM issuers WHERE id = '...' → Record exists with: organisation_number, country, name, status, tenant_id, created_at | |
| DB102 | ☐ | Bank account saved correctly | Query: SELECT * FROM bank_accounts WHERE issuer_id = '...' → Record exists with: iban, bank_name, country, currency | |
| DB103 | ☐ | Contact info saved correctly | Query: SELECT * FROM contact_information WHERE issuer_id = '...' → Record with: phone_number, email, contact_person | |
| DB104 | ☐ | Beneficial owners saved (if provided) | Query: SELECT * FROM beneficial_owners WHERE issuer_id = '...' → 1-4 records with: name, date_of_birth, ownership_percentage | |
| DB105 | ☐ | Status history logged | Query: SELECT * FROM status_history WHERE issuer_id = '...' → Record with: old_status, new_status, changed_by, changed_at | |
| DB106 | ☐ | Decision rules results saved | Query: SELECT * FROM decision_rules WHERE issuer_id = '...' → 9 records (one per rule) with: rule_name, decision, evaluated_at | |

### 3.2 Data Integrity

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DB201 | ☐ | No duplicate issuers allowed | UNIQUE constraint on (organisation_number, country, tenant_id) → Insert duplicate returns error | |
| DB202 | ☐ | Foreign key constraints enforced | Try INSERT bank_account with non-existent issuer_id → Database rejects with FK constraint error | |
| DB203 | ☐ | Required fields validated | NOT NULL constraints on: organisation_number, country, name, status, tenant_id → Cannot insert null | |
| DB204 | ☐ | Audit trail created for changes | Query: SELECT * FROM audit_logs WHERE entity_id = '...' → Log entry for: CREATE, UPDATE, STATUS_CHANGE events | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 4: Third-Party Integrations (P0)

### 4.1 Dun & Bradstreet (D&B)

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| INT101 | ☐ | D&B API returns industry code | Call D&B with org# → Response HTTP 200, body contains: industryCode (5 digits), description → Log response | |
| INT102 | ☐ | D&B API returns failure score | D&B response contains: failureScore (0-100 integer) → Log score value | |
| INT103 | ☐ | D&B API returns company address | D&B response contains: address object with street, city, postalCode, country (ISO code) | |
| INT104 | ☐ | Handle D&B API timeout gracefully | Simulate timeout (> 30s) → System logs error, sets rule decision = REFERRED, doesn't crash | |

### 4.2 Creditsafe

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| INT201 | ☐ | Creditsafe API returns sanction check | Call Creditsafe with company name + org# → Response HTTP 200, sanctionMatch: true/false | |
| INT202 | ☐ | Creditsafe returns PEP status for BOs | Call Creditsafe for each BO → Response includes: pepStatus: true/false, lastScreeningDate | |
| INT203 | ☐ | Handle Creditsafe API error (500/503) | Simulate error → System logs error, sets rule = REFERRED, retries later (exponential backoff) | |

### 4.3 Partner Platform API

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| INT301 | ☐ | Fetch creditor data from Partner Platform | GET /api/creditor/:creditorId → HTTP 200, returns: name, org#, country, address | |
| INT302 | ☐ | Fetch KYC data (beneficial owners) | GET /api/creditor/:creditorId/kyc → HTTP 200, returns: array of BOs with name, DOB, ownership% | |
| INT303 | ☐ | Fetch bank account (IBAN) | GET /api/creditor/:creditorId/bank-account → HTTP 200, returns: iban, bankName, currency | |
| INT304 | ☐ | Fetch signed agreements | GET /api/creditor/:creditorId/agreements → HTTP 200, returns: agreementId, signedDate, signatoryName | |
| INT305 | ☐ | Handle Partner Platform API unavailable | Simulate 503 → System retries with backoff, logs error, marks issuer as NEED_ACTION | |

### 4.4 AML Manager (High-Risk Lists)

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| INT401 | ☐ | Fetch high-risk countries list | GET /api/aml/high-risk-countries → HTTP 200, returns: array of ISO country codes | |
| INT402 | ☐ | Fetch high-risk industries list | GET /api/aml/high-risk-industries → HTTP 200, returns: array of industry codes | |
| INT403 | ☐ | AML service unavailable handled | Simulate timeout → System uses cached list, logs warning, continues processing | |

### 4.5 Internal Blacklist Service

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| INT501 | ☐ | Check if org# is blacklisted | POST /api/internal/blacklist/check with: org#, country → Returns: blacklisted: true/false, reason | |
| INT502 | ☐ | Service responds quickly (< 500ms) | Measure response time → Average < 500ms | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 5: UI Testing (P1)

### 5.1 Issuer Overview Screen

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| UI101 | ☐ | Display issuer list | Navigate to /issuers → Page loads, shows table with: Name, Org#, Country, Status, Created Date | |
| UI102 | ☐ | Status badge color coded | ACTIVE = green, AWAITING_ACTIVATION = yellow, INACTIVE = red, NEED_ACTION = orange | |
| UI103 | ☐ | Search by organisation number works | Enter org# in search → Table filters to show matching issuer(s) | |
| UI104 | ☐ | Filter by status works | Select status filter: ACTIVE → Table shows only active issuers | |
| UI105 | ☐ | Pagination works correctly | List with 50+ issuers → Shows 25 per page, "Next"/"Previous" buttons work | |

### 5.2 Issuer Details Screen

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| UI201 | ☐ | Display issuer details | Click issuer → Details page shows: Name, Org#, Country, Address, Contact info, Bank account | |
| UI202 | ☐ | Display aggregated status | Status card shows: Aggregated status (large), System status, Financial partner status | |
| UI203 | ☐ | Display decision rules results | Tab "Decision Rules" shows: 9 rules with status (GRANTED/DENIED/REFERRED), evaluation date | |
| UI204 | ☐ | Display KYC information | Tab "KYC" shows: List of beneficial owners, Name, DOB, Ownership%, PEP status | |
| UI205 | ☐ | Display status history timeline | Tab "History" shows: Timeline of status changes with: Old status → New status, Date, Changed by | |

### 5.3 Manual KYC Approval

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| UI301 | ☐ | KYC tab shows approval controls | Navigate to KYC tab → Shows "Approve KYC" and "Deny KYC" buttons (if status = AWAITING_ACTIVATION) | |
| UI302 | ☐ | Approve KYC updates status | Click "Approve KYC" → Confirmation modal → Click "Confirm" → Status updates to ACTIVE (if all rules GRANTED) | |
| UI303 | ☐ | Deny KYC requires reason | Click "Deny KYC" → Modal with "Reason" text field (required) → Enter reason → Status = INACTIVE | |
| UI304 | ☐ | Buttons disabled after action | After approve/deny → Buttons become disabled, status badge updates immediately | |

### 5.4 High-Risk Flags Display

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| UI401 | ☐ | Display high-risk flags prominently | Issuer with HIGH_RISK flags → Details page shows red warning banner: "HIGH RISK - EDD Required" | |
| UI402 | ☐ | List specific risk types | Banner lists risk types: "PEP Detected", "High-Risk Country", "High-Risk Industry" | |
| UI403 | ☐ | Show EDD requirement | If requiresEDD = true → Cannot approve KYC until EDD complete, message displayed | |

### 5.5 Deactivation UI

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| UI501 | ☐ | Deactivate button visible for active issuer | Issuer status = ACTIVE → "Deactivate Issuer" button visible in details page | |
| UI502 | ☐ | Deactivation requires reason | Click "Deactivate" → Modal with: "Reason" dropdown (required): Fraud, Default, Request, Other | |
| UI503 | ☐ | Deactivation updates status | Select reason → Click "Confirm" → Status changes to INACTIVE, statusReason saved | |
| UI504 | ☐ | Cannot reactivate without approval | Deactivated issuer → No "Reactivate" button, must contact admin | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 6: Error Handling (P0)

### 6.1 API Errors

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| ERR101 | ☐ | Malformed JSON returns 400 | POST with invalid JSON → HTTP 400, error: "Invalid request body" | |
| ERR102 | ☐ | Missing required field returns 422 | POST without required field → HTTP 422, error lists missing field(s) | |
| ERR103 | ☐ | Invalid UUID format returns 400 | GET /issuer/:id with invalid UUID → HTTP 400, error: "Invalid ID format" | |
| ERR104 | ☐ | Unauthorized access returns 401 | Request without valid JWT → HTTP 401, error: "Unauthorized" | |
| ERR105 | ☐ | Insufficient permissions returns 403 | User without issuer:write tries POST → HTTP 403, error: "Forbidden" | |

### 6.2 Integration Errors

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| ERR201 | ☐ | D&B timeout handled gracefully | Simulate D&B timeout → System logs error, rule = REFERRED, processing continues | |
| ERR202 | ☐ | Creditsafe 500 error handled | Simulate 500 error → System logs error, rule = REFERRED, retry scheduled | |
| ERR203 | ☐ | Partner Platform 404 handled | Creditor not found → System logs error, marks issuer as NEED_ACTION | |
| ERR204 | ☐ | Network error handled | Simulate network disconnect → System retries with exponential backoff, doesn't crash | |

### 6.3 Database Errors

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| ERR301 | ☐ | Duplicate key returns 409 | Try create issuer with existing org# + country → HTTP 409, error: "Issuer already exists" | |
| ERR302 | ☐ | Foreign key violation logged | Try invalid FK → Database error logged, transaction rolled back | |
| ERR303 | ☐ | Connection pool exhaustion handled | Simulate connection exhaustion → System waits for available connection, doesn't crash | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 7: Performance (P1)

### 7.1 Response Times

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| PERF101 | ☐ | Create issuer completes quickly | POST /issuer/event → Response time < 3s (including external API calls) | |
| PERF102 | ☐ | Get issuer status fast | GET /issuer/:id/status → Response time < 500ms | |
| PERF103 | ☐ | List issuers responds quickly | GET /issuers (page size 25) → Response time < 1s | |

### 7.2 Load Testing

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| PERF201 | ☐ | Handle 10 concurrent issuer creations | 10 simultaneous POST requests → All complete successfully within 5s | |
| PERF202 | ☐ | Handle 100 read requests | 100 GET /issuer/:id requests → 95th percentile < 1s | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 8: Security (P1)

### 8.1 Authentication & Authorization

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| SEC101 | ☐ | JWT token required for all endpoints | Request without token → HTTP 401 | |
| SEC102 | ☐ | Expired token rejected | Request with expired JWT → HTTP 401, error: "Token expired" | |
| SEC103 | ☐ | Invalid token signature rejected | Request with tampered token → HTTP 401, error: "Invalid token" | |
| SEC104 | ☐ | RBAC enforced | User without issuer:write permission tries POST → HTTP 403 | |

### 8.2 Data Security

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| SEC201 | ☐ | Sensitive data not logged | Check logs → No IBAN, SSN, full name in plain text | |
| SEC202 | ☐ | HTTPS enforced | Try HTTP request → Redirects to HTTPS or returns error | |
| SEC203 | ☐ | SQL injection prevented | Try SQL injection in org# field → Parameterized query prevents execution | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 9: Deactivation Workflow (P1)

### 9.1 Manual Deactivation

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DEACT101 | ☐ | Deactivate active issuer | PUT /issuer/:id/deactivate with reason → Status changes to INACTIVE | |
| DEACT102 | ☐ | Deactivation reason required | PUT without reason → HTTP 422, error: "Reason required" | |
| DEACT103 | ☐ | Deactivation logged in history | Status history shows: ACTIVE → INACTIVE, deactivation reason, operator name, timestamp | |
| DEACT104 | ☐ | Cannot create invoices after deactivation | POST /invoice/event for deactivated issuer → HTTP 403, error: "Issuer not active" | |

### 9.2 Automatic Deactivation

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| DEACT201 | ☐ | Issuer deactivated on fraud detection | Fraud flag set → System automatically deactivates issuer, reason = "Fraud detected" | |
| DEACT202 | ☐ | Notification sent on auto-deactivation | Auto-deactivation → Email/Slack notification sent to operations team | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## Section 10: Observability & Monitoring (P1)

### 10.1 Logging

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| OBS101 | ☐ | Issuer creation logged | POST /issuer → Log entry with: issuerId, org#, country, timestamp, HTTP status | |
| OBS102 | ☐ | Decision rule evaluation logged | Each rule evaluation → Log entry with: rule name, decision, evaluation time, data source | |
| OBS103 | ☐ | Errors logged with context | Error occurs → Log includes: error message, stack trace, request ID, issuerId | |

### 10.2 Metrics

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| OBS201 | ☐ | Track issuer creation rate | Metric: issuers_created_total (counter) → Increments on each POST | |
| OBS202 | ☐ | Track status transitions | Metric: issuer_status_changes (counter, labeled by: from_status, to_status) | |
| OBS203 | ☐ | Track API response times | Metric: http_request_duration_seconds (histogram, labeled by: endpoint, status_code) | |

### 10.3 Alerts

| ID | ✅ | Test Description | Criteria | Notes |
|----|:---:|------------------|----------|-------|
| OBS301 | ☐ | Alert on high error rate | Error rate > 5% for 5 minutes → Alert sent to on-call engineer | |
| OBS302 | ☐ | Alert on slow API responses | 95th percentile response time > 5s → Alert sent | |
| OBS303 | ☐ | Alert on external API failures | D&B/Creditsafe failure rate > 20% → Alert sent | |

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## 📊 Test Summary

### Essential Tests (P0/P1)

| Section | Total Tests | ☑ Passed | ☒ Failed | ☐ Not Tested |
|---------|:-----------:|:--------:|:--------:|:------------:|
| **Section 1:** Critical API Tests | 12 | 0 | 0 | 12 |
| **Section 2:** Decision Rules (9 Rules + High-Risk) | 31 | 0 | 0 | 31 |
| **Section 3:** Database Validation | 10 | 0 | 0 | 10 |
| **Section 4:** Third-Party Integrations | 17 | 0 | 0 | 17 |
| **Section 5:** UI Testing | 20 | 0 | 0 | 20 |
| **Section 6:** Error Handling | 14 | 0 | 0 | 14 |
| **Section 7:** Performance | 5 | 0 | 0 | 5 |
| **Section 8:** Security | 7 | 0 | 0 | 7 |
| **Section 9:** Deactivation Workflow | 6 | 0 | 0 | 6 |
| **Section 10:** Observability & Monitoring | 9 | 0 | 0 | 9 |
| **TOTAL ESSENTIAL TESTS** | **131** | **0** | **0** | **131** |

### Overall Test Status

| Metric | Value |
|--------|-------|
| **Total Tests** | 131 |
| **Pass Rate** | 0% |
| **Critical Failures (P0)** | 0 |
| **Blockers** | 0 |
| **Test Coverage** | 0% |

> [!NOTE]
> **Priority Definitions:**
> - **P0 (Critical):** Must pass for production deployment - blocking issues
> - **P1 (Major):** Should pass for production - important but not blocking
> - **P2 (Minor):** Nice to have - can be addressed post-launch

---

## ✍️ Sign-Off

### QA Lead Approval

**Name:** _______________________________  
**Signature:** __________________________  
**Date:** _______________________________  
**Result:** ☐ Approved  ☐ Rejected  ☐ Conditional

**Comments:**
```
_______________________________________________________________
_______________________________________________________________
```

### Development Lead Approval

**Name:** _______________________________  
**Signature:** __________________________  
**Date:** _______________________________  
**Result:** ☐ Approved  ☐ Rejected  ☐ Conditional

**Comments:**
```
_______________________________________________________________
_______________________________________________________________
```

### Product Owner Approval

**Name:** _______________________________  
**Signature:** __________________________  
**Date:** _______________________________  
**Result:** ☐ Approved  ☐ Rejected  ☐ Conditional

**Comments:**
```
_______________________________________________________________
_______________________________________________________________
```

[↑ Back to top](#-issuer-onboarding---qa--testing-checklist)

---

## 📝 Document Information

| Field | Value |
|-------|-------|
| **Document Title** | Issuer Onboarding - QA & Testing Checklist |
| **Version** | 2.1 |
| **Last Updated** | 2026-02-20 |
| **Test Environment** | Staging/Sandbox |
| **Test Cycle** | [Cycle Number/Name] |

**Distribution:**
- QA Team
- Development Team
- Product Owners
- Operations Team

---

**END OF DOCUMENT**
