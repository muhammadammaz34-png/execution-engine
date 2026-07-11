## 1. Executive Summary

This API is the contract between three consumers: the Next.js frontend, the Celery-based Execution Engine, and, in later versions, external systems integrating through webhooks or a public API. The design philosophy follows directly from the Vision and Technical Architecture documents: the API layer stays thin and synchronous, while everything stochastic (agent execution) runs asynchronously behind it.

Three principles govern every endpoint in this document.

**The API describes outcomes, not workflow mechanics.** A client calls `POST /api/v1/campaigns` with a goal in natural language. It does not call separate endpoints to "add a research step" or "wire an agent to a tool." The Vision document's "zero scripting required" principle applies to the API surface as much as the UI: there is no endpoint that lets a client construct or rearrange the execution graph, because Version 1 has no dynamic graph to construct.

**Every mutation that touches an AI-generated artifact is auditable and reversible up to the point of publishing.** Draft content can be edited or discarded freely. Once a `Publishing Job` has fired, the API records the action immutably rather than pretending it can be undone.

**Synchronous endpoints return fast, and long-running work is always a background job with a pollable or subscribable status.** No endpoint in this API blocks on an LLM call. `POST /api/v1/campaigns` returns in well under a second with a `202 Accepted` and a resource the client can poll or subscribe to; the actual research, strategy, and content generation happen in Celery workers exactly as described in the Technical Architecture document.

---

## 2. API Architecture

The API is a thin layer inside the FastAPI modular monolith described in the Technical Architecture document. Requests pass through six logical layers, even though they live in one deployable service:

```
Client (Next.js)
      |
      v
Presentation Layer   -- Next.js server/client components, React Query, Zustand
      |  HTTPS / JSON, WebSocket for live workflow status
      v
REST Layer            -- FastAPI routers (api/v1/*), request/response schemas (Pydantic)
      |
      v
Business Layer         -- Domain services (domain/services/*): input validation beyond
      |                     schema shape, authorization checks, orchestration decisions
      v
Service Layer           -- Use-case functions: create_campaign(), approve_content(),
      |                      resume_workflow(). One function per meaningful action, each
      |                      independently testable and independent of the transport layer.
      v
Repository Layer         -- SQLAlchemy repositories, one per aggregate root (Campaign,
      |                       WorkflowExecution, GeneratedContent, etc). No business logic here.
      v
Background Workers        -- Celery tasks, triggered by the Service Layer, never called
                              directly by a router. Workers write results back through the
                              same Repository Layer, keeping the database as the single
                              source of truth for workflow state.
```

Two rules keep this from turning into an accidental big ball of mud as the codebase grows:

- **Routers never talk to the database or to Celery directly.** A router's only job is to validate the shape of a request, call exactly one Service Layer function, and translate the result into an HTTP response. This is what makes it possible to add a GraphQL or gRPC layer later (see Section 16) without touching business logic.
- **Repositories never contain conditionals about business rules.** "Can this user approve this campaign" is a Business Layer question. "Fetch the campaign with this ID scoped to this organization" is a Repository Layer question. Mixing the two is the most common way this kind of codebase becomes hard to test.

---

## 3. Authentication & Authorization

### Clerk authentication

All authentication is delegated to Clerk, matching the Technical Architecture document's security section. The frontend obtains a session JWT from Clerk after login (email/password, SSO, or social). Every API request carries this JWT in the `Authorization: Bearer <token>` header.

### JWT validation

A FastAPI dependency (`get_current_user`) runs on every non-public route:

