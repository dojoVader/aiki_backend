# LinkedIn OAuth Authentication

## Overview

AIKI supports **Sign in with LinkedIn** using LinkedIn's OpenID Connect OAuth 2.0 flow. Users can authenticate via LinkedIn to create an account or link their LinkedIn profile to an existing email-based account.

## How It Works

```
┌──────────┐       ┌──────────┐       ┌──────────┐
│  Client   │       │  AIKI    │       │ LinkedIn │
│ (Browser) │       │  Server  │       │   API    │
└────┬─────┘       └────┬─────┘       └────┬─────┘
     │  1. GET /auth/    │                   │
     │  linkedin/login   │                   │
     ├──────────────────►│                   │
     │                   │                   │
     │  2. 302 Redirect  │                   │
     │◄──────────────────┤                   │
     │                   │                   │
     │  3. User logs in on LinkedIn          │
     ├──────────────────────────────────────►│
     │                   │                   │
     │  4. Redirect to   │                   │
     │  /auth/linkedin/  │                   │
     │  callback?code=X  │                   │
     ├──────────────────►│                   │
     │                   │  5. Exchange code  │
     │                   │  for access token  │
     │                   ├──────────────────►│
     │                   │                   │
     │                   │  6. Fetch user     │
     │                   │  info (/userinfo)  │
     │                   ├──────────────────►│
     │                   │                   │
     │  7. Return JWT    │                   │
     │  tokens + user    │                   │
     │◄──────────────────┤                   │
     │                   │                   │
```

## API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/v1/auth/linkedin/login` | Redirects user to LinkedIn authorization page |
| `GET` | `/api/v1/auth/linkedin/callback` | Handles the OAuth callback, returns JWT tokens |

### Login Endpoint

**`GET /api/v1/auth/linkedin/login`**

Redirects the user to LinkedIn's authorization page. Sets a `linkedin_oauth_state` cookie for CSRF protection.

### Callback Endpoint

**`GET /api/v1/auth/linkedin/callback?code=XXX&state=YYY`**

LinkedIn redirects here after the user authorizes. The server:
1. Validates the CSRF `state` parameter against the cookie
2. Exchanges the authorization `code` for a LinkedIn access token
3. Fetches user info (email, name, LinkedIn ID) from LinkedIn's `/v2/userinfo` endpoint
4. Finds or creates the user:
   - **Existing user by LinkedIn ID** → logs them in
   - **Existing user by email** → links LinkedIn ID and logs them in
   - **New user** → creates account with LinkedIn info, no password required
5. Returns JWT access + refresh tokens

**Success Response (200):**
```json
{
  "status": "success",
  "message": "linkedin login successful",
  "data": {
    "access_token": "eyJhbGciOi...",
    "refresh_token": "a1b2c3d4...",
    "user": {
      "id": 1,
      "first_name": "John",
      "last_name": "Doe",
      "email": "john@example.com",
      "linkedin_id": "abc123",
      "is_active": true,
      "created_at": "2026-03-05T10:00:00Z",
      "updated_at": "2026-03-05T10:00:00Z"
    }
  }
}
```

## Setup & Configuration

### 1. Create a LinkedIn App

1. Go to [LinkedIn Developer Portal](https://www.linkedin.com/developers/apps)
2. Create a new app (or use an existing one)
3. Under **Auth** tab:
   - Note your **Client ID** and **Client Secret**
   - Add the redirect URL: `http://localhost:8080/api/v1/auth/linkedin/callback`
4. Under **Products** tab:
   - Request access to **Sign In with LinkedIn using OpenID Connect**

### 2. Set Environment Variables

Add these to your `.env` file:

```env
LINKEDIN_CLIENT_ID=your_client_id_here
LINKEDIN_CLIENT_SECRET=your_client_secret_here
LINKEDIN_CALLBACK_URL=http://localhost:%s/api/v1/auth/linkedin/callback
```

> **Note:** The `%s` in `LINKEDIN_CALLBACK_URL` is replaced at runtime with the server port.

### 3. Run Database Migrations

Ensure the LinkedIn-related migration has been applied:

```bash
# Migration 000003 adds the linkedin_id column to the users table
migrate -path migrations -database "$DB_URL" up
```

### 4. Run the Server

```bash
go run cmd/api/main.go
```

### 5. Test the Flow

1. Open your browser and navigate to:
   ```
   http://localhost:8080/api/v1/auth/linkedin/login
   ```
2. You'll be redirected to LinkedIn's authorization page
3. Log in and authorize the app
4. LinkedIn redirects back to the callback URL
5. The server responds with JWT tokens and user data

## Files Changed

| File | What Changed |
|------|-------------|
| `internal/handler/auth_handler.go` | Constructor accepts config; fixed OAuth scopes; added `LinkedInCallback` handler |
| `internal/service/auth_service.go` | Added `LinkedInLogin` method (find/create user + issue tokens) |
| `internal/repository/user_repository.go` | Added `GetByLinkedInID`, `CreateLinkedInUser`, `UpdateLinkedInID` |
| `internal/router/router.go` | Registered `GET /linkedin/callback` route |
| `cmd/api/main.go` | Passes config to auth handler |
| `internal/handler/auth_handler_test.go` | Updated mocks and constructor calls |
| `internal/service/auth_service_test.go` | Added mock methods for new repository interface |

## Scopes

The app requests these LinkedIn OpenID Connect scopes:

| Scope | Purpose |
|-------|---------|
| `openid` | Required for OpenID Connect |
| `profile` | Access to first name, last name |
| `email` | Access to email address |

> **Note:** The old scopes `r_liteprofile` and `r_emailaddress` were deprecated by LinkedIn and have been replaced.
