# Activepieces Architecture Blueprint

Generated: 2025-12-29
Scope: Entire Nx monorepo in this workspace (React UI, Server API, Engine, Worker, Shared libs, Pieces framework, EE modules, Docs, Deploy tooling)

## 1. Architecture Detection and Analysis

- Project type: Node.js/TypeScript Nx monorepo
- Major containers:
  - React UI (Vite + Tailwind) in `packages/react-ui`
  - Server API (Fastify) in `packages/server/api`
  - Worker (Fastify plugin runtime for queues/compute) in `packages/server/worker`
  - Engine (Flow execution runtime) in `packages/engine`
  - Shared libraries in `packages/shared`, `packages/server/shared`
  - Pieces framework and community pieces in `packages/pieces`
- Primary architectural patterns:
  - Layered architecture (UI → API → DB/Services)
  - Modular plugin architecture (Fastify modules, "pieces" plugin SDK)
  - Event-driven/async processing (workers, queues, triggers, engine-runner)
  - Monorepo governance via Nx with cached targets and generators

Guiding principles observed:
- Extensibility-first via type-safe pieces framework
- AI-first integrations and MCP server exposure
- Enterprise features (RBAC, SSO/SAML, audit, custom domains)
- Observability and resilience (OTel, Sentry, Redis locks)

## 2. Architectural Overview

- UI (React) provides flow builder, management, and admin features
- API orchestrates authentication, flows/triggers, tables/records, files, templates, projects, EE modules
- Engine executes flows and trigger hooks with a strict contract defined in `@activepieces/shared`
- Worker offloads long-running and async jobs (trigger execution, start runs)
- Data: PostgreSQL via TypeORM; Redis for locks/stores; optional S3 for files
- Integration: Pieces (npm packages) provide connectors, actions, triggers, dynamic props; MCP integrates pieces for LLM tooling

## 3. Architecture Visualization (C4 textual)

- Context: Users and integrators interact with Activepieces via Web UI or API. Activepieces interacts with 3rd-party services through pieces.
- Containers:
  - Web App (React) ↔ API (Fastify) via HTTP + WebSocket
  - API ↔ PostgreSQL (TypeORM) via TCP
  - API ↔ Redis (locks, stores, sockets adapter)
  - API ↔ S3 (file storage) optional
  - API ↔ Worker (internal via Redis/bull-like queues and HTTP callbacks)
  - API ↔ Engine (spawned/managed process, IPC/HTTP depending on runner)
- Components (examples):
  - API modules: `flows`, `trigger`, `tables`, `file`, `pieces`, `platform`, `user`, `ee/*`
  - Engine: `handler/*` (flow-executor, piece-executor, router-executor), `operations/*`, `helper/trigger-helper`
  - Worker: `consume/executors/*`, `compute/*` (engine-runner), `cache/*`
  - Shared: DTOs, schemas, enums, types for contracts between UI/API/Engine

## 4. Core Architectural Components

### React UI (`packages/react-ui`)
- Purpose: Flow builder, administration, templates, forms, tables
- Structure: Vite + Tailwind; features in `src/features/*`, app bootstrap in `src/main.tsx`
- Interaction: REST + WebSocket with API; i18n; Radix UI components
- Evolution: Feature modules can grow independently; state management adheres to feature boundaries

### Server API (`packages/server/api`)
- Purpose: HTTP/WS API surface
- Structure: Fastify server in `src/main.ts` and `src/app/server.ts`; modules registered in `src/app/app.ts`
- Key patterns:
  - Fastify plugins per domain (`flowModule`, `triggerModule`, `tablesModule`, `fileModule`, etc.)
  - Typebox schemas via `@fastify/type-provider-typebox`
  - TypeORM repositories via `repoFactory`
  - Redis connections/locks via `redis-connections.ts`
  - OpenAPI docs via `@fastify/swagger`
  - WebSocket via `fastify-socket` + Redis adapter
- Evolution: Add new modules by composing Fastify plugins with controllers/services/entities

