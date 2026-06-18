# Forge — Architecture Documentation

> **Generated from source code analysis** — Every diagram reflects the actual implementation discovered in the codebase, not assumptions or aspirational design.

---

## Table of Contents

1. [High-Level System Architecture](#1-high-level-system-architecture)
2. [Sentinel Brain Internal Architecture](#2-sentinel-brain-internal-architecture)
3. [Execution Workflow](#3-execution-workflow)
4. [Real-Time Streaming Architecture](#4-real-time-streaming-architecture)
5. [Data & Embedding Architecture](#5-data--embedding-architecture)
6. [Deployment Architecture](#6-deployment-architecture)
7. [End-to-End Sequence Diagram](#7-end-to-end-sequence-diagram)
8. [Architectural Insights](#architectural-insights)

---

## 1. High-Level System Architecture

![High-Level System Architecture](./high-level-architecture.png)

### Overview

Forge is structured as a **monorepo** with clear separation between the frontend, backend API, and multiple Python packages that form the autonomous engineering core.

### Key Components Discovered

| Layer | Technology | Location |
|-------|-----------|----------|
| **Frontend** | Next.js 14 (App Router, RSC + Client Components) | `apps/web/` |
| **Backend** | FastAPI (Python 3.12, async) | `apps/api/` |
| **Sentinel Brain** | LangGraph StateGraph (11-node pipeline) | `packages/workflows/` |
| **Agent Library** | 11 specialized agents (Pydantic + LangChain) | `packages/agents/` |
| **GitHub Integration** | httpx-based REST client | `packages/github/` |
| **Repository Analysis** | tree-sitter + file walker | `packages/repository-analysis/` |
| **Vector Store** | Qdrant async client wrapper | `packages/vector-store/` |
| **Shared Types** | TypeScript shared package | `packages/shared/` |

### External Services

| Service | Purpose | Auth |
|---------|---------|------|
| **PostgreSQL 16** | Primary data store (repos, files, issues, runs) | `DATABASE_URL` |
| **Qdrant** | Vector similarity search for code chunks | `QDRANT_URL` + `QDRANT_API_KEY` |
| **Google Gemini** | LLM (`gemini-2.5-flash`) + Embeddings (`gemini-embedding-001`) | `GOOGLE_API_KEY` |
| **GitHub REST API** | Repo metadata, issues, PR creation | `GITHUB_TOKEN` |

---

## 2. Sentinel Brain Internal Architecture

![Sentinel Brain Architecture](./sentinel-brain-architecture.png)

### LangGraph Pipeline — 11 Nodes, 6 Phases

The Sentinel Brain is implemented as a compiled `langgraph.graph.StateGraph` with **11 nodes** organized into **6 phases**:

| Phase | Node | Agent Class | Output |
|-------|------|-------------|--------|
| **1. Understand** | `repo_analyzer` | `RepoAnalyzerAgent` | `RepoContext` |
| | `issue_analyzer` | `IssueAnalyzerAgent` | `IssueAnalysis[]`, `SelectedIssue` |
| **2. Retrieve** | `retrieve_context` | `RetrieveContextAgent` | `RetrievedChunk[]` |
| | `load_full_files` | `FileLoader` (utility) | `FullFileContext[]` |
| **3. Plan** | `planner` | `PlannerAgent` | `Plan` |
| **4. Execute** | `developer` | `DeveloperAgent` | `CodeChange[]` |
| | `apply_patches` | `PatchExecutor` (utility) | `PatchResult[]` |
| **5. Validate** | `validator` | `ValidatorAgent` | `ValidationResult` |
| | `test_agent` | `TestAgent` | `TestResult[]` |
| | `reviewer` | `ReviewerAgent` | `Review` |
| **6. Ship** | `pr_generator` | `PRGeneratorAgent` | `PullRequestDraft` |

### Self-Repair Feedback Loops

Three conditional edges route execution **back to the developer** node when failures are detected, up to `max_iterations` (default: 3):

| Trigger | Route Function | Condition |
|---------|---------------|-----------|
| Patch application failure | `route_after_apply_patches()` | `any(not r.applied for r in patch_results)` |
| Validation failure | `route_after_validator()` | `validation_result.passed == False` |
| Review rejection | `route_after_reviewer()` | `review.approved == False` |

On retry, a `RepairContext` is assembled containing the specific failure details (failed patches with on-disk file content, validation issues, review comments) so the LLM corrects its exact mistakes.

### Dependency Injection

The graph is **application-agnostic** — it imports nothing from `app.*`:

```python
graph = build_graph(
    llm=ChatGoogleGenerativeAI(model="gemini-2.5-flash"),
    embedder=CodeEmbedder(api_key, model),
    vector_store=VectorStore(qdrant_url, api_key),
).compile()
```

---

## 3. Execution Workflow

![Execution Workflow](./execution-workflow.png)

### Complete Task Lifecycle

```
User Request (POST /api/agent-runs)
  ├── 1. BrainService.start(run_id)
  │     ├── Load AgentRun from PostgreSQL
  │     ├── Clone repo → WorkspaceManager (git clone --depth=1)
  │     └── Load BrainInputs (files, issues from Postgres)
  │
  ├── 2. LangGraph Pipeline (graph.astream)
  │     ├── repo_analyzer    → RepoContext
  │     ├── issue_analyzer   → SelectedIssue
  │     ├── retrieve_context → RetrievedChunk[] (Qdrant search)
  │     ├── load_full_files  → FullFileContext[] (workspace read)
  │     ├── planner          → Plan
  │     ├── developer        → CodeChange[] (+ RepairContext on retry)
  │     ├── apply_patches    → PatchResult[] (git apply)
  │     │   └── ⟲ back to developer if patches fail
  │     ├── validator        → ValidationResult
  │     │   └── ⟲ back to developer if validation fails
  │     ├── test_agent       → TestResult[]
  │     ├── reviewer         → Review
  │     │   └── ⟲ back to developer if review rejects
  │     └── pr_generator     → PullRequestDraft
  │
  ├── 3. Post-Graph Actions (PAT stays at app layer)
  │     ├── GitService.commit_and_push() → GitResult
  │     └── GitHubPRService.create_pull_request() → PR URL
  │
  └── 4. Completion
        ├── Update AgentRun → status: "completed"
        └── WorkspaceManager.cleanup()
```

### Key Design Decision: PAT Security

The Git PAT **never enters `SentinelState`** (which is streamed and persisted). Git operations (`commit_and_push`, `create_pull_request`) run **after** the LangGraph pipeline completes, within `BrainService` at the application boundary.

---

## 4. Real-Time Streaming Architecture

![Real-Time Streaming Architecture](./realtime-streaming.png)

### Implementation: Snapshot Persistence + REST Polling

Forge does **not** use WebSockets. Instead, it implements a snapshot-based streaming pattern:

```
LangGraph graph.astream(state, stream_mode="updates")
  │
  ├── For each node completion:
  │     ├── accumulator.update(partial)        # Merge node output
  │     ├── _active_snapshots[run_id] = snap   # In-memory cache
  │     └── agent_run_repo.update_status(      # Persist to PostgreSQL
  │           session, run_id, status,
  │           current_node=node_name,
  │           result=snapshot)
  │
  └── Frontend polls: GET /api/agent-runs/{run_id}
        └── Returns: AgentRunOut { status, current_node, result }
```

### Persisted Result Fields

```python
_RESULT_FIELDS = (
    "repo_context", "issue_analyses", "selected_issue",
    "plan", "code_changes", "patch_results",
    "validation_result", "test_results", "review",
    "pull_request_draft", "iteration", "repair_context",
)
```

### Frontend Component Mapping

| Result Field | UI Component | File |
|-------------|-------------|------|
| `repo_context` | `OutputCard` | `output-card.tsx` |
| `issue_analyses` | `IssueAnalysisCard` | `issue-analysis-card.tsx` |
| `plan` | `OutputCard` | `output-card.tsx` |
| `code_changes` | `DiffViewer` | `diff-viewer.tsx` |
| `validation_result` | `OutputCard` | `output-card.tsx` |
| `test_results` | `OutputCard` | `output-card.tsx` |
| `review` | `OutputCard` | `output-card.tsx` |
| `pull_request_draft` | `PRDraftCard` | `pr-draft-card.tsx` |
| `pull_request_url` | "View Pull Request" button | `agent-run-detail.tsx` |

---

## 5. Data & Embedding Architecture

![Data & Embedding Architecture](./mcp-architecture.png)

### Repository Ingestion Pipeline

The ingestion pipeline runs as an async background task triggered by repository registration:

```
POST /api/repositories
  ├── GitHubClient.get_repository()    → Fetch metadata
  ├── GitHubClient.get_issues()        → Fetch open issues (paginated)
  ├── git clone --depth=1              → Local clone
  ├── RepositoryAnalyzer.analyze()     → Walk file tree
  │     ├── tree-sitter                → Extract symbols (functions, classes, methods)
  │     └── CodeChunker.chunk()        → Split into embeddable chunks
  ├── file_repo.upsert_files()         → PostgreSQL
  ├── issue_repo.upsert_issues()       → PostgreSQL
  ├── CodeEmbedder.embed_batch()       → Gemini Embedding 001 (batches of 100)
  └── VectorStore.upsert()             → Qdrant collection "repo_{id}"
```

### Vector Storage Schema

Each embedded chunk stored in Qdrant contains:

```json
{
  "id": "sha256(repo_id:file_path:chunk_index)[:16]",
  "vector": [/* float[] from Gemini */],
  "payload": {
    "repository_id": "uuid",
    "file_path": "src/main.py",
    "language": "python",
    "chunk_index": 0,
    "start_line": 1,
    "end_line": 50,
    "content": "def main(): ..."
  }
}
```

### Runtime Retrieval

During agent execution, the `retrieve_context` node:
1. Builds a semantic query from `SelectedIssue.title + body`
2. Embeds the query via `CodeEmbedder`
3. Searches Qdrant with cosine similarity (`limit=5`)
4. Returns `RetrievedChunk[]` for downstream agents

> **Note:** The codebase does not implement MCP (Model Context Protocol) integrations. The embedding and retrieval pipeline serves the role of providing contextual code access to agents.

---

## 6. Deployment Architecture

![Deployment Architecture](./deployment-architecture.png)

### Infrastructure Stack

| Component | Platform | Configuration |
|-----------|----------|--------------|
| **Frontend** | Vercel | Next.js 14, pnpm monorepo, TailwindCSS |
| **Backend** | Railway | Docker (python:3.12-slim), uv workspace, Alembic migrations |
| **PostgreSQL** | Railway / Docker Compose | PostgreSQL 16, asyncpg driver |
| **Qdrant** | Qdrant Cloud / Docker Compose | Latest image, ports 6333/6334 |

### Backend Dockerfile

```dockerfile
FROM python:3.12-slim
RUN apt-get update && apt-get install -y git  # Required by GitPython
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv
COPY . .
RUN uv sync --frozen
CMD uv run alembic upgrade head && \
    uv run uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}
```

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `DATABASE_URL` | PostgreSQL connection (asyncpg) |
| `QDRANT_URL` | Qdrant vector DB endpoint |
| `QDRANT_API_KEY` | Qdrant authentication |
| `GOOGLE_API_KEY` | Gemini LLM + Embeddings |
| `GITHUB_TOKEN` | GitHub API (issues, PRs) |
| `CORS_ORIGINS` | Allowed frontend origins |
| `CORS_ORIGIN_REGEX` | Preview deployment regex |
| `CLONE_BASE_DIR` | Workspace clone directory |

### Monorepo Structure

```
Sentinel/
├── apps/
│   ├── web/          → Next.js 14 (Vercel)
│   └── api/          → FastAPI (Railway)
├── packages/
│   ├── agents/       → forge_agents (11 AI agents)
│   ├── workflows/    → forge_workflows (LangGraph)
│   ├── github/       → forge_github (REST client)
│   ├── repository-analysis/  → tree-sitter + chunker
│   ├── vector-store/ → Qdrant wrapper
│   └── shared/       → TypeScript shared types
├── Dockerfile        → Railway build
├── docker-compose.yml → Local dev (Postgres + Qdrant)
├── pyproject.toml    → uv workspace root
└── pnpm-workspace.yaml → Node workspace root
```

---

## 7. End-to-End Sequence Diagram

![End-to-End Sequence Diagram](./end-to-end-sequence-diagram.png)

### Complete Interaction Flow

The sequence diagram traces every inter-system call for a complete task execution cycle:

1. **Repository Registration**: User → Frontend → Backend → GitHub API → PostgreSQL → Qdrant
2. **Ingestion**: Background task clones repo, analyzes files, embeds chunks
3. **Agent Run Trigger**: User selects issue → POST /api/agent-runs → asyncio.create_task
4. **Pipeline Execution**: 11-node LangGraph pipeline with streaming snapshots
5. **Self-Repair Loops**: Up to 3 developer↔validator/reviewer iterations
6. **Post-Pipeline**: Git commit+push, GitHub PR creation
7. **Frontend Polling**: REST polling with progressive result rendering

---

## Architectural Insights

### Key Design Patterns

| Pattern | Implementation | Rationale |
|---------|---------------|-----------|
| **Dependency Injection** | `build_graph(llm, embedder, vector_store)` | Graph nodes are pure; no database/config imports |
| **Snapshot Streaming** | `_active_snapshots` + DB persistence per node | Frontend sees incremental results without WebSockets |
| **PAT Isolation** | Git ops run post-graph, PAT never in state | Prevents credential leakage in streamed/persisted state |
| **Self-Repair Loops** | 3 conditional edges back to developer | Auto-corrects patch failures, validation issues, review rejections |
| **Workspace Isolation** | Per-run clone directory, cleanup on completion | Prevents cross-run file contamination |
| **Background Tasks** | `asyncio.create_task()` for both ingestion and agent runs | Non-blocking API responses (HTTP 202) |

### Security Considerations

- **PAT never enters SentinelState** — Git and PR operations execute after the pipeline, at the application boundary
- **PAT redaction** — `GitService._redact()` strips tokens from error messages before logging/persistence
- **CORS hardening** — Explicit origin list + optional regex for preview deployments
- **Workspace cleanup** — Always runs in `finally` block, even on failure

### Scalability Notes

- **Single-process execution** — `_active_snapshots` is an in-memory dict; horizontal scaling would require shared state (Redis)
- **Recursion limit** — Hard backstop at 50 LangGraph node executions
- **Embedding batch size** — 100 chunks per Gemini API call
- **Qdrant batching** — 100 vectors per upsert

### Technology Decisions

| Decision | Choice | Alternative Considered |
|----------|--------|----------------------|
| LLM Provider | Google Gemini 2.5 Flash | OpenAI GPT-4 |
| Embedding Model | Gemini Embedding 001 | OpenAI text-embedding-3 |
| Orchestration | LangGraph StateGraph | CrewAI, AutoGen |
| Vector DB | Qdrant | Pinecone, Weaviate |
| Backend | FastAPI (async) | Django, Flask |
| Frontend | Next.js 14 (App Router) | Vite React |
| Package Manager | uv (Python), pnpm (Node) | pip, npm |
| Deployment | Railway + Vercel | AWS, GCP |

---

*Architecture documentation generated from source code analysis of the Forge/Sentinel codebase.*
