# Design Brief

## Scenario
**Cross-Border Health Analytics**: We designed a cross-regional data ingestion pipeline to collect and process self-reported health symptoms and metrics from mobile/wearable devices. The data is validated, aggregated, and stored in compliance with regional privacy regulations, enabling later analysis to detect emerging health risks and provide users with actionable insights.

For example, if a user’s wearable device indicates low daily activity, poor sleep patterns, and the user reports persistent lethargy, the system can combine these signals to identify a potential health risk. The user would then receive a personalized alert recommending actions such as increasing physical activity, improving sleep habits, or consulting a healthcare professional.

## Data Source/Volume
| Source  | Type  | Data Collected  | Regional Considerations                                                                 |
|-------------------------|------------|--------------------------------------------------|------------------------------------------------------------------------------------------------|
| Mobile App Submissions  | App / API  | User-reported symptoms, date/time, optional demographics | Already regionalized: Canada, UK, Brazil. Data stays in local S3 buckets for compliance.       |
| Wearable / Mobile Devices  | Device / API | Heart rate, body temperature, hours of sleep, daily step count | Devices paired to users’ mobile apps; ingestion via secure APIs. Data should also be regionally stored to respect jurisdictional privacy laws. |

**Data Volume**: We anticipate a total daily ingest volume of 2 TB per night, with a roughly equal split of 667 GB each from Canada, the EU, and Brazil.

## Batch Schedule
Data ingestion starts at 00:00 every day, due by 06:00 (local time for both).

| Time (Local) | Event / Batch Step | Region(s) | Runtime / Volume (Estimate) | Notes / Actions |
| :--- | :--- | :--- | :--- | :--- |
| **00:00** | **Data Collection Cutoff** | All | - | End of the 24-hour cycle for the day's activity and sleep data. Raw data is collected/staged over the next 20 minutes. |
| **00:20** | **STRICT START: Step Function Trigger** | All (EU, BR, CA) | - | **EventBridge** triggers the regional **Step Function** pipelines simultaneously. This is the **Batch Window Start**. |
| **00:20 - 00:40** | **Lambda Validation & Regional Write** | All | 20 min / $\approx 667 \text{ GB}$ per region | **Lambda Router** (first state) performs **schema validation** and writes $\approx 2 \text{ TB}$ of **VALID** raw JSON to **Regional S3 Raw Zone**. |
| **00:45** | **Glue Validation & Deduplication** | All | 10 min | **Glue Job 1:** Deeper validation (data range checks, format) on raw data. |
| **00:55** | **Glue ETL Processing Starts** | All | $\approx 2.5 \text{ hours}$ | **Glue Job 2:** **Transform JSON $\rightarrow$ Parquet** and aggregate. **Core ETL.** |
| **03:25** | **EU Processing Completes** | EU | 2.5 hr / 667 GB Raw | Aggregated Parquet stored in **S3 Aggregated Zone - EU**. **Core ETL complete.** |
| **03:40** | **BR Processing Completes** | BR | 2.75 hr / 667 GB Raw | Aggregated Parquet stored in **S3 Aggregated Zone - BR**. **Core ETL complete.** |
| **03:55** | **CA Processing Completes** | CA | 3 hr / 667 GB Raw | Aggregated Parquet stored in **S3 Aggregated Zone - CA**. **Core ETL complete.** |
| **04:10** | **Pipeline Completion Checkpoint (P95 Target)** | All | - | **Signal completion** to downstream services (e.g., trigger the separate Risk Model job). |
| **05:00** | **P95 SLA Deadline** | All | - | **5 hours from the 00:20 trigger.** This is the target for high-priority completion. |
| **05:40** | **Kill-Switch Deadline** | All | - | **Final Emergency Stop.** Allows 20 minutes of buffer to halt the 5% of outlier jobs before the hard deadline. |
| **06:00** | **HARD 6-HOUR DEADLINE** | All | - | **FINAL COMPLETION.** All ETL processing **must** be complete. |

### Critical Safety Mechanism: Kill Switch (Emergency Stop)

