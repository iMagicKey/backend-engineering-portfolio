# RestoVibe — Multi-Tenant Restaurant SaaS Architecture

**Engineering focus:** Node.js backend architecture · PostgreSQL multi-tenancy · Data isolation · Regional placement · Cluster monitoring · Offline-first design

**Stack represented in the project:** Node.js 24, TypeScript, PostgreSQL, Drizzle ORM, SQLite, Next.js, React, Docker, GitHub Actions.

## Project context

RestoVibe is a restaurant SaaS platform designed around customer administration, catalog and availability, point-of-sale workflows, kitchen operations, integrations, and regional expansion.

The backend is organized as a **modular monolith** with distinct API and background-worker roles. A central control database manages platform identity, tenant registry, region and database-cluster metadata; customer business data belongs to separate tenant databases. The design deliberately favors clear domain and data boundaries over introducing microservices prematurely.

This case study focuses on architectural decisions, implemented backend foundations, and their trade-offs. It is **not** a claim that the entire restaurant product, every offline workflow, or international production deployment is complete.

## 1. PostgreSQL database-per-tenant

**Challenge:** isolate customer information while keeping provisioning, migrations, and future cluster expansion manageable.

**Approach:**
- Each customer workspace has its own PostgreSQL database.
- The control database contains the authoritative tenant registry and database connection mapping.
- Tenant routing is based on authenticated identity and membership, not on a tenant identifier supplied by an untrusted request.
- Tenant connection credentials are scoped; sensitive connection information is stored encrypted in the registry.
- Provisioning creates a tenant database, applies migrations, and registers its location.
- The separation allows tenant-specific backup and restore operations without sharing all customer tables in one database.

**Trade-off:** database-level isolation provides stronger recovery and administrative boundaries, but adds complexity to connection pooling, database fleet migrations, provisioning, and cluster capacity management. A shared schema or shared tables with row-level filtering would be simpler operationally, but did not meet the chosen isolation goals.

## 2. Region-aware data placement

**Challenge:** support country-specific storage choices and future international expansion without hard-coding a single database host.

The control plane models **regions** (jurisdictions) and **database clusters** (PostgreSQL infrastructure assigned to a region) separately. A customer is provisioned into an explicitly selected region and cluster; runtime connections use the resulting tenant-registry mapping.

The implemented foundation includes:
- Region and cluster registries, with operator-facing management.
- Cluster status, activation, and capacity metadata.
- Region/cluster selection as part of tenant provisioning.
- A region assignment that cannot be casually changed through an ordinary tenant update.
- Controlled handling of encrypted cluster administration credentials.

**Important boundary:** this is a **regional placement and data-residency foundation**, not proof of live global multi-region operations or legal compliance in every jurisdiction. Cross-region migration and operational multi-region rollout remain future work.

## 3. PostgreSQL cluster health and diagnostics

**Challenge:** avoid registering infrastructure that cannot support provisioning, and expose connectivity degradation before it causes unexplained failures.

Implemented mechanisms include:
- Connection and authentication checks for cluster configuration changes.
- PostgreSQL query and privilege checks for provisioning readiness.
- Periodic background probes, without tying health checks to administrative page loads.
- Distinct `up`, `degraded`, `down`, and `unknown` status semantics.
- Consecutive-failure thresholds to reduce status flapping.
- Historical probe records, bounded retention, and error logging.
- Health visibility for platform administrators.

The documented configuration uses a 30-second probe cadence, three consecutive failures before marking a cluster down, and 14-day probe-history retention. These are configurable operational policies, **not measured availability figures**.

A cluster's reported health informs an operator's decision; the system does not automatically forbid an explicit provisioning choice solely because the last health result is `down`.

## 4. Transactional events and asynchronous workers

**Challenge:** coordinate background work without losing consistency between a committed database change and the event describing it.

The project contains a transactional-outbox implementation, versioned event contracts, and a worker-side dispatcher using PostgreSQL locking and leases. The processing logic includes at-least-once delivery semantics, retries with backoff, attempt limits, and ordering safeguards within an aggregate.

The API and worker roles share the modular backend codebase while running separate responsibilities. This made PostgreSQL-backed asynchronous processing a deliberate alternative to deploying a separate message broker early.

**Boundary:** event transport and workers exist, but not every planned domain or external integration has a completed consumer.

## 5. Offline-first edge: implemented foundation vs target system

**Challenge:** restaurant operations should not be entirely dependent on a cloud connection.

The architecture places an Edge Agent at each venue, intended to serve local POS and kitchen-display clients over a local API and synchronize with the cloud later.

**Implemented foundations:** a Node.js Edge service, local SQLite storage, local outbox/inbox data structures, basic API and token-guard functionality, sync-status visibility, and a background synchronization loop.

**Still planned/incomplete:** the full bidirectional cloud synchronization transport, complete offline ordering/payment workflows, and production-ready POS/KDS behavior. The present sync-protocol implementation contains explicit placeholders for those later stages.

The intended protocol uses acknowledgements, retries, deduplication, and entity-specific conflict policies rather than one global last-write-wins rule. These are **design requirements**, not claims that end-to-end offline synchronization is already operating in production.

## 6. Correctness and operational discipline

The architecture includes domain-level constraints, idempotency patterns, integer minor-unit monetary utilities, migration tooling, environment diagnostics, Docker environments, integration tests, and backup/restore tooling.

Financial modules and complex payment/fiscal flows have different maturity levels. The presence of architectural contracts or monetary utilities should not be mistaken for a finished production payment system.

## 7. Engineering trade-offs

The recurring design principle is to introduce architectural structure where it protects correctness or provides an achievable migration path, while avoiding infrastructure added only for fashion:

- Database-per-tenant **instead of** shared customer tables.
- A tenant registry **instead of** database addresses embedded in application configuration.
- Region/cluster metadata now; **full multi-region operations later**.
- PostgreSQL outbox and worker roles **instead of** an immediate message-broker deployment.
- Local Edge persistence as the foundation of offline operation; **full synchronization still to be delivered**.

## Related engineering work

RestoVibe also contains a separate multi-agent development system that coordinates isolated coding workers, independent reviewers, CI checks, and guarded integration. Its technical architecture is described in [AgentForge — Autonomous Engineering](agentforge.md).

## Authorship and disclosure

The project was developed through human-directed, AI-assisted engineering. My work centered on product and architectural direction, design constraints, decomposition, reviewing engineering decisions, and guiding AI agents through implementation and validation.

This public write-up summarizes engineering patterns without disclosing proprietary product code, customer data, infrastructure addresses, credentials, or operational secrets. Architectural capability and production deployment maturity are intentionally distinguished.
