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
| RPM Devices | Various remote monitoring devices | Blood pressure, glucose, weight | Track clinical responses to treatment and monitor chronic conditions remotely. |

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

| Time (UTC) | Stage | Description | Notes |
|------------|-------|-------------|-------|
| 00:00 – 00:40 | Data Ingestion, Standardization & Compression (Lambda) | Pull wearable, RPM, mHealth, and clinical metadata per region; compress and store in S3 raw zone. | 3 Lambdas per region run concurrently; includes cold start overhead. |
| 00:40 – 04:55 | ETL / Parquet Conversion & Regional Aggregation (Glue) | Transform JSON → Parquet; compute weekly aggregates per user; aggregate by treatment program per region; run local sanity checks. | 3 regional Glue jobs run in parallel, each ~3.5 h. Feature extraction removed; aggregation included. |
| 04:55 – 05:10 | Reporting & Dashboards | Generate clinician and research dashboards; push alerts or summaries. | Parallel per region. |

---

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

| Layer | Baseline Architecture (Naïve) | Chosen Architecture (Compliant & Cost-Constrained) |
|--------|-------------------------------|----------------------------------------------------|
| **1. Data Ingestion** | Single centralized S3 bucket (e.g., `global-health-data`) where all raw data (Canada, EU, Brazil) is uploaded. | Three **regional S3 buckets**: <br>• `s3://health-canada-raw/` (ca-central-1) <br>• `s3://health-eu-raw/` (eu-west-1) <br>• `s3://health-brazil-raw/` (sa-east-1). Each region ingests locally to preserve data residency. |
| **2. ETL / Processing** | One AWS Glue job (in `us-east-1`) processes and converts all data to Parquet format. No compression or schema optimization applied — large storage footprint. | Separate **AWS Glue jobs per region** (1 DPU each), run sequentially via Step Functions to control costs and respect regional boundaries. Converts data to **partitioned Parquet** with **Snappy compression**, stored in `s3://health-<region>-processed/`. |
| **3. Workflow Orchestration** | Minimal orchestration; manual or single Lambda trigger. | **AWS Step Functions** orchestrate the pipeline end-to-end (Lambda → Glue → aggregation). Includes **step-level retries**, **idempotent runs**, and clear failure states. |
| **4. Security / Privacy Controls** | Global IAM roles and a single KMS key. | Region-specific **KMS encryption keys**, least-privilege IAM policies, and **data-at-rest + in-transit encryption** per region. |
| **5. Consent & Deletion Management** | Manual deletions or ad-hoc database updates. | Automated via **`DeletionRequestLog`** + Lambda cleanup job per region. Supports consent withdrawal and deletion SLAs (7–30 days depending on regulation). |
| **6. Monitoring & Cost Control** | Only global CloudWatch metrics. | **CloudWatch dashboards per region** track p95 completion, Glue costs, and SLA compliance. **Billing alerts** ensure spend stays ≤ $500 / month. |
| **7. Compliance Alignment** | Violates GDPR/LGPD/OCAP by moving personal data across regions. | Fully compliant with **GDPR Art. 44**, **LGPD Art. 33**, **PIPEDA (residency & consent)**, and **OCAP®** (Indigenous data sovereignty). |
| **8. Performance Target** | Single job handling all data → longer runtimes. | Sequential per-region processing with optimized Glue jobs → **p95 < 7 hours** (fits 00:00 – 06:00 window). |
| **9. Cost Efficiency** | Centralized processing with unpredictable cost scaling and larger S3 storage due to uncompressed files. | 3 Glue jobs × 1 DPU each; sequential schedule → **≈ $198/month Glue + $100 Lambda/Step Functions + $180 S3 storage = <$500/month total.** Parquet compression reduces S3 costs by ~60–70%. |

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

The following subsections check that the proposed architecture fits with the SLA requirements:

### Batch Window Math
The required DPU sizing is based on the compressed volume the Glue job reads ($136.53 \text{ GB}$ per region), ensuring the 5-hour P95 SLA is met.

| Metric | Value | Calculation | Notes |
| :--- | :--- | :--- | :--- |
| **Raw Data Per Region** ($V_{raw}$) | $\approx 682.67 \text{ GB}$ | $2,048 \text{ GB} / 3 \text{ regions}$ | Volume written by the Lambda Router. |
| **Compressed Data Per Region** ($V_{comp}$) | $\approx 136.53 \text{ GB}$ | $682.67 \text{ GB} / 5$ | Volume Glue reads from S3 (assuming 5x compression). |
| **Compressed Data in MB** ($V_{MB}$) | $139,809 \text{ MB}$ | $136.53 \text{ GB} \times 1,024 \text{ MB/GB}$ | |
| **P95 Target Time** ($T$) | $12,600 \text{ s}$ | $3.5 \text{ hours} \times 60 \text{ s/hr}$ | Maximum time allowed for ETL (P95 SLA). |
| **Required Sustained Throughput** ($R$) | **$11.10 \text{ MB/s}$** | $139,809 \text{ MB} / 12,600 \text{ s}$ | Required speed to meet 3.5-hour SLA per region. |
| **Required vCPUs** ($C_{vCPU}$) | $\approx 3.7$ | $11.10 \text{ MB/s} / 3 \text{ MB/s/vCPU}$ | Based on $3 \text{ MB/s}$ per vCPU assumption. |
| **Required DPUs Per Region** ($D_{req}$) | **1 DPU** | $\lceil 3.7 \text{ vCPUs} / 4 \text{ vCPUs/DPU} \rceil$ | **1 DPU is sufficient** as it provides 4 vCPUs. |

The required configuration is **1 DPU per region** ($3 \text{ regions} \times 1 \text{ DPU} = 3$ total concurrent DPUs).

1.  **Worst-Case DPU-Hours per Run (Total):**
    $$3 \text{ DPUs} \times 5 \text{ hours} = \mathbf{15 \text{ DPU-Hours/Run}}$$

2.  **Total Monthly DPU-Hours (30 runs):**
    $$15 \text{ DPU-Hours/Run} \times 30 \text{ Runs/Month} = \mathbf{450 \text{ DPU-Hours/Month}}$$

### Estimated Monthly Costs
Our estimate is ~$440/month, which meets our cost SLA. Please reference [Architecture.md](Architecture.md) for a detailed cost breakdown.

### Meeting p95 SLA

**Assumptions**:
- Lambda cold-start (p50) ≈ **0.5 s**, p95 ≈ **2.0 s** is neglible
- From calculations below, we use 3 lambdas per region to achieve 40 minute p95.

| Throughput per Lambda | Workers required (digit-by-digit) | Round up |
|----------------------:|----------------------------------:|--------:|
| 30 MB/s | workers = 667000 ÷ (30 × 2400) = 667000 ÷ 72,000 = 9.2639 | **10 workers** |
| 60 MB/s | workers = 667000 ÷ (60 × 2400) = 667000 ÷ 144,000 = 4.6319 | **5 workers** |
| 100 MB/s | workers = 667000 ÷ (100 × 2400) = 667000 ÷ 240,000 = 2.7792 | **3 workers** |

- We have also previously shown that Glue jobs can finish in 3.5 hours using our target number of DPUs (1 per region)

Since retries should only be performed for small batches/per file basis, they should not add a significant amount of latency. Furthermore, there is ~1.8 hours of leeway should there be retries before meeting the 6 hour deadline.


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
