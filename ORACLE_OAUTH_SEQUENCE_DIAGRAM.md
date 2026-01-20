# OAuth 2.0 / OIDC Integration - Oracle MyLearn
## User Authentication Flow - Sequence Diagram

**Document Version:** 1.0
**Date:** January 20, 2026
**Prepared for:** Oracle MyLearn Team
**Integration Type:** OAuth 2.0 + OpenID Connect (Authentication/SSO)

---

## Executive Summary

This document describes the OAuth 2.0 / OpenID Connect (OIDC) authentication flow between TMS (Identity Provider) and Oracle MyLearn (Service Provider). This enables Oracle users to authenticate using their TMS credentials for Single Sign-On (SSO).

**TMS Role**: Identity Provider (IdP)
**Oracle Role**: Service Provider (SP) / Relying Party
**Protocol**: OAuth 2.0 (RFC 6749) + OpenID Connect Core 1.0

---

## Integration Overview

### Purpose
Enable Oracle MyLearn users to authenticate using TMS credentials without separate account creation.

### Flow Type
**Authorization Code Flow** (most secure OAuth 2.0 flow)

### Key Features
- ✅ Single Sign-On (SSO)
- ✅ Auto-approval (no consent screen)
- ✅ Secure token-based authentication
- ✅ Automatic OIDC Discovery

---

## User Journey - Sequence Diagram

```
┌─────────┐      ┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  User   │      │Oracle MyLearn│      │   TMS IdP    │      │  TMS User    │
│         │      │   (Service   │      │   (Identity  │      │  Database    │
│         │      │   Provider)  │      │   Provider)  │      │              │
└────┬────┘      └──────┬───────┘      └──────┬───────┘      └──────┬───────┘
     │                  │                     │                     │
     │ 1. Access Oracle MyLearn                │                     │
     │    (tries to view course)               │                     │
     ├─────────────────>│                     │                     │
     │                  │                     │                     │
     │                  │ 2. Check if user authenticated            │
     │                  │    (no session found)                     │
     │                  │                     │                     │
     │                  │ 3. Redirect to TMS for authentication     │
     │                  │    GET /oauth/authorize                   │
     │                  │    ?client_id=<oracle_client_id>          │
     │                  │    &redirect_uri=<oracle_callback>        │
     │                  │    &response_type=code                    │
     │                  │    &scope=openid+email+profile            │
     │                  │    &state=<random_csrf_token>             │
     │                  ├────────────────────>│                     │
     │                                        │                     │
     │ 4. Redirect to TMS login page          │                     │
     │<───────────────────────────────────────┤                     │
     │                                        │                     │
     │ 5. Enter TMS credentials               │                     │
     │    POST /api/v1/users/sign_in          │                     │
     ├───────────────────────────────────────>│                     │
     │                                        │                     │
     │                                        │ 6. Validate credentials
     │                                        ├────────────────────>│
     │                                        │                     │
     │                                        │ 7. User found & valid
     │                                        │<────────────────────┤
     │                                        │                     │
     │                                        │ 8. Check if Oracle client
     │                                        │    requires approval
     │                                        │    (Oracle = auto-approved)
     │                                        │                     │
     │ 9. Generate authorization code         │                     │
     │    (single-use, 10-min expiry)         │                     │
     │<───────────────────────────────────────┤                     │
     │                                        │                     │
     │ 10. Redirect back to Oracle            │                     │
     │     with authorization code            │                     │
     │     <oracle_callback>?code=<auth_code> │                     │
     │                        &state=<same_state>                   │
     ├─────────────────>│                     │                     │
     │                  │                     │                     │
     │                  │ 11. Validate state parameter              │
     │                  │     (CSRF protection)                     │
     │                  │                     │                     │
     │                  │ 12. Exchange code for tokens              │
     │                  │     POST /oauth/token                     │
     │                  │     grant_type=authorization_code         │
     │                  │     code=<auth_code>                      │
     │                  │     client_id=<oracle_client_id>          │
     │                  │     client_secret=<oracle_secret>         │
     │                  │     redirect_uri=<oracle_callback>        │
     │                  ├────────────────────>│                     │
     │                  │                     │                     │
     │                  │                     │ 13. Validate code   │
     │                  │                     │     - Not expired?  │
     │                  │                     │     - Not used?     │
     │                  │                     │     - Client valid? │
     │                  │                     │                     │
     │                  │                     │ 14. Generate tokens │
     │                  │                     │     - Access Token  │
     │                  │                     │     - ID Token (JWT)│
     │                  │                     │     - Refresh Token │
     │                  │                     │                     │
     │                  │ 15. Return tokens   │                     │
     │                  │     {               │                     │
     │                  │       access_token,  │                     │
     │                  │       id_token,      │                     │
     │                  │       refresh_token, │                     │
     │                  │       token_type: "Bearer",               │
     │                  │       expires_in: 7200                    │
     │                  │     }               │                     │
     │                  │<────────────────────┤                     │
     │                  │                     │                     │
     │                  │ 16. Validate ID Token signature           │
     │                  │     - Fetch JWKS from /oauth/discovery/keys
     │                  │     - Verify RS256 signature              │
     │                  │     - Check expiration                    │
     │                  │                     │                     │
     │                  │ 17. Extract user claims from ID Token     │
     │                  │     {               │                     │
     │                  │       sub: "12345", │                     │
     │                  │       email: "user@example.com",          │
     │                  │       name: "John Doe",                   │
     │                  │       email_verified: true                │
     │                  │     }               │                     │
     │                  │                     │                     │
     │                  │ 18. (Optional) Fetch additional user data │
     │                  │     GET /oauth/userinfo                   │
     │                  │     Authorization: Bearer <access_token>  │
     │                  ├────────────────────>│                     │
     │                  │                     │                     │
     │                  │                     │ 19. Validate token  │
     │                  │                     ├────────────────────>│
     │                  │                     │                     │
     │                  │                     │ 20. Return user data│
     │                  │                     │<────────────────────┤
     │                  │                     │                     │
     │                  │ 21. UserInfo response                     │
     │                  │<────────────────────┤                     │
     │                  │                     │                     │
     │                  │ 22. Create/update Oracle user account     │
     │                  │     - Link to TMS user ID (sub)           │
     │                  │     - Update email, name if changed       │
     │                  │                     │                     │
     │                  │ 23. Create Oracle session                 │
     │                  │     - Store tokens securely               │
     │                  │     - Set session cookie                  │
     │                  │                     │                     │
     │ 24. Grant access to Oracle MyLearn     │                     │
     │<─────────────────┤                     │                     │
     │                  │                     │                     │
     │ 25. User accesses Oracle course        │                     │
     │    (fully authenticated)               │                     │
     │                  │                     │                     │
```

