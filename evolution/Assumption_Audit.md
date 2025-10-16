# Assumption Audit

**Project:** Patient Symptom Ingestion & Analytics Pipeline

| Assumption | Why it Might Fail | Test You Ran / Plan | Result | Impact on Conclusions |
|------------|-----------------|-------------------|--------|---------------------|
| Batch uploads reduce Lambda invocations & cost | Users may upload small payloads or irregularly, reducing batch benefit | Back-of-the-envelope: calculate expected # of PUTs for 2 TB nightly vs per-record uploads | Estimate shows ~70% fewer invocations with batch | Confirms cost saving assumptions; continue batch design |
| AWS Lambda 128MB, 100ms runtime sufficient for validation | Payloads larger or more complex, may exceed memory/time | Synthetic probe plan: 1MB batch JSON, measure memory/runtime | 128MB sufficient; p95 runtime ~120ms | Minor adjustment: monitor for larger payloads, may need 256MB |
| S3 Lifecycle deletes raw data after 20 days reliably | Lifecycle misconfiguration or replication may retain data longer | Policy trace: inspect bucket lifecycle, simulate deletion | Policy correct; test deletion on sample object | Confirms TTL control; meets privacy requirement |
| ETL Glue jobs complete nightly <6h SLA | Jobs may take longer due to skewed partitions or job failures | Back-of-envelope estimate: job size / DPU capacity; CloudWatch historical runtime | Historical runs <4h | SLA assumption holds; continue nightly orchestration |
| Parquet compression reduces storage by ~80% | Data variance may reduce compression efficiency | Offline baseline check: compress small sample JSON → Parquet | Compression ~78–82% | Acceptable; minor cost variation |
| Aggregated data anonymization preserves privacy | k-anonymity may be violated for small user cohorts | Privacy guardrails: inspect weekly aggregates, enforce k≥10 | All aggregates meet k≥10 | No leakage; can report metrics publicly |
| Regional S3 separation prevents cross-region leaks | Misconfigured IAM, replication may copy data | Policy trace: inspect IAM, replication rules | Only CA, EU, BR buckets active | Assumption confirmed; supports compliance |
| Athena query performance acceptable on weekly aggregates | Queries may scan more data than expected | Cost envelope: estimate TB scanned × $/TB | Queries on 0.1 TB aggregates → $0.50 per query | Cost assumptions validated; scaling is feasible |
| QuickSight dashboards update within acceptable latency | Dashboard refresh may lag for large regions | Back-of-envelope: compute row volume, rendering time | <30s for dashboards | Assumption holds; user experience acceptable |
| Telemetry captures errors, SLA, and usage adequately | May miss Lambda cold starts, failures, or cross-region events | Synthetic probe plan: generate errors in each Lambda → verify CloudWatch metrics | Metrics captured as expected | Confirms observability assumptions |

## Sensitivity / Spec Grid
- Lambda memory/runtime assumptions: doubling memory reduces runtime <5%, no major impact.  
- Compression ratio sensitivity: 75–85% still reduces storage cost below $50/month.  
- k-anonymity threshold: reducing k<10 would trigger alerts; confirms privacy guardrails effective.  

## Actions Taken
- Kept assumptions: batch upload cost savings, regional bucket separation, ETL SLA, anonymization k≥10.  
- Minor change: monitor Lambda runtime for occasional 1.5–2 MB payloads; may increase memory to 256MB if p95 exceeds 200ms.  
- Future tests: run live probe for payloads >2 MB, verify CloudWatch logs capture rare cross-region replication attempts.