### Engine (`packages/engine`)
- Purpose: Execute flows and triggers deterministically
- Structure: `handler/*` (executors, context), `operations/*` (flow, trigger-hook, property, metadata), `helper/*`
- Key patterns:
  - Flow executor walks actions sequentially; supports router, loop, code, piece actions
  - Trigger helper supports hook lifecycle (onStart/onEnable/onDisable/test/run)
  - Props resolution and validation across piece properties
- Evolution: New action/trigger types can be added via executors and shared contracts

### Worker (`packages/server/worker`)
- Purpose: Async job execution (trigger polling, starting runs)
- Structure: Executors under `consume/executors/*`, engine runner under `compute/*`
- Key patterns: Fetch flowVersion, call engine-runner, push progress to API, handle failures robustly
- Evolution: New job types implemented as executors with clear contracts

### Shared Libraries (`packages/shared`, `packages/server/shared`)
- Purpose: Type-safe contracts (DTOs, enums, schemas), system utilities, telemetry
- Structure: Re-exported modules in `packages/shared/src/index.ts` for common types across packages
- Key patterns: Engine operations (`EngineOperationType`, `TriggerHookType`), flow structure utils, authentication models

### Pieces Framework & Community Pieces (`packages/pieces`)
- Purpose: Plugin architecture enabling 3rd-party integrations
- Structure: Framework `pieces-framework` and many `community/*` pieces
- Key patterns: Each piece defines actions and triggers, metadata, auth; consumed by engine
- Evolution: Add new pieces as npm packages following framework contracts

## 5. Architectural Layers and Dependencies

- UI → API → Data; Engine/Worker operate as runtime services invoked by API/Worker
- Dependency rules:
  - UI never directly touches DB/Redis
  - API services depend on repoFactory (TypeORM) and system helpers; avoid cross-module domain knowledge leaks
  - Engine is isolated from HTTP surface; interacts via well-defined operations; no DB access directly
- Separation mechanisms:
  - Shared types enforce boundaries
  - Fastify plugin registration scopes routes/controllers
  - repoFactory abstracts transactional vs default repository usage

## 6. Data Architecture

- Entities (representative): User, UserIdentity, Project, Flow, FlowVersion, FlowRun, Folder, Template, Table, Field, Record, File, GitRepo, Tag, AppConnection, AIProvider
- Relationships: Project-centric multi-tenancy via `platformId` and `projectId`; indexing for performance
- Access patterns: Repositories via `repoFactory(Entity)`; paginated queries via paginator helpers
- Transformations: DTO mapping with Typebox; sample data injection for step tests
- Caching: Redis distributed store and lock for migration and concurrency control
- File storage: DB or S3 depending on system props; helper handles uploads/reads
- Validation: Typebox schemas in controllers; property processors in engine

## 7. Cross-Cutting Concerns

- Authentication & Authorization:
  - Local, OTP, Federated (SAML), managed auth; RBAC via project roles
  - Middleware: `authenticationMiddleware`, `authorizationMiddleware`, `rbacMiddleware`
- Error Handling & Resilience:
  - Global error handler; Sentry via `exceptionHandler.initializeSentry(...)`
  - Redis distributed locks; retries in worker; graceful shutdown in `main.ts`
- Logging & Monitoring:
  - OpenTelemetry via `instrumentation.ts` and `@fastify/otel`
  - Structured logging via Fastify logger
- Configuration Management:
  - `AppSystemProp` keys across modules; env-driven
  - Feature flags and EE hooks modules
- Validation:
  - Typebox schemas; engine property validators

## 8. Service Communication Patterns

- Synchronous: HTTP REST endpoints under `/v1/*`; WebSocket events for live progress
- Asynchronous: Worker queues extract payloads and start runs; engine executed via process manager
- Versioning: v1 path prefix; DTO versioning via shared types
- Discovery: Pieces metadata service; tags; platform modules
- Resilience: Trigger retries; progress updates; status tracking in runs

## 9. Technology-Specific Patterns

