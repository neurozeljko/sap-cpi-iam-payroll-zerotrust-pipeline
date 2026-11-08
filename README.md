# Military-Grade Zero-Trust IAM & Asymmetric Payroll Obfuscation Pipeline

[![SAP Integration Suite](https://img.shields.io/badge/SAP%20Integration%20Suite-Cloud%20Integration-0A85EA?style=for-the-badge&logo=sap&logoColor=white)](https://www.sap.com)
[![Zero-Trust Standard](https://img.shields.io/badge/Security-NIST%20SP%20800--207%20Zero--Trust-red?style=for-the-badge)](https://csrc.nist.gov)
[![SuccessFactors BIB](https://img.shields.io/badge/HRIS-SF%20Business%20Integration%20Builder-orange?style=for-the-badge&logo=sap&logoColor=white)](https://www.sap.com)
[![Cryptography](https://img.shields.io/badge/Encryption-PGP%20RSA--4096%20%7C%20FPE%20AES--256-blueviolet?style=for-the-badge)](https://www.openpgp.org)
[![Identity Management](https://img.shields.io/badge/Standard-SCIM%202.0%20(RFC%207643%2F7644)-darkgreen?style=for-the-badge)](https://datatracker.ietf.org/doc/html/rfc7644)
[![Regulatory Compliance](https://img.shields.io/badge/Compliance-GDPR%20Art.%2032%20%7C%20SOX%20404-black?style=for-the-badge)](https://gdpr-info.eu)

---

## Executive Summary

Global enterprises processing cross-border executive compensation, employee banking coordinates, and enterprise identity lifecycles face catastrophic regulatory liabilities under **GDPR Article 83** (administrative fines up to €20M or 4% of total worldwide annual turnover) and **SOX Section 404** if employee banking coordinates, Social Security Numbers (SSN/JMBG), or compensation scales are exposed in transit. Conventional enterprise HR pipelines routinely rely on unencrypted flat CSV files scheduled over nocturnal SFTP batches or unmasked intermediate SAP staging tables, exposing Personally Identifiable Information (PII) to privileged cloud infrastructure administrators and man-in-the-middle (MitM) packet inspection.

This integration architecture establishes a **Military-Grade Zero-Trust IAM & Payroll Obfuscation Pipeline** deployed on **SAP Integration Suite**:
1. **Event-Driven SF BIB Ingestion:** Ingests contractual worker lifecycle events in real time from SAP SuccessFactors Business Integration Builder (BIB) and Intelligent Services, eliminating batch exposure windows.
2. **In-Flight Volatile RAM Tokenization & FPE:** Executes deterministic **Format-Preserving Encryption (FPE-AES-256)** on National IDs and structural masking on IBAN routing accounts within non-persisted JVM heap memory prior to transformation or audit logging.
3. **Decoupled SCIM 2.0 Identity Provisioning:** Maps sanitized worker profiles into standardized **SCIM 2.0 (RFC 7643/7644)** RESTful payloads, orchestrating Just-In-Time (JIT) identity provisioning to **SAP Cloud Identity Services (IAS/IPS)** and **Microsoft Entra ID** with zero access to compensation data.
4. **Asymmetric PGP (RSA-4096 / AES-256) Banking Obfuscation:** Compresses and encrypts banking settlement manifests using the external banking clearing house's public OpenPGP key paired with an SHA-512 digital signature. Payloads remain mathematically unreadable—even to BTP tenant administrators with root permissions—until decrypted inside the bank's Hardware Security Module (HSM).
5. **Tamper-Evident Forensic DLQ:** Quarantines schema discrepancies, replay attempts, or cryptographic signature violations into an append-only Write-Once-Read-Many (WORM) audit store.

---

## Architecture Topology

![SAP CPI Zero-Trust IAM & Payroll Pipeline](sap-cpi-iam-payroll-zerotrust-runtime-architecture.png)

```text
       UNTRUSTED / DMZ BOUNDARY                   VOLATILE JVM RAM ENCLAVE                 ISOLATED ENTERPRISE NETWORKS
  ┌───────────────────────────────────┐    ┌─────────────────────────────────────┐    ┌──────────────────────────────────────┐
  │                                   │    │                                     │    │                                      │
  │  SAP SuccessFactors (BIB Event)   │    │  [Tokenize_PII_Memory.groovy]       │    │  SAP Cloud Identity Services (IAS)   │
  │  - Contract Signed / Role Change  │    │  - FPE AES-256 National ID          │    │  Microsoft Entra ID (RFC 7644)       │
  │  - PII + Banking + Compensation   │    │  - IBAN In-Memory Masking           │    │  (Identity & SAML Claims Only)       │
  │                  │                │    │  - SHA-512 Merkle Digest            │    │                  ▲                   │
  └──────────────────┼────────────────┘    └──────────────────┬──────────────────┘    └──────────────────┼───────────────────┘
                     │ HTTPS / mTLS 1.3                       │                                          │ SCIM 2.0 POST
                     ▼                                        ▼                                          │
        ┌─────────────────────────┐               ┌───────────────────────┐             ┌────────────────┴──────────────────┐
        │  Sender Endpoint:       │               │  Parallel Multicast   │────────────►│ Branch 1: SCIM 2.0 Generator      │
        │  /zerotrust/payroll/    │──────────────►│  Branch Distribution  │             │ [Build_SCIM2_Identity_Resource]   │
        │  contract/ingest        │               └───────────┬───────────┘             └───────────────────────────────────┘
        └─────────────────────────┘                           │
                                                              │                         ┌───────────────────────────────────┐
                                                              └────────────────────────►│ Branch 2: Banking Settlement     │
                                                                                        │ [Prepare_Banking_Settlement_Env]  │
                                                                                        └─────────────────┬─────────────────┘
                                                                                                          │
                                                                                                          ▼
                                                                                        ┌───────────────────────────────────┐
                                                                                        │ PGP Encryptor (RSA-4096 / AES-256)│
                                                                                        │ SHA-512 Digital Signature Engine  │
                                                                                        └─────────────────┬─────────────────┘
                                                                                                          │ Encrypted OpenPGP Stream
                                                                                                          ▼
                                                                                        ┌───────────────────────────────────┐
                                                                                        │ Banking Clearing / SEPA Gateway   │
                                                                                        │ Deutsche Bundesbank / BACS        │
                                                                                        └───────────────────────────────────┘
```

---

## Technical Specifications Matrix

| Dimension | Classical Enterprise HR Integration | Military-Grade Zero-Trust Mesh |
| :--- | :--- | :--- |
| **Trust Model** | Perimeter-based (Implicit Trust inside Corporate WAN) | **NIST SP 800-207 Zero-Trust** (Never Trust, Always Verify) |
| **Transport Layer** | Plain FTP or TLS 1.1 / TLS 1.2 with static service users | **Mutual TLS 1.3 (mTLS)** with X.509 and Anti-Replay Nonces |
| **Data Ingestion** | Scheduled overnight flat CSV batch extraction | **Event-Driven Webhook** via SuccessFactors BIB & Event Mesh |
| **PII at Rest & Transit** | Plaintext SSN, bank routing, and salary data | In-flight **Format-Preserving Encryption (FPE-AES-256-FF3-1)** |
| **Payload Persistence** | Written to unencrypted disk on intermediate SFTP boxes | **Zero disk persistence**; processed strictly in volatile JVM RAM |
| **Identity Provisioning** | Manual IT ticketing / delayed batch synchronization | Automated **SCIM 2.0 (RFC 7643/7644)** real-time JIT provisioning |
| **Banking Payload Security**| Relies solely on transport layer TLS encryption | **Envelope Encryption: PGP (RSA-4096 + AES-256-GCM)** |
| **Non-Repudiation** | Basic server text logs subject to root tampering | **SHA-512 Merkle Root Hashing** with cryptographic audit trail |
| **Insider Threat Defense** | BTP Tenant Admins can inspect employee salaries | Data unreadable even to administrators with BTP root access |
| **Regulatory Standing** | High compliance risk under GDPR Art. 32 & SOX 404 | **Strict Compliance: Zero PII leakage, immutable auditability** |

---

## STRIDE Threat Model & Security Mitigations

| Threat Vector | STRIDE Classification | Attack Scenario | Implemented Architecture Countermeasure |
| :--- | :--- | :--- | :--- |
| **Spoofing** | Identity Spoofing | Adversary impersonates SuccessFactors to push fake worker records. | Sender authenticated via mutual TLS (mTLS) with client certificates; sender IP whitelist enforced via BTP API Gateway. |
| **Tampering** | Data Modification | Malicious administrator alters worker bank details in flight. | Payload integrity validated with SHA-512 Merkle hash; any modification invalidates the cryptographic envelope. |
| **Repudiation** | Disputability | HR department denies initiating an unauthorized executive payout. | Immutable audit log captures `Prop_Audit_CorrelationId`, client certificate DN, and timestamp in WORM forensic storage. |
| **Information Disclosure** | Data Leakage | Packet sniffer or Cloud Admin inspects employee SSN and salary. | In-flight FPE tokenization and PGP RSA-4096 encryption ensure cleartext never exists on disk or in logs. |
| **Denial of Service** | Resource Exhaustion | Flood of oversized worker payloads crashes JVM heap memory. | Streaming `java.io.Reader` implementation prevents Heap exhaustion; linter enforces non-buffering JSON parsing. |
| **Elevation of Privilege** | IAM Misconfiguration | IT Admin granted payroll access gains control over corporate IAM. | Strict separation of concerns via Parallel Multicast: IAM branch receives zero salary data; Banking branch receives zero corporate credentials. |

---

## Cryptographic Envelope & Payload Schemas

### 1. Inbound SuccessFactors BIB Contract Event (`POST /http/zerotrust/payroll/contract/ingest`)
```json
{
  "workerId": "EMP-DE-884102",
  "nationalId": "1234567890",
  "firstName": "Elena",
  "lastName": "Rostova",
  "workEmail": "elena.rostova@enterprise.com",
  "costCenter": "CC-DE-1010-RND",
  "division": "Enterprise Architecture",
  "department": "Security & Cryptography",
  "compensation": {
    "baseSalaryEUR": 18500.00,
    "currency": "EUR",
    "bankAccountIBAN": "DE89370400440532013000",
    "routingCode": "DEUTDEDDFXX"
  }
}
```

### 2. Intermediate In-Flight Tokenized State (RAM-Enclave Only)
```json
{
  "workerId": "EMP-DE-884102",
  "nationalId": "FPE-123-XX-7890",
  "firstName": "Elena",
  "lastName": "Rostova",
  "workEmail": "elena.rostova@enterprise.com",
  "costCenter": "CC-DE-1010-RND",
  "division": "Enterprise Architecture",
  "department": "Security & Cryptography",
  "compensation": {
    "baseSalaryEUR": "OBFUSCATED_IN_MEMORY",
    "currency": "EUR",
    "bankAccountIBAN": "DE89-XXXX-XXXX-3000",
    "routingCode": "DEUTDEDDFXX"
  }
}
```

### 3. Branch 1: SCIM 2.0 User Representation (`RFC 7643 / 7644`)
*Sent to SAP Cloud Identity Services (IAS) & Microsoft Entra ID:*
```json
{
  "schemas": [
    "urn:ietf:params:scim:schemas:core:2.0:User",
    "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User"
  ],
  "userName": "elena.rostova@enterprise.com",
  "name": {
    "givenName": "Elena",
    "familyName": "Rostova"
  },
  "active": true,
  "emails": [
    {
      "value": "elena.rostova@enterprise.com",
      "type": "work",
      "primary": true
    }
  ],
  "urn:ietf:params:scim:schemas:extension:enterprise:2.0:User": {
    "employeeNumber": "EMP-DE-884102",
    "costCenter": "CC-DE-1010-RND",
    "division": "Enterprise Architecture",
    "department": "Security & Cryptography"
  }
}
```

### 4. Branch 2: Asymmetric OpenPGP Banking Settlement Envelope
*Structured payload prior to OpenPGP RSA-4096 / AES-256 encapsulation:*
```json
{
  "settlementHeader": {
    "clearingStandard": "SEPA_PAIN_001_CREDIT_TRANSFER",
    "batchReference": "PAYROLL-B4F81A9C",
    "executionDate": "2026-11-08",
    "integrityDigestSHA512": "4a58b88d5e12...c29b"
  },
  "transactionDetail": {
    "beneficiaryId": "EMP-DE-884102",
    "beneficiaryNationalIdToken": "FPE-123-XX-7890",
    "paymentCurrency": "EUR",
    "instructedAmount": 18500.00,
    "settlementRoutingCode": "DEUTDEDDFXX"
  }
}
```

*Final OpenPGP Encrypted & Signed Wire Stream dispatched to Clearing House:*
```text
-----BEGIN PGP MESSAGE-----
Version: BTP Cryptographic Engine v4.2
Comment: Recipient: Deutsche Bundesbank / Corporate Settlement HSM

hQGMA8X7Xq3r5VdBAQv/Z4Bq6W8uYq+N1u5lI4x5G0qZ5r1R7s2T3v4X5y6Z7a8b
9c0d1e2f3g4h5i6j7k8l9m0n1o2p3q4r5s6t7u8v9w0x1y2z3a4b5c6d7e8f9g0h
1i2j3k4l5m6n7o8p9q0r1s2t3u4v5w6x7y8z9a0b1c2d3e4f5g6h7i8j9k0l1m2n
=W9xA
-----END PGP MESSAGE-----
```

---

## Repository Structure

```text
sap-cpi-iam-payroll-zerotrust-pipeline/
├── README.md                                                 # Comprehensive architecture & operational runbook
├── sap-cpi-iam-payroll-zerotrust-runtime-architecture.png   # Deployed iFlow runtime topology screenshot
├── ADR-013-ZeroTrust-FPE-PGP-Payroll-Architecture.md        # Formal Architecture Decision Record
├── IF_ZeroTrust_Payroll_Obfuscation_IAM_Provisioning.zip     # Production iFlow runtime bundle export
├── cpi/
│   └── scripts/
│       ├── Tokenize_PII_Memory.groovy                        # In-flight FPE tokenization & SHA-512 Merkle engine
│       ├── Build_SCIM2_Identity_Resource.groovy              # RFC 7643/7644 SCIM 2.0 streaming builder
│       ├── Prepare_Banking_Settlement_Envelope.groovy        # Banking manifest compression & settlement builder
│       └── Format_Forensic_Security_DLQ.groovy               # Tamper-evident dead-letter exception handler
├── schemas/
│   ├── SuccessFactors_BIB_Contract_Approval.xsd              # Inbound SuccessFactors BIB contract schema
│   └── scim-core-user-schema.json                            # RFC 7643 SCIM User resource specification
└── tests/
    ├── sample_valid_contract_approval.json                   # Verified production-grade test payload
    └── sample_tampered_security_breach.json                  # Fault test payload triggering forensic DLQ
```

---

## Verification & Operational Runbook

### 1. Inbound Runtime Endpoint
The integration flow is actively deployed and listening on the SAP BTP Cloud Integration tenant:
```text
[https://integration-suite-eu10.it-cpi026-rt.cfapps.eu10-002.hana.ondemand.com/http/zerotrust/payroll/contract/ingest](https://integration-suite-eu10.it-cpi026-rt.cfapps.eu10-002.hana.ondemand.com/http/zerotrust/payroll/contract/ingest)
```

### 2. Dispatched cURL Test (Security Ingestion Test)
Execute an authorized POST request using mutual TLS and valid contract event data:

```bash
curl -i -X POST \
  '[https://integration-suite-eu10.it-cpi026-rt.cfapps.eu10-002.hana.ondemand.com/http/zerotrust/payroll/contract/ingest](https://integration-suite-eu10.it-cpi026-rt.cfapps.eu10-002.hana.ondemand.com/http/zerotrust/payroll/contract/ingest)' \
  -H 'Content-Type: application/json' \
  -H 'X-Anti-Replay-Nonce: 4820a891-64bb-4412-9910-2b0d7b3dcb6d' \
  -d '{
    "workerId": "EMP-DE-884102",
    "nationalId": "1234567890",
    "firstName": "Elena",
    "lastName": "Rostova",
    "workEmail": "elena.rostova@enterprise.com",
    "costCenter": "CC-DE-1010-RND",
    "division": "Enterprise Architecture",
    "department": "Security & Cryptography",
    "compensation": {
      "baseSalaryEUR": 18500.00,
      "currency": "EUR",
      "bankAccountIBAN": "DE89370400440532013000",
      "routingCode": "DEUTDEDDFXX"
    }
  }'
```

*Expected Terminal Output:*
`HTTP/1.1 401 Unauthorized` (or `403 Forbidden` if certificates are missing), confirming that the runtime endpoint is active, listening, and protected by BTP platform security policies.

### 3. Monitoring & Security Audit Trace
1. Open **SAP Integration Suite** ──► **Monitor** ──► **Monitor Message Processing**.
2. Locate the Message Processing Log (MPL) for `IF_ZeroTrust_Payroll_Obfuscation_IAM_Provisioning`.
3. Confirm that:
   * Property `Prop_Worker_ID = EMP-DE-884102`
   * Property `Prop_Tokenized_NationalId = FPE-123-XX-7890`
   * Property `Prop_Payload_Integrity_SHA512` is populated with a valid 128-character hexadecimal digest.
   * Neither raw bank account numbers nor salary amounts appear anywhere in the log or trace body.

---

## Author

**Željko Popov**  
*Integration Architect | SAP Integration Suite & Clean Core Specialist*  
[GitHub Repositories](https://github.com/neurozeljko)
