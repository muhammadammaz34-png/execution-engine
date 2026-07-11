## 1. Executive Summary

This document defines the database architecture for Version 1 of the Autonomous Marketing Operations Engine. The database philosophy is rooted in strict data integrity, deterministic state management for stochastic AI processes, and enterprise-grade multi-tenancy.

Because this system orchestrates autonomous agents, the database is not merely a data store; it is the state machine that governs the execution graph. The architecture strictly separates relational business logic (PostgreSQL), ephemeral execution state and caching (Redis), unstructured asset storage (Supabase Storage), and high-dimensional semantic memory (Qdrant). This separation of concerns ensures that the platform remains highly performant, auditable, and scalable as it evolves into a universal outcome-driven execution layer.

---

## 2. Database Architecture Overview

The persistence layer is distributed across four specialized systems to optimize for their respective workloads:

* **PostgreSQL (Source of Truth):** Hosted via Supabase. Manages all relational data, user identities, role-based access control (RBAC), multi-tenant logical isolation, deterministic workflow states, and structured campaign data.
* **Redis (Ephemeral State & Queuing):** Manages Celery task brokering, rate limiting, and short-term caching of immutable organization settings or frequently accessed, slowly changing dimensions.
* **Qdrant (Semantic Memory):** Manages vector embeddings. Used for semantic search, brand voice retrieval, and contextual memory, allowing AI agents to retrieve historical context without bloating relational queries.
* **Supabase Storage (Object Storage):** Manages BLOBs. Stores user-uploaded brand guidelines (PDFs, DOCX) and generated media assets (CSV exports, final image renders), storing only the URIs in PostgreSQL.

---

## 3. Core Entities

The PostgreSQL schema is logically divided into domains.

### 3.1. Identity & Access Domain

* **Organizations:**
* *Purpose:* The root tenant entity representing a marketing agency.
* *Relationships:* 1:N with Users, Brand Profiles, Campaigns, Billing Records.
* *Lifecycle:* Created on signup, soft-deleted on churn.
* *Indexes:* Primary Key (UUID).


* **Users:**
* *Purpose:* Represents human operators (Agency Owners, Account Managers).
* *Relationships:* M:N with Organizations (via Organization_Users junction table) to support freelancers managing multiple agencies.
* *Lifecycle:* Created via Clerk webhook. Soft-deleted.
* *Indexes:* Primary Key (UUID), Unique index on email.


* **Roles:**
* *Purpose:* Defines RBAC permissions (e.g., Owner, Editor, Viewer).
* *Relationships:* 1:N with Organization_Users.


* **API Keys:**
* *Purpose:* Manages programmatic access for external integrations or future API access.
* *Relationships:* N:1 with Organizations.



### 3.2. Brand & Context Domain

* **Brand Profiles:**
* *Purpose:* Represents the agency's underlying client. Holds specific metadata for a brand.
* *Relationships:* N:1 with Organizations. 1:N with Campaigns, Brand Voice, Knowledge Base.
* *Indexes:* Composite index (organization_id, brand_profile_id).


* **Brand Assets & Media Assets:**
* *Purpose:* Stores metadata and URIs for logos, color palettes, and generated images.
* *Relationships:* N:1 with Brand Profiles.


* **Brand Voice:**
* *Purpose:* Structured rules, negative constraints (e.g., "no emojis"), and tonal guidelines.
* *Relationships:* 1:1 or N:1 with Brand Profiles.


* **Knowledge Base & Embeddings:**
* *Purpose:* Tracks documents uploaded for AI context. The actual vectors live in Qdrant, but Postgres holds the metadata and sync status.
* *Relationships:* N:1 with Brand Profiles.


* **Memory:**
* *Purpose:* Structured logging of user preferences derived from historical edits.
* *Lifecycle:* Continuously appended by the Learning Engine.


* **Prompt Templates:**
* *Purpose:* Version-controlled system prompts for specific AI Agents.
* *Relationships:* N:1 with AI Agents.


* **Tool Connections:**
* *Purpose:* Stores encrypted OAuth tokens (e.g., LinkedIn credentials).
* *Relationships:* N:1 with Brand Profiles.



### 3.3. Workflow & Execution Domain

* **Campaigns:**
* *Purpose:* The parent object for a single user intent execution.
* *Relationships:* N:1 with Brand Profiles, 1:N with Workflow Executions.
* *Indexes:* B-Tree on status, Created_at for sorting.


* **Campaign Goals:**
* *Purpose:* The structured JSON representation of the parsed user intent.
* *Relationships:* 1:1 with Campaigns.


* **Workflow Executions:**
* *Purpose:* Tracks the lifecycle of a DAG run.
* *Relationships:* N:1 with Campaigns, 1:N with Workflow Steps.
* *Lifecycle:* Instantiated as PENDING, transitions to COMPLETED, FAILED, or AWAITING_APPROVAL.


