# SENTINEL Unified RAG Master Test Plan

Status: Canonical unified test-plan document for SENTINEL RAG validation.

Canonical execution document:
- `SENTINEL_UNIFIED_RAG_MASTER_TEST_PLAN.md`

Legacy source files are archived under:
- `archive/RAG_MASTER_TEST_PLAN.md`
- `archive/SENTINEL-RAG-Testing-Guide.md`

Completeness Baseline:
- `SENTINEL_ENTERPRISE_TESTING_COMPREHENSIVE.md`

## Current Code Alignment (2026-03-15)

Profile/auth startup constraints:
- `AUTH_MODE=DEV` without the `dev` Spring profile active triggers a hard startup failure (enforced in `MercenaryApplication.java`). Test startup with mismatched `AUTH_MODE=DEV` + non-dev profile must fail fast with a clear error — not start silently in STANDARD mode.
- `APP_PROFILE=dev` and `SPRING_PROFILES_ACTIVE=dev` are both valid profile selectors. `APP_PROFILE` is the repo-native selector used by harness scripts; `SPRING_PROFILES_ACTIVE` is the direct Spring override. They must not be mixed in the same process.

Post-2026-03-01 changes requiring explicit validation (PRs #413-#442):
- Observability stack (PRs #413-#414): Prometheus metrics endpoint (`/actuator/prometheus`), Grafana dashboard + PrometheusRule alerts, CWE-209 fix (API error responses must not leak stack traces or internal paths).
- Medical TLS enforcement (PR #415): `MedicalSecurityValidator` now enforces TLS when `sentinel.security.enforce-tls=true` regardless of HIPAA strict mode — not just in HIPAA strict mode.
- C²-Cite corpus grounding (PR #439): `CorpusGroundingService.groundCitations()` added to `/api/ask/enhanced` response path — citation spans are now corpus-grounded against source documents. Response shape updated.
- C²-Cite source deduplication (PR #441): `CorpusGroundingService.groundCitations()` deduplicates sources before returning — duplicate source entries in grounded citations are a regression.
- RAGBoost (PR #442): `DocumentBoostService` + `FeedbackService` wired into `RagOrchestrationService`; feedback-driven document re-ranking applied at retrieval time. Flag: `SENTINEL_RAGBOOST_ENABLED` (default `false`).

Recent merged security changes are in scope and must be explicitly validated in full E2E runs:
- `SecurityFilter` path normalization hardening (`/..`, encoded `%2e%2e`, double/triple-encoded traversal payloads, mixed static/public-path traversal attempts).
- `/api/ask/stream` parity with `/api/ask` for:
  - workspace quota enforcement
  - input-query PII redaction before downstream processing
  - prompt injection audit events
  - HIPAA PHI query audit behavior in strict mode
  - output redaction before final response emission
- Null/edge request handling in streaming flows (no unaudited bypasses, no NPE-driven path skips).
- Container and sidecar security regression checks after dependency/base-image updates.
- Government fail-closed startup guardrails:
  - `GovernmentSecurityValidator` enforces SCIF-safe startup requirements for Government/govcloud operation.
  - Unsafe Government runtime settings must fail startup (no fail-open drift).
- Government roadmap/release governance:
  - Government-affecting upgrades must satisfy explicit SCIF release-gate evidence before phase/PR closure.
- Auth/session negative-path API contracts (current implementation) must be explicitly validated:
  - All error responses use standardized `ApiErrorResponse` record: `{error, code, timestamp}` (introduced in phase 9 refactoring).
  - `AuthController` login errors throw `ApiRequestException` with stable codes (`MISSING_CREDENTIALS`, `INVALID_CREDENTIALS`), caught by `GlobalExceptionHandler`.
  - `SessionController` export endpoints return `ApiErrorResponse` with stable codes (`AUTH_REQUIRED`, `SESSION_EXPORT_DISABLED`, `SESSION_NOT_FOUND`, `EXPORT_FAILED`, `ACCESS_DENIED`, `SESSION_FEATURES_DISABLED`).
  - `SessionController` non-export endpoints (create, touch, clearHistory, context, traces) still return empty bodies for auth/permission errors.
  - `SessionController` file export success response uses typed `SessionActionResponse` (`status`, `sessionId`) — intentionally opaque, must not expose server filesystem paths.
  - Export filenames are generated from timestamp and UUID — the public API success response remains opaque through `SessionActionResponse` and must not expose server filesystem paths.
- `GlobalExceptionHandler` standardized `ApiErrorResponse` behavior must be validated:
  - `ApiRequestException` -> status from exception, `{error, code, timestamp}` (`BAD_REQUEST` messages are sanitized)
  - `SecurityException` -> `403` with `error="Access denied"`, `code="ACCESS_DENIED"`, `timestamp`
  - `IllegalArgumentException` -> `400` with sanitized `error`, `code="INVALID_REQUEST"`, `timestamp`
  - unhandled `Exception` -> `500` with `error="Internal server error"`, `code="INTERNAL_ERROR"`, `timestamp`

Post-audit vibecode and deferred-roadmap changes (PRs #311-#326) are in scope:
- LlmCallHelper centralization (PRs #312-#313): timeout/fallback behavior across 19+ RAG services now routed through `LlmCallHelper.callWithTimeout()`. Externally-observable timeout values are unchanged; this is an internal pattern consolidation.
- Controller auth gate extraction (PR #315): `HyperGraphController.validateSectorAccess()` and `SessionController.withSessionOwnership()` are internal refactors; endpoint signatures and error contracts are unchanged.
- @Value → @ConfigurationProperties migration (PRs #317-#319): config binding for RAG strategies, auth/security, and HIPAA services migrated to typed properties. Environment variable names unchanged via Spring relaxed binding. Known remaining services: HiFiRagService, MegaRagService, MiARagService, QuCoRagService, RagPartService, BidirectionalRagService, AgenticRagOrchestrator.
- CatRAG chain-completeness scoring (PR #320): `ChainCompletenessScorer` integrated into `HybridRagService` as a post-RRF-fusion augmentation (`CATRAG_ENABLED=false` by default).
- C²-Cite real-time citation verification (PR #321): `CitationVerificationService.verifyResponse()` wired into `RagOrchestrationService` post-LLM (`SENTINEL_CITATION_REALTIME_VERIFICATION_ENABLED=false` by default).
- A-RAG tool contract formalization (PR #322): `AgenticTool` interface and `ToolContext` record formalize the tool-calling contract for agentic multi-hop reasoning.
- WildGraphBench eval harness (PR #323): `GraphRetrievalEvalRunner` loads WildGraphBench datasets, queries `HGMemQueryEngine`, scores with `GroundTruthEvalService`.
- Col-Bandit reranker routing (PR #324): epsilon-greedy/UCB1 bandit routing between ColBERT and cross-encoder sidecars in `CrossEncoderReranker` when `mode=auto` (`sentinel.hifirag.reranker.bandit-routing.enabled=false` by default).

Findings-driven execution preconditions from `D:\OneDrive\Desktop\Findings\RAG_E2E_QC_Findings_2026-02-16.md`:
- Treat repeated embedding overflow errors (`input length exceeds the context length`) as release-blocking ingestion instability until mitigated.
- Treat response/source mismatch as a quality defect:
  - attached sources present while response is plain `No relevant records found.`
- Treat `NO_RETRIEVAL` route leakage (greeting/conversational query returns retrieval sources) as a functional regression.
- Treat context-query graph emptiness (entity/context graph remains placeholder-only for evidence-backed queries) as a graph-quality regression.

## 0. Document Precedence

When guidance conflicts:
1. Sections `1` through `7` in this document are authoritative (response-integrity section + parity sections 2.8-2.11 are mandatory).
2. Archived legacy documents under `archive/` are reference-only and must not override this plan.
3. `SENTINEL_UNIFIED_RAG_MASTER_TEST_PLAN.md` is the single active source of truth for full end-to-end RAG testing.

This structure keeps execution policy compact and avoids duplicated or conflicting gate definitions.

## 1. Response Integrity and Semantic Validation (Required)

This section is release-blocking and must be executed for every formal run. It exists to catch failures where a response is fluent but wrong, contaminated by prior context, unsupported by sources, or policy-inconsistent.

Non-waivable user directive: every response must match expected semantics, required format, and factual accuracy.

### 1.1 Response Validation Dimensions (All Required)

Every evaluated response must be scored across all dimensions below:
- Semantic correctness: answer matches the intended question target and expected fact anchor(s).
- Evidence binding: core claims are supported by attached sources; no unsupported material claims.
- Context isolation: answer is not contaminated by unrelated prior prompts/docs unless explicitly scoped.
- Safety/policy correctness: guardrail behavior is correct (prompt-injection resistance, PII/PHI handling, RBAC, classification boundaries).
- Abstention correctness: abstain only when evidence is insufficient; do not false-abstain when valid evidence exists.

### 1.2 Assertion Contract (Per Test Case)

Every test case must define all fields below before execution:
- `intent_target`: what the model must answer.
- `must_include`: required tokens/facts/values.
- `must_not_include`: forbidden tokens/facts (contamination markers, leaked/system data, wrong entity/program).
- `expected_sources`: exact/allowed source set expectations.
- `policy_expectation`: one of `ALLOW`, `BLOCK`, `ABSTAIN`, `REDACT`.
- `severity`: `blocker`, `high`, `medium`, `low`.

If any `blocker` assertion fails, run verdict is `FAIL`.

### 1.3 Deterministic Anchor Tests (Mandatory)

The following anchors are required in every release run:
1. Enterprise budget anchor:
   - Query target: Enterprise Transformation Program budget.
   - Must include: $150,000,000 (or equivalent 150 million expression).
   - Must not include: ORION budget markers ($42,000,000, ORION-TEST-001) unless query explicitly targets ORION.
2. ORION poison containment anchor:
   - Poison-doc upload/query must not alter unrelated enterprise baseline answers.
   - After poison scenario, baseline enterprise anchor must remain correct.
3. False-abstention anchor:
   - If relevant source exists, response cannot be plain No relevant records found.
4. Citation coherence anchor:
   - Source list must align with answer claims; orphan sources and unsupported claims fail.
5. Redaction anchor:
   - PII/PHI cases must show redacted output markers and no raw sensitive values.
6. Session continuity follow-up anchor:
   - After an enterprise-budget anchor query, execute follow-up query `What is the total program budget?`.
   - Must include enterprise budget anchor (`$150,000,000` / `150 million`).
   - Must not include ORION contamination markers (`$42,000,000`, `ORION-TEST-001`, `Orion Program`).
   - This case is `blocker` severity and fails on either semantic mismatch or contamination.

### 1.4 Failure Taxonomy and Required Disposition

All response failures must be tagged with one primary code:
- SEMANTIC_MISMATCH
- CONTEXT_CONTAMINATION
- SOURCE_CLAIM_MISMATCH
- FALSE_ABSTENTION
- UNSAFE_DISCLOSURE
- RBAC_POLICY_BYPASS
- CLASSIFICATION_POLICY_BYPASS

Disposition rules:
- SEMANTIC_MISMATCH, CONTEXT_CONTAMINATION, SOURCE_CLAIM_MISMATCH, UNSAFE_DISCLOSURE, and any policy bypass are release-blocking by default.
- Waiver requires explicit owner approval, expiry date, and compensating control; no silent carry-forward.

### 1.5 Manual UI Validation Requirements for Response Issues

Manual UI testing must explicitly validate response integrity, not only UI mechanics:
- Capture screenshot + source panel + query text for each anchor case.
- Record expected vs actual answer values in the run log.
- Record contamination checks after adversarial/poison scenarios.
- For session continuity, capture both:
  - explicit enterprise query answer, and
  - ambiguous follow-up answer (`What is the total program budget?`),
  with expected/actual semantic verdict for each.
- A manual run without semantic assertion evidence is incomplete and must be marked FAIL.

### 1.6 Golden Answer Contract Files (Mandatory)

Every critical query must be defined in a versioned Golden Answer Contract artifact.

Required contract fields:
- case_id
- intent_target
- equired_anchors (must appear)
- orbidden_anchors (must not appear; contamination markers)
- xpected_sources (allowlist/denylist rules)
- ormat_contract (required structure: bullets/table/json/sections)
- policy_expectation (ALLOW, BLOCK, ABSTAIN, REDACT)
- severity

Gate rule:
- Any failure in a locker contract case is release-blocking.

### 1.7 Dual Evaluator Requirement (Independent Validation)

Response quality must be validated by two independent evaluators:
- Evaluator A: deterministic rule-based validator (anchors/forbidden/source checks).
- Evaluator B: model-based grader or rubric scorer.

Pass rule:
- Response passes only when A and B both pass, or a documented adjudication closes disagreement.

Disagreement handling:
- Disagreement rate above 2% across blocker/high cases is release-blocking until triaged.

### 1.8 Semantic Drift Trend Gate (Release-Over-Release)

Semantic quality is trend-gated, not point-in-time only.

Required metrics per release:
- Blocker-case semantic pass rate.
- Format-contract pass rate.
- Source-claim coherence pass rate.
- Contamination incident count.

Hard thresholds:
- Blocker-case semantic pass rate must not regress.
- Any increase in contamination incidents is release-blocking unless explicitly explained and accepted with expiry.
- Format-contract pass rate must remain >= 99% on blocker/high cases.

### 1.9 Visual Format and Presentation Compliance (Required)

Each response must be validated for visual/structural correctness when format is specified.

Required checks:
- Expected structure present (heading levels, bullet lists, numbered lists, JSON schema, table columns).
- No malformed markdown/HTML rendering that changes meaning.
- Source/citation panel displays expected references for citation-required cases.
- UI screenshots captured for each format-sensitive blocker case.

Failure rule:
- Any blocker-case format mismatch is release-blocking.
## 2. Comprehensive Parity Addendum (Required)

This section explicitly incorporates high-signal coverage that must be present to satisfy enterprise-comprehensive parity.

### 2.1 Golden Corpus Reference (Deterministic)

Required golden documents and fixed-fact anchors:
- `enterprise_transformation.txt`
- `enterprise_vendor_mgmt.txt`
- `enterprise_org_structure.txt`
- `enterprise_quarterly_review.txt`
- `legal_contract_review.txt`
- `finance_earnings_q4.txt`
- `legal_ip_brief.txt`

Required corpus scale baseline:
- 7 golden enterprise documents
- 3,000 randomized docs (1,000 per sector: GOV / MED / ENT)
- Multi-format ingestion matrix (`.txt`, `.pdf`, `.docx`, `.html`, `.xlsx`, `.md`, `.csv`, `.json`, `.pptx`, `.ndjson`, `.log`)
- Synthetic PII incidence in randomized corpus for redaction testing
- Multi-modal document corpus: at least 3 documents with embedded images/charts/tables for MegaRAG visual extraction validation
- OCR validation documents: at least 2 scanned/image-only documents for OCR accuracy gate (when `OCR_ENABLED=true`)

### 2.2 Evaluation Framework and Minimum Metric Gates

Air-gap-compatible framework requirement:
- Primary: local evaluation harness (RAGAS / DeepEval-style)
- Secondary: classifier-based evaluation for scale and trend monitoring

RAG triad minimum thresholds:

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

Retrieval minimum thresholds:

| Metric | Threshold |
|---|---|
| Hit Rate @K | >= 0.85 |
| Precision @3 | >= 0.80 |
| Recall @5 | >= 0.70 |
| nDCG @10 | >= 0.75 |
| MRR | >= 0.80 |
| Context Entity Recall | >= 0.70 |

Generation minimum thresholds:

| Metric | Threshold |
|---|---|
| Completeness | >= 0.70 |
| Citation Correctness | >= 0.80 |
| Format Compliance | >= 0.95 |

### 2.2.1 Quality Control Gates (Response + Graph)

The following QC gates are mandatory in addition to retrieval/generation metrics:
- Response relevance gate:
  - answer must address the prompt intent and not default to generic fallback text when evidence is available.
- False-abstention gate:
  - if `sources >= 1`, response must not be plain `No relevant records found.` unless trace-level evidence checks explicitly conclude insufficient evidence.
- Citation/source coherence gate:
  - claims in the response must be supported by attached/cited sources; unsupported claims fail.
- NO_RETRIEVAL determinism gate:
  - greeting/conversational checks (for example `Hello`) must return zero retrieval sources and no source nodes in query graph.
- Graph meaningfulness gate:
  - for contextual evidence queries, entity/context graph must be non-placeholder and contain meaningful nodes/edges.
- Graph-response coherence gate:
  - response concepts and graph entities must overlap (partial term/entity alignment required).
- Ontology typed-relation correctness gate:
  - entity relationships must use valid `RelationType` for their entity type pairs (14 relation types: WORKS_AT, LOCATED_IN, DEPENDS_ON, etc.).
  - canonical `TypePair` resolution must be consistent (A→B and B→A resolve to same canonical pair).
- Ontology multi-hop traversal boundary gate:
  - graph traversal must respect `max-hops` configuration limit.
  - sector-boundary enforcement during traversal: cross-sector hops must be blocked unless user has access to both sectors.
  - max-results limit must be enforced to prevent unbounded graph expansion.

### 2.3 Cross-Tenant Retrieval Pivot and Filter Bypass Coverage

Mandatory multi-tenant attack validation:
- Retrieval Pivot Risk (RPR) measurement
- Leakage@K measurement
- Cross-sector entity pivot attempts
- Per-hop authorization checks for graph traversal modes

Mandatory filter bypass suite:
- Direct sector parameter manipulation
- Injection attempts into filter expression inputs
- Empty filter / wildcard filter scenarios
- Cache-poisoning and stale-cache replay attempts

Cache isolation verification:
- `secureDocCache` (configured in `SecureDocCacheConfig`) must use userId-prefixed cache keys to prevent cross-user document leakage
- Cache key hardening: verify that two users querying the same document receive independently cached results
- Cache invalidation: verify stale cache entries do not serve another user's data after document deletion

Workspace API RBAC validation (`/api/workspaces/**`):
- `GET /api/workspaces` — authenticated list endpoint; unauthenticated must return `401`
- `POST /api/workspaces`, member management, quota, usage management — admin-only; non-admin authenticated requests must return `403`
- Workspace creation must be rejected for non-admin roles in all editions

Pass criteria:
- Zero cross-tenant leakage
- Zero unauthorized-sector source inclusion
- Zero successful filter-expression bypass
- Zero unauthorized workspace management operations

### 2.4 Frontend-Backend Contract Drift Detection

Required contract checks for UI-consuming endpoints:
- `/api/admin/dashboard`
- `/api/admin/health`
- `/api/admin/stats/usage`
- `/api/admin/stats/documents`
- `/api/ask/enhanced` — **response shape updated (PR #439)**: now includes corpus-grounded citation span fields from `CorpusGroundingService.groundCitations()` when `SENTINEL_CITATION_REALTIME_VERIFICATION_ENABLED=true`. Frontend contract test must assert new fields present when enabled, absent when disabled.
- `/api/sessions`

Required method:
1. Extract fields referenced by frontend code.
2. Compare against response object keys from API.
3. Fail build when frontend-required keys are missing.

Required auth/session negative-path contract checks (release-critical backend contracts):

**Note on non-`ApiErrorResponse` endpoints**: The following paths return custom JSON shapes and must NOT be asserted against the standard `ApiErrorResponse` contract:
- **Banner acknowledgment** (`BannerAcknowledgmentFilter`, `AuthModeController`): returns `{"error": "banner_acknowledgment_required", ...}` pre-ack, and `{"acknowledged": true}` on success.
- **Workspace rejection** (`WorkspaceFilter`): returns `{"error": "WORKSPACE_ACCESS_DENIED", "message": "..."}` — not `ApiErrorResponse`.
- **SSE stream errors** (`/api/ask/stream`): errors arrive as `event: error` SSE events, not HTTP error responses (see Section 3.1.1).

All other error payloads use standardized `ApiErrorResponse` record shape: `{error: string, code: string, timestamp: ISO-8601}` unless noted otherwise.

| Endpoint | Scenario | Expected Status | Expected Payload Contract |
|---|---|---|---|
| `/api/auth/login` | Missing username/password | `400` | `ApiErrorResponse` — `code="MISSING_CREDENTIALS"`, `error="Missing credentials"` |
| `/api/auth/login` | Invalid credentials | `401` | `ApiErrorResponse` — `code="INVALID_CREDENTIALS"`, `error="Invalid credentials"` |
| `/api/sessions/{sessionId}/export` | Unauthenticated | `401` | `ApiErrorResponse` — `code="AUTH_REQUIRED"` |
| `/api/sessions/{sessionId}/export` | Session features unavailable | `501` | `ApiErrorResponse` — `code="SESSION_FEATURES_DISABLED"` |
| `/api/sessions/{sessionId}/export` | HIPAA export disabled | `403` | `ApiErrorResponse` — `code="SESSION_EXPORT_DISABLED"`, `error` contains `Session export disabled` |
| `/api/sessions/{sessionId}/export` | Access denied | `403` | `ApiErrorResponse` — `code="ACCESS_DENIED"` |
| `/api/sessions/{sessionId}/export` | Session not found | `404` | `ApiErrorResponse` — `code="SESSION_NOT_FOUND"` |
| `/api/sessions/{sessionId}/export` | IO/export failure | `500` | `ApiErrorResponse` — `code="EXPORT_FAILED"`, `error` contains `Export failed` |
| `/api/sessions/{sessionId}/export/file` | Unauthenticated | `401` | `ApiErrorResponse` — `code="AUTH_REQUIRED"` |
| `/api/sessions/{sessionId}/export/file` | Session features unavailable | `501` | `ApiErrorResponse` — `code="SESSION_FEATURES_DISABLED"` |
| `/api/sessions/{sessionId}/export/file` | HIPAA export disabled | `403` | `ApiErrorResponse` — `code="SESSION_EXPORT_DISABLED"` |
| `/api/sessions/{sessionId}/export/file` | Access denied | `403` | `ApiErrorResponse` — `code="ACCESS_DENIED"` |
| `/api/sessions/{sessionId}/export/file` | Session not found | `404` | `ApiErrorResponse` — `code="SESSION_NOT_FOUND"` |
| `/api/sessions/{sessionId}/export/file` | IO/export failure | `500` | `ApiErrorResponse` — `code="EXPORT_FAILED"` |
| `/api/sessions/{sessionId}/export/file` | Success | `200` | `SessionActionResponse` — `{status: "exported", sessionId}` only (no filename/path disclosure) |
| `/api/sessions/{sessionId}/history` | Success (clear) | `200` | `SessionActionResponse` — `{status: "cleared", sessionId}` |
| `/api/auth/login-banner` | Any | `200` | Banner content payload (not `ApiErrorResponse`) |
| `/api/auth/acknowledge-banner` | Unauthenticated | `401` | `{"error": "authentication_required"}` (custom shape) |
| `/api/auth/acknowledge-banner` | Success | `200` | `{"acknowledged": true}` (custom shape) |
| Global: `ApiRequestException` | Per exception | Per exception | `ApiErrorResponse` — `code` + `error` (BAD_REQUEST messages sanitized) + `timestamp` |
| Global: `SecurityException` | Any | `403` | `ApiErrorResponse` — `code="ACCESS_DENIED"`, `error="Access denied"` |
| Global: `IllegalArgumentException` | Any | `400` | `ApiErrorResponse` — `code="INVALID_REQUEST"`, `error` sanitized |
| Global: unhandled `Exception` | Any | `500` | `ApiErrorResponse` — `code="INTERNAL_ERROR"`, `error="Internal server error"` |

Contract drift handling rule:
1. Any status/payload drift from this table requires explicit approval and update to this plan before release sign-off.
2. Unapproved contract drift is a release fail.

### 2.5 Performance, Scale, Chaos, and Soak Requirements

Required p95 targets:

| Metric | Target |
|---|---|
| TTFT | < 2s |
| End-to-end latency | < 10s |
| Inter-token latency | < 100ms |
| Retrieval latency | < 500ms |
| Embedding latency | < 200ms |
| Goodput | >= 95% |
| Throughput | >= 50 QPS |

Required scale matrix:

| Vectors | Concurrent Users | Purpose |
|---|---|---|
| 100K | 10 | Baseline |
| 1M | 50 | Production range |
| 5M | 100 | Stress |
| 10M | 200 | Ceiling |

Edition-specific p95 latency SLOs (capability promotion gates):

| Edition | p95 End-to-End Target |
|---------|----------------------|
| Government | <= 5s |
| Medical | <= 4s |
| Enterprise | <= 3s |

Edition-specific kill criteria:

| Edition | Kill Criterion | Threshold |
|---------|---------------|-----------|
| All | Faithfulness regression vs prior release | > 2pp (hard block) |
| All | Retrieval or generation regression | > 5% vs baseline |
| Medical | PHI leakage in any response | Zero tolerance |
| Government | Unauthorized classification access | Zero tolerance |
| All | Cross-tenant data leakage | Zero tolerance |

Sparse-dense fusion validation:
- Learned-sparse retrieval (BGE-M3) quality thresholds must meet retrieval minimum gates above.
- Sparse-dense RRF fusion must not degrade dense-only retrieval quality by more than 1pp on any metric.
- Sparse embedding sidecar unavailability must fall back gracefully to dense-only retrieval without error.

Required resilience coverage:
- Chaos scenarios (LLM timeout/crash, Mongo slowdown/unavailable, corruption simulation, concurrent ingest/query)
- Graceful thread pool saturation: verify `ragExecutor` and `rerankerExecutor` (configured in `RagPerformanceConfig`) degrade gracefully under load — rejected tasks must not crash the application or produce silent data loss
- Soak test window: 8-24h at sustained partial peak load

Mandatory soak test protocol (required for release sign-off):
- Entry criteria:
  - Corpus baseline satisfied (7 golden enterprise docs + 3,000 randomized docs across GOV/MED/ENT) with source-doc manifest evidence.
  - Target edition profile running with production-equivalent auth/config for that lane.
  - No active migration jobs that would invalidate latency/throughput interpretation.
  - Any run below corpus baseline is diagnostic-only and cannot be used for release sign-off.
- Execution profile:
  - Duration tiers:
    - Pre-release soak: 8h minimum (all editions)
    - Release-candidate soak: 24h target (required for government and recommended for medical/enterprise)
  - Load envelope:
    - Sustained 40-60% of expected peak
    - Burst spikes to 80% peak for 5 minutes every hour
  - Traffic mix:
    - Query/retrieval: 70%
    - Ingestion/update: 20%
    - Admin/reporting/background: 10%
- Mandatory soak pass gates:
  - p95 latency remains within edition SLOs in at least 95% of 5-minute windows.
  - End-to-end failure rate <= 1% with no sustained (>10 min) degradation trend.
  - No cross-tenant leakage, no classification bypass, no PII/PHI leakage events.
  - No fail-open behavior in security/compliance controls.
  - No memory-leak trend greater than 10% growth from hour-2 baseline to end-of-run (excluding planned warmup).
  - No unbounded queue growth or runaway retry loops.
- Required soak evidence artifacts:
  - Timestamped run config/profile and load-shape definition.
  - Time-series export for latency, error rate, throughput, and resource utilization.
  - Incident log for all retries/timeouts/circuit-breaker transitions during soak window.
  - Final soak verdict with explicit PASS/FAIL against each mandatory gate.

### 2.6 CI/CD Tiered Gates and Hard-Fail Rules

Required CI gate model — 3-tier + build (matches `.github/workflows/ci.yml`):

| Tier | Job Name | Commands | Scope | Dependency |
|------|----------|----------|-------|------------|
| 0 | `preflight` | Detect docs-only HEAD commit | Skip entire pipeline on docs-only change | None |
| 1 | `unit-tests` | `./gradlew test -Plint -PlintWerror` + Gitleaks + `ciE2eTest` + `ciOidcE2eTest` + `ciCrossTenantE2eTest` + `ciStreamingParityTest` | Unit tests, lint, secret scanning, pipeline/OIDC/cross-tenant E2E, streaming parity gate | `preflight` skip != true |
| 2a | `e2e-suites` (matrix, 5 parallel) | `ciEnterpriseE2eTest`, `ciFrontierE2eTest`, `ciDeferredRagE2eTest`, `ciFoundationE2eTest`, `ciEdgeDegradationE2eTest` | Enterprise RAG, frontier strategies, deferred strategies, cloud model stubs, edge degradation | Tier 1 pass |
| 2b | `enterprise-realism` | `ciGroundTruthEvalTest` | Ground-truth eval dataset validation + trend dashboard | Tier 1 pass |
| 3 | `build` | Enterprise build verification + `verifySbom` + SonarCloud (informational) | Build artifact, SBOM, static analysis | Tiers 1-2 pass (runs even if upstream fails) |

E2E test profiles and their strategy coverage:

| Profile | Test Class | Key Strategies Enabled | CI Job |
|---------|------------|----------------------|--------|
| `ci-e2e` | `PipelineE2eTest` | All strategies disabled (pipeline mechanics only) | `unit-tests` |
| `ci-e2e` + `enterprise` | `OidcPipelineE2eTest` | OIDC Bearer token auth path | `unit-tests` |
| `ci-e2e` | `CrossTenantIsolationE2eTest` | Sector/workspace isolation | `unit-tests` |
| `ci-e2e` | `StreamingParityE2eTest` | ask vs ask/stream guardrail identity | `unit-tests` |
| `ci-e2e-enterprise` | `EnterprisePipelineE2eTest` | HybridRAG, RAGPart, MiaRAG, HiFi-RAG | `e2e-suites` |
| `ci-e2e-frontier` | `FrontierPipelineE2eTest` | Enterprise + ContextualRetrieval, SpeculativeRAG | `e2e-suites` |
| `ci-e2e-foundation` | `FoundationPipelineE2eTest` | QuCoRAG, MegaRAG, HiFi-RAG, RAGPart, PII | `e2e-suites` |
| `ci-e2e-deferred` | `DeferredRagPipelineE2eTest` | HtmlRAG, CAG, UniversalRAG, SitEmb | `e2e-suites` |
| `ci-e2e-edge` | `EdgeDegradationE2eTest` | Edge-S profile degradation (Edge-S only in CI) | `e2e-suites` |
| `ci-e2e` | `ClassificationCeilingE2eTest` | Government clearance-level filtering | **non-gating** (not wired to a CI task) |

Merge gate: only the `build` job (Tier 3) is required for PR merge. SonarCloud coverage is informational and non-blocking.

Hard-fail conditions (must block release):
- Any cross-tenant leakage
- Any critical injection/exfiltration success
- Any security regression test failure
- PII/PHI leakage in golden-dataset responses
- Retrieval or generation regression > 5% vs prior release baseline
- Repeated embedding-overflow ingestion failures in run logs
- Any confirmed false-abstention (`sources >= 1` with plain `No relevant records found.`)
- Any confirmed NO_RETRIEVAL source leakage on greeting/conversational control queries
- Any required manual UI Playwright runner not executed and passing for its trigger context
- Any manual UI gate outcome recorded as `FAIL`, `BLOCKED`, `SKIPPED`, or `NOT RUN`
- Any Government-mode startup that succeeds with SCIF-unsafe settings (fail-closed guardrail regression)

### 2.7 Government SCIF Release Gate (Required)

This gate is mandatory when:
- profile is `govcloud`, or
- edition is `GOVERNMENT`, or
- a change can affect Government runtime behavior (config, caching, transport, routing, auth, policy enforcement).

Minimum required evidence:
1. Fail-closed validator unit test pass:
   - `.\gradlew.bat test --tests "*GovernmentSecurityValidatorTest"`
2. Govcloud startup validation:
   - positive case: hardened Government settings boot successfully
   - negative case(s): intentionally unsafe settings fail startup
3. Govcloud smoke/E2E evidence attached:
   - Government edition + govcloud profile run artifacts/logs
4. SCIF testing documentation/evidence updated:
   - `docs/customer/AIRGAP_SCIF.md` aligned with current runtime constraints
   - `docs/evals/government_ato_readiness_packet.md` updated with control mapping and artifact links
5. Deferred backlog discipline:
   - do not begin deferred frontier backlog until SCIF gate evidence is complete and stable
6. Burn-in evidence protocol (required for Government releases):
   - 7-day consecutive govcloud boot stability window with zero SCIF startup-gate failures
   - 3 consecutive govcloud test runs pass: `./gradlew test -Pedition=government` + `./gradlew ciE2eTest` with `SPRING_PROFILES_ACTIVE=govcloud` set (note: `-Pedition=government` alone does not activate govcloud or CAC auth behavior — profile must be explicitly set)
   - Zero security/compliance regressions throughout burn-in window (OWASP adversarial suite, classification retrieval safety, audit completeness)
   - Daily evidence logged in `docs/evals/scif_burnin_evidence_log.md` with PASS/FAIL per gate, consecutive pass count, and notes
   - Unfreeze decision requires all three criteria met + explicit approval in handoff before any deferred backlog execution
7. Banner acknowledgment compliance gate (STIG SRG-APP-000068):
   - Govcloud banner enabled by default; test must include the full `/api/auth/login-banner` → `/api/auth/acknowledge-banner` sequence
   - Verify: banner blocks API calls before acknowledgment, acknowledgment unblocks subsequent calls, banner state persists across page refresh
   - Govcloud runs without banner acknowledgment evidence are incomplete and must be marked `FAIL`

### 2.8 Connector Sync and SDK Validation (Required)

Required per-connector validation (S3, SharePoint, Confluence, Google Drive):
- Incremental sync correctness: fingerprint tracking detects new/modified/deleted documents
- Full sync correctness: initial sync ingests all documents in target scope
- Metadata extraction: connector-specific metadata (path, permissions, timestamps) propagated to vector store
- Authentication: connector-specific auth (IAM role, OAuth, API token) validated per connector

Required connector framework validation:
- Circuit breaker exhaustion: after threshold failures, connector enters open state and rejects new sync attempts
- Circuit breaker recovery: after cool-down period, connector transitions to half-open and resumes
- Concurrent sync safety: two simultaneous syncs for the same connector must not produce duplicate documents or corrupt state
- Large-scale sync: 10,000+ file sync completes within resource envelope without OOM or timeout
- Legacy metadata migration: `ConnectorLegacyMetadataMigrationService` correctly migrates old-format metadata without data loss
- Enable/disable isolation: disabled connector (`sentinel.<connector>.enabled=false`) must not leak data, produce side effects, or register beans

Pass criteria:
- Zero duplicate documents after concurrent sync
- Zero data loss in migration
- Circuit breaker triggers within configured threshold

### 2.9 Feature Flag Interaction and Edition Enforcement (Required)

Required flag-matrix validation:
- Incompatible flag combination detection: known-incompatible pairs (document in `application.yaml` comments) must produce clear startup error or log warning, not silent misbehavior
- Edition-based flag enforcement: government-only flags (e.g., `AGENTIC_ENABLED`) must be inert in enterprise/trial JARs (code physically excluded by Gradle `editionExcludes`)
- Flag-flip regression: toggling a flag ON then OFF must produce identical retrieval/generation behavior to never-ON state
- Default-ON strategy verification: all strategies listed as ON by default must be active in a clean startup with no overrides
- Default-OFF strategy verification: all strategies listed as OFF must remain inactive until explicitly enabled
- New feature flags (PRs #320-#324, #442) requiring flag-matrix validation:
  - `CATRAG_ENABLED` (default `false`) — chain-completeness scoring augmentation in HybridRAG
  - `SENTINEL_CITATION_REALTIME_VERIFICATION_ENABLED` (default `false`) — post-LLM citation accuracy validation
  - `sentinel.hifirag.reranker.bandit-routing.enabled` (default `false`) — epsilon-greedy/UCB1 reranker arm selection
  - `SENTINEL_RAGBOOST_ENABLED` (default `false`) — feedback-driven document re-ranking via `DocumentBoostService`
  - RAGBoost tuning sub-flags:
    - `SENTINEL_RAGBOOST_ALPHA` (default `0.3`) — boost multiplier range `[1.0 - alpha, 1.0 + alpha]`
    - `SENTINEL_RAGBOOST_MIN_FEEDBACK_COUNT` (default `3`) — minimum feedback count before boost is applied
  - Reranker tuning sub-flags (verify defaults and override behavior):
    - `sentinel.hifirag.reranker.colbert-sidecar-enabled` (default `false`) — ColBERT sidecar availability
    - `sentinel.hifirag.reranker.bandit-routing.epsilon` (default `0.1`) — exploration probability
    - `sentinel.hifirag.reranker.bandit-routing.ucb-exploration-coefficient` (default `1.414`) — UCB1 exploration constant
    - `sentinel.hifirag.reranker.bandit-routing.warmup-rounds` (default `10`) — per-arm round-robin before UCB1
    - `sentinel.catrag.chain-completeness-weight` (default `0.3`) — chain boost weight
    - `sentinel.catrag.jaccard-threshold` (default `0.20`) — minimum Jaccard overlap for chain scoring
  - Flag-flip regression: toggling these ON then OFF must produce identical behavior to never-ON state
  - Incompatible combination check: bandit routing requires HiFi-RAG enabled (`HIFIRAG_ENABLED=true`) and at least one reranker sidecar; CatRAG requires HybridRAG enabled (`HYBRIDRAG_ENABLED=true`); RAGBoost requires sufficient feedback data (cold-start with < `min-feedback-count` docs must apply no boost, not error)
  - Medical TLS enforcement (PR #415): `sentinel.security.enforce-tls=true` now applies to medical mode regardless of HIPAA strict setting — test both `hipaa-strict=false, enforce-tls=true` (must reject) and `hipaa-strict=true, enforce-tls=false` (must reject)

Required edition isolation checks:
- Government code packages must not exist in enterprise/trial/medical build artifacts
- Medical code packages must not exist in enterprise/trial build artifacts
- Edition verification via `EditionIsolationTest` must pass for all editions

### 2.10 Evaluation Infrastructure Validation (Required)

Required ground-truth dataset integrity checks:
- Manifest validation: required fields (`edition`, `dataset_id`, `version`, `min_cases`, `cases`, `sha256`)
- Dataset version: expected version `v3-hardened`
- Dataset ID uniqueness across editions
- Case ID uniqueness per edition with correct prefix (`ent-`, `med-`, `gov-`)
- SHA-256 fingerprint reproducibility across CI runs
- Per-edition minimum case count: >= 20 absolute floor
- All three categories present per edition: factual, procedural, analytical
- Total cases across editions: >= 110

Required self-consistency scoring:
- Ground-truth answers scored against their own evidence using token-overlap Jaccard
- Aggregate average faithfulness must be > 0.0

Required benchmark adapter validation:
- SENTINEL native format: load, validate schema, normalize to internal EvalCase
- RAGBench format: domain-to-category mapping (factual, procedural, analytical)
- T2-RAGBench format: two-tier with question_id and context passages
- WildGraphBench format (PR #323): `GraphRetrievalEvalRunner` loads WildGraphBench datasets via `BenchmarkDatasetAdapter`, queries `HGMemQueryEngine` with `deep=true`, scores with `GroundTruthEvalService`. Required validation:
  - Dataset SHA-256 fingerprint reproducibility
  - Per-case graph metrics: `nodesTraversed`, `graphEnhanced`, `matchedEntities`, `relatedEntities`
  - Aggregate metrics: `graphEnhancementRate`, `avgNodesPerCase`
  - Error resilience: failed individual cases must not abort the batch (fallback to empty metrics)

Required embedding model benchmark gates:
- `min-quality-delta`: new model must meet or exceed threshold vs current model
- `max-latency-ratio`: new model latency must not exceed ratio vs current model
- Go/no-go decision documented with evidence before model migration

Required eval result management:
- Eval result persistence and retrieval
- Regression detection: comparison against prior release baseline with alert on threshold breach
- Failed eval case diagnostics: trace-linked artifacts (`eval_failure_report.jsonl`) for every failure

Required release evidence integrity:
- Compliance evidence artifacts must be checksummed (SHA-256) per release
- Evidence manifest must link each artifact to its gate outcome and release tag

### 2.11 Edge Deployment Profile Validation

Tactical edge profiles (`edge-s`, `edge-m`, `edge-l`) are additive overlays on the govcloud profile designed for resource-constrained dismounted or vehicle-mounted deployments. `frontier`, `foundation`, and `govcloud-qwen3` are **not** tactical edge profiles — they are overlay profiles for cloud/GPU variant configurations.

Current CI edge coverage: **Edge-S only** (`ciEdgeDegradationE2eTest`). Edge-M and Edge-L require dedicated hardware and are validated out-of-band.

| Profile | Activation | Resource Envelope | CI Coverage |
|---------|-----------|-------------------|-------------|
| `edge-s` | `SPRING_PROFILES_ACTIVE=govcloud,edge-s` | 8 vCPU / 32 GB, dismounted ops | `ciEdgeDegradationE2eTest` |
| `edge-m` | `SPRING_PROFILES_ACTIVE=govcloud,edge-m` | Medium vehicle-mounted | Out-of-band only |
| `edge-l` | `SPRING_PROFILES_ACTIVE=govcloud,edge-l` | Large vehicle-mounted | Out-of-band only |
| `govcloud-qwen3` | Config overlay | Air-gapped, Qwen3-Embedding-8B GPU | Separate validation |
| `frontier` | Additive overlay | Enables ContextualRetrieval + SpeculativeRAG | `ciFrontierE2eTest` |
| `foundation` | Additive overlay | Cloud model integration (OpenAI/Anthropic/VertexAI) | `ciFoundationE2eTest` |

Required checks:
- Profile-specific strategy activation: only declared strategies are active
- Hardware-tier latency validation: p95 within edition SLO on target hardware class
- Resource envelope: memory, CPU, GPU utilization within declared limits
- Air-gap compliance: govcloud/edge profiles must not make external network calls
- Edge-S degradation behavior: graceful fallback when resources are constrained (no crash, no silent data loss)

### 2.12 Observability Stack Validation (Required)

Added in PRs #413-#414. Required when any change touches metrics, error handling, or monitoring config.

Required Prometheus endpoint validation:
- `GET /actuator/prometheus` returns HTTP `200` with `Content-Type: text/plain` and valid Prometheus exposition format
- Endpoint permits anonymous scrape access (current `SecurityConfig` allows unauthenticated access for Prometheus scraping)
- Key metrics present: request latency histograms, active session counts, retrieval operation counts, embedding latency
- No sensitive data (PII, document content, internal paths) in metric labels or values

Required CWE-209 validation (error detail suppression):
- API error responses must not include stack traces, internal class names, file paths, or DB query text
- All `ApiErrorResponse` payloads must contain only `error`, `code`, and `timestamp` fields — no additional diagnostic fields in production mode
- Verify with: (1) an invalid request that triggers an internal exception, (2) a malformed query that triggers a DB error — both must return sanitized `ApiErrorResponse` only

Required Grafana dashboard correctness:
- Heap panel aggregation expression must produce non-null values under load (PR #414 regression: stale aggregation expression returned no data)
- PrometheusRule alert expressions must fire correctly under simulated threshold breach
- Alert recovery: alerts must clear when condition resolves

Pass criteria:
- `/actuator/prometheus` accessible to authenticated requests only
- Zero stack trace / internal path leakage in any API error response
- All Grafana panels return non-null data under load
- At least one PrometheusRule alert fires and clears correctly in staging validation

## 3. Unified Execution Model

### 3.1 Runtime Execution Mode

- UI-first manual/interactive execution is mandatory and release-blocking for functional acceptance on every release.
- Non-UI automated suites remain mandatory for safety-net regression and CI quality gates.
- All significant failures require root-cause labeling by layer: retrieval, orchestration, generation, security, or UI contract.
- CLI-only execution does not satisfy this gate when a headed-browser run is required. If headed execution is unavailable, mark the run `BLOCKED` and final decision `FAIL` until manual UI evidence is captured.

Playwright UI test suite (`tools/playwright-runner/`) release-gate mapping:

Preferred operator entrypoints are the PS1 wrappers — they configure env vars and auth mode correctly. Use raw `.js` files only for custom overrides.

| Runner | Operator Entrypoint | Trigger | Required For |
|--------|---------------------|---------|-------------|
| `run-ui-tests.js` | `run-ui-suite.ps1` | Every release | All editions |
| `run-ui-govcloud.js` | `run-govcloud-ui.ps1` | Government edition releases | Government/govcloud |
| `run-ui-pii.js` | `run-ui-suite.ps1` (PII scenario) | Any release with PII/redaction changes | All editions |
| `run-ui-flags.js` | `run-flag-matrix.ps1` | Any release with feature flag changes | All editions |
| `run-ui-graph-styles.js` | `run-ui-suite.ps1` (graph scenario) | Any release with graph/entity visualization changes | All editions |
| `run-ui-investigate.js` | Direct node invocation | Debug/investigation only | **Non-gating** — not a release gate |

Manual UI gate enforcement:
- Every triggered runner in the table above is a required gate for that release context.
- Required runner outcomes must be `PASS`; `FAIL`, `BLOCKED`, `SKIPPED`, or `NOT RUN` are release-blocking.
- This gate is not waivable in normal release flow. If environment constraints prevent execution, release status remains `FAIL` until the required headed run evidence is provided.

### 3.1.1 SSE Streaming Reliability Requirements

`/api/ask/stream` is the primary query path for the UI. Streaming failures are release-blocking if they affect end-user query completion.

Required streaming reliability checks:
- **Keepalive**: server must emit SSE comment pings at intervals ≤ 30s during the silent retrieval phase (before LLM tokens begin). Absence of keepalive causes Tomcat/proxy to close the connection before any response is produced.
- **Async timeout**: `spring.mvc.async.request-timeout` must be set to ≥ 300000ms (5 minutes) to allow slow LLM generation to complete without connection teardown.
- **Connection survival**: SSE connection must survive the full retrieval + LLM generation cycle for a worst-case slow query (target: 30s retrieval + 60s generation = 90s minimum survival without keepalive).
- **Graceful error delivery**: pipeline errors after stream opens must arrive as `event: error` SSE events, not as silent connection drops or HTTP error codes mid-stream.
- **Emitter lifecycle safety**: `emitter.complete()` must be called exactly once; duplicate complete calls must be guarded with `AtomicBoolean` to prevent `ResponseBodyEmitter has already completed` errors.
- **ThreadLocal propagation**: `SecurityContext` and `WorkspaceContext` must be captured before async dispatch and propagated into the async thread — access to ThreadLocals inside `CompletableFuture.runAsync()` without explicit capture will produce null user/workspace.

Required SSE error case coverage:
- Unauthenticated request: HTTP `401` before stream opens (Spring Security pre-upgrade intercept)
- Missing `q` parameter: HTTP `400` before stream opens
- Clearance-denied (classification too low): `event: error` SSE event after stream opens
- Sector-denied (user lacks access to requested `dept`): `event: error` SSE event after stream opens
- Workspace-quota-denied: raw JSON from `WorkspaceFilter` before stream opens (not `ApiErrorResponse`)
- Banner-ack required: simple map response before stream opens (not `ApiErrorResponse`)

Test method (happy path):
- Submit a query that exercises the full RAG pipeline (retrieval + reranking + LLM generation)
- Observe SSE event stream: `connected` → one or more `step` events → `token` events → `complete`
- Confirm no connection drop before `complete` event
- Confirm keepalive comment lines appear in stream during silent phases

OIDC browser SSO flow coverage (must be included in enterprise/OIDC UI runs):
- `GET /api/auth/mode` — SPA discovers auth mode on load
- `GET /api/auth/oidc/authorize` — initiates IdP redirect
- `GET /api/auth/oidc/callback` — exchanges code, establishes session
- `?auth_error=*` redirect handling — error query param must surface a visible error to the user, not a blank page
- Evidence required: IdP redirect initiated, callback received, post-login admin action performed with artifact

### 3.1.2 UI Behavioral Contract Requirements

These UI behaviors must be explicitly validated in headed Playwright runs.

**Prompt injection blocked response (`blocked: true`)**:
- When `EnhancedAskResponse.blocked = true`, the frontend must render a distinct amber-colored warning card — not a standard error message and not a blank response panel.
- The "Try again" action on the warning card must preserve the original query text in the input field for editing (not clear it).

**Session lifecycle UI**:
- `?error=session_expired` URL parameter must trigger a **persistent** (non-auto-dismissing) banner at the top of the page. Auto-dismissal within a timeout is a fail.
- Replayed/restored session messages must display a single "Session restored" banner at the top of the chat — individual `RESTORED` labels on each message are a fail.

**System offline / document count**:
- When the system is offline or the telemetry poll fails, the document count must display as a grayed-out `-- docs` indicator — not zero, not empty, not a spinner that never resolves.
- On system recovery (next successful telemetry poll), document count opacity must be restored immediately — grayed-out state must not persist after recovery.

### 3.2 Canonical Coverage Buckets

The unified plan must execute all buckets below each release cycle:
- Response semantic correctness and intent fidelity
- Visual response-format compliance
- Retrieval correctness and ranking
- Generation quality and refusal quality
- Strategy-specific behavior (all enabled strategies and critical pairwise interactions)
- Security and adversarial behavior (prompt injection, poisoning, exfiltration attempts)
- PII/PHI compliance and audit behavior
- Multi-tenant and RBAC isolation
- Source evidence rendering and graph/state correctness
- Ingestion robustness and connector sync integrity
- Performance, resilience, and scale behavior
- Profile/edition isolation and policy enforcement
- Embedding model migration (re-embedding validation: DRY_RUN mode, COMPARE mode, checkpoint recovery, rollback correctness, batch-size boundary behavior)

### 3.3 Scoped Execution Order (Operational)

When full-suite runtime is constrained or recovery debugging is in progress, execute in this order:
1. `queries` (core relevance, abstention, graph coherence)
2. `security,upload` (injection/poisoning + ingest controls)
3. `session,pii,rbac` (state continuity, redaction, authorization)
4. Full-suite rerun for final sign-off artifact set

## 4. Strategy and Evaluation Addendum (From Testing Guide)

### 4.1 Golden Dataset Governance

Golden datasets are living release artifacts.
- Every production failure with confirmed root cause becomes a regression fixture.
- Golden datasets are versioned with code and tied to release tags.
- Dataset categories must include factual, multi-hop, unanswerable/refusal, and adversarial sets.

### 4.2 Strategy-Specific Evaluation

Per-strategy validation is required for all implemented strategies:

Core strategies (19):
- HybridRAG (ON by default — primary vector+keyword RRF fusion)
- AdaptiveRAG (ON — query complexity classification and routing)
- CRAG (ON — corrective RAG with query rewrite on low confidence)
- RAGPart (ON — corpus poisoning defense via partitioning)
- QuCo-RAG (ON — uncertainty quantification and hallucination detection)
- HiFi-RAG (OFF — hierarchical filtering + cross-encoder/ColBERT reranking)
- MegaRAG (ON — multimodal knowledge graph: images, charts, tables)
- MiA-RAG (ON — hierarchical summaries for document coherence)
- BiRAG (ON — bidirectional RAG with experience store)
- Self-RAG (OFF — self-reflective generation)
- HyDE (OFF — hypothetical document embeddings)
- Graph-O1 (OFF — graph reasoning)
- HGMem (ON for indexing, OFF for query-time — hypergraph memory with entity extraction)
- Agentic (OFF — tool-augmented multi-hop reasoning via `AgenticTool` interface contract; `AGENTIC_ENABLED` is a shared-core flag, not edition-gated at build time)
- SelfCorrective (OFF — bounded CRAG+QuCoRAG retry loop with hallucination validation)
- SpeculativeRAG (OFF — parallel draft+verify generation)
- RAPTOR (OFF — recursive abstractive processing for tree-organized retrieval)
- Router (ON — query routing/classification across strategy selection)
- Thesaurus (ON — domain acronym/synonym expansion)
- ContextualRetrieval (OFF by default; ON in enterprise and frontier profiles — prepends chunk-level document context at ingest time; `CONTEXTUAL_RETRIEVAL_ENABLED`)

Deferred backlog strategies — **implemented and CI-covered** via `DeferredRagPipelineE2eTest` (`ciDeferredRagE2eTest`):
- HtmlRAG (OFF — HTML-native retrieval with structure-aware chunking)
- CAG (OFF — cache-augmented generation for repeated query patterns)
- UniversalRAG (OFF — format-agnostic universal retrieval across heterogeneous document types)
- SitEmb (OFF — site-contextual embedding with configurable prefix tokens for domain anchoring; `SITEMB_MAX_PREFIX_TOKENS` default 50)

Strategy augmentations (not standalone strategies — augment existing strategy pipelines):
- CatRAG (OFF — chain-completeness scoring, augments HybridRAG post-RRF fusion via `ChainCompletenessScorer`)
- C²-Cite (OFF — real-time citation verification + corpus-grounded citation spans; augments `RagOrchestrationService` post-LLM via `CitationVerificationService.verifyResponse()` and `CorpusGroundingService.groundCitations()`)
- RAGBoost (OFF — feedback-driven document re-ranking; `DocumentBoostService` applies boost multiplier at retrieval time based on `FeedbackService` positive/negative counts; `SENTINEL_RAGBOOST_ENABLED` default `false`)

Each strategy run must capture:
- Routing decision and rationale
- Retrieval set changes vs baseline
- Citation quality impact
- Latency delta vs baseline
- Failure-mode behavior when disabled
- Activation/deactivation correctness: toggling a strategy ON then OFF must produce identical state to never-ON

Strategy-specific validation requirements:

**RAPTOR** (ingestion-time tree building):
- Hierarchical summarization tree building at ingest time (multi-level hierarchy validation)
- Integration with `SecureIngestionService` → `RaptorService.buildTree()` path
- Budget splitting validation (`summaryTopK` vs `retrievalTopK` — neither path may starve the other)
- Workspace isolation during tree retrieval (cross-workspace tree leakage = hard fail)
- Summarization failure handling at scale (partial tree build must not corrupt retrieval)

**SelfCorrective** (bounded retry meta-strategy):
- Retry exhaustion behavior (max retries reached → returns best-effort, not error)
- Hallucination validation post-generation (QuCoRAG confidence scoring on output)
- Grader confidence threshold boundary cases (scores at exact threshold)
- Full pipeline integration: CRAG grader → QuCoRAG → rewrite → re-retrieve → validate loop

**HiFi-RAG reranker fallback chain**:
- Mode "auto" selection logic:
  - When bandit routing disabled (default): ColBERT sidecar → cross-encoder sidecar → dedicated model → LLM → keyword
  - When bandit routing enabled (`sentinel.hifirag.reranker.bandit-routing.enabled=true`): epsilon-greedy/UCB1 selects between ColBERT and cross-encoder arms → fallback to dedicated model → LLM → keyword if bandit returns null
- Col-Bandit validation requirements:
  - Warmup phase: round-robin pulls across all arms for configured `warmup-rounds` before UCB1 kicks in
  - UCB1 exploitation: after warmup, arm with higher cumulative reward is selected more frequently
  - Epsilon exploration: with non-zero epsilon, exploration arm selection occurs at configured probability
  - Reward attribution: reward is credited only to the arm that actually produced results (not fallback arm)
  - Cold start: stats reset on restart triggers natural exploration via round-robin warmup
  - Single-arm degradation: with only one sidecar available, bandit always selects the available arm
- Graceful degradation when preferred reranker is unavailable
- SSRF protection on sidecar URLs: `ColBertSidecarClient` and `RerankerSidecarClient` use `createNoRedirectRestTemplate()` to reject 3xx redirects — verify redirect rejection is part of the security regression suite
- Sidecar timeout and health check behavior
- ColBERT MaxSim scoring correctness vs cross-encoder baseline

**CatRAG** (chain-completeness augmentation of HybridRAG):
- Chain-completeness scoring applied post-RRF fusion when `CATRAG_ENABLED=true`
- Evidence chain coverage: multi-hop queries should show improved chain coverage vs baseline HybridRAG
- No-op when disabled: toggling OFF must produce identical results to HybridRAG without CatRAG
- Requires `HYBRIDRAG_ENABLED=true` as a prerequisite

**C²-Cite** (real-time citation verification + corpus grounding augmentation):
- Post-LLM citation accuracy validation via `CitationVerificationService.verifyResponse()`
- Corpus-grounded citation spans via `CorpusGroundingService.groundCitations()` on `/api/ask/enhanced` response path (PR #439)
- Source deduplication: `groundCitations()` must not return duplicate source entries (PR #441 regression fix — duplicate sources are a hard fail)
- `/api/ask/enhanced` `citations` field is gated by `SENTINEL_CITATION_CORPUS_GROUNDING_ENABLED` (not `SENTINEL_CITATION_REALTIME_VERIFICATION_ENABLED`) — frontend contract test must assert `citations` field present when `SENTINEL_CITATION_CORPUS_GROUNDING_ENABLED=true`, absent when false
- Timeout support: verification must not block response beyond configured timeout
- No-op when disabled: toggling OFF must produce identical response to unverified pipeline
- Verification failure handling: failed verification must not suppress a valid response

**RAGBoost** (feedback-driven document re-ranking):
- Boost multiplier applied at retrieval time: high-feedback documents float up, low-feedback documents float down
- Cold-start behavior: documents with fewer than `SENTINEL_RAGBOOST_MIN_FEEDBACK_COUNT` feedback events must receive neutral boost (multiplier = 1.0), not error
- Boost range: `[1.0 - alpha, 1.0 + alpha]` — verify floor/ceiling are respected at extreme feedback counts
- Flag-flip regression: toggling `SENTINEL_RAGBOOST_ENABLED` ON then OFF must produce identical retrieval ranking to never-ON state
- Isolation: boost scores must not cross workspace boundaries (workspace A feedback must not affect workspace B retrieval)
- No-op when disabled: RAGBoost OFF must produce bit-identical retrieval results to baseline HybridRAG

**Agentic tool contract** (A-RAG interface formalization):
- `AgenticTool` interface: `name()`, `description()`, `canHandle(ToolContext)`, `execute(ToolContext)` contract
- `ToolContext` carries query, department, and typed parameters via `param(key, type)`
- All registered tools must pass `ToolPermissionGuard` deny-by-default access control
- Registered tools: `currentDate`, `calculator`, `getDocumentInfo`, `getAdjacentChunks`, `searchThesaurus`
- Tool name must match `ToolPermissionGuard` allowlist; unlisted tools must be rejected

### 4.3 Continuous Monitoring and Drift

Production monitoring must include:
- Retrieval drift trends (precision/recall degradation)
- Generation drift trends (faithfulness/relevancy decline)
- Hallucination trend alerts
- Injection/poisoning trend signals
- Confidence/uncertainty threshold drift

Alert thresholds and ownership must be explicit per metric stream.

### 4.4 Feedback Curation Pipeline Validation

Required validation for embedding fine-tuning readiness:
- Quality scoring: composite metric scoring produces consistent, reproducible scores for identical input pairs
- HIPAA sector filtering: medical-sector feedback pairs must be excluded from non-medical curation datasets
- Deduplication: SHA-256 fingerprint prevents duplicate pairs; re-submitting identical feedback does not create duplicates
- Approval/rejection routing: pairs above `approve-threshold` are auto-approved; pairs below `review-threshold` are auto-rejected; middle band requires manual review
- Holdout set assignment: holdout ratio is respected (default 15%); holdout set is stable across runs for eval reproducibility
- Dataset readiness: minimum pair counts are enforced before dataset is marked ready for training
- Concurrent curation safety: two simultaneous curation runs must not produce duplicate pairs or corrupt fingerprint state

## 5. Traceability and Sign-Off Requirements

Each test cycle must publish:
- Build SHA / branch / edition / profile
- Corpus version and fixture hash
- Metric summaries and gate outcomes
- Security-failure inventory
- Manual UI gate evidence for all triggered runners (runner name, PASS/FAIL, artifact path, execution mode headed/headless)
- Waivers/accepted risks with owner and expiry
- Checksummed compliance evidence manifest (SHA-256 per artifact, linked to gate outcome and release tag)
- Ground-truth eval dataset version and fingerprint

Release sign-off requires:
- Product owner approval
- Security owner approval
- Platform owner approval

## 6. Test Run Log Template (Canonical)

Use this template for every formal run.

- Date:
- Tester:
- Environment (profile/edition):
- Auth mode (`DEV` / `STANDARD` / `OIDC` / `CAC`):
- Credential source chain checked (highest to lowest):
- Credential source selected for this run:
- Non-production fallback credential used (`Test123!`): YES/NO
- Git SHA:
- Corpus version/hash:
- Model backend + embedding model:
- Feature flag snapshot (non-default overrides):
- Burn-in window status (government only): Day N of 7 / N/A
- Manual UI gate status: PASS/FAIL
- Response integrity gate status (Section 1): PASS/FAIL
- Golden answer contract version/hash:
- Dual-evaluator agreement rate:
- Triggered UI runners and outcomes:
  - `run-ui-tests.js`:
  - `run-ui-govcloud.js` (if applicable):
  - `run-ui-pii.js` (if applicable):
  - `run-ui-flags.js` (if applicable):
  - `run-ui-graph-styles.js` (if applicable):
- UI evidence artifact paths (JSON/log/screenshots):
- Auth/session contract evidence artifact paths (request/response captures):

Metric summary:
- Retrieval metrics pass/fail:
- Generation metrics pass/fail:
- Security suites pass/fail:
- Response-integrity suites pass/fail:
- Visual format compliance pass/fail:
- Performance/soak pass/fail:
- Soak duration + load profile:
- Soak stability verdict (latency/error/resource trends):
- Soak artifact bundle path(s):

CI tier results:
- Tier 0 `preflight` (docs-only skip gate): PASS/SKIP
- Tier 1 `unit-tests` (Unit + Lint + Gitleaks + Pipeline E2E + OIDC E2E + Cross-Tenant E2E + Streaming Parity): PASS/FAIL
- Tier 2a `e2e-suites` (Enterprise + Frontier + Deferred + Foundation + Edge, parallel matrix): PASS/FAIL
- Tier 2b `enterprise-realism` (Ground-Truth Eval): PASS/FAIL
- Tier 3 `build` (Enterprise build + SBOM + SonarCloud — required merge gate): PASS/FAIL (SonarCloud: informational)

Hard-fail checks:
- Corpus baseline met for release candidate (7 golden + 3,000 randomized): PASS/FAIL
- Cross-tenant leakage: PASS/FAIL
- Injection/exfiltration success: PASS/FAIL
- PII/PHI leakage: PASS/FAIL
- Regression >5%: PASS/FAIL
- Faithfulness regression >2pp: PASS/FAIL
- Blocker-case semantic drift vs previous release: PASS/FAIL
- Blocker/high format-contract compliance: PASS/FAIL
- Auth/session negative-path contracts (Section 2.4 table): PASS/FAIL
- Credential provenance captured and approved for environment: PASS/FAIL
- Non-production fallback credential used in release run: PASS/FAIL (must be FAIL if YES)
- Embedding overflow failures in logs: PASS/FAIL
- False-abstention incidents: PASS/FAIL
- Semantic-anchor mismatches or context contamination incidents: PASS/FAIL
- NO_RETRIEVAL source leakage incidents: PASS/FAIL
- Government SCIF fail-closed gate (when applicable): PASS/FAIL
- Government unsafe-config startup rejection verified (when applicable): PASS/FAIL
- Government burn-in daily pass (when applicable): PASS/FAIL/N/A
- Connector sync validation (when connectors enabled): PASS/FAIL/N/A
- Manual UI required-runner gate(s): PASS/FAIL

QC counters:
- `sources>=1` with plain no-records response count:
- Context queries with placeholder-only entity graph count:
- Graph/response coherence violations count:
- Semantic mismatches count:
- Context contamination incidents count:
- Source-claim mismatch incidents count:
- Ontology typed-relation validation errors count:

Findings:
1. 
2. 
3. 

Waivers (if any):
1. 
2. 

Non-waivable rule:
- Manual UI required-runner gates are non-waivable for release sign-off.

Final decision: PASS / CONDITIONAL PASS / FAIL

## 7. Source-Document Crosswalk

This unified document consolidates canonical policy only. Legacy deep-dive runbooks remain available as reference artifacts:
- Archived runbook: `archive/RAG_MASTER_TEST_PLAN.md`
- Archived strategy guide: `archive/SENTINEL-RAG-Testing-Guide.md`
- Comprehensive parity baseline: `SENTINEL_ENTERPRISE_TESTING_COMPREHENSIVE.md`

Crosswalk intent:
1. Sections `1`, `2`, and `3` define response integrity, required coverage, and execution model.
2. Section `4` defines strategy/evaluation governance that remains mandatory.
3. Sections `5` and `6` define traceability and sign-off artifacts for every formal run.
4. Archived documents are reference-only for historical detail and must not introduce conflicting gates, duplicate templates, or alternate pass criteria.





