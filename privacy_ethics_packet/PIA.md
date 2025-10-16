# Privacy Impact Assessment (PIA) — Patient Symptom Data Ingestion

## 1. Overview
**Purpose:** Collect, process, and analyze patient symptom data from mobile apps, wearables, RPM devices, and clinical records to generate insights on treatment response.  
**Scope:** Regional ingestion in Canada, EU, and Brazil; serverless architecture with S3, Lambda, Glue, Step Functions, Athena, and QuickSight.  

**Data Processing Summary:**  
- **Collection:** Mobile app uploads, wearables API, RPM device uploads, EHR/clinical records (linked subset).  
- **Processing:** Validation, metadata enrichment, JSON → Parquet transformation, anonymization, aggregation.  
- **Sharing:** Aggregated dashboards per jurisdiction; restricted access per role.  
- **Retention:** Raw data TTL = 20 days; aggregated data retained for 2 years; automated deletion policies.

---

## 2. Data Inventory

| Field | Source | Purpose | Lawful Basis | Minimization | Retention | Access Roles |
|-------|--------|--------|--------------|-------------|-----------|--------------|
| `user_id` | Mobile App, Wearables, RPM, EHR subset | Identify user for per-user aggregation | Consent / HIPAA-equivalent | Pseudonymized / hashed | 20 days (raw), 2 years (aggregated) | Data Engineer, Analyst |
| `timestamp` | Mobile App / Wearable / RPM | Time-series alignment | Consent | Rounded to nearest minute/hour | 20 days / 2 years | Data Engineer, Analyst |
| `symptom` | Mobile App | Health monitoring | Consent | Only required fields collected | 20 days / 2 years | Data Engineer, Analyst |
| `duration` | Mobile App | Symptom analysis | Consent | Rounded to nearest hour | 20 days / 2 years | Data Engineer, Analyst |
| `heart_rate`, `sleep_quality`, `steps` | Wearables | Treatment response monitoring | Consent | Only relevant metrics collected | 20 days / 2 years | Data Engineer, Analyst |
| `blood_pressure`, `glucose`, `weight` | RPM Devices | Vital sign monitoring | Consent | Only required metrics collected | 20 days / 2 years | Data Engineer, Analyst |
| `diagnosis_codes` | EHR subset | Establish treatment periods | Consent / Legal Agreement | Only linked subset | 20 days / 2 years | Data Engineer, Analyst |

---

## 3. Linkability & Identifiability
- **Linking:** Data linked per hashed/pseudonymized `user_id`. Aggregation prevents identification outside jurisdiction.  
- **Quasi-identifiers:** IP, device metadata, and timestamps exist but are pseudonymized; risk mitigated by aggregation and anonymization.  
- **Re-identification Risk:** Minimal for dashboards; raw data TTL and strict access control reduce exposure.

---

## 4. Purpose Limitation & Secondary Use
- **Declared Purposes:** Treatment-response monitoring, clinical research, and patient insights.  
- **Approval for New Purposes:** Any secondary use must be approved by governance board and logged; patient opt-in required.  
- **Function Creep Protection:** Anonymization and regional partitioning prevent unapproved linkage or sharing.

---

## 5. Minimization & Retention
- **Aggregation:** Weekly per-user summaries only; removes daily granularity.  
- **Anonymization:** Pseudonymization of `user_id`, truncation of timestamps, removal of rare quasi-identifiers.  
- **Raw TTL:** 20 days; aggregated retention: 2 years.  
- **Secure Deletion:** Lifecycle policies enforce automatic deletion; S3 versioning disabled for privacy-critical buckets.

---

## 6. Access Control & Governance
- **Roles:**  
  - Data Engineers: write raw & aggregated data, run ETL.  
  - Analysts: query aggregated data via Athena.  
  - Clinicians / Researchers: QuickSight dashboard access.  
- **Least Privilege:** IAM policies enforce access per region and role.  
- **Auditability:** CloudTrail + CloudWatch logging for all S3, Lambda, Glue, Athena accesses.  
- **Third Parties:** None currently. If added, contracts include strict data residency and processing clauses.

---

## 7. Transparency & Choice
- **Communication Plan:** App and portal disclose what is collected, why, and retention period.  
- **Opt-in / Opt-out:** Patients must consent to mobile app, wearable, and RPM data collection; aggregated analytics opt-out supported for research dashboards.

---

## 8. Security
- **Threats:** Abuse, scraping, accidental cross-region replication, doxxing.  
- **Mitigations:**  
  - WAF on API Gateway, rate limiting, batch uploads to reduce request noise.  
  - Transport encryption (HTTPS/TLS 1.2+).  
  - Secrets management via AWS Secrets Manager.  
  - Cross-region replication disabled for sensitive Indigenous/Canadian data.

---

## 9. Compliance & Policy Alignment
- **Policies:** OCAP®, GDPR, PIPEDA, LGPD, HIPAA-equivalent compliance.  
- **Implementation:** Regional buckets, lifecycle management, data minimization, audit logging, encryption at rest & in transit.

