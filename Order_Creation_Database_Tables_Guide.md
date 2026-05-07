# Complete Database Tables Reference: V2 Order Creation

**Quick Answer:** When you create a new order in V2, data is written to **PostgreSQL (cep_order_management)** and **READ from SQL Server (CustomerManagement & MGL)**.

---

## Schema Overview

```
┌────────────────────────────────────────────────────────────────┐
│  PostgreSQL Database: cep_order_management                     │
│  Schema: cep_core (application data)                           │
│  Schema: cep_core_audit (audit history - Envers)               │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  THIS IS WHERE V2 WRITES ALL ORDER DATA                        │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  SQL Server Database: CustomerManagement                       │
│  Schema: dbo                                                   │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  READ-ONLY: Physician & Institution master data                │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  SQL Server Database: MGL                                      │
│  Schema: dbo                                                   │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  │
│  READ-ONLY: Gene catalog, test definitions, ICD descriptions   │
└────────────────────────────────────────────────────────────────┘
```

---

## Complete Table Usage Map

### Phase 1: REQUEST ARRIVES (No DB operations yet)

```
POST /api/v1/orders
{
  "testCode": "1300",
  "institutionId": 4521,
  "authorizationProviderId": 1234,
  "patient": { ... },
  "payment": { ... },
  "icdCodes": [ ... ],
  ...
}
```

**Validation happens in memory - no DB queries yet**

---

### Phase 2: AUTHORIZATION & VALIDATION

#### Tables READ (SQL Server - CustomerManagement)

```sql
-- Check if physician exists and is active
SELECT * FROM dbo.Customers 
WHERE Id = 1234 
  AND Discriminator = 'Physician'
  AND Status = 1;  -- Active

-- Check if institution exists  
SELECT * FROM dbo.Customers
WHERE Id = 4521
  AND Discriminator = 'Institution'
  AND Status = 1;

-- Verify institution has this test
SELECT * FROM dbo.CustomerTests
WHERE CustomerId = 4521
  AND TestCode = '1300';
```

**Package:** `com.baylor.cepcommon.entity.datasource.customerManagement`  
**Entity Classes:** `Physician.java`, `Institution.java`, `CustomerTest.java`

---

### Phase 3: CORE ORDER CREATION

#### 3.1 Generate Snowflake Order ID

```java
// No DB operation - generated in application code
String orderId = snowflakeGenerator.generateOrderId();
// Result: "BG-2900-5K8P2A"
```

#### 3.2 Save Order Entity

**Table WRITTEN:** `cep_core.order`

```sql
INSERT INTO cep_core."order" (
    order_id,              -- 'BG-2900-5K8P2A' (Snowflake ID)
    patient_id,            -- FK to patient table (set after patient creation)
    status,                -- 'DRAFT' or 'SUBMIT' or 'VOB_DRAFT' or 'VOB_SENT'
    test_code,             -- 1300 (stored as INTEGER)
    institute_id,          -- 4521
    hospital_code,         -- 'BMGL' (denormalized from institution)
    payment_id,            -- FK to payment table (set after payment creation)
    primary_contact,       -- Physician customer ID (nullable)
    authorizing_provider,  -- 1234 (Physician customer ID)
    genetic_counselor,     -- Physician customer ID (nullable)
    clinical_note,         -- Free text (max 2000 chars)
    is_opt_in_testing,     -- boolean (derived from RNASeq consent)
    if_family_history_of_disease,  -- 'YES', 'NO', or 'UNKNOWN'
    add_on_tests,          -- JSONB array ["2055", "6573"]
    source,                -- 'OOPV2' (Online Order Portal V2)
    source_order_id,       -- For reanalysis orders only
    client_test_code,      -- Institution-specific test code override
    is_patient_discharged, -- boolean
    order_submission_date, -- Timestamp (when status = SUBMIT)
    created_by,            -- User email from JWT
    created_at,            -- Current timestamp
    updated_by,            -- User email from JWT
    updated_at             -- Current timestamp
) VALUES (...);

-- Returns auto-generated id (BIGSERIAL): 98765
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.Order`

---

### Phase 4: PHYSICIAN ASSOCIATIONS

#### 4.1 Save Order-Physician Links

**Table WRITTEN:** `cep_core.order_physician`

