# Telegram-Based Prediction Market Trading Platform

**Active commercial project — architecture case study**

**Role:** Backend Engineering · Architecture & Technical Direction  
**Status:** Currently operational; ongoing development and maintenance (confirmed by the project maintainer)  
**Focus:** Financial Correctness · Concurrent Processing · Recovery & Reconciliation · Security Boundaries · Real-Time Integration · Production Operations  
**Public integrations:** Telegram and Polymarket

> This document describes an active commercial system at the architectural level. Third-party wallet/infrastructure providers, product identifiers, private configuration, and proprietary implementation details are intentionally omitted.

## Overview

A Telegram-based prediction-market trading platform with a Node.js backend and direct integration with **Polymarket**. The user-facing product combines a Telegram bot and Mini App with market discovery, order management, positions, automated trading rules, and wallet-related workflows.

The most important engineering problem was **maintaining correct financial state when internal application state and external trading/blockchain state can temporarily disagree**. Requests may time out after an external system has accepted them; fills may arrive more than once, out of order, or concurrently; blockchain settlement is asynchronous.

The backend therefore needs persistent operation tracking, database-level consistency safeguards, and reconciliation against the external source of truth — not simply an API call followed by a success message.

## 1. Backend architecture and responsibility boundaries

The application is implemented as a **modular Node.js monolith**, not as a fleet of independent microservices. It brings together distinct application responsibilities within a shared runtime:

- **Telegram bot and Mini App:** onboarding, user interaction, market search, trading and portfolio interfaces, and notifications.
- **Application API:** authenticated request handling and WebSocket communication.
- **Polymarket integration:** market data, order placement, exchange status, executions, and settlement-related information.
- **Financial records:** durable operation intents, orders, executions, position lots, and accounting/audit records.
- **Background processing:** scheduled reconciliation, automated-rule evaluation, recovery tasks, and external-state checks.
- **Operations:** an administrative dashboard, service and task health monitoring, critical-event reporting, and a Telegram status chatbot.

The persistent database holds the application's internal accounting and operation history. Polymarket and relevant blockchain records remain authoritative for **external execution and settlement outcomes**. In-memory caches and lightweight coordination mechanisms support performance but are not treated as the ultimate guarantee of financial correctness.

**Trade-off:** a modular monolith keeps closely related financial state changes within understandable transactional boundaries and reduces deployment complexity. The cost is a shared process-level failure domain, so background execution, monitoring, and recovery require care.

## 2. Trading capabilities

The operational platform supports trading **on Polymarket only**. Deposits and withdrawals can involve other blockchain networks, but those networks are **not separate order-execution venues**.

User-facing trading functionality includes:

- Market discovery and trading information.
- Market and limit buy/sell orders.
- Portfolio, order, position, and profit/loss views.
- Automated stop-loss, take-profit, stop-buy, and other conditional triggers.
- Configurable trigger presets/templates.
- Real-time market updates combined with periodic checks.

The case study does not list order execution policies that have not been separately verified (for example, a specific fill-or-kill implementation).

## 3. Idempotent orders, fills, and financial accounting

**Challenge:** repeated requests, retried events, concurrent writers, and partial executions must not create duplicate financial entries or inconsistent positions.

The codebase contains a durable order/operation lifecycle and transactional fill processing. Important engineering mechanisms include:

- Writing an operation intent before certain potentially money-moving external effects.
- De-duplicating fills using business identifiers and database uniqueness constraints.
- Applying a fill, position update, and related accounting changes inside a database transaction.
- Serializing concurrent changes to the same position with row-level locking.
- Recording trade-related position lots and closures, with FIFO accounting and reconstructable aggregates.
- Using precise decimal/NUMERIC arithmetic for financial values.
- Using conditional state transitions so competing workers do not both claim the same automated action.

These choices place correctness at the durable data boundary rather than relying solely on process-local mutexes or distributed-cache locks.

## 4. Ambiguous outcomes and reconciliation

**Challenge:** a timeout after submitting an order or blockchain-related operation does **not** prove that it failed. Blindly repeating such a request may cause another external effect.

The architecture distinguishes three classes of outcome:

- **Accepted/confirmed:** authoritative evidence supports the external result.
- **Refused:** the external operation was rejected.
- **Unconfirmed:** the request's outcome is not yet established.

For uncertain operations, the system retains durable evidence and uses background reconciliation to check Polymarket and applicable blockchain records before deciding how to advance internal state.

Reconciliation also covers delayed fills, position changes, and operations interrupted between an external response and a local database update.

**Engineering principle:** recovery should converge toward observed external facts, rather than inventing success or failure from a network error.

## 5. Deposits, withdrawals, and signing boundaries

