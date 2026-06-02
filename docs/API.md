# API Docs And Usage

HTML version for GitHub Pages: [`index.html`](./index.html)

## Base URLs

```text
Local Backend API:        http://localhost:4000/api
Local Backend Auth Alias: http://localhost:4000/auth
Local Health:             http://localhost:4000/health
Local Frontend:           http://localhost:5173
Production Frontend:      https://hrms-product-i2k7.vercel.app
Production Backend:       https://backend-url.example.com
```

The backend uses JSON for normal API responses and HTTP-only cookies for authentication sessions.

Replace `https://backend-url.example.com` everywhere with the deployed backend domain.

## Response Format

Success:

```json
{
  "success": true,
  "message": "Request completed successfully",
  "data": {},
  "meta": {}
}
```

Error:

```json
{
  "success": false,
  "message": "Validation failed",
  "details": [
    {
      "path": "email",
      "message": "Invalid email"
    }
  ]
}
```

## Frontend Fetch Helper

Use `credentials: "include"` for every authenticated request. This allows the browser to send the Redis-backed session cookie.

```ts
const API_BASE_URL =
  import.meta.env.VITE_API_BASE_URL ?? "http://localhost:4000/api";

export async function apiRequest<T>(
  path: string,
  options: RequestInit = {},
): Promise<T> {
  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...options,
    credentials: "include",
    headers: {
      "Content-Type": "application/json",
      ...options.headers,
    },
  });

  const payload = await response.json().catch(() => null);

  if (!response.ok) {
    throw new Error(payload?.message ?? "API request failed");
  }

  return payload;
}
```

## Authentication

Authentication is server-session based:

- Session storage: Redis
- Browser cookie: HTTP-only cookie named by `SESSION_NAME`
- Default lifetime: 8 hours
- Remember me lifetime: 30 days
- Frontend must use `credentials: "include"`
- In production, cookies are `secure` and `sameSite: "none"` so the Vercel frontend can receive the backend session cookie.

Frontend production origin:

```text
https://hrms-product-i2k7.vercel.app
```

### Register

```text
POST /api/auth/register
```

Registration does not create the database account immediately. It creates a short-lived pending registration in Redis, sends a 6 digit email OTP, and returns `202 Accepted`. The organization/user rows are created only after the OTP is verified.

Request:

```json
{
  "organizationName": "Acme HR",
  "organizationEmail": "admin@acme.com",
  "password": "strong-password",
  "termsAccepted": true,
  "rememberMe": false
}
```

Frontend usage:

```ts
await apiRequest("/auth/register", {
  method: "POST",
  body: JSON.stringify({
    organizationName,
    organizationEmail,
    password,
    termsAccepted: true,
    rememberMe: false,
  }),
});
```

Response:

```json
{
  "success": true,
  "message": "Verification OTP sent. Please verify your email to complete registration.",
  "data": {
    "email": "admin@acme.com",
    "expiresInMinutes": 30
  }
}
```

Postman/curl test:

```bash
curl -X POST http://localhost:4000/api/auth/register \
  -H "Content-Type: application/json" \
  --data-raw '{
    "organizationName": "Zylosis Test Company",
    "organizationEmail": "admin@example.com",
    "password": "Test@123456",
    "termsAccepted": true,
    "rememberMe": true
  }'
```

### Verify Email OTP

```text
POST /api/auth/verify-email
```

Request:

```json
{
  "email": "admin@acme.com",
  "otp": "123456"
}
```

When the OTP is valid:

- Backend creates the organization and owner user.
- Backend deletes the pending registration from Redis.
- Backend creates the same HTTP-only session cookie used by login.

Response:

```json
{
  "success": true,
  "message": "Email verified and account created successfully",
  "data": {
    "organization": {
      "id": 1,
      "name": "Acme HR",
      "email": "admin@acme.com"
    },
    "user": {
      "id": 1,
      "organizationId": 1,
      "firstName": "Admin",
      "lastName": "User",
      "email": "admin@acme.com",
      "role": "owner",
      "status": "active",
      "isAdmin": true
    }
  }
}
```