```sql
-- This is a junction table with ROLE information

-- Authorizing Provider
INSERT INTO cep_core.order_physician (
    order_id,      -- 98765 (database ID, not Snowflake ID)
    physician_id,  -- 1234 (Customer.Id from CustomerManagement)
    role,          -- 'AUTHORIZING_PROVIDER'
    created_at,
    updated_at
) VALUES (...);

-- Primary Contact (if provided)
INSERT INTO cep_core.order_physician (
    order_id,      -- 98765
    physician_id,  -- 5678
    role,          -- 'PRIMARY_CONTACT'
    created_at,
    updated_at
) VALUES (...);

-- Genetic Counselor (if provided)
INSERT INTO cep_core.order_physician (
    order_id,      -- 98765
    physician_id,  -- 9012
    role,          -- 'GENETIC_COUNSELOR'
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderPhysician`

**Roles Enum:**
- `AUTHORIZING_PROVIDER` (required)
- `PRIMARY_CONTACT` (optional)
- `GENETIC_COUNSELOR` (optional)

---

### Phase 5: PATIENT CREATION/RESOLUTION

#### 5.1 Check if Patient Exists

**Table READ:** `cep_core.patient`

```sql
-- Try to find existing patient by MRN
SELECT * FROM cep_core.patient
WHERE medical_record_number = 'MRN-12345'
  AND institute_id = 4521;

-- If not found by MRN, try by demographics
SELECT * FROM cep_core.patient
WHERE first_name = 'Jane'
  AND last_name = 'Doe'
  AND date_of_birth = '1985-03-15'
LIMIT 1;
```

#### 5.2 Create or Update Patient

**Table WRITTEN:** `cep_core.patient`

```sql
-- If patient doesn't exist:
INSERT INTO cep_core.patient (
    first_name,              -- 'Jane'
    middle_name,             -- nullable
    preferred_name,          -- nullable
    last_name,               -- 'Doe'
    date_of_birth,           -- '1985-03-15' (DATE type)
    sex_assigned_at_birth,   -- 'FEMALE', 'MALE', 'INTERSEX', 'UNKNOWN'
    medical_record_number,   -- 'MRN-12345'
    ethnicity,               -- JSONB array ["CAUCASIAN", "ASHKENAZI_JEWISH"]
    preferred_language,      -- 'ENGLISH'
    institute_id,            -- 4521
    created_by,
    created_at,
    updated_by,
    updated_at
) VALUES (...);

-- Returns patient.id: 11111

-- Then UPDATE order with patient_id
UPDATE cep_core."order" 
SET patient_id = 11111
WHERE id = 98765;
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.Patient`

#### 5.3 Patient Contact (if provided)

**Table WRITTEN:** `cep_core.patient_contact`

```sql
INSERT INTO cep_core.patient_contact (
    patient_id,              -- 11111
    relationship,            -- 'LEGAL_GUARDIAN', 'SELF', 'PARENT', 'SPOUSE', etc.
    first_name,
    last_name,
    email,
    phone,
    address_line_1,
    address_line_2,
    city,
    state,
    zip_code,
    country,
    preferred_language,
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.PatientContact`

---

### Phase 6: RELATIVE PATIENTS (Family Testing)

#### 6.1 For Each Relative

**Table WRITTEN:** `cep_core.patient` (one row per relative)

```sql
-- Mother
INSERT INTO cep_core.patient (
    first_name,              -- 'Mary'
    last_name,               -- 'Doe'
    date_of_birth,           -- '1960-05-10'
    sex_assigned_at_birth,   -- 'FEMALE'
    medical_record_number,   -- nullable or separate MRN
    ethnicity,
    institute_id,            -- Same as proband: 4521
    created_by,
    created_at,
    updated_by,
    updated_at
) VALUES (...);
-- Returns patient.id: 11112

-- Father  
INSERT INTO cep_core.patient (...);
-- Returns patient.id: 11113
```

#### 6.2 Link Relatives to Order

**Table WRITTEN:** `cep_core.patient_relative`

```sql
-- Link mother to proband
INSERT INTO cep_core.patient_relative (
    patient_id,              -- 11111 (proband)
    relative_patient_id,     -- 11112 (mother)
    relationship,            -- 'MOTHER'
    is_symptomatic,          -- boolean
    symptomatic_details,     -- text (if symptomatic)
    created_at,
    updated_at
) VALUES (...);

-- Link father to proband
INSERT INTO cep_core.patient_relative (
    patient_id,              -- 11111 (proband)
    relative_patient_id,     -- 11113 (father)
    relationship,            -- 'FATHER'
    is_symptomatic,
    symptomatic_details,
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.PatientRelative`

---

### Phase 7: ADDITIONAL CARE TEAM MEMBERS

#### 7.1 Create Care Team Members

