# Design Brief

## Scenario
**Cross-Border Health Analytics**: nightly 2 TB ingest from Canada/EU/Brazil; comply with PIPEDA, GDPR, LGPD; honour OCAP® principles for Indigenous data.

We designed a pipeline to ingest 2 TB of sensor and mobile health data from individuals in Canada, the EU, and Brazil. The primary goal of the pipeline is to help medical pratitioners determine whether treatments are improving patient well-being.

## Data Source

| Data Source | Providers | Data Type | Treatment-Response Use |
|-------------|-----------------|----------|-----------------------|
| Wearables | Fitbit, Apple Watch, Garmin | Heart rate, step count, sleep quality, stress level, activity energy | Identify changes in rest/activity patterns post-treatment; detect fatigue or improved mobility. |
| Mobile Health Apps | Symptom diaries | Self-reported pain, mood, sleep, side effects | Correlate well-being with treatment plan. |
| EHR / Clinical Records (linked subset) | Hospital EMRs or national digital health portals | Diagnosis codes | Determine treatment start/end points. |

## Data Volume
| Source Type | Typical Data | Size per User / Day | Users | Raw Volume |
|-------------|-------------|------------------|-------|------------|
| Wearables | Heart rate, steps, sleep data | 70 MB | 25,000,000 | 1.75 TB |
| RPM Devices | Blood pressure, glucose, weight | 20 MB | 5,000,000 | 0.1 TB |
| mHealth Apps | Symptom diaries, self-reports | 6 MB | 10,000,000 | 0.06 TB |
| Clinical linkage & metadata | treatment context | 20 MB | 500,000 | 0.01 TB |
| **Total** |  |  |  | **≈ 1.92 TB** |

## Batch Schedule
Data ingestion starts at 00:00 every day, due by 06:00 (local time for both).

| Time (UTC) | Stage | Description |
|------------|-------|-------------|
| 00:00 – 00:30 | Data Ingestion – Canada | Pull wearable, RPM, mHealth, and clinical metadata into the Canadian landing zone. Validate file integrity and completeness. |
| 00:30 – 01:00 | Data Ingestion – EU | Pull data from EU endpoints. Validate GDPR-compliant consent flags and file integrity. |
| 01:00 – 01:30 | Data Ingestion – Brazil | Pull data from Brazilian endpoints. Validate LGPD-compliant consent and file integrity. |
| 01:30 – 02:00 | Data Validation & Schema Normalization | Standardize all regional data into Open mHealth / FHIR schema. Add treatment and jurisdiction metadata. Flag missing or corrupt data. |
| 02:00 – 02:30 | Pseudonymization & Consent Enforcement | Apply salted hashes to patient/device IDs. Filter records based on consent and OCAP® rules. |
| 02:30 – 03:30 | Feature Extraction – Regional Compute | Compute daily well-being metrics (resting HR, sleep efficiency, step counts, self-reported scores). Store in regional analytics tables. |
| 03:30 – 04:30 | Regional Aggregation | Aggregate data by treatment program. Run local sanity checks and prepare data for federated analytics. |
| 04:30 – 05:30 | Federated Analytics / Model Updates | Combine aggregates or model weights across regions. Only aggregate/model outputs are shared; no raw data crosses borders. |
| 05:30 – 06:00 | Reporting & Dashboards | Generate clinician and research dashboards. Push alerts or summaries as needed. |

## Compliance Promises

