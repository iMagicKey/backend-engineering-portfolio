# AgentForge — Autonomous AI Engineering and Orchestration

**Engineering focus:** Agent orchestration · Distributed task coordination · Isolated execution · Independent code review · CI/CD · Failure recovery

**Technologies represented:** Node.js, TypeScript, Bash, PostgreSQL, Fastify, Kysely, Docker, Git, GitHub API, GitHub Actions, Claude Code, SSE, systemd.

## Project context

AgentForge is a standalone, self-hosted platform for coordinating specialized AI coding agents across software-development workflows.

Its architectural direction is informed by an **implemented, project-specific agent system inside RestoVibe**. The standalone AgentForge codebase has substantial functionality of its own, but extraction of all RestoVibe mechanisms into a reusable platform is **not complete**.

The core problem is not generating code with a language model. It is coordinating multiple workers so that task ownership, source changes, review, testing, and integration remain observable, recoverable, and appropriately controlled.

This case study separates the practical RestoVibe implementation from the standalone platform's current maturity.

## Part I — RestoVibe's implemented multi-agent development infrastructure

### 1. GitHub as the shared coordination authority

The RestoVibe implementation uses GitHub Issues for work and dependencies, Pull Requests for implementation candidates, and GitHub Actions for verification evidence. Worker logs and local checkout state are diagnostics rather than canonical task truth.

An orchestrator can reconstruct the current work state from the repository and GitHub without relying on previous conversations or an undocumented server-local backlog.

Task states distinguish in-progress implementation, pending independent review, changes requested, ready-to-merge candidates, integration progress, completed work, and blocked states.

A task is not accepted merely because a worker reports success.

### 2. Deterministic task assignment and claim arbitration

**Problem:** multiple autonomous workers may discover the same available task.

The implementation supports three isolated worker slots and a deterministic claim protocol:
- Workers find eligible tasks using dependency and priority rules.
- Competing workers publish claim markers to a GitHub Issue.
- Claim arbitration uses GitHub-generated comment ordering, not a local process mutex, as its shared decision point.
- Losing workers release their claims without starting duplicate implementation.
- Heartbeats, conservative takeover rules, and durable comments support crash recovery.
- A worker holds only one implementation task at a time.

A free slot can also be dispatched as an independent reviewer when eligible review work exists.

**Trade-off:** GitHub becomes an external coordination dependency. The implementation prefers refusing ambiguous transitions to guessing state when API or CI evidence is unavailable.

### 3. Isolated worker execution

Each worker uses a separate checkout and task branch, along with namespaced local development resources such as databases, Docker Compose projects, ports, and temporary directories.

A persistent worker runtime reads GitHub state, obtains a claim, starts a headless coding-agent session, and prepares the resulting commit and Pull Request. The model is constrained to an allowed subset of tools: privileged publishing and merge operations remain in trusted runtime/orchestrator components.

The worker can resume after a restart by reconstructing its task and branch from the authoritative state. Idempotent transitions avoid recreating an already-open Pull Request.

### 4. Independent, commit-specific code review

**Problem:** an agent should not approve its own implementation or reuse outdated review evidence.

Review is executed by another agent in a fresh, restricted session, with read-only access to a snapshot of the proposed source and the relevant diff.

The review runtime enforces:
- Distinct identities for implementer and reviewer.
- Review evidence associated with a specific commit SHA.
- A new review requirement after that SHA changes.
- Structured findings with severity, source location, and a concrete failure scenario.
- Explicit outcomes: `APPROVED`, `CHANGES_REQUESTED`, or `UNCERTAIN`.

The reviewer is isolated from implementation workspaces and has restricted filesystem/tool access. This reduces the privilege available to an AI session processing potentially untrusted source content. The sandbox is a risk-reduction mechanism, not a claim of complete security.

### 5. Quality gates and controlled integration

The orchestrator evaluates dependencies, the current branch base, commit-specific CI results, review approval, and unresolved review discussions before considering integration.

When a base branch or implementation head moves, stale verification is not treated as valid for the new state.

Workers cannot merge their own Pull Requests. Trusted orchestration is responsible for advancing reviewed changes.

