
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
    """Verifies that uploads without consent are rejected."""
    resp = client.post("/batch-upload", headers={"Authorization": fake_jwt(consent=False)})
    assert resp.status_code == 403, \
        "Consent control failure: API accepted upload without user consent."

```
---

### (Brazil) Clause 2: Data Subject Deletion Rights 

LGPD Article 18(VI) – “The data subject has the right to request deletion of personal data processed with their consent.”

---

#### Control
| Control | Enforcement Point | Implementation Notes |
|----------|------------------|----------------------|
| Enable “Delete My Data” option in mobile app settings. | Mobile App → API Gateway | Allows users to request deletion directly. Sends an authenticated DELETE request with the user’s JWT. |
| Propagate deletion to all regions within SLA. | Lambda → S3 Lifecycle Policy | Lambda triggers lifecycle rule to permanently remove raw and aggregated data for the user within 7 days. |
Log all deletion requests for audit. | CloudWatch → S3 Audit Log | `DeletionRequestLog` entries stored in `s3://symptom-audit-{region}/` and monitored daily for SLA breaches.

#### Test Exemplar

```python
def test_deletion_request_removes_user_data():
    """Verifies data deletion within SLA."""
    delete_user_data(user_id="user123")
    time.sleep(7 * DAY)
    assert not data_exists("user123"), \
        "Deletion control failure: user data still present after 7 days."
```
---

### (EU) Clause 3: Data Protection by Design and by Default  

**Source:**  
GDPR Article 25 – “The controller shall implement appropriate technical and organisational measures, such as pseudonymisation, which are designed to implement data-protection principles and to integrate the necessary safeguards into the processing.”

---

#### Control

| Control | Enforcement Point | Implementation Notes |
|----------|------------------|----------------------|
| Disable background sync by default. | Mobile App | App starts with background sync off; users must opt in manually after reviewing consent and privacy text. |
| Enforce regional data isolation. | S3 Bucket Policy + API Gateway | Each user’s data stored only in region-matched S3 bucket (e.g., `eu-central-1`, `ca-central-1`), ensuring jurisdictional compliance and simpler deletion auditing. |
| Require explicit user action for any broader sharing. | Cognito + Frontend Consent Flow | No third-party data export occurs unless user explicitly re-enables sharing and reconfirms consent. |

---

#### Test Exemplar

```python
def test_privacy_by_design_defaults():
    """Verifies GDPR Art.25 controls: background sync off, region isolation, and explicit sharing."""
    
    # 1. New users start with privacy-protective defaults
    user = create_new_user()
    settings = get_user_settings(user)
    assert settings["background_sync"] is False, \
        "Privacy-by-default failure: background sync should be disabled for new users."
    
    # 2. Data is stored in the correct regional bucket
    region = get_user_data_region(user)
    approved_regions = {"ca-central-1", "eu-central-1", "sa-east-1"}
    assert region in approved_regions, \
        f"Data stored outside approved regions: {region}"
    
    # 3. Broader data sharing requires explicit user action
    response = attempt_third_party_export(user, consent=False)
    assert response.status_code == 403, \
        "Export allowed without explicit re-consent — violates GDPR Art.25 safeguard."
```