| # | Compliance Area | Promise | How This is Accomplished |
|---|-----------------|--------|-------------------------|
| 1 | Data Ownership and Control (OCAP®) | Data will remain in its region of origin and explicit consent will be obtained from Indigenous communities before analytics is applied or data is shared. | Only aggregated or de-identified data are shared with medical practitioners. |
| 2 | Lawful Basis for Processing (GDPR + LGPD) | Explicit patient consent must be obtained prior to any processing. | Consent is collected from patients; validated before analysis or aggregation; consent revocation is permitted and data removed accordingly. |
| 3 | Data Minimization and Purpose Limitation (GDPR + LGPD) | Only collect data necessary for evaluating treatment response. | Restrict ingestion to relevant wearables and patient-reported outcomes; during validation and standardization, only necessary data is processed; purpose for each data source is documented and explained in terms of service. |
| 4 | Data Residency and Cross-Border Rules (GDPR, LGPD, OCAP®) | Health data will not leave its region. | Keep Canada, EU, and Brazil data in local regional storage; cross-border analytics performed using federated methods or aggregates only. |
| 5 | Pseudonymization & Security | Protect patient identities and ensure data confidentiality. | Apply salted hashes/pseudonyms to all patient/device IDs; encrypt data in transit and at rest; enforce role-based access controls; prevent re-identification. |
| 6 | Rights of the Data Subject (GDPR + LGPD) | Patients can access, correct, delete, or export their personal data. | Maintain mechanisms to locate and remove patient data on request; track modifications and deletions; maintain audit logs. |

If users have revoked consent, the pipeline will delete their data from raw dataset. We will also check if the cohort is small and recalculate any aggregate statistics as neede (e.g. treatment cohort for a rare disease). This will prevent re-identification.

**Retention**
- **Deletion jobs** triggered by Lambda must complete within **7 days (p95)** of user request, verified via CloudWatch metrics and acceptance tests.

### Deleting patient data
1. Tag all data with **patient ID + consent**
2. If a deletion request is received:
    - Remove patient's data from regional raw storage (S3)
    - Check aggregate size and schedule a recompute if necessary, to exclude the patient's data
3. Keep audit logs of deletion actions to demonstrate compliance.

## Baseline vs. Improved Architecture
## Architecture Comparison

| Layer | Baseline Architecture (Naïve) | Chosen Architecture (Compliant & Cost-Constrained) |
|--------|-------------------------------|----------------------------------------------------|
| **1. Data Ingestion** | Single centralized S3 bucket (e.g., `global-health-data`) where all raw data (Canada, EU, Brazil) is uploaded. | Three **regional S3 buckets**: <br>• `s3://health-canada-raw/` (ca-central-1) <br>• `s3://health-eu-raw/` (eu-west-1) <br>• `s3://health-brazil-raw/` (sa-east-1). Each region ingests locally to preserve data residency. |
| **2. ETL / Processing** | One AWS Glue job (in `us-east-1`) processes and converts all data to Parquet format. No compression or schema optimization applied — large storage footprint. | Separate **AWS Glue jobs per region** (1 DPU each), run sequentially via Step Functions to control costs and respect regional boundaries. Converts data to **partitioned Parquet** with **Snappy compression**, stored in `s3://health-<region>-processed/`. |
| **3. Workflow Orchestration** | Minimal orchestration; manual or single Lambda trigger. | **AWS Step Functions** orchestrate the pipeline end-to-end (Lambda → Glue → aggregation). Includes **step-level retries**, **idempotent runs**, and clear failure states. |
| **4. Aggregation / Analytics** | Central Redshift or Athena dataset containing global data. | **Federated Athena or Glue Data Catalog views** operate on **regional aggregates only** — no raw cross-border transfers. |
| **5. Security / Privacy Controls** | Global IAM roles and a single KMS key. | Region-specific **KMS encryption keys**, least-privilege IAM policies, and **data-at-rest + in-transit encryption** per region. |
| **6. Consent & Deletion Management** | Manual deletions or ad-hoc database updates. | Automated via **`DeletionRequestLog`** + Lambda cleanup job per region. Supports consent withdrawal and deletion SLAs (7–30 days depending on regulation). |
| **7. Monitoring & Cost Control** | Only global CloudWatch metrics. | **CloudWatch dashboards per region** track p95 completion, Glue costs, and SLA compliance. **Billing alerts** ensure spend stays ≤ $500 / month. |
| **8. Compliance Alignment** | Violates GDPR/LGPD/OCAP by moving personal data across regions. | Fully compliant with **GDPR Art. 44**, **LGPD Art. 33**, **PIPEDA (residency & consent)**, and **OCAP®** (Indigenous data sovereignty). |
| **9. Performance Target** | Single job handling all data → longer runtimes. | Sequential per-region processing with optimized Glue jobs → **p95 < 7 hours** (fits 00:00 – 06:00 window). |
| **10. Cost Efficiency** | Centralized processing with unpredictable cost scaling and larger S3 storage due to uncompressed files. | 3 Glue jobs × 1 DPU each; sequential schedule → **≈ $198/month Glue + $100 Lambda/Step Functions + $180 S3 storage = <$500/month total.** Parquet compression reduces S3 costs by ~60–70%. |

