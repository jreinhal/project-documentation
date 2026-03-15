# HIPAA Slow-Path Implementation Plan (KinCircle)

Date: 2026-01-31
Owner: KinCircle
Scope: Move from consumer prototype to HIPAA-ready architecture and operations.

---

## Phase 0 — Decision & Scope (1–2 weeks)

### Goals
- Confirm HIPAA applicability (Business Associate vs Covered Entity).
- Define data boundaries: what counts as PHI, what must never leave the system.
- Select vendors that can sign BAAs (LLM, storage, analytics, monitoring).

### Deliverables
- Data Flow Diagram (DFD)
- PHI Data Inventory + classification
- Vendor/BAA shortlist
- Compliance scope statement

---

## Phase 1 — Core Security Baseline (3–6 weeks)

### Goals
- Enforce real authentication and multi-tenant isolation server-side.
- Implement auditable access and immutable logging.
- Remove or gate non-compliant integrations (e.g., AI without BAA).

### Engineering Tasks
- Replace anonymous auth with email/password or SSO (Supabase Auth or external IdP).
- Enforce Supabase RLS on all family-scoped tables (already present in `supabase_schema.sql`).
- Add server-side auth verification for the local API proxy.
- Implement API token requirements + origin allowlists in compliance mode.
- Disable AI endpoints by default in compliance mode unless BAA-backed.
- Create admin-only audit log export.

### Deliverables
- Auth-enabled build
- RLS validation scripts
- Audit log export
- Compliance mode configuration guide

---

## Phase 2 — Data Protection & Operations (4–8 weeks)

### Goals
- Harden data storage, backups, and incident response.
- Add operational controls (monitoring, alerting, log retention).

### Engineering Tasks
- Add encryption at rest policies (Supabase + storage buckets).
- Implement key management and rotation procedures.
- Define backup + restore strategy.
- Add monitoring/alerting (auth anomalies, access spikes, API errors).
- Add admin controls for data retention and deletion.

### Deliverables
- Security runbook
- Incident response plan
- Backup/restore playbook
- Monitoring dashboards

---

## Phase 3 — HIPAA Readiness (6–12 weeks)

### Goals
- Meet HIPAA Privacy + Security Rule expectations.
- Prepare for external assessment.

### Engineering / Compliance Tasks
- BAAs executed for all PHI vendors.
- Formal risk assessment and mitigation log.
- Access control and least-privilege review.
- Training + policy documentation.
- External penetration test + remediation.

### Deliverables
- HIPAA Security Risk Assessment report
- Policy suite (privacy, retention, breach notification)
- Pen test report + remediation log

---

## Dependencies / Blockers
- Legal/compliance counsel for scope and BAAs.
- Selection of HIPAA-capable LLM vendor or disabling AI features.
- Budget for audits, security tooling, and incident response readiness.
- Engineering bandwidth for auth + backend hardening.

---

## Immediate Next Steps (this sprint)
1) Add compliance-mode guards (privacy enforced, AI disabled by default, API token required).
2) Add explicit compliance-mode config to docs + .env.example.
3) Plan auth upgrade path (Supabase email OTP or external IdP).

