# MIRTH CONNECT ENGINEERING HANDBOOK
### Complete Healthcare Integration Reference — Mirth Connect 4.5.2

> **Audience:** Interns · Junior Engineers · Senior Engineers · Integration Architects  
> **Stack:** Mirth Connect 4.5.2 · HL7 v2 · PostgreSQL · Java/Rhino JS · FHIR concepts

---

## TABLE OF CONTENTS

1. [Healthcare System Ecosystem](#1-healthcare-system-ecosystem)
2. [Interface Engine Architecture](#2-interface-engine-architecture)
3. [Mirth Channel Architecture](#3-mirth-channel-architecture)
4. [Message Flow and Routing](#4-message-flow-and-routing)
5. [HL7 v2 Complete Reference](#5-hl7-v2-complete-reference)
6. [JavaScript and E4X in Mirth](#6-javascript-and-e4x-in-mirth)
7. [Mirth Maps — Complete Reference](#7-mirth-maps--complete-reference)
8. [Filters — Complete Reference](#8-filters--complete-reference)
9. [Transformers — Complete Reference](#9-transformers--complete-reference)
10. [Source Connectors](#10-source-connectors)
11. [Destination Connectors and Data Types](#11-destination-connectors-and-data-types)
12. [Debugging and Troubleshooting](#12-debugging-and-troubleshooting)
13. [Production Engineering](#13-production-engineering)
14. [Complete Channel Examples](#14-complete-channel-examples)
15. [Best Practices and Anti-Patterns](#15-best-practices-and-anti-patterns)
16. [Mirth Administrator Views](#16-mirth-administrator-views)
17. [Engineering Cheat Sheet](#17-engineering-cheat-sheet)

---

## 1. Healthcare System Ecosystem

### 1.1 Why Healthcare Integration Exists

Modern hospitals run dozens of specialized systems — each built by a different vendor, using a different protocol, storing data in a different format. Without integration, these systems are isolated islands. A lab result sitting in the LIS never reaches the EHR. A patient admitted in the HIS never triggers a billing event. Mirth Connect is the middleware that connects all of these systems.

```
HOSPITAL INFORMATION SYSTEM (HIS)
            |
    ┌───────┴────────┐
    ↓                ↓
   EHR / EMR        LIS (Lab)
    ↓                ↓
   RIS (Radiology)  Billing
    ↓                ↓
   Pharmacy      Analytics / DW
            |
    ┌───────┴────────┐
    ↓                ↓
  MIRTH CONNECT  (Hub)
  Routes / Transforms / Delivers
```

### 1.2 Core Clinical Systems

| System | Full Name | Purpose | Key Data |
|--------|-----------|---------|---------|
| HIS | Hospital Information System | Master hospital software — admissions, beds, departments | ADT events, patient registration |
| EHR/EMR | Electronic Health/Medical Record | Clinical patient records, doctor notes | Encounters, orders, results, medications |
| LIS | Laboratory Information System | Lab sample tracking, result management | ORM orders, ORU results, CBC/BMP panels |
| RIS | Radiology Information System | Imaging scheduling, reporting | Radiology orders, reports, DICOM metadata |
| PACS | Picture Archiving & Comm System | Stores and retrieves medical images | DICOM image files, radiology archive |
| Pharmacy | Pharmacy Management System | Medication dispensing and management | Medication orders, dispense events |
| Billing | Financial / Insurance System | Claims, coding, reimbursement | Charge events, ICD codes, insurance claims |

### 1.3 EHR vs EMR

| | EMR | EHR |
|--|-----|-----|
| Scope | One clinic/practice | Entire patient history across providers |
| Sharing | Usually stays in-house | Designed to be shared |
| Example | Small clinic local system | Epic, Cerner across hospital networks |
| Think of as | Paper chart gone digital | Lifelong health passport |

---

## 2. Interface Engine Architecture

### 2.1 What is an Interface Engine

An interface engine is middleware that sits between all hospital systems, receiving messages from any source, transforming them as needed, and routing them to correct destinations. Mirth Connect is a healthcare-specific interface engine — it understands HL7, speaks MLLP, and is designed for the reliability and audit requirements of clinical data exchange.

> **The Airport Control Tower Analogy:** Think of Mirth as an airport control tower. Aircraft (messages) do not talk to each other — they all communicate through the tower. The tower (Mirth) knows where every aircraft is going, what format it needs, and routes it safely.

### 2.2 Point-to-Point vs Hub-and-Spoke

```
Point-to-Point (5 systems = 20 connections):
HIS ←→ EHR ←→ LIS ←→ RIS ←→ Billing
 ↕       ↕      ↕      ↕
(every system connects to every other — spaghetti)

Mirth Hub-and-Spoke (5 systems = 5 connections):
HIS ──┐
EHR ──┤
LIS ──┼──→ MIRTH CONNECT ──→ routes to all
RIS ──┤
Billing─┘
```

| Approach | Connections for N systems | Add 1 system | Maintenance |
|----------|--------------------------|--------------|-------------|
| Point-to-point | N × (N-1) | N-1 new connections | Nightmare |
| Hub-and-spoke (Mirth) | N | 1 new connection | Centralized |

---

## 3. Mirth Channel Architecture

### 3.1 What is a Channel

A channel is a complete, self-contained integration pipeline. It defines exactly one integration use-case — one source, one set of transformations, and one or more destinations.

```
┌─────────────────────────────────────────────────────┐
│                  MIRTH CHANNEL                      │
│                                                     │
│  External System                                    │
│       ↓                                             │
│  SOURCE CONNECTOR   ← TCP / File / HTTP / DB / JS   │
│       ↓                                             │
│  MESSAGE QUEUE      ← persisted to DB immediately   │
│       ↓                                             │
│  SOURCE FILTER      ← true=continue false=FILTERED  │
│       ↓                                             │
│  SOURCE TRANSFORMER ← extract / map / convert       │
│       ↓                                             │
│  ┌────┴────┐  ┌────┴────┐  ┌────┴────┐            │
│  │ DEST 1  │  │ DEST 2  │  │ DEST 3  │            │
│  │ Filter  │  │ Filter  │  │ Filter  │            │
│  │ Transf. │  │ Transf. │  │ Transf. │            │
│  │ Queue   │  │ Queue   │  │ Queue   │            │
│  │ Deliver │  │ Deliver │  │ Deliver │            │
└──┴─────────┴──┴─────────┴──┴─────────┴────────────┘
```

### 3.2 Channel Lifecycle States

| State | Processes Messages | Description | Action to reach |
|-------|-------------------|-------------|-----------------|
| Undeployed | No | Config saved to DB — server not aware | Save channel |
| Deployed + Stopped | No | Loaded into memory — not running | Right-click → Deploy |
| Started | Yes | Source connector listening — messages flowing | Right-click → Start |
| Paused | Queue only | Accepts into queue, no delivery | Right-click → Pause |

> ⚠️ **CRITICAL:** Most common mistake — channel built and saved but never Deployed + Started. Always: **Save → Deploy → Start**

### 3.3 Message States

| Status | Meaning | Action |
|--------|---------|--------|
| RECEIVED | Raw message arrived, queued to DB | Normal transient |
| QUEUED | Stored, awaiting processing or retry | Investigate if persists |
| PROCESSING | Currently being transformed | Normal transient |
| SENT | All destinations delivered | Nothing to do |
| FILTERED | Filter returned false | Check filter logic |
| ERROR | Exception thrown or delivery failed | Message Browser → Error tab |
| PENDING | Waiting for destination to recover | Fix destination |

---

## 4. Message Flow and Routing

### 4.1 Complete Message Journey

1. External system sends message to source connector
2. Source connector receives raw bytes
3. **Mirth writes raw message to internal database — message is now SAFE**
4. Source filter evaluates — `true` to continue, `false` to FILTER
5. Source transformer runs — extract, map, convert, enrich
6. Message fans out to all destinations in parallel
7. Each destination runs its own filter independently
8. Each destination runs its own transformer independently
9. Each destination connector delivers the message
10. For TCP/MLLP sources — **ACK sent after step 3, not step 9**
11. Full journey stored in message log permanently

### 4.2 Three Levels of Routing

#### Level 1 — Channel-Level Routing (Source Filter)

```javascript
// Source filter — ADT channel only accepts ADT messages
var msgType = msg['MSH']['MSH.9']['MSH.9.1'].toString();
return (msgType === 'ADT');
```

#### Level 2 — Destination-Level Routing (Destination Filter)

```javascript
// Destination 2 filter — only ABNORMAL or CRITICAL results
var priority = channelMap.get('priority');
return (priority === 'ABNORMAL' || priority === 'CRITICAL');
```

#### Level 3 — Channel Sender (Inter-Channel Routing)

The Channel Sender destination type forwards a message directly to another channel in memory — no TCP roundtrip, no network overhead.

### 4.3 The ACK — Why It Is Critical

```
MSH|^~\&|MIRTH|INT|LIS|LAB|20240315103046||ACK^R01|ACK001|P|2.5
MSA|AA|MSG001|Message accepted

MSA.1 = AA → Application Accept
MSA.1 = AE → Application Error
MSA.1 = AR → Application Reject
MSA.2 = echoes back original message control ID
```

> ⚠️ **PRODUCTION RULE:** Never set ACK Mode to None on a TCP Listener. Without ACK the sender retries indefinitely creating **duplicates** in every downstream system.

**Mirth sends ACK after queueing, not after delivery.** This keeps the sender's queue moving while Mirth handles delivery reliability internally.

---

## 5. HL7 v2 Complete Reference

### 5.1 Delimiter Reference

| Character | Name | Separates | Example |
|-----------|------|-----------|---------|
| `\|` | Field separator | Fields within a segment | PID.5 from PID.6 |
| `^` | Component separator | Components within a field | `DOE^JOHN^A` → last^first^middle |
| `~` | Repetition separator | Repeated field values | Multiple phone numbers |
| `\` | Escape character | Special characters | `\F\` = literal pipe |
| `&` | Subcomponent separator | Sub-parts of a component | Rarely used |

### 5.2 Master Reference HL7 Message

```
MSH|^~\&|LIS|LAB|MIRTH|INT|20240315103045||ORU^R01|MSG001|P|2.5
PID|1||MRN98765^^^HOSPITAL^MR||DOE^JOHN^A||19850322|M|||123 MAIN ST^^HOUSTON^TX^77001||5551234567
PV1|1|I|ICU^101^A|||DOC001^SMITH^JAMES|||SUR||||ADM|A|20240315090000
OBR|1|ORD789|LAB567|CBC^COMPLETE BLOOD COUNT^L|||20240315100000
OBX|1|NM|718-7^HEMOGLOBIN^LN||13.5|g/dL|12.0-17.5|N|||F
OBX|2|NM|789-8^RBC^LN||4.8|10*6/uL|4.5-5.5|N|||F
OBX|3|NM|6690-2^WBC^LN||11.2|10*3/uL|4.5-11.0|H|||F
```

### 5.3 Segment Reference

| Segment | Full Name | Key Fields | Common Use |
|---------|-----------|-----------|------------|
| MSH | Message Header | MSH.3 sending app, MSH.9 msg type, MSH.10 control ID | Every message — routing and identification |
| PID | Patient ID | PID.3 MRN, PID.5 name, PID.7 DOB, PID.8 gender | Patient demographics — most extracted segment |
| PV1 | Patient Visit | PV1.2 type, PV1.3 location, PV1.7 attending doctor | Inpatient/outpatient visit details |
| OBR | Observation Request | OBR.2 placer order, OBR.4 test code/name, OBR.7 collection time | Lab/radiology order identification |
| OBX | Observation Result | OBX.3 LOINC, OBX.5 value, OBX.6 units, OBX.8 flag | Individual result values — loops for multiple |
| ORC | Order Common | ORC.1 control, ORC.2 placer order, ORC.5 status | Order control — status updates |
| DG1 | Diagnosis | DG1.3 ICD code, DG1.4 description | Diagnosis codes for billing and clinical |
| NK1 | Next of Kin | NK1.2 name, NK1.3 relationship | Emergency contacts |

### 5.4 Common Message Types

| Code | Event | Trigger | Key Segments |
|------|-------|---------|--------------|
| ADT^A01 | Patient Admitted | Patient registered at front desk | MSH PID PV1 |
| ADT^A02 | Patient Transferred | Moved to different unit | MSH PID PV1 |
| ADT^A03 | Patient Discharged | Patient goes home | MSH PID PV1 |
| ADT^A08 | Patient Info Updated | Demographics changed | MSH PID |
| ORU^R01 | Observation Result | Lab result from LIS | MSH PID OBR OBX |
| ORM^O01 | Order Message | Doctor orders a test | MSH PID ORC OBR |
| SIU^S12 | Schedule Appointment | Appointment booked | MSH PID SCH |

### 5.5 Field Position Quick Reference

| What You Need | Segment.Field.Component | JavaScript Access |
|---------------|------------------------|-------------------|
| Message type | MSH.9.1 | `msg['MSH']['MSH.9']['MSH.9.1'].toString()` |
| Event code | MSH.9.2 | `msg['MSH']['MSH.9']['MSH.9.2'].toString()` |
| Message control ID | MSH.10.1 | `msg['MSH']['MSH.10']['MSH.10.1'].toString()` |
| Sending application | MSH.3.1 | `msg['MSH']['MSH.3']['MSH.3.1'].toString()` |
| Patient MRN | PID.3.1 | `msg['PID']['PID.3']['PID.3.1'].toString()` |
| Last name | PID.5.1 | `msg['PID']['PID.5']['PID.5.1'].toString()` |
| First name | PID.5.2 | `msg['PID']['PID.5']['PID.5.2'].toString()` |
| Date of birth | PID.7.1 | `msg['PID']['PID.7']['PID.7.1'].toString()` |
| Gender | PID.8.1 | `msg['PID']['PID.8']['PID.8.1'].toString()` |
| Test code | OBR.4.1 | `msg['OBR']['OBR.4']['OBR.4.1'].toString()` |
| Test name | OBR.4.2 | `msg['OBR']['OBR.4']['OBR.4.2'].toString()` |
| Placer order # | OBR.2.1 | `msg['OBR']['OBR.2']['OBR.2.1'].toString()` |
| Result value | OBX.5.1 | `msg['OBX'][i]['OBX.5']['OBX.5.1'].toString()` |
| Result units | OBX.6.1 | `msg['OBX'][i]['OBX.6']['OBX.6.1'].toString()` |
| Abnormal flag | OBX.8.1 | `msg['OBX'][i]['OBX.8']['OBX.8.1'].toString()` |
| Result status | OBX.11.1 | `msg['OBX'][i]['OBX.11']['OBX.11.1'].toString()` |

---

## 6. JavaScript and E4X in Mirth

### 6.1 How Mirth Parses HL7

When an HL7 message enters a channel with data type `HL7 v2.x`, Mirth parses it into an **XML tree** before your transformer runs. The `msg` variable holds this XML tree. **E4X** (ECMAScript for XML) is the syntax used to navigate it.

Mirth runs on the **Rhino JavaScript engine** — a Java-based runtime, not Node.js or browser JavaScript. This means you have direct access to Java classes.

### 6.2 The Core Variables

| Variable | Type | Scope | Purpose | Writable |
|----------|------|-------|---------|----------|
| `msg` | E4X XML | Current message | Parsed HL7 — read and modify | Yes |
| `tmp` | String | Current destination | Outbound message content | Yes |
| `channelMap` | Java Map | One message, whole channel | Pass data source → destinations | Yes |
| `connectorMap` | Java Map | One message, one destination | Destination-private scratchpad | Yes |
| `responseMap` | Java Map | One message, HTTP source | Control HTTP response to caller | Yes |
| `sourceMap` | Java Map | One message | Mirth auto-filled metadata | Read only |
| `globalChannelMap` | Java Map | All messages, this channel | State across messages | Yes |
| `globalMap` | Java Map | All messages, all channels | Cross-channel shared state | Yes |
| `configurationMap` | Java Map | Server-wide | Admin-set config — URLs, passwords | Read only |
| `logger` | Log object | Everywhere | Write to server log | N/A |

### 6.3 E4X Navigation — The Three-Level Pattern

```javascript
// Pattern: msg['SEGMENT']['SEGMENT.FIELD']['SEGMENT.FIELD.COMPONENT']

msg['PID']['PID.3']['PID.3.1']   // MRN
msg['PID']['PID.5']['PID.5.1']   // Last name
msg['MSH']['MSH.9']['MSH.9.2']   // Event code (R01, A01)

// ALWAYS call .toString() before comparing
// WRONG — comparing XML node to string (never matches)
if (msg['MSH']['MSH.9']['MSH.9.1'] === 'ORU') { }

// RIGHT
if (msg['MSH']['MSH.9']['MSH.9.1'].toString() === 'ORU') { }
```

### 6.4 Production Safe Field Extraction

```javascript
// Put this at the top of EVERY transformer — prevents NullPointerException
function get(seg, fld, comp) {
    try {
        var v = msg[seg][fld][comp];
        return (v != null) ? v.toString().trim() : '';
    } catch (e) {
        return '';
    }
}

// Safe usage
var mrn      = get('PID', 'PID.3', 'PID.3.1');
var lastName = get('PID', 'PID.5', 'PID.5.1');
var testCode = get('OBR', 'OBR.4', 'OBR.4.1');
var flag     = get('OBX', 'OBX.8', 'OBX.8.1');
```

### 6.5 HL7 Date Conversion Utility

```javascript
function hl7ToISO(d) {
    if (!d || d.length < 8) return '';
    var s = d.substring(0,4) + '-' + d.substring(4,6) + '-' + d.substring(6,8);
    if (d.length >= 14)
        s += 'T' + d.substring(8,10) + ':' + d.substring(10,12) + ':' + d.substring(12,14);
    return s;
}

var dob       = hl7ToISO(get('PID','PID.7','PID.7.1'));  // '1985-03-22'
var collected = hl7ToISO(get('OBR','OBR.7','OBR.7.1')); // '2024-03-15T10:00:00'
```

### 6.6 Looping Multiple OBX Segments

```javascript
var results  = [];
var priority = 'ROUTINE';

if (msg['OBX'] != null && msg['OBX'].length() > 0) {
    var obxList = msg['OBX'];
    for (var i = 0; i < obxList.length(); i++) {
        var o    = obxList[i];
        var flag = o['OBX.8']['OBX.8.1'].toString();

        if (flag === 'HH' || flag === 'LL') {
            priority = 'CRITICAL';
        } else if ((flag === 'H' || flag === 'L') && priority !== 'CRITICAL') {
            priority = 'ABNORMAL';
        }

        results.push({
            code:   o['OBX.3']['OBX.3.1'].toString(),
            name:   o['OBX.3']['OBX.3.2'].toString(),
            value:  o['OBX.5']['OBX.5.1'].toString(),
            units:  o['OBX.6']['OBX.6.1'].toString(),
            flag:   flag,
            status: o['OBX.11']['OBX.11.1'].toString()
        });
    }
}

channelMap.put('priority',    priority);
channelMap.put('resultsJson', JSON.stringify(results));
```

### 6.7 HL7 to JSON — Complete Transformer

```javascript
function get(seg, fld, comp) {
    try { var v = msg[seg][fld][comp]; return (v != null) ? v.toString().trim() : ''; }
    catch (e) { return ''; }
}
function hl7ToISO(d) {
    if (!d || d.length < 8) return '';
    var s = d.substring(0,4)+'-'+d.substring(4,6)+'-'+d.substring(6,8);
    if (d.length >= 14) s += 'T'+d.substring(8,10)+':'+d.substring(10,12)+':'+d.substring(12,14);
    return s;
}

var mrn = get('PID','PID.3','PID.3.1');
if (mrn === '') throw new Error('MRN required');

var payload = {
    meta: {
        messageId:   get('MSH','MSH.10','MSH.10.1'),
        messageType: get('MSH','MSH.9','MSH.9.1') + '^' + get('MSH','MSH.9','MSH.9.2'),
        sentAt:      hl7ToISO(get('MSH','MSH.7','MSH.7.1'))
    },
    patient: {
        mrn:       mrn,
        lastName:  get('PID','PID.5','PID.5.1'),
        firstName: get('PID','PID.5','PID.5.2'),
        dob:       hl7ToISO(get('PID','PID.7','PID.7.1')),
        gender:    get('PID','PID.8','PID.8.1')
    },
    order: {
        testCode: get('OBR','OBR.4','OBR.4.1'),
        testName: get('OBR','OBR.4','OBR.4.2'),
        orderId:  get('OBR','OBR.2','OBR.2.1')
    },
    results: results  // from OBX loop
};

channelMap.put('jsonPayload', JSON.stringify(payload, null, 2));
```

### 6.8 msg vs tmp

| Variable | What it holds | When to use | Scope of effect |
|----------|--------------|-------------|-----------------|
| `msg` | Parsed inbound HL7 XML | Read HL7 fields, modify HL7 segments | Source transformer: all destinations. Destination transformer: this dest only |
| `tmp` | Outbound message content sent to connector | Build custom output — report text, JSON, CSV | Only this destination connector |

### 6.9 Java Classes from JavaScript (Rhino Engine)

```javascript
// Java Date
var now   = new java.util.Date();
var today = new java.text.SimpleDateFormat('yyyyMMdd').format(now);

// UUID
var uuid = java.util.UUID.randomUUID().toString();

// Database connection inside transformer
var dbConn = DatabaseConnectionFactory.createDatabaseConnection(
    'org.postgresql.Driver',
    'jdbc:postgresql://localhost:5432/hospital_db',
    'mirth_user', 'password'
);
try {
    var rs = dbConn.executeCachedQuery('SELECT name FROM doctors WHERE id=?', [doctorId]);
    if (rs.next()) channelMap.put('doctorName', rs.getString('name'));
} finally { dbConn.close(); }
```

---

## 7. Mirth Maps — Complete Reference

### 7.1 Map Hierarchy and Scope

```
SCOPE: one message, one channel
├── channelMap        → you create, lasts one message journey
├── responseMap       → controls HTTP response back to caller
├── connectorMap      → private to one destination only
└── sourceMap         → Mirth auto-fills, read-only metadata

SCOPE: persists across messages
├── globalChannelMap  → your data, survives for life of this channel
└── globalMap         → your data, survives across ALL channels

SCOPE: read-only infrastructure
└── configurationMap  → set by admins in Settings, read-only in scripts
```

### 7.2 All Maps Compared

| Map | Scope | Lifespan | Who fills it | Use for |
|-----|-------|----------|--------------|---------|
| `channelMap` | One message, whole channel | One message | You | Pass data source → destinations |
| `connectorMap` | One message, one destination | One destination | You | Destination-private values |
| `responseMap` | One message, source only | One message | You | HTTP response to caller |
| `sourceMap` | One message | One message | Mirth auto | File info, HTTP headers, caller IP |
| `globalChannelMap` | All messages, one channel | Channel deployed | You | Counter, cache, deduplication |
| `globalMap` | All messages, all channels | Server running | You | Cross-channel shared state |
| `configurationMap` | Everywhere | Permanent | Admin | URLs, passwords, env config |

### 7.3 channelMap — The Workhorse

```javascript
// Store in source transformer
channelMap.put('mrn',      mrn);
channelMap.put('priority', 'CRITICAL');
channelMap.put('count',    String(42));      // always strings
channelMap.put('isCrit',   String(true));    // booleans as strings

// Read in destination transformer or filter
var mrn     = channelMap.get('mrn');
var count   = parseInt(channelMap.get('count'));
var isCrit  = channelMap.get('isCrit') === 'true';

// Check existence
if (channelMap.get('someKey') !== null) { ... }
```

### 7.4 globalChannelMap — Channel Memory

```javascript
// Deduplication
var dedupKey = 'seen_' + msgId;
if (globalChannelMap.get(dedupKey) === 'true') {
    throw new Error('Duplicate message: ' + msgId);
}
globalChannelMap.put(dedupKey, 'true');

// Daily message counter
var today    = new java.text.SimpleDateFormat('yyyyMMdd').format(new java.util.Date());
var countKey = 'count_' + today;
var count    = parseInt(globalChannelMap.get(countKey) || '0');
globalChannelMap.put(countKey, String(count + 1));

// Lookup cache — expensive DB query cached across messages
var cached = globalChannelMap.get('doctor_' + doctorId);
if (cached === null) {
    // do DB lookup, store result
    globalChannelMap.put('doctor_' + doctorId, doctorName);
}
```

### 7.5 configurationMap — Environment Configuration

```
In Settings → Configuration Map:
Key: ehr_api_url     Value: http://ehr-prod.hospital.com/api/v2
Key: db_host         Value: prod-db.hospital.local
Key: environment     Value: PRODUCTION
Key: ehr_api_token   Value: prod_token_abc123
```

```javascript
// In transformer scripts (read-only)
var url = configurationMap.get('ehr_api_url');
var env = configurationMap.get('environment');

var dbUrl = 'jdbc:postgresql://' +
            configurationMap.get('db_host') + ':5432/' +
            configurationMap.get('db_name');
```

### 7.6 responseMap — HTTP Response Control

```javascript
// Control what HTTP caller receives back
responseMap.put('responseCode', '200');
responseMap.put('response', JSON.stringify({
    status:    'accepted',
    mrn:       channelMap.get('mrn'),
    messageId: String(connectorMessage.getMessageId())
}));

// Error response
responseMap.put('responseCode', '422');
responseMap.put('response', JSON.stringify({ status: 'error', message: 'mrn required' }));
```

### 7.7 sourceMap — Auto-Filled Metadata

```javascript
// HTTP Listener automatic variables
var callerIp    = sourceMap.get('remoteAddress');
var httpMethod  = sourceMap.get('http.method');
var authHeader  = sourceMap.get('http.headers.Authorization');
var contentType = sourceMap.get('http.content.type');

// File Reader automatic variables
var filename = sourceMap.get('originalFilename');
var fileSize = sourceMap.get('fileSize');

// TCP Listener
var senderIp = sourceMap.get('remoteAddress');
var localPort = sourceMap.get('localPort');
```

### 7.8 Map Decision Guide

| Situation | Use This Map |
|-----------|-------------|
| Pass data from source transformer to destinations | `channelMap` |
| Control HTTP response body and status code | `responseMap` |
| Store intermediate value in destination transformer | `connectorMap` |
| Read caller IP, HTTP headers, source filename | `sourceMap` (read-only) |
| Count messages, deduplicate, cache lookups | `globalChannelMap` |
| Share auth token between different channels | `globalMap` |
| Store DB URL, API endpoint, password, env name | `configurationMap` (admin sets) |

---

## 8. Filters — Complete Reference

### 8.1 Filter Fundamentals

A filter is a JavaScript function that returns `true` (process) or `false` (discard). It **never modifies** the message and **never stores** values for later. It only decides yes or no.

```
Filter types:
1. Rule Builder      → point-and-click, single field comparison
2. Iterator Filter   → repeating segments (OBX loop), any/all logic
3. JavaScript Filter → full code, most powerful
4. External Script   → shared filter logic in external .js file
```

### 8.2 JavaScript Filter Rules

```javascript
// CORRECT filter
var msgType = msg['MSH']['MSH.9']['MSH.9.1'].toString();
var event   = msg['MSH']['MSH.9']['MSH.9.2'].toString();
return (msgType === 'ORU' && event === 'R01');

// WRONG — missing .toString() (never matches XML node)
if (msg['MSH']['MSH.9']['MSH.9.1'] === 'ORU') return true;

// WRONG — modifying msg in filter
msg['PID']['PID.5']['PID.5.1'] = 'MODIFIED';  // NEVER do this

// WRONG — no return false path (silent bug)
if (msgType === 'ORU') { return true; }
// must always end with: return false;
```

### 8.3 Common Production Filter Patterns

```javascript
// 1. Message type and event
return (msg['MSH']['MSH.9']['MSH.9.1'].toString() === 'ORU' &&
        msg['MSH']['MSH.9']['MSH.9.2'].toString() === 'R01');

// 2. Sending application whitelist
var sender = msg['MSH']['MSH.3']['MSH.3.1'].toString();
return (sender === 'LIS_PROD' || sender === 'LIS_BACKUP');

// 3. Final results only
return (msg['OBX']['OBX.11']['OBX.11.1'].toString() === 'F');

// 4. From channelMap (destination filter)
var priority = channelMap.get('priority');
return (priority === 'CRITICAL' || priority === 'ABNORMAL');

// 5. Any OBX abnormal flag
if (msg['OBX'] == null) return false;
for (var i = 0; i < msg['OBX'].length(); i++) {
    var f = msg['OBX'][i]['OBX.8']['OBX.8.1'].toString();
    if (f === 'H' || f === 'L' || f === 'HH' || f === 'LL') return true;
}
return false;
```

### 8.4 Filter Types Compared

| Filter Type | Requires Code | Best For | Cannot Do |
|-------------|--------------|---------|-----------|
| Rule Builder | No | Single field — equals, contains, exists | Multi-field logic, channelMap access |
| Iterator | No | Repeating segments — any/all OBX condition | Complex per-iteration logic |
| JavaScript | Yes | Everything complex | Nothing — most capable |
| External Script | Yes (in file) | Shared logic across many channels | N/A |

---

## 9. Transformers — Complete Reference

### 9.1 Transformer Types

| Type | Requires Code | Best For | Limitation |
|------|--------------|---------|-----------|
| Mapper | No | Extract one field → store in channelMap | No conditions or loops |
| Message Builder | No | Set one HL7 field value | No conditional logic |
| JavaScript | Yes | Everything — loops, JSON, conditions, API calls | None |
| External Script | Yes — in file | Shared logic across many channels | Requires server file access |

### 9.2 Source vs Destination Transformer

| | Source Transformer | Destination Transformer |
|--|-------------------|------------------------|
| Runs when | After source filter accepts | After destination filter accepts |
| Changes to `msg` | Affect ALL destinations | Affect ONLY this destination |
| `channelMap` writes | Visible to all destinations | Visible to all destinations |
| `connectorMap` writes | Not applicable | Private to this destination only |
| `tmp` | Unusual | Common — sets what connector sends out |
| Use for | Extract, validate, classify | Format output for this specific destination |

### 9.3 Setting Outbound Content — Mirth 4.5.2

```javascript
// Destination transformer — store content in channelMap
var output = '=== REPORT ===\n' +
             'MRN: ' + channelMap.get('mrn') + '\n' +
             'Test: ' + channelMap.get('testCode');

channelMap.put('d1Content', output);

// Template box in File Writer — use $!{} syntax:
// $!{channelMap['d1Content']}

// $!{} means: if null, write nothing (not literal variable name)
// Do NOT use ${channelMap['key']} — may not resolve in all field types
```

> ⚠️ **Mirth 4.5.2 Limitation:** `connectorMessage.setRawData()` does not exist. Use `channelMap` + `$!{channelMap['key']}` in template box instead.

> ⚠️ **HTTP Sender URL Field:** Does NOT support `${channelMap['key']}` or `${configurationMap['key']}` — causes `URISyntaxException`. Use JavaScript Writer destination with `java.net.URL` instead.

### 9.4 Complete Production Transformer (Source)

```javascript
// ─── Utilities ────────────────────────────────────────────────
function get(seg, fld, comp) {
    try {
        var v = msg[seg][fld][comp];
        return (v != null) ? v.toString().trim() : '';
    } catch (e) { return ''; }
}
function hl7ToISO(d) {
    if (!d || d.length < 8) return '';
    var s = d.substring(0,4)+'-'+d.substring(4,6)+'-'+d.substring(6,8);
    if (d.length >= 14) s += 'T'+d.substring(8,10)+':'+d.substring(10,12)+':'+d.substring(12,14);
    return s;
}

// ─── Extract ──────────────────────────────────────────────────
var mrn      = get('PID','PID.3','PID.3.1');
var lastName = get('PID','PID.5','PID.5.1');
var testCode = get('OBR','OBR.4','OBR.4.1');

// ─── Validate ─────────────────────────────────────────────────
if (mrn === '') {
    logger.error('[TRANSFORMER] MRN missing');
    throw new Error('MRN is required');
}

// ─── Process OBX ──────────────────────────────────────────────
var results  = [];
var priority = 'ROUTINE';
if (msg['OBX'] != null) {
    var obxList = msg['OBX'];
    for (var i = 0; i < obxList.length(); i++) {
        var flag = obxList[i]['OBX.8']['OBX.8.1'].toString();
        if (flag === 'HH' || flag === 'LL') priority = 'CRITICAL';
        else if ((flag === 'H' || flag === 'L') && priority !== 'CRITICAL') priority = 'ABNORMAL';
        results.push({
            name:  obxList[i]['OBX.3']['OBX.3.2'].toString(),
            value: obxList[i]['OBX.5']['OBX.5.1'].toString(),
            flag:  flag
        });
    }
}

// ─── Store in channelMap ───────────────────────────────────────
channelMap.put('mrn',         mrn);
channelMap.put('lastName',    lastName);
channelMap.put('testCode',    testCode);
channelMap.put('priority',    priority);
channelMap.put('resultsJson', JSON.stringify(results));

logger.info('Processed ' + testCode + ' for MRN ' + mrn + ' | priority: ' + priority);
```

---

## 10. Source Connectors

### 10.1 Source Connector Comparison

| Connector | Protocol | Trigger | HL7 Standard | Healthcare Use |
|-----------|----------|---------|-------------|----------------|
| File Reader | File system | Polling interval | No | Batch HL7 files, EDI drops, CSV imports |
| JavaScript Reader | Custom code | Polling interval | No | Custom APIs, proprietary queues |
| Database Reader | JDBC SQL | Polling interval | No | Legacy DB polling, order queues |
| HTTP Listener | HTTP/HTTPS | Inbound request | Rare | REST APIs, FHIR, webhooks |
| TCP Listener | TCP/MLLP | Inbound connection | **Yes — standard** | HIS, LIS, EHR — all clinical HL7 |

### 10.2 TCP Listener — Production Configuration

```
Mode:                 Server
Port:                 6661  (use 6661-6670 range for HL7)
Keep Connection Open: Yes   (avoids TCP handshake per message)
Max Connections:      10
ACK Mode:             Auto-generate ACK  ← NEVER leave as None
ACK Code Success:     AA
ACK Code Error:       AE
Inbound Data Type:    HL7 v2.x
```

### 10.3 Database Reader — PostgreSQL Setup

```
Driver:   org.postgresql.Driver
URL:      jdbc:postgresql://localhost:5432/mirth_demo
          (add ?sslmode=disable if SSL required off)
```

```sql
-- Select Statement
SELECT id, mrn, patient_name, test_code, priority, ordered_at
FROM   lab_orders
WHERE  processed_flag = 0
ORDER  BY ordered_at ASC
LIMIT  10

-- Update Statement (runs after each row is processed)
UPDATE lab_orders
SET    processed_flag = 1, status = 'PROCESSING'
WHERE  id = ${id}
```

```xml
<!-- Message format DB Reader produces per row -->
<result>
  <row>
    <id>1</id>
    <mrn>MRN001</mrn>
    <test_code>CBC</test_code>
  </row>
</result>
```

```javascript
// Access in transformer
var mrn      = msg['row']['mrn'].toString();
var testCode = msg['row']['test_code'].toString();
```

### 10.4 HTTP Listener

```
Port:           8082  (use 8081-8090; NEVER 8080/8443 — Mirth admin)
Context Path:   /patient
Inbound Type:   Raw (for JSON body)
```

```javascript
// Read HTTP body in transformer
var rawBody = connectorMessage.getRawData();
var data    = JSON.parse(rawBody);

// Read HTTP metadata
var callerIp    = sourceMap.get('remoteAddress');
var authHeader  = sourceMap.get('http.headers.Authorization');
```

### 10.5 JavaScript Reader — Required Script

```javascript
// Minimum required in the JavaScript box (for Send Message testing)
return null;

// To auto-generate test messages every poll cycle:
var testMsg =
    'MSH|^~\\&|LIS|LAB|MIRTH|INT|20240315||ORU^R01|MSG001|P|2.5\r' +
    'PID|1||MRN98765^^^HOSPITAL^MR||DOE^JOHN||19850322|M\r' +
    'OBR|1|ORD789|LAB567|CBC^COMPLETE BLOOD COUNT^L|||20240315100000\r' +
    'OBX|1|NM|718-7^HEMOGLOBIN||13.5|g/dL|12.0-17.5|N|||F';

return testMsg;
// Change back to: return null; when done testing
```

---

## 11. Destination Connectors and Data Types

### 11.1 Inbound/Outbound Data Type Quick Reference

| Scenario | Source Inbound | Destination Outbound |
|----------|--------------|---------------------|
| HL7 over TCP → forward HL7 | HL7 v2.x | HL7 v2.x |
| HL7 over TCP → write to file as text | HL7 v2.x | Raw |
| DB Reader → write to DB | XML | XML |
| DB Reader → write to file | XML | Raw |
| HTTP POST JSON → DB | Raw | XML |
| File HL7 → File HL7 | HL7 v2.x | HL7 v2.x |
| Image/PDF relay | Raw (Binary=Yes) | Raw (Binary=Yes) |

### 11.2 File Writer — Key Settings

| Setting | Value | Notes |
|---------|-------|-------|
| Directory | `C:/mirth/output` | Forward slashes always — even on Windows |
| Output Filename | `${channelMap['mrn']}_${DATESTAMP}.txt` | DATESTAMP = date+time combined |
| If File Exists | Overwrite or Append | Append for CSV; Overwrite for reports |
| Binary Output | No / Yes | Yes for JPG/PNG/PDF — prevents corruption |
| Template box | `$!{channelMap['d1Content']}` | Required — never leave blank |
| After Processing (File Reader) | Delete or Move | Never None — causes infinite reprocessing |

### 11.3 Database Writer — PostgreSQL SQL Template

```sql
-- Note: double single quotes inside ${} expressions
INSERT INTO processed_log (mrn, test_code, priority)
VALUES
    ('${channelMap[''mrn'']}',
     '${channelMap[''testCode'']}',
     '${channelMap[''priority'']}')

-- WRONG:  '${channelMap['mrn']}'   → SQL syntax error
-- RIGHT:  '${channelMap[''mrn'']}'  → resolves correctly
```

### 11.4 HTTP Sender — JavaScript Writer Pattern (Mirth 4.5.2)

```javascript
// Use JavaScript Writer instead of HTTP Sender for dynamic URLs
var targetUrl = configurationMap.get('ehr_api_url');
var authToken = configurationMap.get('ehr_api_token');
var payload   = channelMap.get('jsonPayload');

var connection = null;
try {
    var url = new java.net.URL(targetUrl);
    connection = url.openConnection();
    connection.setRequestMethod('POST');
    connection.setDoOutput(true);
    connection.setConnectTimeout(10000);
    connection.setReadTimeout(30000);
    connection.setRequestProperty('Content-Type',  'application/json');
    connection.setRequestProperty('Authorization', 'Bearer ' + authToken);

    var writer = new java.io.OutputStreamWriter(connection.getOutputStream(), 'UTF-8');
    writer.write(payload);
    writer.flush();
    writer.close();

    var code = connection.getResponseCode();
    logger.info('[HTTP] Response: ' + code);
    if (code < 200 || code >= 300) throw new Error('HTTP error: ' + code);

} finally {
    if (connection != null) try { connection.disconnect(); } catch(e) {}
}
```

---

## 12. Debugging and Troubleshooting

### 12.1 Dashboard Reading Guide

| Dashboard shows | What stage broke | First investigation |
|----------------|-----------------|-------------------|
| Received=0 | Message never arrived | Channel started? Port open? Source sending? |
| Filtered high | Source filter rejecting | Add logger to filter — log field being compared |
| Error rising | Transformer or destination failing | Message Browser → ERROR → error detail |
| Received=Sent, Error=0 | Everything working | Nothing to do |
| Queued growing | Destination down | Check network; auto-delivers on recovery |
| Sent<Received, no error | Destination filter rejecting | Add logger to dest filter |

### 12.2 Debug Sequence — Run Every Time

```
1. Dashboard → read Received / Filtered / Queued / Sent / Errored numbers
2. Server Log (Events) → read logger.info trail in order
3. Find the last successful log line → error happened RIGHT AFTER
4. Message Browser → click ERROR message row
5. Raw tab → is original input correct? If wrong: sending system problem
6. Encoded tab → transformer output present? If empty: transformer crashed
7. Each destination row → read Error detail → Java exception = exact failure
8. Fix the code in Channel Editor
9. Save → right-click → Redeploy → Start
10. Message Browser → right-click failed message → Reprocess
11. Verify dashboard Sent count increases
```

### 12.3 Error Reference

| Error Message | Stage | Root Cause | Fix |
|---------------|-------|-----------|-----|
| `TypeError: Cannot read property of undefined` | Transformer | Segment/field missing | Use safe `get()` function |
| `NullPointerException` | Transformer | `.toString()` on null node | Null check before `.toString()` |
| `URISyntaxException: Illegal character` | HTTP Sender URL | `${}` not resolved | Use JavaScript Writer |
| `No such file or directory` | File Writer | Output folder missing | Create folder with forward slashes |
| `Access denied` | File Writer | No write permission | Grant Everyone full control |
| `JDBC driver not found` | DB connector | JAR not in custom-lib | Copy JAR, restart Mirth service |
| `Address already in use` | TCP/HTTP Listener | Port conflict | Change port; check with netstat |
| `Duplicate message` (intentional) | Transformer | globalChannelMap dedup | Change message control ID in test |
| Filter always rejects | Source Filter | `.toString()` missing | Add `.toString()` to comparisons |
| SQL syntax error | DB Writer | Single quotes not doubled | Use `${channelMap[''key'']}` |
| `SSL connection required` | PostgreSQL | DB requires SSL | Add `?sslmode=disable` to URL |
| `Connection refused` | TCP/HTTP destination | Target not running | Check firewall, verify target |
| `Cannot find function setRawData` | Transformer | Method removed in 4.5.2 | Use `channelMap` + `$!{}` template |
| `${TIME}` appears literally in filename | File Writer | `${TIME}` not valid | Use `${DATESTAMP}` instead |

### 12.4 Common Beginner Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Channel saved but not deployed/started | Received=0 forever | Deploy → Start |
| Backslashes in file paths | IOException | Use `C:/mirth/path` not `C:\mirth\path` |
| After Processing = None | Dashboard count explodes | Set to Delete or Move |
| No return false in filter | Wrong messages pass through | Always end filter with `return false` |
| `connectorMessage.setRawData()` | TypeError | Use `channelMap` + `$!{}` template |
| Template box left empty | Pink validation error | Put `$!{channelMap['key']}` |
| Edit but forget to Redeploy | Fix has no effect | Save → Redeploy → Reprocess |
| `${TIME}` in filename | Literal `${TIME}` in filename | Use `${DATESTAMP}` |
| DB Reader Update Statement missing | Same rows fetched every poll | Add UPDATE SET processed_flag=1 |

---

## 13. Production Engineering

### 13.1 Environment Architecture

```
DEVELOPMENT          TEST / UAT           PRODUCTION
─────────────        ──────────────       ─────────────
Build channels       Business validation  Live patient data
Test with samples    Anonymized data      24/7 monitoring
Verbose logging      Sign-off required    Change control
Break things freely  Integration testing  Minimal logging

configurationMap     configurationMap     configurationMap
environment=DEV      environment=TEST     environment=PROD
db_host=dev-db       db_host=test-db      db_host=prod-db
ehr_api_url=         ehr_api_url=         ehr_api_url=
http://localhost     https://test.hosp.   https://ehr.hosp.
```

### 13.2 Production Deployment Checklist

- [ ] All channels Started — dashboard shows green
- [ ] `configurationMap` populated with production values
- [ ] All TCP Listeners have ACK Mode = Auto-generate ACK
- [ ] All File Readers have After Processing = Delete or Move
- [ ] Output folders exist with correct permissions
- [ ] JDBC driver JARs in `custom-lib/`, Test Connection green
- [ ] Test message sent and verified in Message Browser
- [ ] Output files/DB rows contain correct data
- [ ] Alerts configured for error thresholds
- [ ] Excessive `logger.info()` removed for production
- [ ] `globalChannelMap` deduplication in place for TCP channels

### 13.3 JDBC Driver Installation

```
1. Download the correct .jar file
2. Copy to: C:/Program Files/Mirth Connect/custom-lib/
3. Restart Mirth service: Services → Mirth Connect → Restart
4. Reopen Administrator
5. Verify in DB connector dropdowns
```

| Database | Driver Class | JDBC URL | JAR |
|----------|-------------|---------|-----|
| PostgreSQL | `org.postgresql.Driver` | `jdbc:postgresql://host:5432/db` | postgresql-42.x.x.jar |
| MySQL | `com.mysql.cj.jdbc.Driver` | `jdbc:mysql://host:3306/db` | mysql-connector-j-8.x.x.jar |
| SQL Server | `com.microsoft.sqlserver.jdbc.SQLServerDriver` | `jdbc:sqlserver://host:1433;databaseName=db` | mssql-jdbc-12.x.x.jre11.jar |
| Oracle | `oracle.jdbc.OracleDriver` | `jdbc:oracle:thin:@host:1521:SID` | ojdbc11.jar |

### 13.4 Monitoring Daily Workflow

```
Morning checks:
1. Dashboard → scan all channels for Error > 0 or Stopped
2. Stopped channels that should run → Start them
3. Error count rising → Message Browser → investigate
4. Queued count growing → destination likely down → check connectivity
5. Events → Server Log → scan for ERROR lines from overnight
6. Alerts → any triggered alerts from overnight
```

---

## 14. Complete Channel Examples

### 14.1 Channel Inventory

| Channel Name | Source | Port/Path | Destinations | Data Types |
|-------------|--------|-----------|-------------|------------|
| File_To_File_HL7 | File Reader | `hl7_input/*.hl7` | File Writer | HL7 in / HL7 out |
| HTTP_API_To_Database | HTTP Listener | `8082 /patient` | DB Writer | Raw in / XML out |
| HTTP_API_To_CSV | HTTP Listener | `8083 /lab-result` | File Writer (append) | Raw in / Raw out |
| Binary_Image_Relay | File Reader | `images_input` | File Writer | Raw+Binary in/out |
| HL7_TCP_To_Database | TCP Listener | `6661` | DB Writer | HL7 in / XML out |
| HL7_To_JSON_HTTP | TCP Listener | `6662` | JS Writer HTTP | HL7 in / Raw out |
| DB_Source_Multi_Dest | DB Reader | `lab_orders` table | DB Writer + 2× File Writer | XML in / mixed |
| HL7_Parse_And_Route | JS Reader | Send Message | File Writer ×2 (all/abnormal) | HL7 in / Raw out |

### 14.2 DB Source Multi-Destination Architecture

```
PostgreSQL lab_orders table (processed_flag = 0)
         ↓  SQL SELECT every 10 seconds
DB READER source connector
         ↓  XML row message: <result><row>...</row></result>
SOURCE FILTER: only status = 'NEW'
         ↓
STEP 1: Extract fields from msg['row']['column']
STEP 2: Normalize priority → ROUTINE / URGENT / CRITICAL
         ↓  fan out to 3 destinations
┌─────────────────┬──────────────────┬─────────────────┐
↓                 ↓                  ↓
D1: DB Writer     D2: File Writer    D3: File Writer
processed_log     all_orders/        urgent_orders/
All orders        All orders         URGENT+CRITICAL only
filter: all       filter: all        filter: isUrgent='true'

After each row: UPDATE processed_flag=1 WHERE id=${id}
```

### 14.3 PostgreSQL Schema for Demo Channel

```sql
-- Source table
CREATE TABLE lab_orders (
    id              SERIAL PRIMARY KEY,
    mrn             VARCHAR(50)  NOT NULL,
    patient_name    VARCHAR(100),
    test_code       VARCHAR(20),
    test_name       VARCHAR(100),
    doctor_name     VARCHAR(100),
    ordered_at      TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    priority        VARCHAR(20)  DEFAULT 'ROUTINE',
    status          VARCHAR(20)  DEFAULT 'NEW',
    processed_flag  SMALLINT     DEFAULT 0
);

-- Destination table
CREATE TABLE processed_log (
    id              SERIAL PRIMARY KEY,
    order_id        VARCHAR(50),
    mrn             VARCHAR(50),
    patient_name    VARCHAR(100),
    test_code       VARCHAR(20),
    priority        VARCHAR(20),
    processed_at    TIMESTAMP    DEFAULT CURRENT_TIMESTAMP,
    channel_name    VARCHAR(100)
);

-- Test data
INSERT INTO lab_orders (mrn, patient_name, test_code, test_name, doctor_name, priority)
VALUES
('MRN001', 'John Doe',    'CBC',  'Complete Blood Count',  'Dr. Smith', 'ROUTINE'),
('MRN002', 'Jane Smith',  'BMP',  'Basic Metabolic Panel', 'Dr. Jones', 'URGENT'),
('MRN003', 'Bob Wilson',  'LFT',  'Liver Function Test',   'Dr. Brown', 'ROUTINE'),
('MRN004', 'Mary Johnson','TROP', 'Troponin',              'Dr. Smith', 'CRITICAL');
```

---

## 15. Best Practices and Anti-Patterns

### 15.1 Best Practices

| Area | Best Practice | Why |
|------|--------------|-----|
| Field extraction | Always use safe `get()` function with try/catch | Missing segments crash naive `.toString()` calls |
| Logging | Log at entry and exit of each transformer step | Server Log shows exactly where processing stopped |
| MRN validation | Throw Error if MRN empty in source transformer | Cannot route or audit without patient identifier |
| Date conversion | Always convert HL7 YYYYMMDD to ISO in transformer | Do it once — all destinations benefit |
| Configuration | Store all URLs/passwords in configurationMap | Promotes across environments without code changes |
| Channel design | One channel per business use-case | Fix one without touching others |
| ACK | Always Auto-generate ACK on TCP Listener | Without ACK sender retries — creates duplicates |
| File processing | Always set After Processing to Delete or Move | Prevents infinite reprocessing loops |
| SQL safety | Replace single quotes before DB write | Apostrophes in patient names break SQL |
| Deduplication | Use globalChannelMap to track seen message IDs | Production systems retry — deduplicate at Mirth |
| Testing | Test every scenario: routine, abnormal, critical, missing MRN | Production messages are always more varied than test messages |

### 15.2 Anti-Patterns — Never Do These

| Anti-Pattern | Problem | Correct Approach |
|-------------|---------|-----------------|
| Modifying `msg` inside a filter | Unpredictable behavior | Move all modifications to transformer |
| `channelMap.put()` in filter | Data unavailable if filter returns false | Store in transformer only |
| Hardcoding URLs in scripts | Breaks when promoting to production | Use `configurationMap` for all env-specific values |
| One giant transformer step | Hard to debug — can't identify failing line | Split into focused named steps |
| `globalMap` for per-channel data | Accidentally shared with other channels | Use `globalChannelMap` for channel-specific data |
| `.toString()` without null check | NullPointerException in production | Always use safe `get()` utility |
| Template box left empty | Pink validation error — can't save | Put `$!{channelMap['key']}` as minimum |
| Not redeploying after edits | Old version still running | Save → Redeploy always |
| Ignoring error messages | Small errors compound to data loss | Review Error tab for every failed message |

---

## 16. Mirth Administrator Views

| View | Location | Purpose | Daily Usage |
|------|----------|---------|-------------|
| Dashboard | Left sidebar | All channels live stats | Morning check — spot errors, stopped channels |
| Channel Editor | Channels → Edit | Build and configure channels | Building and editing integrations |
| Message Browser | View Messages | Every message — Raw, Encoded, Sent, Response, Error | Debug failed, verify output, reprocess |
| Server Log | Events → Server Log | All `logger.info/warn/error` output in real time | Trace transformer execution |
| Events | Left sidebar | Server-level events — deploy, start, stop | Troubleshoot server issues |
| Alerts | Left sidebar | Email notifications on error thresholds | Production monitoring |
| Settings | Left sidebar | Server config, Configuration Map, JDBC resources | Set configurationMap, install JDBC drivers |
| Filter Editor | Source tab → Edit Filter | Build filter steps | Configure message acceptance logic |
| Transformer Editor | Source tab → Edit Transformer | Build transformer steps | Write extraction and conversion logic |

---

## 17. Engineering Cheat Sheet

### 17.1 HL7 Quick Reference

```
MESSAGE TYPES:
ORU^R01   = Lab result
ADT^A01   = Patient admitted
ADT^A02   = Patient transferred
ADT^A03   = Patient discharged
ADT^A08   = Patient info updated
ORM^O01   = Lab/radiology order
SIU^S12   = Appointment scheduled

ABNORMAL FLAGS (OBX.8):
N  = Normal
L  = Low
H  = High
LL = Critically low
HH = Critically high
A  = Abnormal

RESULT STATUS (OBX.11):
F = Final
P = Preliminary
C = Corrected
W = Wrong (retract)
```

### 17.2 JavaScript Snippets

```javascript
// Safe getter (copy to every transformer)
function get(s,f,c){try{var v=msg[s][f][c];return v!=null?v.toString().trim():''}catch(e){return ''}}

// Date converter
function hl7ToISO(d){if(!d||d.length<8)return '';var r=d.substring(0,4)+'-'+d.substring(4,6)+'-'+d.substring(6,8);if(d.length>=14)r+='T'+d.substring(8,10)+':'+d.substring(10,12)+':'+d.substring(12,14);return r}

// CSV escape
function csvEscape(v){if(!v)return '';var s=String(v);if(s.indexOf(',')>-1||s.indexOf('"')>-1)return '"'+s.replace(/"/g,'""')+'"';return s}

// OBX priority scan
var priority='ROUTINE';
if(msg['OBX']!=null){var l=msg['OBX'];for(var i=0;i<l.length();i++){var f=l[i]['OBX.8']['OBX.8.1'].toString();if(f==='HH'||f==='LL')priority='CRITICAL';else if((f==='H'||f==='L')&&priority!=='CRITICAL')priority='ABNORMAL'}}
channelMap.put('priority',priority);

// SQL safe string
function sqlSafe(v){return v?v.toString().replace(/'/g,"''"):''}
```

### 17.3 Port Conventions

| Range | Use | Examples |
|-------|-----|---------|
| 6661–6670 | HL7 TCP/MLLP listeners | 6661=ADT, 6662=ORU, 6663=ORM |
| 8081–8090 | HTTP Listener channels | 8082=/patient, 8083=/lab-result |
| **8080, 8443** | **RESERVED — Mirth admin** | **Never use for channels** |
| 5432 | PostgreSQL default | Change in JDBC URL if different |
| 1433 | SQL Server default | Change in JDBC URL if different |

### 17.4 Filename Variables

| Variable | Example Output | Use For |
|----------|--------------|---------|
| `${DATE}` | 2024-05-14 | Date only |
| `${DATESTAMP}` | 20240514173118 | Date + time combined — safe for filenames |
| `${SYSTIME}` | 1715699478123 | Unix timestamp milliseconds |
| `${channelMap['mrn']}` | MRN98765 | Dynamic value from transformer |
| `${configurationMap['key']}` | PROD | Admin config value |

### 17.5 Inbound/Outbound Data Type Matrix

| Source Data | Inbound Type | Outbound Type |
|-------------|-------------|--------------|
| HL7 v2 over TCP/File | HL7 v2.x | HL7 v2.x (forward) or Raw (custom output) |
| DB Reader rows | XML | XML (DB dest) or Raw (file dest) |
| HTTP JSON body | Raw | Raw |
| CSV / plain text | Raw | Raw |
| Binary files (images/PDF) | Raw + Binary=Yes | Raw + Binary=Yes |

### 17.6 Map Quick Decision

```
Need to pass data source → destinations?         → channelMap
Need to control HTTP response?                   → responseMap
Destination-private intermediate value?          → connectorMap
Read file metadata or HTTP headers?              → sourceMap (read only)
Count messages across multiple messages?         → globalChannelMap
Share token between two different channels?      → globalMap
Store DB URL, API endpoint, environment name?    → configurationMap (admin sets)
```

### 17.7 Production Readiness Checklist

```
CHANNELS:
☐ All channels Started (green dashboard)
☐ One channel per business use-case

CONFIGURATION:
☐ configurationMap has production values (URLs, passwords, env)
☐ JDBC drivers in custom-lib/, Test Connection green

SOURCE CONNECTORS:
☐ TCP Listeners: ACK Mode = Auto-generate ACK
☐ File Readers: After Processing = Delete or Move
☐ HTTP Listeners: using ports 8081-8090 (not 8080/8443)

DESTINATIONS:
☐ Output folders exist with correct permissions (forward slashes)
☐ Template boxes contain $!{channelMap['key']} — not empty
☐ DB Writer SQL uses doubled single quotes in velocity: ''key''

TRANSFORMERS:
☐ safe get() function at top of every transformer
☐ MRN validation with throw on empty
☐ logger.info in each step for trace
☐ Excessive debug logging removed for production

TESTING:
☐ Test messages sent and verified
☐ Message Browser shows SENT
☐ Output files/DB rows contain correct data
☐ Abnormal result routing verified (D2/D3 filters tested)
☐ Error scenario tested — failed message reprocessed successfully

MONITORING:
☐ Alerts configured for error thresholds
☐ globalChannelMap deduplication for TCP channels
```

---

*Mirth Connect Engineering Handbook — Healthcare Integration Reference*  
*Mirth Connect 4.5.2 · HL7 v2 · PostgreSQL · Rhino JavaScript*