**Table WRITTEN:** `cep_core.care_team_member`

```sql
-- For each additional member
INSERT INTO cep_core.care_team_member (
    is_physician,            -- true/false
    physician_id,            -- If physician: Customer.Id from CustomerManagement
    first_name,              -- If non-physician: actual name
    last_name,
    email,
    notification_topics,     -- JSONB array ["REPORT_READY", "SAMPLE_RECEIVED"]
    created_at,
    updated_at
) VALUES (...);
-- Returns care_team_member.id: 22221
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.CareTeamMember`

#### 7.2 Link to Order

**Table WRITTEN:** `cep_core.order_care_team_member`

```sql
INSERT INTO cep_core.order_care_team_member (
    order_id,                -- 98765
    care_team_member_id,     -- 22221
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderCareTeamMember`

---

### Phase 8: PAYMENT INFORMATION

#### 8.1 Create Payment Entity

**Table WRITTEN:** `cep_core.payment`

```sql
INSERT INTO cep_core.payment (
    method,                  -- 'INSURANCE', 'PATIENT_SELF_PAY', 'CLIENT_BILLING', 'DO_NOT_BILL'
    billing_method,          -- Secondary billing override (nullable)
    is_patient_cost_awareness, -- boolean
    insurance_type,          -- 'COMMERCIAL', 'MEDICAID', 'MEDICARE' (nullable)
    created_at,
    updated_at
) VALUES (...);
-- Returns payment.id: 33331

-- Update order with payment_id
UPDATE cep_core."order"
SET payment_id = 33331
WHERE id = 98765;
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.Payment`

#### 8.2 Create Payment Details (Insurances)

**Table WRITTEN:** `cep_core.payment_info_details`

```sql
-- Primary insurance
INSERT INTO cep_core.payment_info_details (
    payment_id,              -- 33331
    priority,                -- 'PRIMARY', 'SECONDARY', 'TERTIARY'
    insurance_company_name,  -- 'Blue Cross Blue Shield'
    insurance_company_code,  -- 'BCBS'
    policy_number,           -- 'POL-123456'
    group_number,
    subscriber_first_name,
    subscriber_last_name,
    subscriber_dob,
    subscriber_relationship, -- 'SELF', 'PARENT', 'SPOUSE', etc.
    insurance_phone,
    insurance_address_line_1,
    insurance_city,
    insurance_state,
    insurance_zip_code,
    created_at,
    updated_at
) VALUES (...);

-- Secondary insurance (if provided)
INSERT INTO cep_core.payment_info_details (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.PaymentInfoDetails`

---

### Phase 9: CLINICAL INFORMATION

#### 9.1 ICD-10 Diagnosis Codes

**Tables:**
1. **READ from MGL:** Get ICD code descriptions

```sql
-- SQL Server - MGL database
SELECT ICD10Code, Description 
FROM dbo.ICD10Codes
WHERE ICD10Code = 'F84.0';
-- Returns: 'Autism spectrum disorder'
```

2. **WRITE to cep_order_management:**

```sql
INSERT INTO cep_core.order_icd10 (
    order_id,                -- 98765
    icd10_code,              -- 'F84.0'
    description,             -- 'Autism spectrum disorder' (from MGL lookup)
    created_at,
    updated_at
) VALUES (...);

-- One row per ICD code
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderIcd10`

#### 9.2 HPO Terms (Human Phenotype Ontology)

**Table WRITTEN:** `cep_core.order_hpo`

```sql
INSERT INTO cep_core.order_hpo (
    order_id,                -- 98765
    hpo_id,                  -- 'HP:0001249'
    name,                    -- 'Intellectual disability'
    created_at,
    updated_at
) VALUES (...);

-- One row per HPO term
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderHpo`

#### 9.3 Genes of Interest

**Tables:**
1. **READ from MGL:** Get gene details

```sql
-- SQL Server - MGL database
SELECT GeneId, GeneSymbol, GeneName
FROM dbo.HgncGenes
WHERE GeneId = 1001;
-- Returns: { GeneId: 1001, GeneSymbol: 'BRCA1', GeneName: 'BRCA1 DNA repair associated' }
```

2. **WRITE to cep_order_management:**

```sql
INSERT INTO cep_core.order_gene (
    order_id,                -- 98765
    gene_id,                 -- 1001 (from MGL)
    gene_symbol,             -- 'BRCA1' (from MGL)
    gene_name,               -- 'BRCA1 DNA repair associated' (from MGL)
    created_at,
    updated_at
) VALUES (...);

-- One row per gene
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderGene`

