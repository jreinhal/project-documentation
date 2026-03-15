# SENTINEL RAG Master Test Plan (Canonical)

Version: 1.0
Last Updated: 2026-02-16
Status: Active master plan

Primary execution document:
- `SENTINEL_UNIFIED_RAG_MASTER_TEST_PLAN.md`

Legacy source files are archived under:
- `archive/RAG_MASTER_TEST_PLAN.md`
- `archive/SENTINEL-RAG-Testing-Guide.md`

Completeness baseline reference:
- `SENTINEL_ENTERPRISE_TESTING_COMPREHENSIVE.md`

---

## 1. Purpose and Scope

This document is the single master RAG test plan for SENTINEL.
It defines full-fidelity validation requirements for retrieval, generation, security, compliance, scale, and operational quality.

This is not a smoke-only plan.

Primary goals:
- Prevent silent RAG quality regressions.
- Detect cross-tenant leakage, prompt injection bypass, and PII/PHI disclosure before release.
- Validate strategy composition behavior across all supported RAG modules.
- Provide deterministic release gates with objective thresholds.

---

## 2. Document Precedence

If guidance conflicts:
1. This document is authoritative.
2. Source docs listed above are supporting references.
3. Release gates and hard-fail criteria in this document are binding.

---

## 3. Testing Philosophy

Principles:
- Separate retrieval correctness from generation quality.
- Isolate component changes and measure metric deltas.
- Use golden datasets as versioned release assets.
- Treat security and tenant isolation as first-class quality gates.
- Validate both functional correctness and operational resilience.

Required test layers:
1. Unit and integration safety net.
2. Retrieval-focused evaluation.
3. Generation-focused evaluation.
4. End-to-end strategy interaction validation.
5. Security/adversarial and privacy/compliance validation.
6. Performance/scale/chaos/soak validation.
7. Edition/profile policy validation.

---

## 4. Test Environments and Profiles

Profiles covered:
- `dev`
- `standard`
- `enterprise`
- `govcloud`
- `foundation` (feature-reduced behavior verification)

Editions covered:
- Trial
- Enterprise
- Medical
- Government

Execution model:
- UI-driven functional validation for user-facing behavior.
- Automated backend suites for regression and gate enforcement.
- Headed browser automation allowed for UI reproducibility.

---

## 5. Golden Corpus and Dataset Governance

### 5.1 Deterministic Golden Enterprise Documents

Required fixed-fact docs:
- `enterprise_transformation.txt`
- `enterprise_vendor_mgmt.txt`
- `enterprise_org_structure.txt`
- `enterprise_quarterly_review.txt`
- `legal_contract_review.txt`
- `finance_earnings_q4.txt`
- `legal_ip_brief.txt`

These must be maintained as stable answer anchors for regression checks.

### 5.2 Corpus Scale Baseline

Minimum corpus baseline:
- 7 golden enterprise docs.
- 3,000 randomized docs (1,000 GOV / 1,000 MED / 1,000 ENT).
- Multi-format ingestion matrix:
  - `.txt`, `.pdf`, `.docx`, `.html`, `.xlsx`, `.md`, `.csv`, `.json`, `.pptx`, `.ndjson`, `.log`
- Synthetic PII inclusion in randomized corpus for redaction validation.

### 5.3 Golden Dataset Categories (per edition)

Maintain versioned datasets:
- Factual (single-hop)
- Multi-hop / multi-document synthesis
- Unanswerable/refusal
- Adversarial (injection, poisoning, distractors)

Governance requirements:
- Version with code.
- Tag with release.
- Add every confirmed production failure as regression fixture.

---

## 6. Evaluation Framework and Objective Quality Metrics

Air-gap-compatible framework:
- Primary: local harness (RAGAS/DeepEval-style methodology).
- Secondary: classifier-assisted trend scoring for scale.

### 6.1 RAG Triad Thresholds

| Metric | Threshold |
|---|---|
| Context Precision | >= 0.80 |
| Context Recall | >= 0.70 |
| Faithfulness | >= 0.75 |
| Answer Relevancy | >= 0.80 |
| Noise Sensitivity | <= 0.15 |
| Citation Accuracy | >= 0.80 |
| Hallucination Rate | <= 5% |
| Refusal Appropriateness | >= 90% |