Postman/curl test:

```bash
curl -X POST http://localhost:4000/api/auth/verify-email \
  -H "Content-Type: application/json" \
  --data-raw '{
    "email": "admin@example.com",
    "otp": "123456"
  }'
```

### Legacy Verify Email Link

```text
GET /api/auth/verify-email?token=<verification-token>
```

This route is kept for backward compatibility. When the token is valid:

- Backend creates the organization and owner user.
- Backend deletes the pending registration from Redis.
- Backend creates the same HTTP-only session cookie used by login.
- Backend redirects to `FRONTEND_EMAIL_VERIFIED_URL` when configured.

If `FRONTEND_EMAIL_VERIFIED_URL` is not configured, the API returns JSON:

```json
{
  "success": true,
  "message": "Email verified and account created successfully",
  "data": {
    "organization": {
      "id": 1,
      "name": "Acme HR",
      "email": "admin@acme.com"
    },
    "user": {
      "id": 1,
      "organizationId": 1,
      "firstName": "Admin",
      "lastName": "User",
      "email": "admin@acme.com",
      "role": "owner",
      "status": "active",
      "isAdmin": true
    }
  }
}
```

### Login

```text
POST /api/auth/login
```

Request:

```json
{
  "email": "admin@acme.com",
  "password": "strong-password",
  "rememberMe": true
}
```

Frontend usage:

```ts
await apiRequest("/auth/login", {
  method: "POST",
  body: JSON.stringify({
    email,
    password,
    rememberMe,
  }),
});
```

Response:

```json
{
  "success": true,
  "message": "Logged in successfully",
  "data": {
    "user": {
      "id": 1,
      "organizationId": 1,
      "firstName": "Admin",
      "lastName": "User",
      "email": "admin@acme.com",
      "role": "owner",
      "status": "active",
      "isAdmin": true
    }
  }
}
```

### Google Login

```text
GET /api/auth/google
GET /auth/google/login
GET /api/auth/google/callback
GET /auth/google/callback
POST /api/auth/google/callback
POST /auth/google/callback
```

Frontend usage:

```ts
window.location.href = "https://backend-url.example.com/auth/google/login";
```

With remember me:

```ts
window.location.href = "https://backend-url.example.com/auth/google/login?rememberMe=true";
```

Backend-only OAuth setup:

```text
Google Authorized redirect URI:
https://backend-url.example.com/auth/google/callback
```

In this flow the frontend only redirects the browser to the backend. The backend sends the user to Google, receives the callback, creates the Redis-backed session cookie, then redirects to `FRONTEND_AUTH_SUCCESS_URL`.

Alternative frontend-callback flow:

```text
https://hrms-product-i2k7.vercel.app/auth/google/callback?code=...&state=...
```

Then the frontend callback page should exchange the code with the backend:

```ts
const params = new URLSearchParams(window.location.search);

await apiRequest("/auth/google/callback", {
  method: "POST",
  body: JSON.stringify({
    code: params.get("code"),
    state: params.get("state"),
  }),
});
```

Flow:

1. Frontend redirects to `/auth/google/login`.
2. Backend stores OAuth `state` in the Redis session.
3. Google redirects to the backend `GOOGLE_CALLBACK_URL`.
4. Backend validates `state`, exchanges the OAuth code, verifies the email, and creates the app session.
5. Backend redirects to `FRONTEND_AUTH_SUCCESS_URL`.

For the current Vercel frontend, use:

```ts
window.location.href = "https://backend-url.example.com/auth/google/login";
```

After success, backend redirects to:

```text
https://hrms-product-i2k7.vercel.app/dashboard
```

### Google Cloud Console Setup

OAuth client type:

```text
Web application
```

