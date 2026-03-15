# HIPAA / Compliance Checklist (KinCircle)

Date: 2026-01-31
Owner: KinCircle
Purpose: Validate HIPAA-readiness and vendor compliance for a healthcare-facing launch.

---

## 1) Scope & Role
- [ ] Confirm whether you are a Business Associate (BA) or Covered Entity (CE)
- [ ] Define PHI boundaries (what data is PHI, what is excluded)
- [ ] Document intended use cases (consumer-only vs healthcare orgs)
- [ ] Identify all data flows (client, API, storage, AI, analytics)

## 2) Vendor & BAA Requirements
- [ ] Maintain a vendor inventory (subprocessors, hosting, storage, AI, analytics)
- [ ] Ensure each PHI-touching vendor will sign a BAA
- [ ] Store signed BAAs centrally with version dates and renewal reminders
- [ ] If AI is used, confirm HIPAA-capable vendor + BAA and data usage limits
- [ ] Verify data residency/region requirements if applicable

## 3) Authentication & Access Control
- [ ] Enforce authenticated users (no anonymous access)
- [ ] Use strong auth (email/password, SSO, or OTP) with MFA for admins
- [ ] Implement server-side authorization and least privilege
- [ ] Confirm role-based access control (RBAC) is enforced server-side

## 4) Data Storage & Encryption
- [ ] Encrypt PHI at rest (DB + object storage)
- [ ] Encrypt PHI in transit (TLS 1.2+)
- [ ] Key management policy (rotation, revocation, access)
- [ ] Backups encrypted and access-controlled

## 5) Audit Logs & Monitoring
- [ ] Record access logs for PHI reads/writes
- [ ] Audit log export available for admins
- [ ] Retention policy enforced (e.g., 6 years if HIPAA applies)
- [ ] Monitoring/alerting for auth anomalies and data access spikes

## 6) Privacy & Consent
- [ ] Privacy policy clearly states data usage and sharing
- [ ] Consent flow for AI processing of PHI
- [ ] Data minimization and redaction before AI use
- [ ] Ability for orgs to disable AI features

## 7) Incident Response & Breach Notification
- [ ] Incident response plan documented
- [ ] Breach notification procedures defined (state + federal)
- [ ] Internal escalation contacts and timelines established

## 8) Data Retention & Deletion
- [ ] Retention policy for PHI defined and documented
- [ ] Account deletion and data purge process verified
- [ ] Support for legal hold if required

## 9) Security Program & Testing
- [ ] Regular vulnerability scanning
- [ ] Pen test completed (external assessor)
- [ ] Secure SDLC (code reviews, CI scanning, secrets detection)
- [ ] Dependency update policy and CVE response plan

## 10) Documentation & Training
- [ ] Admin/user security controls documented
- [ ] Compliance boundary documented (what is not supported)
- [ ] Staff security training completed
- [ ] Change management and release notes tracked

---

## Notes
- HIPAA compliance is not a single switch; it requires policy, process, and evidence.
- If AI is enabled, require BAA and confirm no training on PHI unless explicitly permitted.