---

## Detailed Step-by-Step Flow

### Phase 1: User Initiates Access (Steps 1-4)

**Step 1: User Accesses Oracle**
- User navigates to Oracle MyLearn
- Tries to access a course or resource

**Step 2: Oracle Checks Authentication**
- Oracle checks for existing session
- No session found (user not logged in)

**Step 3: Oracle Redirects to TMS**
Oracle constructs authorization URL and redirects:

```
https://YOUR-TMS-DOMAIN.com/oauth/authorize?
  client_id=ORACLE_CLIENT_ID
  &redirect_uri=https://oracle-mylearn.com/auth/callback
  &response_type=code
  &scope=openid+email+profile
  &state=abc123random
```

**Parameters:**
- `client_id`: Oracle's OAuth client identifier
- `redirect_uri`: Where TMS will send user back
- `response_type=code`: Authorization code flow
- `scope`: Requested user data (openid required, email + profile optional)
- `state`: CSRF protection token (Oracle generates, TMS echoes back)

**Step 4: Browser Redirects**
User's browser follows redirect to TMS login page

---

### Phase 2: User Authentication (Steps 5-9)

**Step 5: User Logs In**
- If not already logged into TMS:
  - User sees TMS login screen
  - Enters email and password
- If already logged in:
  - Skip to Step 8 (seamless SSO)

**Step 6-7: TMS Validates Credentials**
- TMS checks credentials against database
- Verifies user exists and is not deleted
- Checks password hash

**Step 8: Auto-Approval Check**
TMS checks if Oracle client requires user consent:

```ruby
# config/initializers/doorkeeper.rb
skip_authorization do |resource_owner, client|
  client.name == 'Oracle MyLearn'  # Auto-approved
end
```