Authorized JavaScript origins:

```text
https://hrms-product-i2k7.vercel.app
https://backend-url.example.com
```

Authorized redirect URIs for backend-only OAuth:

```text
https://backend-url.example.com/auth/google/callback
```

If using the alternative frontend-callback flow, also add:

```text
https://hrms-product-i2k7.vercel.app/auth/google/callback
```

Current Google project/client:

```text
Project ID: hirex-497006
Client ID: 633915875767-4url3hm953v6mdeguch9cr15e1v2e016.apps.googleusercontent.com
```

Never commit the Google client secret. Store it only as `GOOGLE_CLIENT_SECRET` in deployment environment variables.

### Current Session

```text
GET /api/auth/me
GET /auth/me
```

Frontend usage:

```ts
const session = await apiRequest<{ data: { user: unknown } }>("/auth/me");
```

### Refresh Session

```text
POST /api/auth/refresh
POST /auth/refresh
```

This currently returns the active session user and requires an existing session.

### Logout

```text
POST /api/auth/logout
POST /auth/logout
```

Frontend usage:

```ts
await apiRequest("/auth/logout", {
  method: "POST",
});
```

## Resource Routes

These module routes follow a consistent resource shape. Some resources, such as employees and HR inbox, also expose domain-specific endpoints documented below.

```text
/api/organizations
/api/users
/api/departments
/api/locations
/api/designations
/api/teams
/api/employees
/api/payroll
/api/expenses
/api/billing
/api/inbox
/api/leaves
/api/calendar
/api/recruitment
/api/integrations
```

Most resources support:

```text
GET    /api/<resource>
GET    /api/<resource>/:id
POST   /api/<resource>
PUT    /api/<resource>/:id
DELETE /api/<resource>/:id
```

List query parameters:

```text
organizationId?: number
search?: string
page?: number
limit?: number
```

Example:

```ts
const employees = await apiRequest("/employees?organizationId=1&page=1&limit=20");
```

Employee create example:

```ts
await apiRequest("/employees", {
  method: "POST",
  body: JSON.stringify({
    employeeId: "EMP-1024",
    firstName: "Asha",
    lastName: "Mehta",
    email: "asha@example.com",
    phone: "+91 9876543210",
    department: "Engineering",
    designation: "Frontend Developer",
    employmentType: "Full Time",
    joiningDate: "2026-05-24",
  }),
});
```

## Employee APIs

### List Employees

```text
GET /api/employees
```

Returns employee cards/profile records for the dashboard employee page.

Query parameters:

```text
organizationId?: number
search?: string
page?: number
limit?: number
```

Frontend usage:

```ts
const response = await apiRequest("/employees?page=1&limit=20");
```

### Get Employee By ID

```text
GET /api/employees/:id
```

Returns one employee record by database ID.

Frontend usage:

```ts
const employee = await apiRequest("/employees/12");
```

### Create Employee From Dashboard

```text
POST /api/employees
```

Creates an employee directly from the authenticated dashboard flow. Department and designation names are looked up or created automatically.

Request body:

```json
{
  "employeeId": "EMP-1024",
  "firstName": "Asha",
  "lastName": "Mehta",
  "email": "asha@example.com",
  "phone": "+91 9876543210",
  "dob": "1998-04-10",
  "gender": "Female",
  "department": "Engineering",
  "designation": "Frontend Developer",
  "employmentType": "Full Time",
  "joiningDate": "2026-05-24",
  "manager": "Ravi Sharma",
  "status": "Active",
  "workLocation": "Gurgaon Office",
  "panNumber": "ABCDE1234F",
  "bankName": "HDFC Bank",
  "accountNumber": "1234567890",
  "ifscCode": "HDFC0000211",
  "ctc": "1200000",
  "address": "Sector 44",
  "city": "Gurgaon",
  "state": "Haryana",
  "postalCode": "122001",
  "emergencyName": "Neha Mehta",
  "emergencyRelation": "Sister",
  "emergencyPhone": "+91 9876500000"
}
```