#### 9.4 Indications for Testing

**Table WRITTEN:** `cep_core.order_indication_for_testing`

```sql
INSERT INTO cep_core.order_indication_for_testing (
    order_id,                -- 98765
    indication_name,         -- 'Family history of breast cancer'
    indication_category,     -- 'FAMILY_HISTORY', 'CLINICAL_DIAGNOSIS', 'SCREENING'
    indication_id,           -- External reference ID (nullable)
    created_at,
    updated_at
) VALUES (...);

-- One row per indication
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderIndicationForTesting`

---

### Phase 10: FAMILY HISTORY

**Table WRITTEN:** `cep_core.family_history`

```sql
-- One row per family member with relevant history
INSERT INTO cep_core.family_history (
    order_id,                -- 98765
    relationship,            -- 'MOTHER', 'FATHER', 'MATERNAL_GRANDMOTHER', etc.
    details,                 -- 'Breast cancer at age 45'
    created_at,
    updated_at
) VALUES (...);

-- Maximum 5 entries per order (business rule)
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.FamilyHistory`

**Allowed Relationships:**
- MOTHER, FATHER
- MATERNAL_GRANDMOTHER, MATERNAL_GRANDFATHER
- PATERNAL_GRANDMOTHER, PATERNAL_GRANDFATHER
- SIBLING, MATERNAL_AUNT, MATERNAL_UNCLE
- PATERNAL_AUNT, PATERNAL_UNCLE
- OTHER_RELATIVE

---

### Phase 11: CONSENTS

**Table WRITTEN:** `cep_core.order_consent`

```sql
INSERT INTO cep_core.order_consent (
    order_id,                -- 98765
    consent_type,            -- 'GENETIC_TESTING', 'ACMG_SECONDARY_FINDINGS', 'NY_STATE', etc.
    is_signed,               -- boolean
    consent_value,           -- 'signed', 'declined', 'yes', 'no'
    patient_index,           -- For family testing: which relative (nullable)
    test_index,              -- For multi-test orders (nullable)
    created_at,
    updated_at
) VALUES (...);

-- Multiple rows for expanded consents
-- Example: NY State consent creates one row per test
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderConsent`

**Common Consent Types:**
- `GENETIC_TESTING` (required)
- `ACMG_SECONDARY_FINDINGS` (optional)
- `NY_STATE` (required for NY residents)
- `RESEARCH_PARTICIPATION` (optional)
- `DATA_SHARING` (optional)
- `RNA_SEQ_OPT_IN` (for specific tests)

---

### Phase 12: SAMPLES

#### 12.1 Create Sample Entities

**Table WRITTEN:** `cep_core.sample`

```sql
-- Proband sample
INSERT INTO cep_core.sample (
    patient_id,              -- 11111 (proband)
    sample_type,             -- 'BLOOD', 'SALIVA', 'AMNIOTIC_FLUID', etc.
    status,                  -- 'COLLECTED', 'TO_BE_COLLECTED', 'SEND_A_KIT'
    is_relative_tested,      -- boolean (false for proband)
    collection_date,         -- DATE (nullable if not collected yet)
    quantity,                -- DECIMAL (ml or mg)
    kit_type,                -- If status = SEND_A_KIT
    lab_number,              -- V1 lab number (nullable until V1 assigns)
    cc_volume,               -- For CVS/amniotic fluid (ml)
    mg_quantity,             -- For CVS/amniotic fluid (mg)
    is_ta,                   -- Transabdominal (prenatal)
    is_tc,                   -- Transcervical (prenatal)
    sample_description,      -- Free text
    created_at,
    updated_at
) VALUES (...);
-- Returns sample.id: 44441

-- Mother sample (if family testing)
INSERT INTO cep_core.sample (
    patient_id,              -- 11112 (mother)
    sample_type,             -- 'BLOOD'
    status,                  -- 'TO_BE_COLLECTED'
    is_relative_tested,      -- true
    ...
) VALUES (...);
-- Returns sample.id: 44442

-- Father sample (if family testing)
INSERT INTO cep_core.sample (...);
-- Returns sample.id: 44443
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.Sample`

**Sample Types:**
- BLOOD, SALIVA, BUCCAL_SWAB
- AMNIOTIC_FLUID, CVS (Chorionic Villus Sample)
- TISSUE, BONE_MARROW
- DNA, RNA

**Sample Status:**
- `COLLECTED` - Already collected at time of order
- `TO_BE_COLLECTED` - Will be collected at institution
- `SEND_A_KIT` - Kit to be mailed to patient

