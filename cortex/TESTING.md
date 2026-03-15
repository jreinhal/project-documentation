# CORTEX Testing Checklist

## Zero-Gap E2E Test Protocol (Living)

### Application Context
- CORTEX is a local-first AI orchestration platform with a React 19/Tailwind CSS 4 frontend and a Node.js/Express backend.
- Core flows: spawn specialized agents, manage local reference repos (Knowledge/Skills/Tools/Agents), generate Flight Plans (Markdown).
- Default ports: `localhost:3001` (backend), `localhost:5173` (frontend/Vite dev).

### Critical Path & Functional Logic

1) Setup & Config
- Verify First-Run Wizard or Settings panel updates `config.json` and connects to the repository root.

2) Agent Factory Flow
- Enter a complex goal (example: "Audit auth module for security").
- Verify Generation Chain appears with step-by-step progress.
- Verify Generation Chain displays **real backend timings** after spawn completes (not fallback simulation values — see Generation Chain Timing section below).
- Verify Flight Plan generation completes and output renders.
- Verify Copy to Clipboard works.
- Verify Runs count increments in sidebar after successful spawn.
- Verify Recent Sessions panel updates with new session entry.

3) Generation Chain Timing Validation
- After a successful spawn, the Generation Chain must show real backend performance data.
- **How to verify**: Real timings have distinctive patterns — quick steps like "Create agent profile" and "Analyze goal keywords" show single-digit milliseconds (e.g., 4ms, 20ms), while "Search knowledge base" and "Generate flight plan" show large values (thousands to tens of thousands of ms). Fallback simulation values are evenly spaced (~300ms per step) with several showing 0ms.
- **Architecture**: Backend emits `___CORTEX_META___` sentinel in stdout → `runs.js` parses it → DataContext extracts `data.performance` → OrchestratorView useEffect overlays real durations onto steps.
- **Known failure mode**: If the API rate limiter is exhausted (too many concurrent polling requests), the `/spawn` request itself can fail silently. The Generation Chain will still show 5/5 complete but with fallback timings. Check browser console for "Too many requests" errors if suspected.

4) Decision Matrix Validation
- Confirm the generated Flight Plan includes: Retrieval gate, Query expansion, RAG-Fusion, Hybrid retrieval, RRF fusion.

5) Repository Management
- Perform a Smart Clone and a System Scan.
- Verify repositories are categorized (Agents, Skills, Knowledge, Tools).
- Verify folder sizes are accurate (spot-check against OS properties).

6) Evaluations & Runs
- Create a dataset in Evaluation Lab.
- Run evaluation against a recent spawn.
- Verify scorecard grading (including LLM rubric grading if enabled).

---

## UI Toggle & Button Verification (EVERY interactive element)

### ⚠️ WHY THIS SECTION EXISTS
A class of bug where the UI presents a toggle/button that appears functional but the backend silently ignores or overrides the user's choice. The UI must never lie about the effect of a user action. Every toggle, checkbox, and button must produce the advertised result — or clearly communicate why it cannot.

**Testing principle**: For every UI control, verify the **end-to-end data flow** — that the frontend state change reaches the backend, the backend processes it, and the result matches what the UI promised.

### Agent Factory View (`OrchestratorView.jsx`)

#### Generate Flight Plan button
- [ ] Click generates a spawn request to `/api/spawn`
- [ ] Loading state shows Generation Chain progress
- [ ] Success: Flight Plan renders in output area
- [ ] Failure: Generation Chain steps show error state (red), not misleading success
- [ ] Button is disabled during active spawn

#### Format selector (Universal / Markdown / JSON)
- [ ] Selected format is sent in POST body as `format` field
- [ ] Backend uses the format value (check Flight Plan output structure matches)
- [ ] Selector state resets correctly between spawns

