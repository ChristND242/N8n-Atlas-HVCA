```markdown
# Automated Certificate Issuance via GlobalSign HVCA API using n8n (Heroku Deployment)

## Author
Automation Implementation Report

---

# Table of Contents

1. Executive Summary  
2. Introduction  
3. Objectives  
4. System Architecture  
5. Environment & Deployment Setup  
6. Workflow Design Overview  
7. Implementation Details  
8. CSR Normalization Strategy (Critical Section)  
9. Error Analysis & Resolutions  
10. Security Architecture & Hardening  
11. Polling & Certificate Lifecycle Management  
12. Trust Chain Assembly  
13. Operational Considerations  
14. Advantages of Automation  
15. Risk Analysis & Mitigation  
16. Performance & Reliability Considerations  
17. Conclusion  
18. Future Improvements  

---

# 1. Executive Summary

This document details the full implementation of an automated certificate issuance workflow using:

- GlobalSign HVCA REST API
- Mutual TLS (mTLS)
- Bearer token authentication
- n8n (self-hosted on Heroku)
- Secure secret management via Heroku Config Vars

The automation programmatically:

1. Authenticates to HVCA
2. Submits a CSR
3. Polls until certificate issuance
4. Retrieves the leaf certificate
5. Fetches the trust chain
6. Builds a production-ready `fullchain.pem`

The implementation follows industry best practices in:

- Secure secret handling
- Deterministic PEM normalization
- Resilient polling design
- Observability
- API hygiene
- Automation scalability

---

# 2. Introduction

Manual certificate issuance introduces:

- Human error risk
- Inconsistent CSR formatting
- Operational delays
- Secret leakage risks
- Lack of auditability

The goal was to eliminate manual intervention by implementing a fully automated, reproducible certificate issuance pipeline using n8n as the orchestration engine.

This approach aligns with modern DevSecOps principles: Infrastructure as Code, immutable operations, and secure automation.

---

# 3. Objectives

Primary goals:

- Eliminate manual CSR submission
- Standardize CSR formatting
- Securely manage API credentials
- Automatically retrieve and assemble certificate chain
- Enable scalable repeatable issuance
- Improve reliability and auditability

---

# 4. System Architecture

## Components

| Component | Role |
|------------|------|
| Heroku | Hosts n8n runtime |
| n8n | Workflow orchestration |
| HVCA API | Certificate authority backend |
| mTLS | Transport-level authentication |
| Bearer Token | API-level authorization |
| CSR | Certificate signing request |

## Flow Overview

```

Login → Submit CSR → Extract Serial → Poll Status → Retrieve Leaf → Get Trustchain → Build Fullchain

```

---

# 5. Environment & Deployment Setup

## Platform
- n8n self-hosted on Heroku
- Environment variables configured via Heroku Config Vars

## Required Environment Variables

```

HVCA_BASE_URL
HVCA_KEY
HVCA_SECRET
N8N_ENCRYPTION_KEY

```

### WHY `N8N_ENCRYPTION_KEY` is Critical

Without a stable encryption key:
- Credentials may fail across dyno restarts
- Stored secrets can become unreadable
- Execution consistency breaks

This ensures deterministic encryption behavior.

---

# 6. Workflow Design Overview

## Node Structure

1. Login (POST `/v2/login`)
2. CSR Preparation (Code Node)
3. Cert Request (POST `/v2/certificates`)
4. Extract Serial
5. Poll Certificate Status
6. Store Leaf PEM
7. Get Trust Chain
8. Build Fullchain

Each node has a single responsibility.

This adheres to clean automation architecture principles.

---

# 7. Implementation Details

## 7.1 Login Node

POST:
```

/v2/login

````