### Node.js / Fastify
- Middleware pipeline: CORS, rawBody, multipart, favicon, health
- Plugin modules per domain; route schemas; OpenAPI generation
- TypeORM integration with custom `repoFactory` and transactions

### React
- Feature-based organization in `src/features/*`
- i18n and Radix UI component patterns; Vite bundling

### Nx Monorepo
- Target defaults in `nx.json` with caching and inputs sets
- Generators for React; lint/test/build pipelines; Bun used for speed

## 10. Implementation Patterns

- Interface Design: Shared types in `@activepieces/shared` define contracts for engine/API/UI
- Service Implementation: API services follow clear parameters and return types; logging injected via Fastify
- Repository: `repoFactory(Entity)` to get transactional/default repository; isolated queries
- Controller/API: Fastify route registration by module; Typebox request/response schemas; prefix scoping
- Domain Model: Entities per feature folder; DTO schemas for external API

## 11. Testing Architecture

- Tools: Jest via Nx; e2e tests in `packages/tests-e2e`
- Boundaries: Unit tests for services; integration for controllers; engine tests for executor logic
- Test doubles: Mocks under `__mocks__`; cache helpers; sample data in engine

## 12. Deployment Architecture

- Containerization: Dockerfile with Bun install and Node 20; PM2 support; Nginx conf for React
- Orchestration: Helm chart in `deploy/activepieces-helm`; Pulumi infra scripts in `deploy/pulumi`
- Environments: dev/test/prod compose files; `.env` support with `AppSystemProp` keys
- Observability: OTel exporters; Sentry DSN; health endpoints

## 13. Extension and Evolution Patterns

- Feature Addition:
  - API: add a Fastify module with controller/service/entity; register in `app.ts`
  - Engine: add executor for new action/trigger; extend shared contracts
  - UI: add `src/features/<feature>` with routes/components
- Integration:
  - New piece: implement actions/triggers with framework; publish npm; sync via CLI
  - External systems: adapters through services; anti-corruption via DTOs
- Configuration: extend `AppSystemProp` and validators

## 14. Architectural Pattern Examples (concise)

- Fastify module registration (API `app.ts`):
  - Registers modules for flows, triggers, pieces, users, tables, templates, etc.
- repoFactory usage:
  - `const repo = repoFactory(Entity)` returns transactional/default repo getter
- Engine flow execution:
  - `flowExecutor.execute({ action, executionState, constants })` processes actions sequentially
- Trigger hook run:
  - `triggerHelper.executeTrigger({ params, constants })` with hook lifecycle

## 15. Architectural Decision Records (high-level)

- Monorepo via Nx for developer velocity and caching
- Fastify chosen for performance and plugin ergonomics
- TypeORM provides entity mapping and transactions; repoFactory abstracts usage
- Redis for distributed locks, stores, and socket adapter
- Engine separated from API for determinism and isolation; worker runs async
- Pieces as npm packages for community extensibility and MCP exposure

## 16. Architecture Governance

- Nx target defaults enforce caching and inputs boundaries; lint/test steps
- Commit linting; ESLint configs across packages
- OpenAPI generation validates request/response contracts
- CI configurations implied by Nx and scripts (`push` runs lint --fix)

## 17. Blueprint for New Development

- Workflow:
  - Define shared contracts in `packages/shared` if needed
  - Implement service logic in API; register routes via a new module
  - Add UI features with isolated components; consume API endpoints
  - If async or long-running, add worker executor; call engine if flow/trigger involved
- Templates:
  - Use `FastifyPluginAsyncTypebox` for controllers
  - Use `repoFactory(Entity)` for data access
  - In engine, add executor implementing `BaseExecutor<T>`
- Pitfalls:
  - Avoid cross-module direct imports of entities/services
  - Respect multi-tenancy (platformId/projectId) in queries
  - Validate inputs via Typebox and engine property validators
  - Ensure OTel/Sentry contexts are passed to logs

---

Keep this blueprint updated as modules evolve (check `packages/server/api/src/app/app.ts` for authoritative module registration and `packages/engine/src/lib/operations` for engine operations).
