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

Registration does not create the database account immediately. It creates a short-lived pending registration in Redis, sends an email verification link, and returns `202 Accepted`. The organization/user rows are created only after the verification link is opened.

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
  "message": "Verification email sent. Please verify your email to complete registration.",
  "data": {
    "email": "admin@acme.com",
    "expiresInMinutes": 30
  }
}
```

### Verify Email

```text
GET /api/auth/verify-email?token=<verification-token>
```

The link is sent to the registering user's email. When the token is valid:

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

These module routes are scaffolded with a consistent CRUD shape and are ready for real repository implementations.

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

Each resource supports:

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

Create example:

```ts
await apiRequest("/employees", {
  method: "POST",
  body: JSON.stringify({
    firstName: "Asha",
    lastName: "Mehta",
    email: "asha@example.com",
  }),
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

SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=smtp-user
SMTP_PASS=smtp-password
MAIL_FROM=Zylosis HRMS <no-reply@example.com>
EMAIL_VERIFICATION_TTL_MINUTES=30
```

## Deployment Checklist

1. Deploy backend and set the real backend URL.
2. Set `NODE_ENV=production`.
3. Set `CORS_ORIGIN=https://hrms-product-i2k7.vercel.app`.
4. Set `GOOGLE_CALLBACK_URL=https://<backend-domain>/auth/google/callback`.
5. Add the same callback URL in Google Cloud Console.
6. Set `FRONTEND_AUTH_SUCCESS_URL=https://hrms-product-i2k7.vercel.app/dashboard`.
7. Set `FRONTEND_AUTH_FAILURE_URL=https://hrms-product-i2k7.vercel.app/login?error=google_auth_failed`.
8. Configure Redis and Postgres.
9. Configure SMTP for verification emails.
10. Rotate any exposed Google client secret and store the new value only in hosting env variables.

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
