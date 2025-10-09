Project 2:

[Project group has been formed and project idea was agreed to]

A symptom tracking app where the user logs their symptoms (eg. runny nose, headacche, etc) per day.

![Architecture](architecture.png)

### (Canada) Clause 1: Consent and Purpose Limitation

**Source:**  
PIPEDA Principle 4.3.2 – “Knowledge and consent are required for the collection, use, or disclosure of personal information.”

---

#### Control

| Control | Enforcement Point | Implementation Notes |
|----------|------------------|----------------------|
| Require explicit consent toggle in the mobile app before first `/batch-upload`. | Mobile App → API Gateway | User must actively grant consent before data collection. Cognito user pool attribute `consent_granted=true` is embedded in the JWT. |
| Deny uploads without valid consent. | Lambda (Authorizer + Validator) | Lambda checks JWT claim `consent=True`; rejects with HTTP 403 if missing. CloudWatch metric `ConsentDeniedCount` raised for compliance audit and alerting. |
Disable background sync unless user explicitly re-enables it after giving consent. | Mobile App | Prevents passive or “proxy-like” uploads when the user assumes their data stays local. Users must opt in again if they revoke or change consent.

---

#### Test Exemplar

```python
def test_upload_without_consent_rejected():
    """Failing test (red bar): verifies that uploads without consent are rejected."""
    resp = client.post("/batch-upload", headers={"Authorization": fake_jwt(consent=False)})
    assert resp.status_code == 403, \
        "Consent control failure: API accepted upload without user consent."

```

---


## Empty Chair Perspective — Indigenous Data Steward (OCAP®)

**Scenario:**  
A developer turns on **cross-region replication** in AWS to make a backup of the Canadian S3 bucket.  
This accidentally makes copies of some health data outside of Canada.

**Stakeholder (Empty Chair):**  
Stakeholder: An Indigenous community data steward.  
They are not in the room when this decision happens, but their community’s rule (OCAP®) says their data must stay in Canada and under their control.

**Why It Matters:**  
If Indigenous-tagged health data is copied to another country, it breaks the community’s trust and their right to control how their data is stored and shared.

---

### Control
| Control | Where It Happens | What It Does |
|----------|------------------|--------------|
| Keep Indigenous data only in the Canada region (`ca-central-1`). | S3 bucket settings | Stop any data from being copied to another AWS region. |
| Watch for changes to replication settings. | AWS monitoring tools | Send an alert if anyone tries to turn on replication or copy data. |

---

### Acceptance Test
**Goal:** Make sure no Indigenous data is stored or copied outside Canada.  

**How to Check:**  
- AWS automatically checks that all data tagged with `community='Indigenous'` stays in the Canada region.  
- If any file shows up in another region, an alert is sent to the compliance team.

**What Passing Looks Like:**  
- No alerts are triggered.  
- All Indigenous data stays in `ca-central-1`.