#### "Search online for skills" checkbox
- [ ] When checked: POST body includes `externalSkills: { online: true }`
- [ ] When checked: `runs.js` sets `CORTEX_ONLINE_SKILLS=1` in spawned process env
- [ ] When checked: Backend orchestrator receives `onlineRequested === true`
- [ ] When checked: External skills search ACTUALLY RUNS (not skipped as "disabled")
- [ ] When unchecked: External skills step shows "Skipped (disabled)" — this is correct
- [ ] Generation Chain reflects actual skip/run state from backend response
- [ ] **Config gate bypass**: If `config.externalSkills.enabled` is false but user checks the box, the per-spawn toggle MUST override the global config
- [ ] **Remote gate**: If `config.externalSkills.allowRemote` is false and not in dev mode, shows "remote_not_allowed" skip reason — NOT silent "disabled"
- [ ] Disabled state: checkbox is disabled when `externalSkills.enabled=false` AND `allowRemote=false` AND not in dev mode — with visible explanation text

#### Training mode selector (Blocking / Background)
- [ ] Only visible when "Search online for skills" is checked
- [ ] Selected mode is sent in `externalSkills.trainingMode` field
- [ ] "Blocking" mode: vector index rebuild runs synchronously during spawn
- [ ] "Background" mode: vector index rebuild spawns detached child process
- [ ] Training step in Generation Chain reflects actual training mode used

#### "Run in background queue" checkbox
- [ ] When checked: POST body includes `async: true`
- [ ] When checked AND queue enabled in config: spawn is queued (returns job ID)
- [ ] When checked AND queue NOT enabled in config: behavior is clearly communicated
- [ ] Job appears in Jobs view when queued

#### Save Prompt button
- [ ] Opens title input field
- [ ] Saves prompt to `/api/prompts` with goal text and format
- [ ] Saved prompt appears in Library view → Quick Access
- [ ] Button shows success state briefly after save

#### Copy to Clipboard button
- [ ] Copies Flight Plan markdown to clipboard
- [ ] Shows success feedback (icon change or toast)
- [ ] Works when Flight Plan contains special characters

### Settings Panel (`SettingsPanel.jsx`)

#### Repository Root path input
- [ ] Changing path updates `config.json`
- [ ] Path is validated (directory exists)
- [ ] Repos list refreshes after path change

#### LLM Endpoint input
- [ ] URL is saved to config
- [ ] "Test Connection" button actually hits the endpoint
- [ ] Test Connection shows success/failure feedback
- [ ] Invalid URL shows clear error

#### Test Connection button
- [ ] Sends request to configured LLM endpoint
- [ ] Success: shows green confirmation
- [ ] Failure: shows red error with reason
- [ ] Loading state while request is in flight

#### Rebuild Vector Index button
- [ ] Triggers `/api/vector/rebuild` (or equivalent)
- [ ] Shows progress/loading state during rebuild
- [ ] Success: shows completion confirmation
- [ ] Failure: shows error message

#### Clear All Saved Prompts button
- [ ] Deletes all prompts from storage
- [ ] Library view / Quick Access clears
- [ ] Confirmation dialog appears before destructive action

#### Theme toggle (if present)
- [ ] Switches between light/dark
- [ ] Persists across page reloads
- [ ] All views render correctly in both themes

#### Polling interval input
- [ ] Changes config.pollingInterval
- [ ] Frontend polling frequency actually changes
- [ ] Minimum value enforced (not 0 or negative)

### Knowledge Base / Repos View (`KnowledgeView.jsx`)

#### Add Repository button/input
- [ ] Entering a valid git URL triggers clone
- [ ] Clone progress is shown
- [ ] Successfully cloned repo appears in repo list
- [ ] Invalid URL shows error in notice system + logs
- [ ] Duplicate URL shows notice + log entry

#### Smart Clone button
- [ ] Triggers git clone operation
- [ ] Progress indicator during clone
- [ ] Cloned repo is categorized (Agents/Skills/Knowledge/Tools)

