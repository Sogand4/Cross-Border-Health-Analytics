# Ethics Debt Ledger
**Jurisdiction Bundle:** Option A → Canada (PIPEDA) + Brazil (LGPD) + EU (GDPR)  

Status icons: 🔴 test failing (debt), 🟡 partial mitigation (design stage), 🟢 enforced.

---

| Status | Risk / Promise | Who it Protects | Control Owner | Acceptance Test | Enforcement Point | Next Action | Last Reviewed |
|:-------:|----------------|----------------|----------------|-----------------|------------------|--------------|----------------|
| 🟡 | “Collect health data only with explicit consent” (PIPEDA 4.3.2) | Canadian users providing symptom data | App Dev / API Team | `test_upload_without_consent_rejected` | Mobile App → API Gateway → Lambda | Verify consent toggle flow implemented; confirm CloudWatch metric `ConsentDeniedCount` is raised on 403 events | 2025-10-13 |
| 🟡 | “Allow data subjects to request deletion” (LGPD Art. 18 VI) | Brazilian users; any user withdrawing consent | Backend Engineer / Data Ops | `test_deletion_request_removes_user_data` | Lambda → S3 Lifecycle Policy | Configure deletion lifecycle rule ≤ 7 days; verify audit entry in `DeletionRequestLog` | 2025-10-13 |
| 🟡 | “Apply privacy by design & default” (GDPR Art. 25) | EU residents and cross-region users | Cloud Architect / App Dev | `test_privacy_by_design_defaults` | Mobile App + API Gateway + S3 | Confirm region isolation (`ca-central-1`, `eu-central-1`, `sa-east-1`) and disabled background sync by default | 2025-10-13 |
| 🟡 | “Notify users and obtain consent for secondary processing” (GDPR Art. 13(3)) | EU data subjects whose personal data may be reused for secondary purposes | Cloud Architect / Backend Team | `test_secondary_processing_requires_consent` | Lambda + App Notification + SecondaryProcessingLog | Implement notification flow for secondary data use; enforce Lambda check for consent before processing; re-run pytest to verify all non-consented processing is blocked | 2025-10-15 |

**Ethics Automation Summary:**  
All four Clause→Control→Test bundles include clear enforcement points and red-bar pytest evidence (conceptual). Each guardrail is tracked with a 🟡 status (“Designed / Pending Enforcement”) and is actionable in future CI.


---