### Create Employee Self-Service Form Link

```text
POST /api/employees/share-form
```

Creates a tokenized link for the "Share Candidate Form" dashboard action. This does not bypass the dashboard; it only creates a separate self-fill form link that can later submit to the backend.

Authentication:

```text
Requires an authenticated admin session.
```

Request body:

```json
{
  "recipientEmail": "asha@example.com",
  "recipientName": "Asha Mehta",
  "expiresInDays": 7
}
```

Response data:

```json
{
  "id": 1,
  "token": "generated-token",
  "url": "https://hrms-product-i2k7.vercel.app/employee/self-service?token=generated-token",
  "status": "active",
  "expiresAt": "2026-05-31T00:00:00.000Z",
  "recipientEmail": "asha@example.com",
  "recipientName": "Asha Mehta"
}
```

Frontend usage:

```ts
const link = await apiRequest("/employees/share-form", {
  method: "POST",
  body: JSON.stringify({
    recipientEmail: "asha@example.com",
    recipientName: "Asha Mehta",
    expiresInDays: 7,
  }),
});
```

### Get Employee Self-Service Form Configuration

```text
GET /api/employees/share-form/:token
```

Public endpoint used by the separate self-fill form page. It validates the token and returns form metadata, organization details, recipient details, expiry, and the field list to render.

Response data:

```json
{
  "token": "generated-token",
  "expiresAt": "2026-05-31T00:00:00.000Z",
  "recipientEmail": "asha@example.com",
  "recipientName": "Asha Mehta",
  "organization": {
    "id": 1,
    "name": "Zylosis",
    "logoUrl": null
  },
  "fields": ["firstName", "lastName", "email", "employeeId"]
}
```

### Submit Employee Self-Service Form

```text
POST /api/employees/share-form/:token/submit
```

Public endpoint used by the separate self-fill form page. It creates the employee under the organization attached to the link and marks the link as submitted so it cannot be reused.

Request body uses the same employee fields as dashboard employee creation, except `organizationId` is not accepted from the public form.

Frontend usage:

```ts
await apiRequest(`/employees/share-form/${token}/submit`, {
  method: "POST",
  body: JSON.stringify({
    employeeId: "EMP-1024",
    firstName: "Asha",
    lastName: "Mehta",
    email: "asha@example.com",
    phone: "+91 9876543210",
    department: "Engineering",
    designation: "Frontend Developer",
    joiningDate: "2026-05-24",
  }),
});
```

Token validation rules:

```text
status must be active
token must not be expired
token must not already be submitted
```

## HR Inbox Mailbox APIs

HR Inbox supports multiple organizations and multiple HR users per organization. Each logged-in HR user can save one mailbox connection. Sync reads only that HR user's mailbox and inbox messages are scoped by both `organizationId` and `recipientUserId`.

Data model:

```text
organizations
  users
    hr_inbox_mailboxes
      inbox_messages
```

Example: HR 1 in Organization A can connect `hr1@gmail.com`, HR 2 in the same organization can connect `hr2@company.com`, and both see only their own synced HR emails.

Frontend setup location:

```text
Settings → Organization Settings → HR Inbox Mailbox
```

HR users enter their email provider, username, IMAP details, and app password from this settings card. The Inbox page also exposes a manual `Sync` action and attempts sync when opened.

### Get Current HR Mailbox Settings

```text
GET /api/inbox/settings
```

Authentication: requires an active session.

Response:

```json
{
  "success": true,
  "message": "HR inbox settings fetched",
  "data": {
    "id": 1,
    "organizationId": 1,
    "userId": 2,
    "status": "connected",
    "lastSyncAt": "2026-05-31T17:30:00.000Z",
    "settings": {
      "email": "hr1@gmail.com",
      "provider": "gmail",
      "port": 993,
      "secure": true,
      "username": "hr1@gmail.com",
      "hasAppPassword": true
    }
  }
}
```

