# SENTINEL / Mercenary — Vibecode Remediation Plan

**Author:** Claude Opus 4.6 (incorporating inputs from Gemini 3 and Codex 5.3)
**Date:** 2026-02-28
**Purpose:** Systematically eliminate patterns of AI-generated code that lack consolidation, producing a codebase that reads like it was designed rather than accumulated.

---

## What "Vibecoded" Means Here

This isn't about whether AI wrote the code — it's about whether the code was **consolidated after generation**. The architecture is real (edition isolation, custom vector store, multi-tenant workspace model). But the implementation layer shows signs of rapid feature addition without going back to extract shared patterns, decompose god classes, or enforce consistency.

The result: a codebase where each piece makes sense in isolation, but the whole feels like accretion rather than design.

---

## Current State (Evidence Summary)

| Indicator | Measurement | Healthy Range | Verdict |
|-----------|-------------|---------------|---------|
| Largest class | 3,766 lines (RagOrchestrationService) | <500 | Way over |
| Largest controller | 1,529 lines (MercenaryController) | <300 | Way over |
| Max constructor dependencies | 38 (MercenaryController) | 5-10 | Kitchen sink |
| Max imports in one file | 87 (MercenaryController) | 15-25 | Sprawl |
| RAG strategies implementing shared interface | 2 of 16 | 16 of 16 | No contract |
| Strategies with ContentSanitizer | 2 of 16 | 16 of 16 | Inconsistent |
| Identical `isEnabled()` implementations | 16 copies | 1 (in base class) | Pure duplication |
| `@Value` annotations | 313 across 69 files | Grouped into config classes | Scattered |
| Boolean feature flags | 67+ | Grouped, documented | Scattered |
| Duplicated auth guard logic | 3 copies (ask/askEnhanced/askStream) | 1 (extracted method) | Copy-paste |
| Source files without tests | 147 of 246 (60%) | <20% | Major gap |
| Inline prompt templates | 16 independent strings | Centralized registry | Sprawl |
| Static regex patterns in one class | 13 | Extracted to utility | Accumulation |
| Empty catch blocks | 3+ in RagOrchestrationService | 0 | Silent failures |

---

## Remediation Items

### 1. CRITICAL — Decompose the God Classes

**Problem:** `RagOrchestrationService` (3,766 lines, 37 dependencies, 12 distinct responsibilities) and `MercenaryController` (1,529 lines, 38 dependencies) are textbook god classes. They're the #1 reason the codebase feels vibecoded — every feature was wired into the same two files.

**Why it matters:** Beyond aesthetics, god classes make bugs invisible. The security audit found cross-tenant issues in RAG strategies partly because the orchestration layer is too large for any reviewer (human or AI) to hold in context. A 3,766-line class with 37 dependencies cannot be meaningfully reviewed.

**Extraction targets for RagOrchestrationService:**

| Extracted Class | Responsibility | Estimated Lines |
|-----------------|---------------|-----------------|
| `QueryAuthorizationService` | Permission checks, sector validation, quota enforcement | ~150 |
| `QueryIntentAnalyzer` | Query complexity routing, decomposition, intent classification | ~200 |
| `RagStrategyRouter` | Strategy selection, delegation, fallback logic | ~300 |
| `SnippetExtractor` | Document-to-snippet conversion, truncation, formatting | ~150 |
| `CitationRepairService` | Citation pattern matching, repair, formatting | ~200 |
| `DocumentNormalizer` | Score normalization, deduplication, ranking | ~150 |
| `ResponseSynthesizer` | Final answer assembly, metadata composition, audit logging | ~200 |

After extraction, `RagOrchestrationService` becomes a thin coordinator (~400 lines) that calls these services in sequence. Each extracted class is independently testable.

**Extraction targets for MercenaryController:**

| Extracted Class | Responsibility | Estimated Lines |
|-----------------|---------------|-----------------|
| `AskController` | `/api/ask`, `/api/ask/stream` endpoints only | ~200 |
| `IngestController` | `/api/ingest` endpoints | ~150 |
| `InspectController` | `/api/inspect`, `/api/documents` endpoints | ~100 |

Controllers should be thin — validate input, call service, return response. Any logic beyond that belongs in the service layer.

**The `ask()` / `askEnhanced()` / `askStream()` duplication:** These three methods share ~95% identical guard logic (auth checks, quota enforcement, PII redaction). Extract the common preamble into a private method like `prepareQuery()` that returns a validated `QueryContext` object, then each method only contains its unique retrieval/response logic.

**Approach:**
- One PR per extracted class (WIP=1)
- Extract, move tests, verify CI — no behavior changes
- Start with `QueryAuthorizationService` (smallest, most self-contained, eliminates the duplicated guard clauses across 3 methods)

---

### 2. CRITICAL — Create BaseRagStrategy Abstract Class