| Mechanism | Implementation (in AWS) | Trigger Condition |
| :--- | :--- | :--- |
| **Ingestion Kill Switch** | **Disable the API Gateway / HTTP API.** | Detection of unauthorized access or a systemic failure in the **Lambda Router** that could lead to cross-regional data leakage or storage of unencrypted PII. |
| **Processing Kill Switch** | **Pause the AWS Step Functions scheduler (CloudWatch/EventBridge Rule).** | Systemic Data Quality Failure (e.g., Glue jobs repeatedly corrupting data or failing schema validation). Prevents cost overruns and pollution of the data lake. |
| **Data Access Kill Switch** | **Deny all external IAM roles access to the `S3 Aggregated Zone`.** | Suspected security incident or need to immediately isolate the stored PHI/PII from analysts/tools. |


## Design Considerations

- **Regional independence:** Each country’s data processed locally first (LGPD, GDPR, PIPEDA compliant)  
- **Compression:** Use Parquet (≈5× compression) to reduce I/O and cost  
- **Scalability:** Schedule via AWS Step Functions or Managed Airflow, tagged by `region`, `dataset`, `batch_id`  
- **Resilience:** Retries and quarantine for partial batch failures  
- **Monitoring metrics:**  
  - Batch duration ≤ 240 min  
  - Record drop rate ≤ 1 %  
  - Compliance pass rate ≥ 99 %  
  - Cost per batch (tracked for optimization)


## Service Level Agreement (SLA)
**Batch Performance**
- p95 completion time: **< 5 hours** per nightly run, including automatic retries and AWS Glue cold start 
- End-to-end target: all 2 TB of uploads validated, transformed, and aggregated **before 06:00 local time** in each region.  
- Step Functions monitors runtime and triggers an alert if any job exceeds **3 hours**.

**Cost Boundaries**
- Total cost <$500, with a daily target **< $16 per run** (≈ $440/month projected)
- CloudWatch Budgets monitor cumulative spend and trigger SNS notifications if forecast > $480.

## Batch Window Math
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

#### DPU Cost Implications

The required configuration is **1 DPU per region** ($3 \text{ regions} \times 1 \text{ DPU} = 3$ total concurrent DPUs).

1.  **Worst-Case DPU-Hours per Run (Total):**
    $$3 \text{ DPUs} \times 5 \text{ hours} = \mathbf{15 \text{ DPU-Hours/Run}}$$

2.  **Total Monthly DPU-Hours (30 runs):**
    $$15 \text{ DPU-Hours/Run} \times 30 \text{ Runs/Month} = \mathbf{450 \text{ DPU-Hours/Month}}$$

Based on a typical AWS Glue rate ($\approx \$0.44/\text{DPU-Hour}$), the estimated monthly cost is:
$$\$0.44/\text{DPU-Hour} \times 450 \text{ DPU-Hours} \approx \mathbf{\$198 \text{ per month}}$$


**Compliance & Retention**
- **Deletion jobs** triggered by Lambda must complete within **7 days (p95)** of user request, verified via CloudWatch metrics and acceptance tests.
- **Data residency:** no cross-region replication; each user’s data stays within its jurisdiction (`ca-central-1`, `eu-central-1`, or `sa-east-1`).
- **Consent**: users must give explicit consent to provide metadata and telemetry.
    - Our app provides an opt-in button in the sign up page to explicitly ask users for consent.
    - Users are also given the option to opt-out at any time by navigating to the `settings` section of the app.

**Reliability & Retry Policy**
- **Automatic retry limit:** 3 attempts for transient Lambda or Glue failures.  
- **Fail-fast rule:** jobs exceeding 3 retries abort and raise an alert to operations.
- SLA breaches (runtime, cost, or retention) are logged to the **Ethics Debt Ledger** and reviewed weekly.

**Regulatory Deadlines**
- **Canada (PIPEDA):** Deletion or consent withdrawal requests are processed within 7 days, based on our internal SLA target that interprets PIPEDA’s requirement to act “promptly.” This ensures user requests are handled in a timely and measurable way, even though the regulation does not define an exact deadline.
- **EU (GDPR Art. 12–17):** Data-subject requests (access, deletion, correction) completed within **30 days**; monitored through CloudWatch deletion metrics.  
- **Brazil (LGPD Art. 18):** Data-subject deletion requests honored within **7 days**, logged to `DeletionRequestLog` for audit.  
- All regions: nightly aggregation jobs must complete **before 06:00 local time** to ensure regulators could audit fresh, up-to-date datasets on request.

TODO modify the architecture diagram to include EventBridge, ingest data from mobile device/wearable device, move data out from centralized data lake, and compress S3 data