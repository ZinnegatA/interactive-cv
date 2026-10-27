# ADR-003: Repository structure - monorepo with pnpm workspaces

## Context

The project has two deployables: a NestJS API (modular monolith, ADR-002, which will later also run
as a worker process) and a Next.js web app. They share API contracts (request/response shapes for CV
data and chat). One engineer develops everything, with CI enforced from the start (ADR-001). The
repository is also part of the portfolio, so reviewers will read its structure.

The structure must provide:

1. Atomic changes across a contract and its consumers: one PR, one review, one CI run.
2. One source of truth for tooling: TypeScript config, lint, commit rules, CI.
3. Low maintenance overhead for a solo developer.
4. Independent build and deployment of web and API.
5. Lean production images: the API image contains no frontend dependencies, and vice versa.
6. Clarity for external reviewers.

With one team and a monolithic backend, polyrepo offers little. The practical question is whether
the repo is a single package or a workspace-based monorepo, and how much tooling the monorepo needs.

## Decision

The repository is a **monorepo with pnpm workspaces** and no build orchestrator. `pnpm --filter`
covers targeted builds and tests.

```
/
├── apps/
│   ├── api/              # NestJS monolith (ADR-002): HTTP and, later, worker entrypoints
│   └── web/              # Next.js
├── packages/
│   ├── contracts/        # shared API schemas/types; no Nest/React dependencies
│   └── config/           # shared tsconfig and lint presets
├── docs/
├── .github/workflows/
├── pnpm-workspace.yaml
└── package.json          # repo-wide dev tooling and scripts only
```

- **Dependency direction.** `apps/*` may depend on `packages/*`; `packages/*` never depend on
  `apps/*`; apps never import each other. Enforced in CI with dependency-cruiser, alongside the
  module boundaries from ADR-002.
- **Sharing.** `packages/contracts` is the only code shared between `api` and `web`. No generic
  `shared`/`utils` package.
- **Runtimes.** Additional runtimes of the API, such as the worker, are entrypoints inside
  `apps/api` (e.g. `src/worker.ts` next to `src/main.ts`), not separate apps. They run the same
  modules, and a separate app would have to import `apps/api`.
- **Contracts.** `contracts` is organised per bounded context (`contracts/src/<context>/`),
  mirroring the API modules; cross-cutting shapes such as problem details go in
  `contracts/src/common/`. In `apps/api`, only the `presentation` layer and HTTP plumbing in
  `shared/` import it. Presentation maps contract DTOs to use-case inputs and outputs, so the domain
  and the published contracts change independently. Enforced by dependency-cruiser.
- **Dependencies.** Declared in the workspace that uses them; the root `package.json` holds only
  repo-wide dev tooling. Internal dependencies use `workspace:*`.
- **Consumption.** Internal packages are consumed through their built output via `package.json`
  `exports`; `contracts` is compiled with `tsc` and built before the apps. No TypeScript `paths`
  aliases across package boundaries, so what works locally is what runs in the container.
- **Pinning.** The pnpm version is pinned through the `packageManager` field (Corepack).

## Alternatives considered

- **Polyrepo:** hard isolation and independent versioning, but every contract change needs
  coordinated PRs across repos plus a published package (private registry or git dependency), and
  lint, commit rules, and CI are duplicated. Independent versioning has no value when each contract
  has exactly one producer and one consumer.
- **Single-package repo:** simplest setup, but frontend and backend dependencies mix in one manifest
  and one image, the web app can't be hosted separately from the API, and splitting later means a
  disruptive restructure.
- **Worker as a separate app (`apps/worker`):** requires moving the modules into `packages/*` so
  both apps can import them, splitting each bounded context across a package boundary. Worth it only
  if the worker needs a different dependency set or release cadence.
- **Nx:** dependency graph, affected commands, caching, generators, and built-in boundary rules, at
  the cost of a large configuration and convention surface for two apps and two packages. Its
  plugins also couple NestJS and Next.js upgrades to Nx releases.
- **Turborepo (deferred, not rejected):** a thin layer over workspaces with task caching and
  `turbo prune` for Docker builds; easy to add and to remove. Not needed at the current size.

## Consequences

- A contract change updates `contracts`, `api`, and `web` in one PR, validated by one CI run. Commit
  rules and CI apply to the whole repo from one place.
- pnpm's strict `node_modules` layout prevents phantom dependencies between workspaces.
- Web and API build and deploy independently. The API image is built from the repo root with
  `pnpm deploy --filter api --prod` to get a pruned bundle without frontend dependencies.
- No task caching: by default CI builds and tests everything. Per-app workflows use `paths` filters
  covering the app, its internal dependencies, and the lockfile (e.g. `apps/api/**`,
  `packages/contracts/**`, `packages/config/**`, `pnpm-lock.yaml`), so a contract change still
  rebuilds its consumers. `pnpm --filter "...[origin/master]"` is the alternative.
- The worker ships with the API's full dependency set, since both run the same modules. Accepted.
- The lockfile is shared, so a frontend dependency bump triggers the API pipeline. Accepted at this
  scale.
- The structure is revisited only when one of these is observed:
  1. PR CI wall time consistently exceeds ~10 minutes: adopt Turborepo with caching;
  2. a third deployable app or an independently released package appears: re-evaluate Turborepo/Nx;
  3. a component needs different access control (e.g. infrastructure with secrets): extract that
     component only, not the whole repo.
