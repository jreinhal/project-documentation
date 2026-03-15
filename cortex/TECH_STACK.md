# CORTEX Tech Stack

## Overview

CORTEX is a self-hosted AI orchestration platform built with a React frontend and Node.js/Express backend. It uses file-based JSON persistence (no database), a custom vector search engine, and optional LLM integration for evaluation grading.

---

## Backend

| Technology | Version | Purpose |
|-----------|---------|---------|
| Node.js | 18.x / 20.x | Runtime (CI tests both) |
| Express | 5.2.1 | HTTP server framework |
| body-parser | 2.2.2 | Request body parsing middleware |
| cors | 2.8.6 | Cross-Origin Resource Sharing |
| express-rate-limit | 8.2.1 | API rate limiting (1000 req/min general, 10 req/min writes, 20 req/min auth) |
| zod | 4.3.6 | Schema validation and input parsing |
| string-similarity | 4.0.4 | Fuzzy string matching for search ranking |
| concurrently | 9.2.1 | Parallel process runner (backend + frontend dev) |

### Backend Architecture

- **Orchestrator** (`orchestrator.js`) — Generates Flight Plans via agent spawning; emits `___CORTEX_META___` performance sentinel in stdout
- **Agent Selector** (`agent-selector.js`) — Selects optimal agent profile based on goal analysis and stack profile alignment
- **Resource Matcher** (`resource-matcher.js`) — Scores and ranks repository resources against agent needs
- **Vector Index** (`vector-index.js`) — Hybrid TF-IDF hash-based (512-dimensional) + optional OpenAI-compatible embeddings
- **Query Expander** (`query-expander.js`) — Expands user queries for broader retrieval
- **LLM Reranker** (`llm-reranker.js`) — Optional LLM-based result reranking
- **Late Interaction Reranker** (`late-interaction-reranker.js`) — Secondary reranking strategy
- **Retrieval Gate** (`retrieval-gate.js`) — Gating mechanism for retrieval inclusion
- **Decision Trace** (`decision-trace.js`) — Records decision points during orchestration
- **Goal Analyzer** (`goal-analyzer.js`) — Parses and classifies user goals
- **Stack Profile Parser** (`stack-profile-parser.js`) — Parses technology stack profiles for scoring

### Authentication & Authorization

- Custom JWT implementation (HS256 signing, Base64URL encoding) — no third-party auth library
- Role hierarchy: `viewer` (1) < `editor` (2) < `admin` (3)
- Permission-based resource access control
- Auth is optional for localhost use

### Storage (File-Based JSON)

| Store | File | Purpose |
|-------|------|---------|
| Storage helper | `storage.js` | Atomic file I/O with error handling |
| Runs | `runs.json` | Run traces and metrics (max 200, LIFO) |
| Vector Index | `vector_index.json` | Semantic search index (per-workspace) |
| Embeddings | `vector_index_embeddings.json` | Embedding vectors (per-workspace) |
| Auth | `auth-store.js` | Tokens and sessions |
| Logs | `log-store.js` | Execution logs |
| Evaluations | `evaluations-store.js` | Evaluation results |
| Datasets | `datasets-store.js` | Dataset metadata |
| Eval Templates | `evaluation-templates-store.js` | Grading templates |
| Config | `config.json` | User configuration (gitignored) |

### Observability

- **Performance Tracking** — Backend emits `___CORTEX_META___` JSON sentinel parsed by `runs.js`
- **Audit Log** (`audit-log.js`) — User action audit trail with workspace metadata
- **Job Queue** (`job-queue.js`) — In-memory background job queue with file persistence
- **Observability Module** (`observability.js`) — Token and cost aggregate metrics

### API Route Structure

| Route | Method(s) | Purpose |
|-------|-----------|---------|
| `/api/repos` | GET | Repository listing and categories |
| `/api/spawn` | POST | Flight plan generation (write-limited) |
| `/api/config` | GET, POST | Configuration management |
| `/api/setup` | POST | First-run wizard |
| `/api/sessions` | GET, POST | Session history |
| `/api/analytics` | GET | Usage statistics and telemetry |
| `/api/categories` | GET | Repository category breakdown |
| `/api/runs` | GET | Run history with performance data |
| `/api/logs` | GET | Execution logs |
| `/api/evaluations` | GET, POST | Evaluation results |
| `/api/prompts` | GET, POST, DELETE | Saved prompts |
| `/api/jobs` | GET | Background job status |
| `/api/auth` | POST | Login, bootstrap, status |
| `/api/workspaces` | GET, POST, DELETE | Workspace management |
| `/api/users` | GET | User management |
| `/api/scim` | GET, POST | SCIM provisioning |
| `/api/stack-profile` | POST | Stack profile upload and parsing |
| `/manual` | GET | Static documentation |

---

## Frontend

