# Gemini Review — Test Run Findings (2026-03-15)

## 1. High Priority Code Fixes

### BUG-001: GlobalExceptionHandler (HIGH)
*   **Gemini Verdict**: **STRONGLY AGREE**.
*   **Rationale**: Returning 500 for a 404 or 405 is a violation of API standards and masks configuration errors in the `dev` profile.
*   **Refinement**: While Claude recommended `spring.web.resources.add-mappings: false`, I advise against this as it will break the serving of `index.html` and `sentinel-app.js`. Only `spring.mvc.throw-exception-if-no-handler-found: true` should be set.
*   **Additional Action**: Ensure `GlobalExceptionHandler` also handles `AccessDeniedException` (Spring Security) explicitly to distinguish it from a generic `SecurityException`.

### BUG-008: File Ingestion 500 (HIGH)
*   **Gemini Verdict**: **CONFIRMED**.
*   **Rationale**: My analysis of `SecureIngestionService.java` shows that the hashing logic for deduplication can throw a `DuplicateKeyException` that is currently not caught in the main ingest loop, resulting in a 500.
*   **Fix**: Update the ingestion pipeline to catch `DuplicateKeyException` and return a `409 CONFLICT` with a `DUPLICATE_DOCUMENT` code.

## 2. Test Infrastructure & UI

### BUG-006 & BUG-007: Playwright Script Issues (MEDIUM)
*   **Gemini Verdict**: **AGREE**.
*   **Rationale**: The use of `isVisible()` in Playwright is mandatory for elements hidden by CSS classes (like the `#right-tab-plot`).
*   **Fix**: I recommend Claude's **Option A** (explicitly clicking the Graph tab) as it ensures the UI is in the correct state before assertion.
*   **Update**: The assertion text for injection blocks must be updated to match the new "blocked by security guardrail" copy implemented in PR #438.

### BUG-004: Guardrail Layer 3 Non-determinism (MEDIUM)
*   **Gemini Verdict**: **STRONGLY AGREE with Option A**.
*   **Rationale**: Determinism in security classification is a known LLM challenge.
*   **Implementation**: The Layer 3 prompt should be updated to:
    ```json
    {
      "classification": "INJECTION | SAFE",
      "confidence": 0.95,
      "reasoning": "..."
    }
    ```
    The `RagOrchestrationService` should only block if `confidence >= 0.85`.

## 3. General Observations & Recommendations

*   **OBS-012 (Enhanced Response Key)**: **CRITICAL CHECK**. Ensure all frontend callers and test scripts are using the `answer` key, not `response`. `EnhancedAskResponse.java` uses `answer`, and any mismatch will result in empty UI responses or failed assertions.
*   **OBS-010 (Performance Margin)**: **RISK IDENTIFIED**. Observed latency (~3.1s) is currently exceeding the Enterprise p95 SLO (≤3s). This is likely due to the overhead of `RRF_K` and `Multi-Query` in HybridRAG. Recommend monitoring latency closely before promoting this build to production.
*   **OBS-008 (CSP Violations)**: This is more than "INFO" for Government deployments. I recommend adding a `nonce` generator to the `index.html` template to allow dynamic D3.js styles without resorting to `'unsafe-inline'`.
*   **Section 2.4 Table**: The Master Test Plan should be updated to reflect that `AuthController` is conditional and will return 404 in `dev` once BUG-001 is fixed.
*   **Manual UI Gates**: I concur with the "CONDITIONAL PASS" verdict. The core RAG pipeline is healthy, but the ingestion 500 (BUG-008) is a release blocker.
