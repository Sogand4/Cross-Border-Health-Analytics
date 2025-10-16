# Socratic Log

**Context:** Designing a pipeline for patient symptom ingestion, aggregation, and analytics while ensuring privacy and patient control over data.

---

## Prompt 1: How can patients delete their data once it has been aggregated?
**Response / AI Guidance:** Only the raw data needs to be deleted from the source. Aggregated statistics do not need to be deleted as long as it is impossible to identify the patient.

---

## Prompt 2 (Red-Team / Risk Check): Could aggregated data ever be reverse-engineered to reveal individual patients?
**Response / AI Guidance:** Risk exists if group sizes are very small (e.g., <10 patients per aggregation). Mitigation includes enforcing k-anonymity thresholds and suppressing low-count aggregates.

---

## Inflection Point
I initially assumed that once data was aggregated, patient deletion requests could be ignored. The AI highlighted the re-identification risk in small cohorts, prompting a review of aggregation rules. This led to implementing a privacy guardrail enforcing k ≥ 10 for all public or semi-public aggregates.

---

## Evidence
- Aggregation logic in Glue ETL ensuring anonymization  
- CloudWatch logs confirming raw data deletion triggers  
- Test queries in Athena verifying that no single patient can be identified from weekly aggregates  

---

## Outcome
- Raw patient records are deleted on request, triggering lifecycle events in S3.  
- Aggregated data is retained with enforced k-anonymity, ensuring no identifiable information leaks.  
- Privacy guardrails are documented and tied to acceptance tests for compliance audits.

---

## Attributions
- AI suggested the k-anonymity check and flagged reverse-engineering risks  
- Team implemented raw deletion, aggregation logic, and monitoring for compliance  
- Evidence collection and testing performed by team members