* **Workflow Steps & Tasks:**
* *Purpose:* Represents individual nodes in the DAG (e.g., "Research", "Draft Copy").
* *Relationships:* N:1 with Workflow Executions.


* **AI Agents & Agent Executions:**
* *Purpose:* Registry of available agents and their execution logs.
* *Relationships:* 1:N with Tasks.


* **Task Results:**
* *Purpose:* Stores the intermediate JSON outputs of an agent.
* *Lifecycle:* Immutable once written.



### 3.4. Content & Output Domain

* **Generated Content:**
* *Purpose:* Stores the draft and approved text for LinkedIn posts.
* *Relationships:* N:1 with Campaigns.
* *Indexes:* JSONB GIN index on metadata.


* **Carousel Slides & Image Prompts:**
* *Purpose:* Child entities of Generated Content specifically for multi-modal LinkedIn formats.
* *Relationships:* N:1 with Generated Content.



### 3.5. Publishing & Analytics Domain

* **Publishing Jobs:**
* *Purpose:* The queue table for sending approved content to LinkedIn.
* *Relationships:* 1:1 with Generated Content, N:1 with Publishing Platforms.
* *Indexes:* B-Tree on scheduled_time.


* **Publishing Platforms:**
* *Purpose:* Registry of supported platforms (V1: LinkedIn).


* **Analytics Reports & Content Performance:**
* *Purpose:* Stores time-series data ingested from LinkedIn APIs (impressions, clicks).
* *Relationships:* N:1 with Generated Content.



### 3.6. System & Audit Domain

* **Approvals:**
* *Purpose:* Tracks who approved what content and when.
* *Relationships:* N:1 with Generated Content, N:1 with Users.


* **Notifications & System Events:**
* *Purpose:* Tracks asynchronous system alerts to users.


* **Audit Logs:**
* *Purpose:* Immutable ledger of all state changes (especially human edits to AI content).
* *Lifecycle:* Insert-only. Never deleted.


* **Feature Flags:**
* *Purpose:* Controls rollout of new agents or DAG structures.


* **Usage Records & Billing Records:**
* *Purpose:* Tracks token usage, task execution duration, and API calls for future monetization.
* *Indexes:* Composite index (organization_id, billing_period).



---

## 4. Entity Relationships

* **One-to-Many (1:N):** The primary relationship model. Organizations own Brand Profiles; Brand Profiles own Campaigns; Campaigns own Workflow Executions; Executions own Tasks.
* **Many-to-Many (M:N):** Implemented using junction tables exclusively for `Organization_Users` to allow agencies to invite external contractors or fractional CMOs to specific workspaces.
* **Cascade Behavior:**
* **Strict RESTRICT:** Deleting an Organization is restricted if active Billing Records or Audit Logs exist.
* **Logical CASCADE:** If a Campaign is soft-deleted, its child Generated Content is marked as soft-deleted via application-level logic to preserve data integrity for the Audit Log. Hard cascades are avoided to prevent accidental catastrophic data loss.



---

## 5. Database Normalization Strategy

The database adheres strictly to Third Normal Form (3NF) for core transactional entities (Users, Organizations, Campaigns) to guarantee data integrity and eliminate redundancy.

**Exceptions for Denormalization:**

* **Task Results & Campaign Goals:** These utilize PostgreSQL `JSONB` columns. Since the schema of AI outputs is highly dynamic and hierarchical, flattening these into relational tables would require excessive JOINs and rigid schema migrations. JSONB allows for schema-on-read flexibility while maintaining transactional integrity.
* **Analytics Reports:** Denormalized to support fast read-heavy dashboard queries without joining against historical task logs.

---

## 6. Indexing Strategy

* **Primary Keys:** `UUIDv7` is mandated across all tables. UUIDv7 includes a timestamp prefix, ensuring sequential database inserts, drastically reducing page fragmentation in B-Tree indexes compared to UUIDv4, while maintaining global uniqueness.
* **Foreign Keys:** Every foreign key column will have a corresponding B-Tree index to prevent full table scans during JOINs and constraint validations.
* **Composite Indexes:** Heavy reliance on composite indexes starting with `organization_id` (e.g., `CREATE INDEX ON campaigns (organization_id, status)`). This optimizes the most common multi-tenant query patterns.
* **GIN Indexes:** Applied to `JSONB` columns (like `Task Results` and `Generated Content` metadata) to allow efficient querying of unstructured AI parameters.
* **Full-Text Indexes:** PostgreSQL `tsvector` indexes on `Generated Content` to allow users to quickly search historical posts for specific keywords.

---

## 7. Multi-Tenant Strategy

* **Tenant Isolation:** Logical isolation is enforced. Every table (except system catalogs) must contain an `organization_id` column.
* **Row Level Security (RLS):** Leveraging Supabase, RLS policies will be strictly enforced at the database level. Even if the application logic contains a vulnerability, the database will refuse to return rows where the `organization_id` does not match the authenticated user's JWT claim.
* **Future Enterprise Support:** By making `organization_id` the partition key candidate, the system is prepared for future horizontal sharding (Citus or native declarative partitioning) when scaling to enterprise volumes.