The project also contains a **merge train** that groups compatible approved candidates, runs validation against their combined tree, and uses guarded sequential integration and tree-identity checks. Failing combinations can be split for diagnosis rather than assuming individually green changes work together.

**Engineering principle:** evidence must correspond to the code state that is actually being accepted.

### 6. Reliability and observability

Autonomous systems face ambiguous intermediate states: a process can crash after a claim, a model response can be malformed, a worker can lose connectivity, or a reviewer can fail before publishing a verdict.

The practical infrastructure addresses these cases with bounded attempts, restart-safe state reconstruction, auditable markers, explicit `BLOCKED` conditions, backoff strategies, and operator escalation where verification cannot be established.

The repository includes simulation tests for claim races, restart recovery, stale branches, reviewer independence, merge guards, and combined CI behavior. These tests are evidence of implemented protocols, **not a guarantee of production uptime**.

## Part II — Standalone AgentForge

### 7. Reusable platform boundaries

The separate AgentForge repository generalizes orchestration away from a single product's GitHub conventions.

Its implementation includes:
- Domain models and PostgreSQL persistence.
- Declarative workflow definitions and dependency validation.
- Workflow execution and task scheduling.
- Versioned agent roles and configuration registries.
- Agent adapters with a Claude-specific implementation.
- Git operations and deterministic verification policies.
- A privileged Runtime Supervisor for container operations.
- Control-plane API and web interfaces.
- Transactional domain events and Server-Sent Events with replay.
- Review findings, bounded correction loops, and human-intervention workflows.

The platform deliberately separates orchestration from privileged runtime execution. A Fastify control plane handles domain and workflow decisions; dedicated components own container operations; model-provider-specific details stay behind adapters.

This design reduces coupling between AI vendors, Git operations, business rules, and infrastructure privileges.

### 8. Declarative engineering workflow

One example workflow expresses the following dependency-driven stages:

`Intake → Requirements Review → Planning → Architecture/Product Review → Implementation → Correctness Review → Integration → Retrospective`

These stages describe the intended composition of specialized agents. The standalone code includes workflow execution and validation mechanisms, but the existence of a declarative definition is not itself proof that every stage has been run successfully end-to-end with a live model.

### 9. Transactional events and recovery

In AgentForge, a domain mutation and its associated event records are persisted in the **same PostgreSQL transaction**. A post-commit dispatcher notifies subscribers; persisted events remain the recoverable record.

This supports idempotent APIs and SSE clients that replay from a known event cursor after reconnecting.

A runtime scheduler, lease handling, review gates, and reconciliation logic provide additional recovery and consistency boundaries.

### 10. Current maturity

**RestoVibe agent subsystem:** implementation and simulation coverage exist for coordination, coding-agent runtime, independent review, guarded integration, and merge-train behavior. Current deployment state and operational reliability would require live environment evidence.

**Standalone AgentForge:** the repository documents a v0.1 release candidate, with substantial domain, runtime, policy, API, and workflow code and automated tests. A real Claude session-resumption scenario has been demonstrated. The full real-agent intake-to-merge lifecycle and first production rollout are still listed as operator follow-up work.

**Relationship between the projects:** RestoVibe is the practical project-specific agent system; AgentForge is the separate generalization effort. These are related engineering efforts, not an already completed code migration.

## Engineering decisions demonstrated

- Reconstructable coordination state instead of undocumented local memory.
- Deterministic ownership under concurrent worker claims.
- Role and permission separation between implementation, review, and merging.
- Commit-bound review and CI evidence.
- Controlled integration of concurrent changes.
- Explicit failure states, bounded retries, and observability.
- Modular platform architecture with domain, workflow, adapter, and runtime boundaries.

## Authorship and disclosure

This work is **human-directed and AI-assisted**. I defined architectural direction, operational constraints, expected behavior, review requirements, and design trade-offs; AI coding agents assisted in implementing and iterating on the systems.

This case study is a technical overview, not a reproduction of private repositories. It avoids proprietary source code, credentials, internal deployment identifiers, customer data, and confidential operational settings.
