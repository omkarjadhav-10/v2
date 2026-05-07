# New Intern Guide: V1 & V2 Order Creation Flow

**Last Updated:** May 2026  
**Target Audience:** New developers joining the Baylor Genetics team  
**Prerequisites:** Basic understanding of REST APIs, Java/Spring Boot, and SQL

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Understanding the Migration](#understanding-the-migration)
3. [V2 Order Creation (New System)](#v2-order-creation-new-system)
4. [V1 Order Creation (Legacy System)](#v1-order-creation-legacy-system)
5. [V1-V2 Synchronization](#v1-v2-synchronization)
6. [Common Scenarios & Examples](#common-scenarios--examples)
7. [Troubleshooting Guide](#troubleshooting-guide)
8. [FAQs](#faqs)

---

## System Overview

### What Are We Building?

We're building an online portal for genetic testing laboratories to manage patient orders. Think of it like an e-commerce checkout flow, but for medical genetic tests instead of products.

**Key Players:**
- **Physicians**: Order genetic tests for their patients
- **Patients**: People receiving genetic testing
- **Lab**: Baylor Genetics Laboratory that processes samples and generates reports
- **Portal**: Web application where physicians place orders

### The Two Systems

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  V1 (Legacy System)                                             │
│  - Built in .NET                                                │
│  - SQL Server database                                          │
│  - Been running for years                                       │
│  - Still processes all lab orders                               │
│  - We CANNOT directly modify its database                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↑
                              │ REST API calls
                              │
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  V2 (New System - What You'll Work On)                          │
│  - Built in Java 21 + Spring Boot 3                             │
│  - PostgreSQL database (cep_order_management)                   │
│  - Modern React frontend                                        │
│  - Better user experience                                       │
│  - This is our codebase: CEP-ONLINE-PORTAL-V2                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why Two Systems?

We're in the **middle of a migration**. V1 has been running for years and can't be shut down overnight because:
- The lab systems (LIMS, billing) are integrated with V1
- Thousands of historical orders exist in V1
- We need zero downtime during the transition

**Current Strategy:** V2 is the new front door. When someone creates an order in V2, we automatically sync it to V1 so the lab can continue working as usual.

---

## Understanding the Migration

### Migration Phase: Dual-System Operation

Think of this like renovating a house while people are living in it. You can't just tear everything down and rebuild—you need to keep things working.

```
┌──────────────────────────────────────────────────────────────────┐
│                    Current State (2026)                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Frontend (React) → V2 API → V2 Database (PRIMARY)               │
│                       │                                          │
│                       └──→ Async Sync → V1 API → V1 Database     │
│                                                                  │
│  • New orders go to V2 first                                     │
│  • V2 is "system of record" (source of truth)                    │
│  • V1 gets a copy so lab systems keep working                    │
│  • If V1 sync fails, the order is still saved in V2              │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Database Architecture

You'll interact with **THREE databases**:

#### 1. cep_order_management (PostgreSQL) - V2's Database
**This is where YOU write data**

```sql
-- Core tables you'll touch:
orders               -- Main order data
patient              -- Patient demographics  
sample               -- Sample collection info
payment              -- Payment/insurance
order_icd10          -- Diagnosis codes
order_hpo            -- Clinical phenotypes
order_consent        -- Legal consents
legacy_order         -- V2 ↔ V1 ID mapping
order_sync_log       -- Sync audit trail
```

**Connection:** `orderManagementTransactionManager`  
**Package:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement`

#### 2. CustomerManagement (SQL Server) - Shared Identity
**This is READ-ONLY for V2**

```sql
-- Tables you'll READ from:
Customers           -- Physicians + Institutions (single-table inheritance)
CustomerTests       -- What tests each institution offers
```

**Why read-only?** V1 still writes to this database. We don't want to risk breaking V1 by changing physician or institution data.

**How to identify:** Look for `Discriminator` column:
- `"Physician"` = Doctor/medical professional
- `"Institution"` = Hospital/clinic/organization

#### 3. MGL (SQL Server) - Reference Data
**Also READ-ONLY**

Gene catalogs, test definitions, and other reference data. Used for lookups but never modified.

---

## V2 Order Creation (New System)

This is the main flow you'll be working with. Let's walk through it step by step.

### High-Level Flow

```
1. User fills out order form in React frontend
2. Frontend sends JSON to POST /api/v1/orders
3. V2 validates and saves to PostgreSQL
4. V2 returns success immediately 
5. V2 asynchronously syncs to V1 in background
```

**Key Insight:** The user gets a response from step 4, NOT step 5. This is called "async sync" and it's intentional—we don't want V1 problems to slow down V2.

### Detailed Step-by-Step

#### Step 1: Request Arrives

```java
// OrderController.java
@PostMapping("/api/v1/orders")
public ResponseEntity<ApiResponse<OrderResponse>> createOrder(
    @Valid @RequestBody CreateOrderRequest requestDto
) {
    // Two-gate security check
    if (status is SUBMIT or VOB_SENT) {
        SecurityUtil.authorizeUserToSubmitOrder();
    } else {
        SecurityUtil.authorizeUserToCreateOrder();
    }
    
    return orderService.createOrder(requestDto);
}
```

**What's happening:**
- `@Valid` triggers bean validation (checks `@NotNull`, `@NotBlank`, etc.)
- Security checks ensure user has permission for their institution
- Different permissions for DRAFT (save) vs SUBMIT (send to lab)

#### Step 2: The Order Service (The Brain)

```java
// OrderServiceImpl.java
@Transactional  // Everything or nothing—no partial saves
public OrderResponse createOrder(CreateOrderRequest request) {
    
    // 1. VALIDATION PHASE
    orderValidator.validate(request);              // General rules
    submitOrderValidator.validate(request);        // SUBMIT-only rules
    testConfig.checkRequest(request);              // Test-specific rules
    
    // 2. ID GENERATION
    String orderId = snowflakeGenerator.generateOrderId();
    // Example: "BG-2900-5K8P2A" 
    // (not a random string—encodes timestamp + machine ID)
    
    // 3. SAVE ORDER ENTITY
    Order order = orderMapper.toEntity(request, orderId);
    order = orderRepository.save(order);  // Gets database ID
    
    // 4. SAVE RELATED ENTITIES (in order)
    saveOrderPhysicians();        // Who ordered it
    handlePatient();              // Create/update patient
    addAdditionalMembers();       // Care team
    addPaymentInformation();      // Insurance
    addICD10Codes();              // Diagnosis codes
    addHPOTerms();                // Clinical phenotypes
    addGenes();                   // Genes of interest
    saveConsents();               // Legal consents
    createIndications();          // Why this test?
    saveFamilyHistory();          // Family medical history
    
    // If prenatal order:
    saveGestation();
    saveFetusInfo();
    
    // 5. SAMPLES
    setOrderSample();             // Link samples to order
    
    // 6. DOCUMENTS (only for SUBMIT status)
    if (status == SUBMIT) {
        uploadOrderSummaryDocument();  // Converts HTML to PDF
    }
    
    // 7. VOB (Verification of Benefits)
    if (isVOB) {
        saveVobData();
    }
    
    // 8. BUILD RESPONSE
    OrderResponse response = buildResponse(order);
    
    return response;
    
    // Transaction commits here!
}
```

**After transaction commits:**
```java
// This runs OUTSIDE the transaction, in a separate thread
v1SyncService.submitNewOrderToV1(request, response, orderId);
```

#### Step 3: The Snowflake ID

**Why not just use a database auto-increment ID?**

```
Old way (V1):  Order ID = 1, 2, 3, 4, 5...
Problems:
  - Predictable (security concern)
  - Reveals volume (competitor intelligence)
  - Requires database round-trip
  - Doesn't work with multiple servers

New way (V2):  Order ID = BG-2900-5K8P2A
Benefits:
  - Globally unique (multiple servers can generate independently)
  - Time-sortable (encodes creation timestamp)
  - Human-readable
  - No database needed to generate
```

**Format:** `BG-{MACHINE_ID}-{TIMESTAMP}`
- `BG` = Baylor Genetics
- `2900` = Base-36 encoded machine identifier
- `5K8P2A` = Base-36 encoded seconds since Jan 1, 2025

#### Step 4: Understanding the Data Model

**One-to-Many Relationships:**
```
Order (1) ──→ (Many) OrderIcd10
Order (1) ──→ (Many) OrderHpo
Order (1) ──→ (Many) OrderGene
Order (1) ──→ (Many) OrderConsent
Order (1) ──→ (Many) FamilyHistory
```

**Many-to-Many (Junction Tables):**
```
Order (Many) ←─→ (Many) Sample
  via order_sample junction table

Order (Many) ←─→ (Many) Physician  
  via order_physician junction table (with role)
```

**Example: Multiple Physicians per Order**
```sql
-- order_physician table
order_id              | physician_id | role
"BG-2900-5K8P2A"      | 1234         | AUTHORIZING_PROVIDER
"BG-2900-5K8P2A"      | 5678         | PRIMARY_CONTACT
"BG-2900-5K8P2A"      | 9012         | GENETIC_COUNSELOR
```

### The Request Payload Explained

Here's what the frontend sends (simplified):

```json
{
  "testCode": "1300",                    // What test to run
  "institutionId": 4521,                 // Which hospital
  "instituteCode": "BMGL",               
  "authorizationProviderId": 1234,       // Which doctor ordered it
  "status": "SUBMIT",                    // DRAFT or SUBMIT
  
  "patient": {                           // The person being tested
    "firstName": "Jane",
    "lastName": "Doe",
    "dateOfBirth": "1985-03-15",
    "sexAssignedAtBirth": "FEMALE",
    "medicalRecordNumber": "MRN-12345",
    
    "samples": [{                        // What samples to collect
      "status": "TO_BE_COLLECTED",
      "sample": {
        "type": "BLOOD"
      }
    }],
    
    "relatives": [                       // Family members (if applicable)
      {
        "relationship": "MOTHER",
        "firstName": "Mary",
        "sample": { "type": "BLOOD" }
      }
    ]
  },
  
  "payment": {                           // How to bill
    "method": "INSURANCE",
    "insurances": [
      {
        "insuranceCompanyName": "Blue Cross",
        "policyNumber": "POL-123456"
      }
    ]
  },
  
  "clinicalNotes": "Patient presents with developmental delay.",
  
  "icdCodes": [                          // Diagnosis codes
    {
      "code": "F84.0",
      "description": "Autism spectrum disorder"
    }
  ],
  
  "hpoTerms": [                          // Clinical phenotypes
    {
      "hpoId": "HP:0001249",
      "name": "Intellectual disability"
    }
  ],
  
  "genesOfInterest": [                   // Specific genes to look at
    {
      "geneId": 1001,
      "geneName": "BRCA1"
    }
  ],
  
  "consents": [                          // Legal agreements
    {
      "consentType": "GENETIC_TESTING",
      "signed": true
    }
  ],
  
  "orderSummaryHtml": "<html>...</html>" // Required for SUBMIT
}
```

### Order Status Flow

```
┌─────────────────────────────────────────────────────────────┐
│                   Status Transitions                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  DRAFT ──────→ DRAFT (auto-save, can edit)                  │
│    │                                                        │
│    └──────────→ SUBMIT (terminal, no more edits)            │
│                                                             │
│  VOB_DRAFT ───→ VOB_DRAFT (auto-save)                       │
│    │                                                        │
│    └──────────→ VOB_SENT (terminal)                         │
│                                                             │
│                                                             │
│  Once SUBMIT or VOB_SENT: Order is locked                   │
│  Only lab can update it after this point                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**What is VOB?** Verification of Benefits—a pre-authorization process where we check with insurance BEFORE running the test.

### Validation Layers

**Three layers protect data quality:**

```java
// Layer 1: Bean Validation (happens automatically)
public class CreateOrderRequest {
    @NotBlank(groups = {DraftValidation.class, SubmitValidation.class})
    private String instituteCode;
    
    @NotNull(groups = SubmitValidation.class)  // Only required for SUBMIT
    private String orderSummaryHtml;
}

// Layer 2: Order Validator (custom business rules)
OrderValidator.validate(request);
// Checks: patient has required fields, samples match patients, etc.

// Layer 3: Submit Validator (only runs for SUBMIT status)
SubmitOrderValidator.validate(request);
// Checks: no duplicate samples, orderSummaryHtml present, etc.
```

---

## V1 Order Creation (Legacy System)

V1 is a black box to us—we can only interact via REST API. Here's what you need to know.

### V1 API Contract

**Endpoint:** `POST /api/Orders/{institutionCode}`

**Authentication:**
```
Authorization: Bearer <JWT_TOKEN>
institutioncodes: BMGL
Content-Type: application/json
```

### V1 Payload Structure

The V1 payload is **very different** from V2. Here are key differences:

```
┌──────────────────────┬─────────────────────────┬────────────────────────┐
│      Concept         │          V1             │          V2            │
├──────────────────────┼─────────────────────────┼────────────────────────┤
│ Order ID             │ Integer (12345)         │ String (BG-2900-...)   │
├──────────────────────┼─────────────────────────┼────────────────────────┤
│ Physician            │ Full object inline      │ Just ID (1234)         │
├──────────────────────┼─────────────────────────┼────────────────────────┤
│ Patient & Relatives  │ Both arrays at root     │ Patient has relatives  │
├──────────────────────┼─────────────────────────┼────────────────────────┤
│ Sample link          │ testIndex (0, 1, 2)     │ Direct FK relationship │
├──────────────────────┼─────────────────────────┼────────────────────────┤
│ Missing fields       │ "Na" or "None"          │ null                   │
└──────────────────────┴─────────────────────────┴────────────────────────┘
```

### Key V1 Quirks

#### 1. Duplicate Patient Data
```json
{
  "patient": { "firstName": "Jane", "testIndex": 0 },
  "patients": [
    { "firstName": "Jane", "testIndex": 0 },  // Same as above!
    { "firstName": "Mary", "testIndex": 1 }   // Mother
  ]
}
```

**Why?** Legacy code. Some V1 endpoints read `patient`, others read `patients[0]`. Both must be present and identical.

#### 2. Test Index Linking

Everything links via `testIndex`:

```json
{
  "tests": [
    { "testCode": "1300", "testName": "Blueprint Panel" }
  ],
  "patients": [
    { "firstName": "Jane", "testIndex": 0 }  // Links to tests[0]
  ],
  "samples": [
    { "sampleTypeName": "Blood", "testIndex": 0 }  // Links to tests[0]
  ],
  "consents": [
    { "consentId": 101, "testIndex": 0 }  // Links to tests[0]
  ]
}
```

#### 3. Sentinel Values

V1 uses special strings instead of null:

```json
{
  "accessionNumber": "Na",              // Not "null"—means "not assigned"
  "medicalRecordNumber": "None"         // Not null
}
```

**Why?** V1's .NET code has `[Required]` attributes that reject null. These strings are sentinels meaning "no value yet."

#### 4. Array Fields That Look Singular

```json
{
  "EmailNotificationRecipient": ["email1@example.com", "email2@example.com"]
}
```

Notice the field name is **singular** but the value is an **array**. This is a legacy naming mistake that can't be changed without breaking V1.

---

## V1-V2 Synchronization

This is where it gets interesting. After V2 saves an order, it needs to tell V1 about it.

### Sync Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                   Async Post-Commit Sync                       │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. V2 Transaction Commits (order saved to PostgreSQL)         │
│                                                                │
│  2. Spring fires TransactionSynchronizationAdapter             │
│     (this is like a "after save" hook)                         │
│                                                                │
│  3. V1SyncService runs in separate thread pool                 │
│     (v1SyncExecutor - 10 worker threads)                       │
│                                                                │
│  4. Transform V2 DTO → V1 DTO                                  │
│     (V1CreateOrderMapper does the heavy lifting)               │
│                                                                │
│  5. POST to V1 API                                             │
│                                                                │
│  6a. SUCCESS:                                                  │
│      - Save mapping to legacy_order table                      │
│      - Log to order_sync_log (status=SUCCESS)                  │
│      - If SUBMIT: Call V1 /approve endpoint                    │
│                                                                │
│  6b. FAILURE:                                                  │
│      - Log to order_sync_log (status=FAILED)                   │
│      - Send notification to ops team                           │
│      - Order remains in V2, needs manual retry                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### The Sync Service

```java
// V1SyncServiceImpl.java
public void submitNewOrderToV1(
    CreateOrderRequest v2Request,
    OrderResponse v2Response,
    String v2OrderId
) {
    // 1. Log sync start
    orderSyncLog.create(v2OrderId, INIT_CREATE, PENDING);
    
    // 2. Transform V2 → V1
    CreateOrderRequestDto v1Request = 
        v1CreateOrderMapper.toV1CreateOrderRequest(v2Request);
    
    // 3. Apply field filtering (some tests don't need all fields)
    fieldFilteringService.applyFieldFiltering(v1Request, testCodes);
    
    // 4. Log full payloads (for debugging failed syncs)
    orderSyncLog.update(v2OrderId, CREATE, IN_PROGRESS, 
                        v2Payload, v1Payload);
    
    // 5. Call V1 API
    CreateOrderResponseDto v1Response = 
        v1ApiClient.createOrder(instituteCode, v1Request);
    
    // 6. Save mapping
    legacyOrderRepository.save(
        new LegacyOrder(v2OrderId, v1Response.getId())
    );
    
    // 7. If SUBMIT status, also approve in V1
    if (isSubmit) {
        v1ApiClient.approveOrder(instituteCode, v1Response.getId());
    }
    
    // 8. Log success
    orderSyncLog.update(v2OrderId, CREATE, SUCCESS, v1Response);
}
```

### Handling Sync Failures

**Important:** If V1 sync fails, the V2 order is still saved! This is intentional.

```
Scenario: V1 is down for maintenance

User creates order in V2:
  ✅ V2 saves order to PostgreSQL
  ✅ User receives success response
  ❌ V1 sync fails (V1 is unreachable)

Result:
  - Order exists in V2 ✓
  - Order does NOT exist in V1 ✗
  - order_sync_log shows FAILED status
  - Ops team gets notification

Manual resolution needed:
  1. Check order_sync_log for failed syncs
  2. Verify V1 is back online
  3. Re-trigger sync (TBD: automated retry not implemented)
```

### The Mapping Tables

Two tables track V1 ↔ V2 relationships:

#### legacy_order
```sql
CREATE TABLE legacy_order (
    id BIGSERIAL PRIMARY KEY,
    order_id VARCHAR(50) NOT NULL,        -- V2 Snowflake ID
    legacy_order_id BIGINT NOT NULL,      -- V1 integer ID
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

-- Example:
order_id              | legacy_order_id
"BG-2900-5K8P2A"      | 12345
```

**Used for:** Looking up "What's the V1 ID for this V2 order?"

#### id_mapping_log
```sql
CREATE TABLE id_mapping_log (
    id BIGSERIAL PRIMARY KEY,
    v2_order_id VARCHAR(50),
    v1_order_id VARCHAR(50),
    v2_sample_id BIGINT,
    v1_sample_id BIGINT,
    entity_type VARCHAR(50),              -- SAMPLE, PATIENT, etc.
    -- more fields...
);
```

**Used for:** Granular entity mappings, especially for samples and patients. Also used by the migration service for historically migrated orders.

### Sync Monitoring

**Where to check sync status:**

```sql
-- Recent syncs
SELECT 
    order_id,
    sync_operation,
    sync_status,
    initiated_at,
    error_message
FROM order_sync_log
ORDER BY initiated_at DESC
LIMIT 20;

-- Failed syncs needing attention
SELECT * FROM order_sync_log
WHERE sync_status = 'FAILED'
AND initiated_at > NOW() - INTERVAL '24 hours';
```

**Sync operations you'll see:**
- `INIT_CREATE` - Sync started
- `CREATE` - V1 POST called
- `UPDATE` - V1 PATCH called
- `APPROVE` - V1 approve endpoint called
- `VOB_SEND` - V1 VOB endpoint called
- `SWITCH_TO_CREATE` - Update fell back to create (V1 ID not found)

---

## Common Scenarios & Examples

### Scenario 1: Creating a Simple Draft Order

**User Story:** Doctor wants to start an order but finish it later.

**Request:**
```json
POST /api/v1/orders
{
  "testCode": "1300",
  "institutionId": 4521,
  "instituteCode": "BMGL",
  "authorizationProviderId": 1234,
  "status": "DRAFT",
  "patient": {
    "firstName": "Jane",
    "lastName": "Doe",
    "dateOfBirth": "1985-03-15",
    "sexAssignedAtBirth": "FEMALE"
  }
}
```

**What happens:**
1. OrderController validates minimal required fields
2. OrderService creates order with status=DRAFT
3. Snowflake ID generated: "BG-2900-5K8P2A"
4. Saved to V2 database
5. Response returned immediately
6. V1 sync triggered async (V1 gets the draft)

**Database state:**
```sql
-- orders table
id    | order_id         | status | test_code | patient_id
98765 | BG-2900-5K8P2A   | DRAFT  | 1300      | 11111

-- patient table  
id    | first_name | last_name | date_of_birth
11111 | Jane       | Doe       | 1985-03-15

-- order_sync_log
order_id         | sync_operation | sync_status | v1_order_id
BG-2900-5K8P2A   | CREATE         | SUCCESS     | 12345
```

### Scenario 2: Updating a Draft Order (Auto-Save)

**User Story:** Doctor adds more information to the draft.

**Request:**
```json
PATCH /api/v1/orders/BG-2900-5K8P2A
{
  "testCode": "1300",               // Same (immutable)
  "institutionId": 4521,            // Same (immutable)
  "instituteCode": "BMGL",          // Same (immutable)
  "authorizationProviderId": 1234,
  "status": "DRAFT",                // Still DRAFT
  "patient": { ... },               // Same patient
  "clinicalNotes": "Patient has family history of heart disease.",
  "icdCodes": [
    { "code": "Z82.49", "description": "Family history of heart disease" }
  ]
}
```

**What happens:**
1. OrderController validates orderId exists
2. OrderService checks immutable fields (testCode, etc.)
3. Validates status transition (DRAFT → DRAFT is OK)
4. **Deletes** existing ICD codes
5. **Inserts** new ICD codes
6. Updates clinicalNotes
7. V1 sync creates JSON Patch and sends to V1:
   ```json
   [
     { "op": "replace", "path": "/additionalClinicalInformation", 
       "value": "Patient has family history..." },
     { "op": "add", "path": "/indicationsForTesting/0",
       "value": { "indicationId": "Z82.49", ... } }
   ]
   ```

**Key insight:** V2 does delete-and-insert. V1 receives a delta patch.

### Scenario 3: Submitting an Order to the Lab

**User Story:** Doctor is done filling out the form and wants to send it to the lab.

**Request:**
```json
PATCH /api/v1/orders/BG-2900-5K8P2A
{
  // ... all previous fields ...
  "status": "SUBMIT",  // Status change: DRAFT → SUBMIT
  "orderSummaryHtml": "<html><body>Order Summary...</body></html>",
  "consents": [
    { "consentType": "GENETIC_TESTING", "signed": true },
    { "consentType": "ACMG_SECONDARY_FINDINGS", "signed": true }
  ]
}
```

**What happens:**
1. SubmitOrderValidator runs (in addition to OrderValidator)
   - Checks orderSummaryHtml is present
   - Checks no duplicate samples
   - Validates all required fields for submit
2. OrderService transitions status: DRAFT → SUBMIT
3. **HTML-to-PDF conversion:**
   ```java
   byte[] pdfBytes = convertHtmlToPdf(orderSummaryHtml);
   DocumentUploadResponse response = 
       documentServiceClient.upload(pdfBytes, "order-summary.pdf");
   orderDocumentRepository.save(
       new OrderDocument(orderId, response.getDocumentId())
   );
   ```
4. Order saved with status=SUBMIT (now locked—no more edits)
5. V1 sync does two API calls:
   ```java
   // Call 1: Update order in V1
   v1ApiClient.updateOrder(instituteCode, v1OrderId, patches);
   
   // Call 2: Approve order (tells V1 lab can start processing)
   v1ApiClient.approveOrder(instituteCode, v1OrderId);
   ```

**After submit:**
- Order is locked in V2 (any PATCH will fail with error ORD_2007)
- Lab receives notification
- Lab assigns accession number
- Sample collection begins

### Scenario 4: Reanalysis Order

**User Story:** Patient had a test 2 years ago. New genes discovered. Doctor wants to re-run analysis on the same sample.

**Request:**
```json
POST /api/v1/orders
{
  "testCode": "1300",
  "institutionId": 4521,
  "instituteCode": "BMGL",
  "authorizationProviderId": 1234,
  "status": "SUBMIT",
  "sourceOrderId": "BG-1800-3J5K1M",  // Original order from 2 years ago
  "patient": { ... },  // Same patient
  // lockPatientInfo will be set to true automatically
}
```

**What happens:**
1. OrderService detects sourceOrderId is present
2. Looks up original order: `Order sourceOrder = orderRepository.findByOrderId(sourceOrderId)`
3. Copies relevant data:
   - Patient demographics (locked—can't change)
   - Original sample information
4. Creates new order with source linkage
5. V1 sync includes reanalysis context:
   ```java
   v1Request.setOrderOptions({
       "reanalysisOf": v1SampleId,        // V1's sample ID from original
       "assignNewLabNumber": false        // Use existing sample
   });
   ```

**Key fields:**
- `sourceOrderId` - V2's Snowflake ID of original order
- `reanalysisOf` (in V1) - V1's integer sample ID
- `lockPatientInfo` - Prevents accidental patient data changes

### Scenario 5: Prenatal Order with Mother + Father Samples

**User Story:** Pregnant patient needs prenatal genetic testing. Lab needs samples from mother, father, and fetal tissue.

**Request:**
```json
POST /api/v1/orders
{
  "testCode": "1900",  // Prenatal exome
  "status": "SUBMIT",
  "patient": {
    "firstName": "Sarah",
    "lastName": "Smith",
    "dateOfBirth": "1990-05-10",
    "sexAssignedAtBirth": "FEMALE",
    "samples": [{
      "status": "TO_BE_COLLECTED",
      "sample": { "type": "AMNIOTIC_FLUID" },
      "ccVolume": 15.0,
      "isTA": true  // Transabdominal collection
    }],
    "relatives": [
      {
        "relationship": "MOTHER",
        "firstName": "Sarah",
        "lastName": "Smith",
        "samples": [{ 
          "status": "TO_BE_COLLECTED",
          "sample": { "type": "BLOOD" }
        }]
      },
      {
        "relationship": "FATHER",
        "firstName": "John",
        "lastName": "Smith",
        "samples": [{
          "status": "TO_BE_COLLECTED",
          "sample": { "type": "BLOOD" }
        }]
      }
    ]
  },
  "imaging": [
    {
      "imagingType": "ULTRASOUND",
      "findings": "Normal fetal anatomy",
      "date": "2026-04-15"
    }
  ],
  "prenatalTestingStatus": [
    {
      "testType": "NIPT",
      "result": "NORMAL",
      "date": "2026-03-10"
    }
  ]
}
```

**What happens:**
1. OrderService generates family number: "BG-FAM-98765"
2. Creates 3 patients:
   - Proband (Sarah, testIndex=0)
   - Mother (Sarah again, testIndex=1) 
   - Father (John, testIndex=2)
3. Creates 3 samples (amniotic fluid, 2 blood samples)
4. Saves gestation data and fetus info
5. V1 sync sends all three as `patients` array with family linkage

**Prenatal-specific tables used:**
- `gestation` - Pregnancy details
- `patient_order_fetus_info` - Fetal information
- `order_imaging` - Ultrasound findings
- `prenatal_testing_status` - Prior test results (NIPT, AFP, etc.)

---

## Troubleshooting Guide

### Problem: Order created in V2 but not in V1

**Symptoms:**
- User reports order "missing" in lab system
- order_sync_log shows FAILED status

**Diagnosis:**
```sql
SELECT 
    order_id,
    sync_operation,
    sync_status,
    error_message,
    http_status_code,
    v1_response
FROM order_sync_log
WHERE order_id = 'BG-2900-5K8P2A';
```

**Common causes:**
1. **V1 API down** - Check V1 health endpoint
2. **Authentication expired** - Token in order_sync_log might be expired
3. **V1 validation failed** - Check v1_response for error details
4. **Network timeout** - Check webclient timeout configs

**Resolution:**
1. Fix underlying issue (V1 back online, etc.)
2. Retrieve failed payload from order_sync_log
3. Manually retry (or trigger automated retry if implemented)

### Problem: Order stuck in DRAFT, can't submit

**Symptoms:**
- User clicks "Submit" but gets validation error
- Frontend shows "Missing required fields"

**Diagnosis:**
```java
// Check what SubmitOrderValidator is complaining about
try {
    submitOrderValidator.validate(request);
} catch (ValidationException e) {
    log.error("Validation errors: {}", e.getErrors());
}
```

**Common causes:**
1. **Missing orderSummaryHtml** - Frontend didn't generate it
2. **Duplicate samples** - Same sample linked twice
3. **Missing consents** - Required consents not signed
4. **Invalid test code** - Test not available for institution

**Resolution:**
- Check validation error response (400 Bad Request body)
- Frontend should display specific missing fields
- Ensure all conditional validations pass for the test type

### Problem: Patient data mismatch between V1 and V2

**Symptoms:**
- Patient name shows differently in V1 vs V2
- MRN doesn't match

**Diagnosis:**
```sql
-- Check V2 patient
SELECT * FROM patient WHERE id = (
    SELECT patient_id FROM "order" WHERE order_id = 'BG-2900-5K8P2A'
);

-- Check V1 payload sent
SELECT v1_payload->'patient' FROM order_sync_log
WHERE order_id = 'BG-2900-5K8P2A';
```

**Common causes:**
1. **Mapping bug** - V1CreateOrderMapper has error
2. **Locked patient info** - Reanalysis order but patient changed
3. **Async timing** - Patient updated in V2 after V1 sync

**Resolution:**
- Never modify patient data on reanalysis orders
- If patient data wrong, cancel and recreate order
- Check PatientMapper logic for transformation errors

### Problem: Sync log shows SUCCESS but V1 order not found

**Symptoms:**
- order_sync_log.sync_status = SUCCESS
- V1 API returns 404 for the order ID

**Diagnosis:**
```sql
-- Check mapping
SELECT * FROM legacy_order WHERE order_id = 'BG-2900-5K8P2A';

-- Check if multiple syncs happened
SELECT * FROM order_sync_log 
WHERE order_id = 'BG-2900-5K8P2A'
ORDER BY initiated_at;
```

**Common causes:**
1. **V1 order deleted** - Someone cancelled in V1
2. **Wrong V1 institution code** - Synced to wrong tenant
3. **Mapping corruption** - legacy_order has wrong V1 ID

**Resolution:**
- Verify in V1 system directly (not just API)
- Check institution code matches between V2 and V1
- If needed, trigger full re-sync with updated mapping

### Problem: Transaction timeout during order creation

**Symptoms:**
- 500 Internal Server Error
- Logs show: `Transaction timeout after 30s`

**Diagnosis:**
```java
// Check slow operations
@Transactional(timeout = 30)  // 30 seconds
public OrderResponse createOrder(...) {
    // If any step takes > 30s total, transaction rolls back
}
```

**Common causes:**
1. **PDF upload slow** - Document service unreachable
2. **Database connection pool exhausted** - Check HikariCP
3. **V1 API call in transaction** - Should never happen (it's async)

**Resolution:**
- Check document service health
- Increase connection pool size if needed
- Verify V1 sync is truly async (post-commit)
- Consider breaking up large transactions

---

## FAQs

### Q: Why do we have both POST and PATCH using the same DTO?

**A:** The frontend form is stateful—it holds the complete order in memory and sends the full current state on every auto-save. So whether it's the first save (POST) or tenth save (PATCH), the payload structure is identical. 

Having one DTO means:
- Less code duplication
- Same validation rules apply to both
- Simpler mapping logic
- Frontend doesn't need to compute deltas

### Q: What happens if I send the same POST request twice?

**A:** You'll create two separate orders with different Snowflake IDs. There's **no idempotency key** at the API level.

For DRAFT orders, this might be acceptable (user can delete duplicates). For SUBMIT orders, the SubmitOrderValidator checks for duplicate samples, which provides some protection.

### Q: Can I change the testCode after order is created?

**A:** No. testCode, instituteCode, and institutionId are **immutable**. If you need a different test, create a new order.

Why? These fields determine:
- Which lab processes the order
- Pricing and billing rules
- Sample requirements
- Consent requirements

Changing them mid-order would create chaos.

### Q: How do I know if an order came from V2 or was migrated from V1?

**A:** Check the `source` field:

```sql
SELECT order_id, source FROM "order";

-- Values:
-- "OOPV2" = Created in V2 portal
-- "V1_MIGRATION" = Migrated from V1
-- "API" = Created via V2 API (not portal)
```

Also check if there's a V1 ID:
```sql
SELECT 
    o.order_id,
    o.source,
    lo.legacy_order_id,
    CASE 
        WHEN lo.legacy_order_id IS NOT NULL THEN 'V2-created, synced to V1'
        WHEN im.v1_order_id IS NOT NULL THEN 'Migrated from V1'
        ELSE 'V2-only (sync failed or pending)'
    END as origin
FROM "order" o
LEFT JOIN legacy_order lo ON o.order_id = lo.order_id
LEFT JOIN id_mapping_log im ON o.order_id = im.v2_order_id;
```

### Q: What's the difference between orderPhysician and vobRequestor in V1?

**A:** Both reference the same physician, but serve different purposes:

- **orderPhysician**: Clinical role—authorized to order the genetic test
- **vobRequestor**: Administrative role—requests insurance pre-authorization

They're often the same person. V1 separates them because the workflows are different (clinical vs billing).

### Q: Why does V1 use testIndex instead of foreign keys?

**A:** V1 is a flat-table design from before modern ORMs. `testIndex` is a positional link:

```
tests[0] = Exome test
patients[0].testIndex = 0 (linked to Exome)
samples[0].testIndex = 0 (linked to Exome)
```

V2 uses proper foreign keys:
```sql
sample.order_id → order.id
order_sample.order_id → order.id
order_sample.sample_id → sample.id
```

### Q: When does V1 assign the lab number?

**A:** When the order is **approved** in V1. The flow:

1. V2 creates order, syncs to V1 (no lab number yet)
2. V2 calls `POST /api/Orders/{code}/{id}/approve`
3. V1 assigns lab number (e.g., "BGL-98765")
4. V1 returns lab number in response
5. V2 could update its sample record (not currently implemented)

The lab number is V1's internal tracking ID for the physical sample.

### Q: Why can't I edit an order after SUBMIT?

**A:** Once submitted, the order enters the lab's processing pipeline. Allowing edits would create:

- **Clinical risk**: Lab might process old version of order
- **Billing issues**: Insurance already billed for specific test
- **Audit problems**: Changes after submission hard to track
- **Sample confusion**: Sample already in transit with label

If changes needed after SUBMIT, standard practice is:
1. Cancel the order
2. Create new order with corrections

### Q: How do I test V1 sync locally?

**A:** You'll need:

1. **V1 dev environment URL** in application-local.yml:
   ```yaml
   v1:
     api:
       base-url: https://v1-dev.baylorgenetics.com
   ```

2. **Valid JWT token** for V1 authentication

3. **Test institution code** that exists in both V2 and V1

4. **Test physician** that exists in CustomerManagement

Then:
```bash
# Create order in V2
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d @test-order.json

# Check sync log
psql -d cep_order_management \
  -c "SELECT * FROM order_sync_log ORDER BY initiated_at DESC LIMIT 1;"

# Verify in V1 (if you have access)
curl https://v1-dev.baylorgenetics.com/api/Orders/BMGL/12345 \
  -H "Authorization: Bearer V1_TOKEN"
```

### Q: What's the retry strategy for failed syncs?

**A:** Currently, there's **no automatic retry**. Failed syncs are logged and require manual intervention.

For UPDATE operations, there's a 3-attempt retry loop to find the V1 order ID (in case of race condition), but if the actual API call fails, it doesn't retry.

Future improvement: Implement a cron job that retries FAILED syncs from order_sync_log.

### Q: How do I add a new field to the order?

**A:** Here's the checklist:

1. **Add to DTO**:
   ```java
   // CreateOrderRequest.java
   @Size(max = 100)
   private String myNewField;
   ```

2. **Add to Entity**:
   ```java
   // Order.java
   @Column(name = "my_new_field", length = 100)
   private String myNewField;
   ```

3. **Create migration**:
   ```sql
   -- V2.5__add_my_new_field.sql
   ALTER TABLE cep_core.order 
   ADD COLUMN my_new_field VARCHAR(100);
   ```

4. **Update mapper**:
   ```java
   // OrderMapper.java
   order.setMyNewField(request.getMyNewField());
   ```

5. **Add to V1 mapper** (if V1 needs it):
   ```java
   // V1CreateOrderMapper.java
   v1Dto.setMyV1FieldName(v2Request.getMyNewField());
   ```

6. **Update response builder**:
   ```java
   // OrderResponseBuilder.java
   response.setMyNewField(order.getMyNewField());
   ```

7. **Add tests**:
   ```java
   @Test
   void shouldSaveMyNewField() {
       CreateOrderRequest request = ...;
       request.setMyNewField("test value");
       
       OrderResponse response = orderService.createOrder(request);
       
       assertEquals("test value", response.getMyNewField());
   }
   ```

---

## Next Steps for New Interns

### Week 1: Read & Understand
- [ ] Read this entire guide
- [ ] Review the architecture diagrams
- [ ] Explore the V2 codebase (OrderController → OrderService → Repositories)
- [ ] Set up local development environment
- [ ] Create a test order in local V2 instance

### Week 2: Hands-On
- [ ] Debug through createOrder() flow with breakpoints
- [ ] Query order_sync_log and understand sync states
- [ ] Make a small change to validation rules (with mentor)
- [ ] Add a new field to OrderResponse (with mentor)

### Week 3: Integration
- [ ] Understand V1 payload structure
- [ ] Review V1CreateOrderMapper
- [ ] Test V1 sync in dev environment
- [ ] Investigate a failed sync and document findings

### Week 4: Contribute
- [ ] Pick a small bug from backlog
- [ ] Fix it with tests
- [ ] Submit PR with detailed description
- [ ] Deploy to QA and verify

---

## Useful Resources

### Code Locations
```
services/cep-order-service/
├── controller/
│   └── OrderController.java          ← HTTP endpoints
├── service/
│   ├── OrderServiceImpl.java         ← Business logic
│   └── V1SyncServiceImpl.java        ← V1 synchronization
├── mapper/
│   ├── OrderMapper.java              ← V2 DTO → Entity
│   └── V1CreateOrderMapper.java      ← V2 → V1 transformation
├── validator/
│   ├── OrderValidator.java           ← Draft validation
│   └── SubmitOrderValidator.java     ← Submit validation
└── repository/
    ├── OrderRepository.java
    └── LegacyOrderRepository.java

shared/cep-common/
└── entity/
    └── datasource/
        ├── cepOrderManagement/       ← V2 entities
        └── customerManagement/       ← Shared identity entities
```

### Database Schema Scripts
```
scripts/db/
├── setup-database.sql                ← PostgreSQL setup
└── migrations/                       ← Liquibase changesets
```

### API Documentation
- Swagger UI: `http://localhost:8080/swagger-ui.html`
- OpenAPI JSON: `http://localhost:8080/v3/api-docs`

### Logging
```bash
# Watch order creation logs
tail -f logs/cep-order-service.log | grep "OrderServiceImpl"

# Watch V1 sync logs  
tail -f logs/cep-order-service.log | grep "V1SyncService"

# Watch database queries
tail -f logs/cep-order-service.log | grep "Hibernate:"
```

---

## Glossary

**ACMG** - American College of Medical Genetics. Sets standards for genetic testing.

**Add-on tests** - Extra optional tests bundled with primary test (e.g., mtDNA analysis added to exome sequencing).

**BPCO** - Blueprint Carve-Out. Specific insurance panel tests with add-on options.

**Consent** - Legal agreement patient signs to allow genetic testing and data use.

**DRAFT** - Order status meaning "in progress, not yet sent to lab."

**HPO** - Human Phenotype Ontology. Standardized vocabulary for clinical features.

**ICD-10** - International Classification of Diseases codes. Used for billing and medical records.

**Indication** - Medical reason/justification for ordering a genetic test.

**Institution** - Hospital, clinic, or medical practice placing orders.

**Kit** - Sample collection kit sent to patient's home.

**Lab number** - V1's internal tracking ID for a physical sample (e.g., "BGL-98765").

**MRN** - Medical Record Number. Patient identifier within a hospital system.

**Phenotype** - Observable characteristics/symptoms of a patient.

**Physician** - Doctor or medical professional authorized to order genetic tests.

**Proband** - The primary patient being tested (vs relatives).

**Reanalysis** - Re-running analysis on previously collected sample with updated methods.

**Relative** - Family member tested alongside proband (mother, father, sibling).

**Snowflake ID** - Distributed unique identifier algorithm (time-based + machine ID).

**SUBMIT** - Order status meaning "sent to lab for processing."

**Test code** - Numeric identifier for type of genetic test (e.g., 1300 = Blueprint Panel).

**VOB** - Verification of Benefits. Insurance pre-authorization check before running test.

---

## Getting Help

- **Code questions**: Ask your mentor or team lead
- **V1 API issues**: Contact V1 team via Slack #v1-integration
- **Database questions**: DBA team #database-support
- **Deployment issues**: DevOps #deployments
- **Urgent production issues**: Page on-call via PagerDuty

**Remember:** There are no stupid questions. This is a complex system with lots of moving parts. We'd rather you ask than make incorrect assumptions!

---

**Document Version:** 1.0  
**Last Updated:** May 6, 2026  
**Maintained By:** Engineering Team Lead  
**Feedback:** Send improvements to #engineering-docs
