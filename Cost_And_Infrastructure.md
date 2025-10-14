*For a full cost and architectural breakdown, see [Architecture.md](./Architecture.md).*

# Cost & Infrastructure Plan  
**Scenario:** Cross-Border Health Analytics (Canada / EU / Brazil)  
**Goal:** Process 2 TB nightly ingest and aggregation under **$500 USD / month**, completing before 06:00 local time.

---

## Estimated Daily & Monthly Cost

| Component | Daily Cost (USD) | Monthly Cost (USD) | Key Driver |
|------------|-----------------|--------------------|-------------|
| **API Gateway (HTTP API)** | $0.26 | $7.80 | 2 M PUT requests per night across 3 regions |
| **AWS Lambda (Validation + Routing)** | $0.08 | $2.49 | 2 M invocations × 100 ms @ 128 MB |
| **Amazon S3 (Raw + Aggregated Storage + Requests)** | $12.00 | $370.16 | 17 TB compressed data across CA/EU/BR; includes PUT/GET ops |
| **AWS Step Functions + CloudWatch** | ≈ $0 | $0.10 | < 1 K transitions/month; free tier covers most usage |
| **Athena + QuickSight (Query + Visualization)** | $2.00 | $60.00 | 6 TB/month queries + 3 dashboard users |
| **Total** | **≈ $15 / day** | **≈ $440 / month** | **Below $500 budget ceiling** |

*Assumptions: 2 TB nightly upload, 5× Parquet compression, regional separation (ca-central-1 / eu-central-1 / sa-east-1).*

---

## Kill-Switch & Auto-Response Playbook

| Trigger | Detection Source | Automated Response |
|----------|-----------------|--------------------|
| **Monthly cost > $500 (projected)** | CloudWatch Budget Alert → SNS email | Pause Glue ETL; disable new `/batch-upload` via API Gateway until manual review. |
| **Batch runtime > 6 h SLA** | Step Functions timeout state | Abort job, send alert to operations channel. |
| **Lambda error rate > 1 %** | CloudWatch metric alarm | Roll back latest Lambda deployment, trigger audit. |
| **S3 storage growth > 10 %/day** | S3 Storage Lens metrics | Suspend new ingest; run cleanup and check lifecycle policy. |

---

## Sensitivity & Contingency Analysis

| Scenario | Cost Impact | Result |
|-----------|-------------|---------|
| Compression efficiency drops 50 % | + $40 USD | ≈ $480 USD < ceiling |
| Query volume doubles | + $30 USD | ≈ $470 USD < ceiling |
| Most expensive region (Brazil) +10 % cost increase | + $16 USD | ≈ $456 USD < ceiling |
| Total worst-case estimate | — | **≈ $490 USD < budget limit** |

---

## Infrastructure Notes

- **Serverless stack:** API Gateway → Lambda → S3 → Glue → Athena → QuickSight  
  - Zero idle cost; scales automatically for 2 TB nightly throughput.  
- **Lifecycle policies:**  
  - Raw zone = 20 days TTL → auto-deletion.  
  - Aggregated zone = 2 years retention → trend analysis only.  
- **Regional isolation:**  
  - `ca-central-1`, `eu-central-1`, `sa-east-1` → aligns with PIPEDA, GDPR, LGPD.  
- **Monitoring:**  
  - CloudWatch alarms + SNS notifications for SLA, cost, and error breaches.  
- **Environmental efficiency:**  
  - Parquet compression cuts storage and query cost ≈ 80 %, reducing carbon footprint.

---

**Summary:**  
This infrastructure plan maintains compliance and observability across three jurisdictions while keeping monthly operating cost ≈ **$440 USD**, safely below the $500 ceiling. Kill-switches and cost alerts ensure quick response if batch duration or budget limits are exceeded.
