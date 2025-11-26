# VulnVibes - Corporate Document System

A microservices-based document management system with JWT authentication and role-based access control.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client (Browser)                          │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼ :8080
┌─────────────────────────────────────────────────────────────────┐
│                      Nginx Gateway                               │
│  ┌─────────────────┬─────────────────┬─────────────────┐        │
│  │   /auth/*       │    /api/*       │      /*         │        │
│  │   → :3001       │    → :8000      │    → :3000      │        │
│  └─────────────────┴─────────────────┴─────────────────┘        │
└─────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│ auth-service│      │   doc-api   │      │frontend-app │
│  (Node.js)  │      │  (FastAPI)  │      │  (Next.js)  │
│    :3001    │      │    :8000    │      │    :3000    │
└─────────────┘      └─────────────┘      └─────────────┘
```

## Services

| Service | Technology | Port | Description |
|---------|------------|------|-------------|
| **gateway** | Nginx | 8080 | API gateway, routes requests to appropriate services |
| **auth-service** | Node.js/Express | 3001 | Handles authentication, issues JWT tokens |
| **doc-api** | Python/FastAPI | 8000 | Document API with role-based access control |
| **frontend-app** | Next.js/React | 3000 | Web user interface |

## Quick Start

```bash
# 1. Configure auth-service environment
cp ../auth-service/.env.sample ../auth-service/.env
# Edit .env to set your own JWT_SECRET and user credentials

# 2. Configure doc-api environment
cp ../doc-api/.env.sample ../doc-api/.env
# Edit .env to set JWT_SECRET (must match auth-service) and documents

# 3. From the infra-ops directory, start all services
docker-compose up --build

# 4. Access the application
open http://localhost:8080
```

## Authentication

### Credentials

User credentials are configured via environment variables in `auth-service/.env`. 

Default credentials (from `.env`):

| Username | Password | Role | Access Level |
|----------|----------|------|--------------|
| `admin` | <redacted> | sys_admin | All documents |
| `staff` | <redacted> | staff | Public documents only |

To customize, edit the `USERS` variable in `auth-service/.env`.

### Authentication Flow

1. User submits credentials via the login form
2. Frontend sends `POST /auth/login` with `{ username, password }`
3. Auth-service validates credentials against user database (in-memory)
4. On success, auth-service returns a signed JWT token containing:
   - `username`
   - `role`
   - `name`
   - `exp` (expiration - 1 hour)
5. Frontend stores the token in `localStorage`
6. Subsequent API requests include the token in `Authorization: Bearer <token>` header

### JWT Structure

```json
{
  "username": "admin",
  "role": "sys_admin",
  "name": "System Administrator",
  "iat": 1699999999,
  "exp": 1700003599
}
```

## API Endpoints

### Auth Service (`/auth/*`)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| POST | `/auth/login` | No | Authenticate and receive JWT token |
| GET | `/auth/verify` | Yes | Verify token validity |
| GET | `/auth/health` | No | Service health check |

### Document API (`/api/*`)

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| GET | `/api/documents` | Yes | Get documents (filtered by role) |
| GET | `/api/documents/public` | No | Get public documents only |
| GET | `/api/health` | No | Service health check |

## Request Flow Examples

### 1. Login Request

```
Browser                    Gateway                   Auth-Service
   │                          │                           │
   │ POST /auth/login         │                           │
   │ {username, password}     │                           │
   │─────────────────────────>│                           │
   │                          │ POST /login               │
   │                          │ {username, password}      │
   │                          │──────────────────────────>│
   │                          │                           │ Validate credentials
   │                          │                           │ Generate JWT
   │                          │         {token, user}     │
   │                          │<──────────────────────────│
   │      {token, user}       │                           │
   │<─────────────────────────│                           │
   │                          │                           │
   │ Store token in           │                           │
   │ localStorage             │                           │
```

### 2. Fetch Documents (Authenticated)

```
Browser                    Gateway                   Doc-API
   │                          │                           │
   │ GET /api/documents       │                           │
   │ Authorization: Bearer X  │                           │
   │─────────────────────────>│                           │
   │                          │ GET /documents            │
   │                          │ Authorization: Bearer X   │
   │                          │──────────────────────────>│
   │                          │                           │ Decode JWT
   │                          │                           │ Extract role
   │                          │                           │ Filter documents
   │                          │        [documents]        │
   │                          │<──────────────────────────│
   │       [documents]        │                           │
   │<─────────────────────────│                           │
```

### 3. Access Public Documents (No Auth)

```
Browser                    Gateway                   Doc-API
   │                          │                           │
   │ GET /api/documents/public│                           │
   │─────────────────────────>│                           │
   │                          │ GET /documents/public     │
   │                          │──────────────────────────>│
   │                          │                           │ Return public docs
   │                          │     [public documents]    │
   │                          │<──────────────────────────│
   │    [public documents]    │                           │
   │<─────────────────────────│                           │
```

## Role-Based Access Control

| Role | Public Documents | Restricted Documents |
|------|------------------|---------------------|
| sys_admin | ✅ | ✅ |
| staff | ✅ | ❌ |
| unauthenticated | ✅ (via /public) | ❌ |

## Configuration

### Environment Variables

Both `auth-service` and `doc-api` require `.env` files. Copy the samples and customize:

```bash
cp auth-service/.env.sample auth-service/.env
cp doc-api/.env.sample doc-api/.env
```

**Important:** `JWT_SECRET` must be identical in both services.

| Variable | Service | Required | Description |
|----------|---------|----------|-------------|
| `JWT_SECRET` | auth-service, doc-api | Yes | Secret key for signing/verifying JWTs (must match!) |
| `USERS` | auth-service | Yes | JSON array of user objects |
| `DOCUMENTS` | doc-api | Yes | JSON array of document objects |

#### USERS Format (auth-service)

```json
[
  {"username": "admin", "password": "secret", "role": "sys_admin", "name": "Admin User"},
  {"username": "staff", "password": "secret", "role": "staff", "name": "Staff User"}
]
```

Each user object must have:
- `username` - Login username
- `password` - Login password (plaintext)
- `role` - Either `sys_admin` (full access) or `staff` (public docs only)
- `name` - Display name

#### DOCUMENTS Format (doc-api)

```json
[
  {"id": 1, "title": "Public Doc", "content": "Content here...", "access": "public"},
  {"id": 2, "title": "Secret Doc", "content": "Restricted content...", "access": "restricted"}
]
```

Each document object must have:
- `id` - Unique document ID (integer)
- `title` - Document title
- `content` - Document content
- `access` - Either `public` (visible to all) or `restricted` (sys_admin only)

### Gateway Routing (nginx.conf)

```nginx
location /auth/  → http://auth-service:3001/
location /api/   → http://doc-api:8000/
location /       → http://frontend-app:3000/
```

## Development

### Rebuild a specific service

```bash
docker-compose up --build <service-name>
# e.g., docker-compose up --build doc-api
```

### View logs

```bash
docker-compose logs -f <service-name>
```

### Stop all services

```bash
docker-compose down
```

## Security Notes

- `.env` file contains secrets - never commit it to version control (already in `.gitignore`)
- Passwords are stored in plaintext in `.env` - use proper hashing (bcrypt) and a database in production
- CORS is permissively configured - restrict origins in production
- HTTPS should be enabled in production
- Generate a strong `JWT_SECRET` for production (e.g., `openssl rand -base64 32`)