### Save Current HR Mailbox Settings

```text
PUT /api/inbox/settings
```

Saves mailbox settings for the logged-in HR user only. If `appPassword` is omitted during an update, the existing saved app password is kept.

Request:

```json
{
  "email": "hr1@gmail.com",
  "provider": "gmail",
  "username": "hr1@gmail.com",
  "appPassword": "gmail-app-password",
  "port": 993,
  "secure": true
}
```

Provider defaults:

```text
gmail:              imap.gmail.com:993 secure=true
outlook/office365:  outlook.office365.com:993 secure=true
yahoo:              imap.mail.yahoo.com:993 secure=true
custom:             provide host, port, secure
```

### Sync Current HR Mailbox

```text
POST /api/inbox/sync
```

Connects to the logged-in HR user's mailbox, fetches unread emails from INBOX, parses sender/subject/body, filters HR-related content, saves matching emails, and marks processed mailbox emails as seen.

Response:

```json
{
  "success": true,
  "message": "HR inbox sync checked",
  "data": {
    "synced": true,
    "fetched": 8,
    "saved": 3,
    "ignored": 5
  }
}
```

### List HR Inbox Messages

```text
GET /api/inbox/messages
```

Returns only messages for the logged-in HR user. The frontend Inbox page uses this API instead of static demo data.

Query parameters:

```text
type?: all | leave | resignation | query | payroll | attendance | document | general
status?: open | approved | rejected | moved | archived
unread?: boolean
search?: string
page?: number
limit?: number
```

Frontend usage:

```ts
const response = await apiRequest("/inbox/messages?type=leave&search=sick");
```

### Update Inbox Message

```text
PATCH /api/inbox/messages/:id
```

Marks a message read/unread or updates workflow status. The backend only updates messages owned by the logged-in HR user.

Request:

```json
{
  "unread": false,
  "status": "approved"
}
```

### Manual Email Ingest

```text
POST /api/inbox/messages
```

Useful for testing or webhook-based providers. The backend still validates that the message belongs to the logged-in HR user's configured mailbox and filters non-HR emails.

Request:

```json
{
  "mailboxEmail": "hr1@gmail.com",
  "senderName": "Asha Mehta",
  "senderEmail": "asha@company.com",
  "subject": "Leave Request - 5 Days",
  "body": "I would like to request leave from Monday to Friday.",
  "unread": true
}
```

Filtering categories include `resignation`, `leave`, `payroll`, `attendance`, `document`, `query`, and `general`. Non-HR messages are ignored during ingest.

Security note: use mailbox app passwords or provider-approved credentials. Do not use a normal Gmail/Outlook password. App passwords are currently stored in the database and should be encrypted before production use.

## Recruitment APIs

### Generate Job Description

```text
POST /api/recruitment/job-descriptions/generate
```

Generates a company-standard job description from the JD Generation screen fields. If `LLM_API_KEY` is configured, the backend tries the LLM provider first. If the provider is unavailable or not configured, the backend returns a built-in company-standard template so the feature remains usable.

Request body:

```json
{
  "title": "Senior Frontend Developer",
  "department": "Engineering",
  "experience": "Senior",
  "employmentType": "Full-time",
  "workMode": "Hybrid",
  "roleFocus": "Build product UI",
  "roleLevel": "Individual Contributor",
  "companyStage": "Startup",
  "jdStyle": "Company Standard",
  "location": "Gurgaon",
  "salaryRange": "15-22 LPA",
  "openings": "2",
  "noticePeriod": "30 Days",
  "education": "Bachelors Degree",
  "industry": "SaaS",
  "reportingTo": "Engineering Manager",
  "interviewProcess": "3 Rounds",
  "experienceYears": "4-6 Years",
  "hiringPriority": "High",
  "shiftTiming": "General",
  "skills": ["React", "TypeScript"],
  "techStack": ["React", "Node.js"],
  "benefits": ["Health Insurance"],
  "customInstructions": "Mention ownership and collaboration."
}
```