#### 12.2 Link Samples to Order

**Table WRITTEN:** `cep_core.order_sample`

```sql
-- Link proband sample
INSERT INTO cep_core.order_sample (
    order_id,                -- 98765
    sample_id,               -- 44441
    created_at,
    updated_at
) VALUES (...);

-- Link mother sample
INSERT INTO cep_core.order_sample (
    order_id,                -- 98765
    sample_id,               -- 44442
    created_at,
    updated_at
) VALUES (...);

-- Link father sample
INSERT INTO cep_core.order_sample (
    order_id,                -- 98765
    sample_id,               -- 44443
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderSample`

#### 12.3 Sample Delivery (if SEND_A_KIT)

**Table WRITTEN:** `cep_core.sample_delivery`

```sql
INSERT INTO cep_core.sample_delivery (
    sample_id,               -- 44441
    delivery_type,           -- 'SHIPPING', 'PICKUP'
    recipient_first_name,
    recipient_last_name,
    address_line_1,
    address_line_2,
    city,
    state,
    zip_code,
    country,
    phone,
    email,
    special_instructions,
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.SampleDelivery`

---

### Phase 13: DOCUMENTS (SUBMIT status only)

#### 13.1 Convert HTML to PDF

```java
// Application code - no DB operation
byte[] pdfBytes = pdfService.convertHtmlToPdf(orderSummaryHtml);
```

#### 13.2 Upload to V1 Document Service

```java
// External HTTP call - not a DB operation
DocumentUploadResponse response = 
    v1DocumentServiceClient.upload(pdfBytes, "order-summary.pdf");
// Returns: { documentId: "DOC-123456", url: "..." }
```

#### 13.3 Save Document Reference

**Table WRITTEN:** `cep_core.order_document`

```sql
INSERT INTO cep_core.order_document (
    order_id,                -- 98765
    document_id,             -- 'DOC-123456' (from V1 document service)
    document_type,           -- 'ORDER_SUMMARY', 'INSURANCE_CARD', 'CLINICAL_NOTE', etc.
    file_name,               -- 'order-summary.pdf'
    upload_date,             -- Current timestamp
    created_at,
    updated_at
) VALUES (...);

-- One row per document
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderDocument`

#### 13.4 Additional Clinical Documents (if provided)

```sql
-- Insurance card front
INSERT INTO cep_core.order_document (
    order_id,                -- 98765
    document_id,             -- 'DOC-123457'
    document_type,           -- 'INSURANCE_CARD_FRONT'
    file_name,               -- 'bcbs-card-front.jpg'
    ...
) VALUES (...);

-- Insurance card back
INSERT INTO cep_core.order_document (
    order_id,                -- 98765
    document_id,             -- 'DOC-123458'
    document_type,           -- 'INSURANCE_CARD_BACK'
    ...
) VALUES (...);
```

#### 13.5 Payment Documents Link

**Table WRITTEN:** `cep_core.payment_document`

```sql
-- Link insurance documents to payment
INSERT INTO cep_core.payment_document (
    payment_id,              -- 33331
    order_document_id,       -- ID from order_document table
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.PaymentDocument`

---

### Phase 14: PRENATAL-SPECIFIC DATA (If Applicable)

#### 14.1 Gestation Information

**Table WRITTEN:** `cep_core.gestation`

```sql
INSERT INTO cep_core.gestation (
    order_id,                -- 98765
    gestational_age_weeks,   -- INTEGER
    gestational_age_days,    -- INTEGER (0-6)
    estimated_due_date,      -- DATE
    last_menstrual_period,   -- DATE
    conception_method,       -- 'NATURAL', 'IVF', 'IUI', etc.
    is_multiple_gestation,   -- boolean
    number_of_fetuses,       -- INTEGER (1, 2, 3...)
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.Gestation`

#### 14.2 Fetus Information

**Table WRITTEN:** `cep_core.patient_order_fetus_info`

```sql
-- One row per fetus
INSERT INTO cep_core.patient_order_fetus_info (
    order_id,                -- 98765
    patient_id,              -- 11111 (mother/carrier patient)
    fetus_number,            -- 1, 2, 3... (for twins/triplets)
    fetal_sex,               -- 'MALE', 'FEMALE', 'UNKNOWN'
    fetal_sex_determination_method, -- 'ULTRASOUND', 'NIPT', 'AMNIOCENTESIS'
    is_affected,             -- boolean
    affected_description,    -- Text if is_affected = true
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.PatientOrderFetusInfo`

