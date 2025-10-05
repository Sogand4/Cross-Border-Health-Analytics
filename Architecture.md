[Mobile App]


[API Gateway]


[AWS Lambda Router]


[S3 Buckets (CA/EU/BR, Raw → Aggregated)]


[AWS Glue Job (Nightly Transform)]


[S3 Aggregated Zone]


[Athena / QuickSight Dashboards]


---

Design Choice Explanations:

### API Gateway
- Exposes HTTPS `/upload` endpoint for mobile app uploads.
- Triggers Lambda; no caching used (uploads are unique).
- Regional pricing:
  - Canada (Central): $3.50 / 1M requests
  - EU (Frankfurt): $3.70 / 1M requests
  - Brazil (São Paulo): $4.25 / 1M requests
- Example: 2M PUT requests each → ≈ $20.65 / month
- Reference: [AWS API Gateway Pricing](https://aws.amazon.com/api-gateway/pricing/)


---

### AWS Lambda
- Processes each `/upload` request from API Gateway, validates metadata, and routes to the correct S3 bucket.
- Config: 128 MB memory, ~100 ms duration per invocation.
- Regional pricing (x86):
  - Canada (Central): $0.20 / 1 M requests + $0.00001667 per GB-second  
  - EU (Frankfurt): $0.20 / 1 M requests + $0.00001667 per GB-second  
  - Brazil (São Paulo): $0.20 / 1 M requests + $0.00001667 per GB-second
- Example: 2 M requests / region → ≈ $0.82 per region (≈ $2.50 total for 3 regions).
- Reference: [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)


---

### Amazon S3
- Stores **raw (20 days)** and **aggregated (2 years)** sleep data per jurisdiction.  
- **Buckets per region:**
  - Canada (Central): `s3://sleep-raw-ca/`, `s3://sleep-agg-ca/`
  - EU (Frankfurt): `s3://sleep-raw-eu/`, `s3://sleep-agg-eu/`
  - Brazil (São Paulo): `s3://sleep-raw-br/`, `s3://sleep-agg-br/`

- **Storage classes:**
  - Raw → **S3 Intelligent-Tiering**  
    - $0.025 / GB-month (CA), $0.0245 / GB-month (EU), $0.0405 / GB-month (BR)  
  - Aggregated → **S3 Glacier Instant Retrieval**  
    - $0.005 / GB-month (CA, EU), $0.0083 / GB-month (BR)  
  - Requests:  
    - PUT = (0.0055 CA | 0.0054 EU | 0.007 BR) per 1 000 requests  
    - GET = (0.00044 CA | 0.00043 EU | 0.00056 BR) per 1 000 requests  

- **Assumptions:**
  - Total ~17 TB before compression (2 TB raw + 15 TB aggregated across 3 regions)  
  - Per-region: 0.67 TB raw + 5 TB aggregated  
  - Raw TTL = 20 days  
  - Compression = 5×  
  - 2 million PUT + 2 million GET requests per region  

#### Regional Cost Breakdown

**Canada (Central):**
- Raw: (0.67 TB × 20 days × 1 000 GB/TB × $0.025 / 5) ≈ $66.67  
- Aggregated: (5 TB × 1 000 GB/TB × $0.005) = $25.00  
- PUT: 0.0055 × (2 000 000 / 1 000) = $11.00  
- GET: 0.00044 × (2 000 000 / 1 000) = $0.88  
- **Total ≈ $103.55**

**EU (Frankfurt):**
- Raw: $65.33  
- Aggregated: $25.00  
- PUT: $10.80  
- GET: $0.86  
- **Total ≈ $101.99**

**Brazil (São Paulo):**
- Raw: $108.00  
- Aggregated: $41.50  
- PUT: $14.00  
- GET: $1.12  
- **Total ≈ $164.62**

### **Estimated Monthly Total ≈ $370.16 USD**
**Reference:** [AWS S3 Pricing](https://aws.amazon.com/s3/pricing/)

---

### AWS Step Functions + CloudWatch
- Orchestrates the nightly ETL workflow:
  - Start → Ingest → Transform (Glue) → Aggregate → Validate → Alert.
- **CloudWatch** monitors job duration, errors, and cost alarms.  
- Sends **SNS email alerts** if SLA (6 h) breached.

**Trade-offs**
- Visual, auditable workflow for compliance.  
- Built-in retry and timeout policies reduce manual restarts.  
- Minor per-transition cost.

**Assumptions**
- ~11 state transitions per workflow × 3 regions × 30 days ≈ 1 000 transitions/month.  
- First 4 000 transitions / month are **free** (permanent AWS free tier).

**Regional pricing (after first 4000 transitions)**
- Canada (East – ca-central-1): $0.025 / 1 000 transitions  
- Europe (Frankfurt – eu-central-1): $0.025 / 1 000 transitions  
- South America (São Paulo – sa-east-1): $0.0375 / 1 000 transitions  

**Cost estimate**
- 1 000 transitions ≈ $0.03 (CA/EU) or $0.04 (BR).  
- Across all regions: ≈ **$0.10 / month total** (rounded up to $1 budget buffer).  

**Source:** [AWS Step Functions Pricing](https://aws.amazon.com/step-functions/pricing/)

---

### Athena + QuickSight (Analytics / Publish)
- Athena queries aggregated Parquet data directly from S3.  
- QuickSight dashboard visualizes:  
  - Sleep goal completion %  
  - Irregular pattern detection  
  - Trends by age cohort and region.  

**Trade-offs**
- Fully serverless (no database or server to manage).  
- Scales automatically; pay only for what you use.  
- Query latency (seconds) for large scans.  
- Higher cost in South America region.

---

**Athena Pricing**
- SQL queries (per TB scanned):  
  - Canada / EU: **$5 per TB**  
  - Brazil (São Paulo): **$9 per TB**  
- Aggregated data stored as compressed **Parquet** → up to **90% cost savings**.  
- Example: 0.1 TB/query × 60 queries / month = 6 TB scanned → ≈ $30/month (all regions combined).  

---

**QuickSight Pricing**
- **Author** = $24 / user / month (builds dashboards)  
- **Reader** = $3 / user / month (views dashboards)  
- Example: 1 Author + 2 Readers = $30 / month total.  

---

**Estimated Total (Athena + QuickSight)**
- ≈ **$40–60 USD per month** (depending on query volume and compression).  

**Sources:**  
- [AWS Athena Pricing](https://aws.amazon.com/athena/pricing/)  
- [AWS QuickSight Pricing](https://aws.amazon.com/quicksight/pricing/)


---

## Total Estimated Monthly Cost
About $460/month


---

## ⚖️ Key Trade-offs Summary

| Decision | Why | Stakeholder Impact |
|-----------|-----|--------------------|
| **Serverless stack (API Gateway + Lambda + Glue)** | No ops cost, auto-scales | Reliable uploads → user trust |
| **Regional S3 buckets** | Simplifies legal compliance | Regulators see clear residency boundaries |
| **Lifecycle deletion (20 days raw, 2 years aggregated)** | Balance privacy vs. utility | Protects user data while keeping useful summaries |
| **Parquet compression** | Cut cost 60 % | Lower carbon footprint + cheaper storage |
| **Glue over EMR** | Faster setup, less tuning | Devs spend less time on infra |
| **Step Functions monitoring** | Automatic retries + alerting | Maintains SLA ≤ 6 h |

---