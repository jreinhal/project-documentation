# Clean Code Consolidated Implementation Plan

Date: 2026-02-22
Status: Final consensus
Source: Codex + Claude + Gemini consensus cycle

## Objective
Improve maintainability and code quality without breaking runtime behavior.

## Hard Constraints
- Do not break existing behavior.
- One phase per PR.
- Next phase cannot start until prior phase PR is merged.
- Each phase PR must include:
  - behavior-preservation proof,
  - contract/regression test evidence,
  - rollback plan,
  - zero unresolved blocking review threads before merge.
- If a phase introduces behavior drift, fix in the same phase before any next PR.

## Final Phase Plan

### Phase 1 - Security and Safety Fixes (LOW)
Scope:
- Fix ThreadLocal leak by adding `RequestExecutionContext.clear()` in `finally` for:
  - `RagOrchestrationService.ask`
  - `RagOrchestrationService.askEnhanced`
  - `RagOrchestrationService.askStream`
- Add explicit filter ordering for `PreAuthRateLimitFilter`.
- Normalize URI before exemption checks in pre-auth rate limiting.
- Fail closed on wildcard CORS in non-dev profiles.

Merge gate:
- Security tests green, full test suite green, E2E green.

### Phase 2 - Safety Harness and Test Infrastructure (LOW)
Scope:
- Characterization tests for ask/askEnhanced/askStream and ingestion.
- Architecture boundary tests (controller thinness, service boundaries).
- Add test builders/factories for stable fixture setup.
- Cover currently untested controllers with basic contract tests.

Merge gate:
- No API/schema drift.

### Phase 3 - Controller De-duplication and Extraction (MODERATE)
Scope:
- Make controller transport-only.
- Centralize auth/sector validation helper logic.
- Remove duplicated retrieval helper logic from controller.
- Extract endpoint groups into focused controllers as needed.

Merge gate:
- Endpoint contracts unchanged, SSE behavior unchanged.

### Phase 4 - Orchestration Decomposition (HIGH)
Scope:
- Decompose `RagOrchestrationService` into collaborators:
  - security checks,
  - retrieval orchestration,
  - response synthesis,
  - post-processing/citation/evidence.
- Keep public entrypoints stable.

Merge gate:
- No quality regression in eval gates.
- Latency regression within agreed tolerance.

### Phase 5 - Ingestion Separation and PII Registry (HIGH)
Scope:
- Split `SecureIngestionService` responsibilities:
  - validation,
  - extraction,
  - chunking,
  - checkpoint/resilience,
  - persistence/indexing.
- Extract PII pattern registry from redaction algorithm flow.

Merge gate:
- Ingestion and connector tests green, security checks unchanged.

### Phase 6 - Vector Store Decomposition (HIGH)
Scope:
- Refactor `LocalMongoVectorStore` to delegate:
  - embedding generation/batching,
  - dense/sparse search providers.
- Preserve existing VectorStore external behavior.

Merge gate:
- Dense/sparse retrieval parity and latency checks pass.

### Phase 7 - RAG Strategy Consolidation (MODERATE)
Scope:
- Define shared strategy interface.
- Extract shared strategy utilities (department normalization, keyword extraction, retrieval budgeting).
- Standardize strategy error contract.

Merge gate:
- Strategy tests and E2E suites pass without quality regression.

### Phase 8 - Frontend Modularization with Leak/Race Hardening (HIGH)
Scope:
- Modularize `sentinel-app.js` by domain.
- Fix memory leak vectors (interval/observer/listener cleanup).
- Fix global-state race conditions.
- Deduplicate shared utilities between app/admin scripts.

Merge gate:
- UI automation green, manual smoke green, no console regression.

### Phase 9 - API Error and DTO Standardization (MODERATE)
Scope:
- Standardize exception-to-response mapping.
- Replace string-encoded service errors with domain exceptions where safe.
- Normalize immutable DTO response patterns.

Merge gate:
- API error contract tests pass, frontend handling verified.

### Phase 10 - Test Seam Cleanup (LOW)
Scope:
- Replace reflection-heavy test wiring with explicit constructors/builders.
- Reduce `CALLS_REAL_METHODS` dependence.
- Keep or improve coverage quality.

Merge gate:
- Equivalent coverage retained, suite remains green.

## Execution Status (As Of 2026-02-23)

All defined phases in this plan are complete and merged.

| Phase | Status | Evidence |
|---|---|---|
| Phase 1-3 (combined delivery) | COMPLETE | PR #208 (`fix/phase1-3-safety-harness-refactor`) |
| Phase 4 (decomposition slices) | COMPLETE | PR #214, PR #215 |
| Phase 5 | COMPLETE | PR #218 |
| Phase 6 | COMPLETE | PR #219 |
| Phase 7 | COMPLETE | PR #220 |
| Phase 8 | COMPLETE | PR #221 |
| Phase 9 | COMPLETE | PR #222 |
| Phase 10 | COMPLETE | PR #223 |

Phase advancement rule at `PR=0`:
- If additional refactor work is desired, create a new phase charter (Phase 11+) with scope, tests, and rollback criteria before source edits.
- Until a new charter is approved, remain in monitoring mode and do not start ad hoc refactor changes.

## Execution Protocol
- Keep WIP focused to the current phase only.
- No source edits outside the active phase scope.
- Any spawned/dependent PR must follow the same full review cycle.
- Required checks and blocking reviews must be resolved before merge.

## Realtime Governance Protocol
- Codex is the document owner and moderator for this implementation plan.
- Codex may update this plan in real time whenever changes are necessary to reduce risk, preserve scope, or improve execution quality.
- Codex must post handoff updates so Claude remains current on:
  - plan changes,
  - phase decisions,
  - PR lifecycle events.
- Every time a PR is pushed/opened/updated for this plan, Codex posts a handoff entry with PR identifier, scope, and required reviewer actions.
- Claude monitors handoff and PR status, reviews each new PR, and responds to replies on Claude review comments until blocking threads are resolved.
- The no-break constraint remains mandatory: if behavior drift appears, stop and remediate before phase advancement.

## Stop Conditions
Pause and remediate before proceeding if any occur:
- API/SSE behavior drift,
- quality regression in evals,
- latency degradation beyond tolerance,
- security/compliance regression,
- unstable CI unrelated to active phase.

## Deliverable Note
This consolidated file replaces individual reviewer reports by explicit user directive.
