# Mercenary Login Testing Cheat-Sheet

Last validated against code: `2026-03-15`

This repo does not store real passwords in source control. For STANDARD auth, set the admin password via environment variables; the app stores only a hash in MongoDB.

## Credential Hygiene and Release Guardrails (Mandatory)
- Never paste raw secrets into test logs, handoff entries, PR comments, screenshots, or committed files.
- Record credential provenance in run artifacts using variable names only (for example `SENTINEL_ADMIN_PASSWORD`, `ADMIN_PASS`), never values.
- `Test123!` fallback is non-production only and must never be used for release sign-off evidence.
- For govcloud (`AUTH_MODE=CAC`), password fallback is out of scope; auth evidence must come from CAC/trusted-proxy path validation.

## DEV (no login)
- `APP_PROFILE=dev` is the repo-native profile selector — used by harness scripts (`run-ui-suite.ps1`, `run_e2e_profiles.ps1`) and maps to `spring.profiles.active` via `application.yaml:5`.
- `SPRING_PROFILES_ACTIVE=dev` also works when launching Spring directly (e.g. `./gradlew bootRun`).
- Either sets `app.auth-mode=DEV`, bypassing the login screen. Do not mix both in the same process.
- UI/API bypass auth. DEV headed Playwright runs do **not** provide meaningful login or RBAC evidence — login/RBAC gates apply to STANDARD/OIDC/CAC runs only.

## STANDARD (username/password)
- Set:
  - `SENTINEL_BOOTSTRAP_ENABLED=true`
  - `SENTINEL_ADMIN_PASSWORD=<your_password>` (also accepted as `SENTINEL_BOOTSTRAP_ADMIN_PASSWORD` — both bind via Spring relaxed binding in `application.yaml:1133-1135`)
- If the DB already has users and you need to force reset:
  - `SENTINEL_BOOTSTRAP_RESET_ADMIN=true`
- Login is `admin / <SENTINEL_ADMIN_PASSWORD>`.

## Automated UAT Credential Set (Current Harness Defaults)
- Admin login:
  - Username: `admin`
  - Password source precedence:
    - `ADMIN_PASS` (runner env)
    - `SENTINEL_ADMIN_PASSWORD`
    - `SENTINEL_BOOTSTRAP_ADMIN_PASSWORD`
    - fallback (non-production only): `Test123!`
- Viewer RBAC spot-check login (requires `test-users` Spring profile to auto-seed account via `TestDataInitializer`):
  - Username: `viewer_unclass`
  - Password: `TestPass123!`
  - Override via `VIEWER_USER` / `VIEWER_PASS`
  - **Note:** this account is only present when `test-users` profile is active; do not assume it exists in non-test environments.
- Harness scripts export both `APP_PROFILE` and `AUTH_MODE` (e.g. `run-ui-suite.ps1:116-118`). Both variables are set for the Playwright runner process; `APP_PROFILE` is the primary profile selector.
- Enterprise UI UAT commonly runs `APP_PROFILE=enterprise` with `AUTH_MODE=STANDARD` for deterministic local validation.
- For non-OIDC enterprise test runs, set local placeholders if missing:
  - `OIDC_ISSUER=http://127.0.0.1/oidc-placeholder`
  - `OIDC_CLIENT_ID=sentinel-ui-suite`

## ENTERPRISE / OIDC
- Set `AUTH_MODE=OIDC` or `APP_PROFILE=enterprise`.
- No local password; use IdP users.
- For local testing, enterprise may be run with `AUTH_MODE=STANDARD`.
- Admin-only endpoints require an IdP user mapped to `ADMIN`.

### Enterprise Browser SSO Flow (Manual Testing)
- The SPA calls `GET /api/auth/mode` (`AuthModeController`) to discover the active auth mode.
- If OIDC, the frontend redirects to `GET /api/auth/oidc/authorize` (`OidcBrowserFlowController`), which initiates the IdP redirect.
- After IdP login, the callback arrives at `GET /api/auth/oidc/callback` (`OidcBrowserFlowController`), which exchanges the code and establishes the session.
- Manual test evidence must include: IdP redirect initiated, callback received, post-login admin action performed, and artifact captured.

## GOVCLOUD / CAC
- Set `AUTH_MODE=CAC` or `APP_PROFILE=govcloud`.
- No password; auth via CAC cert subject DN.
- Local testing may simulate trusted-proxy flow via `X-Client-Cert` header when CAC auto-provision is enabled for the test lane.