Result: **No consent screen shown** - user automatically approved

**Step 9: Generate Authorization Code**
TMS creates one-time authorization code:
- **Format**: Random secure token
- **Expiration**: 10 minutes
- **Single-use**: Destroyed after token exchange
- **Stored**: In `oauth_access_grants` table

---

### Phase 3: Callback to Oracle (Step 10)

TMS redirects user back to Oracle:

```
https://oracle-mylearn.com/auth/callback?
  code=a1b2c3d4e5f6g7h8i9j0
  &state=abc123random
```

**Parameters:**
- `code`: One-time authorization code
- `state`: Same value Oracle sent (CSRF verification)

---

### Phase 4: Token Exchange (Steps 11-15)

**Step 11: Oracle Validates State**
Oracle verifies `state` matches what it sent (prevents CSRF attacks)

**Step 12: Oracle Exchanges Code for Tokens**
Oracle makes back-channel POST request to TMS:

```http
POST https://YOUR-TMS-DOMAIN.com/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=a1b2c3d4e5f6g7h8i9j0
&client_id=ORACLE_CLIENT_ID
&client_secret=ORACLE_CLIENT_SECRET
&redirect_uri=https://oracle-mylearn.com/auth/callback
```

**Step 13: TMS Validates Request**
- Code exists and not expired?
- Code not already used?
- Client credentials correct?
- Redirect URI matches?

**Step 14: TMS Generates Tokens**

Three tokens created:

1. **Access Token** (opaque or JWT)
   - Used for API calls
   - Expiration: 2 hours
   - Can be refreshed

2. **ID Token** (JWT - signed)
   - Contains user identity claims
   - Expiration: 1 hour
   - Signed with RS256

3. **Refresh Token**
   - Used to get new access tokens
   - Long-lived
   - Rotates on each use

**Step 15: TMS Returns Tokens**

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "Bearer",
  "expires_in": 7200,
  "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "scope": "openid email profile",
  "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "created_at": 1704542400
}
```

---

### Phase 5: Token Validation & User Data (Steps 16-21)

**Step 16: Oracle Validates ID Token**

Oracle verifies the JWT signature:

1. Fetch TMS public keys:
   ```
   GET https://YOUR-TMS-DOMAIN.com/oauth/discovery/keys
   ```

2. Verify signature using RS256 algorithm

3. Check standard claims:
   - `iss` (issuer) matches TMS URL
   - `aud` (audience) matches Oracle client_id
   - `exp` (expiration) not passed
   - `iat` (issued at) reasonable

**Step 17: Extract User Claims**

Decode ID Token payload:

```json
{
  "sub": "12345",
  "iss": "https://YOUR-TMS-DOMAIN.com",
  "aud": "ORACLE_CLIENT_ID",
  "exp": 1704546000,
  "iat": 1704542400,
  "email": "user@example.com",
  "email_verified": true,
  "name": "John Doe",
  "given_name": "John",
  "family_name": "Doe",
  "preferred_username": "johndoe",
  "locale": "en"
}
```

**Key Claims:**
- `sub`: Unique TMS user ID (use as primary identifier)
- `email`: User's email address
- `email_verified`: Email confirmation status
- `name`: Full name
- `locale`: Language preference

**Step 18-19: Fetch Additional Data (Optional)**

If Oracle needs more user info:

```http
GET https://YOUR-TMS-DOMAIN.com/oauth/userinfo
Authorization: Bearer <access_token>
```

**Step 20-21: UserInfo Response**

Returns same user claims as ID token (for verification)

---

### Phase 6: User Session Creation (Steps 22-25)

**Step 22: Oracle Creates/Updates User**

Oracle creates or updates user record:
- **Primary Key**: `sub` from ID token (TMS user ID)
- **Email**: From `email` claim
- **Name**: From `name` claim
- **Profile**: Additional claims

**Step 23: Oracle Creates Session**

Oracle establishes authenticated session:
- Store tokens securely (encrypted)
- Set session cookie
- Associate with Oracle user account

**Step 24-25: Access Granted**

User is fully authenticated and can access Oracle MyLearn resources

---

## Token Refresh Flow

When access token expires, Oracle can refresh without user interaction:

```
┌──────────────┐                     ┌──────────────┐
│Oracle MyLearn│                     │   TMS IdP    │
└──────┬───────┘                     └──────┬───────┘
       │                                    │
       │ 1. Access token expired            │
       │                                    │
       │ 2. POST /oauth/token               │
       │    grant_type=refresh_token        │
       │    refresh_token=<old_token>       │
       │    client_id=<client_id>           │
       │    client_secret=<secret>          │
       ├───────────────────────────────────>│
       │                                    │
       │                                    │ 3. Validate refresh token
       │                                    │    - Not expired?
       │                                    │    - Not revoked?
       │                                    │    - Client valid?
       │                                    │
       │                                    │ 4. Generate new tokens
       │                                    │    - New access token
       │                                    │    - New refresh token
       │                                    │    - Revoke old refresh token
       │                                    │
       │ 5. Return new tokens               │
       │    {                               │
       │      access_token: "new...",       │
       │      refresh_token: "new...",      │
       │      expires_in: 7200              │
       │    }                               │
       │<───────────────────────────────────┤
       │                                    │
       │ 6. Update stored tokens            │
       │    Continue user session           │
       │                                    │
