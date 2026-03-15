# Codex Review of Claude's Test Run Recommendations

Review date: 2026-03-15
Source reviewed: `SENTINEL_TEST_RUN_FINDINGS_AND_FIX_RECOMMENDATIONS_2026_03_15.md`

## Overall Assessment

Claude's write-up is directionally strong. The highest-value items are still:
- fixing the 4xx-vs-500 exception mapping gap
- fixing the Playwright deep-analysis false-positive
- fixing the stale injection-block assertion
- root-causing the real `POST /api/ingest/file` 500

Where I would tighten the recommendations is in the implementation detail. Two of the proposed fixes are too blunt for the current codebase:
- the Spring MVC exception fix should target the actual Boot 3.4 / Spring 7 exception mix, not only `NoHandlerFoundException`
- the upload fix should not add speculative duplicate/length exception branches until the live 500 is reproduced with server logs

## Agree

- `BUG-001` is real.
  `GlobalExceptionHandler` currently falls through to the catch-all `Exception` handler for unhandled MVC exceptions and returns `500 INTERNAL_ERROR`.
  That is the wrong contract for missing routes, missing required params, wrong methods, malformed bodies, and unsupported media types.

- `BUG-002` is real as a symptom of `BUG-001`.
  Missing `q` or `dept` on `/api/ask/stream` is a request-binding problem and should resolve as a `400`, not a `500`.

- `BUG-006` is a genuine test-runner defect.
  `ensureDeepAnalysis()` in `run-ui-tests.js` still treats `tab.style.display !== 'none'` as visibility, which misses parent-container hidden states.
  The same brittle pattern also exists in `run-ui-govcloud.js`, so the fix should be applied consistently across runners.

- `BUG-007` is a genuine test expectation drift.
  The UI now renders `Query blocked by security guardrail`, while the runner still hardcodes `SECURITY ALERT`.
  That assertion should be updated, ideally with a temporary dual-anchor window for backwards compatibility.

- `BUG-008` deserves top priority.
  A valid text upload returning `500` in a live run is release-relevant even though isolated controller/service tests cover many happy and mapped-failure paths.

## Adjust

- `BUG-001` fix should be narrower and more current-stack-aware than the draft suggests.
  Since this repo is on Spring Boot `3.4.13` and Spring Web MVC `7.0.5`, the handler set should cover both `NoHandlerFoundException` and `org.springframework.web.servlet.resource.NoResourceFoundException`, plus `MissingServletRequestParameterException`, `HttpRequestMethodNotSupportedException`, `HttpMessageNotReadableException`, and `HttpMediaTypeNotSupportedException`.
  I would not recommend `spring.web.resources.add-mappings=false` here; that is risky because the app serves static UI assets from `src/main/resources/static`.

- `BUG-001` also needs real MVC integration coverage, not only unit coverage on the advice class.
  There is already a `GlobalExceptionHandlerTest`; extend the coverage with MockMvc-style integration tests that hit an unmapped API path and missing required params, because those failures occur in the MVC dispatch layer, not by directly calling the advice methods.

- `BUG-002` does not need an `AskController` workaround.
  Missing required request params prevent method entry, so `SseEmitter.completeWithError(...)` inside `askStream()` will never run for the reported cases.
  The correct fix point is the MVC exception mapping, not controller-local defensive code.

- `BUG-003` reads as stale.
  `run-ui-tests.js` already has the stronger `selectSector()` implementation: it waits for options, verifies the option exists, and uses a 30-second timeout.
  I would mark this closed in the recommendations unless it is still reproducible in another runner.

- `BUG-004` should not default low-confidence Layer 3 outcomes to `ALLOW` without security review.
  The current `PromptGuardrailService` is intentionally fail-closed on ambiguous LLM outcomes, timeouts, and circuit-open states.
  It already carries a `confidenceScore` in `GuardrailResult`, so the safer next step is to improve telemetry and audit visibility of Layer 3 decisions, then tune the schema/prompt or domain allowlist with the security owner.

- `BUG-005` looks expected, not buggy.
  `SessionController.getSessionRemaining()` explicitly returns `{expired:true, remainingSeconds:0}` when `request.getSession(false)` is null, and that behavior is already asserted in `SessionControllerTest`.
  In DEV mode, auto-authenticated requests can still have no HTTP session, so I would reclassify this as a DEV-mode caveat rather than a defect.

- `BUG-008` should be investigated before changing exception taxonomy.
  `IngestController` already maps `SecureIngestionException`, `IllegalArgumentException`, `SecurityException`, and quota failures to non-500 responses.
  `SecureIngestionServiceTest` also proves ordinary `text/plain` ingest succeeds in isolation.
  That combination suggests the live 500 is likely an environment/runtime-path failure, not evidence that we should immediately add `DuplicateKeyException` or `DOCUMENT_TOO_SHORT` branches.
  First step should be: reproduce once with server logs, capture the stack trace, then add the smallest regression test around that exact failure.

## Additional Notes

- Claude's `OBS-012` should stay low priority.
  `run-ui-tests.js` extracts `responseText` from the DOM, not from the JSON `answer` key on `/api/ask/enhanced`, so it is unlikely to be the main cause of the Playwright failures described here.

- If the Playwright fixes are made, keep the runners aligned.
  `run-ui-tests.js`, `run-ui-govcloud.js`, and any other graph-focused runner should share the same deep-analysis visibility and injection-block text logic so the suite does not drift again.

## Recommended Action Order

1. Fix MVC exception mapping with integration tests for unmapped routes and missing params.
2. Fix `ensureDeepAnalysis()` and the injection-block assertion across all relevant Playwright runners.
3. Reproduce the live upload 500 with server logs, then patch the exact failure path and add a regression test.
4. Treat the guardrail false-positive work as a security-policy tuning task, not a quick product bugfix.
5. Reclassify `/api/sessions/remaining` in DEV mode from bug to expected caveat unless a non-DEV repro appears.