#### 14.3 Prenatal Imaging

**Table WRITTEN:** `cep_core.prenatal_imaging`

```sql
INSERT INTO cep_core.prenatal_imaging (
    order_id,                -- 98765
    imaging_type,            -- 'ULTRASOUND', 'MRI'
    findings,                -- 'Normal fetal anatomy' or 'Ventriculomegaly detected'
    imaging_date,            -- DATE
    performed_at_weeks,      -- INTEGER
    performed_at_days,       -- INTEGER
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.PrenatalImaging`

#### 14.4 Prenatal Testing Status

**Table WRITTEN:** `cep_core.prenatal_testing_status`

```sql
INSERT INTO cep_core.prenatal_testing_status (
    order_id,                -- 98765
    test_type,               -- 'NIPT', 'FIRST_TRIMESTER_SCREEN', 'AFP', 'QUAD_SCREEN'
    result,                  -- 'NORMAL', 'ABNORMAL', 'INCONCLUSIVE'
    result_details,          -- Free text description
    test_date,               -- DATE
    performed_at_weeks,      -- INTEGER
    performed_at_days,       -- INTEGER
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.PrenatalTestingStatus`

---

### Phase 15: KNOWN FAMILY VARIANT (KFV) DATA

#### 15.1 Family Variant Information

**Table WRITTEN:** `cep_core.patient_order_family_variant`

```sql
INSERT INTO cep_core.patient_order_family_variant (
    order_id,                -- 98765
    has_known_family_variant, -- boolean
    variant_details,         -- Free text
    gene_name,               -- 'BRCA1'
    variant_notation,        -- 'c.68_69delAG'
    zygosity,                -- 'HETEROZYGOUS', 'HOMOZYGOUS'
    inheritance_pattern,     -- 'AUTOSOMAL_DOMINANT', 'AUTOSOMAL_RECESSIVE', etc.
    affected_relative,       -- 'MOTHER', 'FATHER', etc.
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.PatientOrderFamilyVariant`

---

### Phase 16: VOB (Verification of Benefits) DATA

**Table WRITTEN:** `cep_core.order_vob`

```sql
INSERT INTO cep_core.order_vob (
    order_id,                -- 98765
    vob_status,              -- 'DRAFT', 'SENT', 'APPROVED', 'DENIED'
    vob_requestor_id,        -- Physician customer ID (from CustomerManagement)
    additional_vob_test_codes, -- JSONB array of test codes
    vob_sent_date,           -- Timestamp (when status = SENT)
    vob_response_date,       -- Timestamp (when approved/denied)
    vob_notes,               -- Free text
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderVob`

---

### Phase 17: AUDIT TRAIL (Automatic via Envers)

**Schema:** `cep_core_audit`

Hibernate Envers automatically creates audit tables for every entity:

```sql
-- Automatically created on INSERT to order table
INSERT INTO cep_core_audit.order_aud (
    id,                      -- Same as order.id: 98765
    rev,                     -- Revision number
    revtype,                 -- 0 = INSERT, 1 = UPDATE, 2 = DELETE
    order_id,
    patient_id,
    status,
    test_code,
    -- ... all other order fields ...
    created_at,
    updated_at
) VALUES (...);

-- Similar audit inserts for:
-- patient_aud, sample_aud, payment_aud, order_icd10_aud, etc.
```

**Audit tables created:**
- `order_aud`
- `patient_aud`
- `sample_aud`
- `payment_aud`
- `order_icd10_aud`
- `order_hpo_aud`
- `order_gene_aud`
- ... (one for each main table)

**Revision Info Table:**

```sql
INSERT INTO cep_core_audit.revinfo (
    rev,                     -- Auto-increment revision ID
    revtstmp,                -- Timestamp of change
    user_id,                 -- Who made the change (from JWT)
    ip_address               -- Request IP
) VALUES (...);
```

---

### Phase 18: V1 SYNC TRACKING

**Table WRITTEN:** `cep_core.order_sync_log`