```

**Benefits:**
- User stays logged in seamlessly
- No re-authentication required
- Automatic token rotation for security

---

## Logout Flow

When user logs out of Oracle:

```
┌──────────────┐                     ┌──────────────┐
│Oracle MyLearn│                     │   TMS IdP    │
└──────┬───────┘                     └──────┬───────┘
       │                                    │
       │ 1. User clicks logout              │
       │                                    │
       │ 2. POST /oauth/revoke              │
       │    token=<access_or_refresh_token> │
       │    client_id=<client_id>           │
       │    client_secret=<secret>          │
       ├───────────────────────────────────>│
       │                                    │
       │                                    │ 3. Revoke token
       │                                    │    Mark as revoked in DB
       │                                    │
       │ 4. Token revoked                   │
       │<───────────────────────────────────┤
       │                                    │
       │ 5. Clear Oracle session            │
       │    Delete stored tokens            │
       │    Redirect to logout page         │
       │                                    │
```

**Note:** This logs user out of Oracle, but NOT out of TMS. If user visits Oracle again, they may be auto-logged in if TMS session still active.

---

## OIDC Discovery Flow

Oracle can auto-configure using OIDC Discovery:

```
┌──────────────┐                     ┌──────────────┐
│Oracle MyLearn│                     │   TMS IdP    │
│Configuration │                     │              │
└──────┬───────┘                     └──────┬───────┘
       │                                    │
       │ 1. Admin enters discovery URL      │
       │    in Oracle config                │
       │                                    │
       │ 2. GET /.well-known/openid-configuration
       ├───────────────────────────────────>│
       │                                    │
       │ 3. Return configuration            │
       │    {                               │
       │      issuer: "https://...",        │
       │      authorization_endpoint: "...",│
       │      token_endpoint: "...",        │
       │      userinfo_endpoint: "...",     │
       │      jwks_uri: "...",              │
       │      scopes_supported: [...],      │
       │      ...                           │
       │    }                               │
       │<───────────────────────────────────┤
       │                                    │
       │ 4. Auto-populate all settings      │
       │    Save configuration              │
       │                                    │
```

**Discovery URL:**
```
https://YOUR-TMS-DOMAIN.com/.well-known/openid-configuration
```

---

## Security Features

### CSRF Protection
- **State parameter**: Random value generated by Oracle
- **Validation**: TMS echoes back, Oracle verifies match
- **Prevents**: Cross-Site Request Forgery attacks

### Replay Attack Prevention
- **Authorization codes**: Single-use only
- **Expiration**: 10 minutes
- **Destruction**: Code deleted after token exchange

### Token Signature Verification
- **Algorithm**: RS256 (RSA with SHA-256)
- **Key Size**: RSA-2048
- **Public Keys**: Available at JWKS endpoint
- **Validation**: Oracle verifies all JWT signatures

### Redirect URI Validation
- **Whitelist**: Only registered URIs allowed
- **Exact Match**: No wildcards or subdomain matching
- **Prevents**: Open redirect attacks

### Client Authentication
- **Confidential Client**: Client secret required
- **Back-channel Only**: Secrets never exposed to browser
- **Secure Storage**: Encrypted storage recommended

### Token Expiration
| Token Type | Expiration | Renewal |
|------------|------------|---------|
| Authorization Code | 10 minutes | N/A (single-use) |
| Access Token | 2 hours | Via refresh token |
| ID Token | 1 hour | N/A (re-authenticate) |
| Refresh Token | Long-lived | Rotates on use |

### HTTPS Enforcement
- **Production**: All endpoints require HTTPS
- **Development**: HTTP allowed (localhost only)
- **Certificate**: Valid SSL/TLS certificate required

---

## Error Scenarios

### User Not Found
```
Step 6: TMS validates credentials
        ↓
     ✗ User not found in database
        ↓
     Return: 401 Unauthorized
     Display: "Invalid email or password"