---

## Failure/Retry Policy
In order to ensure idempotency, each Lambda/Glue task to not depend on previous runs by writing to consisent S3 Key patterns (e.g. `/region/date/device_id.parquet`) and `upsert` is used to add data to analytics tables. Exceptions are thrown on a per file basis, so retries are scoped to that single file. 

We also use the **fail-fast rule:** jobs exceeding 3 retries abort and raise an alert to operations.

A more detailed retry policy is outlined below:

| Stage | Component | Responsibility | Retry Policy | Backoff / Interval | Notes |
|-------|-----------|---------------|-------------|-----------------|-------|
| Ingestion – Canada | Lambda | Pull raw data from wearables, RPM devices, mHealth apps, clinical metadata | Retry up to 3 times for transient errors | Interval: 60s, BackoffRate: 2 | Validate data; idempotent; permanent errors routed to failure branch |
| Ingestion – EU | Lambda | Pull raw data from EU sources | Retry up to 3 times | Interval: 60s, BackoffRate: 2 | Validate GDPR consent; idempotent |
| Ingestion – Brazil | Lambda | Pull raw data from Brazil sources | Retry up to 3 times | Interval: 60s, BackoffRate: 2 | Validate LGPD consent; idempotent |
| Data Validation & Standardization | Lambda | Validate schema, filter unnecessary fields | Retry 2–3 times | Interval: 60s, BackoffRate: 2 | Corrupt/missing records sent to failure branch |
| ETL / Parquet Conversion | AWS Glue | Convert raw data to Parquet, standardize schema | Retry up to 2 times | Interval: 120s, BackoffRate: 2 | Use Glue job bookmarks to prevent duplicate processing |
| Regional Aggregation | AWS Glue / Lambda | Aggregate metrics by cohort or treatment program | Retry 2 times | Interval: 60s, BackoffRate: 2 | Partial failures reprocess only affected partitions |

## Service Level Agreement (SLA)
**Batch Performance**
- p95 completion time: **< 6 hours** per nightly run, including automatic retries and AWS Glue cold start 
- End-to-end target: all 2 TB of uploads validated, transformed, and aggregated **before 06:00 local time** in each region.  
- Step Functions monitors runtime and triggers an alert if any job exceeds **4 hours**.

**Cost Boundaries**
- Total cost <$500, with a daily target **< $16 per run** (≈ $440/month projected)
- CloudWatch Budgets monitor cumulative spend and trigger SNS notifications if forecast > $480.


Here is a more detailed table outlining how SLAs are measured:

| SLA Metric | Target | Measurement | Notes / Justification |
|------------|--------|------------|---------------------|
| **Pipeline Completion Time (p95)** | ≤ 6 hours per nightly run | Measure duration from Step Functions start → all regional Glue jobs & aggregation complete | Ensures data is processed and available for analytics each morning; p95 accounts for 95th percentile of runs to tolerate occasional transient delays |
| **Cost per Nightly Run** | ≤ $16–20 per region (~$50–60 total) | Track DPU-hours for Glue, Lambda invocations, and Step Functions usage | Stays within $500/month budget; cost includes compute and S3 storage for active pipeline |
| **Data Availability for Analytics** | ≥ 99.5% of ingested data available for dashboards/reports | Compare ingested files vs successfully processed Parquet tables | Ensures minimal data loss; supports treatment response analysis |
| **Compliance / Regulatory Deadlines** | Data processing and retention adhere to **PIPEDA, GDPR, LGPD, OCAP®** | Audit logs, consent checks, and regional storage compliance | Must meet regulator-imposed timelines for data handling; ensures Indigenous and patient consent honored |
| **Error Handling / Retry Success Rate** | ≥ 95% of transient errors resolved via Step Functions retries without manual intervention | Track retry attempts and failed file partitions | Ensures high pipeline reliability while respecting idempotency and cost constraints |
| **Monitoring & Alerting** | Failures detected within 5 minutes | CloudWatch alarms, Step Functions execution status, SNS notifications | Enables quick intervention for permanent failures; supports SLA adherence |