Response:

```json
{
  "success": true,
  "message": "Job description generated",
  "data": {
    "template": "company-standard",
    "provider": "template",
    "document": {
      "title": "Senior Frontend Developer",
      "summary": "This is a company-standard job description for a Senior Frontend Developer role in Engineering, optimized for clarity, candidate readability, and hiring team alignment.",
      "metadata": [
        "Engineering",
        "Hybrid",
        "Full-time",
        "Gurgaon",
        "15-22 LPA",
        "4-6 Years"
      ],
      "sections": [
        {
          "heading": "About the Role",
          "body": "We are looking for a Senior Frontend Developer to join our Engineering team."
        },
        {
          "heading": "Key Responsibilities",
          "bullets": [
            "Lead execution for build product UI within the Engineering function."
          ]
        }
      ]
    },
    "rawText": "# Senior Frontend Developer\nEngineering | Hybrid | Full-time | Gurgaon | 15-22 LPA | 4-6 Years\n\n...",
    "generatedAt": "2026-05-26T16:16:30.474Z"
  }
}
```

`provider` is `llm` when the configured LLM returns a valid response, otherwise `template`.

Frontend usage:

```ts
await apiRequest("/recruitment/job-descriptions/generate", {
  method: "POST",
  body: JSON.stringify(payload),
});
```

## Environment Required For API Usage

Local development example:

```env
DATABASE_URL=postgres://user:password@localhost:5432/zylosis_hrms
REDIS_URL=redis://localhost:6379
SESSION_SECRET=replace-with-a-minimum-32-character-session-secret
SESSION_NAME=hrms.sid
PORT=4000
NODE_ENV=development
CORS_ORIGIN=http://localhost:5173,http://localhost:5174,https://hrms-product-i2k7.vercel.app

GOOGLE_CLIENT_ID=633915875767-4url3hm953v6mdeguch9cr15e1v2e016.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=rotate-this-secret-and-set-it-only-in-env
GOOGLE_AUTH_URL=https://accounts.google.com/o/oauth2/v2/auth
GOOGLE_TOKEN_URL=https://oauth2.googleapis.com/token
GOOGLE_USERINFO_URL=https://www.googleapis.com/oauth2/v3/userinfo
GOOGLE_CALLBACK_URL=https://backend-url.example.com/auth/google/callback
FRONTEND_AUTH_SUCCESS_URL=https://hrms-product-i2k7.vercel.app/dashboard
FRONTEND_AUTH_FAILURE_URL=https://hrms-product-i2k7.vercel.app/login?error=google_auth_failed
FRONTEND_EMAIL_VERIFIED_URL=https://hrms-product-i2k7.vercel.app/login?verified=true
EMPLOYEE_SELF_SERVICE_FORM_URL=https://hrms-product-i2k7.vercel.app/employee/self-service
EMPLOYEE_SELF_SERVICE_FORM_TTL_DAYS=7
SMTP_HOST=smtppro.zoho.in
SMTP_PORT=465
SMTP_SECURE=true
SMTP_USER=yourname@yourdomain.com
SMTP_PASS=your-zoho-app-password
MAIL_FROM=Zylosis HRMS <yourname@yourdomain.com>
EMAIL_VERIFICATION_TTL_MINUTES=30
LLM_API_KEY=
LLM_API_URL=https://api.openai.com/v1/chat/completions
LLM_MODEL=gpt-4o-mini
```

Zoho notes:

- Use `smtppro.zoho.in` for paid/professional India-region Zoho domain mail.
- Use the full mailbox address in `SMTP_USER`.
- If Zoho 2FA is enabled, use an app-specific password in `SMTP_PASS`.
- `MAIL_FROM` should match `SMTP_USER` or a verified Zoho alias.