| Technology | Version | Purpose |
|-----------|---------|---------|
| React | 19.2.0 | UI library |
| React DOM | 19.2.0 | DOM rendering |
| React Router DOM | 7.13.0 | Client-side routing |
| Vite | 7.2.4 | Build tool and dev server |
| @vitejs/plugin-react | 5.1.1 | React Fast Refresh for Vite |
| Tailwind CSS | 4.1.18 | Utility-first CSS framework |
| @tailwindcss/postcss | 4.1.18 | PostCSS plugin for Tailwind |
| PostCSS | 8.5.6 | CSS transformation pipeline |
| Autoprefixer | 10.4.24 | Browser vendor prefix automation |
| Framer Motion | 12.29.2 | Declarative animations and transitions |
| Lucide React | 0.563.0 | Icon library |
| clsx | 2.1.1 | Conditional CSS class composition |
| tailwind-merge | 3.4.0 | Tailwind class conflict resolution |

### Frontend Architecture

- **Context Providers** — `DataContext`, `ConfigContext`, `AuthContext`, `WorkspaceContext`
- **Views** — Home, Orchestrator, Runs, Knowledge, Jobs, Evaluations, Audit, Library, Settings
- **Components** — ChecklistModal, DirectoryBrowser, EmptyState, NavItem, RepoTable, Sparkline, SpawnTimeline, TrendCard
- **Utilities** — `api.js` (fetch wrapper), `constants.js`, `utils.js`
- **Vite Config** — Code splitting with vendor bundles: `react-vendor`, `motion-vendor`, `ui-vendor`, `style-vendor`

### Custom Color Palette (Tailwind)

Steel, Indigo, Green, Gold, Red — dark theme with `bg-steel-950` base

---

## Testing

| Technology | Version | Purpose |
|-----------|---------|---------|
| Vitest | 4.0.18 | Unit and integration test runner |
| Playwright | 1.52.0 | E2E browser automation |
| Supertest | 7.2.2 | HTTP endpoint assertion |
| jsdom | 28.0.0 | DOM environment for component tests |
| @testing-library/react | 16.3.2 | React component testing utilities |
| @testing-library/jest-dom | 6.9.1 | Custom DOM matchers |

### Test Structure

| Layer | Location | Runner | Environment |
|-------|----------|--------|-------------|
| Server unit/integration | `server/__tests__/**/*.test.js` | Vitest | Node |
| Client component | `client/src/__tests__/**/*.test.{js,jsx}` | Vitest | jsdom |
| E2E | `tests/e2e/**/*.spec.js` | Playwright | Chromium + Edge |

---

## Code Quality

| Technology | Version | Purpose |
|-----------|---------|---------|
| ESLint | 10.0.0 (root) / 9.39.1 (client) | Linting |
| Prettier | 3.8.1 | Code formatting |
| eslint-config-prettier | 10.1.8 | Disable conflicting ESLint rules |
| eslint-plugin-react-hooks | 7.0.1 | React Hooks best practices |
| eslint-plugin-react-refresh | 0.4.24 | Vite React Refresh compatibility |

---

## CI/CD

| Platform | Workflow | Purpose |
|----------|----------|---------|
| GitHub Actions | `ci.yml` | Build matrix (Node 18.x + 20.x): install, build, lint, format, unit tests, E2E |
| GitHub Actions | `marketability-artifacts.yml` | Demo artifact generation |

### CI Pipeline Steps

1. Checkout code
2. Install dependencies (root + client)
3. Build frontend (`vite build`)
4. Lint checks (`eslint`)
5. Format validation (`prettier --check`)
6. Unit/integration tests (`vitest run`)
7. E2E tests (`playwright test`)

---

## Key Architectural Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Database | None — file-based JSON | Local-first tool; simplicity over scale |
| ORM | None | Direct file I/O with atomic writes |
| Message Queue | In-memory with file persistence | Single-user localhost; no external deps |
| Vector Search | Custom hash TF-IDF + optional embeddings | Zero-dependency default; opt-in LLM embeddings |
| Auth Library | Custom JWT (HS256) | Minimal attack surface for local tool |
| CSS Framework | Tailwind CSS 4 | Utility-first, rapid iteration |
| Build Tool | Vite 7 | Fast HMR, ESM-native |
| State Management | React Context | Sufficient for single-page app complexity |
| Process Management | concurrently | Simple dual-server dev mode |

---

## Ports

| Service | Default Port | Notes |
|---------|-------------|-------|
| Backend (Express) | 3001 | API server |
| Frontend (Vite) | 5173 | Dev server with HMR |
| LLM Endpoint (Ollama) | 11434 | Optional, configurable |
| LLM Endpoint (LM Studio) | 1234 | Optional, configurable |

---

## Dependency Counts

| Category | Count |
|----------|-------|
| Root production dependencies | 6 |
| Root dev dependencies | 8 |
| Client production dependencies | 7 |
| Client dev dependencies | 16 |
| **Total direct production** | **13** |
| **Total direct dev** | **24** |