**Problem:** 16 RAG strategies exist as standalone classes with no shared contract. Only 2 implement the `RagStrategy` interface. Each independently implements:
- `isEnabled()` — 16 identical copies
- `@PostConstruct init()` — 16 copies with same structure, different log messages
- `buildFilterExpression()` — 8+ reimplementations
- `standardRetrieval()` — 5+ nearly identical fallback methods
- Timeout/exception handling — 3+ identical catch blocks
- WorkspaceContext capture — scattered, inconsistent

**Why it matters:** This is the root cause of half the security audit findings. When each strategy independently decides whether to apply tenant filtering, content sanitization, and classification ceilings, inconsistency is inevitable. The audit found 12 strategies missing classification ceiling and 14 missing content sanitization — because there's no mechanism forcing them to include it.

**Design:**

```java
public abstract class BaseRagStrategy {

    // --- Shared state (eliminates 16 copies of @Value + isEnabled()) ---
    private final boolean enabled;
    private final VectorStore vectorStore;
    private final ContentSanitizer contentSanitizer;

    // --- Template method: enforces filtering + sanitization ---
    public final List<Document> retrieve(String query, String department) {
        if (!enabled) return List.of();

        String workspaceId = WorkspaceContext.getCurrentWorkspaceId();
        FilterExpression filter = buildMandatoryFilter(department, workspaceId);
        List<Document> raw = doRetrieve(query, filter);
        return contentSanitizer.sanitizeBatch(raw);
    }

    // --- Subclass extension point ---
    protected abstract List<Document> doRetrieve(String query, FilterExpression filter);

    // --- Shared utilities (eliminates 8+ reimplementations) ---
    protected final FilterExpression buildMandatoryFilter(String dept, String workspace) {
        return FilterExpressionBuilder.create()
            .forDepartment(dept)
            .forWorkspace(workspace)
            .forClassificationCeiling(SecurityContextHolder.getContext()...)
            .build();
    }

    protected final List<Document> timedRetrieval(
            CompletableFuture<List<Document>> future, long timeoutSeconds) {
        // Shared timeout + exception handling (eliminates 3+ copies)
    }

    // --- Lifecycle (eliminates 16 identical @PostConstruct methods) ---
    @PostConstruct
    void logInit() {
        log.info("{} initialized (enabled={})", strategyName(), enabled);
    }

    public abstract String strategyName();
}
```

**Migration path:**
1. Create `BaseRagStrategy` with template method
2. Migrate one strategy (start with `HydeService` — simplest) to extend it
3. Verify CI, then migrate remaining 15 strategies (one PR per 3-4 strategies)
4. Add ArchUnit rule: all classes in `rag/` subpackages with `@Component` must extend `BaseRagStrategy`
5. Delete the now-impossible unfiltered query paths

**After migration, adding a new RAG strategy requires:**
- Extend `BaseRagStrategy`
- Implement `doRetrieve()` and `strategyName()`
- Tenant filtering, sanitization, enablement checks, and lifecycle logging are automatic

Compare to today: copy an existing strategy, hope you remembered all the security checks, discover 6 months later in an audit that you forgot `forClassificationCeiling()`.

---

### 3. HIGH — Consolidate Configuration Into Typed Config Classes

**Problem:** 393 properties scattered across `application.yaml`, consumed by 313 `@Value` annotations in 69 files. 67+ boolean feature flags with no grouping. Overlapping properties (3 different "timeout" properties, 3 different "max-doc-chars" properties). RagOrchestrationService alone has 17 `@Value` annotations.

**Why it matters:** Configuration sprawl makes it impossible to understand what the application actually does without reading 69 files. It also makes deployment error-prone — misconfigure one of 393 properties and behavior changes silently.

**Fix: Replace `@Value` with `@ConfigurationProperties` classes:**

```java
// Before: 17 @Value annotations scattered in RagOrchestrationService
@Value("${sentinel.rag.top-k:10}") private int topK;
@Value("${sentinel.rag.similarity-threshold:0.3}") private double similarityThreshold;
@Value("${sentinel.rag.max-doc-chars:8000}") private int maxDocChars;
// ... 14 more

// After: One injected config object
@ConfigurationProperties(prefix = "sentinel.rag")
public record RagConfig(
    int topK,
    double similarityThreshold,
    int maxDocChars,
    Map<String, StrategyConfig> strategies  // per-strategy config
) {
    public record StrategyConfig(
        boolean enabled,
        int topK,          // override per strategy
        double threshold   // override per strategy
    ) {}
}
```

**Groupings:**

| Config Class | Prefix | Absorbs |
|-------------|--------|---------|
| `RagConfig` | `sentinel.rag` | All RAG tuning knobs, strategy enable flags, retrieval budgets |
| `SecurityConfig` (props) | `sentinel.security` | Rate limits, guardrail thresholds, auth timeouts |
| `IngestionConfig` | `sentinel.ingestion` | Chunk sizes, PII settings, Tika limits, post-write hooks |
| `PerformanceConfig` | `sentinel.performance` | Thread pools, timeouts, queue capacities |
| `EnterpriseConfig` | `sentinel.enterprise` | Memory, admin dashboard, reporting flags |

