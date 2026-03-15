# SENTINEL / Mercenary — Security Audit Process & Methodology

**Author:** Claude Opus 4.6 (incorporating inputs from Gemini 3 and Codex 5.3)
**Date:** 2026-02-28
**Repository:** `jreinhal/mercenary`
**Purpose:** Define a repeatable, converging audit process that eliminates bug classes rather than chasing individual symptoms.

---

## Why This Document Exists

Two consecutive audit cycles revealed a structural problem:

- **Audit 1** (2026-02-27): Found 55 findings → 48-item remediation campaign (21 phases, PRs #273-#293)
- **Audit 2** (2026-02-28): Found 35 findings — many in the **same categories** as Audit 1 (missing sector isolation, missing classification ceilings, inconsistent sanitization)

The audit → fix → audit loop was treating symptoms. The same root causes kept producing new findings in different files. This process adds **root-cause analysis**, **architectural fixes**, and **automated enforcement** to make the cycle converge.

---

## Process Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  Step 0: SCOPE & BASELINE FREEZE                                     │
│    ↓                                                                 │
│  Step 1: AUDIT (Deep Scan)                                           │
│    ↓                                                                 │
│  Step 2: TRIAGE & ROOT-CAUSE ANALYSIS                                │
│    ↓                                                                 │
│  Step 3: RED-TEAM REPRODUCTION ("Pin" the Exploit)                   │
│    ↓                                                                 │
│  Step 4: DESIGN REVIEW (Pre-Implementation Gate)                     │
│    ↓                                                                 │
│  Step 5: ARCHITECTURAL FIX (Eliminate the Bug Class)                 │
│    ↓                                                                 │
│  Step 6: MECHANICAL FIX (Apply Pattern to All Instances)             │
│    ↓                                                                 │
│  Step 7: AUTOMATED ENFORCEMENT (ArchUnit / CI / Runtime)             │
│    ↓                                                                 │
│  Step 8: REGRESSION TESTING (Security-Specific Tests)                │
│    ↓                                                                 │
│  Step 9: RE-AUDIT (Verify & Discover New Classes)                    │
│    ↓                                                                 │
│  Step 10: DOCUMENTATION & SIGN-OFF                                   │
│                                                                      │
│  Exit condition: Re-audit finds only NEW categories, not repeats.    │
│  If repeats appear → the automated enforcement step failed.          │
│  Fix the enforcement, don't just fix the instance.                   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step 0: SCOPE & BASELINE FREEZE

*Adapted from Codex 5.3's workflow — establishes a stable target before scanning.*

**Goal:** Define what's being audited and freeze the target so findings don't shift mid-audit.

**Actions:**

1. **Record baseline commit SHA and branch.** All findings reference this exact snapshot.
2. **Define scope boundaries:**
   - Which code paths (all editions? specific packages?)
   - Which profiles/deployment targets (dev, standard, enterprise, govcloud?)
   - Which threat model assumptions (internal users only? public internet? air-gapped?)
3. **Freeze high-risk changes.** No merges to the audited branch during the active audit window. PRs can queue but don't merge until the audit pass completes.
4. **Identify prior audit artifacts.** Pull the previous cycle's root-cause catalog, enforcement rules, and residual risk register so the new audit can check whether prior fixes held.

**Output:** Audit scope document (can be a simple header block in the findings doc):
```
Baseline: master @ e82d49c (2026-02-28)
Scope: Full stack — backend, frontend, infra, CI/CD, all RAG strategies
Profiles: dev, standard, enterprise, govcloud
Threat model: Multi-tenant SaaS with clearance-level isolation
Prior cycle: Audit 1 (2026-02-27), 55 findings, 48 remediated
```

**Key principle:** An audit against a moving target produces findings that may already be fixed or that can't be reproduced. Pin the target first.

---

## Step 1: AUDIT (Deep Scan)

**Goal:** Discover as many raw findings as possible across all security domains.

**Method:** Launch parallel audit agents, each with a focused scope:

| Agent | Scope |
|-------|-------|
| Endpoints & Authorization | Every `@*Mapping`, `@PreAuthorize`, IDOR, sector/workspace access checks |
| Data Flow & Injection | Input → storage → output tracing. PII, prompt injection, log injection, XSS |
| Frontend Security | innerHTML/XSS, CSP, SRI, CORS, cookies, clickjacking, nonce usage |
| RAG Strategy Security | Tenant isolation in vector queries, content sanitization, resource exhaustion |
| Config, Deps, Infra, CI/CD | Secrets, Docker, workflows, crypto, edition isolation, dependency CVEs |

**Output:** Raw findings document with severity ratings (CRITICAL / HIGH / MEDIUM / LOW / INFO).

**Key principle:** Audit what's MISSING, not just what EXISTS. Enumerate all endpoints/strategies/filters and check each one — don't assume consistency.

**Confirmed vs. Suspicious:** Separate findings into two tiers (from Codex 5.3):
- **Confirmed vulnerabilities** — reproducible exploit path, direct evidence (file:line, repro steps)
- **Suspicious patterns** — looks wrong but exploit path is unclear or blocked by other controls

Both get tracked, but only confirmed findings require immediate remediation. Suspicious patterns get investigated during triage (Step 2).

### Audit Checklist

For every audit pass, verify these specific areas:

**Endpoint Authorization:**
- [ ] Every `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` has `@PreAuthorize` or explicit auth check
- [ ] Every endpoint accepting `dept`/`sector` parameter validates user access to that sector
- [ ] Every endpoint accepting `userId` or resource ID validates ownership or admin role
- [ ] Every database/vector store query includes sector filtering

**Data Flow:**
- [ ] Trace data from input to storage — redacted/sanitized before persisting?
- [ ] Trace data from storage to output — sector filtering at query time, not just in-memory?
- [ ] Cache keys compound (include sector + workspace) to prevent cross-tenant leakage?
- [ ] User-controlled strings sanitized before logging (LogSanitizer)?

**RAG Strategies (per strategy):**
- [ ] `similaritySearch()` called with non-null `FilterExpression`?
- [ ] FilterExpression includes sector/department filter?
- [ ] FilterExpression includes workspace filter?
- [ ] `forClassificationCeiling()` applied?
- [ ] Retrieved content passed through `ContentSanitizer` before intermediate LLM calls?
- [ ] Resource bounds on loops/iterations (no unbounded retries)?

**Frontend:**
- [ ] No `innerHTML` with unsanitized content — use `textContent` or DOMPurify
- [ ] CSP nonce attributes present on all inline scripts
- [ ] SRI hashes on all vendor/CDN scripts
- [ ] CORS policy restricts origins in non-dev profiles

**Filters & Interceptors:**
- [ ] All `@Order` values unique — no conflicts
- [ ] Auth-dependent filters run AFTER authentication filter
- [ ] `X-Forwarded-For` only trusted behind known proxies

---

## Step 2: TRIAGE & ROOT-CAUSE ANALYSIS

**Goal:** Group findings by root cause, not by file or severity. Reduce N findings into M root causes where M << N.

**Method:**

1. **List every finding** from Step 1.
2. **Tag each finding with a root-cause category.** Common categories:
   - `MISSING_FILTER` — Vector store query without sector/workspace/classification filter
   - `MISSING_SANITIZATION` — User-controlled data reaching log/LLM/output without sanitization
   - `MISSING_AUTHZ` — Endpoint or operation without authorization check
   - `MISSING_VALIDATION` — Input accepted without bounds/type/ownership validation
   - `INCONSISTENT_PATTERN` — A security pattern exists but isn't applied everywhere
   - `ARCHITECTURAL_GAP` — No mechanism exists to enforce the constraint
   - `CONFIGURATION` — Insecure default, missing hardening, secret exposure
   - `DEPENDENCY` — Third-party CVE or outdated library
3. **Score exploitability and blast radius** (from Codex 5.3). For each finding:
   - **Exploitability:** How easy is it to trigger? (Requires auth? Requires admin? Requires specific profile?)
   - **Blast radius:** What's the worst outcome? (Single user? Cross-tenant? Full data breach?)
   - **Ease of abuse:** Can it be automated? Does it require insider knowledge?
4. **Count instances per category.** If a category has 3+ instances, it's a systemic issue that needs an architectural fix, not individual patches.
5. **Prioritize categories**, not individual findings. A category with 12 instances of MEDIUM severity is more important than 1 isolated HIGH.
6. **Explicit deferral with risk acceptance** (from Codex 5.3). If a finding is intentionally deferred, document:
   - Why it's being deferred (low exploitability, blocked by other controls, etc.)
   - Who accepted the risk and when
   - Review date — when the deferral will be re-evaluated
   - Never defer silently. No finding should disappear without a written decision.

**Output:** Root-cause analysis table:

| Root Cause Category | Instance Count | Severity (Worst) | Exploitability | Blast Radius | Architectural Fix Needed? |
|---------------------|---------------|-------------------|----------------|--------------|--------------------------|
| MISSING_FILTER | 14 | CRITICAL | Low (requires auth) | Cross-tenant data leak | Yes — shared query builder |
| MISSING_SANITIZATION | 8 | HIGH | Medium | Prompt injection via LLM | Yes — mandatory sanitization hook |
| MISSING_AUTHZ | 4 | MEDIUM | Low (requires admin) | Cross-sector privilege escalation | No — mechanical fix |
| ... | ... | ... | ... | ... | ... |

**Key principle:** If you're about to create 12 similar PRs fixing the same pattern in different files, stop. That's a signal you need Step 5 first.

---

## Step 3: RED-TEAM REPRODUCTION ("Pin" the Exploit)

*Adapted from Gemini 3's "P5" lifecycle — Phase 2 "Pin." Prove the exploit exists before writing the fix.*

**Goal:** Write a failing test that demonstrates each root-cause category is actually exploitable. This prevents fixing theoretical issues that aren't real, and creates an unambiguous success criterion for the fix.

**Method:**

For each root-cause category, write a "Red Team" test that **fails** because the vulnerability exists:

```java
// Example: QuCoRagService cross-tenant leakage
@Test
void exploitCrossTenantLeakage_qucorag() {
    // Setup: Ingest a document into Department A's workspace
    ingestDocument("secret-doc.pdf", Department.ENGINEERING);

    // Exploit: Query as a user in Department B
    authenticateAs(userInDepartment(Department.SALES));
    List<Document> results = qucoRagService.query("secret contents");

    // This SHOULD return empty, but currently returns the secret doc
    // Test FAILS — proving the exploit is real
    assertThat(results).isEmpty();
}
```

**Rules:**
- The test must fail specifically because of the identified vulnerability — not for any other reason.
- The test must be CI-ready (no external dependencies, runs in `ci-e2e` profile).
- One test per root-cause category minimum. High/Critical categories deserve multiple exploit variants.

**Lane separation:** In this project, Codex writes the red-team tests. Claude provides the root-cause categories and theoretical exploit paths. Codex writes the failing test, then Claude implements the fix that makes it pass.

**Output:** Failing test suite that serves as both proof-of-exploit and acceptance criteria. After Step 6, every one of these tests must pass.

**Key principle:** If you can't write a test that exploits the vulnerability, either (a) it's not actually exploitable and can be downgraded, or (b) you don't understand the vulnerability well enough to fix it yet. Either way, this step forces clarity.

---

## Step 4: DESIGN REVIEW (Pre-Implementation Gate)

*Adapted from Gemini 3's "P5" Phase 3 and Codex 5.3's "Fix Design Review."*

**Goal:** Review the proposed fix approach before anyone writes production code. Catch architectural drift and unintended consequences early.

**Checklist — the fix must satisfy all of these before implementation begins:**

- [ ] **No fail-open defaults.** All security logic must default to DENY if context is missing or ambiguous. (Gemini golden rule #1)
- [ ] **No edition leakage.** Features restricted to GOVERNMENT or MEDICAL are excluded via Gradle source-sets, not runtime flags. (Gemini golden rule #2)
- [ ] **No unsanitized logging.** No raw exception messages or user-controlled strings in log statements without LogSanitizer. (Gemini golden rule #3)
- [ ] **No direct data access bypass.** All data access must pass through WorkspaceContext or SectorConfig routing layers. (Gemini golden rule #4)
- [ ] **Backward compatibility.** Does the fix change any public API contract? Any config property meaning?
- [ ] **Rollout constraints.** Can the fix be deployed incrementally, or is it all-or-nothing?
- [ ] **Blast radius contained.** Does the fix touch only the affected code, or does it ripple into unrelated modules?
- [ ] **Tests defined before code.** The red-team tests from Step 3 exist and fail. The fix must make them pass — no other success criteria.

**Who reviews:** In the three-agent model:
- **Gemini** (Architect) reviews for architectural drift and edition isolation
- **Claude** (Engineer) reviews for implementation feasibility and side effects
- **Codex** (SDET) reviews for testability and confirms red-team tests cover the exploit path

**Output:** "Design Approved" signal in the handoff log. If rejected, the rejection includes specific feedback on what to change.

**Key principle:** 30 minutes of design review prevents 3 hours of rework. A fix that introduces architectural drift is worse than the original vulnerability because drift compounds silently.

---

## Step 5: ARCHITECTURAL FIX (Eliminate the Bug Class)

**Goal:** Make the root cause impossible or trivially detectable by changing the architecture.

**Method:** For each root cause category with 3+ instances, design a structural fix:

### Example Architectural Fixes

**Root Cause: `MISSING_FILTER` (vector store queries without tenant isolation)**

Bad approach — patch each RAG strategy individually:
```java
// In QuCoRagService.java — add filter
// In GraphO1Service.java — add filter
// In AdaptiveRagService.java — add filter
// ... 12 more files
```

Good approach — create a safe abstraction that makes the unsafe path impossible:
```java
// New: TenantAwareVectorQuery.java
public class TenantAwareVectorQuery {
    public static SearchRequest forUser(String query, int topK, UserContext user) {
        FilterExpression filter = FilterExpressionBuilder.create()
            .forDepartment(user.getDepartment())
            .forWorkspace(user.getWorkspaceId())
            .forClassificationCeiling(user.getClearanceLevel())
            .build();
        return SearchRequest.query(query).withTopK(topK).withFilterExpression(filter);
    }
    // No method exists to create an unfiltered query
}
```

Then update all RAG strategies to use `TenantAwareVectorQuery.forUser()` instead of raw `SearchRequest`. The unfiltered path simply doesn't exist anymore.

**Root Cause: `MISSING_SANITIZATION` (content reaching LLM without sanitization)**

Good approach — make the RAG pipeline enforce sanitization:
```java
// In the shared retrieval path, sanitize BEFORE returning results
// so individual strategies can't forget
List<Document> results = vectorStore.similaritySearch(request);
return contentSanitizer.sanitizeBatch(results); // always applied
```

**Root Cause: `INCONSISTENT_PATTERN` (some strategies do X, others don't)**

Good approach — extract shared behavior into a base class or template method:
```java
public abstract class BaseRagStrategy {
    // Template method — subclasses implement retrieveInternal(),
    // base class handles filtering + sanitization
    public final List<Document> retrieve(String query, UserContext user) {
        SearchRequest request = TenantAwareVectorQuery.forUser(query, getTopK(), user);
        List<Document> raw = retrieveInternal(request);
        return contentSanitizer.sanitizeBatch(raw);
    }

    protected abstract List<Document> retrieveInternal(SearchRequest request);
}
```

**Output:** PRs implementing the architectural changes. These PRs should be small, focused, and testable in isolation. They don't fix all instances yet — they just create the safe path.

**Key principle:** A good architectural fix makes the wrong thing harder to do than the right thing. Ideally, the wrong thing becomes impossible without deliberate circumvention.

---

## Step 6: MECHANICAL FIX (Apply Pattern to All Instances)

**Goal:** Migrate all existing code to use the new safe patterns from Step 3.

**Method:**

1. **Enumerate every instance** of the old pattern (grep/search).
2. **Migrate each instance** to the new pattern. This should now be mechanical — no design decisions, just substitution.
3. **One PR per root-cause category** if the changes are cohesive, or one PR per subsystem if the diff is large.

**Output:** PRs migrating old patterns to new safe patterns.

**Key principle:** This step should be boring. If it requires creative thinking, the architectural fix from Step 3 wasn't complete enough — go back and improve it.

---

## Step 7: AUTOMATED ENFORCEMENT (ArchUnit / CI / Runtime)

**Goal:** Prevent the root cause from ever recurring. Turn audit findings into automated checks at three levels.

**This is the most important step.** Without it, the next developer (or AI agent) who adds a new RAG strategy will make the same mistake, and you'll be back to audit → fix → audit.

**Method:** For each root-cause category, write enforcement at the appropriate level:

### Level 1: ArchUnit Rules (compile-time / test-time)

```java
// Rule: Every class calling vectorStore.similaritySearch() must use TenantAwareVectorQuery
@ArchTest
static final ArchRule vectorStoreQueriesMustBeFiltered = noClasses()
    .that().resideOutsideOfPackage("..vector..")
    .should().callMethod(VectorStore.class, "similaritySearch", SearchRequest.class)
    .because("Use TenantAwareVectorQuery.forUser() to ensure tenant isolation");

// Rule: Every RAG strategy must extend BaseRagStrategy
@ArchTest
static final ArchRule ragStrategiesMustExtendBase = classes()
    .that().resideInAPackage("..rag..")
    .and().haveSimpleNameEndingWith("Service")
    .and().areAnnotatedWith(Component.class)
    .should().beAssignableTo(BaseRagStrategy.class)
    .because("BaseRagStrategy enforces tenant filtering and content sanitization");
```

### Level 2: CI Pipeline Checks (grep-based)

```yaml
# In .github/workflows/ci.yml
- name: Security invariant checks
  run: |
    # No raw similaritySearch without filter
    if grep -rn "similaritySearch(SearchRequest\." src/main/java --include="*.java" \
       | grep -v "TenantAwareVectorQuery" | grep -v "//.*similaritySearch"; then
      echo "FAIL: Raw similaritySearch detected. Use TenantAwareVectorQuery."
      exit 1
    fi

    # No innerHTML in JS files
    if grep -rn "innerHTML" src/main/resources/static --include="*.js" \
       | grep -v "DOMPurify" | grep -v "//.*innerHTML"; then
      echo "FAIL: innerHTML without DOMPurify detected."
      exit 1
    fi
```

### Level 3: Runtime Enforcement (Startup Validation & Monitoring)

*From Codex 5.3 — enforcement beyond test-time.*

**Startup guards** — Validate dangerous config combinations at boot, fail fast:
```java
@PostConstruct
void validateSecurityConfig() {
    if (!"dev".equals(activeProfile) && corsAllowedOrigins.contains("*")) {
        throw new IllegalStateException(
            "CORS wildcard origin is forbidden outside dev profile");
    }
    if (authMode == AuthMode.DEV && !"dev".equals(activeProfile)) {
        throw new IllegalStateException(
            "DEV auth mode cannot be used with profile: " + activeProfile);
    }
}
```

**Runtime monitoring** — Detect abuse signals that static analysis can't catch:
- Log and alert on cross-sector query attempts (queries where the response filter removes >50% of raw results)
- Monitor rate of `RejectedExecutionException` from thread pool exhaustion
- Track authentication failures by source IP for brute-force detection
- Alert on config changes at runtime (property refresh events)

### Pre-Commit Hooks (optional, fast feedback)

For rules that can be checked locally, add pre-commit hooks so developers get feedback before pushing.

**Output:**
- ArchUnit test class(es) with rules for each root-cause category
- CI pipeline additions for checks that can't be expressed as ArchUnit rules
- Startup validation for dangerous config combinations
- Monitoring/alerting rules for runtime abuse signals
- Documentation of what each rule prevents and why

**Key principle:** The rule should fail with a message that tells the developer what to do instead. Not just "this is wrong" but "use X instead of Y because Z."

---

## Step 8: REGRESSION TESTING (Security-Specific Tests)

**Goal:** Write tests that verify the security properties directly, not just functional correctness.

**Method:**

For each root-cause category, write at least one test that:
1. **Positive case:** Verifies the security property holds (e.g., query includes sector filter)
2. **Negative case:** Verifies that a violation is caught (e.g., ArchUnit rule fires on bad code)
3. **Boundary case:** Tests edge cases (e.g., user with multiple sector access, workspace switch mid-session)

### Security Test Categories

| Category | Test Type | Example |
|----------|-----------|---------|
| Tenant isolation | Integration | Query as User A → verify no User B documents returned |
| Classification ceiling | Integration | SECRET user queries → verify no TOP_SECRET docs |
| Content sanitization | Unit | Inject prompt injection marker → verify stripped |
| Auth enforcement | MockMvc | Hit endpoint without auth → verify 401/403 |
| IDOR | MockMvc | Access resource with wrong user ID → verify 403 |
| Rate limiting | Integration | Exceed rate limit → verify 429 |
| XSS prevention | Frontend/E2E | Inject `<script>` → verify escaped in output |

**Lane separation note:** In this project, Claude writes production code and Codex writes tests. Codex should receive the root-cause categories and write targeted security regression tests for each.

**Output:** Security regression test suite that runs as part of CI.

**Additional test category** (from Codex 5.3):

| Category | Test Type | Example |
|----------|-----------|---------|
| Config-profile safety | Unit/Integration | Startup with CORS wildcard + production profile → verify boot fails |
| Startup guards | Unit | Dangerous config combo → verify `IllegalStateException` on boot |

**Runtime validation** (from Codex 5.3): Beyond CI tests, validate security behavior in a realistic deployment context — with proxy, sidecar, Docker networking, actual profile activation. The `ciE2eTest` and `ciEnterpriseE2eTest` tasks partially cover this, but a full validation runs the Docker Compose stack and hits endpoints through the actual filter chain.

**Key principle:** Security tests should test the **property** (tenant isolation holds), not the **implementation** (FilterExpression is non-null). Properties survive refactors; implementation checks break.

---

## Step 9: RE-AUDIT (Verify & Discover New Classes)

**Goal:** Confirm that all instances of known root causes are fixed, and discover genuinely new vulnerability classes.

**Method:**

1. Run the same audit process as Step 1.
2. For each finding, check:
   - **Is this a repeat of a known root cause?** → The enforcement from Step 5 failed. Fix the enforcement rule, not just the instance.
   - **Is this a genuinely new category?** → Add it to the root-cause catalog and go through Steps 3-6 for this category.
3. Track metrics:

| Metric | Target |
|--------|--------|
| Repeat findings (same root cause) | 0 |
| New categories found | Decreasing each cycle |
| Total findings | Decreasing each cycle |
| Automated rules added | Increasing each cycle |

**Exit condition:** The re-audit finds zero repeat-category findings. Any new findings are in genuinely new categories not previously examined.

**Key principle:** If the same root cause produces findings in two consecutive audits, the problem is in Step 5 (enforcement), not Step 4 (fixing). Don't fix more instances — fix the rule that should have caught them.

---

## Step 10: DOCUMENTATION & SIGN-OFF

**Goal:** Record what was found, what was fixed, why, and what prevents recurrence.

**Deliverables:**

1. **Audit Findings Document** — Raw findings with severity, file locations, descriptions (the `Claude_4.6.md` style document)
2. **Root-Cause Analysis** — Categorized findings with instance counts and architectural fix decisions
3. **Remediation Evidence** — PRs, test results, ArchUnit rule outputs proving each root cause is addressed
4. **Enforcement Catalog** — List of all automated rules, what they prevent, and where they live:

| Rule | Type | Location | Prevents |
|------|------|----------|----------|
| `vectorStoreQueriesMustBeFiltered` | ArchUnit | `SecurityArchRulesTest.java` | Cross-tenant vector query leakage |
| `ragStrategiesMustExtendBase` | ArchUnit | `SecurityArchRulesTest.java` | Missing sanitization/filtering in new strategies |
| `no-raw-innerHTML` | CI grep | `.github/workflows/ci.yml` | XSS via innerHTML |
| ... | ... | ... | ... |

5. **Quality Ledger Entry** — Update `docs/evals/quality-ledger.md` with audit cycle results
6. **Residual Risk Register** (from Codex 5.3) — Dated list of deferred findings with risk acceptance justification and review dates. No finding disappears without a written decision.

### Required Artifacts Per Finding

*From Codex 5.3 — structured evidence template for every finding that reaches remediation.*

| Artifact | Description |
|----------|-------------|
| Finding ID & Severity | Unique identifier and CRITICAL/HIGH/MEDIUM/LOW/INFO |
| Impact Statement | What can an attacker do? Blast radius? |
| Evidence | File:line, repro steps, or red-team test name |
| Root-Cause Category | Which category from Step 2 triage |
| Fix Summary | What was changed and why |
| Tests Added | Red-team test (Step 3) + regression tests (Step 8) |
| Enforcement Rule | Which ArchUnit/CI/runtime rule prevents recurrence |
| Verification | Commands run and outcomes observed |
| Residual Risk | What risk remains after the fix, if any |

### Definition of Done (Per Finding)

*From Codex 5.3 — a finding is only "Done" when ALL of these are true:*

- [ ] Root cause fixed in code (not just the symptom)
- [ ] Red-team test from Step 3 now passes
- [ ] Regression tests added and passing in CI
- [ ] Automated enforcement rule exists for this failure mode
- [ ] Runtime behavior validated in intended deployment context (not just unit tests)
- [ ] Documentation updated if operational behavior changed
- [ ] Finding marked closed with evidence artifacts

A finding with a code fix but no enforcement rule is NOT done — it will recur.

---

## Remediation Cadence

*From Codex 5.3 — daily rhythm during active remediation.*

**Daily cycle (during active remediation):**
- **Morning:** Triage new findings + assign root-cause categories
- **Midday:** Implement fixes + write tests
- **End of day:** Verification + delta re-audit of fixed categories

**Weekly:**
- Full-system re-audit (Step 9)
- Control effectiveness review (are enforcement rules catching anything?)
- Residual risk register review

**Per root-cause category (WIP=1):**
- Complete Steps 3 → 8 for one category before starting the next
- Each category produces: architectural fix PR + mechanical fix PR + enforcement rule PR + test PR
- Don't batch all categories into one giant remediation campaign (that's what created the first audit's 48-item pile)

---

## Remediation Health Metrics

*Adapted from Gemini 3's KPIs and expanded.*

| Metric | Target | What It Measures |
|--------|--------|------------------|
| **Reproduction Rate** | 100% | % of findings "Pinned" (red-team test) before being fixed |
| **Drift Instances** | 0 | Times a fix required a secondary fix in a later phase |
| **Gate Failures** | 0 | PRs blocked by ArchUnit or security CI checks |
| **Repeat Findings** | 0 | Same root-cause category appearing in consecutive audits |
| **Enforcement Coverage** | 100% | % of root-cause categories with automated enforcement rules |
| **Deferral Ratio** | <10% | % of findings deferred (high ratio = risk accumulation) |
| **Time to Enforcement** | <1 cycle | How quickly a new finding class gets an automated rule |

---

## Applying This to the Current Audit (2026-02-28)

The 35 findings from the latest audit map to approximately 6 root-cause categories:

| # | Root Cause | Instances | Worst Severity | Architectural Fix |
|---|-----------|-----------|----------------|-------------------|
| 1 | **MISSING_FILTER** — Vector queries without sector/workspace/classification filter | ~14 (12 strategies + QuCoRAG + GraphO1) | CRITICAL | `TenantAwareVectorQuery` safe builder; all strategies must use it |
| 2 | **MISSING_SANITIZATION** — Content reaching intermediate LLM calls unsanitized | ~6 | HIGH | Sanitization hook in shared retrieval path |
| 3 | **MISSING_AUTHZ_SCOPE** — Admin endpoints not validating sector/workspace membership | ~4 | MEDIUM | Shared `@SectorAccess` annotation or interceptor |
| 4 | **CACHE_ISOLATION** — Cache keys not compound with tenant context | ~3 | MEDIUM | Tenant-prefixed cache key builder |
| 5 | **CONFIGURATION_HARDENING** — Insecure defaults, missing headers, CORS in dev | ~5 | LOW-MEDIUM | Config review + profile-specific overrides |
| 6 | **RESOURCE_BOUNDS** — Unbounded loops/retries in RAG strategies | ~3 | MEDIUM | Configurable bounds with safe defaults |

**Recommended execution order:**

1. Root cause #1 (MISSING_FILTER) — critical severity, highest instance count, most impactful architectural fix
2. Root cause #2 (MISSING_SANITIZATION) — high severity, compounds with #1
3. Root cause #3 (MISSING_AUTHZ_SCOPE) — medium but easy architectural fix
4. Root causes #4-6 — lower severity, can be batched

Each root cause should go through Steps 3 → 8 (Pin → Design Review → Arch Fix → Mechanical Fix → Enforcement → Tests) before moving to the next, maintaining WIP=1 discipline.

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It Fails | Do This Instead |
|--------------|-------------|-----------------|
| Fix each finding individually | Same bug class reappears in new files | Fix the root cause architecturally |
| Audit → fix all → audit (big batch) | Fixes take weeks, context is lost | Fix one root-cause category at a time, then re-check that category |
| Skip automated enforcement | Next feature reintroduces the bug | ArchUnit rule before moving to next category |
| Copy-paste the same filter code into 19 strategies | One gets missed, all diverge over time | Shared base class or mandatory utility |
| Mark low/info findings as "won't fix" without analysis | Some are symptoms of a deeper issue | Categorize first, then decide per-category |
| Run audit only when "ready" | Issues accumulate undetected | Enforcement rules run on every CI build |
| Treat test pass as closure without enforcement | Test proves it works today; nothing prevents regression tomorrow | Definition of Done requires enforcement rule |
| Fix one endpoint while equivalent paths remain unprotected | Partial fix creates false confidence | Root-cause analysis identifies all instances first |
| Ignore profile/config drift between local and production | "Works on my machine" with dev profile hides production bugs | Runtime validation in realistic deployment context |
| Defer medium findings indefinitely | They accumulate into critical exposure over time | Written risk acceptance with review dates |
| Log "in progress" without actionable evidence | Status theater — no one can verify actual progress | Required artifacts per finding |
| Fix without proving the exploit first | May fix a theoretical issue while missing the real vulnerability | Red-team reproduction (Step 3) before implementation |
| Allow fail-open defaults in security logic | Missing context = full access instead of no access | Default to DENY; require explicit grants |

---

## Metrics & Convergence

Track these across audit cycles to verify the process is converging:

| Cycle | Date | Total Findings | Repeat Categories | New Categories | Enforcement Rules Added |
|-------|------|---------------|-------------------|----------------|------------------------|
| 1 | 2026-02-27 | 55 | — | 8+ | 0 |
| 2 | 2026-02-28 | 35 | ~4 | ~2 | 2 (ArchUnit edition isolation) |
| 3 | TBD | Target: <15 | Target: 0 | TBD | Target: 6+ |
| 4 | TBD | Target: <5 | Target: 0 | TBD | Target: 8+ |

**Convergence means:** each cycle finds fewer findings, and zero repeat-category findings. If repeat categories appear, the enforcement step needs strengthening, not more manual fixes.

---

## Role Assignments (Three-Agent Model)

*From Gemini 3's P5 lifecycle — explicit ownership per step.*

| Step | Claude 4.6 (Engineer) | Codex 5.3 (SDET) | Gemini 3 (Architect) |
|------|----------------------|-------------------|---------------------|
| 0. Scope & Freeze | Define scope | — | Review threat model |
| 1. Audit | Run audit agents | — | Run parallel audit |
| 2. Triage | Categorize findings | Assess testability | Review categorization |
| 3. Red-Team Pin | Provide exploit paths | **Write failing tests** | Validate exploit theory |
| 4. Design Review | Propose fix approach | Confirm test coverage | **Approve/reject design** |
| 5. Architectural Fix | **Implement** | — | Review for drift |
| 6. Mechanical Fix | **Implement** | — | Spot-check |
| 7. Enforcement | Write ArchUnit/CI rules | Write enforcement tests | Review rule completeness |
| 8. Regression Tests | — | **Write security tests** | Review coverage |
| 9. Re-Audit | Run audit agents | Verify test pass | Run parallel audit |
| 10. Documentation | Write findings doc | Write test evidence | Review sign-off |

**Coordination:** Via handoff doc (`docs/evals/CLAUDE_CODEX_HANDOFF.md`) and PR review comments. No direct channel between agents.

---

## Sources

This document synthesizes approaches from three independent audit methodology documents:

- **Claude Opus 4.6** — Root-cause analysis, architectural fix patterns, enforcement-first convergence model
- **Gemini 3** — "P5" lifecycle (Identify → Pin → Plan → Patch → Gate), red-team reproduction, fail-open golden rules, drift metrics
- **Codex 5.3** — Scope freeze, confirmed vs. suspicious distinction, exploitability scoring, Definition of Done, required artifacts, runtime enforcement, cadence recommendations, risk acceptance documentation
