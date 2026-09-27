# Industrial ERP System - Architecture & Project Showcase

Technical showcase website and interactive architecture specifications for an industrial-grade Enterprise Resource Planning (ERP) platform built with **Golang 1.22+** and **Next.js 15 App Router**.

## Showcase Features
- **Overall System Architecture**: Multi-tier architecture covering Caddy reverse proxy, domain modules, Valkey task queues, Floci S3 storage, and PostgreSQL relational clusters.
- **Backend Clean Architecture**: Ports & Adapters layer segregation, type-safe SQL with `sqlc`, pure DDL migrations with `goose`, and OpenAPI 3.0 contracts.
- **Frontend Architecture**: Next.js 15 App Router, React Server Components (RSC), TanStack Table v8, Zustand state stores, and Tailwind CSS / shadcn/ui design tokens.

## Interactive Models
- [System Architecture Model](./architecture/erp-architecture.html)
- [Backend Engine & Services Model](./backend/backend-architecture.html)
- [Frontend Component & State Model](./frontend/frontend-architecture.html)