```sql
INSERT INTO cep_core.order_sync_log (
    trace_id,                -- UUID for tracking
    order_id,                -- 'BG-2900-5K8P2A' (V2 Snowflake ID)
    v1_order_id,             -- V1's integer ID (null initially, filled after sync)
    sync_operation,          -- 'INIT_CREATE'
    sync_status,             -- 'PENDING'
    initiated_at,            -- Current timestamp
    last_attempted_at,       -- null
    completed_at,            -- null
    v2_payload,              -- JSONB - full CreateOrderRequest
    v1_payload,              -- JSONB - V1CreateOrderRequestDto (filled before POST)
    v1_response,             -- TEXT - V1 API response (filled after POST)
    error_code,              -- VARCHAR(50) - if failed
    error_message,           -- TEXT - if failed
    error_stacktrace,        -- TEXT - if failed
    http_status_code,        -- INTEGER - HTTP status from V1
    auth_token,              -- TEXT - JWT used for V1 call
    institution_code         -- VARCHAR(50) - 'BMGL'
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.OrderSyncLog`

This row will be updated multiple times during the sync process:

```sql
-- Before V1 POST
UPDATE order_sync_log
SET sync_operation = 'CREATE',
    sync_status = 'IN_PROGRESS',
    v1_payload = '{...}',
    last_attempted_at = NOW()
WHERE order_id = 'BG-2900-5K8P2A';

-- After successful V1 POST
UPDATE order_sync_log
SET sync_status = 'SUCCESS',
    v1_order_id = '12345',
    v1_response = '{...}',
    completed_at = NOW(),
    http_status_code = 201
WHERE order_id = 'BG-2900-5K8P2A';

-- If SUBMIT status, additional sync operation logged
INSERT INTO cep_core.order_sync_log (
    order_id,                -- Same order
    sync_operation,          -- 'APPROVE'
    sync_status,             -- 'SUCCESS'
    ...
) VALUES (...);
```

---

### Phase 19: V1-V2 ID MAPPING

**Table WRITTEN:** `cep_core.legacy_order`

```sql
INSERT INTO cep_core.legacy_order (
    order_id,                -- 'BG-2900-5K8P2A' (V2 Snowflake ID)
    legacy_order_id,         -- 12345 (V1 integer ID from V1 API response)
    created_at,
    updated_at
) VALUES (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.LegacyOrder`

**Purpose:** Quick lookup to find "What's the V1 ID for this V2 order?" Used during PATCH operations for V1 sync.

---

### Phase 20: SAMPLE ID MAPPINGS

**Table WRITTEN:** `cep_core.id_mapping_log`

```sql
-- Map proband sample
INSERT INTO cep_core.id_mapping_log (
    v2_order_id,             -- 'BG-2900-5K8P2A'
    v1_order_id,             -- '12345'
    v2_sample_id,            -- 44441 (V2 sample.id)
    v1_sample_id,            -- 55555 (V1 sample ID from V1 response)
    entity_type,             -- 'SAMPLE'
    mapping_source,          -- 'V2_SYNC'
    created_at
) VALUES (...);

-- Map mother sample
INSERT INTO cep_core.id_mapping_log (...);

-- Map father sample
INSERT INTO cep_core.id_mapping_log (...);
```

**Entity:** `com.baylor.cepcommon.entity.datasource.cepOrderManagement.IdMappingLog`

**Purpose:** 
- Track which V2 samples correspond to which V1 samples
- Critical for reanalysis orders (need to reference original V1 sample)
- Used by migration service for historical data

---

## Complete Table Summary

### Tables WRITTEN (PostgreSQL - cep_order_management)

#### Core Order Tables (Always)
1. `cep_core.order` ✓
2. `cep_core.order_physician` ✓ (1-3 rows)
3. `cep_core.patient` ✓
4. `cep_core.sample` ✓ (1+ rows)
5. `cep_core.order_sample` ✓ (junction, 1+ rows)

#### Clinical Data (If Provided)
6. `cep_core.order_icd10` (0+ rows)
7. `cep_core.order_hpo` (0+ rows)
8. `cep_core.order_gene` (0+ rows)
9. `cep_core.order_indication_for_testing` (0+ rows)
10. `cep_core.order_consent` (0+ rows)
11. `cep_core.family_history` (0-5 rows)

#### Payment (If Provided)
12. `cep_core.payment` ✓
13. `cep_core.payment_info_details` (1-3 rows for insurances)
14. `cep_core.payment_document` (0+ rows)

#### Care Team (If Provided)
15. `cep_core.care_team_member` (0+ rows)
16. `cep_core.order_care_team_member` (junction, 0+ rows)

#### Documents (SUBMIT only)
17. `cep_core.order_document` (1+ rows)

#### Patient Relations (If Family Testing)
18. `cep_core.patient_relative` (0+ rows)
19. `cep_core.patient_contact` (0-1 rows)

