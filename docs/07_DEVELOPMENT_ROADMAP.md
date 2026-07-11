## 1. Purpose of this Roadmap

A roadmap exists to answer one question before any code is written: in what order do we build this, and why that order and not another. Every one of the six documents preceding this one (01 through 06) describes a finished state: what the product is, what it does, how the data is shaped, how the API behaves, how the interface looks. None of them describe how to get from an empty repository to that finished state without wasting months on the wrong thing first. That is this document's only job.

Sequencing reduces risk in a specific, structural way for a product like this one: the most expensive mistakes to make late are the ones made in the foundation, and the most expensive mistakes to make early are the ones made in a part of the system nobody has validated yet. A workflow execution state machine (04_DATABASE_DESIGN.md, Section 8) that gets the wrong shape costs weeks to fix once fifty tables depend on it. A LinkedIn carousel formatting detail (02_PRD.md, Section 7) that gets the wrong shape costs an hour to fix regardless of when it is discovered. This roadmap is built around that asymmetry: build the things that are expensive to get wrong, and expensive to change later, first and carefully. Build the things that are cheap to get wrong last, and build them fast, because the fastest way to learn whether a LinkedIn carousel format is right is to put it in front of an account manager, not to deliberate about it in a design review.

Each phase in Section 3 exists because the phase after it has a hard dependency on it, not because of an arbitrary ordering convention. Phase 4 (Workflow Engine) comes before Phase 5 (AI Agents) because an agent has nothing to run inside of until the execution graph, checkpointing, and state machine exist to host it, not because "engine before agents" sounds like the right order in the abstract. Where a dependency is soft rather than hard, meaning two phases could technically run in parallel with enough engineers, this document says so explicitly (see the parallelization notes throughout Section 4), because pretending every dependency is hard when it is not is how roadmaps waste calendar time on a team that could have been running two workstreams at once.

This roadmap assumes a small team: two to four backend-capable engineers, one to two frontend-capable engineers, with meaningful overlap between the two given the stack's shared TypeScript surface (the OpenAPI-generated types from 05_API_DESIGN.md, Section 15, are designed specifically to make that overlap productive). It does not assume a platform team, a dedicated SRE, or a dedicated QA organization; those roles are folded into the engineering team's responsibilities throughout, and called out explicitly wherever that matters (Section 9, Section 10).

---

## 2. Development Philosophy