```

### Invalid Client Credentials
```
Step 13: TMS validates token request
         ↓
      ✗ Client secret doesn't match
         ↓
      Return: 401 Unauthorized
      Response: {
        "error": "invalid_client",
        "error_description": "Client authentication failed"
      }
```

### Expired Authorization Code
```
Step 13: TMS validates authorization code
         ↓
      ✗ Code expired (>10 minutes old)
         ↓
      Return: 400 Bad Request
      Response: {
        "error": "invalid_grant",
        "error_description": "Authorization code expired"
      }
```

### Invalid State Parameter
```
Step 11: Oracle validates state
         ↓
      ✗ State doesn't match sent value
         ↓
      Display error to user
      Log security incident
```

### Invalid Redirect URI
```
Step 13: TMS validates redirect_uri
         ↓
      ✗ URI not registered for client
         ↓
      Return: 400 Bad Request
      Do NOT redirect (security)
```

---

## Configuration Details

### TMS Configuration (Identity Provider)

**Environment Variable Required:**
```bash
OIDC_ISSUER=https://your-production-domain.com
```

**Doorkeeper Settings:**
- Grant flows: `authorization_code`, `refresh_token`
- Scopes: `openid` (default), `email`, `profile`
- Token expiration: Codes 10min, Access 2hrs
- Auto-approval: Oracle MyLearn (no consent screen)

**OIDC Claims Mapping:**

| TMS Data Source | OIDC Claim | Scope | Required |
|-----------------|------------|-------|----------|
| `user.id` | `sub` | openid | Yes |
| `user.email` | `email` | email | No |
| `user.confirmed?` | `email_verified` | email | No |
| `profile.name` | `name` | profile | No |
| `profile.name` (first) | `given_name` | profile | No |
| `profile.name` (rest) | `family_name` | profile | No |
| `user.username` | `preferred_username` | profile | No |
| `profile.language` | `locale` | profile | No |
| `user.updated_at` | `updated_at` | profile | No |

**Fallback Values:**
- If no profile: `name` = `user.email`
- If no locale: `locale` = `'en'`

---

### Oracle Configuration (Service Provider)

**Required from TMS:**
1. Client ID
2. Client Secret (confidential - secure storage)
3. Discovery URL (recommended) OR individual endpoints

**Recommended Setup:**

Use OIDC Discovery URL:
```
https://YOUR-TMS-DOMAIN.com/.well-known/openid-configuration
```

This auto-populates:
- Issuer
- Authorization endpoint
- Token endpoint
- UserInfo endpoint
- JWKS URI
- Supported scopes
- Signing algorithms

**Manual Configuration (if discovery not supported):**

| Setting | Value |
|---------|-------|
| Issuer | `https://YOUR-TMS-DOMAIN.com` |
| Authorization Endpoint | `https://YOUR-TMS-DOMAIN.com/oauth/authorize` |
| Token Endpoint | `https://YOUR-TMS-DOMAIN.com/oauth/token` |
| UserInfo Endpoint | `https://YOUR-TMS-DOMAIN.com/oauth/userinfo` |
| JWKS URI | `https://YOUR-TMS-DOMAIN.com/oauth/discovery/keys` |
| Revocation Endpoint | `https://YOUR-TMS-DOMAIN.com/oauth/revoke` |

**Scopes to Request:**
```
openid email profile
```