1. Verifies the JWT signature against Clerk's JWKS endpoint (cached, refreshed on a schedule, not on every request, to avoid adding latency to the hot path).
2. Extracts `user_id` (Clerk's subject claim) and the `org_id` custom claim Clerk attaches when a user selects an active organization in the frontend.
3. Loads the corresponding `Users` and `Organization_Users` rows, attaching `current_user` and `current_membership` to the request context.
4. Rejects with `401 Unauthorized` if the signature is invalid, the token is expired, or no matching user exists. This is a distinct failure from `403 Forbidden`, which means the user is real but not allowed to do this specific thing.

### Role-based access control

Three roles, matching the Database Design document's `Roles` entity and the PRD's personas:

| Role | Maps to persona | Can do |
|---|---|---|
| **Owner** | Agency Owner | Everything, including billing, inviting/removing users, deleting the organization |
| **Editor** | Account Manager, Creative Strategist | Create campaigns, edit generated content, approve and publish, manage brand profiles and knowledge base |
| **Viewer** | Any read-only stakeholder (e.g. a client given visibility) | Read campaigns, content, and analytics; cannot create, edit, approve, or publish |

Role checks happen in the Business Layer, expressed as a decorator or dependency (`require_role("editor")`) rather than scattered `if` statements inside handlers, so the required role for an action is visible at the router definition.

### Organization access and permission model

Because a single user can belong to multiple organizations (the Database Design document's `Organization_Users` junction table exists specifically to support freelancers and fractional staff working across agencies), every request that touches organization-scoped data must resolve *which* organization it applies to. This API uses two mechanisms, and every route uses exactly one of them, never a mix:

- **Header-scoped for reads and writes on nested resources**: `X-Organization-Id` header, required on all `/campaigns`, `/brands`, and related routes. The API rejects the request with `403 Forbidden` if the authenticated user has no `Organization_Users` row for that organization, and Postgres Row Level Security (per the Database Design document) provides a second, independent enforcement layer beneath the application check.
- **Path-scoped for organization-level administration**: `/api/v1/organizations/{organization_id}/members`, where the ID in the path is itself the authorization boundary.

This split exists because header-scoping keeps the vast majority of endpoint paths clean (`/campaigns/{id}` rather than `/organizations/{org_id}/campaigns/{id}`), while administrative actions benefit from having the organization explicit and visible in the URL for audit and caching purposes.

---

## 4. API Versioning Strategy

All routes are prefixed `/api/v1/`. Version 1 is a whole-API version, not a per-resource version: there is no world in Version 1 where `/api/v1/campaigns` and `/api/v2/brands` coexist, because the surface area is small enough that splitting versioning per resource would add complexity without benefit at this stage.

**Forward compatibility within v1**: additive, non-breaking changes (new optional fields, new endpoints, new enum values consumers are expected to handle gracefully) ship without a version bump. Clients are contractually expected to ignore unrecognized fields and unrecognized enum values rather than fail closed.

**What forces a v2**: removing a field, changing a field's type or meaning, changing an endpoint's URL or HTTP method, or changing authentication semantics. Given the small initial user base, a v2 is more likely to be triggered by the transition to dynamic workflow graphs (Phase 2/3 of the Vision document) than by incremental Version 1 iteration.

**Deprecation strategy**: when v2 ships, v1 does not disappear immediately. Deprecated endpoints return a `Deprecation` and `Sunset` HTTP header (per RFC 8594) pointing to the replacement and the shutdown date, giving integrators, including the platform's own frontend, a minimum 90-day migration window before v1 is removed.

---

## 5. API Naming Conventions

### Resource naming

Resources are plural nouns, lowercase, hyphenated for multi-word resources: `/campaigns`, `/workflow-executions`, `/knowledge-base`. Nesting is used only where a resource cannot exist without its parent (`/campaigns/{id}/workflow-executions`), and capped at two levels deep; anything that would require a third level (for example, a single carousel slide) is instead addressed by its own top-level ID (`/carousel-slides/{id}`) to avoid unwieldy URLs.

### HTTP methods

| Method | Use |
|---|---|
| `GET` | Retrieve a resource or collection. Never mutates state. |
| `POST` | Create a resource, or trigger an action that does not map cleanly to a resource state change (e.g. `POST /workflow-executions/{id}/retry`) |
| `PATCH` | Partial update to an existing resource |
| `PUT` | Not used in this API. Full-resource replacement adds risk for AI-generated content with many fields; `PATCH` with explicit fields is safer and matches how the WYSIWYG editor sends edits. |
| `DELETE` | Soft-delete (sets `deleted_at`), per the Database Design document's global soft-delete convention. Hard deletes exist only in the GDPR erasure worker, never as a client-facing endpoint. |

### Status codes

| Code | Meaning in this API |
|---|---|
| `200 OK` | Successful `GET` or `PATCH` |
| `201 Created` | Successful `POST` that created a resource synchronously (e.g. a brand profile) |
| `202 Accepted` | Successful `POST` that enqueued asynchronous work (e.g. campaign creation); response body includes the resource in its initial state, not the final result |
| `204 No Content` | Successful `DELETE` |
| `400 Bad Request` | Request shape is invalid (fails Pydantic validation) |
| `401 Unauthorized` | Missing or invalid authentication |
| `403 Forbidden` | Authenticated, but not authorized for this organization or role |
| `404 Not Found` | Resource does not exist, or exists but is outside the caller's organization (the API returns 404, not 403, when a resource is invisible to the caller, to avoid confirming the existence of other organizations' data) |
| `409 Conflict` | Action is invalid given the resource's current state (e.g. approving a campaign that is not `AWAITING_APPROVAL`) |
| `422 Unprocessable Entity` | Request is well-formed and passes schema validation but fails a business rule (e.g. a negative constraint violation caught before an agent runs) |
| `429 Too Many Requests` | Rate limit exceeded, see Section 12 |
| `500 Internal Server Error` | Unhandled server fault |
| `503 Service Unavailable` | A required upstream (LLM provider, LinkedIn API) is down; used instead of 500 when the API can distinguish the failure as external |

### Error responses

A single error envelope shape is used everywhere in the API (full schema in Section 11).

### Pagination

Cursor-based (keyset) pagination on every collection endpoint, matching the Database Design document's UUIDv7 primary key strategy, which makes ID-based cursors naturally chronological without a separate `created_at` cursor column.

```
GET /api/v1/campaigns?limit=25&cursor=01930a2e-...
```

Response includes `next_cursor` (null when no further pages exist) rather than a total count, since counting large, frequently-written tables like `Audit Logs` or `Content Performance` is expensive and rarely what the client actually needs.

### Filtering and sorting

Filtering uses explicit query parameters rather than a generic query language, to keep validation simple and indexes predictable: `GET /api/v1/campaigns?status=awaiting_approval&brand_profile_id=...`. Sorting is limited to a fixed, documented set of fields per resource (`?sort=created_at&order=desc`), matching the indexes actually defined in the Database Design document, so a sort parameter can never silently trigger a full table scan.

---

## 6. Core API Modules

Every endpoint below is implicitly `Authorization: Bearer <clerk_jwt>` plus, where noted, `X-Organization-Id`. Request and response bodies are JSON. Fields shown are representative, not exhaustive, but every field shown is a real field from the Database Design document's schema, not a placeholder.

### 6.1 Authentication

**Purpose**: Session and identity endpoints not already handled by Clerk's own hosted UI.

- `GET /api/v1/auth/me` — returns the current user plus their organization memberships and roles
- `POST /api/v1/auth/switch-organization` — validates the target organization membership and returns an updated context for the frontend to store

**Validation**: none beyond JWT presence. **Authorization**: any authenticated user.

### 6.2 Organizations

**Purpose**: The root tenant entity (Vision document's agency).

- `POST /api/v1/organizations` — creates an organization; the creator becomes `Owner`
- `GET /api/v1/organizations/{id}` — organization profile and plan/billing status
- `PATCH /api/v1/organizations/{id}` — update name, settings
- `GET /api/v1/organizations/{id}/members` — list members and roles
- `POST /api/v1/organizations/{id}/members` — invite a user by email with a role
- `PATCH /api/v1/organizations/{id}/members/{user_id}` — change a member's role
- `DELETE /api/v1/organizations/{id}/members/{user_id}` — remove a member

**Validation**: organization name 1 to 100 characters; role must be one of `owner`, `editor`, `viewer`; cannot remove the last remaining `Owner`. **Authorization**: `Owner` only for membership mutations; any member for `GET`.

### 6.3 Users

**Purpose**: The authenticated individual's own profile (Clerk owns credentials; this module owns app-specific profile data).

- `GET /api/v1/users/me` — profile, notification preferences
- `PATCH /api/v1/users/me` — update display name, notification preferences

**Validation**: notification preferences restricted to a known enum set. **Authorization**: self only; there is no endpoint to fetch another user's profile directly, membership lists (6.2) are the sanctioned way to see who else is in an organization.

### 6.4 Brands

**Purpose**: The `Brand Profiles` entity, the agency's client.

- `POST /api/v1/brands` — create a brand profile
- `GET /api/v1/brands` — list brands for the current organization, paginated
- `GET /api/v1/brands/{id}` — brand detail, including linked brand voice and asset counts
- `PATCH /api/v1/brands/{id}` — update brand metadata
- `DELETE /api/v1/brands/{id}` — soft-delete
- `GET /api/v1/brands/{id}/voice` / `PUT /api/v1/brands/{id}/voice` — get/set structured brand voice rules and negative constraints (e.g. "no emojis")

Example request for the voice endpoint, because negative constraints are directly load-bearing for the Validation Agent described in the PRD:

```json
PUT /api/v1/brands/b1f0.../voice
{
  "tone_descriptors": ["direct", "technical", "no marketing fluff"],
  "negative_constraints": ["no emojis", "no exclamation points", "never say 'game-changing'"],
  "reading_level": "professional",
  "preferred_cta_style": "question-based"
}
```

**Validation**: `negative_constraints` capped at 50 entries to keep prompt context bounded; brand must belong to the caller's organization. **Authorization**: `Editor` or `Owner` to write, any role to read.

### 6.5 Campaigns

**Purpose**: The parent object for a single goal-to-execution run. The most important resource in the API.

- `POST /api/v1/campaigns` — the primary entry point described in the Vision document: submit a natural-language goal, get a campaign back in `PENDING` state, and a workflow execution enqueued
- `GET /api/v1/campaigns` — list, filterable by `status`, `brand_profile_id`, `created_after`
- `GET /api/v1/campaigns/{id}` — full detail including current workflow execution status
- `PATCH /api/v1/campaigns/{id}` — update metadata (name, deadline) not the goal itself, once execution has started the goal is immutable for that run; a changed goal is a new campaign
- `DELETE /api/v1/campaigns/{id}` — soft-delete, cancels any in-flight workflow execution first

```json
POST /api/v1/campaigns
{
  "brand_profile_id": "b1f0...",
  "goal": "Launch a LinkedIn campaign about Kubernetes security aimed at platform engineering leads",
  "constraints": {
    "post_count": 5,
    "deadline": "2026-07-25",
    "include_carousel": true
  }
}
```

```json
202 Accepted
{
  "id": "c9a2...",
  "status": "pending",
  "workflow_execution_id": "we0b...",
  "created_at": "2026-07-11T10:02:00Z"
}
```

**Validation**: `goal` required, 10 to 2000 characters; `constraints.post_count` between 1 and 20 for Version 1; organization must have available quota (Section 12). **Authorization**: `Editor` or `Owner` to create; any role to read.

### 6.6 Campaign Goals

**Purpose**: The structured, parsed representation of the natural-language goal (the Intent Engine's output described in the PRD).

- `GET /api/v1/campaigns/{id}/goal` — returns the structured `{objective, target_audience, tone, deadline}` the Planner extracted from the raw goal text

This is intentionally read-only in Version 1. The PRD's out-of-scope section excludes "dynamic, real-time workflow reconfiguration," and allowing edits to the parsed goal after planning has started would reopen that door. **Authorization**: any role.

### 6.7 Workflow Executions

**Purpose**: Tracks the lifecycle of a single DAG run. Covered in full in Section 7.

- `GET /api/v1/workflow-executions/{id}` — status, current step, timestamps
- `POST /api/v1/workflow-executions/{id}/pause`
- `POST /api/v1/workflow-executions/{id}/resume`
- `POST /api/v1/workflow-executions/{id}/cancel`
- `POST /api/v1/workflow-executions/{id}/retry`
- `GET /api/v1/workflow-executions/{id}/logs`

### 6.8 Workflow Steps

**Purpose**: Individual DAG nodes (Research, Strategy, and so on).

- `GET /api/v1/workflow-executions/{id}/steps` — ordered list of steps with status, timing, and token cost per step
- `GET /api/v1/workflow-steps/{id}` — single step detail including the agent that ran it

**Authorization**: any role, read-only. There is no create/update endpoint; steps are written exclusively by the Execution Engine's internal service layer, never by an API client.

### 6.9 Generated Content

**Purpose**: The `Generated Content` entity, LinkedIn post drafts and their approved versions.

- `GET /api/v1/campaigns/{id}/content` — all generated content for a campaign
- `GET /api/v1/generated-content/{id}` — single post, both `draft_content` and `approved_content` if it has been edited
- `PATCH /api/v1/generated-content/{id}` — the WYSIWYG editor's save action; writes to `approved_content` and appends an `Audit Logs` row and a `Memory` entry recording the diff, per the PRD's requirement that edits feed back into learning

```json
PATCH /api/v1/generated-content/gc44.../
{
  "approved_content": "Kubernetes RBAC misconfigurations are the single most common..."
}
```

**Validation**: content length within LinkedIn's post limits (3,000 characters for a standard post); cannot edit content belonging to a campaign in `completed` or `cancelled` state. **Authorization**: `Editor` or `Owner`.

### 6.10 Carousel Slides

**Purpose**: Child entities of Generated Content for the multi-slide carousel format.

- `GET /api/v1/generated-content/{id}/carousel-slides` — ordered slides
- `PATCH /api/v1/carousel-slides/{id}` — edit a single slide's text

**Validation**: slide order is fixed at generation time; Version 1 does not support reordering or adding slides, only editing text within a slide, per the PRD's fixed-pipeline scope.

### 6.11 Image Prompts

**Purpose**: Text prompts intended for the user's own Midjourney or Canva workflow, per the PRD's explicit Version 1 scope (no native image generation).

- `GET /api/v1/generated-content/{id}/image-prompts`
- `PATCH /api/v1/image-prompts/{id}` — edit prompt text
- `POST /api/v1/image-prompts/{id}/copy-event` — optional analytics ping so the product can measure how often prompts are actually used, supporting the Zero-Touch Rate metric

### 6.12 Media Assets

**Purpose**: Brand logos, color palettes, and any final images a user uploads back into the platform after generating them externally.

- `POST /api/v1/media-assets` — `multipart/form-data` upload, returns a Supabase Storage URI
- `GET /api/v1/brands/{id}/media-assets`
- `DELETE /api/v1/media-assets/{id}`

**Validation**: file type restricted to `image/png`, `image/jpeg`, `image/webp`; max 10MB. **Authorization**: `Editor` or `Owner`.

### 6.13 Approvals

**Purpose**: Tracks who approved what, when, mapped directly to the Database Design document's `Approvals` entity and the Human Approval Flow.

- `POST /api/v1/campaigns/{id}/approve` — the single most consequential endpoint in the API; transitions a campaign from `awaiting_approval` to resume execution toward publishing
- `POST /api/v1/campaigns/{id}/request-changes` — sends the campaign back to content generation with the reviewer's notes attached as additional context for the Content agents

```json
POST /api/v1/campaigns/c9a2.../approve
{
  "approved_content_ids": ["gc44...", "gc45...", "gc46..."],
  "note": "Approved, ship as scheduled"
}
```

**Validation**: campaign must be in `awaiting_approval` state, otherwise `409 Conflict`; every `generated_content_id` in the request must belong to this campaign. **Authorization**: `Editor` or `Owner`, matching the PRD's mandatory human-in-the-loop requirement, this cannot be bypassed by any role in Version 1.

### 6.14 Publishing Jobs

**Purpose**: The queue for sending approved content to LinkedIn.

- `GET /api/v1/campaigns/{id}/publishing-jobs`
- `GET /api/v1/publishing-jobs/{id}` — status (`queued`, `scheduled`, `published`, `failed`), scheduled time, LinkedIn post ID once live
- `PATCH /api/v1/publishing-jobs/{id}` — reschedule `scheduled_time`, only while status is `queued` or `scheduled`
- `POST /api/v1/publishing-jobs/{id}/cancel`

**Validation**: `scheduled_time` must be in the future; cannot patch a job that has already published, that returns `409 Conflict`. **Authorization**: `Editor` or `Owner`.

### 6.15 Analytics

**Purpose**: Post-campaign performance, ingested from LinkedIn per the PRD.

- `GET /api/v1/campaigns/{id}/analytics` — impressions, clicks, engagement rate against the original goal
- `GET /api/v1/brands/{id}/analytics/summary` — rolled-up performance across a brand's campaigns, the input to the Phase 2 "closed-loop performance ingestion" described in the Vision document

**Authorization**: any role, read-only.

### 6.16 Memory

**Purpose**: Exposes the `Memory` entity, structured learnings derived from historical edits, primarily for transparency (the Vision document's "Radical Clarity" principle) rather than direct manipulation.

- `GET /api/v1/brands/{id}/memory` — list learned preferences (e.g. "user consistently deletes hashtags"), each with the evidence count and the date it was last reinforced
- `DELETE /api/v1/memory/{id}` — lets a user retract a learned preference that no longer applies

**Authorization**: `Editor` or `Owner`.

### 6.17 Knowledge Base

**Purpose**: Uploaded brand guideline documents and their embedding sync status.

- `POST /api/v1/brands/{id}/knowledge-base` — `multipart/form-data` upload (PDF, DOCX, TXT); returns immediately with `sync_status: pending`, embedding happens asynchronously
- `GET /api/v1/brands/{id}/knowledge-base` — list documents and sync status
- `DELETE /api/v1/knowledge-base/{id}` — removes both the Postgres record and the corresponding Qdrant vectors

**Validation**: file type restricted to `application/pdf`, `text/plain`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`; max 25MB.

### 6.18 Prompt Templates

**Purpose**: Version-controlled system prompts per agent, per the Database Design document. Internal/admin surface, not exposed to ordinary agency users in Version 1.

- `GET /api/v1/admin/prompt-templates` — list, filterable by agent
- `POST /api/v1/admin/prompt-templates` — create a new version
- `PATCH /api/v1/admin/prompt-templates/{id}/activate` — promote a version to active

**Authorization**: platform-internal role, not any of the three organization-scoped roles; enforced by a separate `is_platform_admin` claim, not by `X-Organization-Id` at all.

### 6.19 Tool Connections

**Purpose**: Encrypted OAuth credentials for external tools, currently LinkedIn only.

- `GET /api/v1/brands/{id}/tool-connections` — connection status per platform, never returns the token itself
- `POST /api/v1/brands/{id}/tool-connections/linkedin/authorize` — returns a LinkedIn OAuth URL to redirect the user to
- `GET /api/v1/tool-connections/linkedin/callback` — OAuth callback, exchanges the code, encrypts and stores the token (AES-256-GCM per the Technical Architecture document), never returned in any subsequent response body
- `DELETE /api/v1/brands/{id}/tool-connections/linkedin` — revoke

**Authorization**: `Owner` only, given the sensitivity of publishing credentials.

### 6.20 Notifications

**Purpose**: In-app and email notifications (campaign awaiting approval, publishing failed, and so on).

- `GET /api/v1/notifications` — the current user's notifications, paginated, filterable by `read`
- `PATCH /api/v1/notifications/{id}` — mark read
- `POST /api/v1/notifications/mark-all-read`

### 6.21 Audit Logs

**Purpose**: The insert-only ledger. Read-only surface for Version 1; no client ever writes to this table directly, every write is a side effect of another action.

- `GET /api/v1/organizations/{id}/audit-logs` — filterable by `actor_id`, `resource_type`, `date_range`

**Authorization**: `Owner` only.

### 6.22 Feature Flags

**Purpose**: Internal rollout control for new agents or DAG structures.

- `GET /api/v1/feature-flags` — flags active for the caller's organization

**Authorization**: read-only, any role; writes happen through an internal admin tool, not this API, in Version 1.

### 6.23 Health Check and System Settings

- `GET /api/v1/health` — no auth required; returns `200 OK` with database, Redis, and Qdrant connectivity status, used by the deployment platform's health probes
- `GET /api/v1/system/settings` — public, non-sensitive platform configuration the frontend needs at boot (feature flags aside), such as supported file types and size limits

---

## 7. Workflow Execution APIs

These endpoints operate on the state machine described in the Technical Architecture document (`PENDING`, `RESEARCHING`, `STRATEGIZING`, `GENERATING_CONTENT`, `AWAITING_APPROVAL`, `PUBLISHING`, `COMPLETED`, `FAILED`, `CANCELLED`).

**Start**: implicit. There is no separate "start workflow" endpoint; `POST /api/v1/campaigns` both creates the campaign and enqueues its workflow execution in one call, because in Version 1 a campaign cannot exist without exactly one workflow execution.

**Pause**: `POST /api/v1/workflow-executions/{id}/pause`. Valid from any in-progress state. Sets status to `PAUSED` and prevents the next Celery task in the chain from being enqueued once the current step finishes; it does not kill an already-running step, since interrupting an in-flight LLM call generates no useful state to resume from.

**Resume**: `POST /api/v1/workflow-executions/{id}/resume`. Valid from `PAUSED` or `AWAITING_APPROVAL` (the latter is what the approval endpoint in Section 6.13 calls internally). Re-enqueues the next step using the checkpointed `Task Results` context.

**Cancel**: `POST /api/v1/workflow-executions/{id}/cancel`. Terminal. Valid from any non-terminal state. Any in-flight Publishing Job is cancelled with it if it has not yet fired.

**Retry**: `POST /api/v1/workflow-executions/{id}/retry`. Valid only from `FAILED`. Re-enqueues the failed step using its last checkpoint, incrementing `retry_count`; after 3 automatic retries (matching the exponential backoff policy in the Technical Architecture document) the workflow moves to `FAILED` and requires this explicit, human-triggered retry rather than retrying indefinitely on its own.

```json
GET /api/v1/workflow-executions/we0b...
{
  "id": "we0b...",
  "campaign_id": "c9a2...",
  "status": "generating_content",
  "current_step": "copywriter_agent",
  "started_at": "2026-07-11T10:02:03Z",
  "steps_completed": 3,
  "steps_total": 6,
  "retry_count": 0
}
```

**Track progress**: `GET /api/v1/workflow-executions/{id}` (poll) or a WebSocket at `WS /api/v1/workflow-executions/{id}/stream` (push), matching the Technical Architecture document's system diagram, which shows both REST and WebSocket channels between the frontend and the API layer. The WebSocket exists specifically so the "real-time terminal/graph UI" described in the PRD's user journey does not require aggressive polling.

**Retrieve logs**: `GET /api/v1/workflow-executions/{id}/logs` returns the ordered list of step transitions, each with a human-readable message ("Research Agent found 12 relevant sources", "Validation Agent flagged a tone deviation in post 3"), not raw stack traces. This directly implements the Architecture Review's recommendation, carried over from the broader analysis this project grew out of, that failure summaries need their own design attention rather than surfacing raw logs to a non-technical account manager.

---

## 8. AI Agent APIs

Agents are never called directly by frontend clients. There is no `POST /api/v1/agents/research` endpoint, and there should not be one: exposing agents individually would let a client construct ad hoc workflows, which contradicts the "goal-driven, never workflow-driven" principle from the Vision document. Agents are invoked exclusively by the Execution Engine's internal Agent Router. This section documents their contracts because the Workflow Steps and Logs endpoints (6.8, 7) surface agent inputs and outputs to the frontend, and the shapes need to be stable.

| Agent | Inputs | Outputs | Error handling |
|---|---|---|---|
| **Research** | Goal, target audience, brand domain | Structured `ContextDocument` (sources, key facts, competitor notes) | On zero usable sources found, does not fabricate; returns a `ContextDocument` flagged `low_confidence`, which the Strategy Agent must handle explicitly rather than silently proceeding |
| **Strategy** | `ContextDocument`, goal, brand voice | `CampaignArchitecture` (post count, sequence, format per post, messaging pillars) | Validates its own output against `constraints.post_count` from the Campaign Goal before returning; a mismatch is a hard failure, not a silent adjustment |
| **Copywriter** | `CampaignArchitecture`, brand voice, top-3 similar past posts from Qdrant | Draft LinkedIn post text per post in the architecture | Enforces LinkedIn's character limit at generation time, not just at save time |
| **Carousel** | `CampaignArchitecture` entries flagged as carousel format | Ordered slide text | Caps slide count at 10 per LinkedIn's own carousel limit |
| **Image Prompt** | Draft post text, brand visual guidelines | Text prompts for external image tools | N/A, no external call to fail; pure generation |
| **Validator** | All draft content, brand voice negative constraints, `ContextDocument` | Pass/fail per piece of content, with specific flagged spans and reasons | This is the "internal adversarial network" from the PRD; a fail here routes the workflow to `AWAITING_APPROVAL` with the flags attached rather than silently blocking, so a human sees exactly what was caught and why |
| **Publisher** | Approved content, LinkedIn tool connection, scheduled time | `Publishing Job` status transition | On a LinkedIn API error (rate limit, token expired), retries per the standard backoff policy; on token expiry specifically, also fires a notification (6.20) telling the Owner to reauthorize, since this failure mode cannot be resolved by retrying alone |
| **Analytics** | LinkedIn post IDs from completed Publishing Jobs | Impression/engagement records written to `Content Performance` | Runs on a schedule (not part of the synchronous DAG), so its errors never affect a campaign's completion status |

---

## 9. File Upload APIs

Three upload surfaces exist, all using `multipart/form-data` against a FastAPI endpoint that streams directly to Supabase Storage rather than buffering the full file in application memory:

- **Brand assets**: logos, color palette references. See 6.12.
- **Knowledge base documents**: brand guidelines, past post examples. See 6.17. These trigger an asynchronous embedding job (chunk, embed via the OpenAI embeddings API, write to Qdrant, update `sync_status`) rather than blocking the upload response.
- **Generated files**: CSV exports of a campaign's content, produced on demand rather than stored: `GET /api/v1/campaigns/{id}/export.csv`, streamed directly rather than pre-generated and stored, since it is cheap to regenerate and storing it would be one more thing to keep in sync.

**Storage strategy**: every upload gets a Supabase Storage object under a path namespaced by organization ID (`{organization_id}/brands/{brand_id}/knowledge-base/{uuid}.pdf`), matching the Database Design document's tenant isolation principle at the storage layer, not just the database layer. Only the URI is ever stored in Postgres.

---

## 10. Webhook Architecture

Version 1 has one true inbound webhook consumer (LinkedIn) and one outbound webhook capability (for the future public API in Section 16, stubbed but not exposed to customers yet).

**Publishing callbacks**: LinkedIn does not push webhooks for post status in the way this document would ideally want; the Publisher Agent polls for confirmation after scheduling. This is documented here explicitly because it is a real constraint the Technical Architecture document does not call out: `Publishing Jobs` status is confirmed via polling with backoff, not a callback, until LinkedIn's API offers one.

**Third-party integrations (outbound, future)**: when the platform exposes a public API (Section 16), organizations will be able to register a webhook URL to receive `campaign.completed`, `content.approved`, and `publishing.failed` events. Designing the event names and payload shapes now, even though no endpoint fires them yet, avoids a breaking change later:

```json
{
  "event": "campaign.completed",
  "organization_id": "...",
  "campaign_id": "...",
  "timestamp": "2026-07-11T10:14:02Z"
}
```

**Retry policy (future)**: exponential backoff, 5 attempts over 24 hours, then the webhook is marked `failed` and surfaced in a dashboard rather than retried indefinitely.

**Security (future)**: every outbound webhook payload is signed with an HMAC-SHA256 secret unique to the organization, sent in an `X-Signature` header, so a receiving system can verify authenticity without exposing any platform credential.

---

## 11. Error Handling Strategy

Every error response, regardless of source, uses one envelope:

```json
{
  "error": {
    "code": "validation_error",
    "message": "goal must be between 10 and 2000 characters",
    "status": 400,
    "details": [
      {"field": "goal", "issue": "too_short"}
    ],
    "request_id": "req_9f2c..."
  }
}
```

`request_id` is generated per request and attached to every log line and Langfuse trace touched during that request, so a support conversation about a specific failure can go straight from a user-reported ID to the exact trace, without grepping logs by timestamp.

- **Validation errors** (`400`): Pydantic schema failures, caught by a global exception handler, never a bare FastAPI default error shape.
- **Authentication errors** (`401`): missing, malformed, or expired JWT.
- **Business logic errors** (`409`, `422`): state conflicts and rule violations, always with a `message` a non-technical account manager can read directly in a toast notification, not a stack trace fragment.
- **Rate limiting** (`429`): includes a `Retry-After` header in seconds, not just the error body.
- **Server errors** (`500`, `503`): the response body never includes exception details or stack traces, only the `request_id`, full detail goes to Sentry, matching the Technical Architecture document's observability section.

---

## 12. Rate Limiting

Redis-backed sliding window, keyed by different dimensions depending on what is being protected, matching the Technical Architecture document's rate limiting note but made specific here:

| Limit | Key | Default | Reason |
|---|---|---|---|
| General API requests | `user_id` | 300 requests/minute | Protects against runaway frontend polling loops, the most likely accidental abuse pattern |
| Campaign creation | `organization_id` | Plan-dependent (e.g. 10/day on a starter tier) | Directly bounds LLM spend; this is a billing control as much as an infrastructure one |
| AI generation (retry, request-changes) | `organization_id` | 50 agent invocations/hour | Prevents a confused or automated client from looping the Validator/Copywriter cycle indefinitely and running up token cost |
| Webhook delivery (future) | `organization_id` | 100/minute outbound | Protects the platform's own egress and the receiving system |

Limits are enforced in the Business Layer, not the Repository Layer, and return `429` with `Retry-After` before any database or LLM call is made, so a rate-limited request never consumes the resource it was trying to protect.

---

## 13. API Security

- **Authentication**: Clerk-issued JWT, verified on every request per Section 3.
- **Authorization**: RBAC plus organization scoping per Section 3, enforced twice, once in the Business Layer and once at the database level via Postgres Row Level Security, so an application-layer bug cannot by itself leak cross-tenant data.
- **Input validation**: every request body is a Pydantic model with explicit field constraints (length, format, enum membership); no endpoint accepts an untyped dictionary.
- **Prompt injection protection**: this deserves more than a bullet point, because the platform's tool list (LinkedIn publishing, document ingestion, web research) makes it a real, not theoretical, risk. Three concrete controls: user-supplied goal text and any content fetched by the Research Agent are passed to the LLM in a clearly delimited data field, never concatenated into the system prompt string; the Validator Agent independently reviews all generated content before it can reach `awaiting_approval`, acting as a second, differently-prompted check rather than trusting the generating agent's own judgment; and the Publisher Agent will only ever act on content that has passed through the `Approvals` endpoint (6.13), so even a fully successful prompt injection against an earlier agent cannot cause a LinkedIn post to go out without a human clicking approve.
- **Secret management**: LinkedIn OAuth tokens and any platform API keys are encrypted at rest with AES-256-GCM; the encryption key itself lives in the deployment platform's secret manager, never in the database or in environment variables checked into source control.
- **CORS**: the API allows only the deployed frontend origin(s) in production; local development origins are permitted only when `ENVIRONMENT=development`.
- **CSRF**: not applicable in the traditional sense, since the API is a pure JSON API authenticated via bearer token rather than cookies, which is itself the standard mitigation for CSRF.
- **SQL injection protection**: SQLAlchemy's parameterized queries throughout; no endpoint constructs SQL from string concatenation, including the filtering and sorting parameters in Section 5, which are mapped through an explicit allowlist of column names, never interpolated directly.

---

## 14. Background Processing APIs

These endpoints expose Celery's state to clients without exposing Celery itself.

- `GET /api/v1/tasks/{task_id}` — generic task status (`pending`, `started`, `success`, `failure`), used internally by the Workflow Execution polling endpoints rather than called directly by the frontend in most cases
- Queue management and retry configuration are operational concerns, not exposed via the public API; they are configured through Celery's own configuration and monitored through Flower or an equivalent internal-only dashboard, not through `/api/v1/*`

The distinction matters: `Workflow Executions` (Section 7) is the product-facing abstraction over background work, and `Tasks` (this section) is the low-level primitive underneath it. Keeping them as separate API surfaces means the platform could swap Celery for Temporal, as the Technical Architecture document's Future Evolution section anticipates, without changing a single frontend-facing endpoint.

---

## 15. API Documentation Strategy

FastAPI generates an OpenAPI 3.1 schema automatically from the route definitions and Pydantic models, which is the primary source of truth rather than hand-written documentation that can drift from the implementation.

- **Swagger UI** at `/api/docs`, enabled only in development and staging, disabled in production to avoid exposing the full schema and example payloads publicly before there is a public API product to justify it.
- **Redoc** at `/api/redoc` for a more readable reference view, same environment restriction.
- **SDK generation**: the OpenAPI schema is the input to auto-generated TypeScript types for the Next.js frontend (via `openapi-typescript` or equivalent), so the frontend's request and response types are mechanically derived from the backend contract rather than hand-maintained and prone to drifting out of sync.

---

## 16. Future API Evolution

- **GraphQL**: not planned for Version 1 or 2. The API's read patterns are shallow enough (a campaign, its content, its status) that REST's over-fetching cost is low; GraphQL would add real complexity for a benefit that does not exist yet. Revisit only if a future analytics or reporting surface needs deep, client-specified nested queries.
- **Streaming APIs**: the WebSocket endpoint in Section 7 is the first streaming surface. If agent output itself needs to stream token-by-token to the frontend, for perceived latency, that is a Version 2 conversation, since Version 1's Validator gate means partial, unvalidated content should not reach the user regardless.
- **WebSockets**: already present for workflow status (Section 7); expansion to a general-purpose event bus is a natural Version 2 step once there is more than one kind of real-time event worth pushing.
- **Public API**: the versioning strategy in Section 4 and the webhook shapes stubbed in Section 10 are deliberately designed now so that exposing `/api/v1/*` (or a dedicated `/api/public/v1/*`) to third-party developers later is additive, not a rewrite. This is the natural on-ramp for the Vision document's Phase 2 and Phase 3 expansion into other operational verticals: an external system integrating with Sales Ops or Finance Ops workflows would consume the same architecture through a public-facing version of this same API.
- **Marketplace API and Plugin APIs**: explicitly out of scope until the platform has moved from the fixed DAG described in the Technical Architecture document to the dynamic, MCP-based tool discovery its own Future Evolution section describes. Designing a plugin API against a fixed-DAG backend would mean redesigning it again the moment MCP adoption happens, so this is intentionally deferred rather than built early.