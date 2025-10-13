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

## Empty Chair Perspective — Data Scientist Under Time Pressure  

**Scenario:**  
An analyst working on nightly batch jobs wants to speed up model training and temporarily disables data-deletion jobs to avoid delays.  
Old health data remains stored longer than allowed by policy.

**Stakeholder (Empty Chair):**  
Stakeholder: The data scientist running the batch analytics workflow.  
They are not thinking about legal retention limits; their focus is model accuracy and deadlines.

**Why It Matters:**  
If retention limits are ignored, personal health data may remain in storage indefinitely, violating deletion clauses and inflating compliance risk.

### Control
| Control | Where It Happens | What It Does |
|----------|------------------|--------------|
| Lock deletion jobs to run automatically regardless of model schedule. | Step Functions / Orchestrator | Ensures data cleanup cannot be skipped manually. |
| Add retention-limit alerts to dashboard. | CloudWatch / Monitoring | Warns the team if objects older than 7 days remain. |


### Acceptance Test
**Goal:** Confirm that old data is always deleted on schedule.  

**How to Check:**  
- Attempt to pause or skip the deletion step; orchestrator should reject or reschedule it.  
- Query S3 after 7 days and confirm no files older than the policy limit exist.  

**What Passing Looks Like:**  
- Deletion job runs automatically after each batch.  
- No retained data exceeds the allowed age threshold.  