**Response Type:**
```
code
```

**Grant Type:**
```
authorization_code
```

**Token Endpoint Auth Method:**
```
client_secret_post
OR
client_secret_basic
```

---

## User Data Mapping

### Recommended Mapping

Oracle should map TMS claims to Oracle user fields:

| Oracle Field | OIDC Claim | Source | Notes |
|--------------|------------|--------|-------|
| **User ID** | `sub` | User ID | **Primary key** - never changes |
| **Email** | `email` | user.email | May change - don't use as primary key |
| **Email Verified** | `email_verified` | user.confirmed? | Boolean |
| **Full Name** | `name` | profile.name | Display name |
| **First Name** | `given_name` | Extracted from name | Optional |
| **Last Name** | `family_name` | Extracted from name | Optional |
| **Username** | `preferred_username` | user.username | Optional |
| **Language** | `locale` | profile.language | For localization |
| **Last Updated** | `updated_at` | user.updated_at | Unix timestamp |

### Account Linking Strategy

**First-time User:**
1. Extract `sub` (TMS user ID)
2. Check if Oracle user exists with this `sub`
3. If not, create new Oracle user
4. Store `sub` as foreign key to TMS

**Returning User:**
1. Extract `sub` from ID token
2. Find Oracle user by `sub`
3. Update email/name if changed
4. Grant access

**Important:** Always use `sub` as the linking key, not `email` (email can change)

---

## Testing Checklist

### Pre-Integration Testing

- [ ] OIDC Discovery endpoint returns valid JSON
- [ ] JWKS endpoint returns public keys
- [ ] Client credentials created in TMS
- [ ] Redirect URI configured correctly
- [ ] HTTPS enforced in production

### Integration Testing

- [ ] User can initiate login from Oracle
- [ ] TMS login page displays correctly
- [ ] Successful authentication redirects back to Oracle
- [ ] Authorization code exchange succeeds
- [ ] ID token signature validates
- [ ] User claims map correctly to Oracle fields
- [ ] UserInfo endpoint returns expected data
- [ ] Session persists across Oracle pages
- [ ] Logout revokes tokens

### Security Testing

- [ ] Invalid state parameter rejected
- [ ] Expired authorization code rejected
- [ ] Wrong client secret rejected
- [ ] Invalid redirect URI rejected
- [ ] Token signature validation works
- [ ] CSRF attack prevented (state validation)
- [ ] Replay attack prevented (single-use codes)

---

## Summary for Oracle Team

### Integration Type
**OAuth 2.0 + OpenID Connect** (Authentication/SSO only)

### What This Provides
- ✅ User authentication via TMS credentials
- ✅ Single Sign-On (SSO)
- ✅ Automatic user account linking
- ✅ Secure token-based access
- ✅ No consent screen (auto-approved)

### What This Does NOT Provide
- ❌ Course launches (that would be LTI 1.3)
- ❌ Grade synchronization (that would be LTI AGS)
- ❌ Roster provisioning (that would be LTI NRPS)

### Flow Summary
1. User clicks "Login" on Oracle
2. Oracle redirects to TMS
3. User logs in (if not already)
4. TMS redirects back with code
5. Oracle exchanges code for tokens
6. Oracle validates tokens
7. Oracle creates session
8. User accesses Oracle

### Key Endpoints

**For Oracle to Call:**
- Authorization: `/oauth/authorize` (browser redirect)
- Token: `/oauth/token` (back-channel POST)
- UserInfo: `/oauth/userinfo` (optional, back-channel GET)
- JWKS: `/oauth/discovery/keys` (public keys)
- Revoke: `/oauth/revoke` (logout)

**For Oracle Configuration:**
- Discovery: `/.well-known/openid-configuration` (auto-setup)

---

## Next Steps

1. **Oracle Team**: Configure OAuth client using provided credentials
2. **Both Teams**: Schedule integration testing session
3. **Oracle Team**: Test authentication flow in staging
4. **Both Teams**: Verify user data mapping
5. **Both Teams**: Production deployment coordination

---

**Document Version:** 1.0
**Last Updated:** January 20, 2026
**Protocol:** OAuth 2.0 (RFC 6749) + OpenID Connect Core 1.0
**Environment:** Staging → Production
