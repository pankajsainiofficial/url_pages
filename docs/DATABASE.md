# Database Guide

## Drizzle + PostgreSQL

The schema lives in `src/db/schema.ts` and maps the provided HRMS DBML-style model into Drizzle `pg-core` tables.

## Main Domains

- Tenant/account: `organizations`, `users`
- Org structure: `departments`, `locations`, `designations`, `teams`, `team_members`
- Employee master: `employees`, `employee_documents`
- Finance: `payroll_batches`, `payroll_entries`, `expense_categories`, `expenses`
- Billing: `billing_plans`, `subscriptions`, `invoices`, `payment_methods`
- Communication/workflow: `inbox_messages`, `leave_requests`, `meeting_events`
- Recruitment: `job_descriptions`, `jobs`, `job_skills`, `job_benefits`, `candidates`, `candidate_skills`, `job_applications`, `pipeline_stages`, `hiring_pipelines`, `pipeline_stage_assignments`, `candidate_stage_history`, `interview_schedules`
- Automation: `integrations`, `automation_workflows`

## Migration Workflow

1. Edit `src/db/schema.ts`.
2. Generate migration:

```bash
npm run db:generate
```

3. Review generated SQL under `src/db/migrations`.
4. Apply migration:

```bash
npm run db:migrate
```

For fast local prototyping only:

```bash
npm run db:push
```

## Environment

```env
DATABASE_URL=postgres://user:password@localhost:5432/zylosis_hrms
REDIS_URL=redis://localhost:6379
SESSION_SECRET=replace-with-a-minimum-32-character-session-secret
SESSION_NAME=hrms.sid
PORT=4000
NODE_ENV=development
CORS_ORIGIN=http://localhost:5173,https://hrms-product-i2k7.vercel.app
GOOGLE_CLIENT_ID=633915875767-4url3hm953v6mdeguch9cr15e1v2e016.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=rotate-this-secret-and-set-it-only-in-env
GOOGLE_AUTH_URL=https://accounts.google.com/o/oauth2/v2/auth
GOOGLE_TOKEN_URL=https://oauth2.googleapis.com/token
GOOGLE_USERINFO_URL=https://www.googleapis.com/oauth2/v3/userinfo
GOOGLE_CALLBACK_URL=https://backend-url.example.com/auth/google/callback
FRONTEND_AUTH_SUCCESS_URL=https://hrms-product-i2k7.vercel.app/dashboard
FRONTEND_AUTH_FAILURE_URL=https://hrms-product-i2k7.vercel.app/login?error=google_auth_failed
FRONTEND_EMAIL_VERIFIED_URL=https://hrms-product-i2k7.vercel.app/login?verified=true
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=smtp-user
SMTP_PASS=smtp-password
MAIL_FROM=Zylosis HRMS <no-reply@example.com>
EMAIL_VERIFICATION_TTL_MINUTES=30
```

## Multi-Tenant Rule

Every organization-owned table includes `organization_id`. Application queries should always filter by the authenticated user's `organizationId`, except for platform-level tables such as `billing_plans`.

## Data Type Notes

- Money values use `numeric(14, 2)`.
- Dates without time use `date`.
- Events and audit timestamps use timezone-aware `timestamp`.
- Dynamic frontend fields use `jsonb`.
