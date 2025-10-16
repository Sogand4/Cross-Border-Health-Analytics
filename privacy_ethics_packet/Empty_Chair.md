## Empty Chair Perspective — Indigenous Data Steward (OCAP®)

**Scenario:**  
A developer turns on **cross-region replication** in AWS to make a backup of the Canadian S3 bucket.  
This accidentally makes copies of some health data outside of Canada.

**Stakeholder (Empty Chair):**  
Stakeholder: An Indigenous community data steward.  
They are not in the room when this decision happens, but their community’s rule (OCAP®) says their data must stay in Canada and under their control.

**Why It Matters:**  
If Indigenous-tagged health data is copied to another country, it breaks the community’s trust and their right to control how their data is stored and shared.

### Control
| Control | Where It Happens | What It Does |
|----------|------------------|--------------|
| Keep Indigenous data only in the Canada region (`ca-central-1`). | S3 bucket settings | Stop any data from being copied to another AWS region. |
| Watch for changes to replication settings. | AWS monitoring tools | Send an alert if anyone tries to turn on replication or copy data. |

### Acceptance Test
**Goal:** Make sure no Indigenous data is stored or copied outside Canada.  

**How to Check:**  
- AWS automatically checks that all data tagged with `community='Indigenous'` stays in the Canada region.  
- If any file shows up in another region, an alert is sent to the compliance team.

**What Passing Looks Like:**  
- No alerts are triggered.  
- All Indigenous data stays in `ca-central-1`.

---

## Empty Chair Perspective — EU Patient (GDPR)

**Scenario:**  
A developer schedules a nightly ETL job in AWS Glue that processes user data across regions.  
They forget to filter out records for patients who requested deletion earlier in the day.

**Stakeholder (Empty Chair):**  
Stakeholder: An EU patient who requested deletion of their personal health data.  
They are not in the room when this decision happens, but GDPR Article 17 requires their personal data be deleted promptly.

**Why It Matters:**  
If the ETL job processes data that should have been deleted, the organization violates GDPR, risking legal penalties and eroding patient trust.

### Control
| Control | Where It Happens | What It Does |
|----------|-----------------|--------------|
| Filter ETL input by deletion requests. | AWS Glue Job Scripts | Ensures any patient data flagged for deletion is excluded from processing. |
| Track deletion requests and ETL runs. | CloudWatch + DynamoDB / metadata logs | Confirms each request is honored before processing; raises alerts if patient data is found in ETL input. |

### Acceptance Test
**Goal:** Make sure all patient deletion requests are honored before any ETL processing.

**How to Check:**  
- Query logs to ensure records with deletion flags do not enter Glue transformations.  
- Run automated checks comparing requested deletions vs processed data daily.

**What Passing Looks Like:**  
- No patient records that requested deletion are included in ETL outputs.  
- Alerts remain clear, confirming compliance with GDPR deletion timelines.