**Benefits:**
- IDE autocomplete for all config properties
- Validation at startup (typos caught immediately instead of at query time)
- Config documentation generated from class structure
- Reduces `@Value` count from 313 to ~30 (one per config class injection point)

---

### 4. HIGH — Eliminate Duplicated Utility Code Across Strategies

**Problem:** 8+ private helper methods are reimplemented across RAG strategies with minor variations:

| Method | Copies | Variation |
|--------|--------|-----------|
| `isEnabled()` | 16 | None — identical |
| `@PostConstruct init()` | 16 | Different log message, same structure |
| `buildFilterExpression()` | 8+ | Same builder calls, sometimes with extras |
| `standardRetrieval()` | 5+ | Same try/catch, different log prefix |
| `extractKeywords()` | 3+ | Same logic, same stop words list |
| Timeout handling | 3+ | Same catch blocks, different variable names |
| WorkspaceContext capture | 5+ | Same `getCurrentWorkspaceId()` call |

**Fix:** Most of this is absorbed by `BaseRagStrategy` (Item #2). The remaining shared utilities go into:

```java
// RagQueryUtils.java — static utilities for all strategies
public final class RagQueryUtils {
    public static Set<String> extractKeywords(String query) { ... }
    public static Set<String> extractEntities(String query) { ... }
    // One implementation, used everywhere
}

// PromptTemplates.java — centralized prompt definitions
public final class PromptTemplates {
    public static final String RELEVANCE_GRADER = "You are a relevance grader...";
    public static final String SELF_REFLECTION = "You are a self-reflective AI...";
    public static final String QUERY_ROUTER = "You are an expert query router...";
    public static final String HYDE_GENERATOR = "You are an expert knowledge assistant...";
    // ... all 16 prompts in one place
}
```

**Why separate from BaseRagStrategy:** Not all strategies need these utilities. Static utility methods are better than base class methods for optional functionality. Base class = mandatory contracts. Utilities = optional helpers.

---

### 5. HIGH — Fix Silent Exception Handling

**Problem:** Multiple locations silently swallow exceptions:

```java
// RagOrchestrationService:3672 — silent score parsing failure
catch (NumberFormatException ignored) {}

// RagOrchestrationService:3184 — silent department parsing failure
catch (IllegalArgumentException ignored) {}

// RagOrchestrationService:2909 — another silent NumberFormatException
catch (NumberFormatException ignored) {}
```

Plus widespread catch-and-log-only patterns where the caller has no idea an error occurred:
```java
catch (Exception memEx) {
    log.warn("Session persistence failed (non-fatal): {}", memEx.getMessage());
}
```

**Why it matters:** Silent failures are debugging black holes. When something goes wrong in production, these catch blocks turn a 5-minute diagnosis into a 5-hour investigation because the error was swallowed at the source.

**Fix rules:**

| Pattern | When Acceptable | When Not |
|---------|----------------|----------|
| `catch (X ignored) {}` | Never | Always fix — at minimum `log.debug()` |
| `catch (X e) { log.warn(...) }` | Genuinely non-fatal optional enrichment | Core logic paths, data integrity operations |
| `catch (X e) { return default }` | Input parsing with defined fallback | When the default hides data quality bugs |

**Specific fixes:**
- Score parsing: return `OptionalDouble.empty()` instead of silently returning 0.0 — let the caller decide
- Department parsing: throw or return a clear error — an invalid department is a bug, not a recoverable condition
- Session persistence: if this is truly non-fatal, document WHY in a comment. If it's actually fatal, let it propagate

**Approach:** Grep for `ignored` and empty catch blocks, fix each one. Small PR, high impact on debuggability.

---

### 6. HIGH — Enforce Fail-Closed Defaults (Zero-Trust)

*From Gemini 3's "Pillar 2" — the codebase uses soft fallbacks in security paths to keep the UI working during development. These must become hard failures in production.*

**Problem:** Security-critical code paths use `orElse(null)`, `orElse(defaultValue)`, or silent fallbacks when context is missing. This is a "fail-open" pattern — if a workspace, user, or department can't be resolved, the system defaults to *permissive* behavior instead of *denying* access.

**Examples of fail-open patterns to find and fix:**
- `WorkspaceContext.getCurrentWorkspaceId()` returning null → query runs without workspace filter (cross-tenant leak)
- `User` object missing → code falls back to a default/guest user instead of throwing
- `Department.fromString()` failing → catches `IllegalArgumentException` and continues with a default
- `orElse(null)` on Optional security lookups → downstream code skips the check entirely

**Fix — Strict Context Requirement:**

```java
// Before (fail-open):
String workspaceId = WorkspaceContext.getCurrentWorkspaceId(); // might be null
// ... query proceeds without workspace filter if null

// After (fail-closed):
String workspaceId = WorkspaceContext.requireCurrentWorkspaceId(); // throws if null

// WorkspaceContext.java:
public static String requireCurrentWorkspaceId() {
    String id = getCurrentWorkspaceId();
    if (id == null) {
        throw new SecurityContextMissingException(
            "WorkspaceContext not set — cannot execute tenant-isolated query");
    }
    return id;
}
```

**Rules:**
- If `WorkspaceContext` or `User` is missing in a service method, **throw immediately** — never default
- If `Department` parsing fails, **throw** — an invalid department is a bug, not a recoverable condition
- All `orElse(null)` in security paths must be audited and replaced with `orElseThrow()`
- The dev profile may provide sensible defaults for convenience, but non-dev profiles must fail-closed

**Approach:** Grep for `orElse(null)`, `orElse("")`, `catch.*ignored`, and silent fallbacks in security-adjacent code. Fix each to throw. Small, high-value PR.

---

### 7. HIGH — Decompose Frontend JS Monoliths

*From Codex 5.3 — I completely missed the frontend as a vibecoding concern.*

**Problem:** `sentinel-app.js` and `admin.js` are large monolithic files carrying repeated rendering and state patterns. Large frontend files make security review harder (the XSS audit had to scan thousands of lines), and make safe refactoring nearly impossible without risking regressions.

**Fix — Split by bounded context:**

```
sentinel-app.js → modules:
  chat/         ← message rendering, SSE streaming, input handling
  sources/      ← document list, citation display, evidence panel
  graph/        ← D3/force-graph visualization, entity explorer
  reports/      ← reporting UI, export functionality
  settings/     ← user preferences, department selection

admin.js → modules:
  overview/     ← dashboard stats, system health
  users/        ← user management, role assignment
  workspaces/   ← workspace CRUD, quota management
  connectors/   ← S3/SharePoint/Confluence config
  reports/      ← admin reporting, audit log viewer
```

**Frontend rendering standards (from Codex 5.3):**
- Ban direct dynamic `innerHTML` unless value is fully escaped or static-only
- Require safe helper wrappers for all DOM manipulation:
  - `setHtmlSafeStatic(element, staticHtml)` — for known-safe static content
  - `setHtmlEscapedTemplate(element, template, values)` — escapes interpolated values
  - `setText(element, text)` — always safe, uses `textContent`
- Add grep-based CI gate: any new `innerHTML` assignment without a safe wrapper = build failure

**Testing (from Codex 5.3):**
- Add snapshot and payload-based XSS tests for every rendering module
- Add explicit tests for markdown/link sanitization paths

**Why high priority:** The frontend is where users interact with the system. A single unescaped `innerHTML` in the chat rendering path is an XSS vulnerability. Modularizing makes it possible to audit each module independently.

---

### 8. HIGH — Implement Exception Taxonomy

*From Codex 5.3 — goes beyond "fix silent catches" (Item #5) to establish a systematic exception strategy.*

**Problem:** High use of broad `catch (Exception)` across critical paths. Over-broad catches reduce signal quality and hide regressions. When everything is caught as `Exception`, you can't distinguish between a network timeout, a parsing error, and a security violation.

**Fix — Typed exceptions per layer:**

| Layer | Exception Types | Example |
|-------|----------------|---------|
| Controller | `ValidationException`, `AuthorizationException`, `RateLimitException` | User input fails validation → 400, not 500 |
| Service | `RetrievalException`, `LlmProviderException`, `TenantIsolationException` | Vector store timeout → graceful degradation |
| Connector | `ConnectorAuthException`, `ConnectorTransportException`, `SyncConflictException` | S3 creds expired → clear error message |
| RAG Strategy | `StrategyTimeoutException`, `StrategyDisabledException` | Individual strategy failure → fallback to next |

**Logging policy — every caught exception must include:**
- Operation name (what were we trying to do)
- Correlation/request ID (link to the HTTP request)
- Sanitized message (no raw user data via `LogSanitizer`)
- Stable error code (machine-readable, for monitoring/alerting)

**Unified error responses (from Gemini 3's Pillar 4):**
- Implement a canonical `SentinelErrorResponse` DTO
- Add `@ControllerAdvice` global exception handler
- Follow RFC 7807 (Problem Details for HTTP APIs) for structured error responses
- No more ad-hoc `"{\"error\":\"...\"}"` JSON strings

```java
// @ControllerAdvice global handler
@ExceptionHandler(TenantIsolationException.class)
ResponseEntity<ProblemDetail> handleTenantIsolation(TenantIsolationException ex) {
    ProblemDetail problem = ProblemDetail.forStatusAndDetail(
        HttpStatus.FORBIDDEN, "Access denied to requested resource");
    problem.setProperty("errorCode", "TENANT_ISOLATION_001");
    return ResponseEntity.status(HttpStatus.FORBIDDEN).body(problem);
}
```

**Sprint budget (from Codex 5.3):** 20-30 `catch (Exception)` replacements per sprint. Start with the hotspot files: `RagOrchestrationService`, `MercenaryController`, then ingestion + connector paths.

---

### 9. HIGH — Production Profile Validator & Startup Safety

*From Gemini 3's Pillar 5 and Codex 5.3's "Security-by-Default Hardening."*

**Problem:** Heavy reliance on `.env` files and placeholder values in `application.yaml` for critical secrets (OIDC client secrets, AES keys, HMAC secrets). If an operator deploys with the placeholder values, the system runs with known/guessable keys.

**Fix — `ProductionProfileValidator`:**

```java
@Component
@Profile("!dev & !test & !ci-e2e")
public class ProductionProfileValidator {

    private static final Set<String> PLACEHOLDER_VALUES = Set.of(
        "change_me", "demo-key", "CHANGE_ME", "your-secret-here",
        "placeholder", "default-secret", "TODO");

    @PostConstruct
    void validateProductionSecrets() {
        // Fail startup if any critical secret contains a placeholder
        validateNotPlaceholder("OIDC_CLIENT_SECRET", oidcClientSecret);
        validateNotPlaceholder("AES_KEY", aesKey);
        validateNotPlaceholder("HMAC_SECRET", hmacSecret);
        validateNotPlaceholder("MONGODB_URI", mongoUri);
    }

    private void validateNotPlaceholder(String name, String value) {
        if (value == null || PLACEHOLDER_VALUES.stream().anyMatch(
                p -> value.toLowerCase().contains(p))) {
            throw new IllegalStateException(
                "FATAL: " + name + " contains a placeholder value. "
                + "Set a real secret before starting in production mode.");
        }
    }
}
```

**Additional startup safety (from Codex 5.3):**
- Require explicit profile selection in non-dev deployments (no "default profile = dev")
- Add startup log summary showing safe/unsafe config status:
  ```
  ============ SENTINEL SECURITY STATUS ============
  Profile:     enterprise
  Auth mode:   OIDC        [OK]
  CORS origin: https://app.example.com  [OK]
  CSRF:        enabled     [OK]
  Secrets:     all validated [OK]
  TLS:         enforced    [OK]
  ================================================
  ```
- Dangerous config combinations fail startup:
  - CORS wildcard + non-dev profile → fail
  - DEV auth mode + non-dev profile → fail
  - Air-gap mode + external LLM provider configured → fail

---

### 10. MEDIUM — Centralize Pattern/Regex Definitions

**Problem:** `RagOrchestrationService` defines 13 `static final Pattern` objects inline (lines 125-147). These are citation patterns, hint patterns, metric patterns — all used for response post-processing. They belong in a dedicated class, not scattered in a 3,766-line file.

**Fix:**

```java
public final class CitationPatterns {
    public static final Pattern STRICT_CITATION = Pattern.compile("...");
    public static final Pattern CITATION_FILENAME = Pattern.compile("...");
    public static final Pattern INLINE_CITATION = Pattern.compile("...");
    // etc.
}

public final class QueryHintPatterns {
    public static final Pattern METRIC_HINT = Pattern.compile("...");
    public static final Pattern NAME_CONTEXT = Pattern.compile("...");
    public static final Pattern SUMMARY_HINT = Pattern.compile("...");
    // etc.
}
```

**Also fixes:** Duplicated constants like `DOC_SEPARATOR`, `MAX_ACTIVE_FILES`, and `MAX_TIKA_CHARS` that are defined in multiple files. Move to a `Constants` class or into the relevant `@ConfigurationProperties` class.

---

### 11. MEDIUM — Standardize RAG Strategy Naming

**Problem:** Inconsistent naming across 16 strategies:

| Current Name | Package | Suffix | Issue |
|-------------|---------|--------|-------|
| `HybridRagService` | `hybridrag/` | `RagService` | Good |
| `AdaptiveRagService` | `adaptiverag/` | `RagService` | Good |
| `HiFiRagService` | `hifirag/` | `RagService` | Good |
| `AgenticRagOrchestrator` | `agentic/` | `Orchestrator` | Different suffix |
| `HyperGraphMemory` | `hgmem/` | `Memory` | Different suffix |
| `CragGraderService` | `crag/` | `GraderService` | No main `CragService` |
| `RewriteService` | `crag/` | `Service` | Missing `Crag` prefix |
| `OntologyQueryService` | `hgmem/ontology/` | `QueryService` | Nested, different naming |

**Fix:** After `BaseRagStrategy` migration, all strategies should follow `{Name}RagService` naming:
- `AgenticRagOrchestrator` → `AgenticRagService` (if it extends BaseRagStrategy)
- `HyperGraphMemory` → keep as-is (it's not a retrieval strategy, it's an indexing service)
- `CragGraderService` → keep as-is, but create `CragRagService` as the entry point that extends BaseRagStrategy and delegates to `CragGraderService` + `RewriteService`

Naming consistency makes it possible to write ArchUnit rules like "all classes matching `*RagService` must extend `BaseRagStrategy`."

---

### 12. MEDIUM — Clean Up Over-Documentation

**Problem:** AI-generated code tends to over-explain. Signs in this codebase:
- `L-XX` comment references (audit ticket numbers in inline comments)
- Verbose Javadoc on internal methods that are only called from one place
- Comments that restate what the code does rather than why
- Academic paper citations in class-level Javadoc (useful for README, noise in code)

**Rules for comment cleanup:**

| Keep | Remove |
|------|--------|
| Comments explaining WHY (business rules, non-obvious constraints) | Comments restating WHAT the code does |
| `@param`/`@return` on public API methods | Javadoc on private methods |
| Paper citations in a `README` or `docs/` file | Paper citations in class Javadoc |
| Security-critical explanations (why a check exists) | `L-XX` ticket references (belong in git history, not code) |
| Edge case documentation ("this is null when...") | "This method returns the result" style comments |

**Example cleanup:**
```java
// Before (vibecoded):
/**
 * RAPTOR-style hierarchical retrieval service.
 *
 * Builds a document abstraction tree at ingestion time by recursively
 * summarizing groups of chunks into higher-level summaries. At query
 * time, retrieves from multiple levels and reconciles evidence with
 * anti-duplication logic.
 *
 * Based on: "RAPTOR: Recursive Abstractive Processing for Tree-Organized
 * Retrieval" (Sarthi et al., 2024).
 *
 * Lifecycle: OFF by default. Activate via sentinel.raptor.enabled=true.
 */
public class RaptorService { ... }

// After (human-written):
/** Hierarchical retrieval via recursive chunk summarization (RAPTOR). */
public class RaptorService extends BaseRagStrategy { ... }
```

The paper reference moves to `docs/engineering/RAG_FEATURES.md`. The lifecycle note is unnecessary because `BaseRagStrategy.isEnabled()` makes it self-documenting.

**Approach:** Don't do a comment-cleanup-only PR. Clean up comments as part of each refactoring PR — when you touch a file for Items #1-7, also clean its comments.

---

### 13. MEDIUM — Align Optional Injection with Null Checks

**Problem:** Inconsistent use of `@Autowired(required = false)`:
- Some services are declared `required = false` but never null-checked
- Some services are constructor-injected (never null) but defensively null-checked
- `RagOrchestrationService` has 218 null-check operations, many on final fields that can't be null

**Fix rules:**

| Injection Style | Null Check | Action |
|----------------|------------|--------|
| Constructor-injected (final field) | Has null check | Remove the null check — it can't be null |
| `@Autowired(required = false)` | No null check | Add null check, or make required |
| `@Autowired(required = false)` | Has null check | Correct — leave as-is |

**After BaseRagStrategy migration:** Most optional RAG strategy injections in `RagOrchestrationService` become unnecessary. The `RagStrategyRouter` (extracted in Item #1) will hold a `Map<String, BaseRagStrategy>` populated by Spring autowiring all `BaseRagStrategy` beans. No more 15 individual `@Autowired(required = false)` strategy fields.

---

### 14. MEDIUM — Close the Test Gap

**Problem:** 147 of 246 source files (60%) have no corresponding test. Untested areas include:
- 15 Spring configuration classes (initialization paths unvalidated)
- Multiple controllers (security annotations like `@PreAuthorize` not tested)
- Several services (`CaseService`, `ConnectorService`, `PageRenderService`)

**Why it matters beyond coverage numbers:** Untested code is where vibecoded patterns hide. If a method has never been tested, no one has verified that its behavior is intentional rather than accidental.

**Priority testing targets:**

| Priority | Category | Why |
|----------|----------|-----|
| P0 | Config classes (`SecurityConfig`, `SectorConfig`) | Misconfiguration = security bypass |
| P0 | Controller auth annotations | `@PreAuthorize` typo = open endpoint |
| P1 | RAG strategy tenant isolation | Cross-tenant leak = critical vulnerability |
| P1 | PII redaction edge cases | Missed PII = compliance violation |
| P2 | Remaining services | Functional correctness |
| P3 | Utility classes | Low risk |

**Approach:** Don't write tests for the sake of coverage. Write tests that verify security properties and behavioral contracts. Use the `BaseRagStrategy` migration (Item #2) to make strategies testable — right now, testing a strategy requires mocking 7+ dependencies because there's no abstraction layer.

**Lane separation:** Codex writes tests. Claude provides the list of files + what properties each test should verify.

---

### 15. LOW — Reduce Import Sprawl

**Problem:** MercenaryController has 87 imports. RagOrchestrationService has 70. These are symptoms, not root causes — fixing Items #1-4 will organically reduce imports.

**After fixes:**
- Decomposing MercenaryController into 3 focused controllers → each has ~15-20 imports
- RagOrchestrationService extracting 7 services → coordinator has ~20 imports
- BaseRagStrategy absorbing strategy dependencies → individual strategies drop from 30+ to ~10

**No separate PR needed.** This resolves itself as a side effect of decomposition.

---

### 16. LOW — Extract Inline Prompt Templates

**Problem:** 16 RAG strategies define their own `SYSTEM_PROMPT` strings inline. No central registry.

**Fix:**

```java
public final class RagPrompts {
    // Retrieval prompts
    public static final String RELEVANCE_GRADER = """
        You are a relevance grader. Given a document and query, ...
        """;
    public static final String QUERY_ROUTER = """
        You are an expert query router. ...
        """;
    // Generation prompts
    public static final String SELF_REFLECTION = """
        You are a self-reflective AI. ...
        """;
    // ... all prompts centralized
}
```

**Benefits:**
- Easy to audit all prompts for injection patterns in one file
- Easy to update prompt style consistently
- Enables future prompt versioning or A/B testing

**Lower priority because:** Prompts don't cause bugs or security issues. This is purely maintainability.

---

## Execution Order

Items are ordered by impact and dependency. Later items become easier (or automatic) after earlier ones are done.

```
Phase A: Structural (eliminate god classes, create abstractions)
  Item #2: BaseRagStrategy              ← Enables #4, #11, #13, #14, #15
  Item #1: Decompose god classes        ← Enables #10, #13, #15
  Item #6: Fail-closed defaults         ← Independent, pairs well with #2

Phase B: Safety & Hardening
  Item #9: Production profile validator ← Independent, high value, small PR
  Item #8: Exception taxonomy           ← Builds on #1 (ControllerAdvice)
  Item #5: Fix silent exceptions        ← Subset of #8, can merge into it
  Item #7: Frontend decomposition       ← Independent, parallel workstream

Phase C: Consolidation (DRY, config, patterns)
  Item #4: Eliminate duplicated code    ← Mostly done by Item #2
  Item #3: Typed config classes         ← Independent, high value
  Item #10: Centralize patterns         ← Falls out of Item #1

Phase D: Polish (naming, docs, tests)
  Item #11: Standardize naming          ← After Item #2 migration
  Item #12: Clean up comments           ← During all other PRs
  Item #13: Align null checks           ← After Items #1 and #2
  Item #14: Close test gap              ← After Items #1 and #2 (testable now)
  Item #15: Import sprawl               ← Automatic after Items #1 and #2
  Item #16: Prompt templates            ← Lowest priority, whenever convenient
```

**Estimated PR count:** ~20-25 PRs across all phases (WIP=1).

**Phase A is the highest leverage.** Items #1, #2, and #6 together eliminate the root causes of ~10 other items. If you only do three things from this list, do those three.

---

## 30/60/90 Day Plan

*Adapted from Codex 5.3's timeline structure.*

### Days 0-30 (Foundation)
- **Item #2:** Create `BaseRagStrategy`, migrate first 4 strategies
- **Item #9:** Ship `ProductionProfileValidator` + startup config summary
- **Item #6:** Audit and fix top 10 fail-open defaults in security paths
- **Item #5:** Fix all empty catch blocks (small, can ship early)
- Enable blocking CI gates: unsafe `innerHTML` grep, `@PreAuthorize` coverage check

### Days 31-60 (Structural)
- **Item #2:** Complete remaining 12 strategy migrations to `BaseRagStrategy`
- **Item #1:** Extract `QueryAuthorizationService` and `RagStrategyRouter` from god class
- **Item #7:** First frontend split — `chat/` and `sources/` modules from `sentinel-app.js`
- **Item #8:** Exception taxonomy for controller + service layers, `@ControllerAdvice` handler
- **Item #3:** First `@ConfigurationProperties` class (RagConfig)

### Days 61-90 (Consolidation)
- **Item #1:** Complete remaining god class extractions (CitationRepair, SnippetExtractor, etc.)
- **Item #7:** Complete frontend modularization (admin.js split)
- **Item #3:** Remaining config classes, `@Value` count < 50
- **Item #14:** Test gap closure — config classes + controller auth annotations (P0 targets)
- Establish monthly security control effectiveness review
- Add ArchUnit rules preventing re-accumulation

---

## How to Prevent Re-Accumulation

After remediation, add these ArchUnit rules to prevent vibecoding patterns from returning:

```java
// No class over 500 lines
@ArchTest
static final ArchRule noGodClasses = classes()
    .should(haveAtMostNLines(500))
    .because("Classes over 500 lines should be decomposed");

// No constructor with more than 10 parameters
@ArchTest
static final ArchRule noKitchenSinkConstructors = constructors()
    .should(haveAtMostNParameters(10))
    .because("Inject a facade or config object instead of 10+ dependencies");

// All RAG strategies must extend BaseRagStrategy
@ArchTest
static final ArchRule ragStrategyContract = classes()
    .that().resideInAPackage("..rag..")
    .and().haveSimpleNameEndingWith("RagService")
    .should().beAssignableTo(BaseRagStrategy.class)
    .because("BaseRagStrategy enforces tenant filtering and content sanitization");

// No @Value annotations outside @ConfigurationProperties classes
@ArchTest
static final ArchRule noScatteredValues = noFields()
    .that().areAnnotatedWith(Value.class)
    .should().beDeclaredInClassesThat()
    .areNotAnnotatedWith(ConfigurationProperties.class)
    .because("Use @ConfigurationProperties instead of scattered @Value");

// Layered architecture: controllers cannot access repositories directly (from Gemini)
@ArchTest
static final ArchRule layeredArchitecture = noClasses()
    .that().resideInAPackage("..controller..")
    .should().accessClassesThat().resideInAPackage("..repository..")
    .because("Controllers must go through the service layer");

// All @RestController methods must have @PreAuthorize (from Gemini)
@ArchTest
static final ArchRule allEndpointsAuthorized = methods()
    .that().areDeclaredInClassesThat().areAnnotatedWith(RestController.class)
    .and().areAnnotatedWith(anyOf(GetMapping.class, PostMapping.class,
                                  PutMapping.class, DeleteMapping.class))
    .should().beAnnotatedWith(PreAuthorize.class)
    .because("Every endpoint must have explicit authorization");
```

### CI Quality Gates (from Gemini)

Beyond ArchUnit, enforce measurable quality thresholds:
- **SonarCloud maintainability:** Must stay at 'A' rating
- **Code duplication:** No more than 3% across security-sensitive packages
- **Audit coverage:** Every new controller endpoint must be paired with an `AuditEvent` log entry

These rules make the vibecoding patterns a CI failure, not a code review opinion.

---

## Success Criteria

The codebase no longer feels vibecoded when:

| Metric | Current | Target |
|--------|---------|--------|
| Largest class | 3,766 lines | <500 lines |
| Max constructor dependencies | 38 | <10 |
| RAG strategies extending BaseRagStrategy | 2/16 | 16/16 |
| Duplicated isEnabled() implementations | 16 | 1 (in base class) |
| `@Value` annotations | 313 | <30 |
| Source files without tests | 60% | <20% |
| Empty catch blocks | 3+ | 0 |
| Broad `catch (Exception)` in critical paths | High | Down 60%+ |
| Unsafe dynamic `innerHTML` occurrences | Present | 0 |
| Frontend monolith files | 2 (sentinel-app.js, admin.js) | 10+ focused modules |
| Fail-open security defaults | Present | 0 — all fail-closed |
| Placeholder secrets accepted at startup | Yes | No (validator blocks) |
| Ad-hoc JSON error responses | Scattered | Unified RFC 7807 |
| Security finding recurrence rate | ~4 categories repeated | 0 repeat categories |
| Findings with enforced CI gate | ~10% | 100% |
| SonarCloud maintainability | Unknown | 'A' rating |
| Code duplication (security packages) | Unknown | <3% |
| A new RAG strategy requires security boilerplate | Yes (copy-paste 50+ lines) | No (extend BaseRagStrategy, implement 1 method) |
| Can explain the system by reading 5 files | No (need 30+) | Yes |

---

## Non-Negotiable Rules

*Consolidated from all three auditors.*

1. No closure of any item without tests + enforcement gates
2. No silent fallback for auth/authorization failures — fail-closed always
3. No new broad `catch (Exception)` blocks in critical paths without explicit waiver
4. No unbounded frontend monolith growth without modular review
5. No direct `innerHTML` with unsanitized content
6. No placeholder secrets accepted outside dev profile
7. No new `@Value` annotations — use `@ConfigurationProperties`
8. No new RAG strategy without extending `BaseRagStrategy`
9. No class exceeding 500 lines without architectural justification

---

## Sources

This document synthesizes approaches from three independent analyses:

- **Claude Opus 4.6** — God class decomposition, BaseRagStrategy design, config consolidation, duplication analysis, ArchUnit enforcement, test gap analysis
- **Gemini 3** — Fail-closed defaults (Zero-Trust), unified error DTOs (RFC 7807), ProductionProfileValidator, layered architecture ArchUnit rules, quality metric gates (SonarCloud 'A', 3% duplication), "vibe-to-enterprise transformation" framing
- **Codex 5.3** — Frontend decomposition by bounded context, rendering safety standards, exception taxonomy per layer, logging policy, sprint budgeting, 30/60/90 day plan, startup config summary, phase drift verification, non-negotiable rules
