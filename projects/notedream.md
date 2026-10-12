# Notedream — Reliable Voice Processing & AI-Assisted Journaling

**Project status:** Active development with real users (confirmed by the project maintainer)  
**Role:** Backend Architecture · Technical Direction · AI-Assisted Development  
**Engineering focus:** Node.js · Asynchronous Workflows · Reliability · Idempotency · PostgreSQL · AI Integration · Data Isolation

## Overview

Notedream is a Telegram-first dream journal and lucid-dreaming practice application. Users can record dreams as text or voice messages, review and manually edit structured AI-assisted interpretations, explore recurring dream signs and mood patterns, and complete daily exercises.

The central backend challenge is coordinating input capture, local audio transcription, and external AI processing **without losing work or overwriting newer user decisions**. Background jobs can be retried, processes can restart, and AI responses can arrive after the original content has changed.

The product is under active development and already has real users, as reported by the project maintainer. No user counts or usage metrics are claimed.

## My role and engineering approach

I defined the product goals, core requirements, backend architectural direction, component boundaries, and expected reliability behavior. I developed the system collaboratively with AI coding agents, which assisted with implementation details, edge cases, testing, and iterative refinement.

My engineering contribution focuses on directing architectural choices, reasoning about failure modes, reviewing the resulting design, and making the components work as a coherent application. This is **human-directed, AI-assisted development**, not a claim that every source file was written manually.

## System architecture

Notedream is a **process-separated modular Node.js application** in a monorepo. It is operated as one supervised deployment rather than independently deployed microservices.

The runtime has four cooperating processes:

- **Telegram bot:** receives text and voice submissions and communicates with the API rather than connecting directly to the database.
- **API:** authenticates users, handles journal and domain operations, manages transactions, and processes the asynchronous AI-analysis outbox.
- **Web application:** provides the Next.js user interface and frontend shell.
- **ASR worker:** handles queued transcription workloads, including audio conversion and calls to local speech recognition.

**PostgreSQL** is the system of record for identities, journal entries, transcription versions, AI-processing state, and domain information. **Redis/BullMQ** powers the speech-recognition job queue. **S3-compatible object storage** stores uploaded audio, and **whisper.cpp** handles local speech recognition.

**Trade-off:** separate runtime responsibilities keep resource-intensive transcription apart from ordinary API requests, while a shared deployment is simpler to operate. The deployment remains a single supervised stack, not independently scalable microservices.

## 1. Durable voice-to-text processing

The voice-processing workflow is:

`Telegram voice → authenticated upload → object storage → PostgreSQL outbox → BullMQ → FFmpeg → whisper.cpp → transcript → AI analysis`

The Telegram bot receives audio and sends it to an authenticated API upload flow. Short-lived, single-use upload tokens and bounded input validation constrain submissions.

After the audio object is stored, a **voice draft and transcription-outbox event are committed together in PostgreSQL**. An outbox dispatcher publishes the task to BullMQ. The ASR worker then retrieves the audio, uses **FFmpeg to convert the audio format**, and calls a local **whisper.cpp** service to recognize speech.

The transcription stage uses bounded per-process concurrency, stable job identifiers, retry/backoff behavior, and idempotent database persistence.

**Engineering decision:** the system accepts **at-least-once job processing** while protecting persistence from duplicate effects. A stable queue job ID is useful for deduplication, but does not imply exactly-once execution.

## 2. Transactional outbox and recovery

Publishing directly to a queue immediately after committing a database entry creates a failure window: the process can crash between those two operations and leave a saved entry with no work scheduled.

The system uses a **transactional outbox** so the new voice draft and the instruction to process it are committed in the same database transaction. A separate dispatcher can retry queue publication.

Audio storage is a separate external system and cannot participate in the PostgreSQL transaction. The upload workflow also handles **ambiguous database commit outcomes** by checking durable state before compensating a failed upload, rather than accidentally deleting audio associated with a committed entry.