#### System Scan button
- [ ] Scans repository root for all repos
- [ ] New repos appear in list
- [ ] Log entries are created for scan action
- [ ] Category sizes update

#### Delete/Remove repository button (if present)
- [ ] Removes repo from tracked list
- [ ] Does NOT delete files from disk (or confirms if it does)
- [ ] Repo disappears from list and categories

### Evaluations View (`EvaluationsView.jsx`)

#### Create Dataset button
- [ ] Opens dataset creation form
- [ ] Saves dataset to storage
- [ ] Dataset appears in dataset list

#### Run Evaluation button
- [ ] Triggers evaluation against selected run/dataset
- [ ] Progress shown during evaluation
- [ ] Results render in scorecard format
- [ ] LLM rubric grading works when LLM endpoint is configured

#### Create Retrieval Benchmark button (if present)
- [ ] Creates precision/recall/MRR benchmark
- [ ] Results render with metrics

### Runs View (`RunsView.jsx`)

#### Run detail expansion
- [ ] Clicking a run shows full details
- [ ] Performance data displays if available
- [ ] Comparison deltas shown between runs

### Jobs View (`JobsView.jsx`)

#### Cancel Job button
- [ ] Cancels queued/running job
- [ ] Job status updates to cancelled
- [ ] Status pill color changes

### Audit Trail View (`AuditView.jsx`)

#### Export button (CSV/JSON)
- [ ] Downloads file without errors
- [ ] File contains expected audit entries
- [ ] Filename includes timestamp or identifier

#### Filter/Search input
- [ ] Filters audit entries in real time
- [ ] Stays responsive with >50 entries
- [ ] Clear filter restores full list

### Library View (`LibraryView.jsx`)

#### Quick Access prompt cards
- [ ] Clicking a saved prompt loads it into Agent Factory
- [ ] Goal text and format are restored

#### Delete prompt button
- [ ] Removes prompt from storage
- [ ] Prompt disappears from list

### Navigation & Sidebar

#### All navigation links
- [ ] Command Center / Home
- [ ] Agent Factory / Orchestrator
- [ ] Runs / Run Explorer
- [ ] Knowledge Base / Repos
- [ ] Jobs
- [ ] Evaluations
- [ ] Library
- [ ] Audit Trail
- [ ] Settings
- [ ] Each link navigates to correct view
- [ ] Sidebar active state highlights current view
- [ ] Page header updates to match current view

### Auth Screens (when auth enabled)

#### Login form
- [ ] Username/password fields accept input
- [ ] Submit triggers `/api/auth/login`
- [ ] Success: redirects to main app
- [ ] Failure: shows error message
- [ ] Rate limited after repeated failures

#### Bootstrap form (first admin setup)
- [ ] Only appears when no users exist
- [ ] Creates admin account
- [ ] Redirects to login after bootstrap

### Workspace Selector (if present)

#### Create workspace
- [ ] Creates new workspace
- [ ] Workspace appears in selector

#### Switch workspace
- [ ] Data refreshes for selected workspace
- [ ] Vector index loads for correct workspace
- [ ] All views show workspace-scoped data

#### Delete workspace (admin only)
- [ ] Removes workspace
- [ ] Cannot delete default workspace
- [ ] Confirmation required

---

## Config-to-UI Consistency Checks

### ⚠️ WHY THIS SECTION EXISTS
Bugs where config.json gates silently override UI toggles. Every config-dependent feature must either:
1. Respect the user's per-action override, OR
2. Clearly disable the UI control with an explanation

| Config Key | UI Control | Expected Behavior |
|-----------|-----------|-------------------|
| `externalSkills.enabled` | "Search online for skills" checkbox | Per-spawn checkbox overrides global config |
| `externalSkills.allowRemote` | Same checkbox (disabled state) | Checkbox disabled with explanation text when remote not allowed |
| `externalSkills.mode` | Training mode selector | Mode sent to backend matches selector |
| `queue.enabled` | "Run in background queue" checkbox | Checkbox hidden or disabled when queue not enabled |
| `auth.enabled` | Login screen | Auth screens shown only when auth enabled |
| `pollingInterval` | Frontend polling frequency | Actual polling matches config value |
| `theme` | Theme toggle | UI theme matches config |

