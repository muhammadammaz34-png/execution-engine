You are a Principal Software Architect at Google, a Staff Engineer at OpenAI, and an AI Infrastructure Architect specializing in enterprise AI systems.

You have received the following documents:

1. Vision Document
2. Product Requirements Document (PRD)

Treat them as the single source of truth.

Your task is to design the complete technical architecture for Version 1 of the product.

==========================
PROJECT
==========================

Product:

Autonomous Marketing Operations Engine

Purpose:

A user gives one business goal.

Example:

"Launch a LinkedIn campaign about Kubernetes security."

The platform autonomously executes the workflow by coordinating multiple AI agents.

Version 1 focuses only on LinkedIn marketing for small marketing agencies.

==========================
SYSTEM REQUIREMENTS
==========================

The architecture should support:

• Multiple AI Agents

• Workflow orchestration

• Long-running workflows

• Human approval

• Retry mechanism

• Validation

• Tool integrations

• Memory

• Analytics

• Future enterprise scaling

==========================
TECH STACK
==========================

Frontend

Next.js
TypeScript
Tailwind CSS

Backend

FastAPI (Python)

Database

PostgreSQL

Cache

Redis

Queue

Celery + Redis

LLMs

OpenAI
Google Gemini
Anthropic Claude

Vector Database

Qdrant

Storage

Supabase Storage

Authentication

Clerk

Deployment

Frontend → Vercel

Backend → Railway or Render

Database → Supabase PostgreSQL

Monitoring

Langfuse

Sentry

OpenTelemetry

==========================
ARCHITECTURE REQUIREMENTS
==========================

Create a complete technical architecture document.

Include the following sections.

# 1 Executive Summary

High-level explanation of the system.

------------------------------------------------

# 2 High-Level Architecture

Explain every layer.

Presentation Layer

API Layer

Execution Layer

AI Layer

Data Layer

Integration Layer

Infrastructure Layer

------------------------------------------------

# 3 System Diagram

Generate a Mermaid diagram.

------------------------------------------------

# 4 Frontend Architecture

Folder structure

Pages

Components

State Management

Routing

Authentication Flow

------------------------------------------------

# 5 Backend Architecture

Folder structure

Modules

Services

Controllers

Dependency Injection

Task Queue

------------------------------------------------

# 6 Execution Engine

Explain in detail.

Planner

Workflow Manager

Task Scheduler

Agent Router

Execution Context

Retry Logic

Recovery Logic

Checkpointing

Human Approval

------------------------------------------------

# 7 AI Agent Architecture

Research Agent

Strategy Agent

Copywriter Agent

Carousel Agent

Image Prompt Agent

Validator Agent

Publisher Agent

Analytics Agent

Explain:

Responsibilities

Inputs

Outputs

Memory usage

Communication

------------------------------------------------

# 8 Memory Architecture

Conversation Memory

Campaign Memory

User Memory

Brand Memory

Knowledge Base

Vector Search

------------------------------------------------

# 9 Database Architecture

High-level database overview.

Explain every major entity.

(No SQL yet.)

------------------------------------------------

# 10 API Architecture

REST APIs

Authentication

Rate Limiting

Versioning

Error Handling

------------------------------------------------

# 11 Security Architecture

Authentication

Authorization

Secrets

Encryption

Prompt Injection Protection

Audit Logs

------------------------------------------------

# 12 Scalability Strategy

Horizontal Scaling

Background Workers

Caching

Queues

Load Balancing

------------------------------------------------

# 13 Observability

Logging

Tracing

Metrics

Monitoring

Alerts

------------------------------------------------

# 14 Deployment Architecture

Development

Staging

Production

CI/CD

------------------------------------------------

# 15 Recommended Folder Structure

Complete folder tree.

------------------------------------------------

# 16 Technology Decisions

Explain WHY every technology was chosen.

------------------------------------------------

# 17 Future Evolution

How this architecture grows into a multi-domain autonomous execution platform.

==========================
OUTPUT REQUIREMENTS
==========================

Produce a professional Technical Architecture Document.

Enterprise quality.

Markdown.

Very detailed.

Implementation ready.

No placeholders.

No emojis.

No generic explanations.

Every decision must be justified.

This document should be strong enough that a senior engineering team could begin implementation immediately.