The platform supports funding and withdrawal workflows involving **multiple blockchain networks**, including conversion or routing of deposited assets into the collateral format used for Polymarket trading. Actual orders are executed through Polymarket.

The security architecture separates roles and permissions:

- A restricted backend-related signing capability is used for authorized trading workflows.
- More sensitive withdrawal and payout-claim operations require the wallet owner's authorization/signature.
- A separate wallet/signing component establishes a boundary between the application backend and owner-controlled asset operations.
- The backend records operation intent and verifies externally observable transaction results before treating relevant claims or settlements as confirmed.

This separation is designed to prevent compromise of ordinary backend trading credentials from automatically granting withdrawal authority. **It is not a claim that backend compromise creates zero financial risk:** trading permissions, external dependencies, and operational defects still have their own risk profiles.

Third-party custody, wallet, signing, blockchain-access, and infrastructure vendors are intentionally not identified.

## 6. Automated trading and real-time processing

Trading automation evaluates conditions using market information and scheduled processing. Features include stop-loss, take-profit, stop-buy, and other configurable trigger rules.

Both real-time observations and background scans may discover a trigger. The backend uses database-backed conditional transitions to claim execution and reduce the risk of firing the same rule twice.

Background reconciliation can re-examine rules whose execution was interrupted or whose linked order's final state is not yet known.

A shared connection to Polymarket market data supports live updates, with reconnect and stale-feed handling. Application WebSocket communication distributes relevant updates to connected clients.

## 7. Administration, monitoring, and incident response

The product is **currently operating**, and its backend is actively maintained.

Its operational tooling includes:

- **Administrative dashboard:** application administration and visibility into trading-related and system state.
- **System and task monitoring:** background-job records, health information, diagnostics, and error/audit history.
- **Critical-event reporting:** Telegram notifications for exceptional situations requiring operator attention.
- **Operational Telegram chatbot:** an interface for checking current service status without opening the dashboard.
- **Recovery and maintenance tooling:** operational diagnostics, controlled maintenance procedures, and tools for investigating inconsistent or uncertain states.
- **Deployment safeguards:** migration serialization and health-checked deployment/rollback mechanisms represented in the codebase.

The monitoring implementation is project-specific. This case study does **not** claim a full distributed tracing stack, guaranteed alert delivery, audited uptime figures, or fully automated resolution of every possible financial discrepancy.

## 8. Engineering ownership and AI-assisted development

My contribution focused on setting the **high-level technical direction** and defining the principal engineering constraints:

- System structure and the main backend/database boundaries.
- Financial state and operation-lifecycle requirements.
- Approaches to duplicate protection and idempotency.
- Concurrency safeguards and retry/recovery strategies.
- Requirements for safe wallet operations and operational visibility.
- Prioritizing architectural trade-offs as the product evolved.

Implementation was developed **collaboratively with AI coding agents**. I defined the architectural direction and expected behavior, while agents helped work through lower-level implementation details, edge cases, and iterative refinements. This was a joint engineering process, not a claim that I personally hand-wrote every component.

The project demonstrates hands-on technical ownership beyond implementation of isolated endpoints: defining correctness requirements, reviewing architectural outcomes, and supporting a functioning commercial product.

## 9. Architecture and operational trade-offs

- **Database transactions and unique constraints vs. cache-only deduplication:** durable financial state must be correct even when coordination caches fail.
- **Explicit reconciliation vs. blind retries:** an uncertain external result requires evidence before repeating a potentially money-moving action.
- **Restricted trading access vs. withdrawal authorization:** separate credentials and signing boundaries reduce the blast radius of compromised application credentials.
- **Modular monolith vs. independent services:** simpler deployment and local transaction scope at the expense of shared runtime failure impact.
- **Periodic checks alongside real-time feeds:** resilience when streams stall or notifications arrive late, while taking care to avoid double execution.

## Engineering strengths demonstrated

Node.js backend architecture · PostgreSQL financial modeling · Idempotency and concurrency control · External API integration · Asynchronous reconciliation · Real-time market data · Conditional automation · Wallet authorization boundaries · Operational monitoring · Production maintenance · Collaborative architecture and AI-assisted engineering.

## Confidentiality and evidence limits

The product is commercially active, so this document does not include its brand name, private providers, key material, customer or wallet identifiers, commercial parameters, infrastructure topology, operational endpoints, or proprietary code.

The accompanying **static source-code audit** supports the described architectural mechanisms, but did not execute the complete test suite or independently verify live production status. Current operation and the multi-network funding experience were confirmed by the project maintainer. No figures for users, trade volume, revenue, uptime, or performance are asserted.

This case study describes engineering intentions and implemented safeguards without claiming that financial systems are immune to all faults or security risks.