This is a practical example of handling dual-write boundaries and partial failures without claiming a distributed transaction across storage, PostgreSQL, and Redis.

## 3. Reliable, version-aware AI analysis

A second, **PostgreSQL-backed outbox** drives asynchronous structured AI analysis through a provider adapter. Unlike transcription, this queue is processed within the API process.

The analysis workflow uses:

- durable processing attempts and single-flight constraints to limit duplicate paid AI calls;
- leases, version checks, and database fencing to reject outdated results;
- bounded retries, cooldowns, and a provider-level circuit breaker;
- input/output limits and normalization of structured model responses;
- recorded source artifacts and versions to match the AI result to the correct journal content.

The AI analysis extracts structured characteristics such as themes, emotional tone, dream signs, and lucidity-related fields. Its output is application data, **not a medical or psychological diagnosis**.

## 4. Protecting manual edits from late AI responses

A user may modify a dream or correct its AI-generated analysis while a background model request is still running.

Notedream handles this as a concurrency problem:

- Source-version checks prevent an AI result based on an older text revision from being applied to a newer entry.
- A user-edit transaction marks an analysis as **manually owned** and cancels pending work where applicable.
- The final AI write is guarded by a database condition so a delayed model response cannot silently replace manually edited fields.

**Engineering principle:** user-confirmed changes remain authoritative over asynchronous AI-generated suggestions unless the user explicitly requests renewed processing.

## 5. Data isolation and domain modeling

Notedream deals with highly personal journal content. The backend includes security and isolation mechanisms such as:

- Telegram authentication verification with freshness and replay checks.
- Bearer sessions with hashes stored in the database.
- **PostgreSQL Row-Level Security** for user-owned records, alongside transaction-scoped user context.
- Restricted runtime database roles and separation of infrastructure credentials between processes.

The domain layer separates probabilistic AI extraction from deterministic application rules. A recurring-pattern engine identifies dream signs using recorded journal activity and statistical safeguards rather than counting missing journal days as negative evidence. The application also implements reports, daily tasks, progression points, and streak tracking.

These features support personal reflection and lucid-dreaming practice; the case study makes no claim of clinical effectiveness.

## 6. Operational and testing considerations

Operational tooling in the repository includes supervised process startup, readiness sequencing, graceful shutdown handling, versioned database migrations, health checks, and PostgreSQL backup/restore utilities.

The architecture includes automated tests for workflows such as authentication, repository behavior, concurrent operations, queue processing, AI integration, and domain rules. A source-code audit identified **160 test files and 897 test cases**; architecture and type checks passed during that audit. The complete test suite was not executed as part of that read-only review, so no green-suite or coverage-percentage claim is made.

## Selected engineering decisions

- **Transactional outbox instead of fire-and-forget dispatch:** maintains a durable processing request alongside the stored journal entry.
- **At-least-once processing with idempotent persistence:** treats retries as a normal property of background work.
- **Local speech recognition with bounded concurrency:** separates resource-intensive ASR from synchronous API traffic.
- **Durable attempts and circuit breaking for paid AI calls:** limits duplicate requests and uncontrolled retry behavior.
- **Source-version fencing and manual-edit protection:** prevents stale AI results from overriding authoritative user data.
- **PostgreSQL RLS and restricted roles:** supports user isolation at both application and database boundaries.

## Technology stack

Node.js, JavaScript/TypeScript, PostgreSQL, Redis, BullMQ, S3-compatible object storage, FFmpeg, whisper.cpp, Next.js, React, Telegram Bot API, REST, SQL, Mocha/Chai, Vitest, systemd.

## Scope and disclosure

This is an architectural overview, not a public release of the product's source code. It omits private journal content, audio samples, customer identifiers, credentials, production endpoints, and proprietary AI-provider details.

Architecture and implementation details are based on a read-only technical audit of the codebase; the existence of real users is confirmed by the project maintainer. No unverified traffic, availability, performance, or revenue figures are stated.
