# 🧠 Ohara
### Your Personal AI Second Brain

> **Capture everything useful. Understand it. Connect it. Act on it. Keep it accessible.**

Ohara is a planned, privacy-conscious personal AI productivity and knowledge system. It is designed to capture information from everyday life, understand it with local AI, organize it into useful structures, and make it searchable and accessible across devices.

The goal is to give tasks, ideas, projects, reminders, voice notes, documents, links, and personal knowledge one connected home—without depending on a continuously running cloud AI server.

> **Project status (2026-11-13):** Product requirements and high-level architecture are approved. Implementation, repository structure, database schema, tests, and deployment have **not yet been verified**. The technology choices below are proposed directions, not claims about an existing implementation.

---

## Contents

- [What Ohara is](#what-ohara-is)
- [Core principles](#core-principles)
- [Product capabilities](#product-capabilities)
- [Architecture at a glance](#architecture-at-a-glance)
- [Capture and processing lifecycle](#capture-and-processing-lifecycle)
- [Technology direction](#technology-direction)
- [Hardware and compute plan](#hardware-and-compute-plan)
- [Data domains](#data-domains)
- [Data retention rules](#data-retention-rules)
- [Privacy, security, and reliability](#privacy-security-and-reliability)
- [Backup and recovery plan](#backup-and-recovery-plan)
- [Development roadmap](#development-roadmap)
- [Current project status](#current-project-status)
- [Documentation and development workflow](#documentation-and-development-workflow)

---

## What Ohara is

Ohara is intended to work as a personal second brain with three connected responsibilities:

1. **Capture:** Collect information quickly from a phone or web interface.
2. **Understand and organize:** Use AI to identify what the input represents and turn it into structured, editable information.
3. **Recall and act:** Connect related knowledge, answer questions from saved information, surface reminders, and support approved workflows.

Examples of information Ohara should handle:

- Tasks, deadlines, reminders, and recurring activities.
- Projects, milestones, decisions, and progress.
- Ideas, plans, and opportunities.
- Personal memories and useful context.
- Research links, articles, repositories, tutorials, and documents.
- Voice notes, images, screenshots, and mixed inputs.

The system is designed around **cloud-accessible data with user-owned local AI processing**. A local worker processes queued inputs when a supported machine is online. Previously synchronized information remains accessible through the application even when that worker is offline.

## Core principles

| Principle | Product requirement |
|---|---|
| **Capture first, organize second** | Never lose an input just because classification or extraction failed. |
| **Cloud is the source of truth** | Supabase is the planned central store for synchronized application data. |
| **Local AI processing** | Use the user's own machines for inference and background AI work; rented cloud compute is not currently desired. |
| **User remains in control** | AI-generated information must be editable; consequential actions require approval. |
| **Preserve valuable knowledge** | Ideas, research, and project history must not be removed by routine cleanup. |
| **Reliable over flashy** | Prioritize data integrity, retries, backups, clear states, and usable UX. |
| **Replaceable AI components** | Keep model and provider integrations modular so they can be changed without redesigning the whole product. |

## Product capabilities

These are approved product requirements. They are **planned scope**, not proof that the modules have been implemented.

### Core modules

1. **Universal Capture Inbox** — Capture text, voice, images, screenshots, documents, links, and mixed inputs.
2. **AI Brain / Intelligent Organizer** — Classify, extract, summarize, connect, and organize captured information.
3. **Project Workspace** — Track project details, tasks, milestones, decisions, resources, and progress.
4. **Smart Tasks & Timeline** — Manage personal tasks, project tasks, deadlines, scheduled work, and recurring tasks.
5. **Personal Memory** — Preserve useful, user-approved personal context, people, dates, and details.
6. **Research & Knowledge Library** — Store and organize links, articles, repositories, tutorials, documents, and notes.
7. **AI Daily Briefing** — Summarize upcoming tasks, deadlines, and project updates.
8. **AI Search / Ask Your Second Brain** — Search and answer questions using saved personal information.
9. **Smart Notifications & Reminders** — Support time-based, deadline-based, recurring, and relevant alerts.
10. **Automation & Agentic Capabilities** — Run approved workflows and monitor relevant activity.

### Additional intelligence features

- **Idea evolution tracking:** Maintain the history of how an idea changes.
- **Idea scoring and opportunity analysis:** Help evaluate potential, effort, value, and feasibility.
- **Duplicate and related idea detection:** Find similar records and suggest connections.
- **Project health monitoring:** Surface inactivity, risks, blockers, and overdue work.
- **Knowledge graph:** Connect projects, ideas, people, tasks, research, and concepts.

### Planned application navigation

- Dashboard
- Capture Inbox
- Timeline & Tasks
- Projects
- Ideas
- Personal Memory
- Knowledge Library
- AI Search
- Automations
- Settings

The initial interface should remain simple, practical, and responsive, with mobile capture and review treated as important use cases.

## Architecture at a glance

### Cloud-first storage, local AI processing

Supabase is the planned central data store and synchronization point. The user's MacBook—and later, optionally, a desktop PC—runs the local processing worker and AI models.

```text
┌─────────────────────────────┐
│        PHONE / WEB APP      │
│   Capture · Search · Tasks  │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│        SUPABASE CLOUD       │
│                             │
│  PostgreSQL database        │
│  Supabase Storage            │
│  Authentication + RLS       │
│  Pending processing records │
│  Processed user data        │
└──────────────┬──────────────┘
               │ Pending input sync
               ▼
┌─────────────────────────────┐
│         LOCAL WORKER        │
│      MacBook / Desktop      │
│                             │
│  Fetch → Validate           │
│  Transcribe → Understand    │
│  Classify → Extract         │
│  Persist → Sync results     │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│          LOCAL AI           │
│  Ollama + local models      │
│  Speech-to-text             │
│  Embeddings / retrieval     │
└─────────────────────────────┘
```

### Three logical layers

| Layer | Responsibility |
|---|---|
| **Application** | User interface, authentication, APIs, database operations, tasks, projects, reminders, and user-facing workflows. |
| **AI Intelligence** | Transcription, classification, extraction, summarization, context linking, search, and recommendations. |
| **Automation** | Queue processing, scheduling, retries, notifications, cleanup, and approved agent workflows. |

### Offline behaviour

When the local AI machine is offline:

- The phone/web app must still access data already synchronized to Supabase.
- New inputs must still be accepted and persisted in the cloud.
- Those inputs remain visibly pending until a worker is available.
- AI processing is delayed; the application must not present pending work as completed.
- When a worker reconnects, it fetches and processes pending inputs, then syncs the results to Supabase.

A worker outage must delay processing, **not cause data loss or block access to already-synchronized records**.

## Capture and processing lifecycle

The expected processing flow is:

```text
CAPTURED
   ↓
STORED IN CLOUD
   ↓
PENDING PROCESSING
   ↓
FETCHED BY LOCAL WORKER
   ↓
VALIDATED
   ↓
AI PROCESSING
   ↓
STRUCTURED RESULT PERSISTED
   ↓
RESULT SYNCED TO CLOUD
   ↓
SOURCE CLEANUP, ONLY WHEN ALLOWED
   ↓
COMPLETED
```

The implementation must use:

- Stable input and record IDs.
- Explicit processing states and visible errors.
- Idempotency to prevent duplicate records on repeated delivery.
- Retry-safe jobs and recoverable failures.
- Persistence verification before deleting any source data.
- Confidence/review metadata for uncertain AI extractions.

## Technology direction

The following stack was approved as the **proposed direction**. Exact versions are deliberately not pinned. Verify current stable releases, official documentation, and compatibility before implementation.

| Area | Proposed technology | Purpose / note |
|---|---|---|
| Web frontend | Next.js + TypeScript | Responsive application |
| Styling | Tailwind CSS | Layout and styling |
| UI components | shadcn/ui | Reusable accessible components |
| Backend API | Python + FastAPI | Application and worker-facing APIs |
| Database | Supabase PostgreSQL | Central source of truth |
| ORM | SQLAlchemy | Python database access |
| Authentication | Supabase Auth | User authentication |
| Object storage | Supabase Storage | Persistent files and temporary audio |
| Local LLM runtime | Ollama | Local language and embedding models |
| Speech-to-text | faster-whisper | Local transcription |
| Embeddings | Local embedding model | Semantic retrieval |
| Vector search | PostgreSQL pgvector / Supabase vector support | Design to be verified |
| Background processing | Python worker | Local queue processing |
| Scheduler | APScheduler initially, or another verified fit | Local scheduled jobs |
| Local cache | SQLite, only if useful | Optional cache/offline support |

### Technical constraints

- Do not add Redis, paid queue infrastructure, or rented compute without a demonstrated need.
- Avoid competing sources of truth; Supabase remains the planned central store.
- Prefer a PostgreSQL-backed queue initially if it meets reliability needs.
- Keep model integrations behind replaceable interfaces.
- Never expose Supabase service-role credentials in frontend code.
- Record exact dependency versions in the repository after they have been verified and installed.

## Hardware and compute plan

| Machine | Supplied configuration | Planned role |
|---|---|---|
| MacBook Air | Apple M5, 24 GB unified memory | Initial development machine and local AI worker |
| Desktop PC | NVIDIA RTX 4080, Intel i9-13900K, 32 GB RAM; VRAM reported as approximately 12–16 GB | Optional heavier AI worker in a later phase |

No rented cloud AI compute is currently planned. Cloud database and object storage are acceptable. Start with the MacBook; add desktop worker coordination only after the core pipeline is reliable.

Hardware details are planning inputs and should be rechecked before performance tuning.

## Data domains

The following are conceptual data domains. They are **not a finalized database schema**.

- **User:** Account identity, settings, timezone, and preferences.
- **Capture Inbox:** Original input metadata, source, timestamps, and processing state.
- **Attachments:** Image/document metadata and storage references.
- **Tasks:** Title, details, task type, due date, completion state, and relationships.
- **Projects:** Project details, status, milestones, linked tasks, and resources.
- **Ideas:** Original idea, evolution, scoring, relationships, and implementation status.
- **Research:** URLs/documents, sources, notes, extracted information, and project links.
- **Personal Memory:** User-approved facts, people, dates, and context.
- **AI Processing:** Job state, model, output, confidence, errors, and retry metadata.
- **Reminders:** Trigger time, timezone, recurrence, and delivery state.
- **Knowledge Links:** Relationships between ideas, projects, tasks, people, and research.
- **Backups:** Manifest, timestamp, backup type, integrity result, and restore metadata.

Final table names, fields, constraints, indexes, relationships, migrations, and Row Level Security policies must be designed after repository and schema inspection.

## Data retention rules

These rules are explicit product requirements and must be enforced by the application, worker, and scheduled cleanup logic.

| Data type | Required behaviour |
|---|---|
| **Personal one-time tasks** | After completion, keep in a temporary archive for **7 days**, then permanently delete. |
| **Project-related tasks** | Preserve completed tasks in project history. Do not move them to the personal-task archive or automatically delete them. |
| **Original voice recordings** | Delete immediately after transcription and extracted information have been successfully verified and persisted. |
| **Failed voice processing** | Retain the original audio so processing can be retried. |
| **Transcriptions** | Preserve useful verified transcript/extracted information as structured records according to its data type. |
| **Ideas and research** | Preserve indefinitely; the 7-day task cleanup must never apply to them. |
| **Implemented ideas** | Keep the record and update its status to `Implemented`. |
| **Completed research / implementation** | Keep the record and mark it `Done` where appropriate. |

**Important:** The seven-day cleanup applies only to completed personal one-time tasks. It is not a general data-retention policy.

## Privacy, security, and reliability

The implementation must provide:

- Supabase Auth for authentication.
- Row Level Security (RLS) for all user-owned data exposed through Supabase.
- Explicit ownership checks for every data mutation.
- Secure secret handling; service-role keys must never enter frontend bundles or logs.
- File type and size validation for uploads.
- Restricted access to temporary audio and private documents.
- Secure transport and suitable encryption at rest.
- Careful logging that avoids raw voice recordings, sensitive personal content, and credentials.
- Export and recovery paths for user data.
- Retry-safe background jobs and auditable cleanup.
- Additive, reversible database migrations where practical.
- A verified backup and rollback plan before destructive data changes.

AI must not invent missing facts or dates. Ask for essential missing information, make calculated dates explicit, and use the user's applicable timezone. AI-generated records must be editable through text, voice/dictation, or manual UI. Risky, external, or irreversible actions require explicit user approval.

## Backup and recovery plan

The approved direction is to create automated local backups on the MacBook.

| Setting | Planned value |
|---|---|
| Frequency | Once every 24 hours |
| Preferred schedule | Approximately 2:00 AM local time |
| Destination | Dedicated MacBook backup directory |
| Retention | Latest **30 valid daily backup sets** |
| Offline behaviour | Catch up when the MacBook next becomes available |
| Validation | Integrity checks and a backup manifest |
| Recovery | Periodic restore tests |

A backup set must cover:

1. Supabase PostgreSQL database export.
2. Persistent Supabase Storage objects, including retained images and documents.
3. Metadata/manifests needed to map stored objects to database records.
4. Backup logs and integrity results.

A database dump alone does not back up the actual Storage objects. Do not delete older valid backups until the new backup has passed validation. Encrypt backup files at rest where practical, and consider an additional encrypted copy on the desktop or an external drive.

The schedule and retention values are approved plans; backup automation and restore capability are **not yet verified as implemented**.

## Development roadmap

The development strategy is incremental. Timelines are estimates and should be revised after repository inspection.

| Phase | Scope | Status |
|---|---|---|
| **Phase 0 — Requirements & architecture** | Product scope, architecture direction, retention rules, UI navigation, and roadmap | Decisions approved |
| **Phase 1 — Working MVP** | Responsive shell, auth, capture, cloud inbox, local worker, local AI processing, basic tasks/projects/reminders, and retention enforcement | Pending |
| **Phase 2 — Functional product** | Advanced timeline, personal memory, knowledge library, semantic/hybrid search, notifications, briefings, and richer project workspace | Pending |
| **Phase 3 — Professional polish** | Security/RLS review, backup and restore automation, performance, mobile UX, logging, monitoring, and recovery | Pending |
| **Phase 4 — Advanced AI ecosystem** | Knowledge graph, idea scoring/evolution, duplicate detection, project health, agentic workflows, and multi-machine compute coordination | Future |

### First implementation milestone

Build and verify this end-to-end flow before expanding into advanced AI features:

**Phone/Web capture → Supabase persistence → pending queue → local worker → local AI processing → verified structured result → Supabase sync → visible across devices.**

The MVP is acceptable only when pending inputs survive worker downtime/failure, processing is retry-safe, results synchronize across devices, and the retention rules are enforced correctly.

## Current project status

**As of 2026-11-13:**

- ✅ Product vision and core principles agreed.
- ✅ Product modules and future capability scope agreed.
- ✅ Cloud-first storage / local AI processing architecture approved.
- ✅ Proposed technology direction selected, subject to version and compatibility checks.
- ✅ Data retention rules finalized.
- ✅ AI clarification, editability, and approval requirements defined.
- ✅ Initial navigation and design direction agreed.
- ✅ Backup strategy approved as a plan.
- ⏳ Repository and code implementation not verified.
- ⏳ Database schema and migrations not verified.
- ⏳ Frontend, backend, and local worker not verified.
- ⏳ Automated tests, deployment, backups, and restore process not verified.

Do not describe approved requirements as completed software. Update this section only when there is concrete implementation evidence.

## Documentation and development workflow

### `STATUS.md` — detailed project source of truth

`STATUS.md` is the authoritative project tracker and AI/developer handoff document. It contains:

- Detailed requirements and architecture decisions.
- Full module scope and backlog identifiers.
- Data retention and backup requirements.
- AI behaviour and safety rules.
- Roadmap, acceptance criteria, and required test scenarios.
- Decision log, session log, open verification items, and next-session instructions.

Read `STATUS.md` before making significant architectural or implementation changes.

### Development workflow

1. Inspect the real repository, current branch, code, dependencies, environment, and database schema.
2. Compare the implementation with the approved requirements in `STATUS.md`.
3. Verify current stable versions and official documentation before adding or upgrading dependencies.
4. Plan changes and migrations before modifying data or architecture.
5. Implement in small, testable steps, prioritizing the capture pipeline.
6. Run relevant checks and tests; record exact commands and actual outcomes.
7. Update `STATUS.md` after each meaningful milestone with files changed, evidence, test results, remaining issues, and the next task.

### Setup and run instructions

**Not yet documented.** The repository structure, installed dependencies, environment variables, and executable commands have not been inspected or verified. Add accurate installation, configuration, migration, worker, and run commands once the initial repository implementation exists. Do not use guessed commands as project instructions.

---

*Ohara — Capture everything useful. Understand it. Connect it. Act on it. Keep it accessible.*