### 6.2 Retrieval Thresholds

| Metric | Threshold |
|---|---|
| Hit Rate @K | >= 0.85 |
| Precision @3 | >= 0.80 |
| Recall @5 | >= 0.70 |
| nDCG @10 | >= 0.75 |
| MRR | >= 0.80 |
| Context Entity Recall | >= 0.70 |

### 6.3 Generation Thresholds

| Metric | Threshold |
|---|---|
| Completeness | >= 0.70 |
| Citation Correctness | >= 0.80 |
| Format Compliance | >= 0.95 |

---

## 7. Full-Fidelity Functional Validation

### 7.1 Routing and Retrieval Behavior

Validate:
- `NO_RETRIEVAL` paths return no retrieved sources and no stale graph/source nodes.
- `CHUNK` paths return focused factual retrieval with correct source grounding.
- `DOCUMENT` paths return multi-source synthesis with citation integrity.

Signal-level coverage:
- HyDE signal behavior.
- Multi-hop detection behavior.
- Named-entity signal behavior.
- Combined signal interactions.

### 7.2 Strategy-Specific Validation (All Supported RAG Modules)

Modules requiring explicit ON/OFF and behavioral verification:
- HybridRAG
- AdaptiveRAG
- CRAG
- RAGPart
- QuCo-RAG
- HiFi-RAG
- MegaRAG
- MiA-RAG
- BiRAG
- Self-RAG
- HyDE
- Graph-O1
- HGMem
- Agentic

Per-module required evidence:
- Routing/reasoning trace step presence.
- Source-set effect versus baseline.
- Quality delta versus baseline.
- Latency delta versus baseline.
- Degradation behavior when disabled.

### 7.3 Pairwise and Composition Testing

Mandatory pairwise examples:
- `HYDE + QUCORAG`
- `CRAG + SELFRAG`
- `AGENTIC + HYDE`
- `CRAG + QUCORAG`
- `HIFIRAG + MEGARAG`
- `BIRAG + MIARAG`
- `GRAPHO1 + HYDE`

### 7.4 Graceful Degradation Scenarios

Required scenarios:
- Default-ON engines disabled one at a time.
- Sequential engine failure simulation.
- Empty vector store behavior.
- Partial dependency outage with fallback behavior.

### 7.5 Complex Compound Query Testing

Required coverage:
- Cross-document entity correlation.
- Cross-document fact synthesis.
- High-complexity analytical prompts.
- Cross-sector compound prompts.
- Large response / large-graph prompts.

Pass criteria:
- Correct routing.
- Source integrity.
- Citation correctness.
- No stale graph or source state.

---

## 8. Security and Adversarial Testing

### 8.1 OWASP LLM Top 10 Coverage

Minimum mapped coverage:
- LLM01 Prompt Injection
- LLM04 Data Poisoning
- LLM06 Sensitive Information Disclosure
- LLM08 Vector/Embedding Weaknesses
- LLM09 Misinformation

### 8.2 Prompt Injection Defense Validation

Mandatory tests:
- Direct override attempts.
- System prompt extraction attempts.
- Role manipulation and jailbreak attempts.
- Delimiter and output manipulation attempts.
- Obfuscation (Base64, homoglyph, zero-width, multilingual).
- Multi-turn persistence attacks.
- Indirect injection via retrieved documents.

### 8.3 Document Poisoning Validation

Method:
1. Baseline golden score.
2. Inject small poisoned corpus.
3. Re-run full evaluation.
4. Measure ASR and normal-operation degradation.

Defense validation targets:
- RAGPart
- CRAG
- Self-RAG
- HiFi-RAG reranking

### 8.4 Property-Based API Fuzz Testing

Target endpoint families:
- `/api/ask/**`
- `/api/ingest/**`
- `/api/auth/**`
- `/api/workspace/**`
- `/api/feedback/**`

Required fuzz dimensions:
- malformed payloads
- boundary values
- header manipulation
- concurrent race conditions
- schema conformance

