# SENTINEL Unified RAG Master Test Plan Review Recommendations

Review date: 2026-03-14
Source reviewed: `SENTINEL_UNIFIED_RAG_MASTER_TEST_PLAN.md`

Scope:
- Cross-checked the current plan against `.github/workflows/ci.yml`, `build.gradle`, active Spring profiles, auth/session/workspace controllers and filters, Playwright runners, and the currently wired RAG strategy surface.

## Summary

The master plan has already been partially refreshed, but it still drifts in four material areas:
- CI gate and job mapping
- auth/session/workspace contract coverage
- profile and Playwright runner guidance
- several stale or over-broad strategy/profile assertions

## Add

- Add explicit browser OIDC coverage.
  Include `/api/auth/mode`, `/api/auth/oidc/authorize`, `/api/auth/oidc/callback`, `auth_error=*` redirect handling, and the frontend SSO branch used by `sentinel-app.js`.

- Add login-banner and acknowledgment coverage.
  The plan should cover `/api/auth/login-banner`, `/api/auth/acknowledge-banner`, and the enforced pre-ack `403 banner_acknowledgment_required` path now used when banner acknowledgment is required.

- Add workspace auth coverage.
  Include `X-Workspace-Id` header resolution, SSE `workspaceId` query-param fallback, cookie fallback, regulated-edition hard `403 WORKSPACE_ACCESS_DENIED`, and non-regulated fallback to the default workspace.

- Add `/api/workspaces/**` auth and RBAC coverage.
  The current plan does not explicitly cover the authenticated list endpoint or the admin-only create/member/quota/usage management surface.

- Add current `/api/ask/stream` contract coverage beyond parity.
  Include SSE event expectations (`connected`, `step`, `token`, `complete`, `error`) plus unauthenticated, missing-query, clearance-denied, sector-denied, and workspace-quota-denied cases.

- Add newer auth/session endpoints.
  The plan should now cover `/api/auth/unlock-account`, `/api/auth/change-password`, and `/api/sessions/remaining`.

- Add Contextual Retrieval to the strategy inventory.
  It is implemented and explicitly exercised by the frontier profile, but it is not listed in the strategy section today.

## Edit

- Edit the CI tier tables and run-log template to match the actual workflow.
  Current workflow shape is:
  `preflight`
  `unit-tests` with `test -Plint -PlintWerror`, `ciE2eTest`, `ciOidcE2eTest`, `ciCrossTenantE2eTest`, `ciStreamingParityTest`
  `e2e-suites` matrix with `ciEnterpriseE2eTest`, `ciFrontierE2eTest`, `ciDeferredRagE2eTest`, `ciFoundationE2eTest`, `ciEdgeDegradationE2eTest`
  `enterprise-realism` with `ciGroundTruthEvalTest`
  `build` with enterprise packaging, `verifySbom`, and optional Sonar

- Edit stale CI test and class names.
  Use `DeferredRagPipelineE2eTest`, not `DeferredRagE2eTest`.
  Either remove or clearly label `ClassificationCeilingE2eTest` as non-gating, because it is not wired to a live `ci*` Gradle task or CI workflow job.

- Edit the Edge Deployment Profile Validation section.
  `frontier`, `foundation`, and `govcloud-qwen3` are additive overlays, not the actual tactical edge profiles.
  The real edge profiles are `edge-s`, `edge-m`, and `edge-l`, and current automated edge coverage is Edge-S only.

- Edit the Playwright release-gate section to reference real operator entrypoints.
  Prefer `run-ui-suite.ps1`, `run-govcloud-ui.ps1`, `run-flag-matrix.ps1`, and `tools/run_e2e_profiles.ps1` in the execution guidance rather than only raw Node filenames.

- Edit the manual UI gate language for DEV runs.
  DEV-mode UI runs can legitimately skip the login modal, so they do not provide final login/RBAC evidence for STANDARD, OIDC, or CAC sign-off.

- Edit credential notes in the run template.
  The fallback credential line still says `Test123!`, but the current seeded viewer credential is `viewer_unclass / TestPass123!`, and that account exists only when the `test-users` profile is active.