---

## Manual UI & Visual Integrity Audit
- Knowledge Base: "Add Repository" label does not overlap border or focus ring.
- Focus rings remain within fields and do not obscure text.
- Sidebar active state highlights current view; page header updates correctly.
- System Logs persist across view changes.
- Telemetry/analytics increment after successful spawns.

## UX & Edge Case Hunting
- Ambiguous goal input: verify "Requires Review" or low-confidence routing is flagged.
- Error states: invalid repo URLs and duplicate repo additions trigger notice system + log entries.
- Spawn failure: if backend is unreachable, Generation Chain steps should show error state (not misleading success with fallback timings).
- Toggle disconnect: UI toggle is checked but backend skips the feature with no visible feedback.
- Silent config override: config.json setting silently blocks a user-initiated action.

---

## Structured Report Template (Fill for Each E2E Run)

### Step-by-Step Execution Log
- [ ] Step 1: [clicks + inputs + view], Result:
- [ ] Step 2: [clicks + inputs + view], Result:
- [ ] Step 3: [clicks + inputs + view], Result:
- [ ] Step 4: [clicks + inputs + view], Result:
- [ ] Step 5: [clicks + inputs + view], Result:

### Functional Pass/Fail
| Workflow | Status | Notes |
| --- | --- | --- |
| Spawn (Agent Factory) |  |  |
| Generation Chain Timings |  |  |
| External Skills Toggle |  |  |
| Clone (Smart Clone) |  |  |
| Scan (System Scan) |  |  |
| Evaluate (Evaluation Lab) |  |  |

### UI Toggle/Button Audit
| Control | View | Sends Correct Data | Backend Processes | Result Matches UI |
| --- | --- | --- | --- | --- |
| Generate Flight Plan | Agent Factory |  |  |  |
| Format selector | Agent Factory |  |  |  |
| Search online for skills | Agent Factory |  |  |  |
| Training mode | Agent Factory |  |  |  |
| Run in background | Agent Factory |  |  |  |
| Save Prompt | Agent Factory |  |  |  |
| Copy to Clipboard | Agent Factory |  |  |  |
| Test Connection | Settings |  |  |  |
| Rebuild Index | Settings |  |  |  |
| Clear Prompts | Settings |  |  |  |
| Add Repository | Knowledge Base |  |  |  |
| Smart Clone | Knowledge Base |  |  |  |
| System Scan | Knowledge Base |  |  |  |
| Create Dataset | Evaluations |  |  |  |
| Run Evaluation | Evaluations |  |  |  |
| Cancel Job | Jobs |  |  |  |
| Export Audit | Audit Trail |  |  |  |

### Visual/UX Bug Log
| Issue | View | Severity | Notes |
| --- | --- | --- | --- |
|  |  |  |  |

### Telemetry Check
- totalSpawns updated: [yes/no]
- recent sessions updated: [yes/no]
- log entries created for clone/scan/errors: [yes/no]

---

## Manual Smoke
- First-run wizard appears on fresh install
- Settings panel updates config correctly
- Agent Factory generates valid flight plans
- Generation Chain shows real backend durations (not ~300ms simulation values)
- "Search online for skills" checkbox actually triggers external skills search (not silently skipped)
- Command Center loads with run/eval/prompt counts
- Run Explorer shows recent run details + comparison deltas
- Jobs view lists queued/active jobs
- Observability panel shows token + cost aggregates
- Evaluations view can create a dataset + evaluation with per-item grading (LLM rubric if enabled)
- Evaluations view can create retrieval benchmarks with precision/recall/MRR
- Library view shows saved prompts + agent templates
- Session history persists
- Decision Matrix section appears in generated plans
- Decision Trace summary appears in generated plans
- Retrieval gate + query expansion + RAG-Fusion + hybrid retrieval + RRF lines appear in Decision Matrix
- Semantic index line appears in Decision Matrix
- AGENTS.md appears first in Required Reading when present
- Telemetry increments after a successful spawn
- Repository size totals reflect actual folder sizes (spot-check against OS properties)
- Vector index rebuild succeeds (Settings → Rebuild index)
- Auth bootstrap + login flow works when auth is enabled
- RBAC policy editor saves valid JSON and blocks forbidden actions
- Audit Trail view lists recent events with user/workspace metadata
- Audit Trail export downloads CSV/JSON without errors
- Workspaces can be created, switched, and deleted (admin only)