Pass criteria:
- no unhandled 5xx from expected malformed inputs
- no stack-trace leakage
- consistent error schema

---

## 9. PII/PHI and Privacy Controls

### 9.1 PII Redaction

Required pattern classes:
- SSN
- email
- phone
- credit card
- DOB
- IP address
- names/addresses in supported contexts

Adversarial format checks:
- spaced/hyphen variants
- Unicode/homoglyph variants
- zero-width insertion

### 9.2 Medical/HIPAA Paths

Required checks:
- PHI detection and redaction behavior.
- PHI reveal authorization and audit trails.
- break-the-glass flow controls and auditing.
- HIPAA strict-mode feature restrictions.

---

## 10. Access Control and Tenant Isolation

### 10.1 Sector and Workspace Isolation

Required guarantees:
- No cross-sector retrieval leakage.
- No cross-workspace retrieval leakage.
- Cache-key isolation includes tenant dimensions.

### 10.2 Retrieval Pivot and Leakage Metrics

Mandatory measurements:
- Retrieval Pivot Risk (RPR)
- Leakage@K

Pass criteria:
- RPR within approved threshold.
- Leakage@K == 0 for unauthorized scope.

### 10.3 Filter Bypass and Injection Tests

Required attempts:
- sector parameter manipulation
- empty/wildcard filters
- injected filter payloads
- cache poisoning patterns

Pass criteria:
- No successful bypass.

### 10.4 RBAC Matrix

Required authorization checks:
- `/api/admin/*` non-admin denial
- user-to-user session isolation
- admin override behavior only where explicitly allowed

---

## 11. Authentication, Session, and Identity Security

Required coverage:
- OIDC/JWT token validation and claim handling.
- user auto-provisioning path behavior.
- gov CAC/PIV authentication path validation.
- session fixation protection.
- session timeout behavior.
- concurrent session policy behavior.
- secure cookie policy validation.

---

## 12. Audit, Observability, and Compliance

### 12.1 Audit Event Integrity

Audit events must include:
- event type
- principal/user
- workspace/sector context
- timestamp
- source IP/correlation ID where applicable
- outcome/details per event class

### 12.2 Fail-Closed Behavior

Mandatory for govcloud and HIPAA strict contexts.

When enabled, audit-write failure must block protected operation completion.

### 12.3 Frontend-Backend Contract Drift

Contract checks required for UI-driven endpoints:
- `/api/admin/dashboard`
- `/api/admin/health`
- `/api/admin/stats/usage`
- `/api/admin/stats/documents`
- `/api/ask/enhanced`
- `/api/sessions`

### 12.4 Compliance Artifact Verification

Required checks:
- control mapping currency
- ATO/FedRAMP evidence completeness
- auditability and data-flow traceability

---

## 13. Ingestion and Connector Validation

Required ingestion checks:
- magic-byte validation
- extension-spoof rejection
- malformed/large-file handling
- embedded active-content handling
- chunk integrity (no mojibake / binary artifact leakage)
- metadata preservation and correctness

Connector checks:
- incremental sync
- delete propagation
- sync consistency
- retry/backoff behavior

---

## 14. Graph, Source Evidence, and UI State Integrity

Required checks:
- source rendering endpoints (`/api/source/page`, `/api/source/region`)
- graph node/state correctness by routing mode
- stale-state reset across query transitions
- context graph and sector graph consistency
- entity explorer search/edges correctness

---

## 15. Performance, Scale, and Resilience

### 15.1 p95 Targets

| Metric | Target |
|---|---|
| TTFT | < 2s |
| End-to-end latency | < 10s |
| Inter-token latency | < 100ms |
| Retrieval latency | < 500ms |
| Embedding latency | < 200ms |
| Goodput | >= 95% |
| Throughput | >= 50 QPS |

### 15.2 Scale Matrix

| Vectors | Users | Purpose |
|---|---|---|
| 100K | 10 | baseline |
| 1M | 50 | production |
| 5M | 100 | stress |
| 10M | 200 | ceiling |

### 15.3 Chaos and Soak

