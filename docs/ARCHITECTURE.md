# Backend Architecture

## Goals

This backend is structured for a growing HRMS product where each frontend feature can evolve without turning the API into one large routes folder.

## Folder Structure

```text
src/
  bootstrap/
    app.ts                Express app composition
  server.ts               HTTP server bootstrap
  api/
    middleware/           HTTP middleware such as auth and errors
    routes/               API route composition and edge route modules
  features/
    <feature>/            route/controller/service/repository ownership
  shared/
    errors/               AppError and error helpers
    http/                 response helpers and async handler
    crud/                 generic CRUD router scaffolding and resource registry
    types/                shared Express request context
    validation/           reusable Zod schemas and middleware
    logger.ts             shared logger
  infrastructure/
    config/               environment parsing, sessions, Redis
    llm/                  LLM provider adapters
    mail/                 external mail delivery adapter
    storage/              file/object storage adapters
  db/
    client.ts             Drizzle + pg pool
    schema.ts             complete PostgreSQL schema
    migrations/           generated Drizzle migrations
    seeds/                seed scripts
  tests/                  automated tests
```

## Feature Boundaries

Each module owns one frontend-facing business area:

- `organizations`: company profile and tenant metadata
- `users`: account settings, roles, and preferences
- `departments`, `locations`, `designations`, `teams`: organization setup data
- `employees`: employee master data, documents, reporting structure
- `payroll`: salary runs and payroll entries
- `expenses`: categories, expense claims, receipts, approvals
- `billing`: plans, subscriptions, invoices, payment methods
- `inbox`: HR inbox messages
- `leaves`: leave requests and approvals
- `calendar`: meetings and interview events
- `recruitment`: jobs, candidates, applications, pipelines
- `integrations`: integrations and automation workflows

## Request Flow

```text
HTTP request
  -> app middleware
  -> api/routes/index.ts
  -> feature route
  -> validation middleware
  -> controller
  -> service
  -> repository
  -> Drizzle/Postgres
```

## Coding Contract

- Controllers handle HTTP input/output only.
- Services own business rules and orchestration.
- Repositories own database queries.
- Feature code should depend on ports/interfaces when it needs external systems.
- Concrete adapters live in `infrastructure/`.
- Feature `*.dependencies.ts` files are the allowed composition boundary between feature code and infrastructure adapters.
- Validation is done with Zod before controller logic.
- Responses use `{ success, message, data, meta }`.
- Throw `AppError` for expected API errors.
- Keep multi-tenant queries scoped by `organizationId`.
- Authentication uses Redis-backed server sessions and HTTP-only cookies.
- Google OAuth uses the same session store and validates OAuth `state` before logging the user in.
- Email/password registration is pending in Redis until the user verifies the email link; only then are organization/user records created.