## UI Visual Checks
- Repos view: "Add Repository" label does not overlap border or focus ring
- Repos view: input focus ring stays within the field and doesn't obscure text
- Jobs view shows status pill and cancel button for queued/running jobs
- Audit Trail filter/search stays responsive with >50 entries
- Evaluation Trends panel renders response + retrieval sparklines

## Automated E2E (Playwright)
- Navigation between all primary views (Command Center, Runs, Evaluations, Library, Knowledge Base)
- Save Prompt flow + Quick Access rendering
- Repository size labels populate (not "—")
- Invalid repo URL shows log error
- Duplicate repo shows notice + log entry
- Scan action emits log entries
- Spawn generates flight plan (Decision Matrix + AGENTS in output) and telemetry updates
- External skills checkbox: checked state produces non-"disabled" skip reason in response

### Run in Edge (Manual UI Pass)
- `npm run e2e:edge` (headed Edge run)
- `npm run e2e:edge:ui` (Playwright UI runner using Edge)
- `npm run e2e:edge -- tests/e2e/screenshots.spec.js` (capture docs screenshots)

### LLM Rubric (Optional)
- Start a local OpenAI-compatible endpoint:
  - **Ollama**: `ollama serve` then `ollama pull qwen2.5:14b-instruct` (or any model)
    - Endpoint: `http://localhost:11434/v1/chat/completions`
  - **LM Studio**: start server
    - Endpoint: `http://localhost:1234/v1/chat/completions`
- Update **Settings → LLM Endpoint** to match.
- Use **Test Connection** in Settings.
- Re-run an evaluation with LLM rubric items.

## Rate Limiter Notes
- The general API rate limiter (`apiLimiter`) is set to 1000 req/min for localhost use.
- The frontend polls 14+ endpoints concurrently; lower limits can exhaust the budget and block spawn requests.
- The write limiter (`writeLimiter`) is 10 req/min for expensive endpoints (spawn, clone, scan, rebuild).
- If spawns fail with "Too many requests", check `server/middleware/rate-limit.js`.

## Known Bug Classes

### Silent Config Override (FIXED: external skills gate)
**Pattern**: UI toggle sends correct data to backend, but a config.json gate silently blocks the action. The UI shows the toggle as active but the backend skips the feature.
**Detection**: For every UI toggle, verify the backend response reflects the toggle state — not just that the request was sent.
**Example**: "Search online for skills" checkbox was checked, frontend sent `externalSkills.online=true`, but `config.externalSkills.enabled=false` in config.json silently overrode the per-spawn toggle.

### Rate Limiter Exhaustion (FIXED: bumped to 1000 req/min)
**Pattern**: Frontend polls 14+ endpoints concurrently, exhausting the rate limiter budget before spawn/clone/scan requests can get through. These expensive operations fail silently.
**Detection**: Check browser console for "Failed to fetch" or "Too many requests" during spawn operations.

### Silent Error Propagation (FIXED: handleSpawn returns result)
**Pattern**: DataContext catches errors internally but returns void, so calling views have no way to distinguish success from failure.
**Detection**: After any async operation, verify the calling component receives and acts on the success/failure status.

## Cleanup
- Delete any saved prompts created solely for testing after verification completes.