### Batch Window Math
The required DPU sizing is based on the compressed volume the Glue job reads ($136.53 \text{ GB}$ per region), ensuring the 5-hour P95 SLA is met.

| Metric | Value | Calculation | Notes |
| :--- | :--- | :--- | :--- |
| **Raw Data Per Region** ($V_{raw}$) | $\approx 682.67 \text{ GB}$ | $2,048 \text{ GB} / 3 \text{ regions}$ | Volume written by the Lambda Router. |
| **Compressed Data Per Region** ($V_{comp}$) | $\approx 136.53 \text{ GB}$ | $682.67 \text{ GB} / 5$ | Volume Glue reads from S3 (assuming 5x compression). |
| **Compressed Data in MB** ($V_{MB}$) | $139,809 \text{ MB}$ | $136.53 \text{ GB} \times 1,024 \text{ MB/GB}$ | |
| **P95 Target Time** ($T$) | $18,000 \text{ s}$ | $5 \text{ hours} \times 3,600 \text{ s/hr}$ | Maximum time allowed for ETL (P95 SLA). |
| **Required Sustained Throughput** ($R$) | **$7.77 \text{ MB/s}$** | $139,809 \text{ MB} / 18,000 \text{ s}$ | Required speed to meet 5-hour SLA per region. |
| **Required vCPUs** ($C_{vCPU}$) | $\approx 2.59$ | $7.77 \text{ MB/s} / 3 \text{ MB/s/vCPU}$ | Based on $3 \text{ MB/s}$ per vCPU assumption. |
| **Required DPUs Per Region** ($D_{req}$) | **1 DPU** | $\lceil 2.59 \text{ vCPUs} / 4 \text{ vCPUs/DPU} \rceil$ | **1 DPU is sufficient** as it provides 4 vCPUs. |

The required configuration is **1 DPU per region** ($3 \text{ regions} \times 1 \text{ DPU} = 3$ total concurrent DPUs).

1.  **Worst-Case DPU-Hours per Run (Total):**
    $$3 \text{ DPUs} \times 5 \text{ hours} = \mathbf{15 \text{ DPU-Hours/Run}}$$

2.  **Total Monthly DPU-Hours (30 runs):**
    $$15 \text{ DPU-Hours/Run} \times 30 \text{ Runs/Month} = \mathbf{450 \text{ DPU-Hours/Month}}$$

Based on a typical AWS Glue rate ($\approx \$0.44/\text{DPU-Hour}$), the estimated monthly cost is:
$\$0.44/\text{DPU-Hour} \times 450 \text{ DPU-Hours} \approx \mathbf{\$198 \text{ per month}}$

### Estimated Monthly Costs (~$289/month -> Meets SLA)
To compute an upper bound, we assumed 5 hours of DPU operation time. Our estimate is ~$289/month, which meets our cost SLA.