Production deployment example:

```env
DATABASE_URL=postgres://user:password@host:5432/zylosis_hrms
REDIS_URL=redis://user:password@host:6379
SESSION_SECRET=use-a-long-random-secret-at-least-32-characters
SESSION_NAME=hrms.sid
PORT=4000
NODE_ENV=production
CORS_ORIGIN=https://hrms-product-i2k7.vercel.app

GOOGLE_CLIENT_ID=633915875767-4url3hm953v6mdeguch9cr15e1v2e016.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=set-this-in-hosting-env-only
GOOGLE_AUTH_URL=https://accounts.google.com/o/oauth2/v2/auth
GOOGLE_TOKEN_URL=https://oauth2.googleapis.com/token
GOOGLE_USERINFO_URL=https://www.googleapis.com/oauth2/v3/userinfo
GOOGLE_CALLBACK_URL=https://backend-url.example.com/auth/google/callback
FRONTEND_AUTH_SUCCESS_URL=https://hrms-product-i2k7.vercel.app/dashboard
FRONTEND_AUTH_FAILURE_URL=https://hrms-product-i2k7.vercel.app/login?error=google_auth_failed
FRONTEND_EMAIL_VERIFIED_URL=https://hrms-product-i2k7.vercel.app/login?verified=true
EMPLOYEE_SELF_SERVICE_FORM_URL=https://hrms-product-i2k7.vercel.app/employee/self-service
EMPLOYEE_SELF_SERVICE_FORM_TTL_DAYS=7

SMTP_HOST=smtppro.zoho.in
SMTP_PORT=465
SMTP_SECURE=true
SMTP_USER=yourname@yourdomain.com
SMTP_PASS=set-this-in-hosting-env-only
MAIL_FROM=Zylosis HRMS <yourname@yourdomain.com>
EMAIL_VERIFICATION_TTL_MINUTES=30
LLM_API_KEY=set-this-in-hosting-env-only
LLM_API_URL=https://api.openai.com/v1/chat/completions
LLM_MODEL=gpt-4o-mini
```

## Deployment Checklist

1. Deploy backend and set the real backend URL.
2. Set `NODE_ENV=production`.
3. Set `CORS_ORIGIN=https://hrms-product-i2k7.vercel.app`.
4. Set `GOOGLE_CALLBACK_URL=https://<backend-domain>/auth/google/callback`.
5. Add the same callback URL in Google Cloud Console.
6. Set `FRONTEND_AUTH_SUCCESS_URL=https://hrms-product-i2k7.vercel.app/dashboard`.
7. Set `FRONTEND_AUTH_FAILURE_URL=https://hrms-product-i2k7.vercel.app/login?error=google_auth_failed`.
8. Set `EMPLOYEE_SELF_SERVICE_FORM_URL` to the hosted self-fill employee form URL.
9. Configure Redis and Postgres.
10. Configure SMTP for verification emails.
11. Rotate any exposed Google client secret and store the new value only in hosting env variables.

## Local Verification

```bash
cd Backend
npm run build
npm run test
```

Run Redis and Postgres before starting the server:

```bash
npm run dev
```

## Docs Hosting And Push Scripts

HTML API docs live at:

```text
docs/index.html
```

Docs-only public repo:

```text
git@github.com:pankajsainiofficial/url_pages.git
```

Expected GitHub Pages URL:

```text
https://pankajsainiofficial.github.io/url_pages/
```

Push only API docs to the public docs repo:

```bash
npm run push:docs
```

Push only backend code to the backend repo:

```bash
npm run push:code
```

Push backend code and API docs:

```bash
npm run push:split
```

Override defaults:

```bash
DOCS_REPO=git@github.com:pankajsainiofficial/url_pages.git \
DOCS_BRANCH=main \
DOCS_COMMIT_MESSAGE="Update API docs" \
npm run push:docs
```