Body:
```json
{
  "key": "{{HVCA_KEY}}",
  "secret": "{{HVCA_SECRET}}"
}
````

Response:

* Bearer token

WHY:
Separates transport authentication (mTLS) from API authentication (Bearer token).

---

## 7.2 CSR Submission

POST:

```
/v2/certificates
```

Payload contains:

* validity
* subject_dn
* san
* public_key (CSR)

Critical: CSR must be properly formatted PEM.

---

# 8. CSR Normalization Strategy (Critical Section)

## Problem Encountered

Error:

```
422 - public_key: invalid encoding
```

Root cause:
CSR was submitted as a single-line string with spaces instead of properly wrapped PEM with newline characters.

HVCA requires strict PEM formatting.

---

## Resolution: Deterministic CSR Normalization

Key operations:

1. Remove all whitespace
2. Validate BEGIN/END markers
3. Validate base64 charset
4. Rewrap base64 at 64 characters
5. Rebuild PEM with newline characters

Why this matters:

* PEM is not "free text"
* API validators enforce strict ASN.1 expectations
* Copy/paste corruption introduces Unicode dash variants

Result:
Fully compliant PEM encoding.

---

# 9. Error Analysis & Resolutions

## Error 415 — Bad Content-Type

Cause:
Incorrect Content-Type header.

Resolution:

```
Content-Type: application/json; charset=utf-8
```

WHY:
HVCA expects explicit JSON encoding.

---

## Error 401 — No Authorization Header

Cause:
Bearer token not included.

Resolution:

```
Authorization: Bearer <access_token>
```

---

## Error 405 — Method Not Allowed

Cause:
Incorrect endpoint (`/certificate` instead of `/certificates`)

Resolution:
Corrected endpoint path.

WHY:
REST APIs are path-sensitive.

---

## Error 422 — public_key invalid encoding

Cause:
CSR flattened into single line with spaces.

Resolution:
Implemented strict normalization logic.

---

## Error — No certificate field found

Cause:
Response was nested under `body` due to full response option.

Resolution:
Accessed data via:

```
$json.body.certificate
```

---

# 10. Security Architecture & Hardening

## Secrets Management

Secrets stored in:

* Heroku Config Vars
* Not hard-coded in workflow

WHY:

* Prevents source exposure
* Enables rotation without workflow edits
* Follows 12-factor app methodology

---

## mTLS

Ensures:

* Mutual authentication
* Encrypted transport
* Certificate-bound API access

---

## Execution Logging Control

Reduced logging of:

* Tokens
* Full response bodies

WHY:
Tokens in logs are a high-risk vector.

---

# 11. Polling & Certificate Lifecycle

Certificate issuance is asynchronous.

Implemented:

* GET `/v2/certificates/{serial}`
* Loop until `status = ISSUED`
* Controlled wait interval

WHY:
Avoids premature retrieval attempts.

---

# 12. Trust Chain Assembly

GET:

```
/v2/trustchain
```

Then:

```
fullchain = leaf + intermediates + root
```

WHY:
Most web servers require full chain for TLS trust.

---

# 13. Operational Considerations

* Execution retry logic
* Token expiration awareness
* API timeout settings
* Idempotency awareness

---

# 14. Advantages of Automation

## Security

* Eliminates manual secret exposure
* Reduces human formatting errors
* Enforces CSR integrity

## Scalability

* One workflow handles unlimited domains
* Suitable for multi-tenant environments

## Reliability

* Deterministic CSR formatting
* Controlled polling
* Structured error handling

## Compliance

* Repeatable process
* Auditable execution history

---

# 15. Risk Analysis & Mitigation

| Risk            | Mitigation                  |
| --------------- | --------------------------- |
| CSR corruption  | Deterministic normalization |
| Token leakage   | Reduced logging             |
| Secret exposure | Heroku Config Vars          |
| Endpoint misuse | Explicit URL versioning     |
| API downtime    | Poll + retry logic          |

---

# 16. Performance & Reliability

* CSR normalization: O(n)
* Chain assembly: O(n)
* Polling interval adjustable
* Stateless workflow execution

---

# 17. Conclusion

This automation successfully transforms manual certificate issuance into:

* Deterministic
* Secure
* Scalable
* Reproducible
* Production-ready workflow

Major challenges solved:

* PEM encoding compliance
* Header authentication correctness
* Response structure handling
* Trust chain construction
* Secure secret management

The final system aligns with modern DevSecOps best practices.

---

# 18. Future Improvements

* Automated renewal scheduler
* Secret manager integration (Vault/KMS)
* Multi-environment routing (staging vs prod)
* Slack/Email notification integration
* Kubernetes secret auto-update
* ACME fallback integration
* Observability dashboard

---

# Final Outcome

End-to-end automated certificate issuance pipeline:

✔ Login
✔ CSR validation
✔ Certificate issuance
✔ Status polling
✔ Leaf retrieval
✔ Trustchain retrieval
✔ Fullchain generation
✔ Secure secret handling

System is production-ready and scalable.

