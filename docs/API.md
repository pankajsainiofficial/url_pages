# API Docs And Usage

HTML version for GitHub Pages: [`index.html`](./index.html)

## Base URLs

```text
Backend API:  http://localhost:4000/api
Health:       http://localhost:4000/health
Frontend:     http://localhost:5173
```

The backend uses JSON for normal API responses and HTTP-only cookies for authentication sessions.

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
GET /api/auth/google/callback
```

Frontend usage:

```ts
window.location.href = "http://localhost:4000/api/auth/google";
```

With remember me:

```ts
window.location.href = "http://localhost:4000/api/auth/google?rememberMe=true";
```

Flow:

1. Frontend redirects to `/api/auth/google`.
2. Backend stores OAuth `state` in the Redis session.
3. Google redirects to `/api/auth/google/callback`.
4. Backend validates `state`, exchanges the OAuth code, verifies the email, and creates the app session.
5. Backend redirects to `FRONTEND_AUTH_SUCCESS_URL`.

### Current Session

```text
GET /api/auth/me
```

Frontend usage:

```ts
const session = await apiRequest<{ data: { user: unknown } }>("/auth/me");
```

### Refresh Session

```text
POST /api/auth/refresh
```

This currently returns the active session user and requires an existing session.

### Logout

```text
POST /api/auth/logout
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

```env
DATABASE_URL=postgres://user:password@localhost:5432/zylosis_hrms
REDIS_URL=redis://localhost:6379
SESSION_SECRET=replace-with-a-minimum-32-character-session-secret
SESSION_NAME=hrms.sid
PORT=4000
NODE_ENV=development
CORS_ORIGIN=http://localhost:5173

GOOGLE_CLIENT_ID=your-google-client-id
GOOGLE_CLIENT_SECRET=your-google-client-secret
GOOGLE_CALLBACK_URL=http://localhost:4000/api/auth/google/callback
FRONTEND_AUTH_SUCCESS_URL=http://localhost:5173/dashboard
FRONTEND_AUTH_FAILURE_URL=http://localhost:5173/login?error=google_auth_failed
FRONTEND_EMAIL_VERIFIED_URL=http://localhost:5173/login?verified=true
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=smtp-user
SMTP_PASS=smtp-password
MAIL_FROM=Zylosis HRMS <no-reply@example.com>
EMAIL_VERIFICATION_TTL_MINUTES=30
```

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
