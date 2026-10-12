# Backend Engineering Portfolio

**Backend Node.js Engineer · JavaScript / TypeScript · Systems Architecture**

## About

I'm a software engineer with approximately **15 years of programming experience**, focused primarily on hands-on backend development with **Node.js and JavaScript**.

I work on backend architecture, API design, asynchronous workflows, reliable integrations, and reusable infrastructure. My experience includes PostgreSQL, Redis, real-time communication, and distributed task processing. I especially enjoy problems involving concurrency, failure recovery, and making complex systems understandable and maintainable.

This repository is a curated overview of backend engineering work, publicly released packages, and selected architectural case studies.

## Architecture case studies

Selected technical write-ups describe the problem, architecture, engineering trade-offs, and which parts are implemented versus planned. They do not expose private source code or operational secrets.

- **[RestoVibe — Multi-Tenant Restaurant SaaS Architecture](projects/restovibe.md):** database-per-tenant PostgreSQL, region/cluster placement, cluster health, transactional outbox, and the offline-first Edge foundation.
- **[AgentForge — Autonomous AI Engineering](projects/agentforge.md):** GitHub-first worker orchestration, isolated agent execution, independent commit-bound review, merge-train validation, and reusable platform design.
- **[Distributed Creator Operations Platform](projects/distributed-creator-platform.md):** active commercial system co-founded and maintained by me; distributed Node.js backend, workload orchestration, performance optimization, and infrastructure/application observability.

## Open-source Node.js libraries

I've developed a set of **iMagic** packages around common backend building blocks. Each package has its own repository and documentation.

| Library | What it provides |
| --- | --- |
| [imagic-web-server](https://github.com/iMagicKey/web-server) | Lightweight HTTP/HTTPS server with routing, middleware support, static assets, and response helpers. |
| [imagic-middleware](https://github.com/iMagicKey/middleware) | Reusable request body parsing, CORS, and rate limiting middleware. |
| [imagic-validator](https://github.com/iMagicKey/validator) | Chainable schema validation for strings, numbers, objects, arrays, and other values. |
| [imagic-auth](https://github.com/iMagicKey/auth) | JWT signing and verification, API-key helpers, and permission checks. |
| [imagic-scheduler](https://github.com/iMagicKey/scheduler) | In-process scheduling with named tasks, intervals, and timeout reporting. |

Explore additional libraries and projects on [my GitHub profile](https://github.com/iMagicKey).

## Engineering focus

- **Backend architecture:** Node.js services, API design, modularity, and clear component boundaries.
- **Data and reliability:** PostgreSQL, Redis, concurrency, idempotency, and recovery workflows.
- **Developer tooling:** reusable packages, testing, automation, and operational tooling.

---

*Case studies describe engineering decisions and distinguish implemented capabilities from ongoing work and roadmap plans.*