---

## 10. Residual Risks & Trade-offs
- **Residual Risks:**  
  - Risk of re-identification from small cohorts.  
  - ETL/Glue misconfiguration exposing raw data outside jurisdiction.  
  - Delays in dashboard updates if Step Functions fail.  
- **Contingency Measures:**  
  - Daily monitoring via CloudWatch/SNS alerts.  
  - Manual intervention plan and SLA enforcement (<6h ETL).  
  - Regular review of anonymization and aggregation policies.

---

# Telemetry Decision Matrix

| Telemetry Item | Source / System | Purpose | Level of Identifiability | Storage Location / Zone | Retention | Access Roles | Notes / Controls |
|----------------|----------------|--------|------------------------|-----------------------|-----------|--------------|----------------|
| Mobile App Logs (API calls, errors) | Mobile app → API Gateway | Monitor ingestion, debug failures, SLA adherence | Low (no user payload, only request metadata) | CloudWatch Logs | 30 days | DevOps, Data Engineer | Mask user IPs; batch logs; rate limiting applied |
| Batch Upload Payload Metadata | API Gateway / Lambda | Verify batch completion, user activity | Medium (hashed user ID) | S3 Raw Zone per region | 20 days | Data Engineer | Payload content anonymized before aggregation; sensitive fields hashed/pseudonymized |
| Lambda Execution Metrics | AWS Lambda | Monitor function runtime, errors, invocations | None | CloudWatch Metrics | 90 days | DevOps | No payload data included |
| Glue Job Metrics | AWS Glue | ETL pipeline monitoring | None | CloudWatch Metrics / Logs | 90 days | Data Engineer | Includes job duration, processed record counts |
| Step Functions Transitions | Step Functions | ETL orchestration monitoring | None | CloudWatch Metrics | 90 days | Data Engineer, DevOps | SLA alerting on >6h completion |
| S3 Object Access Logs | S3 (Raw & Aggregated) | Audit data reads/writes, detect anomalies | Medium (hashed user IDs in paths) | S3 Audit bucket per region | 1 year | Compliance, Data Engineer | Cross-region replication disabled; logging read/write only |
| Athena Query Logs | Athena | Track queries for auditing, cost analysis | Low (aggregated queries) | CloudTrail / S3 | 90 days | Data Engineer, Analyst | Only aggregated data queried; user-level identifiers removed |
| QuickSight Dashboard Access | QuickSight | User activity, audit, licensing | Low (role-based) | QuickSight Audit Logs | 1 year | Compliance, Product Owner | Role-based access only; aggregated results |
| Error & Alert Notifications | SNS / CloudWatch | Incident response | None | CloudWatch / SNS | 30 days | DevOps, Compliance | Alerts on SLA breach, ETL failure, unusual access |

---

## Recourse & Remedy

**Objective:**  
Provide patients and stakeholders with clear mechanisms to address concerns, errors, or unauthorized access related to their health data.

### Patient Recourse
| Issue | Mechanism | Responsible Party | SLA / Response Time |
|-------|-----------|-----------------|------------------|
| Incorrect or incomplete symptom data | Submit request via mobile app “Report an Issue” feature | Data Steward / Data Engineer | Acknowledge within 24h; fix within 72h |
| Unauthorized data access | Contact compliance team via secure email / portal | Compliance Officer | Investigation initiated within 24h; resolution within 7 days |
| Data deletion / retention query | Submit Data Subject Access Request (DSAR) | Data Steward | Response within 30 days (per GDPR / local privacy laws) |
| Discrepancies in aggregated insights | Contact clinical analytics support | Data Analyst / Compliance | Confirm calculation & report back within 5 business days |

### Operational Remedies
| Issue | Remedy | Notes / Controls |
|-------|--------|----------------|
| Cross-region data transfer violation | Immediate suspension of replication; audit logs reviewed; alert to compliance team | AWS region-level guardrails enforced via IAM policies |
| Data breach or exfiltration | Incident response plan triggered; notify affected patients per legal requirements | WAF, encryption at rest/in transit, anomaly detection in CloudWatch |
| ETL or aggregation errors | Re-run nightly Glue jobs with previous raw snapshots; verify aggregates | Versioned S3 raw data retained for 20 days for remediation |
| SLA breach (>6h ETL pipeline) | Step Functions retries; SNS alert triggers on-call engineer | SLA monitoring and escalation policy enforced |

**Key Principles:**  
1. **Transparency:** Patients are notified of any data issue impacting them.  
2. **Timeliness:** Defined response times ensure issues are addressed promptly.  
3. **Auditability:** All requests, investigations, and remediation actions are logged for compliance review.  
4. **Jurisdictional Compliance:** Remedies respect regional data residency requirements (CA, EU, BR).  
5. **Continuous Improvement:** Root causes are tracked and mitigation strategies implemented to prevent recurrence.