# Project Charter - Interactive CV

Owner: Ruslan Ibragimov | Version: 1.0

## 1. Problem/Motivation

A PDF CV claims skills (NestJS, event-driven, AI tooling, CI/CD) but proves none.
This project is a living artifact that demonstrates them in production: the website is the CV,
and the repository behind it - code, ADRs, CI, architecture - is the evidence.

## 2. Goals (measurable)

- G1: A visitor can get answers about my experience via an AI chat grounded in my CV.
  Answers cite source documents; the golden eval set passes in CI.
- G2: The public repo demonstrates senior-level practices: hexagonal modular monolith with
  boundaries enforced in CI, unit/integration/e2e tests, CI/CD, IaC, and an ADR for every
  hard-to-reverse decision.
- G3: The site URL replaces the PDF as the primary link in every job application
  and on my LinkedIn and GitHub profiles.

## 3. Non-goals (as important as goals)

- Not a CMS/multi-user platform - no user accounts or auth
- Not a blog (v1)
- No Kubernetes, no Kafka, no multi-LLM-provider abstraction
- Not a general-purpose chatbot - the chat answers questions about me and nothing else
- No heavy frontend animation; design polish comes after v1

## 4. Audience & key use cases

- Recruiter: who is he, stack, contact, download PDF, ask the chat a quick question
- Hiring manager/tech lead: architecture, repo, ADRs, CI, chat
- Owner (me): update CV content by editing versioned seed data and merging - no code changes

## 5. Scope (v1/MVP)

In: CV page (content served from Postgres via the API), AI chat with guardrails and evals,
static PDF download, contact links (email, LinkedIn, GitHub), CI/CD pipeline, deployed URL

Later: event-driven backbone (transactional outbox → queue → worker), contact form with
notifications, OpenTelemetry tracing incl. LLM spans, "Under the hood" page, MCP server,
PDF generated from data, design polish and dark mode

## 6. Success criteria/Definition of Done for v1

- Deployed on a public domain over HTTPS
- Chat answers only from CV data, cites its sources, and refuses or redirects off-topic and
  prompt-injection attempts; the golden eval set runs in CI and gates merges that touch the chat
- CV page LCP p75 < 2.5 s (Core Web Vitals "good"); chat time-to-first-token p95 < 3 s
- LLM cost ≤ $5/month, enforced by a daily token budget with a circuit breaker, plus per-IP
  rate limits
- Pipeline: lint → typecheck → test → build → deploy on merge to `master`

## 7. Constraints

- Budget: $0/month hosting (free tiers only); LLM ≤ $5/month, hard-capped
- Tech constraints fixed upfront: NestJS, TypeScript, PostgreSQL, Next.js
- Free tiers imply cold starts, quotas, and terms that can change without notice

## 8. Risks & mitigations

- LLM hallucinates about my experience → grounding/RAG on CV only, system prompt guardrails,
  mandatory citations, eval set of test questions
- Cost abuse/prompt injection on public chat → rate limiting, token caps, daily budget circuit
  breaker, budget alerts
- Over-engineering/never shipping → strict MVP scope, small shippable milestones, Phase 3+
  explicitly deferred
- Personal data exposure → publish only what is on the public CV; no secrets or visitor data in logs
- Static PDF drifts from site content → regenerate it as a release checklist item until it is
  generated from data

## 9. Milestones

- Sprint 0: charter, ADR-0001..0003, monorepo, CI, API foundation, Profile module, tests,
  web renders CV from the API. Deploy blocked on the hosting decision
- Sprint 1: hosting decision + deploy pipeline, full CV data model and seed, PDF download,
  contact links, domain + HTTPS
- Sprint 2: Knowledge module - document ingestion, chunking, embeddings
- Sprint 3: Chat module - RAG, `LlmPort` with a single adapter, SSE streaming, citations
- Sprint 4: guardrails (rate limits, token budget, injection defence), golden eval set in CI
- Sprint 5: hardening, performance check against §6, v1 launch

## 10. Open questions

- Development stack: NestJS, TypeScript, PostgreSQL, Next.js → ADR in Sprint 0
- Hosting within $0 (blocks deploy). Evaluate against free-tier candidates
  (e.g. Vercel Hobby + Neon free + a free container host) → ADR in Sprint 1
- LLM provider and model within the $5/month cap → ADR in Sprint 3