| Principle | What it rules out |
|---|---|
| **Build foundations first** | Starting the Workflow Builder UI before the Workflow Execution state machine exists. A screen with nothing real behind it produces false confidence and gets rebuilt. |
| **Ship vertical slices** | A phase is not "done" when its backend pieces exist in isolation. Phase 4 is not complete until a workflow can be created, executed, and observed end to end for at least one trivial goal, even before any real agent exists behind it (Section 4, Phase 4's Definition of Done uses a stub agent for exactly this reason). |
| **Build for production from day one** | Writing a throwaway prototype and rewriting it once it works. Every piece of infrastructure introduced in this roadmap (Alembic migrations from Phase 3, Row Level Security from Phase 3, structured logging from Phase 1) is the real, production version, not a prototype standing in for one. |
| **Test continuously** | Treating testing as Phase 9. Phase 9 in this roadmap is hardening and the test types that only make sense once the whole system exists (load, security, end-to-end); unit and integration tests are written inside every phase from Phase 1 onward, per that phase's own Definition of Done. |
| **Security by default** | Adding authentication, RBAC, and secret encryption as a pre-launch checklist item. Phase 2 exists in week 3, not week 20, because every phase after it needs a real identity and permission model to build against, and retrofitting RBAC onto forty existing endpoints is measurably more expensive than building fifteen endpoints against RBAC that already exists. |
| **API-first development** | Frontend engineers waiting on backend engineers to finish an endpoint before starting UI work. 05_API_DESIGN.md's OpenAPI schema is generated from route definitions the moment a router exists with real Pydantic models, even before the handler logic is complete, so frontend work against typed mocks can start in parallel (see Phase 6's parallelization note). |
| **Documentation-driven development** | Documents 01 through 06 are not written to be read once and archived. Every phase in Section 4 cites the specific section of the specific document it implements, and a pull request that diverges from that section either updates the document in the same PR or is wrong. |
| **Modular architecture** | The layered separation from 03_TECHNICAL_ARCHITECTURE.md (routers never touch the database, repositories never contain business rules) is enforced from the first endpoint in Phase 1, not introduced later once the codebase is already tangled enough to make it painful. |
| **Incremental delivery** | A 12-week silent build followed by a big-bang internal launch. Phase 11 assumes a private beta with two to three real agency partners starting the moment Phase 8 is functionally complete, so the last months of the roadmap are shaped by real usage, not by internal guessing. |
| **Human approval before automation** | This is a product principle from 01_VISION.md as much as an engineering one, and it shapes sequencing directly: the Approval Modal and the mandatory `awaiting_approval` state (02_PRD.md, Section 14's mitigation for User Trust risk) are built in Phase 4, as part of the core state machine, not added in Phase 7 as a feature. There is no version of this roadmap where publishing capability exists before approval capability does. |

---

## 3. High Level Timeline

Approximately 30 weeks (7 months) from an empty repository to production launch, with an additional 4 to 8 weeks of beta stabilization folded into the tail of Phase 11, putting real production readiness in the 8 to 9 month range the way most of this kind of build actually lands once real customer feedback is incorporated rather than assumed.

```
Week   1    3    5    7    9    11   13   15   17   19   21   23   25   27   29   31
       |----|----|----|----|----|----|----|----|----|----|----|----|----|----|----|
Ph 0   [==]
Ph 1     [====]
Ph 2          [===]
Ph 3            [====]
Ph 4                 [======]
Ph 5                        [========]
Ph 6                             [========]          (starts mid-Phase 5, parallel)
Ph 7                                     [======]
Ph 8                                           [====]
Ph 9                                                [=====]
Ph 10                                                     [====]
Ph 11                                                          [========...]
```

| Phase | Name | Duration | Primary dependency | Success criteria |
|---|---|---|---|---|
| 0 | Project Foundation | 1.5 weeks | None | A commit to `main` deploys to a live staging URL via CI with zero manual steps |
| 1 | Core Backend | 2 weeks | Phase 0 | Health check, error envelope, and one real CRUD resource (Organizations) live in staging, fully tested |
| 2 | Authentication | 1.5 weeks | Phase 1 | A real user can sign up, log in via Clerk, and hit an authenticated endpoint that correctly enforces RBAC |
| 3 | Database | 2 weeks | Phase 2 | Full schema from 04_DATABASE_DESIGN.md live via Alembic, RLS policies verified with a cross-tenant access test that fails closed |
| 4 | Workflow Engine | 3 weeks | Phase 3 | A campaign with a stub agent runs end to end through every state in the state machine, including a forced failure and retry |
| 5 | AI Agents | 4 weeks | Phase 4 | All eight agents from 05_API_DESIGN.md, Section 8, produce real output against a real goal, orchestrated, not tested in isolation only |
| 6 | Frontend | 4 weeks, overlapping Phase 5 | Phase 2 (auth), Phase 4 (API contracts) | Every page in 06_FRONTEND_DESIGN.md, Section 5, renders against the real API with real loading, empty, and error states |
| 7 | Execution Engine (integration) | 3 weeks | Phase 5, Phase 6 | The Execution View (06_FRONTEND_DESIGN.md, Section 9) reflects a real running campaign live via WebSocket, with working pause, resume, retry, and approval |
| 8 | Publishing | 2 weeks | Phase 7 | An approved campaign publishes to a real LinkedIn account on schedule and ingests real analytics afterward |
| 9 | Testing | 2.5 weeks | Phase 8 | Load test sustains the Phase 12 100-user target with no degraded p95 latency; security review closes with no unresolved high-severity finding |
| 10 | Deployment | 2 weeks | Phase 9 | Staging and production environments are both live, blue/green deploy has been exercised at least once, rollback has been exercised at least once |
| 11 | Production Launch | 4+ weeks | Phase 10 | Two to three paying or design-partner agencies running real campaigns, with a live feedback loop feeding the Version 2 backlog |

---

## 4. Phase-by-Phase Development Plan

### Phase 0: Project Foundation (1.5 weeks)

**Purpose**: everything after this phase assumes a working deploy pipeline exists. Building that pipeline after there is already product code to migrate onto it is a common, avoidable source of lost weeks.

**Goals**: a monorepo (or two coordinated repos, backend and frontend) with CI running on every push; a working local `docker-compose` environment (Postgres, Redis, Qdrant) matching the Technical Architecture document's local development story; staging infrastructure provisioned, even though nothing product-specific is deployed to it yet.

**Architecture decisions**: monorepo versus polyrepo, decided here rather than revisited later, since splitting a monorepo later is mechanical but tedious, and merging two polyrepos later is worse. Recommendation, consistent with the Technical Architecture document's modular-monolith stance: one repo, `backend/` and `frontend/` as top-level directories, one CI pipeline with path-based job triggers, so a frontend-only change does not run the backend test suite and vice versa.

**Tasks**: repository scaffolding; `docker-compose.yml`; GitHub Actions workflow skeleton (lint, type-check, test, build, deploy-to-staging on merge to `main`); base FastAPI app with a single `/health` route; base Next.js app with the route groups from 06_FRONTEND_DESIGN.md, Section 11, stubbed but empty; environment variable strategy and secret management tool selected (see Phase 10 for the production version of this).

**Milestones**: CI green on an empty commit; staging URL live and reachable; `docker-compose up` produces a working local environment for a new engineer in under 15 minutes, timed and documented, since onboarding friction compounds across every engineer who joins later.

**Risks**: over-investing in tooling perfection here. This phase has a hard time-box specifically because foundation work has no natural stopping point, an engineer can always find one more CI optimization; the definition of done below exists to force a stop.

**Dependencies**: none.

**Deliverables**: working CI/CD to staging, local dev environment, empty but structured repos.

**Definition of Done**: a new engineer can clone the repo, run one documented command, and have a working local environment; a merged PR deploys to staging without a human touching a server.

### Phase 1: Core Backend (2 weeks)

**Purpose**: establish the layered architecture (03_TECHNICAL_ARCHITECTURE.md's Presentation/Business/Service/Repository split) with one real resource end to end, so every phase after this one is extending a proven pattern rather than inventing it under deadline pressure with a real feature at stake.

**Goals**: the Organizations resource (05_API_DESIGN.md, Section 6.2), fully implemented across all four layers, with tests at each layer; the standard error envelope (05_API_DESIGN.md, Section 11) implemented as a global FastAPI exception handler; structured logging with request IDs wired through every layer.

**Architecture decisions**: ORM chosen and committed (SQLAlchemy 2.0, per the Database Design document's migration tooling section); the Repository Layer's interface shape decided here (one repository class per aggregate root, matching the pattern that will repeat for every resource through Phase 5).

**Tasks**: SQLAlchemy models for `Organizations` and `Users`; the four-layer implementation of `POST/GET/PATCH /organizations`; global exception handler and error envelope; request ID middleware; base Pydantic schema conventions (request/response model naming, matching Section 20 conventions that will apply for the rest of the backend).

**Milestones**: one resource, fully tested, code-reviewed, and treated as the template every future resource copies.

**Risks**: getting the layering wrong here is the single most expensive mistake in the entire backend roadmap, since every one of the next twenty-plus resources copies this pattern. This phase gets a full design review before implementation, not just before merge.

**Dependencies**: Phase 0.

**Deliverables**: Organizations resource live in staging; documented "how to add a new resource" pattern other engineers can follow without re-deriving the architecture.

**Definition of Done**: Organizations CRUD passes integration tests hitting a real (test) Postgres instance, not mocked; the four-layer pattern is written up as a one-page internal doc other engineers reference for every subsequent resource.

### Phase 2: Authentication (1.5 weeks)

**Purpose**: every phase from here forward needs a real identity to build against; Phase 2 exists this early specifically to avoid retrofitting auth onto a growing set of unauthenticated endpoints.

**Goals**: Clerk integration live; JWT validation dependency (05_API_DESIGN.md, Section 3) implemented and applied globally; RBAC role checks implemented as the `require_role` dependency described in the same section; Row Level Security policies stubbed (full policies land in Phase 3 once the full schema exists, but the RLS mechanism itself, tied to the JWT's organization claim, is proven here).

**Tasks**: Clerk webhook handler syncing `Users` on signup; `get_current_user` dependency; `require_role` dependency; the `X-Organization-Id` header-scoping mechanism from the API design; a cross-tenant access test proving a user cannot read another organization's Organizations row even with a manually crafted request.

**Milestones**: a real signup-to-authenticated-request flow works end to end against staging.

**Risks**: RBAC bugs are the highest-severity class of bug this phase can introduce, and they are easy to miss in normal testing since they only manifest with a second tenant. The cross-tenant test above is treated as a release blocker, not a nice-to-have.

**Dependencies**: Phase 1.

**Deliverables**: working authentication and authorization, provable with an automated cross-tenant test in CI, not just manual verification.

**Definition of Done**: the cross-tenant access test runs in CI on every subsequent PR touching an authenticated route, not just once in this phase.

### Phase 3: Database (2 weeks)

**Purpose**: the full schema from 04_DATABASE_DESIGN.md needs to exist before the Workflow Engine (Phase 4) has anywhere to persist state, and getting the schema wrong here is expensive to fix once real campaign data exists on top of it.

**Goals**: every entity from the Database Design document's domains (Identity, Brand and Context, Workflow and Execution, Content and Output, Publishing and Analytics, System and Audit) implemented as Alembic migrations; RLS policies applied to every table with an `organization_id` column; indexing strategy (composite indexes, GIN indexes on JSONB, full-text indexes) applied per the Database Design document's Section 6, not deferred to a later performance pass.

**Architecture decisions**: UUIDv7 generation strategy (application-side versus a Postgres extension) decided and applied consistently; the soft-delete convention (`deleted_at`, globally scoped queries) implemented once as a SQLAlchemy mixin rather than per-model, so it cannot be forgotten on a future table.

**Tasks**: full model definitions across all six domains; Alembic migration chain; RLS policy SQL, tested per table; seed data script for local development; the `Workflow Definitions` table stubbed even though it is not populated until the dynamic-DAG future phase (04_DATABASE_DESIGN.md, Section 13), so the schema does not need a breaking migration later.

**Milestones**: full schema live in staging; every table has a passing RLS test.

**Risks**: schema churn after this phase is the most expensive kind of rework in the roadmap. This phase includes an explicit schema review against every one of the previous documents, particularly 05_API_DESIGN.md's Core API Modules, since every module in that document implies a table shape the schema must actually support.

**Dependencies**: Phase 2 (RLS depends on the JWT organization claim).

**Deliverables**: complete, migrated, indexed, RLS-protected schema.

**Definition of Done**: every table in the Database Design document exists in staging with a passing migration and a passing RLS isolation test; the soft-delete mixin is applied with no manual per-model exceptions.

### Phase 4: Workflow Engine (3 weeks)

**Purpose**: this is the architectural core of the product, the deterministic execution graph, checkpointing, and state machine that every agent in Phase 5 will run inside of. It is built and proven with a stub agent before any real, expensive-to-iterate-on AI logic touches it, so state machine bugs are found against a trivial agent, not a slow, costly real one.

**Goals**: the `Workflow Executions` and `Workflow Steps` state machine (04_DATABASE_DESIGN.md, Section 8) implemented via Celery, with checkpointing after every step; the `pending / researching / strategizing / generating_content / awaiting_approval / publishing / completed / failed / cancelled` state transitions all implemented and covered by tests, including the failure and retry path; the risk-tiered tool registry concept (read-only, reversible, irreversible) scaffolded, even though only stub tools exist yet.

**Architecture decisions**: this phase is where the Celery-versus-Temporal evaluation referenced in the earlier architecture analysis gets made concretely, not deferred. Recommendation for Version 1, given the small team and the 30-week timeline: ship on Celery and Redis, since the added operational surface of introducing Temporal this early is not worth paying before there is a single real workflow in production; revisit in Phase 12 (Scaling Roadmap) once real execution volume exists to justify it.

**Tasks**: `POST /campaigns` enqueuing a workflow execution; a stub agent that sleeps briefly and writes a fixed output, standing in for every real agent until Phase 5; checkpoint-and-resume logic, tested by killing a worker mid-execution and confirming resume from the last checkpoint rather than a restart; the pause/resume/cancel/retry endpoints from 05_API_DESIGN.md, Section 7; the Approval endpoint and the mandatory human checkpoint, built now as a core state transition, not bolted on later.

**Milestones**: a campaign with the stub agent runs from `pending` to `completed` through every intermediate state, including a deliberately forced failure that recovers via the retry endpoint.

**Risks**: the temptation to build this phase and Phase 5 together, since a real agent "isn't much more work" than a stub. Resist this; the entire value of testing the state machine against a stub is that state machine bugs surface in seconds per test run, not minutes, and the phase boundary is what prevents agent development from starting on an unproven foundation.

**Dependencies**: Phase 3.

**Deliverables**: a fully functional, agent-agnostic execution engine, provable with a stub agent.

**Definition of Done**: every state transition in the state machine has an automated test, including the checkpoint-and-resume path under a simulated worker crash.

### Phase 5: AI Agents (4 weeks)

**Purpose**: with a proven execution engine to run inside of, this phase replaces the stub agent with the eight real agents from 05_API_DESIGN.md, Section 8, one at a time, so a bug in the Copywriter Agent is diagnosed against a known-good engine, not conflated with a possible engine bug.

**Goals**: Research, Strategy, Copywriter, Carousel, Image Prompt, Validator, Publisher (stubbed until Phase 8's real LinkedIn integration), and Analytics (stubbed until Phase 8) agents implemented against their documented input/output contracts; prompt templates versioned per 04_DATABASE_DESIGN.md's `Prompt Templates` entity from the start, not added retroactively; the Qdrant-backed semantic memory (brand voice retrieval) wired into the Copywriter and Validator agents specifically, since those are the two agents whose output quality depends most directly on it.

**A note on naming, for consistency with all prior documents**: this phase implements a Planner as part of the combined Interpreter and Planner service described in the architecture documents, not as a ninth standalone orchestrated agent, and implements Memory as the shared substrate every agent reads from and writes to, not as an agent that appears in the Execution Timeline. Both are real, first-class engineering work in this phase; neither is a separate row in the agent table in 05_API_DESIGN.md, Section 8, and this roadmap keeps that distinction rather than introducing a naming inconsistency the earlier documents do not have.

**Architecture decisions**: agent development order within the phase follows the data dependency chain, not alphabetical or arbitrary order: Research first (nothing downstream can run without a Context Document), then Strategy, then Copywriter and Carousel and Image Prompt in parallel (three engineers can genuinely work these simultaneously, since they share an input contract but not implementation), then Validator (needs real content from the previous three to validate against), then stub Publisher and Analytics (full implementations wait for Phase 8's real LinkedIn credentials).

**Tasks**: per-agent prompt engineering and evaluation harness (a small, curated golden set of goals with expected-quality output, checked on every prompt change, not just at ship time); per-agent unit tests using recorded/mocked LLM responses for fast, deterministic CI runs, separate from a smaller suite of live-LLM integration tests run less frequently given their cost and latency.

**Milestones**: each agent passes its evaluation harness independently before orchestration begins; the full eight-agent chain (with stub Publisher/Analytics) produces a real, human-reviewable campaign from a real goal.

**Risks**: prompt quality is inherently harder to gate in CI than code correctness. The evaluation harness is the mitigation, not a guarantee; this phase budgets explicit time for manual review of the first several dozen real outputs by someone with marketing judgment, not just engineering judgment, before calling any agent done.

**Dependencies**: Phase 4.

**Deliverables**: all eight agents, individually evaluated and orchestrated together against the proven execution engine.

**Definition of Done**: a real goal, submitted by a non-engineer team member, produces a full campaign draft that the Validator agent passes and a human reviewer finds genuinely usable without major rewriting.

### Phase 6: Frontend (4 weeks, overlapping the back half of Phase 5)

**Purpose**: the frontend has a real, typed API contract to build against from the moment Phase 1's OpenAPI schema exists, so it does not need to wait for Phase 5 to finish; this phase starts in parallel once Phase 4's endpoints (workflow status, campaigns) are stable, even before Phase 5's real agent output exists, using the stub agent's output as realistic-enough placeholder data.

**Goals**: the application shell (06_FRONTEND_DESIGN.md, Section 4); every page in Section 5 implemented against real API calls, not static mocks, by the end of this phase; the six Zustand stores and the React Query integration from Section 10; dark mode as the shipped default per the design system.

**Architecture decisions**: build order within the phase follows the same "foundation before feature" logic as the roadmap overall: design tokens and primitives (Section 3) first, then the application shell and routing (Sections 4, 11), then Dashboard and Projects (the simplest data-bound pages, good for proving the React Query and typed-API pattern), then the Workflow Builder and Execution View last, since they are the most complex pages and benefit from every pattern below them being proven first.

**Tasks**: Tailwind theme configuration matching Section 3's tokens; shadcn/ui primitive installation and the minimal customization described in Section 20; the WebSocket hook (`use-workflow-stream.ts`) built and tested against Phase 4's execution engine directly, since this is the one piece of frontend work with a hard, non-negotiable dependency on backend behavior rather than just an API contract; every page's loading, empty, and error states, per the Definition of Done in Section 20 of the frontend document, built alongside the happy path, not after it.

**Milestones**: Dashboard and Projects fully functional against real staging data; Workflow Builder produces a real campaign; Execution View reflects a real running campaign, live.

**Risks**: building against the stub agent's placeholder output risks the Execution View looking correct with fake data but breaking against real agent output shapes once Phase 5 finishes; mitigated by a short, explicit integration checkpoint in the last week of this phase, after Phase 5's real agents exist, specifically to re-verify every page against real data before Phase 6 is called done.

**Dependencies**: Phase 2 (for auth), Phase 4 (for stable API contracts); soft dependency on Phase 5 for final data-shape verification.

**Deliverables**: the full frontend application, live in staging, functional against the real backend.

**Definition of Done**: every page in 06_FRONTEND_DESIGN.md, Section 5, is reachable, functional, and has been manually verified against real (not mocked) staging data including at least one real failure state per page.

### Phase 7: Execution Engine Integration (3 weeks)

**Purpose**: Phases 4 through 6 each proved their own layer independently; this phase proves the full stack together under conditions closer to real usage, specifically the interaction between live WebSocket updates, the approval flow, and human-in-the-loop timing, which is difficult to fully exercise until all three layers are complete and connected.

**Goals**: the full goal-to-approval-to-publish-ready loop working end to end through the real UI, with no stubbed layer remaining except LinkedIn publishing itself (Phase 8); pause, resume, cancel, and retry all exercised through the actual Execution View, not just via direct API calls.

**Tasks**: closing integration gaps found in the Phase 6 checkpoint; performance pass on the WebSocket connection under multiple concurrent campaigns per organization, since this is the first phase where that scenario is realistically testable; the "request changes" loop (02_PRD.md's User Journey step 3) fully wired, sending a campaign back through content generation with reviewer notes attached.

**Milestones**: an internal, non-engineering team member runs a full campaign start to finish, unassisted, using only the UI.

**Risks**: this is the phase most likely to surface UX friction that was invisible on paper; budget is intentionally included for UI adjustments discovered here, not just bug fixes, since some findings will be "this works but is confusing," not "this is broken."

**Dependencies**: Phase 5, Phase 6.

**Deliverables**: a fully integrated product, missing only real publishing.

**Definition of Done**: three consecutive full campaign runs, executed by three different internal, non-engineering testers, complete without an engineer intervening.

### Phase 8: Publishing (2 weeks)

**Purpose**: publishing is deliberately the last piece of the core loop to go live, since it is the one irreversible action in the entire product (per the risk-tiered autonomy model in the architecture documents), and it is far cheaper to validate everything upstream of it with a stub than to debug publishing issues while also debugging upstream issues.

**Goals**: real LinkedIn OAuth (05_API_DESIGN.md, Section 6.19); the Publisher agent's real implementation, including the token-expiry notification path; the Analytics agent's real implementation, ingesting real LinkedIn engagement data on a schedule.

**Tasks**: LinkedIn API integration and its adapter-pattern isolation (02_PRD.md, Section 14's mitigation for API volatility risk); scheduled publishing via Celery Beat or equivalent; the analytics ingestion job; encryption at rest for the OAuth token, verified, not assumed.

**Milestones**: a real post, generated by the real pipeline, approved by a real human, appears on a real LinkedIn account at the scheduled time.

**Risks**: LinkedIn's API and rate limits are the one dependency in this entire roadmap outside the team's control; this phase includes explicit buffer for API quirks discovered only in integration, not present in documentation.

**Dependencies**: Phase 7.

**Deliverables**: fully functional, real publishing and analytics.

**Definition of Done**: an end-to-end campaign, from typed goal to a live LinkedIn post to an ingested analytics report, completes with no manual intervention beyond the required human approval step.

### Phase 9: Testing (2.5 weeks)

**Purpose**: this phase exists for the test types that genuinely require a complete system and cannot be meaningfully run earlier: load, security, and end-to-end tests that span every layer built in Phases 0 through 8.

**Goals and tasks**: covered in full in Section 9 of this document, applied here as a phase rather than described again.

**Milestones**: load test passes at the Phase 12 100-user target; independent security review (internal or contracted) closes with no unresolved high-severity finding, particularly around the prompt injection and tenant isolation risks named throughout the architecture documents.

**Risks**: discovering a fundamental issue this late is expensive; this is mitigated by the fact that this roadmap has already tested continuously per phase (Section 2's principle), so Phase 9 is closing gaps and testing emergent, whole-system behavior, not writing the first tests for any given feature.

**Dependencies**: Phase 8.

**Deliverables**: a production-readiness signoff.

**Definition of Done**: every item in Section 9's test matrix has run at least once against a staging environment matching production configuration.

### Phase 10: Deployment (2 weeks)

**Purpose**: production infrastructure, as distinct from the staging infrastructure that has existed since Phase 0, needs its own explicit phase because production has requirements (blue/green deploy, monitoring, alerting, backup verification) that staging deliberately does not, and conflating the two environments' readiness is a common way teams discover a gap during an actual incident instead of before one.

**Goals and tasks**: covered in full in Section 10 of this document.

**Milestones**: a full deploy-rollback cycle has been exercised at least once against production infrastructure before real user traffic exists on it.

**Dependencies**: Phase 9.

**Deliverables**: a live, monitored, production environment.

**Definition of Done**: an on-call engineer (even if that is a rotating role across a two-person team at this stage) can find and resolve a simulated incident using only the monitoring and alerting built in this phase, with no undocumented tribal knowledge required.

### Phase 11: Production Launch (4+ weeks, ongoing)

**Purpose**: the roadmap's own Development Philosophy (Section 2) commits to incremental delivery shaped by real usage; this phase is where that commitment is honored rather than treated as a slogan.

**Goals**: two to three design-partner agencies onboarded and running real campaigns; a live feedback channel (structured, not just informal) feeding directly into a Version 2 backlog; the Node Acceptance Rate and Zero-Touch Rate metrics from 01_VISION.md and 02_PRD.md tracked from the first real campaign onward, not retrofitted once someone asks for them.

**Tasks**: partner onboarding; a lightweight in-app feedback mechanism; a weekly review of real Validator flag rates and human edit rates, feeding directly into the prompt evaluation harness from Phase 5, since this is the first point in the roadmap where real usage data, not the curated golden set, is available to improve prompt quality against.

**Milestones**: first real, human-approved, real-money-relevant campaign published for a design partner; first Version 2 backlog item sourced directly from partner feedback rather than internal speculation.

**Dependencies**: Phase 10.

**Deliverables**: a live product with real users and a real feedback loop.

**Definition of Done**: this phase does not have a traditional Definition of Done, since production launch is the beginning of an ongoing process, not a discrete deliverable; it is considered stable once the Success Metrics in Section 14 have at least two consecutive weeks of real data to report against.

---

## 5. Backend Development Roadmap

This section sequences the technical components within the phases above at a finer grain than Section 4 covers, specifically for backend engineers planning individual sprints.

| Order | Component | Built in | Why this position |
|---|---|---|---|
| 1 | FastAPI app skeleton, routing, exception handling | Phase 1 | Nothing else has anywhere to live without it |
| 2 | SQLAlchemy models and Alembic | Phase 1 to 3 | Started with the first resource in Phase 1, expanded to the full schema in Phase 3, never introduced as a big-bang migration |
| 3 | Redis | Phase 1 (caching), Phase 4 (Celery broker) | Introduced early for its simpler use (rate limit counters, Section 12 of the API design) before its more complex use (task queuing) |
| 4 | Authentication and RBAC | Phase 2 | Before any business-logic resource beyond Organizations exists, per Section 2's security-by-default principle |
| 5 | Celery | Phase 4 | The Workflow Engine is Celery's first real consumer; introducing it earlier would have nothing meaningful to queue |
| 6 | Qdrant | Phase 5 | Its only consumers (Copywriter and Validator agents' brand voice retrieval) do not exist before this phase |
| 7 | LangGraph | Phase 4 to 5 | The execution graph shape is proven with the Phase 4 stub agent using LangGraph's state graph primitives directly, then populated with real agent logic in Phase 5, rather than introduced fresh in Phase 5 |
| 8 | OpenAI/Anthropic integration | Phase 5 | Agent implementation is this integration's only purpose in Version 1 |
| 9 | Workflow Engine (state machine, checkpointing) | Phase 4 | Covered fully in Section 4 |
| 10 | Scheduler (Celery Beat or equivalent) | Phase 8 | Its only Version 1 consumer is scheduled publishing |
| 11 | Memory system (episodic, semantic, procedural) | Phase 5, extended in Phase 11 | Episodic and semantic memory are needed for agent context from Phase 5 onward; procedural memory (template promotion) is genuinely a Phase 11-and-beyond feature, since it requires real repeated successful executions to promote from, which do not exist until real usage does |
| 12 | Publishing system | Phase 8 | Covered fully in Section 4 |
| 13 | Analytics ingestion | Phase 8 | Depends on Publishing existing first, since there is nothing to measure the analytics of otherwise |
| 14 | Notifications | Phase 6 to 8, incremental | The notification data model exists from Phase 3, but real triggers are added incrementally as the events that should fire them (approval needed in Phase 4, token expiry in Phase 8) come online |
| 15 | Admin APIs (prompt templates, feature flags) | Phase 5, extended through launch | Prompt template versioning is needed the moment real prompt iteration starts in Phase 5; feature flags are added as specific rollout needs arise rather than built speculatively ahead of any flag to control |

---

## 6. Frontend Development Roadmap

| Order | Component | Built in | Why this position |
|---|---|---|---|
| 1 | Design tokens, Tailwind config | Phase 6, week 1 | Every component built after this depends on the token layer existing; changing it later means restyling everything downstream |
| 2 | shadcn/ui primitives | Phase 6, week 1 | Same reasoning; primitives are the base every domain component composes |
| 3 | Authentication (Clerk integration, protected routes, middleware) | Phase 6, week 1, parallel with tokens | Nothing behind the `(app)` route group can be built or demoed without it |
| 4 | Application shell (Top Nav, Sidebar, Command Palette) | Phase 6, week 1 to 2 | Every page from here forward is built inside this shell, not before it |
| 5 | Dashboard | Phase 6, week 2 | The simplest data-bound page, deliberately chosen to prove the React Query and typed-API pattern before the harder pages use it |
| 6 | Workflow Builder | Phase 6, week 2 to 3 | Depends on the Campaigns API being stable (Phase 4), and is simpler than the Execution View it feeds into |
| 7 | Execution View, realtime updates | Phase 6, week 3 to 4, hardened in Phase 7 | The most complex page in the application; built last among the core pages deliberately, once the WebSocket pattern and every simpler page's data-fetching pattern is proven |
| 8 | Campaign Detail, Reports, Charts | Phase 6, week 4 | Depend on real Generated Content and Analytics data shapes, which are most reliably available once Phase 5's real agents exist, hence positioned at the tail of Phase 6 |
| 9 | Settings, Team, Billing | Phase 6, week 4, can slip into Phase 7 without blocking anything | Lowest-risk, most conventional CRUD pages in the application; deliberately scheduled last within the phase since they have the least dependency risk if timeline pressure requires trimming |
| 10 | Dark mode | Built alongside tokens in week 1, not as a separate pass | Dark-first is a Section 3 design system requirement, not a feature added after a light-mode-only build |
| 11 | Accessibility | Built into every component from its first implementation, verified in Phase 9 | Retrofitting ARIA roles and focus management onto a finished component library is dramatically more expensive than building them in from the first component |

---

## 7. AI Development Roadmap

Each agent is built and evaluated independently against its documented contract (05_API_DESIGN.md, Section 8) before orchestration begins, per Phase 5's architecture decision in Section 4. The build order follows the data dependency chain of a real campaign, not an arbitrary list:

1. **Research Agent** first. Every other agent's input either directly or indirectly depends on the Context Document this agent produces.
2. **Strategy Agent** second. Depends on Research's output; nothing about Strategy can be meaningfully evaluated with a fake Context Document.
3. **Copywriter, Carousel, and Image Prompt Agents**, built in parallel by up to three engineers once Strategy's Campaign Architecture output is stable, since all three share that same input contract but have independent implementations and independent evaluation harnesses.
4. **Validator Agent** fourth. Requires real content from step 3 to validate against; building it earlier would mean testing it against synthetic content that does not represent real failure modes.
5. **Publisher and Analytics Agents**, stubbed in Phase 5 to unblock orchestration and frontend work, fully implemented in Phase 8 once real LinkedIn credentials and a real publishing target exist.
6. **The Planner**, developed as part of the Interpreter and Planner service throughout Phase 5, not as a discrete numbered step, since its typed-plan-schema output (the architecture's "LLM proposes, code disposes" pattern) needs to be co-designed with the Workflow Engine's state machine, which was already built in Phase 4. In practice this means the Planner's schema is drafted in Phase 4 alongside the state machine and its actual LLM-driven implementation is completed at the start of Phase 5.
7. **Memory**, developed incrementally: the episodic and semantic layers are functional infrastructure by the end of Phase 5, since the Copywriter and Validator agents depend on semantic (brand voice) retrieval to function at all; the procedural layer (template promotion from repeated successful executions) is real Phase 11-and-beyond work, as noted in Section 5, because it requires production usage history that does not exist before then.

**Independent development before orchestration**: every agent above ships with its own evaluation harness (a golden set of realistic goals with expected-quality output, checked automatically on every prompt change) before it is wired into the full orchestrated chain. This exists specifically so a regression is attributable to one agent's prompt change, not diagnosed by re-running the entire eight-agent pipeline and guessing which step degraded.

---

## 8. Infrastructure Roadmap

| Component | Phase | Notes |
|---|---|---|
| Docker | Phase 0 | Base images for FastAPI and Next.js, multi-stage builds to keep production images lean |
| Docker Compose | Phase 0 | Local development only; production uses the deployment platform's own orchestration, not Compose directly |
| CI/CD (GitHub Actions) | Phase 0, extended through Phase 10 | Lint/type-check/test/build from Phase 0; deploy-to-production with the blue/green mechanics added in Phase 10 |
| Monitoring | Phase 1 (basic), Phase 10 (full) | Structured logs and request IDs from Phase 1; full dashboards (latency, error rate, queue depth, LLM cost) assembled in Phase 10 once every component they monitor exists |
| Logging | Phase 1 | Structured JSON logs from the first endpoint, shipped to a centralized log store from Phase 10 onward |
| Secrets Management | Phase 0 (mechanism chosen), Phase 2 (first real secrets), Phase 10 (production hardening) | The deployment platform's native secret manager, never environment files committed to source control, from the very first secret onward |
| Backups | Phase 3 (mechanism verified), Phase 10 (restore drill) | Supabase-managed daily backups and WAL archiving exist from the moment the schema does; the restore drill, actually restoring to a point in time and verifying data integrity, happens explicitly in Phase 10, since an untested backup is not a real backup |
| Cloud Storage | Phase 6 (media assets), Phase 5 (knowledge base documents) | Supabase Storage, tenant-namespaced paths per the Database Design document, from the first upload feature |
| Reverse Proxy / HTTPS | Phase 0 (staging), Phase 10 (production, custom domain) | Staging can use the platform's default TLS termination; production gets a custom domain and certificate management in Phase 10 |
| Domain | Phase 10 | Deliberately not acquired earlier than necessary, since domain and DNS setup has no dependency on anything built before Phase 10 |
| Production Environment | Phase 10 | Fully covered in Section 10 |

---

## 9. Testing Strategy

| Test type | Scope | Tooling | When it runs | Owner |
|---|---|---|---|---|
| Unit | Individual functions, Service Layer logic, Repository Layer queries | pytest (backend), Vitest (frontend) | Every PR, every phase from Phase 1 onward | Feature engineer |
| Integration | API endpoint through to a real test database | pytest with a test-scoped Postgres instance | Every PR touching an endpoint | Feature engineer |
| API (contract) | Request/response shape against the OpenAPI schema | Schemathesis or an equivalent contract-testing tool | CI, every PR | Backend engineer |
| Frontend (component) | Individual components in isolation | React Testing Library | Every PR touching a component | Feature engineer |
| Agent testing | Individual agent input/output contract | pytest with recorded/mocked LLM responses for speed; a smaller live-LLM suite for realism | Recorded suite on every PR; live suite nightly | AI-focused engineer |
| Prompt testing | Output quality against the golden evaluation set from Section 7 | Custom harness scoring against expected-quality rubrics | On every prompt template change | AI-focused engineer |
| End-to-end | Full user journeys through the real UI against a real staging backend | Playwright | Nightly from Phase 7 onward; every PR to `main` from Phase 9 onward | Whole team, rotating ownership |
| Load | Concurrent campaign execution, API latency under load | k6 or Locust, targeting the Phase 12 user-count milestones | Phase 9, then before every major scaling milestone | Backend engineer |
| Security | RBAC boundaries, RLS isolation, prompt injection resistance, OWASP Top 10 | Manual review plus automated scanning (e.g. OWASP ZAP for the API surface) | Phase 9, then before every production release touching auth or tool access | Security-focused engineer or external reviewer |
| User Acceptance | Real campaign execution by non-engineering testers | Structured test scripts, manual | End of Phase 7, repeated in Phase 9 | Product/design, with engineering support |

The recorded-versus-live split for agent testing exists because live LLM calls are slow and non-deterministic enough to make them a poor fit for a fast CI feedback loop, but skipping live testing entirely risks the recorded fixtures drifting from real model behavior; the nightly live suite is the check against that drift.

---

## 10. Deployment Roadmap

**Environments**:

| Environment | Purpose | Deploy trigger |
|---|---|---|
| Development | Local, per-engineer | `docker-compose up` |
| Staging | Integration testing, demo, the environment every phase's Definition of Done is verified against | Every merge to `main` |
| Production | Real users | Manual promotion from a verified staging build, per the blue/green process below |

**Blue/Green deployment**: two full production environments (blue and green), only one live behind the load balancer at a time. A release deploys to the idle environment, runs an automated smoke test suite against it directly, and only then is traffic cut over. This is chosen over a rolling deploy specifically because the Workflow Engine has long-running Celery tasks; a rolling deploy risks killing a worker mid-execution in a way blue/green's clean cutover avoids, since the outgoing environment can finish draining its in-flight tasks before being decommissioned.

**Rollback**: cutting traffic back to the previous environment, which remains warm (not torn down) for a defined window after every deploy specifically to make rollback a traffic-routing decision, not a redeploy, keeping rollback time in the range of seconds rather than the minutes a fresh deploy would take.

**Database migrations**: run as a separate step before traffic cutover, using the forward-and-backward-compatible migration discipline from the Database Design document, Section 14. A migration that would break the currently-live environment is not deployed until that environment has been fully cut over away from, avoiding any window where old code runs against new schema or new code runs against old schema.

**Monitoring and alerting**: the dashboards from Section 8, with alert thresholds defined explicitly in this phase rather than left as "we'll notice if something breaks": API p95 latency, workflow failure rate, Celery queue depth, and LLM spend rate all get a defined threshold and an on-call notification path before Phase 10 is considered complete.

---

## 11. Security Roadmap

This roadmap does not repeat 05_API_DESIGN.md, Section 13's security architecture in full; it sequences when each control goes live.

| Control | Phase | Note |
|---|---|---|
| Authentication (Clerk, JWT validation) | Phase 2 | Foundational, nothing else in this table matters without it |
| Authorization (RBAC, RLS) | Phase 2 to 3 | Application-layer RBAC in Phase 2, database-layer RLS once the full schema exists in Phase 3 |
| Secrets and encryption at rest | Phase 2 (mechanism), Phase 8 (first real sensitive secret, the LinkedIn OAuth token) | AES-256-GCM per the API design; the encryption key itself never touches application code or the database |
| Rate limiting | Phase 4 | Introduced alongside the Workflow Engine specifically because campaign creation and agent invocation, the two most expensive rate-limited actions, do not exist before this phase |
| Audit logs | Phase 3 (schema), Phase 4 to 8 (real events populate it incrementally as the actions worth auditing come online) | |
| Prompt injection protection | Phase 5 | Delimited data fields and the independently-prompted Validator agent as a second check, built as part of every agent's implementation from the start, not added afterward |
| LLM-specific security (output filtering, jailbreak resistance) | Phase 5, hardened in Phase 9 | Baseline guardrails in Phase 5; adversarial testing specifically targeting these controls happens in Phase 9's security review |
| OWASP Top 10 review | Phase 9 | The formal, whole-system review; individual OWASP concerns (injection, broken auth, sensitive data exposure) are addressed per-phase as each relevant surface is built, per the security-by-default principle in Section 2 |

---

## 12. Scaling Roadmap

| Scale | Architectural posture | What changes from the current phase's baseline |
|---|---|---|
| 100 users | The Phase 0 to 11 baseline exactly as built: single Postgres instance, Redis, a handful of Celery workers | Nothing; this is the target the roadmap is built to hit at launch, not a future milestone |
| 1,000 users | Worker autoscaling, queue partitioning so one large agency's campaign volume cannot starve smaller organizations, dashboard-level monitoring on stuck or runaway jobs | Infrastructure configuration changes; no architectural rewrite |
| 10,000 users | Read replicas for analytics and reporting queries, Qdrant given a real sharding or managed-service strategy rather than a single instance, the Celery-versus-Temporal decision from Phase 4 revisited with real execution volume data to justify the migration cost this time | The first point in the roadmap where a genuine architectural evolution, not just infrastructure scaling, is likely justified |
| 100,000 users | Durable-execution engine migration (if not already done at 10,000), cost-control tooling promoted from an internal dashboard to a first-class product feature (per-workflow cost caps, since a runaway agent loop is a real financial event at this volume), horizontal database partitioning by organization ID, the partition key the schema was deliberately designed around from Phase 3 |

This table deliberately does not prescribe a rewrite at any point. Every step is evolution of the Phase 0 to 11 architecture, made possible specifically because Phase 3's schema design (composite indexes, the `organization_id` partition key candidate) and Phase 4's Celery-with-a-documented-Temporal-alternative decision were both made with this table in mind from the start, not discovered as a scaling emergency later.

---

## 13. Risks

| Risk | Category | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| Agent output quality is inconsistent enough to require heavy human editing, undermining the Zero-Touch Rate target | AI | Medium-high | High | The evaluation harness from Phase 5, plus the explicit manual-review budget in that phase's risk note; the Node Acceptance Rate metric (Section 14) is tracked from the first real campaign specifically to catch this early rather than at launch |
| LinkedIn API changes or rate-limits the integration | Infrastructure | Medium | Medium | Adapter-pattern isolation (Phase 8), buffer time budgeted explicitly in that phase |
| RBAC or RLS gap allows cross-tenant data access | Security | Low, if Phase 2 and 3's testing discipline holds | Severe | Automated cross-tenant tests as a release-blocking CI check from Phase 2 onward, not a one-time manual verification |
| Prompt injection via research content or uploaded documents | Security | Medium | High | Delimited data fields, independent Validator review, and the architectural rule that Publisher only acts on content that passed the Approvals endpoint, all built in from Phase 5, not retrofitted |
| Small team cannot sustain the 30-week timeline given the phase count | Team | Medium | Medium | Phase durations in Section 3 include an implicit buffer via the "4+ weeks, ongoing" framing of Phase 11; if a phase runs long, the roadmap absorbs it in the launch stabilization tail rather than compressing testing or security phases, which are treated as non-negotiable in duration |
| LLM API cost exceeds unit economics assumptions at real usage volume | Business | Medium | High | Per-agent token cost tracked from Phase 5's evaluation harness onward, not discovered at the first real invoice; rate limiting (Phase 4) bounds worst-case runaway cost per organization |
| Design-partner agencies do not find the product reliable enough to trust with real client campaigns | Business | Medium | Severe | The mandatory human approval gate (built into the core state machine since Phase 4, not a bolt-on) exists specifically so a reliability gap in agent output never becomes a reliability gap in what gets published |
| Team underestimates frontend complexity of the Execution View specifically | Technical | Medium | Medium | Phase 6's build order deliberately schedules the Execution View last among core pages, after every simpler pattern is proven, and Phase 7 exists as a dedicated integration-hardening phase specifically because this page was expected to need it |

---

## 14. Success Metrics

**Engineering KPIs**, tracked from Phase 9 onward via the monitoring stack built in Phase 10:

| Metric | Target |
|---|---|
| Deployment frequency | At least weekly once past Phase 10, daily during active Phase 11 stabilization |
| API p95 latency | Under 300ms for synchronous endpoints (excludes the intentionally async `POST /campaigns`) |
| Workflow completion rate | Over 90% of started workflows reach `completed` or `awaiting_approval` without an unrecovered failure |
| LLM cost per campaign | Tracked from Phase 5 onward; target set once the first month of real Phase 11 data establishes a real baseline, deliberately not guessed at in advance |
| Approval rate (content approved without edit) | This is the Zero-Touch Rate from 02_PRD.md, Section 13; target over 40% in Version 1, improving over time |
| System uptime | 99.5% for Version 1, appropriate for an early-stage product with a small design-partner base rather than an enterprise SLA the team is not yet resourced to guarantee |
| Developer productivity | Measured qualitatively through the Phase-by-Phase Definition of Done being met on schedule, rather than a proxy metric like lines of code or story points, which correlate poorly with the kind of foundational work this roadmap prioritizes |

---

## 15. Future Roadmap

**Version 2**: multi-channel campaigns (01_VISION.md, Phase 2 of the product roadmap) and closed-loop performance ingestion, where the Analytics Agent's output feeds back into future Strategy Agent runs, the first real instance of the procedural memory layer meaningfully improving planning rather than just storing history.

**Version 3**: the transition from a fixed DAG to dynamic, MCP-discovered workflow graphs (03_TECHNICAL_ARCHITECTURE.md's stated evolution path, and the reason 06_FRONTEND_DESIGN.md's Execution Timeline was built on a data-driven step model from Version 1 rather than hardcoded step types). This is the single largest architectural shift anticipated anywhere in these seven documents, and it is deliberately not attempted before Version 3, once real usage data across a real vertical exists to justify the added complexity.

**Enterprise Edition**: SSO, dedicated audit export, custom data residency, and a formal SLA, gated behind actual enterprise inbound demand rather than built speculatively, consistent with the earlier competitive analysis's finding that horizontal, unproven platforms rarely win enterprise trust before a narrow vertical has proven the reliability model first.

**Marketplace**: community and partner-built templates and tool connections, explicitly deferred (per 06_FRONTEND_DESIGN.md, Section 5's page description) until the Version 3 dynamic-tool architecture exists to support it safely.

**Multi-agent workflows across verticals**: the expansion into Sales Ops and Finance Ops named in 01_VISION.md's Phase 3, built by decoupling the core execution pipeline from marketing-specific logic, the same decoupling work that Version 3's dynamic DAG transition requires, making these two future efforts naturally sequential rather than independent.

**Self-improving memory**: the full realization of the procedural memory concept, genuine planning improvement from execution history rather than template promotion alone. This roadmap deliberately treats this as a multi-year research-adjacent effort, not a Version 2 or 3 deliverable, consistent with the earlier architecture analysis's finding that this capability remains at the edge of what is reliably buildable industry-wide as of this writing.

**Plugin ecosystem**: the natural end state of the Version 3 MCP-based tool architecture, allowing third parties to register new tools without a platform code change, deferred until that foundation exists for the same reason the Marketplace is deferred.

---

## 16. Appendix

**Folder strategy**: fully specified in 03_TECHNICAL_ARCHITECTURE.md (backend) and 06_FRONTEND_DESIGN.md, Section 12 (frontend). This appendix does not repeat those structures; it covers only the process around them.

**Git strategy and branching model**: trunk-based development. `main` is always deployable (per Phase 0's Definition of Done) and deploys automatically to staging. Feature branches are short-lived, ideally under three days, merged via pull request; long-lived feature branches are avoided specifically because they conflict with the "ship vertical slices" principle in Section 2, large, infrequent merges are where integration risk concentrates.

**Code review policy**: every PR requires one approval before merge; PRs touching authentication, authorization, or the encryption/secrets handling described in Section 11 require two approvals given the severity of a mistake in those areas. Reviews check against the specific document section the PR implements (per Section 2's documentation-driven principle), not just code style.

**Commit conventions**: Conventional Commits (`feat:`, `fix:`, `refactor:`, `docs:`, `test:`) enforced by a commit-lint CI check, since consistent commit messages are what make an automated changelog (used in Phase 10's release process) possible without manual curation.

**Documentation policy**: a PR that changes behavior described in any of documents 01 through 07 updates that document in the same PR. A document that drifts from the implementation it describes is worse than no document, since it actively misleads the next engineer who reads it.

**Release strategy**: releases are cut from `main` at any point it is green, not on a fixed calendar cadence, consistent with the "deployment frequency" KPI in Section 14; a release note is generated from the Conventional Commits history since the last release, reviewed by a human before publishing, not auto-published verbatim.

**Definition of Done, roadmap-wide**: a phase is done when its individual Definition of Done (Section 4) is met, its automated tests are green in CI, its relevant section of documents 01 through 06 has been verified against the real implementation (and updated if it drifted), and, from Phase 6 onward, at least one non-engineering team member has exercised the feature manually and confirmed it behaves as the product documents describe. Code that works but has not been checked against the product intent behind it is not done, it is merely functioning.