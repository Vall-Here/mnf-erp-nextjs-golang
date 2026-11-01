# Industrial ERP System - Architecture & Project Showcase

Technical showcase website and interactive architecture specifications for an industrial-grade Enterprise Resource Planning (ERP) platform built with **Golang 1.24+** and **Next.js 15 App Router**.

## Showcase Features
- **Overall System Architecture**: Multi-tier architecture covering Caddy reverse proxy, 20 domain modules, Valkey task queues, Floci S3 storage, and PostgreSQL relational clusters.
- **Backend Clean Architecture**: Ports & Adapters layer segregation, explicit parameterized SQL with `pgx/v5`, pure DDL migrations with `goose`, and OpenAPI 3.0 contracts documenting 193 API paths.
- **Frontend Architecture**: Next.js 15 App Router, React Server Components (RSC), TanStack Table v8, Zustand state stores, and Tailwind CSS / shadcn/ui design tokens.
- **Operational Coverage**: Procure-to-Pay, Order-to-Cash, MRP and job costing, perpetual inventory, PSAK 65 consolidation, budget encumbrance, ESS, electronic bank reconciliation, corporate documents hub, and B2B vendor and customer portals.

## Interactive Models
- [System Architecture Model](./architecture/erp-architecture.html)
- [Backend Engine & Services Model](./backend/backend-architecture.html)
- [Frontend Component & State Model](./frontend/frontend-architecture.html)
- [Software Design Document Matrix](./sdd/README.md)