---

## 8. Workflow Persistence

Because the system orchestrates long-running AI tasks, state machine persistence is critical.

* **State Storage:** The `Workflow Executions` and `Tasks` tables store state Enums (`PENDING`, `IN_PROGRESS`, `FAILED`, `AWAITING_APPROVAL`, `COMPLETED`).
* **Checkpoints:** The `Task Results` table acts as a checkpoint. Before a task completes, its context snapshot is written. If the worker crashes, the system retrieves the last successful `Task Result` and resumes the DAG.
* **Retries:** The `Tasks` table includes `retry_count` and `last_error` columns to track exponential backoff states managed by Celery.

---

## 9. AI Memory Persistence

* **Conversation & Workflow Memory:** Stored in PostgreSQL `Task Results` (JSONB) representing the exact procedural state of a campaign run.
* **Semantic Brand Memory:** Stored in Qdrant. When a user uploads a Brand Guideline, Supabase Storage holds the PDF, PostgreSQL creates a `Knowledge Base` record mapping to the Qdrant `collection_id`, and Qdrant stores the HNSW-indexed vector embeddings for retrieval by the Validation Agent.
* **Procedural Memory:** The `Memory` table in Postgres stores explicit learnings (e.g., "User consistently deletes hashtags").

---

## 10. Audit & Compliance

* **Audit Logs:** An insert-only `Audit Logs` table records every destructive action, state change, and AI generation, tracking the `actor_id` (User or AI Agent).
* **Soft Deletes:** Standardized via a `deleted_at` timestamp on all primary entities. Queries are globally scoped to exclude `deleted_at IS NOT NULL`.
* **GDPR Compliance:** A dedicated background worker will process hard-deletion requests, scrubbing PII from Users and Organizations, but maintaining anonymized `Usage Records` for financial compliance.

---

## 11. Performance Strategy

* **Partitioning:** `Audit Logs` and `Content Performance` (Analytics) tables will utilize PostgreSQL declarative partitioning by date (e.g., monthly partitions) to ensure query speed and efficient data archival as the system scales.
* **Connection Pooling:** PgBouncer is enforced at the infrastructure level to handle the high volume of brief connections from the FastAPI worker nodes.
* **Pagination:** Keyset pagination (cursor-based using UUIDv7) is required for API endpoints retrieving Campaigns or Generated Content, preventing the performance degradation associated with deep `OFFSET` queries.

---

## 12. Backup & Disaster Recovery

* **Backup Strategy:** Supabase manages daily full backups and continuous WAL (Write-Ahead Log) archiving.
* **Restore Strategy:** Point-in-Time Recovery (PITR) enabled, allowing the database to be restored to any exact minute in the last 7 days to recover from catastrophic application bugs (e.g., an errant script mutating campaign data).
* **High Availability:** Production instances will utilize read replicas to offload heavy analytical queries (e.g., dashboard loading) from the primary writer node.

---

## 13. Future Database Evolution

The schema is designed to absorb the Phase 2 and Phase 3 roadmaps gracefully.

* **Multiple Channels:** `Publishing Platforms` and `Generated Content` are abstracted. Supporting Twitter or Newsletters simply requires adding new platform records and mapping new JSONB schemas in `Generated Content`.
* **Plugin Ecosystem:** As the system moves toward the Model Context Protocol (MCP), `Tool Connections` will expand to store dynamic OAuth scopes, and `Tasks` will log executions of generic external tools rather than hardcoded AI agents.
* **Dynamic DAGs:** Currently, workflows are hardcoded DAGs. In the future, a `Workflow Definitions` table will be introduced to store JSONB representations of dynamic graphs generated by the AI, mapping 1:N to `Workflow Executions`.

---

## 14. Recommended Migration Strategy

* **Tooling:** SQLAlchemy models will define the schema, and Alembic will generate the migrations.
* **Best Practices:**
* All migrations must be forward and backward compatible to support zero-downtime deployments.
* Destructive changes (dropping columns) require a two-phase deployment: Phase 1 ignores the column, Phase 2 drops it.
* Index creation on large tables must use `CREATE INDEX CONCURRENTLY` in Alembic to avoid locking the tables during production deployments.



---

## 15. Final Recommendations

This database architecture prioritizes **state determinism and logical isolation**. By leveraging PostgreSQL for strict relational integrity and state machine tracking, we ensure that the inherent unpredictability of LLMs is boxed into a highly rigid, auditable structure. The use of UUIDv7, RLS, and JSONB provides the perfect balance of enterprise scalability, security, and the flexibility required for rapid AI product iteration. This foundation will comfortably scale from the V1 MVP to processing millions of autonomous tasks.