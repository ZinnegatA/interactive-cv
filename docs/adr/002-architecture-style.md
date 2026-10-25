# ADR-002: Architecture style - hexagonal modular monolith

## Context

The backend serves CV content and, later, an AI chat grounded in that content. It is also part of what the project presents, so the architecture must be proportionate to the problem and hold up under review by experienced engineers.

- One maintainer (me) handles all development and operations.
- Traffic is low and bursty; there is no sustained-load requirement.
- Hosting budget is ~$0/month.
- Workloads differ: CV reads are cheap and fast; chat holds long-lived streaming responses, depends on a third-party LLM, and costs money per request.
- Later phases add asynchronous work (re-embedding, notifications) that must run outside the request path.

The style shapes the repository layout, module boundaries, CI checks, and deployment topology, so it is decided before scaffolding.

## Decision

The backend is a **modular monolith**: one deployable NestJS application, split into modules that each own one business capability, with hexagonal layering inside each module. Boundaries are checked by dependency-cruiser in CI; a violation fails the build.

- **Public surface only.** A module exposes its application ports and DTOs through a single entry point. Other modules import only from it, never from internals.
- **Data ownership.** A module's tables are read and written only by its own infrastructure layer, regardless of where schema files live. Other modules get that data through the owner's ports.
- **Layering.** `domain` imports nothing outside itself (no NestJS, Drizzle, Zod, I/O etc.); `application` imports `domain`; `infrastructure` implements their ports; `presentation` calls use cases, never repositories.
- **Communication.** Cross-module calls go through exported application ports. Domain events come later via a transactional outbox, with the mechanism decided in its own ADR.
- **Shared kernel.** `shared/` holds config, logging, error handling, and the DB client. It contains no business logic and imports no feature module.
- **Runtimes.** Background processing runs as a separate worker entrypoint built from the same codebase, not as a separate service.

| Module            | Responsibility                                                            | Depends on                      |
| ----------------- | ------------------------------------------------------------------------- | ------------------------------- |
| `profile`         | CV content (experience, skills, projects) served to the web app           | -                               |
| `knowledge`       | Source documents, chunking, embeddings in pgvector                        | `profile` (CV content as input) |
| `chat`            | RAG answers with citations, LLM integration, guardrails and cost controls | `knowledge`                     |
| `contact` (later) | Contact form and notifications                                            | -                               |

## Alternatives considered

- **Unstructured monolith:** fastest start, but boundaries erode, modules couple through shared internals and tables, and later extraction gets expensive.
- **Microservices:** independent deployment, scaling, and failure isolation, at the cost of multiple pipelines and runtimes, network failure handling, contract versioning, and likely a broker. Not needed for the current scope.

## Consequences

- One application to build, deploy, monitor, and host; boundaries are explicit and verified on every PR.
- Boundary tooling and layering add setup cost and some boilerplate. Read-model modules like `profile` will have a thin domain layer; that is accepted, not filled with ceremony.
- `chat` shares a process with everything else, so streaming connections, LLM latency, or provider failures could degrade the CV page. Mitigations: timeouts, concurrency limits, per-IP rate limits, a daily token budget, and graceful degradation (the CV stays usable when chat is down).
- Modules can't be scaled or deployed independently. A module is extracted only when one of these is observed, not anticipated:
  1. its load degrades other modules and in-process limits don't fix it;
  2. it needs runtime characteristics the shared host can't provide (e.g. long-lived streams under a request-duration limit);
  3. its failures or spend need hard isolation;
  4. it needs a different deploy cadence or risk controls.

  `chat` is the module most likely to be extracted into its own service. Because it reaches other modules only through ports, extraction means swapping in-process implementations of those ports for network clients (HTTP or a queue); domain and application code don't change.

- This ADR does not choose hosting or the AI agent design. Both are separate decisions and must support the shape decided here: one API deployable, plus a worker entrypoint later.