| Component | Assumptions | Estimated Monthly Cost | Notes |
|-----------|------------|----------------------|-------|
| **S3 Storage (Standard)** | 2 TB total (~0.67 TB per region) | $47 | $0.023/GB × 2,048 GB; includes raw + processed data |
| **S3 Requests** | 1M PUT/GET per month | $6 | PUT/POST/COPY ~ $0.005/1,000; GET ~ $0.0004/1,000 |
| **AWS Lambda** | 3 functions per region, 1,000 invocations per night, 512 MB memory, 2 s avg runtime | ~$25 | 3 regions × 3 functions × 30 nights × compute cost (~$0.00001667/GB-s) |
| **AWS Glue** | 3 regional jobs, nightly ETL, estimated $198/month | $198 | Adjusted for chosen DPU configuration and runtime; main cost driver |
| **Step Functions** | 1 execution per region per night, 20 state transitions | ~$3 | 3 regions × 30 nights × 20 states × $0.025 per 1,000 state transitions |
| **CloudWatch Monitoring** | Logs + metrics for Lambda & Glue | ~$10 | Estimate for 2 TB nightly logs; minimal cost with log retention |
| **Total** |  | **≈ $289 / month** | Uunder $500 budget; leaves room for scaling or additional analytics |

### p95 Estimate (5 h 15/night -> Meets SLA)

| Step | Stage | Stage p95 (minutes) | Cumulative minutes | Cumulative h : m |
|------|-------|---------------------:|-------------------------------------:|------------------:|
| 1 | Ingestion (Lambda) | 40 | 40 = 40 | 0:40 |
| 2 | Validation & Standardization (Lambda) | 45 | 40 + 45 = 85 | 1:25 |
| 3 | ETL / Parquet Conversion (Glue) — tuned | 120 | 85 + 120 = 205 | 3:25 |
| 4 | Feature Extraction (Glue) — tuned | 45 | 205 + 45 = 250 | 4:10 |
| 5 | Regional Aggregation — tuned | 40 | 250 + 40 = 290 | 4:50 |
| 6 | Reporting & Dashboards | 25 | 290 + 25 = 315 | **5:15** |

**Final p95: 315 minutes = 5 hours 15 minutes.**

**Notes:**  
- Assumes regions run in **parallel**
- Retries included in estimate

## Cloud Services Justification
- We selected AWS Glue Batch compute primarily because we do not need to pay for continuous compute. An EC2-based Spark cluster would have higher operational overhead and will require us to pay for it to run 24/7 when we only needed 6 hours of runtime nightly. Kinesis + Lambda/Glue Streaming would also require continuous compute costs, which does not work with our tight budget. Other reasons we picked Glue include:
    - Supports schema validation and Parquet conversion which fits our desired storage format.
    - Supports **job bookmarks** which ensures already-processed partitions are skipped on retries. This prevents duplicate processing and meets our retry/failure handling goals.
    - We can tune Glue jobs (select # DPUs per job) to balance runtime versus cost, which was important to help us achieve the $500/month cap.
    - There are no idle compute costs. Jobs are only run when triggered by Step Functions.
- Step Functions were chosen for orchestration because Glue integrates seamlessly. It also allows us to use a centralized retry logic and sequential orchestration across regions, without extra infrastructure.
    - Step functions also alllow coordination of Glue Jobs for all regions in a single workflow
    - The execution graph shows workflow progress, success, failure and retries which helps the pipeline be more observable
- We used Amazon S3 Standard for all raw and processed data due to the ability to enforce per-region buckets, encrpytion, and its cost effectiveness (~$47/month across all regions)for short-term use. Data is retrieved nightly, so Standard level avoids retrieval fees associated with IA or Glacier Tiers.
- For monitoring, we chose CloudWatch which is useful for searching, filtering and specifying retention policies. It also integrates well with our other chosen services.


### Kill Switch

| Mechanism | Implementation (in AWS) | Trigger Condition |
| :--- | :--- | :--- |
| **Ingestion Kill Switch** | **Disable the API Gateway / HTTP API.** | Detection of unauthorized access or a systemic failure in the **Lambda Router** that could lead to cross-regional data leakage or storage of unencrypted PII. |
| **Processing Kill Switch** | **Pause the AWS Step Functions scheduler (CloudWatch/EventBridge Rule).** | Systemic Data Quality Failure (e.g., Glue jobs repeatedly corrupting data or failing schema validation). Prevents cost overruns and pollution of the data lake. |
| **Data Access Kill Switch** | **Deny all external IAM roles access to the `S3 Aggregated Zone`.** | Suspected security incident or need to immediately isolate the stored PHI/PII from analysts/tools. |