Required chaos scenarios:
- LLM timeout/crash
- Mongo slowdown/unavailable
- vector corruption simulation
- concurrent ingest + query pressure

Soak requirement:
- 8-24 hours sustained run with stability and error-rate monitoring.

---

## 16. Air-Gap / SCIF and Supply Chain Assurance

Mandatory checks for relevant deployments:
- zero external network egress during runtime window
- offline startup/operation validation
- no runtime model/download dependency
- reproducible lockfile and dependency pinning
- image digest pinning and vulnerability baseline
- SBOM availability and review

---

## 17. Edition Isolation and Build Verification

Required build isolation checks:
- Trial/Enterprise must exclude medical/government-only code.
- Medical must exclude government-only code.
- Government includes full superset where intended.

Required runtime checks:
- excluded controllers/routes are unavailable in restricted editions.
- edition-specific provider contracts resolve correctly.

---

## 18. CI/CD Quality Gates (Binding)

### Gate 1 (Every PR)
- unit/integration/lint strict
- CI-lite E2E paths
- OIDC path
- static security checks
- API fuzz smoke on critical endpoints

### Gate 2 (Nightly)
- retrieval benchmarks
- generation grounding/faithfulness benchmarks
- adversarial suites
- load micro-benchmarks

### Gate 3 (Release Candidate)
- enterprise-realism E2E with advanced strategy set
- multi-turn suite
- chaos + soak
- governance evidence package

### Hard-Fail Criteria
Any one of the following is release-blocking:
- cross-tenant leakage > 0
- critical injection/exfiltration success
- security regression suite failure
- PII/PHI leakage in golden-dataset responses
- retrieval or generation regression > 5% versus prior release baseline

---

## 19. Execution Artifacts and Reporting

Each formal run must publish:
- profile/edition/build SHA
- corpus version/hash
- metric summaries by category
- security failure inventory
- waivers (owner + expiry)
- final decision (PASS / CONDITIONAL PASS / FAIL)

### Canonical Run Log Template

- Date:
- Tester:
- Edition/Profile:
- Git SHA:
- Corpus version/hash:
- Model + embedding:

Results:
- Retrieval metrics: PASS/FAIL
- Generation metrics: PASS/FAIL
- Security/adversarial: PASS/FAIL
- Performance/scale: PASS/FAIL
- Compliance/audit checks: PASS/FAIL

Hard-fail checks:
- cross-tenant leakage: PASS/FAIL
- critical injection/exfiltration: PASS/FAIL
- PII/PHI leakage: PASS/FAIL
- >5% regression: PASS/FAIL

Final decision:
- PASS / CONDITIONAL PASS / FAIL

Sign-off:
- Product Owner:
- Security Owner:
- Platform Owner:

---

## 20. Coverage Checklist (Master)

Mark each category complete per release cycle:
- [ ] Golden corpus + dataset governance
- [ ] Retrieval thresholds
- [ ] Generation thresholds
- [ ] Strategy-specific module coverage
- [ ] Pairwise strategy coverage
- [ ] Graceful degradation coverage
- [ ] Compound query suites
- [ ] Prompt injection and poisoning suites
- [ ] OWASP LLM Top 10 mapped checks
- [ ] Property-based fuzz suite
- [ ] PII/PHI and HIPAA checks
- [ ] Tenant isolation and leakage metrics
- [ ] RBAC/auth/session checks
- [ ] Audit and fail-closed checks
- [ ] Ingestion/connector validation
- [ ] Graph/source/UI contract checks
- [ ] Performance/scale/chaos/soak
- [ ] Air-gap/SCIF and supply chain checks
- [ ] Edition build isolation checks
- [ ] CI/CD gate evidence complete

---

## 21. Traceability Pointers (Source of Detailed Cases)

Detailed query banks, endpoint-level steps, and controller-level test inventory are drawn from and should be maintained in alignment with:
- `SENTINEL_UNIFIED_RAG_MASTER_TEST_PLAN.md`
- `SENTINEL_ENTERPRISE_TESTING_COMPREHENSIVE.md`

This master document defines required scope and release gate policy; implementation teams may keep auxiliary working sheets, but they cannot weaken these requirements.
