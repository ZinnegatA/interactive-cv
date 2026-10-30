# Roadmap

Progress towards v1 of the Interactive CV. Scope and goals are defined in the [charter](charter.md);
decisions are recorded as [ADRs](adr/). This file tracks sequencing and progress only, so it carries
no dates or estimates.

**How to read it**

- Work is grouped into sprints, in order. Each sprint has a goal; it is done when every item is
  checked and its exit criteria hold.
- An item is checked only when it is merged to `master` with green CI.
- Items that need a tool or design decision first are listed under
  [Open decisions](#open-decisions). A decision leaves that table when its ADR or PR is merged.
- This file is updated in the same PR that completes an item.

## Status

| Sprint | Theme                   | Status      |
| ------ | ----------------------- | ----------- |
| 0      | Walking skeleton        | In progress |
| 1      | Deploy and CV content   | Not started |
| 2      | Knowledge               | Not started |
| 3      | Chat                    | Not started |
| 4      | Guardrails and evals    | Not started |
| 5      | Hardening and v1 launch | Not started |

## Sprint 0 - Walking skeleton

**Goal:** a thin end-to-end slice works: Next.js renders the CV using data it fetches from the
NestJS API, which reads it from Postgres. Every merge to `master` is gated by green CI, and
production images build in CI. Deployment itself is Sprint 1.

### Foundations

- [x] Project charter
- [x] ADR-001: Git workflow - squash-merge, Conventional Commit PR titles
- [x] ADR-002: Architecture style - hexagonal modular monolith
- [x] ADR-003: Repository structure - pnpm workspaces monorepo
- [x] Roadmap
- [ ] README: what the project is, architecture summary, local setup, testing, ADR index

### Monorepo skeleton

- [ ] Workspaces: `apps/api`, `apps/web`, `packages/contracts`
- [x] Node.js and pnpm versions pinned
- [x] Shared strict TypeScript config and lint/format configs at the repository root
- [x] LF line endings enforced via `.gitattributes`
- [ ] Workspace dependency direction (apps depend on packages, never the reverse) enforced by
      dependency-cruiser
- [ ] Build, lint and typecheck pass for all workspaces from a clean clone

### CI

- [x] PR title validation (ADR-001)
- [ ] PR workflow: cached install, lint, typecheck, test, build
- [x] Branch protection on `master`: required checks, squash-merge only

### API foundation

- [ ] Configuration validated at startup; `.env.example` documents every variable
- [ ] Structured JSON logs with request IDs
- [ ] RFC 9457 problem-details errors from a single global exception filter
- [ ] Health checks: `GET /health/live`, `GET /health/ready`
- [ ] OpenAPI served at `/api/docs` outside production and exported as a CI artifact
- [ ] Graceful shutdown
- [ ] Security baseline: security headers, CORS allowlist, request body size limit
- [ ] Database access, with migrations run as a separate step, never on boot
- [ ] Profile module: `GET /api/v1/profile` built with hexagonal layering, backed by a minimal seed
- [ ] Module boundary and layering rules enforced in CI

### Tests

- [ ] Integration test: Profile repository against a real, pgvector-capable Postgres
- [ ] E2E test: `GET /api/v1/profile` through the full application with a real database
- [ ] Both run in CI

### Web

- [ ] Next.js renders the CV from the API using types from `packages/contracts`

### Deployable artifacts

- [ ] API image built with `pnpm deploy`, containing production dependencies only
- [ ] Web image built independently of the API image
- [ ] Both images build in CI (not pushed or deployed)

**Exit criteria:** a fresh clone runs the stack locally by following the README; the web app renders
seeded CV data from `GET /api/v1/profile`; every merge to `master` passes the full PR workflow.

## Sprint 1 - Deploy and CV content

**Goal:** the site is live on a public domain over HTTPS and deploys automatically on merge.

- [ ] Infrastructure as code for the chosen hosting platform
- [ ] Deploy pipeline: lint → typecheck → test → build → deploy on merge to `master`
- [ ] Migrations run as a pipeline step before the new version receives traffic
- [ ] Global rate limiting before public exposure
- [ ] Full CV data model and versioned seed; updating the CV requires no code change
- [ ] Static PDF download
- [ ] Contact links: email, LinkedIn, GitHub
- [ ] Custom domain with HTTPS

## Sprint 2 - Knowledge

**Goal:** CV content is ingested, chunked and embedded so the chat can retrieve it.

- [ ] `knowledge` module reads CV content through the `profile` module's ports
- [ ] Document ingestion and chunking
- [ ] Embeddings stored in pgvector
- [ ] Re-indexing on demand when CV content changes (event-driven re-embedding comes post-v1)
- [ ] Retrieval verified against a set of sample questions

## Sprint 3 - Chat

**Goal:** visitors can ask questions about the CV and get streamed, cited answers.

- [ ] LLM access behind an `LlmPort` with a single adapter
- [ ] RAG answers with citations to source documents
- [ ] Tool use
- [ ] Streaming over SSE
- [ ] Isolation from the CV page: timeouts, concurrency limits, graceful degradation when chat is
      unavailable

## Sprint 4 - Guardrails and evals

**Goal:** the chat stays on topic, resists abuse, stays within budget, and is regression-tested.

- [ ] Scope restriction: answers only questions about Ruslan, redirects off-topic questions
- [ ] Prompt-injection defence
- [ ] Per-IP and per-session rate limits
- [ ] Daily token budget with a circuit breaker
- [ ] Answers cite source documents and never invent experience
- [ ] Golden eval set (deterministic checks and LLM-as-judge) runs in CI and gates merges that touch
      the chat

## Sprint 5 - Hardening and v1 launch

**Goal:** every item in the v1 definition of done is verified in production.

- [ ] Performance measured against the v1 targets
- [ ] v1 definition of done verified
- [ ] v1 launched

## v1 definition of done

From [charter §6](charter.md). Checked when verified in production.

- [ ] Public domain over HTTPS
- [ ] Chat answers only from CV data, cites sources, and refuses or redirects off-topic and
      prompt-injection attempts
- [ ] Golden eval set gates merges that touch the chat
- [ ] CV page LCP p75 < 2.5 s
- [ ] Chat time-to-first-token p95 < 3 s
- [ ] LLM cost ≤ $5/month, enforced by a daily token budget with a circuit breaker and per-IP limits
- [ ] Pipeline: lint → typecheck → test → build → deploy on merge to `master`

## Post-v1

- [ ] Event-driven backbone: transactional outbox, relay, queue, worker entrypoint; idempotent
      consumers, retries with backoff, dead-letter queue and redrive runbook
- [ ] `contact` module with notifications
- [ ] End-to-end OpenTelemetry tracing, including LLM spans (tokens, latency, cost)
- [ ] "Under the hood" page: architecture, ADRs, CI and eval status, live trace of a chat request
- [ ] MCP server exposing the CV and projects as tools
- [ ] PDF generated from CV data
- [ ] Failure-test write-up
- [ ] Design polish and dark mode

## Open decisions

| Decision                                                 | Needed by                 |
| -------------------------------------------------------- | ------------------------- |
| Test tooling                                             | Sprint 0 - CI and tests   |
| ORM, migrations and schema location                      | Sprint 0 - API foundation |
| Contract authoring (shared schemas vs OpenAPI-generated) | Sprint 0 - API foundation |
| Config validation and logging libraries                  | Sprint 0 - API foundation |
| Hosting platform                                         | Sprint 1                  |
| Domain name                                              | Sprint 1                  |
| Embedding model                                          | Sprint 2                  |
| LLM provider and model                                   | Sprint 3                  |