### Banner Acknowledgment (Govcloud — Enabled by Default)
- Govcloud profile enables the legal/policy banner by default (`application-govcloud.yaml:48-51`).
- When enabled, authenticated API requests are blocked by `BannerAcknowledgmentFilter` until the user calls `POST /api/auth/acknowledge-banner`.
- `GET /api/auth/banner-status` returns current acknowledgment state.
- Manual govcloud test evidence **must** include: banner displayed, acknowledgment POST called, subsequent API call succeeds.
- If banner gate is not exercised, the manual auth gate is incomplete and must be marked `FAIL`.

## Workspace / Tenant Auth Coverage (Required)
- Workspace context is resolved in `WorkspaceFilter` via (in order): `X-Workspace-Id` header → `workspaceId` query parameter → session cookie.
- Streaming requests (`/api/ask/stream`) pass workspace via `workspaceId` query param (wired in `sentinel-app.js`).
- Regulated editions (medical, government): missing or unauthorized workspace returns hard `403` (`WorkspacePolicy`).
- Non-regulated editions: workspace defaults gracefully when header/param is absent.
- Required test cases:
  - Non-regulated: omit `X-Workspace-Id` — request succeeds with default workspace.
  - Regulated: supply unauthorized `workspaceId` — expect `403 Access Denied`.
  - Admin-only: `GET/POST /api/workspaces/**` requires `ADMIN` role; viewer role must receive `403`.

## Manual UI Authentication Gate (Mandatory)
- STANDARD runs must include one headed-browser login and one auth-required API action with artifact capture.
- OIDC runs must include browser IdP login flow validation (authorize → callback → post-login admin action) and evidence.
- CAC runs must include trusted-proxy/CAC identity assertion evidence and protected endpoint access verification.
- Govcloud runs must include banner acknowledgment flow evidence (see section above).
- DEV runs do **not** satisfy this gate — login evidence must come from STANDARD, OIDC, or CAC runs.
- If any required manual UI auth gate is `FAIL`, `BLOCKED`, `SKIPPED`, or `NOT RUN`, release sign-off is `FAIL`.

## Required Auth/Session Negative-Path Contract Checks

The table below applies to REST endpoints only. SSE and workspace error contracts differ — see subsections below.

All REST error payloads use standardized `ApiErrorResponse` record: `{error: string, code: string, timestamp: ISO-8601}`.

| Endpoint | Scenario | Expected Status | Expected Payload Contract |
|---|---|---|---|
| `/api/auth/login` | Missing username/password | `400` | `ApiErrorResponse` — `code="MISSING_CREDENTIALS"`, `error="Missing credentials"` |
| `/api/auth/login` | Invalid credentials | `401` | `ApiErrorResponse` — `code="INVALID_CREDENTIALS"`, `error="Invalid credentials"` |
| `/api/sessions/{sessionId}/export` | Unauthenticated | `401` | `ApiErrorResponse` — `code="AUTH_REQUIRED"` |
| `/api/sessions/{sessionId}/export` | HIPAA export disabled | `403` | `ApiErrorResponse` — `code="SESSION_EXPORT_DISABLED"` |
| `/api/sessions/{sessionId}/export` | Session not found | `404` | `ApiErrorResponse` — `code="SESSION_NOT_FOUND"` |
| `/api/sessions/{sessionId}/export` | Export IO failure | `500` | `ApiErrorResponse` — `code="EXPORT_FAILED"` |
| `/api/sessions/{sessionId}/export/file` | Session not found | `404` | `ApiErrorResponse` — `code="SESSION_NOT_FOUND"` |
| `/api/sessions/{sessionId}/export/file` | Success | `200` | `SessionActionResponse` — `{status: "exported", sessionId}` only |
| `/api/auth/unlock-account` | Any | per impl | `ApiErrorResponse` shape |
| `/api/auth/change-password` | Any | per impl | `ApiErrorResponse` shape |
| `/api/sessions/remaining` | Unauthenticated | `401` | `ApiErrorResponse` — `code="AUTH_REQUIRED"` |

Contract drift rule:
- Any status/payload drift requires documented approval and test-plan update before release sign-off.
- Error `code` values are machine-readable and stable — frontend/test code should assert on `code`, not `error` text.