- Edit the C2-Cite response-shape guidance.
  The `/api/ask/enhanced` `citations` field is gated by `SENTINEL_CITATION_CORPUS_GROUNDING_ENABLED`, not `SENTINEL_CITATION_REALTIME_VERIFICATION_ENABLED`.

- Edit the deferred-strategy wording.
  HtmlRAG, CAG, UniversalRAG, and SitEmb are implemented and covered by `DeferredRagPipelineE2eTest`; they are no longer just backlog items.
  UniversalRAG and SitEmb descriptions should also be updated to match the current implementations.

- Edit or remove the percentage-based `@Value` migration progress note.
  The named remaining examples are valid, but the `~65% complete` figure is the part most likely to go stale.

- If the heading is meant as the review date, edit `Current Code Alignment (2026-03-15)`.
  The current local repo date is 2026-03-14, so the heading is future-dated unless that is intentional.

## Remove Or Clarify

- Remove the blanket claim that all auth/session negative-path responses use `ApiErrorResponse`.
  Current behavior is mixed across `ApiErrorResponse`, filter-generated JSON maps, empty bodies, and SSE `error` events.

- Remove the stale `sanitizeSessionTokenForFile()` assertion.
  Current session export filenames are generated from timestamp plus UUID, and the public API success response remains opaque through `SessionActionResponse`.

- Remove the “Agentic is government-only” wording.
  `AGENTIC_ENABLED` is a shared-core feature flag and is not edition-gated by the current build rules.

- Replace the govcloud burn-in command examples that use only `-Pedition=government` plus `ciE2eTest`.
  That setup does not actually activate `govcloud` or CAC behavior.

- Move `run-ui-investigate.js` out of the release-gate runner table, or mark it explicitly non-gating.
  It is an ad hoc investigation utility, not a stable release gate.

- Remove the statement that `/actuator/prometheus` must return `401` when unauthenticated.
  Current security config permits anonymous scrape access.

## Key Evidence Files

- `.github/workflows/ci.yml`
- `build.gradle`
- `src/main/resources/application.yaml`
- `src/main/resources/application-govcloud.yaml`
- `src/main/resources/application-frontier.yaml`
- `src/main/resources/application-foundation.yaml`
- `src/main/resources/application-govcloud-qwen3.yaml`
- `src/main/resources/application-edge-s.yaml`
- `src/main/resources/application-edge-m.yaml`
- `src/main/resources/application-edge-l.yaml`
- `src/main/java/com/jreinhal/mercenary/controller/AuthModeController.java`
- `src/main/java/com/jreinhal/mercenary/controller/OidcBrowserFlowController.java`
- `src/main/java/com/jreinhal/mercenary/controller/AuthController.java`
- `src/main/java/com/jreinhal/mercenary/controller/SessionController.java`
- `src/main/java/com/jreinhal/mercenary/controller/AskController.java`
- `src/main/java/com/jreinhal/mercenary/filter/BannerAcknowledgmentFilter.java`
- `src/main/java/com/jreinhal/mercenary/filter/WorkspaceFilter.java`
- `src/main/java/com/jreinhal/mercenary/workspace/WorkspacePolicy.java`
- `src/main/java/com/jreinhal/mercenary/workspace/WorkspaceController.java`
- `src/main/java/com/jreinhal/mercenary/config/SecurityConfig.java`
- `src/main/java/com/jreinhal/mercenary/service/ContextualRetrievalService.java`
- `src/main/java/com/jreinhal/mercenary/service/CorpusGroundingService.java`
- `src/main/java/com/jreinhal/mercenary/service/RagOrchestrationService.java`
- `src/main/java/com/jreinhal/mercenary/enterprise/memory/SessionPersistenceService.java`
- `src/main/java/com/jreinhal/mercenary/config/TestDataInitializer.java`
- `tools/playwright-runner/run-ui-suite.ps1`
- `tools/playwright-runner/run-govcloud-ui.ps1`
- `tools/playwright-runner/run-flag-matrix.ps1`
- `tools/playwright-runner/run-ui-tests.js`
- `tools/run_e2e_profiles.ps1`