#### Prenatal (If Prenatal Order)
20. `cep_core.gestation` (0-1 rows)
21. `cep_core.patient_order_fetus_info` (0+ rows)
22. `cep_core.prenatal_imaging` (0+ rows)
23. `cep_core.prenatal_testing_status` (0+ rows)

#### Known Family Variant (If Applicable)
24. `cep_core.patient_order_family_variant` (0-1 rows)

#### VOB (If VOB Order)
25. `cep_core.order_vob` (0-1 rows)

#### Sample Delivery (If SEND_A_KIT)
26. `cep_core.sample_delivery` (0+ rows)

#### Sync & Audit (Always)
27. `cep_core.order_sync_log` ✓ (1+ rows)
28. `cep_core.legacy_order` ✓ (1 row, after V1 sync succeeds)
29. `cep_core.id_mapping_log` ✓ (1+ rows, after V1 sync succeeds)
30. `cep_core_audit.order_aud` ✓ (automatic)
31. `cep_core_audit.patient_aud` ✓ (automatic)
32. `cep_core_audit.sample_aud` ✓ (automatic)
33. ... (audit table for each entity)

---

### Tables READ (SQL Server - CustomerManagement)

1. `dbo.Customers` - Physicians & Institutions
2. `dbo.CustomerTests` - Test availability per institution

---

### Tables READ (SQL Server - MGL)

1. `dbo.ICD10Codes` - ICD code descriptions
2. `dbo.HgncGenes` - Gene information
3. `dbo.Tests` - Test catalog (if needed)

---

## Query Example: Find All Data for an Order

```sql
-- Main order
SELECT * FROM cep_core."order" WHERE order_id = 'BG-2900-5K8P2A';

-- Patient
SELECT p.* FROM cep_core.patient p
JOIN cep_core."order" o ON o.patient_id = p.id
WHERE o.order_id = 'BG-2900-5K8P2A';

-- Physicians
SELECT op.role, c.FirstName, c.LastName, c.NPI
FROM cep_core.order_physician op
JOIN cep_core."order" o ON o.id = op.order_id
LEFT JOIN CustomerManagement.dbo.Customers c ON c.Id = op.physician_id
WHERE o.order_id = 'BG-2900-5K8P2A';

-- Samples
SELECT s.* FROM cep_core.sample s
JOIN cep_core.order_sample os ON os.sample_id = s.id
JOIN cep_core."order" o ON o.id = os.order_id
WHERE o.order_id = 'BG-2900-5K8P2A';

-- Clinical data
SELECT * FROM cep_core.order_icd10 
WHERE order_id = (SELECT id FROM cep_core."order" WHERE order_id = 'BG-2900-5K8P2A');

SELECT * FROM cep_core.order_hpo
WHERE order_id = (SELECT id FROM cep_core."order" WHERE order_id = 'BG-2900-5K8P2A');

SELECT * FROM cep_core.order_gene
WHERE order_id = (SELECT id FROM cep_core."order" WHERE order_id = 'BG-2900-5K8P2A');

-- Payment
SELECT p.*, pid.* FROM cep_core.payment p
LEFT JOIN cep_core.payment_info_details pid ON pid.payment_id = p.id
WHERE p.id = (SELECT payment_id FROM cep_core."order" WHERE order_id = 'BG-2900-5K8P2A');

-- V1 mapping
SELECT * FROM cep_core.legacy_order WHERE order_id = 'BG-2900-5K8P2A';

-- Sync history
SELECT * FROM cep_core.order_sync_log 
WHERE order_id = 'BG-2900-5K8P2A'
ORDER BY initiated_at DESC;

-- Audit trail
SELECT * FROM cep_core_audit.order_aud
WHERE id = (SELECT id FROM cep_core."order" WHERE order_id = 'BG-2900-5K8P2A')
ORDER BY rev DESC;
```

---

## Key Takeaways

1. **Primary Database:** ALL writes go to PostgreSQL `cep_order_management`

2. **Minimum Tables for Simple Order:**
   - cep_core.order
   - cep_core.patient
   - cep_core.sample
   - cep_core.order_sample
   - cep_core.order_physician
   - cep_core.order_sync_log
   - cep_core.legacy_order (after V1 sync)

3. **Read-Only Databases:**
   - CustomerManagement: Physician/institution lookup
   - MGL: Gene/ICD code enrichment

4. **Audit Everything:** Envers creates _aud tables automatically

5. **V1 Sync Tables:** order_sync_log, legacy_order, id_mapping_log track synchronization

6. **Schema Structure:**
   - `cep_core` = application data
   - `cep_core_audit` = audit history
   - `dbo` = SQL Server default schema