### SSE / Streaming Error Contract (`/api/ask/stream`)
`/api/ask/stream` is authenticated (`@PreAuthorize("isAuthenticated()")`). Errors on this path do **not** use `ApiErrorResponse`. Instead:
- Auth failures (unauthenticated): HTTP `401` before SSE stream opens (Spring Security intercepts pre-upgrade).
- Workspace denial: raw JSON `{"error": "..."}` from `WorkspaceFilter` before stream opens.
- Banner acknowledgment required: simple map response `{"error": "Banner acknowledgment required"}` from `AuthModeController`.
- Pipeline errors after stream is open: SSE `event: error` with `{"error": "<message>"}` data field (see `AskController.sendSseError()`).
- Test code must handle SSE event parsing separately from REST `ApiErrorResponse` assertions.

## Connector Admin Testing
- Endpoints: `/api/admin/connectors/status`, `/api/admin/connectors/catalog`, `/api/admin/connectors/sync`
- Required role: `ADMIN`
- STANDARD profile: local `admin / <SENTINEL_ADMIN_PASSWORD>`
- ENTERPRISE profile: OIDC IdP account mapped to `ADMIN`

## Post-Login Formatting Gate (Mandatory)
- After successful login in any profile, run one representative summary query.
- Verify response contains no formatting artifacts:
  - no mojibake/replacement characters (`\uFFFD`)
  - no binary-signature noise in prose (`PK...`, `Rar!...`, class magic-byte gibberish)
  - no mirrored duplicate clauses (`X - X`)
- If present, mark run `FAIL` and open a defect before sign-off.

## One-Time Legacy Connector Migration (Startup)
- Migration flags are not credentials, but typically run by an admin/operator deployment:
  - `SENTINEL_CONNECTORS_LEGACY_MIGRATION_ENABLED=true`
  - `SENTINEL_CONNECTORS_LEGACY_MIGRATION_DRY_RUN=true` (recommended first pass)
  - `SENTINEL_CONNECTORS_LEGACY_MIGRATION_FORCE=false` (default safety)
- After reviewing dry-run logs, rerun with `...DRY_RUN=false` to apply and mark complete.

## Active Harness Defaults (Current)
- `tools/playwright-runner/run-ui-suite.ps1`:
  - Exports `APP_PROFILE` (repo-native selector) and `AUTH_MODE` for the runner process.
  - Uses `SENTINEL_ADMIN_PASSWORD` when set.
  - Falls back to `Test123!` only when password input is missing (non-production behavior).
  - Sets `ADMIN_PASS` for the Playwright runner process.
- `tools/run_e2e_profiles.ps1`:
  - Requires `SENTINEL_ADMIN_PASSWORD` for STANDARD auth profiles and throws if missing.

## Credential Provenance Record (Required in Formal Runs)
- Auth mode:
- Username source:
- Password source selected (variable name only):
- Fallback credential used (`Test123!`): YES/NO
- Environment classification (`local-dev`, `uat`, `release-candidate`):
- Evidence artifact path (login trace / API capture / UI screenshot):

## Post-2026-03-01 Auth Validation Notes
- OIDC defaults hardening (PR #310): enterprise profile defaults to `auto-provision=false` and `require-approval=true` (`application.yaml:1051-1052`). New OIDC users are not auto-provisioned; admin approval is required. Does not change credential flow but affects first-login behavior.
- Controller auth gate extraction (PR #315): `HyperGraphController.validateSectorAccess()` and `SessionController.withSessionOwnership()` are internal refactors — endpoint signatures, error codes, and credential flows are unchanged.
- `@Value` → `@ConfigurationProperties` migration (PRs #317-#319): environment variable names (`SENTINEL_ADMIN_PASSWORD`, `SENTINEL_BOOTSTRAP_ENABLED`, etc.) continue to bind via Spring relaxed binding; no credential config changes.
- Banner acknowledgment (post-2026-03-01): `POST /api/auth/acknowledge-banner` and `GET /api/auth/banner-status` added to `AuthModeController`. Govcloud enables banner by default. See Banner Acknowledgment section above.
- Account management endpoints added: `/api/auth/unlock-account`, `/api/auth/change-password` — require `ADMIN` role for unlock, authenticated user for password change.
- Session remaining endpoint added: `/api/sessions/remaining` — returns session quota info, requires authentication.

## Where It's Configured
- `src/main/resources/application.yaml`
- `src/main/resources/application-govcloud.yaml`
- `src/main/java/com/jreinhal/mercenary/config/DataInitializer.java`
