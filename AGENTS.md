# AGENTS.md

## Project Overview

This repository implements an internal browser-based application for producing Trainer/Assessor Skills Matrices.

The system must:
- load qualification and unit data from training.gov.au using the best currently available authoritative interface
- persist normalized qualification snapshots for reproducibility and superseded records
- ingest evidence documents
- extract text using native parsing first and OCR second
- map evidence to qualification units using deterministic rules
- generate editable matrix draft content
- export a final DOCX that matches the approved compliance template

This is an MVP for controlled internal use. Prioritize correctness, traceability, reproducibility, and export fidelity over feature breadth or clever abstractions.

## Architecture Boundaries

The repo is a monorepo with three runtime services:

- `apps/web`: Next.js + TypeScript UI
- `apps/api`: FastAPI service for orchestration and business APIs
- `apps/worker`: Python worker for OCR, extraction, mapping, drafting, and export jobs

Shared code belongs in packages, not copied across services.

Recommended layout:

```text
/apps
  /web
  /api
  /worker
/packages
  /domain
  /source-adapter
  /shared-types
  /docx-template
  /config
/sql
  /migrations
  /seeds
/test-fixtures
  /qualifications
  /evidence
  /exports
Service Ownership
apps/web

Owns:

user-facing workflow screens
qualification load initiation
elective selection UI
evidence upload flow
review grid and row editing
export trigger and export status display

Must not own:

qualification normalization logic
mapping rules
OCR logic
DOCX rendering logic
direct database access
apps/api

Owns:

request validation
orchestration
transaction boundaries
source adapter invocation
persistence coordination
presigned upload/download URL generation
enqueueing worker jobs
project-level validation and export blocking checks

Must not own:

heavy OCR execution
long-running extraction or mapping work
DOCX rendering internals beyond orchestration
apps/worker

Owns:

native text extraction
OCR
candidate unit extraction
mapping engine execution
column drafting execution
DOCX rendering
retry-safe job execution

Must not own:

UI concerns
HTTP orchestration decisions
mutable business rules defined only in ad hoc code without shared domain types
Source Adapter Rules

training.gov.au is the authoritative source domain.

Implementation rule:

use the best currently available authoritative integration surface now
prefer REST once the REST API provides the required qualification, unit, packaging, and supersession data
until then, use SOAP where it provides better completeness

The source adapter must be isolated in packages/source-adapter.

The rest of the codebase must never depend directly on SOAP or REST payload shapes.

Adapter responsibilities:

fetch qualification by code
fetch packaging rules
fetch unit details
fetch unit elements
fetch supersession/equivalence relationships
normalize into canonical internal models
attach provenance metadata to every snapshot

Rules:

snapshots are immutable once stored
persist source type per snapshot: rest, soap, or approved fallback
never silently merge conflicting source records
when source data is incomplete, fail explicitly with typed errors
Data and Storage Rules

Primary persistence:

PostgreSQL for system state
S3-compatible object storage for evidence, templates, source cache payloads, and DOCX exports

Rules:

no direct file-system persistence for durable business files
no hard delete of mappings, export jobs, or source snapshots in MVP
all important entities must have created/updated timestamps
schema changes must go through SQL migrations
object keys must be deterministic and environment-safe

Evidence storage rules:

upload through presigned URLs
validate MIME type and size before finalize
calculate checksum when feasible
preserve original uploaded file
Mapping Rules

Mapping must be deterministic and explainable.

Allowed statuses:

exact_match
superseded_equivalent
superseded_nonequivalent
experience_supported
needs_review

Decision order:

exact match
official equivalent
official non-equivalent supersession result
experience-supported rule
needs review

Rules:

never auto-upgrade non-equivalent to equivalent
every mapping must store reason code, confidence, and supporting snippet
manual override must preserve prior mapping history
unresolved rows must block export when required by the rules
OCR and Extraction Rules

Extraction pipeline order:

native text extraction when the document already contains text
OCR fallback only when needed

OCR must run on internal infrastructure.

Rules:

prefer deterministic parsing over AI interpretation
preprocess scanned pages before OCR
store extracted text and extraction metadata
surface extraction failure clearly
do not hide unreadable evidence behind fake success states
Drafting Rules

Column drafting must be structured and low-variance.

Rules:

generate editable drafts, not final unquestionable answers
use evidence snippets, trainer profile, and unit metadata as inputs
prefer templated composition over free-form generation
if required support is missing, leave the row blocked rather than inventing content
DOCX Export Rules

DOCX export must preserve the approved template behavior.

Rules:

render from persisted matrix rows only
do not render directly from transient UI state
preserve formatting constraints from the spec
generate validation output with blocking issues and warnings
exported documents must be reproducible from stored project state
API and Domain Rules
define request/response models explicitly
keep shared enums and domain types centralized
validate all external input at the API boundary
use typed error codes, not vague string failures
keep transaction boundaries inside the API service
worker jobs must be idempotent
Testing Expectations

Every meaningful change should include the narrowest tests that prove it works.

Minimum expectations:

unit tests for pure domain logic
integration tests for database and storage flows
fixture-based tests for source adapter normalization
regression tests for mapping outcomes
golden-file or structural tests for DOCX formatting-sensitive output when export logic changes
Definition of Done

A task is only done when:

code compiles or runs cleanly
tests for the changed behavior exist and pass
types and schemas are updated consistently
migrations are included when persistence changes
logs and errors are meaningful
no architecture boundary is violated
no hidden manual step is required for the feature to work
MVP Out of Scope

Do not introduce these unless explicitly requested:

multi-tenant architecture
collaborative editing
complex role-based access control
broad user-configurable export templates
autonomous mapping without review
replacing deterministic rules with opaque AI-only decisions
Working Style for Codex

When implementing tasks in this repository:

make the smallest correct change that satisfies the task
do not refactor unrelated code opportunistically
preserve existing boundaries between web, api, worker, and shared packages
prefer explicitness over cleverness
when uncertain, choose traceability and maintainability over speed
add TODOs only when they are specific, actionable, and justified
keep acceptance criteria visible in code comments or tests where helpful
Preferred Task Format

Implementation tasks should specify:

objective
files or modules likely to change
constraints
acceptance criteria
out-of-scope notes

This file is the standing instruction layer for all repository work unless a task explicitly overrides a narrow detail.

After you save that, I’ll send the full contents for `docs/tickets/E1-T1-monorepo-scaffold.md` in one block too.
